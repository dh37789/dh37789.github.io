---
title:  "[Java] Database Lock을 이용해 동시성 제어하기"

categories: Java

toc: true
toc_sticky: true

date: 2023-03-23
last_modified_at: 2022-03-23
---

# Database Lock을 이용해 동시성 제어하기

## 동시성 (Race Condition)

Race Condition이란 두개 이상의 동시적인(Concurrent) 프로세스나 스레드들이 하나의 리소스에 접근하기 위해 경쟁하는 상태를 말한다.

이러한 상태가 발생했을때 자료의 일관성을 해치는 결과가 나타날 수 있다.

예를 들면 1개 남은 재고의 N개 이상의 프로세스(스레드)가 접근한다면? 겨우 1개의 재고가 남은 상품이 2번 최악의 경우 N번의 주문이 완료되는 상황이 생길 수 있다.

이러한 경우를 방지하려면 어떻게 해야할까?

여러가지 방법이 있겠지만, 여기선 Lock을 이용하여 Race Condition을 제어해보고자 한다.

Application에는 이러한 상태를 제어하기 위한 Lock을 지원한다.

그중 **낙관적 락 (Optimistic Lock)**과 **비관적 락 (Pessimistic Lock)**에 대해 알아보도록 하자.


## Lock

낙관적 락과 비관적 락에 대해 알아보기 전에 Database의 Lock에대해 간략하게 짚고 넘어가고자 한다.

### 공유 락 (Shared Lock)

공유 락은 **데이터를 변경하지 않는 읽기 명령에 대해 주어지는 락**으로, `Read Lock` 또는 `S Lock` 으로 표기하기도 한다.  
읽기의 경우 동시에 데이터에 접근해도 데이터의 일관성에 영향을 주지 않기 때문에, 공유락끼리는 동시에 접근이 가능하다.

### 베타 락 (Exclusive Lock)

베타 락은 **데이터에 변경을 가하는 쓰기 명령에 대해 주어지는 락**으로 `Write Lock` 또는 `X Lock` 으로 표기하기도 한다.  
Lock이 걸릴 경우 Lock이 해제될 때까지 SELECT를 포함한 모든 트랜잭션은 해당 데이터에 접근 할 수 없다.


낙관적 락과 비관적 락중 비관적 락은 이 두가지를 이용해 Application에서 Lock을 거는 형태로 지원된다.


## 낙관적 락

Optimistic은 사전적 의미로는 '낙관적인' 으로 트랜잭션이 서로 충돌하지 않을 것이라고 긍적적으로 가정하는 제어법을 의미한다.

낙관적 락은 위에서 설명한 DB에서 제공하는 락의 기능을 사용하지는 않고, JPA가 제공하는 버전 관리 기능을 사용한다. 그러므로 Entity에 version이라는 컬럼이 추가되어야 한다.

또한 낙관적 락은 트랜잭션을 커밋하기 전까지는 트랜잭션의 충돌을 알 수가 없다. 그러므로, 충돌 후 트랜잭션 롤백 및 재시작하는 로직은 개발자가 직접 구현을 해주어야 한다.


## 비관적 락

Pessimistic은 사전적 의미로는 '비관적인' 으로 트랜잭션이 충돌한다고 부정적으로 가정하여, Lock을 우선 걸고 실행하는 제어법을 의미한다.

DB에서 제공하는 락의 기능을 사용하며, 트랜잭션이 시작할 때 Shared Lock 또는 Exclusive Lock 을 걸고 시작한다. 그래서 쿼리를 살펴보면 SELECT ... FOR UPDATE 구문이 붙어서 요청되는걸 확인 할 수 있다.

정보를 조회 또는 수정단계에서 Lock을 걸기 때문에, 수정단계에서 충돌유무를 확인할 수 있다. 하지만 Lock에 의해 교착상태(DeadLock)에 빠질 위험이 있다.


출처 : [Optimistic vs. Pessimistic locking - Stack Overflow](https://stackoverflow.com/questions/129329/optimistic-vs-pessimistic-locking)