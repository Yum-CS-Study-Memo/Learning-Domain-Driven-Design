# 트랜잭션 스크립트 vs 도메인 모델

---

## 1. 도입 배경

### 소프트웨어가 복잡해진 역사

초기 소프트웨어는 단순했다.

```
1970~80년대
"사용자 정보를 DB에 저장해줘"
"주문 내역을 조회해줘"
→ 그냥 절차대로 위에서 아래로 짜면 됐음
```

그런데 비즈니스가 커지면서 요구사항이 복잡해졌다.

```
2000년대 이후 e-commerce 예시
"주문할 때:
 - 재고 확인하고
 - 쿠폰 적용하고
 - 포인트 계산하고
 - 결제 수단별 할인 적용하고
 - 배송비 계산하고
 - 재고 차감하고
 - 알림 보내고
 - 정산 처리하고..."
```

이 복잡성을 어떻게 코드로 표현할 것인가? 라는 문제가 생겼고, 두 가지 접근법이 등장했다.

- **트랜잭션 스크립트** — Martin Fowler가 *Patterns of Enterprise Application Architecture* (2002)에서 정리. "절차적으로 짜되, 잘 짜자"
- **도메인 모델** — Eric Evans가 *Domain-Driven Design* (2003)에서 정리. "현실 세계를 코드로 그대로 표현하자"

---

## 2. 왜 필요한가?

### 트랜잭션 스크립트가 필요한 이유

비즈니스 로직이 단순할 때, 굳이 복잡한 구조를 만들 필요가 없다.

```
"방문 횟수를 기록해줘"
→ DB에서 읽고, +1 하고, 저장. 끝.
→ 이걸 위해 복잡한 객체 구조를 만드는 건 오히려 낭비
```

**과도한 설계(Over-engineering)를 방지**하고, 빠르게 구현할 수 있다.

### 도메인 모델이 필요한 이유

비즈니스 로직이 복잡해지면 트랜잭션 스크립트로는 한계가 온다.

```python
# 트랜잭션 스크립트로 복잡한 주문 처리를 짜면?
class OrderService:
    def place_order(self, user_id, items):
        user = db.get_user(user_id)
        if user.grade == "VIP":
            discount = 0.2
        elif user.grade == "GOLD":
            discount = 0.1
        else:
            discount = 0

        total = 0
        for item in items:
            stock = db.get_stock(item.id)
            if stock < item.quantity:
                raise Exception("재고 부족")
            total += item.price * item.quantity

        total = total * (1 - discount)

        if user.has_coupon:
            coupon = db.get_coupon(user.coupon_id)
            if coupon.min_amount <= total:
                total -= coupon.discount_amount

        # ... 계속 이어짐
        # 코드가 수백 줄이 됨 😱
```

이렇게 되면 코드가 **하나의 메서드에 몰려서** 읽기도 어렵고, 수정하면 어디서 버그가 날지 모른다.

도메인 모델을 쓰면 각 개념이 **자기 책임**을 가지므로 복잡성이 분산된다.

---

## 3. 무엇인가?

### 트랜잭션 스크립트

> **"Service가 모든 절차를 지시하는 방식"**

데이터(User, Order 등)는 그냥 **정보를 담는 그릇**이고,
로직은 전부 **Service 메서드 안**에 순서대로 작성된다.

```python
class User:
    # 데이터만 있음. 아무것도 못 함
    def __init__(self):
        self.visits = 0
        self.name = ""

class UserService:
    def log_visit(self, user_id):
        user = db.get(user_id)   # 1. 꺼내고
        user.visits += 1          # 2. 계산하고
        db.save(user)             # 3. 저장
        # Service가 1,2,3 전부 지시
```

**핵심:** User는 수동적. Service가 능동적.

---

### 도메인 모델

> **"객체가 스스로 자기 일을 처리하는 방식"**

데이터와 로직이 **같은 객체 안**에 있다.
현실 세계의 개념("유저는 방문할 수 있다")을 코드로 그대로 표현한다.

```python
class User:
    def __init__(self):
        self.visits = 0
        self.name = ""

    # 로직이 User 안에 있음
    def log_visit(self):
        self.visits += 1  # 내가 알아서 처리

    def apply_discount(self, price):
        if self.grade == "VIP":
            return price * 0.8
        return price * 0.95

class UserService:
    def log_visit(self, user_id):
        user = db.get(user_id)
        user.log_visit()   # User한테 시키기만 함. 어떻게 하는지는 User가 앎
        db.save(user)
```

**핵심:** User가 능동적. Service는 조율만 함.

---

### 한눈에 비교

| | 트랜잭션 스크립트 | 도메인 모델 |
|---|---|---|
| 데이터 객체 | 데이터만 있음 | 데이터 + 로직 같이 있음 |
| 로직 위치 | Service 메서드 안 | 각 객체 안 |
| 객체의 역할 | 수동적 (시키는 대로) | 능동적 (스스로 처리) |
| 코드 스타일 | "1번 하고, 2번 하고, 3번 해" | "User야, 방문 처리해" |

---

## 4. 언제 쓰이는가?

### 트랜잭션 스크립트를 쓸 때

비즈니스 로직이 **단순하고 직선적**일 때.

```
✅ 이런 경우에 적합
- 방문 횟수 기록
- 로그 저장
- 단순 CRUD (생성/조회/수정/삭제)
- 외부 API 호출 후 결과 저장
- 배치 작업 (데이터 일괄 처리)
```

```python
# 딱 봐도 단순한 로직 → 트랜잭션 스크립트로 충분
class LogService:
    def log_visit(self, user_id, visits):
        db.execute(
            "UPDATE Users SET visits = @p1 WHERE user_id = @p2",
            visits, user_id
        )
```

### 도메인 모델을 쓸 때

비즈니스 규칙이 **복잡하게 얽혀있을 때**.

```
✅ 이런 경우에 적합
- 주문/결제처럼 여러 규칙이 연관될 때
- 상태 변화가 복잡할 때 (주문 → 결제 → 배송 → 완료)
- 같은 도메인 규칙이 여러 곳에서 반복될 때
- 비즈니스 규칙이 자주 바뀔 때
```

```python
# 규칙이 복잡 → 도메인 모델로 각자 책임 분리
class Order:
    def place(self):
        self._validate_items()      # 주문 검증은 Order가 앎
        self._apply_discount()      # 할인 계산도 Order가 앎
        self.status = "PLACED"

class User:
    def apply_discount(self, price):
        return price * (1 - self._discount_rate())  # 할인율은 User가 앎
```

---

### 판단 기준 요약

```
비즈니스 로직이 단순해?
→ YES: 트랜잭션 스크립트
→ NO: 도메인 모델

로직이 여러 곳에서 중복되고 있어?
→ YES: 도메인 모델로 객체 안에 모으기
→ NO: 트랜잭션 스크립트로도 충분

나중에 규칙이 자주 바뀔 것 같아?
→ YES: 도메인 모델 (변경이 한 곳에만 영향)
→ NO: 트랜잭션 스크립트
```

> 처음부터 도메인 모델을 쓸 필요는 없다.
> 트랜잭션 스크립트로 시작하다가 복잡해지면 도메인 모델로 전환하는 것도 좋은 전략이다.