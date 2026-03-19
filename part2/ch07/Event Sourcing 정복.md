# 이벤트 소싱(Event Sourcing) 완전 정복

---

## 0. 읽기 전에: 이 글의 목표

이 글은 단순히 "이벤트 소싱이 뭔지" 정의를 외우는 게 아니라,  
"왜 이런 게 생겼는지", "실제로 코드가 어떻게 동작하는지", "왜 이렇게 복잡하게 짜는지"를  
처음 접하는 사람도 이해할 수 있도록 설명한다.

---

## 1. 근본적인 질문: 데이터를 어떻게 저장해야 할까?

프로그램은 결국 데이터를 다루는 것이다. 그런데 데이터를 저장하는 방식에는 두 가지 철학이 있다.

### 철학 1: "지금 상태만 중요하다" (상태 기반 저장)

일반적인 애플리케이션은 현재 상태만 DB에 저장한다. 은행 계좌를 예로 들면:

```
처음: 잔액 = 0원
→ 200만원 입금 → DB 업데이트: 잔액 = 200만원
→ 50만원 출금  → DB 업데이트: 잔액 = 150만원
→ 50만원 출금  → DB 업데이트: 잔액 = 100만원
```

최종적으로 DB에는 `잔액: 100만원` 만 남는다.  
중간 과정은 모두 덮어쓰여서 사라진다.

**문제**: "이 돈이 어디서 왔고 어디로 갔지?"를 알 방법이 없다.  
나중에 "왜 잔액이 100만원이야?"라고 물어봤을 때 DB는 그냥 "100만원이에요"라고만 답할 수 있다.

### 철학 2: "일어난 일이 중요하다" (이벤트 소싱)

이벤트 소싱은 현재 상태 대신, **일어난 사건들을 순서대로 모두 저장**한다.

```
이벤트 스토어:
1. MoneyDeposited(amount: 200만원, timestamp: 2024-01-01 09:00)
2. MoneyWithdrawn(amount: 50만원,  timestamp: 2024-01-02 14:00)
3. MoneyWithdrawn(amount: 50만원,  timestamp: 2024-01-03 11:00)
```

"현재 잔액이 얼마야?"라는 질문에 답하려면 이 이벤트들을 처음부터 재계산한다:  
0 + 200만 - 50만 - 50만 = **100만원**

이벤트는 절대 수정하거나 삭제하지 않는다. 오직 추가만 할 수 있다.  
이것을 **불변성(Immutability)** 이라고 한다.

> 💡 사실 이벤트 소싱은 새로운 개념이 아니다. 금융 산업에서는 수백 년 전부터 써왔다.  
> 회계 장부(원장)가 바로 이벤트 소싱이다. 원장은 모든 거래를 순서대로 기록하고,  
> 절대 지우지 않는다. 잔액은 원장의 기록을 처음부터 합산해서 계산한다.

---

## 2. 핵심 용어 완전 정리

### 애그리게이트 (Aggregate)

애그리게이트는 **하나의 일관된 단위로 취급되는 객체 묶음**이다.  
쉽게 말하면 "함께 움직이는 데이터 덩어리"다.

은행 계좌를 생각해보자. 계좌에는 계좌번호, 잔액, 소유자 정보, 거래 한도 등 여러 데이터가 있다.  
이것들은 항상 함께 읽히고 함께 변경된다. 이 묶음 전체가 하나의 애그리게이트다.

이벤트 소싱에서 중요한 것은 **이벤트는 항상 특정 애그리게이트에 속한다**는 것이다.  
계좌 A의 이벤트와 계좌 B의 이벤트는 서로 완전히 독립적이다.  
"계좌 A의 현재 상태를 알고 싶다"면 계좌 A의 이벤트만 재생하면 된다.

### 도메인 이벤트 (Domain Event)

도메인 이벤트는 **비즈니스적으로 의미 있는 사건**을 나타낸다.  
중요한 특징은 **과거형**으로 표현한다는 것이다. "~이 발생했다"는 뜻이다.

- `TicketCreated` (티켓이 생성됐다)
- `TicketEscalated` (티켓이 에스컬레이션 됐다)
- `MoneyDeposited` (돈이 입금됐다)
- `OrderPlaced` (주문이 접수됐다)

이벤트는 이미 일어난 사실이기 때문에 **취소하거나 수정할 수 없다**.  
잘못됐다면 그것을 되돌리는 새 이벤트를 추가해야 한다.

### 명령 (Command) vs 이벤트 (Event)

이벤트 소싱을 이해할 때 명령과 이벤트의 차이를 구분하는 것이 매우 중요하다.

| 구분 | 명령 (Command) | 이벤트 (Event) |
|------|--------------|--------------|
| 시제 | 현재/미래형: "~해라" | 과거형: "~이 됐다" |
| 예시 | RequestEscalation | TicketEscalated |
| 의미 | 아직 안 일어남, 실패할 수도 있음 | 이미 일어남, 되돌릴 수 없음 |
| 결과 | 비즈니스 로직 판단 후 이벤트 발생 | 상태 변경의 원인 |

명령이 들어오면 시스템이 판단한다. "이 명령을 실행할 수 있는가?"  
조건이 맞으면 이벤트를 발생시키고, 조건이 안 맞으면 아무것도 안 한다.

예를 들어 `RequestEscalation` 명령이 들어왔을 때:
- 이미 에스컬레이션 됐거나, 아직 처리 시간이 남아있다면 → 아무 이벤트도 발생하지 않음
- 에스컬레이션이 안 됐고, 처리 시간이 초과됐다면 → `TicketEscalated` 이벤트 발생

### 프로젝션 (Projection)

프로젝션은 **이벤트 목록을 처음부터 순서대로 재생해서 현재 상태를 계산하는 과정**이다.  
영어로 "project"는 "투영하다"는 뜻인데, 이벤트들을 상태 공간에 투영해서 현재 상태를 만든다고 이해하면 된다.

```
이벤트1 → 이벤트2 → 이벤트3 → ... → 이벤트N
                                         ↓
                                    현재 상태 (프로젝션 결과)
```

프로젝션은 **순수 함수**처럼 동작해야 한다.  
같은 이벤트 목록을 주면 항상 같은 결과가 나와야 한다.  
이것이 보장되면 언제든지 이벤트를 처음부터 재생해서 상태를 복원할 수 있다.

### 리하이드레이션 (Rehydration)

리하이드레이션은 프로젝션과 거의 같은 말이지만, 특히 **DB에서 이벤트를 불러와서  
메모리 상의 애그리게이트 객체를 되살리는 과정**을 강조할 때 쓴다.

"rehydrate"는 말 그대로 "다시 수분을 보충하다"는 뜻이다.  
이벤트 스토어에는 사건 기록만 있고(건조된 상태), 이것을 재생하면 살아있는 객체가 된다(수분 보충).

코드에서는 `new Ticket(events)` 처럼 생성자에 이벤트 목록을 넘겨서 객체를 만드는 방식으로 구현된다.

### 이벤트 스토어 (Event Store)

이벤트 스토어는 **이벤트만 저장하는 전용 데이터베이스**다.  
일반 DB와 결정적으로 다른 점은 **수정(UPDATE)과 삭제(DELETE)가 없다**는 것이다.  
오직 읽기(READ)와 추가(APPEND)만 가능하다. 이것을 **append-only** 저장소라고 한다.

왜 수정/삭제를 막는가? 이벤트는 이미 일어난 사실이기 때문이다.  
과거를 수정한다는 것은 역사를 조작하는 것과 같다.  
이것을 허용하면 이벤트 소싱의 핵심 이점(완전한 감사 추적, 시간 여행)이 무너진다.

---

## 3. 이벤트 스토어 인터페이스 코드 분석

```csharp
interface IEventStore
{
    IEnumerable<Event> Fetch(Guid instanceId);
    void Append(Guid instanceId, Event[] newEvents, int expectedVersion);
}
```

딱 두 개의 메서드만 있다. 이것이 이벤트 스토어의 최소 요건이다.

### Fetch: 이벤트 목록 조회

```csharp
IEnumerable<Event> Fetch(Guid instanceId);
```

`instanceId`는 어떤 애그리게이트의 이벤트를 가져올지 지정하는 ID다.  
`IEnumerable<Event>`는 이벤트들의 순서 있는 목록을 반환한다.  
이 목록은 **시간 순서대로 정렬**되어 있어야 한다. 순서가 틀리면 프로젝션 결과가 달라진다.

### Append: 새 이벤트 추가

```csharp
void Append(Guid instanceId, Event[] newEvents, int expectedVersion);
```

`newEvents`는 새로 추가할 이벤트 배열이다. 한 번의 작업에서 여러 이벤트가 발생할 수 있으므로 배열로 받는다.

`expectedVersion`이 핵심이다. 이것이 **낙관적 동시성 제어(Optimistic Concurrency Control)** 를 가능하게 한다.

---

## 4. 낙관적 동시성 제어 (Optimistic Concurrency Control) 깊게 이해하기

### 왜 필요한가?

분산 시스템이나 멀티스레드 환경에서 같은 데이터를 동시에 수정하려는 경우가 생긴다.  
이것을 처리하는 방법에는 두 가지 철학이 있다.

**비관적 동시성 제어 (Pessimistic)**: "충돌이 날 것 같으니 미리 잠금(Lock)을 건다"  
→ 한 사람이 데이터를 사용하는 동안 다른 사람은 기다려야 한다. 안전하지만 느리다.

**낙관적 동시성 제어 (Optimistic)**: "충돌이 별로 없을 테니 일단 진행하고, 충돌 시에만 막는다"  
→ 잠금 없이 진행하다가 저장할 때 버전을 체크한다. 충돌이 드물 때 훨씬 빠르다.

### 어떻게 동작하는가?

```
시나리오: 두 사용자가 동시에 티켓 #42를 수정하려 한다

[사용자 A] 티켓 #42 읽음 → 현재 버전: 5
[사용자 B] 티켓 #42 읽음 → 현재 버전: 5

[사용자 A] 이벤트 추가 시도 → expectedVersion=5
  → DB 확인: 현재 버전이 5가 맞음 → ✅ 성공! → 버전이 6으로 증가

[사용자 B] 이벤트 추가 시도 → expectedVersion=5
  → DB 확인: 현재 버전이 6임! (A가 이미 바꿔놨음) → ❌ 동시성 예외 발생!
```

사용자 B는 예외를 받고 처음부터 다시 시도해야 한다.  
다시 시도하면 이번엔 버전 6으로 읽고, 새 이벤트를 추가해서 버전 7을 만들 수 있다.

### 왜 "낙관적"인가?

대부분의 경우 두 사람이 완전히 동시에 같은 데이터를 수정하는 일은 드물다.  
그래서 "충돌이 별로 없을 거라고 낙관하고" 잠금 없이 진행한다.  
충돌이 날 때만 재시도하는 방식이라 평균적으로 훨씬 빠르다.

---

## 5. 이벤트 소싱 애그리게이트 전체 흐름 (TicketAPI 코드 완전 분석)

### 전체 코드 다시 보기

```csharp
// TicketAPI (1~17번 줄) - 외부 요청을 받아서 흐름 조율
public void RequestEscalation(TicketId id, EscalationReason reason)
{
    var events = _ticketsRepository.LoadEvents(id);           // 8번 줄
    var ticket = new Ticket(events);                           // 9번 줄
    var originalVersion = ticket.Version;                      // 10번 줄
    var cmd = new RequestEscalation(reason);                   // 11번 줄
    ticket.Execute(cmd);                                       // 12번 줄
    _ticketsRepository.CommitChanges(ticket, originalVersion); // 13번 줄
}
```

이 6줄의 흐름이 이벤트 소싱의 핵심 패턴이다. 각 줄을 깊이 파고들어 보자.

---

### 8번 줄: 과거 이벤트 전부 불러오기

```csharp
var events = _ticketsRepository.LoadEvents(id);
```

이벤트 스토어에서 이 티켓 ID에 대한 **모든 과거 이벤트**를 시간 순서로 가져온다.  
처음 만들어진 순간부터 지금까지 이 티켓에 일어난 모든 일이 여기 담겨 있다.

예를 들면 이런 식이다:
```
[
  TicketCreated(id: #42, title: "로그인 안됨", timestamp: 2024-01-01),
  CommentAdded(content: "확인 중입니다", timestamp: 2024-01-02),
  PriorityChanged(newPriority: HIGH, timestamp: 2024-01-03),
  ...
]
```

---

### 9번 줄: 리하이드레이션 (프로젝션)

```csharp
var ticket = new Ticket(events);
```

이것이 **프로젝션이자 리하이드레이션**이다.  
빈 `Ticket` 객체를 만들고, 과거 이벤트들을 처음부터 하나씩 재생해서 현재 상태를 복원한다.

내부적으로는 이런 일이 일어난다:

```csharp
// Ticket 생성자 (25~32번 줄)
public Ticket(IEnumerable<IDomainEvents> events)
{
    _state = new TicketState();   // 빈 상태로 시작 (모든 필드가 기본값)
    foreach(var e in events)
    {
        AppendEvent(e);           // 이벤트 하나씩 재생
    }
}
```

`TicketState`는 처음에 아무 데이터도 없는 빈 껍데기다.  
이벤트를 하나씩 `AppendEvent`에 넣으면서 상태가 채워진다.  
모든 이벤트를 다 재생하고 나면 `_state`에는 **현재 상태**가 담겨 있다.

왜 생성자에서 이렇게 하는가? 객체를 만들 때부터 올바른 상태를 보장하기 위해서다.  
`Ticket` 객체가 존재한다는 것은 항상 이벤트가 재생된 올바른 상태임을 의미한다.

---

### 10번 줄: 버전 기억

```csharp
var originalVersion = ticket.Version;
```

리하이드레이션이 끝난 직후의 버전을 기억해둔다.  
나중에 13번 줄에서 이벤트를 저장할 때 이 값을 `expectedVersion`으로 사용한다.  
"내가 이 시점에 읽었을 때 버전이 X였으니, 저장할 때도 X여야 한다"는 의미다.

---

### 11번 줄: 명령 객체 생성

```csharp
var cmd = new RequestEscalation(reason);
```

명령(Command)은 **"무언가를 해달라"는 의도를 담은 객체**다.  
여기서는 "에스컬레이션을 요청한다"는 명령을 만든다. `reason`에는 왜 에스컬레이션이 필요한지 이유가 담긴다.

명령과 이벤트를 분리하는 이유는 명확한 의도 표현을 위해서다.  
함수에 원시 값을 넘기는 대신 명령 객체를 만들면, 이 작업이 무엇을 의도하는지 코드만 봐도 알 수 있다.

---

### 12번 줄: 비즈니스 로직 실행

```csharp
ticket.Execute(cmd);
```

이것이 **비즈니스 판단이 일어나는 곳**이다.

```csharp
// Execute 메서드 (39~44번 줄)
public void Execute(RequestEscalation cmd)
{
    if (!_state.IsEscalated && _state.RemainingTimePercentage <= 0)
    {
        var escalatedEvent = new TicketEscalated(_id, cmd.Reason);
        AppendEvent(escalatedEvent);
    }
}
```

여기서 두 가지 조건을 확인한다:
- `!_state.IsEscalated`: 아직 에스컬레이션이 안 된 상태여야 함 (중복 방지)
- `_state.RemainingTimePercentage <= 0`: 처리 시간이 다 됐어야 함 (규칙 준수)

두 조건이 모두 충족되면 `TicketEscalated` 이벤트를 만들어서 `AppendEvent`에 넘긴다.  
조건이 충족되지 않으면 아무 일도 일어나지 않는다. 이벤트도 없고 상태 변화도 없다.

**중요**: `IsEscalated = true`로 직접 바꾸지 않는다!  
상태를 직접 바꾸면 그 변경이 기록되지 않는다. 반드시 이벤트를 통해서만 상태가 바뀌어야 한다.  
이것이 이벤트 소싱의 핵심 규칙이다.

---

### AppendEvent의 역할 (33~38번 줄)

```csharp
private void AppendEvent(IDomainEvent @event)
{
    _domainEvents.Append(@event);
    ((dynamic)_state).Apply((dynamic)@event);
}
```

`AppendEvent`는 두 가지 일을 동시에 한다.

**첫째**: `_domainEvents` 리스트에 이벤트를 추가한다.  
이 리스트는 아직 DB에 저장되지 않은 "새로 발생한 이벤트들"을 임시로 보관한다.  
나중에 13번 줄에서 이것들을 이벤트 스토어에 저장한다.

**둘째**: `_state.Apply(event)`를 호출해서 메모리 상의 상태를 즉시 바꾼다.  
`TicketEscalated` 이벤트가 오면 `TicketState.Apply(TicketEscalated)`가 호출되어  
`IsEscalated = true`가 된다.

`((dynamic)_state).Apply((dynamic)@event)` 이 이상해 보이는 코드는 C#의 동적 디스패치다.  
이벤트 타입이 `TicketEscalated`이면 `Apply(TicketEscalated)`를 찾아 호출하고,  
`TicketInitialized`이면 `Apply(TicketInitialized)`를 찾아 호출한다.  
이벤트 종류마다 `if-else`로 분기하지 않아도 된다.

---

### TicketState: 상태 표현만 담당 (50~71번 줄)

```csharp
public class TicketState
{
    public TicketId Id { get; private set; }
    public int Version { get; private set; }
    public bool IsEscalated { get; private set; }

    public void Apply(TicketInitialized @event)
    {
        Id = @event.Id;
        Version = 0;
        IsEscalated = false;
    }

    public void Apply(TicketEscalated @event)
    {
        IsEscalated = true;
        Version += 1;
    }
}
```

`TicketState`는 **상태값 보관 + 이벤트 적용** 만 담당한다.  
비즈니스 판단은 하지 않는다. 비즈니스 판단은 `Ticket.Execute()`가 한다.

`private set`은 외부에서 직접 값을 바꾸지 못하게 막는다.  
값을 바꾸는 유일한 방법은 `Apply` 메서드를 통해서다.  
즉, 이벤트 없이는 상태가 바뀔 수 없도록 구조적으로 강제하는 것이다.

`Version += 1`이 각 Apply에 있는 이유는, 이벤트가 하나 추가될 때마다 버전이 하나씩 올라가도록 하기 위해서다.

---

### 13번 줄: 새 이벤트 영구 저장

```csharp
_ticketsRepository.CommitChanges(ticket, originalVersion);
```

지금까지는 모든 것이 메모리 안에서 일어났다. 이벤트도, 상태 변경도 모두 메모리에만 있었다.  
이 줄에서 비로소 새로 생긴 이벤트들을 이벤트 스토어(DB)에 저장한다.

내부적으로는 이렇게 동작한다:
```
1. ticket._domainEvents 에 담긴 새 이벤트들 꺼냄
2. 이벤트 스토어의 Append(instanceId, newEvents, originalVersion) 호출
3. expectedVersion(originalVersion)이 현재 DB 버전과 다르면 예외 발생 (동시성 충돌)
4. 같으면 이벤트 저장 성공 → 버전 +1
```

---

## 6. 전체 흐름을 이야기로 따라가기

```
[고객] "티켓 #42 에스컬레이션 요청합니다!"
          ↓
[TicketAPI] RequestEscalation(id=#42, reason="처리 시간 초과")
          ↓
[이벤트 스토어] 티켓 #42의 이벤트 전부 꺼내옴
  → [TicketCreated, CommentAdded, PriorityChanged]  (버전 3)
          ↓
[Ticket 생성자] 이벤트 3개를 처음부터 재생
  → 빈 TicketState에서 시작
  → Apply(TicketCreated)   → Id=#42, Version=0, IsEscalated=false
  → Apply(CommentAdded)    → Version=1 (댓글은 에스컬레이션 상태 안 바꿈)
  → Apply(PriorityChanged) → Version=2
  → 현재 상태: IsEscalated=false, RemainingTime=0%, Version=3
          ↓
[originalVersion = 3 기억]
          ↓
[Execute(RequestEscalation)]
  → IsEscalated=false ✅ && RemainingTime=0% ✅ → 조건 충족!
  → TicketEscalated 이벤트 생성
  → AppendEvent 호출
    → _domainEvents 리스트에 TicketEscalated 추가
    → _state.Apply(TicketEscalated) → IsEscalated=true, Version=4
          ↓
[CommitChanges(ticket, originalVersion=3)]
  → 이벤트 스토어에 Append([TicketEscalated], expectedVersion=3)
  → DB 확인: 현재 버전 3 == expectedVersion 3 ✅
  → 저장 성공! → DB 버전이 4가 됨
          ↓
[결과] 이벤트 스토어:
  [TicketCreated, CommentAdded, PriorityChanged, TicketEscalated]
```

---

## 7. 성능 문제와 스냅샷 패턴

### 왜 성능 문제가 생기는가?

이벤트 소싱의 근본적인 성능 비용은 **리하이드레이션(프로젝션)** 에 있다.  
어떤 티켓을 수정하려면 항상 과거 이벤트 전부를 불러와서 재생해야 한다.

이벤트가 10개라면 10번 반복. 아무 문제 없다.  
이벤트가 10,000개라면 10,000번 반복. 눈에 띄게 느려진다.

특히 같은 애그리게이트를 자주 수정하는 시스템이라면 이 비용이 빠르게 쌓인다.

### 벤치마킹이 왜 중요한가?

"느릴 것 같다"는 직감만으로 스냅샷을 도입하면 안 된다.  
스냅샷은 코드 복잡도를 상당히 높이는 트레이드오프가 있기 때문이다.

그래서 실제로 측정해서 "이 시스템에서 이벤트가 몇 개일 때부터 문제가 생기는가"를 확인해야 한다.  
이것이 벤치마킹이다. 측정 없이 최적화하지 말라(Don't optimize without measurement)는 개발 원칙이다.

책에서 제시하는 기준은 **애그리게이트당 이벤트 10,000개**다.  
대부분의 시스템에서 하나의 애그리게이트가 10,000개 이상의 이벤트를 갖는 경우는 드물다.  
내 시스템이 이 수준에 도달하지 않는다면 스냅샷 없이도 충분하다.

### 스냅샷 패턴

스냅샷은 특정 시점의 상태를 캐시처럼 저장해두는 것이다.  
게임으로 비유하면 세이브 포인트다. 다음에 게임을 켤 때 처음부터 플레이하지 않고 세이브 포인트부터 시작한다.

```
[스냅샷 없을 때]
이벤트1 → 이벤트2 → ... → 이벤트9000 → 이벤트9001 → ... → 이벤트10000
(매번 10,000번 재생)

[스냅샷 있을 때]
스냅샷(이벤트9000까지 적용된 상태) → 이벤트9001 → ... → 이벤트10000
(1,000번만 재생)
```

스냅샷 생성 시점은 보통 이벤트가 N개 쌓일 때마다 비동기로 생성한다.  
그림 7-2에서 보여주는 흐름은 이렇다:

```
① 앱이 스냅샷 캐시에서 가장 최근 스냅샷을 가져옴
   (스냅샷에는 "마지막으로 적용된 이벤트 ID"가 함께 저장되어 있음)
② 이벤트 스토어에서 그 이벤트 ID 이후의 이벤트만 가져옴
③ 스냅샷 상태에 새 이벤트들만 적용해서 최신 상태 완성
```

스냅샷도 이벤트처럼 **변경하지 않는다**. 새 스냅샷을 만들어서 추가한다.

---

## 8. 데이터 삭제 문제: GDPR vs 이벤트 불변성

### 충돌하는 두 원칙

이벤트 소싱의 핵심 원칙: **이벤트는 삭제하지 않는다**  
GDPR(유럽 개인정보보호법)의 요구: **사용자가 요청하면 개인정보를 삭제해야 한다**

이벤트에 이름, 주소, 주민번호 같은 개인정보가 직접 포함되어 있다면 어떻게 삭제하는가?  
이벤트를 지우면 이벤트 소싱의 근본이 흔들린다.

### 해결책: 잊어버릴 수 있는 페이로드 패턴 (Forgettable Payload Pattern)

핵심 아이디어는 **민감한 정보를 이벤트에 직접 저장하지 않는다**는 것이다.  
대신 이렇게 한다:

```
1. 민감 정보는 암호화해서 별도의 키-값 저장소에 보관한다
   키 저장소: { "user-123-key": "AES_암호화키" }
   
2. 이벤트에는 암호화된 데이터만 저장한다
   TicketCreated {
     userId: "user-123",
     encryptedName: "a3f8b2c1...",  ← 암호화된 이름
     encryptedEmail: "d4e7f6g5..."  ← 암호화된 이메일
   }
   
3. 데이터를 읽을 때는 키 저장소에서 키를 가져와서 복호화한다

4. 삭제 요청이 오면? 키 저장소에서 그 사용자의 키만 삭제한다
   → 이벤트는 그대로 남아있지만, 키가 없으니 복호화 불가 = 사실상 삭제
```

이 방법은 이벤트의 불변성을 유지하면서도 GDPR을 준수할 수 있다.  
법적으로도 "복호화 키가 없어 접근 불가한 데이터"는 삭제된 것으로 인정받는다.

---

## 9. 잘못된 대안들과 그 이유

### "로그 파일에 기록하면 안 되나?"

상태 DB에도 저장하고, 별도 로그 파일에도 기록하는 방식은 **두 저장소 간의 일관성 문제**가 있다.

트랜잭션은 하나의 시스템 안에서만 원자성(atomicity)을 보장한다.  
DB에 저장하는 것과 파일에 쓰는 것은 서로 다른 시스템이라 동시에 성공을 보장할 수 없다.

- DB 저장 성공, 파일 쓰기 실패 → DB에는 데이터 있는데 로그에는 없음
- DB 저장 실패, 파일 쓰기 성공 → 로그에는 있는데 실제 저장은 안 됨

어느 쪽이 진실인가? 알 수 없다. 이것이 **데이터 불일치**다.

### "DB에 이력 테이블 따로 만들면 안 되나?"

이 방법의 문제는 **"왜 바뀌었는가"를 기록할 수 없다**는 것이다.

이력 테이블은 이런 식으로 생겼다:
```
| 시간 | 필드 | 이전값 | 새값 |
|------|------|--------|------|
| 14:30 | IsEscalated | false | true |
```

"IsEscalated가 true로 바뀌었다"는 사실은 있지만,  
"왜 에스컬레이션이 됐는가? 어떤 이유로? 누가 요청했는가?"는 없다.

이벤트 소싱은 이것이 다르다:
```
TicketEscalated {
  reason: "SLA 위반 - 48시간 초과",
  requestedBy: "customer-456",
  escalatedTo: "senior-team"
}
```

단순히 값이 바뀐 사실이 아니라, **비즈니스 맥락이 있는 사건**을 기록한다.  
이 차이가 나중에 분석하고 디버깅할 때 엄청난 차이를 만든다.

---

## 10. 이벤트 소싱을 쓰면 뭐가 좋은가?

### 완전한 감사 추적 (Audit Trail)
모든 변경의 이유와 시점이 기록된다. 금융, 의료, 법률 같은 규제 산업에서 필수다.

### 시간 여행 (Time Travel)
"3일 전 오후 2시에 이 티켓 상태가 어땠지?"  
→ 그 시점 이전의 이벤트만 재생하면 그 순간의 상태를 복원할 수 있다.  
일반 DB에서는 불가능한 일이다.

### 버그 재현
프로덕션에서 이상한 일이 생겼을 때, 이벤트 목록을 그대로 개발 환경에 복사해서 재생하면  
버그 발생 순간을 완벽하게 재현할 수 있다.

### 이벤트 기반 통합
`TicketEscalated` 이벤트가 발생하면 다른 서비스들이 이 이벤트를 구독해서 반응할 수 있다.  
"에스컬레이션 이메일 발송 서비스", "슬랙 알림 서비스" 등이 느슨하게 연결된다.  
이것이 다음 챕터에서 나올 CQRS 패턴과 연결되는 부분이다.

### 읽기 모델의 유연성
이벤트 목록을 다른 방식으로 프로젝션하면 완전히 다른 형태의 읽기 모델을 만들 수 있다.  
같은 이벤트 데이터로 여러 개의 다른 뷰를 생성할 수 있다.

---

## 11. 이벤트 소싱의 단점과 언제 쓰면 안 되는가

이벤트 소싱이 항상 좋은 것은 아니다. 복잡도 비용이 크기 때문에 신중하게 선택해야 한다.

**단점들**:
- 코드 복잡도가 크게 높아진다 (일반 CRUD보다 훨씬 많은 코드가 필요)
- 팀원들이 패턴을 이해해야 한다 (러닝 커브)
- 이벤트 스키마 변경이 까다롭다 (과거 이벤트는 변경 불가이므로 마이그레이션이 복잡)
- 단순 조회 성능이 낮다 (항상 프로젝션을 거쳐야 하므로)

**쓰면 좋은 경우**:
- 감사 추적이 법적/비즈니스적으로 반드시 필요한 경우 (금융, 의료, 법률)
- 복잡한 비즈니스 규칙이 있고 그 판단 과정이 중요한 경우
- 이벤트 기반 아키텍처(마이크로서비스 등)와 함께 사용할 경우

**쓰지 않아도 되는 경우**:
- 단순한 CRUD 작업이 대부분인 시스템
- 이력 추적이 비즈니스적으로 불필요한 경우
- 팀이 작고 빠른 개발 속도가 최우선인 경우

---

## 12. 전체 구조 요약

```
외부 요청 (예: 에스컬레이션 요청)
    ↓
TicketAPI          ← 흐름 조율자. 저장소에서 불러오고, 실행하고, 저장한다
    ↓
이벤트 스토어      ← 과거 이벤트 전부 반환 (append-only DB)
    ↓
new Ticket(events) ← 리하이드레이션: 이벤트 재생 → 현재 상태 복원
    ↓
TicketState        ← 상태값 보관. Apply(event)로만 값 변경 가능
    ↓
ticket.Execute(cmd)← 비즈니스 판단: 조건 충족 시 이벤트 생성
    ↓
AppendEvent(event) ← ① _domainEvents 리스트에 추가
                     ② _state.Apply() 호출 → 즉시 상태 반영
    ↓
CommitChanges()    ← 새 이벤트를 이벤트 스토어에 영구 저장
                     (expectedVersion으로 동시성 충돌 방지)
```

---

*이벤트 소싱은 "과거를 완벽하게 기억하기 위해" 현재를 계산하는 방식으로 바꾼 것이다.  
복잡해 보이지만 모든 복잡함에는 이유가 있고, 그 이유를 이해하면 전체 그림이 보인다. 💪*