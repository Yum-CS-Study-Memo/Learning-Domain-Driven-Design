# 낙관적 동시성 제어 (Optimistic Concurrency Control)

> 📖 *도메인 주도 설계 첫걸음* Ch05 — 간단한 비즈니스 로직 구현하기 (p.70~71)

---

## 1. 왜 필요한가? — 필요한 상황

여러 사용자가 **같은 데이터를 동시에 수정**하려 할 때 문제가 생긴다.

### 문제 상황: Lost Update (갱신 손실)

```
현재 DB: visits = 5

사용자 A가 읽음: visits = 5  →  5+1 = 6으로 쓰려고 준비 중...
사용자 B가 읽음: visits = 5  →  5+1 = 6으로 쓰려고 준비 중...

A가 먼저 씀: visits = 6  ✅
B가 그 다음 씀: visits = 6  💥  (7이 돼야 하는데!)
```

방문이 **2번** 일어났는데 visits는 **1만** 증가했다.  
B가 A의 작업을 덮어씌워버린 것 — 이게 **Lost Update(갱신 손실)** 문제다.

---

## 2. 예시 코드

### 예시 코드 1 — 문제가 있는 코드 (visits를 값으로 덮어씌움)

```csharp
public class LogVisit
{
    public void Execute(Guid userId, long visits)
    {
        _db.Execute(
            "UPDATE Users SET visits = @p1 WHERE user_id = @p2",
            visits,   // 호출자가 계산한 값을 그대로 덮어씌움
            userId
        );
    }
}
```

```
A가 읽음: 5  →  6으로 덮어씌우려 함
B가 읽음: 5  →  6으로 덮어씌우려 함

A 실행: visits = 6  ✅
B 실행: visits = 6  💥  (A의 작업이 사라짐)
```

> 이 코드는 멱등성(같은 요청을 여러 번 해도 결과가 동일)은 보장하지만,  
> **동시에 여러 명이 수정할 때의 충돌은 막지 못한다.**

---

### 예시 코드 2 — 낙관적 동시성 제어를 적용한 코드

```csharp
public class LogVisit
{
    public void Execute(Guid userId, long expectedVisits)
    {
        _db.Execute(
            @"UPDATE Users SET visits = visits + 1
              WHERE user_id = @p1
              AND visits = @p2",   // 내가 읽은 값과 현재 DB 값이 같을 때만 실행
            userId,
            expectedVisits         // 내가 처음 읽어온 값
        );
    }
}
```

```
A가 읽음: expectedVisits = 5
B가 읽음: expectedVisits = 5

A 실행: WHERE visits = 5 → DB도 5 → ✅ 성공!  visits = 6
B 실행: WHERE visits = 5 → DB는 이미 6 → ❌ 0행 업데이트 (충돌 감지!)

B는 실패를 감지 → 다시 읽음: visits = 6 → 재시도 → visits = 7  ✅
```

> `AND visits = @p2` 조건 하나만으로 충돌을 감지할 수 있다.

---

## 3. 낙관적 동시성 제어란?

> "아마 내가 읽는 동안 아무도 안 건드리겠지~ 일단 해보고, 충돌나면 그때 처리하자"

UPDATE를 시도할 때 **내가 읽었던 값이 아직도 그대로인지** 같이 확인하는 방식이다.

- 충돌이 **없으면** → 그냥 성공
- 충돌이 **있으면** → UPDATE가 0건으로 끝나므로 감지 가능 → 재시도

반대 개념인 **비관적 동시성 제어**는 "누군가 건드릴 수도 있으니 미리 잠가놓자"는 방식으로, `SELECT FOR UPDATE` 같은 락(Lock)을 미리 걸어둔다.

| | 비관적 | 낙관적 |
|---|---|---|
| 전략 | 미리 잠금(Lock) | 일단 진행, 충돌 시 감지 |
| 비유 | 화장실 들어가기 전에 잠금 | 들어갔더니 누가 있으면 나옴 |
| 성능 | 느림 | 빠름 |
| 적합한 상황 | 충돌이 잦을 때 | 충돌이 드물 때 |

---

## 4. 이를 통해 무엇을 해결할 수 있을까?

### ✅ Lost Update 방지
여러 사용자가 동시에 같은 데이터를 수정할 때 한쪽의 변경이 사라지는 문제를 막는다.

### ✅ 락 없이 동시성 보장
DB 락을 걸지 않아도 되므로 **성능 저하 없이** 안전한 업데이트가 가능하다.

### ✅ 충돌 감지 후 재시도 가능
0행 업데이트라는 명확한 신호로 충돌을 감지하고, 애플리케이션 레벨에서 재시도 로직을 구현할 수 있다.

---

## 5. 결론

낙관적 동시성 제어는 **충돌이 드문 상황**에서 락 없이 데이터 정합성을 지키는 방법이다.

핵심은 단순하다:

```sql
WHERE user_id = @p1 AND visits = @p2
-- "내가 읽은 값이 아직 그대로일 때만 업데이트해"
```

이 조건 하나가 Lost Update를 막아준다.  
단, 충돌이 발생했을 때 **재시도 로직은 호출자(애플리케이션)가 직접 처리**해야 한다는 점을 기억하자.