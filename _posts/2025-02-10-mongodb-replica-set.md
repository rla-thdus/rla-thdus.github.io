---
title: MongoDB 트랜잭션 사용 시 Replica Set 환경이여야 하는 이유
date: 2025-02-10
categories: [db]
tags: [db, mongodb]
---

MongoDB에서 트랜잭션을 사용하려고 하면 Replica Set 환경이여 한다는 것만 알고 있지, 어떤 이유에서 인지는 알고 넘어가지 못해서 알아보려고 한다.

## Replica Set
- MongoDB에서 고가용성과 데이터 복제를 위해서 사용하는 데이터베이스 클러스터 구조이다.
- 최소 3개 이상의 노드로 구성되며, 하나의 Primary 노드와 여러 개의 Secondary 노드가 데이터를 복제하는 방식으로 동작한다.

### 구성
Replica Set은 최소 3개 이상의 노드로 구성된다.
- **`Primary`**: 모든 쓰기 연산을 담당하며, 클라이언트가 데이터를 저장하는 노드이다.
- **`Secondary`**: Primary 노드의 데이터를 복제 받아 백업 및 읽기 작업 수행하는 노드이다.
- **`Arbiter`** (선택 사항): Primary 선거에만 참여하며, 데이터를 저장하지는 않는 노드이다.

#### PSS
![PSS 구성 사진](https://www.mongodb.com/ko-kr/docs/manual/images/replica-set-primary-with-two-secondaries.bakedsvg.svg)
- 하나의 Primary 노드와 2개의 Secondary 노드로 이뤄진 구성이다.
- Primary 노드에 장애가 발생해도, Secondary 노드가 2개나 대신할 수 있기 때문에 안정성이 높다.

#### PSA
![PSA 구성 사진](https://www.mongodb.com/ko-kr/docs/manual/images/replica-set-primary-with-secondary-and-arbiter.bakedsvg.svg)
- Primary 노드, Secondary 노드, Arbiter 노드 각각 1개로 이뤄진 구성이다.
- Arbiter 노드는 서버의 리소스를 많이 필요하지 않지만 데이터를 실제로 담고있지는 않기 때문에 상대적으로는 구성의 안정성이 낮다.

### 동작 방식
#### 데이터 복제
- Primary 노드에서 변경된 데이터를 [oplog](https://www.mongodb.com/ko-kr/docs/manual/core/replica-set-oplog/)에 기록한다.
- Secondary 노드에서는 oplog를 읽어와서 데이터를 동기화 한다.

#### Primary 노드 장애 시
- Secondary 노드와 Aribter 노드끼리 투표를 통해 새로운 Primary를 자동으로 선출한다.
  - 선출 기준은
    - 우선 순위가 높은 노드가 선호된다.
    - 가장 최근 oplog를 가진(Optime) 노드만 Primary가 될 수 있다.
    - 과반수의 노드와 연결이 가능해야 한다.
- 클라이언트는 새로운 Primary에 자동으로 연결된다.

#### 읽기 및 쓰기 연산 처리
- 쓰기 연산
  - Primary 노드에서만 가능하며, `writeConcern` 설정을 통해 특정 개수의 Secondary 노드에 복제될 때까지 기다릴 수 있다.
- 읽기 연산
  - 기본적으로 Primary 노드에서 수행하지만, `readPreference` 설정을 통해 Secondary 노드에서 읽기 수행 가능하도록 할 수 있다. (읽기 부하 분산)

### Replica Set을 왜 써야 할까?
#### 1. 고가용성 보장
- Primary 노드에 장애 발생한 경우 새로운 Primary를 선출해 서비스를 지속 가능하도록 한다.
- 즉 Standalone 환경에서는 서버 다운 시, 서비스가 불가능 하지만 Replica Set 환경에서는 자동 복구가 가능하다.

#### 2. 데이터 일관성 보장
- 트랜잭션 수행 시 Primary 장애가 발생하면, 새로운 Primary가 승격되어 미완료 트랜잭션을 롤백 할 수 있다.
- oplog를 기반으로 모든 노드가 동일한 데이터를 유지한다.

#### 3. 트랜잭션 및 다중 문서 원자성 보장
- Replica Set 환경에서 다중 문서 트랜잭션이 가능해, 데이터 일관성 보장이 가능하다.

#### 4. 데이터 백업 및 복구 용이
- Primary 노드에 직접 영향을 주지 않고, Secondary 노드를 활용해 백업 및 데이터 복구를 진행할 수 있다.

#### 5. 읽기 성능 향상
- `readPreference` 옵션을 통해 Secondary 노드에서 읽기를 수행하도록 하면, Primary 부하를 줄일 수 있다.

## 그래서 Replica Set 환경에서만 다중 문서 트랜잭션이 가능한 이유는?
트랜잭션은 기본적으로 데이터베이스에서 여러 개의 작업을 하나의 단위로 묶어, 모두 성공하거나 모두 실패하도록 보장한다.
Replica Set 환경에서는 장애 시 oplog를 통해 트랜잭션을 복구 및 롤백이 가능하지만, Standalone 환경에서는 복제본이 없어 데이터 불일치 등이 발생할 가능성이 높기 때문이다.
