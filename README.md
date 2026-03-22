# 📚 도메인 주도 설계 첫걸음 — 이해 안 되는 내용 정리

> **Learning Domain-Driven Design** (Vlad Khononov 저)을 읽으며  
> 헷갈렸던 개념, 직접 정리한 설명을 모아두는 스터디 저장소입니다.

---

## 🗂️ 책 정보

| 항목 | 내용 |
|------|------|
| 제목 | 도메인 주도 설계 첫걸음 (Learning Domain-Driven Design) |
| 저자 | Vlad Khononov |
| 출판사 | O'Reilly / 한빛미디어 |
| 분류 | 소프트웨어 아키텍처, 설계 패턴 |

---

## 🎯 학습 목표

- DDD의 핵심 개념(도메인, 바운디드 컨텍스트, 애그리게이트 등)을 **내 말로 설명**할 수 있게 되기
- 읽다가 막히는 부분을 그냥 넘기지 않고 **질문 → 이해 → 정리** 사이클 만들기
- 실무에서 DDD를 어떻게 적용할 수 있을지 감 잡기

---

## 📝 정리 방식

- 각 챕터마다 폴더 생성 (`ch01/`, `ch02/`, ...)
- 파일 구성:
  - `README.md` — 챕터 요약 및 핵심 개념
  - `개념명.md` — 이해 안 된 부분, 스스로 던진 질문

---

## 📖 목차

### Part 1. 전략적 설계

| 챕터 | 제목 | 상태 |
|------|------|------|
| Ch01 | 비즈니스 도메인 분석하기 | ✅ |
| Ch02 | 도메인 지식 찾아내기 | ✅ |
| Ch03 | 도메인의 복잡성 다루기 | ✅ |
| Ch04 | 바운디드 컨텍스트 통합하기 | ✅ |

### Part 2. 전술적 설계

| 챕터 | 제목 | 상태 |
|------|------|------|
| Ch05 | 간단한 비즈니스 로직 구현하기 | ✅ |
| Ch06 | 복잡한 비즈니스 로직 다루기 | ✅ |
| Ch07 | 시간 차원 모델링하기 | ✅ |
| Ch08 | 아키텍처 패턴 | ✅ |

### Part 3. DDD 적용하기

| 챕터 | 제목 | 상태 |
|------|------|------|
| Ch09 | 커뮤니케이션 패턴 | ⬜ |
| Ch10 | 설계 휴리스틱 | ⬜ |
| Ch11 | 진화하는 설계 결정 | ⬜ |
| Ch12 | 이벤트 스토밍 | ⬜ |
| Ch13 | DDD의 실제 적용 | ⬜ |

> ⬜ 미시작 &nbsp;|&nbsp; 🔄 진행 중 &nbsp;|&nbsp; ✅ 완료

---

## 💡 주요 개념 빠른 참조

- [V] [낙관적 동시성 제어 (Optimistic Concurrency Control)](https://github.com/Yum-CS-Study-Memo/Learning-Domain-Driven-Design/blob/main/part2/ch05/%EB%82%99%EA%B4%80%EC%A0%81%20%EB%8F%99%EC%8B%9C%EC%84%B1%20%EC%A0%9C%EC%96%B4%20(Optimistic%20Concurrency%20Control).md)
- [V] [트랜잭션 스크립트 & 도메인 모델](https://github.com/Yum-CS-Study-Memo/Learning-Domain-Driven-Design/blob/main/part2/ch05/%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%98%20%EC%8A%A4%ED%81%AC%EB%A6%BD%ED%8A%B8%20vs%20%EB%8F%84%EB%A9%94%EC%9D%B8%20%EB%AA%A8%EB%8D%B8.md)
---

## 🛠️ 사용 언어 / 환경

- 예제 코드: `C#` (책 기준), 필요 시 `Python`으로 재작성
- 다이어그램: Mermaid 또는 이미지 첨부
)
---
