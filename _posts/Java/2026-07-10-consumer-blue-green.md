---
title:  "[Java] Kafka Consumer 서비스에 Blue-Green 배포 적용기"

layout: post
categories: Java

toc: true
toc_sticky: true

date: 2026-07-10
last_modified_at: 2026-07-10
---


주문 이벤트를 처리하는 Kafka Consumer 서비스의 배포 방식을 Rolling 배포에서 Blue-Green 배포로 전환했다.

일반적인 Web API 서버라면 Blue-Green 배포는 비교적 단순하다. 새 버전을 띄우고 health check를 통과하면 트래픽을 전환하면 된다.


## Kafka Consumer와 WEB API 서비스의 차이

Kafka Consumer 서비스는 조금 다르다.

Consumer는 HTTP 트래픽을 받는 서버가 아니라 Kafka topic의 메시지를 poll하고 처리한다. 따라서 Blue와 Green 인스턴스가 같은 consumer group에 동시에 참여하면 rebalance가 발생하고, 전환 타이밍에 따라 메시지 처리 공백이나 중복 처리 위험이 생길 수 있다.

이 글에서는 Kafka Consumer 서비스에 Blue-Green 배포를 적용하면서 마주친 문제와 해결 과정을 정리한다.

특히 다음 질문을 중심으로 다룬다.

1. Kafka Consumer 서비스에서 Blue-Green 배포가 Web API 서버와 다른 이유는 무엇인가?
2. Green 인스턴스를 먼저 띄우면서도 consumer group에는 참여시키지 않으려면 어떻게 해야 하는가?
3. 기존 Blue Consumer를 어떻게 안전하게 멈추고 Green Consumer로 전환할 수 있는가?
4. 배포 실패 시 이전 슬롯으로 빠르게 되돌리려면 어떤 흐름이 필요한가?

---

## 기존 환경

배포 대상 서비스는 주문 이벤트를 처리하는 Spring Boot 기반 Kafka Consumer 서비스였다.

| 항목 | 내용 |
|---|---|
| Framework | Spring Boot 2.1.x |
| Java | JDK 8 |
| Kafka | Spring Kafka |
| CI/CD | GitLab CI/CD, Shell Executor |
| 서버 | 운영 서버 2대 |
| 트래픽 제어 | HAProxy |
| 프로세스 관리 | systemd |

기존에는 Rolling 방식으로 배포했다.

```text
1. replica1 배포
replica1 → HAProxy disable → restart → health check → HAProxy enable

2. replica2 배포
replica2 → HAProxy disable → restart → health check → HAProxy enable
```

Rolling 배포는 단순했지만 몇 가지 문제가 있었다.

- 배포 중 버전이 섞인다.
- 문제가 생겼을 때 즉시 롤백하기 어렵다.
- 롤백하려면 이전 버전으로 다시 배포해야 한다.
- health check가 로그 파일 tail 기반이라 불안정했다.
- Kafka Consumer 특성상 배포 중 rebalance와 처리 공백을 명확히 통제하기 어려웠다.

---

## 왜 Blue-Green이 필요했나

운영 배포에서 원했던 것은 세 가지였다.

1. 새 버전을 미리 띄워 검증한 뒤 전환할 것
2. 문제가 생기면 이전 버전으로 즉시 되돌릴 수 있을 것
3. Kafka Consumer 전환 시 메시지 처리 공백과 중복 처리 위험을 줄일 것

그래서 환경별로 배포 전략을 분리했다.

| 환경 | 전략 | 이유 |
|---|---|---|
| develop | Blue-Green | 운영 전 배포 흐름 검증 |
| staging | Blue-Green | 운영과 동일한 전환 흐름 검증 |
| production | Blue-Green | 무중단 전환과 즉시 롤백 |
| feature/branch | Rolling | 임시 인스턴스, 무중단 요구 낮음 |
| canary | Rolling | 단일 서버 검증 목적 |

모든 환경을 무조건 Blue-Green으로 바꾸지는 않았다. 운영과 유사한 환경에만 Blue-Green을 적용하고, 임시성 환경은 Rolling으로 유지했다.

---

## 포트와 슬롯 설계

같은 서버에서 Blue와 Green 두 인스턴스를 동시에 띄워야 하므로 포트와 systemd unit을 분리했다.

| 슬롯 | 포트 | systemd unit 예시 |
|---|---:|---|
| Blue | 9998 | consumer-blue |
| Green | 9997 | consumer-green |

중요한 점은 Active Slot을 별도 상태 파일로 관리하지 않았다는 것이다.

처음에는 `active-slot` 같은 파일을 두고 현재 활성 슬롯을 기록하는 방식을 고려했다. 하지만 파일과 실제 프로세스 상태가 불일치할 수 있었다.

예를 들어 배포 중간에 실패하거나 서버가 재부팅되면 상태 파일은 Green을 가리키는데 실제로는 Blue만 살아 있을 수 있다.

그래서 최종적으로 실제 systemd 상태를 기준으로 판단했다.

```bash
get_active_slot() {
  local server_ip=$1

  local blue_active
  local green_active

  blue_active=$(ssh devops@${server_ip} \
    "sudo systemctl is-active ${BLUE_SERVICE_UNIT} 2>/dev/null || true")

  green_active=$(ssh devops@${server_ip} \
    "sudo systemctl is-active ${GREEN_SERVICE_UNIT} 2>/dev/null || true")

  if [[ "$blue_active" == "active" && "$green_active" == "active" ]]; then
    echo "ERROR: blue와 green이 모두 active 상태입니다." >&2
    return 1
  elif [[ "$blue_active" == "active" ]]; then
    echo "blue"
  elif [[ "$green_active" == "active" ]]; then
    echo "green"
  else
    echo "blue"
  fi
}
```

이 방식의 장점은 명확했다.

- 별도 상태 파일을 관리하지 않아도 된다.
- 실제 프로세스 상태와 일치한다.
- 서버 재부팅 후에도 현재 상태를 다시 판단할 수 있다.

---

## Health Check 개선

기존 health check는 로그 파일을 감시하는 방식이었다.

```bash
tail -fn0 ${LOG_FILE} | while read -t 60 line; do
  [[ "$line" == *"JVM running for"* ]] && exit 0
  [[ "$line" == *" ERROR "* ]] && exit 1
done
```

이 방식은 다음 문제가 있었다.

- 로그 파일 생성 타이밍에 의존한다.
- 로그 포맷이 바뀌면 health check가 깨진다.
- 애플리케이션이 실제로 HTTP 요청을 받을 수 있는지 확인하지 못한다.

그래서 `actuator` 기반으로 변경했다.

```bash
health_check() {
  local server=$1
  local port=$2
  local server_ip=${HOSTS_MAP[$server]}
  local url="http://localhost:${port}/actuator/health"

  local i=0
  while [[ ${i} -lt 40 ]]; do
    status=$(ssh devops@${server_ip} \
      "sudo curl -s -o /dev/null -w '%{http_code}' ${url}" \
      2>/dev/null || echo "000")

    [[ "${status}" == "200" ]] && return 0

    i=$((i + 1))
    sleep 3
  done

  return 1
}
```

helth check를 확인하는 방법은 `/actuator/health/readiness`를 사용하는편이 더 좋았겠지만   Spring Boot 2.1.x에서는 사용할 수 없었다. readiness endpoint는 Spring Boot 2.3 이후의 기능이기 때문이다.

따라서 현재 버전에서는 `/actuator/health`를 사용했다. 서비스의 health indicator에 DB, Kafka 연결 상태가 포함되어 있어 기동 확인 용도로 충분하다고 판단했다.

---

## Warm-up 처리

`/actuator/health`가 200을 반환해도 첫 비즈니스 요청에서 지연이 발생할 수 있었다.

Spring MVC 환경에서 `DispatcherServlet`이 첫 요청 시점에 초기화되면서 수백 ms 정도의 지연이 발생할 수 있기 때문이다.

이를 줄이기 위해 health check 통과 후 `/actuator/info`를 한 번 호출해 HTTP 요청 경로를 미리 태웠다.

```bash
warm_up() {
  local server=$1
  local port=$2
  local server_ip=${HOSTS_MAP[$server]}

  status=$(ssh devops@${server_ip} \
    "sudo curl -s -o /dev/null -w '%{http_code}' http://localhost:${port}/actuator/info")

  echo "warm-up 완료: ${server}:${port} (HTTP ${status})"
}
```

warm-up은 실패해도 배포를 중단하지 않았다. 목적은 배포 성공 여부 판단이 아니라 초기 요청 지연을 줄이는 것이었기 때문이다.

---

## Kafka Consumer 전환

Blue-Green 배포에서 까다로운 부분은 Kafka Consumer 전환이었다.

Rest API 서버라면 새 인스턴스를 띄운 뒤 HAProxy만 전환하면 된다. 하지만 Consumer 서비스는 같은 consumer group에 참여해 메시지를 가져간다.

만약 Blue와 Green Consumer가 동시에 같은 consumer group에 참여하면 다음 문제가 발생할 수 있다.

1. rebalance가 발생한다.
2. 파티션 할당이 흔들린다.
3. 처리 중인 메시지와 신규 메시지의 경계가 애매해진다.
4. 전환 타이밍에 따라 중복 처리나 처리 공백이 발생할 수 있다.

따라서 Green 애플리케이션은 먼저 띄우되, Kafka Consumer는 바로 시작하지 않도록 했다.

---

## Green 인스턴스는 기동하되 Consumer는 지연 시작

systemd unit에 환경 변수를 추가했다.

```ini
Environment="SPRING_KAFKA_LISTENER_AUTO_STARTUP=false"
```

`application.yml`에서는 이 값을 Spring Kafka listener 설정에 연결했다.

```yaml
spring:
  kafka:
    listener:
      auto-startup: ${SPRING_KAFKA_LISTENER_AUTO_STARTUP:true}
```

이렇게 하면 Green 인스턴스는 health check까지 통과할 수 있지만, Kafka listener는 자동으로 시작하지 않는다.

즉, Green은 준비 상태가 되지만 아직 consumer group에는 참여하지 않는다.

```text
Green process: running
Green HTTP endpoint: ready
Green Kafka consumer: stopped
```

---

## Consumer 전환 전략: pause → polling → shutdown → start

처음에는 Blue를 그냥 `stop()`하면 된다고 생각했다. 하지만 단순 `stop()`은 처리 중인 메시지가 끝나기 전에 shutdown timeout에 걸릴 수 있다.

고정 시간 `sleep 5` 같은 방식도 안전하지 않았다.

처리 중인 메시지가 1초 만에 끝날 수도 있고, 30초 이상 걸릴 수도 있기 때문이다.

최종적으로는 다음 순서를 사용했다.

```text
1. Blue consumer pause
   - 새 메시지 poll 중단
   - 이미 처리 중인 메시지는 완료되도록 둠

2. paused 상태 polling
   - 실제로 pause 요청이 반영되었는지 확인
   - 최대 60초, 3초 간격

3. Blue consumer shutdown
   - consumer group에서 탈퇴

4. Green consumer start
   - 새 버전 consumer가 메시지 처리 시작
```

고정 sleep이 아니라 실제 상태를 확인하므로 처리 시간이 긴 메시지가 있어도 더 안전하게 전환할 수 있었다.

---

## Kafka Consumer 제어용 Custom Actuator Endpoint

Spring Boot 2.1.x에서는 `KafkaListenerEndpointRegistry`를 제어할 수 있는 표준 actuator endpoint가 없었다.

그래서 직접 Actuator endpoint를 만들었다.

```java
@Component
@Endpoint(id = "kafkaConsumer")
public class KafkaConsumerEndpoint {

    private final KafkaListenerEndpointRegistry registry;

    public KafkaConsumerEndpoint(KafkaListenerEndpointRegistry registry) {
        this.registry = registry;
    }

    @WriteOperation
    public String control(String action) {
        if ("start".equals(action)) {
            registry.start();
            registry.getListenerContainers()
                    .forEach(MessageListenerContainer::resume);
            return "started";
        }

        if ("pause".equals(action)) {
            registry.getListenerContainers()
                    .forEach(MessageListenerContainer::pause);
            return "pausing";
        }

        if ("shutdown".equals(action)) {
            registry.stop();
            return "stopped";
        }

        return "unknown action";
    }

    @ReadOperation
    public Map<String, Object> status() {
        boolean running = registry.isRunning();
        boolean allPauseRequested = registry.getListenerContainers().stream()
                .allMatch(MessageListenerContainer::isPauseRequested);

        return Map.of(
                "running", running,
                "pauseRequested", allPauseRequested
        );
    }
}
```

기존 명령 이름은 `stop`이 pause를 의미하고, `shutdown`이 실제 stop을 의미했다. 하지만 공개용 코드에서는 혼동을 줄이기 위해 `pause`와 `shutdown`으로 역할을 분리하는 것이 더 명확하다.

배포 스크립트에서는 다음 순서로 호출한다.

```bash
# 1. Blue consumer pause
curl -X POST \
  -H 'Content-Type: application/json' \
  -d '{"action":"pause"}' \
  'http://localhost:9998/actuator/kafkaConsumer'

# 2. pause 요청 반영 확인
for i in $(seq 1 20); do
  status=$(curl -s 'http://localhost:9998/actuator/kafkaConsumer')
  echo "$status" | grep -q '"pauseRequested":true' && break
  sleep 3
done

# 3. Blue consumer shutdown
curl -X POST \
  -H 'Content-Type: application/json' \
  -d '{"action":"shutdown"}' \
  'http://localhost:9998/actuator/kafkaConsumer'

# 4. Green consumer start
curl -X POST \
  -H 'Content-Type: application/json' \
  -d '{"action":"start"}' \
  'http://localhost:9997/actuator/kafkaConsumer'
```

---

## CI/CD 구조 정리

Blue-Green 배포를 적용하면서 CI/CD 파일도 함께 정리했다.

기존에는 단일 `.gitlab-ci.yml` 파일이 점점 커지고 있었다. 환경별 조건, 배포 스크립트, notification, cleanup 설정이 한 파일에 섞여 있어 변경하기 어려웠다.

그래서 stage별로 파일을 분리했다.

```text
.gitlab-ci.yml
.gitlab/ci/
├── build.yml
├── develop.yml
├── staging.yml
├── canary.yml
├── production.yml
├── cleanup.yml
└── notification.yml

scripts/deploy/
├── deploy.sh
├── config/
│   ├── develop.env
│   ├── staging.env
│   └── production.env
├── lib/
│   ├── common.sh
│   └── health.sh
└── strategies/
    ├── blue-green.sh
    └── rolling.sh
```

관리 원칙은 다음처럼 나눴다.

| 영역 | 담당 |
|---|---|
| 언제 실행할지 | GitLab CI rules |
| 어떤 runner에서 실행할지 | GitLab CI tags |
| 어떤 환경에 배포할지 | deploy config |
| 어떤 전략으로 배포할지 | deploy strategy script |
| 공통 로깅/health check | deploy lib |

이렇게 분리하니 CI 파일은 파이프라인 제어에 집중하고, 배포 스크립트는 실제 배포 로직에 집중할 수 있었다.

---

## `only/except`에서 `rules`로 전환

기존에는 `only/except` 조합으로 브랜치 조건을 관리했다.

```yaml
only:
  refs:
    - branches
except:
  refs:
    - master
    - production
```

이 방식은 조건이 늘어날수록 의도를 파악하기 어려웠다. 그래서 `rules`로 전환했다.

```yaml
rules:
  - if: '$CI_COMMIT_BRANCH == "master" || $CI_COMMIT_BRANCH == "production"'
    when: never
  - if: '$CI_COMMIT_BRANCH'
    when: manual
```

`rules`는 위에서 아래로 순차 평가되기 때문에 제외 조건과 실행 조건을 명확히 표현할 수 있었다.

---

## 배포 설정을 config로 분리

환경별 포트, 서버, systemd unit, HAProxy backend는 CI 파일이 아니라 config 파일로 분리했다.

```bash
# scripts/deploy/config/production.env
DEPLOY_STRATEGY=blue-green
DEPLOY_HOSTS="prod-server1 prod-server2"

BLUE_PORT=9998
GREEN_PORT=9997

BLUE_SERVICE_UNIT=consumer-blue
GREEN_SERVICE_UNIT=consumer-green

HAPROXY_BACKEND="consumer-backend"
```

CI job은 환경명만 넘긴다.

```yaml
production:
  script:
    - sh scripts/deploy/deploy.sh production
```

이렇게 하면 배포 대상 설정을 바꾸기 위해 `.gitlab-ci.yml`을 건드리지 않아도 된다.

---

## 운영 서버 사전 준비에서 만난 문제들

### 1. executable jar의 `.conf` 포트 설정

Spring Boot executable jar는 같은 이름의 `.conf` 파일을 자동으로 읽는다. 하지만 다음처럼 단독 환경 변수를 넣는 것만으로는 `server.port`가 적용되지 않았다.

```bash
# 동작하지 않음
SERVER_PORT=9997
```

JVM 시스템 프로퍼티로 전달해야 했다.

```bash
JAVA_OPTS="... -Dserver.port=9997"
```

### 2. Spring Cloud Config 우선순위

Config Server에서 `server.port`를 내려주면 로컬 `-Dserver.port`가 기대대로 동작하지 않았다.

해당 서비스에서는 Config Server의 `server.port` 설정을 제거해 해결했다.

설정 우선순위는 프로젝트 구성에 따라 달라질 수 있으므로, 외부 설정을 사용하는 서비스는 어떤 설정이 최종 적용되는지 반드시 확인해야 한다.

### 3. 포트 충돌 사전 체크

기존 슬롯을 내리지 않은 상태에서 target 슬롯을 띄우기 때문에 포트 충돌 가능성이 있었다.

배포 스크립트에서 target 포트가 다른 프로세스에 의해 점유되어 있는지 확인했다.

```bash
check_port_available() {
  local server=$1
  local port=$2
  local target_unit=${3:-}
  local server_ip=${HOSTS_MAP[$server]}

  if [[ -n "$target_unit" ]]; then
    unit_status=$(ssh devops@${server_ip} \
      "sudo systemctl is-active ${target_unit} 2>/dev/null || true")

    if [[ "$unit_status" == "active" ]]; then
      return 0
    fi
  fi

  result=$(ssh devops@${server_ip} "sudo ss -tlnp | grep ':${port} ' || true")

  if [[ -n "$result" ]]; then
    return 1
  fi

  return 0
}
```

판단 기준은 다음과 같다.

| 상황 | 판단 | 결과 |
|---|---|---|
| 포트 미사용 | 사용 가능 | 배포 진행 |
| 같은 systemd unit이 점유 | restart로 처리 가능 | 배포 진행 |
| 다른 프로세스가 점유 | 충돌 | 배포 중단 |

---

## 최종 배포 흐름

최종 배포 흐름은 네 단계로 정리된다.

```text
Phase 1. Target 슬롯 배포 및 기동
    - jar 전송
    - 포트 체크
    - Green systemd unit 시작
    - Kafka consumer는 auto-startup=false로 비활성 상태

Phase 2. Target 슬롯 검증
    - /actuator/health 확인
    - /actuator/info warm-up

Phase 3. Kafka Consumer 전환
    - Blue consumer pause
    - pause 상태 polling
    - Blue consumer shutdown
    - Green consumer start

Phase 4. 정리
    - HAProxy 전환
    - 이전 슬롯 중지
```

기존에는 서버별로 `기동 → health check`를 끝낸 뒤 다음 서버로 넘어갔다. 개선 후에는 모든 서버의 target 슬롯을 먼저 기동하고, health check를 순차 확인했다.

이 방식으로 전체 배포 시간이 줄었다.

```text
Before: 서버별 순차 기동 + 순차 대기
After : 전체 서버 기동 후 순차 health check
```

Phase 1~2 동안 기존 Blue consumer가 계속 메시지를 처리하므로, 실제 메시지 처리 공백은 Phase 3의 consumer 전환 구간으로 제한된다.

---

## 롤백 전략

Blue-Green의 장점은 이전 버전이 이미 서버에 남아 있다는 점이다.

배포중 문제가 발생하면 이전 버전의 서버 슬롯을 다시 시작하고 consumer와 HAProxy를 되돌리면 된다.

단순화하면 다음과 같다.

```bash
rollback() {
  # target slot 중지
  for server in ${DEPLOY_HOSTS}; do
    ssh devops@${server} "sudo systemctl stop ${TARGET_SERVICE_UNIT}"
  done

  # previous slot consumer 재활성화
  for server in ${DEPLOY_HOSTS}; do
    ssh devops@${server} \
      "sudo curl -X POST -H 'Content-Type: application/json' \
       -d '{\"action\":\"start\"}' \
       http://localhost:${PREVIOUS_PORT}/actuator/kafkaConsumer"
  done

  # HAProxy를 previous slot으로 복구
  switch_haproxy_to_previous
}
```

Rolling 배포에서는 롤백을 위해 이전 버전을 다시 배포해야 했지만, Blue-Green에서는 이전 슬롯이 남아 있어 전환만으로 빠르게 되돌릴 수 있다.

---

## 결과

| 항목 | Before | After |
|---|---|---|
| 배포 전략 | Rolling | Blue-Green |
| 버전 혼재 | 발생 가능 | 전환 시점 통제 |
| 롤백 | 이전 버전 재배포 필요 | 이전 슬롯 즉시 복구 |
| Health Check | 로그 파일 tail | `/actuator/health` |
| Warm-up | 없음 | `/actuator/info` 호출 |
| Kafka Consumer 전환 | restart/rebalance 허용 | pause → polling → shutdown → start |
| Green Consumer 시작 | 앱 기동과 동시에 시작 | 명시적으로 start |
| 배포 설정 | CI 파일에 혼재 | config 파일로 분리 |
| CI 구조 | 단일 파일 중심 | stage별 include 분리 |
| 배포 로그 | 단순 echo | step 단위 로깅 |
| 배포 시간 | 서버별 순차 대기 | 병렬 기동 + 순차 확인 |

---

## 결론

이번 작업에서 가장 크게 배운 점은 Blue-Green 배포가 서비스 유형에 따라 전혀 다르게 설계되어야 한다는 것이다.

Web API 서버의 Blue-Green 배포는 트래픽 전환이 핵심이다. 하지만 Kafka Consumer 서비스는 트래픽이 아니라 **consumer group 참여 시점**이 핵심이었다.

최종적으로 선택한 전환 흐름은 다음과 같다.

```text
Green 인스턴스 기동
    단, consumer는 아직 시작하지 않음
        ↓
Green health check / warm-up
        ↓
Blue consumer pause
        ↓
Blue consumer가 실제로 pause 상태가 되었는지 polling
        ↓
Blue consumer shutdown
        ↓
Green consumer start
        ↓
HAProxy 전환
        ↓
이전 슬롯 정리
```

핵심은 다음 세 가지였다.

1. Green 인스턴스는 먼저 띄우되 Kafka Consumer는 자동 시작하지 않는다.
2. Blue Consumer는 바로 죽이지 않고 `pause → 상태 확인 → shutdown` 순서로 내린다.
3. 고정 sleep이 아니라 실제 상태를 polling해서 전환한다.

이를 통해 배포 중 메시지 처리 공백을 최소화하고, consumer group에 Blue/Green이 동시에 참여하는 시간을 통제할 수 있었다.

정리하면 다음과 같다.

1. Kafka Consumer 서비스의 Blue-Green 배포에서는 consumer 자동 시작을 제어해야 한다.
2. 같은 consumer group에 Blue/Green이 동시에 참여하는 시간을 최소화해야 한다.
3. Consumer를 내릴 때는 고정 sleep보다 실제 상태 polling이 안전하다.
4. Spring Boot 버전에 따라 readiness endpoint 사용 가능 여부가 다르다.
5. Health check는 로그보다 HTTP endpoint 기반이 안정적이다.
6. Blue-Green 상태 판단은 파일보다 실제 프로세스 상태를 기준으로 하는 편이 안전하다.
7. 배포 전략과 CI/CD 구조를 함께 정리해야 운영 부담이 줄어든다.

---

## 마무리

Kafka Consumer 서비스에 Blue-Green 배포를 적용하면서 가장 중요한 것은 “새 버전을 어떻게 띄울 것인가”보다 “언제 consumer group에 참여시킬 것인가”였다.

최종적으로는 Green 인스턴스를 먼저 띄우고 검증한 뒤, Blue Consumer를 안전하게 멈추고 Green Consumer를 명시적으로 시작하는 구조로 정리했다.

덕분에 배포 중 버전 혼재와 롤백 부담을 줄였고, Kafka Consumer 전환 과정도 더 예측 가능하게 만들 수 있었다.

Blue-Green 배포를 Consumer 서비스에 적용할 때는 단순히 Web 서버의 배포 패턴을 그대로 가져오기보다, 메시지 처리 모델과 consumer group 동작 방식을 기준으로 전환 순서를 설계해야 한다.
