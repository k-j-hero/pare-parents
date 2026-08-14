# 시스템 아키텍처

## 1. 목표와 판단 기준

Pare의 아키텍처는 다음 순서로 최적화한다.

1. 잘못된 육아 안내가 공개되지 않는가
2. 출처에서 사용자 답변까지 추적 가능한가
3. 개인정보와 검토 전 자료의 노출 범위가 최소인가
4. 부모가 필요한 내용을 모바일에서 빠르게 확인할 수 있는가
5. AI 또는 외부 서비스 장애 시에도 승인된 핵심 가이드를 제공하는가
6. 작은 팀이 운영하고 점진적으로 확장할 수 있는가
7. 실제 품질 개선 없이 비용과 복잡도를 늘리지 않는가

초기 목표는 최대 확장성이 아니라 **안전하게 변경 가능한 구조**다. 모듈 경계는 처음부터 명확히 하되, 측정된 필요가 생기기 전에는 네트워크 경계와 운영 구성요소를 늘리지 않는다.

## 2. 핵심 결정

Pare는 하나의 모노레포와 PostgreSQL을 기반으로 하되 실행 경계를 세 부분으로 분리한다.

- Delivery Plane: 부모에게 승인된 가이드를 빠르고 읽기 전용으로 제공
- Control Plane: 자료, 근거, 검토, 승인과 공개를 관리
- Processing Plane: 자료 수집, 갱신, 철회 확인, 검색 인덱스와 AI 보조 작업 처리

```text
                         Control Plane
                  +------------------------+
                  | Studio / Review API    |
                  | Evidence + Guide Write |
                  +-----------+------------+
                              |
                              v
Sources -> Quarantine -> PostgreSQL Authoring Model
                              |
                       approve + publish
                              |
                    transaction + outbox
                              |
                              v
                  Immutable Publication Snapshot
                              |
               +--------------+--------------+
               |                             |
               v                             v
        Public read model              Search index
               |                             |
               +--------------+--------------+
                              v
                         Delivery Plane
                  CDN / Public Web / Read API
                              |
                              v
                            Parent
```

AI 제공자는 모든 영역에서 선택적 외부 의존성이다. 공개된 기본 가이드 조회는 AI 없이 동작해야 한다.

## 3. 왜 현재 설계를 이렇게 보강하는가

기존 모듈형 모놀리스 방향은 유지한다. 다만 다음 문제를 해결한다.

| 기존 위험 | 개선 |
|---|---|
| 사용자 웹과 운영자 기능이 같은 배포 경계 | `web`과 `studio`를 별도 앱·권한·배포로 분리 |
| 사용자 요청이 복잡한 편집 모델을 직접 조회 | 공개 시점에 불변 snapshot과 read model 생성 |
| 공개 과정 중 일부 단계만 성공할 수 있음 | DB transaction과 outbox로 원자적 공개 요청 기록 |
| worker 실패 시 중복·유실 가능 | lease, idempotency key, 재시도와 dead-letter 상태 사용 |
| 여러 코드가 AI를 직접 호출할 가능성 | AI Gateway만 외부 모델 호출 허용 |
| 검색이 원시 논문과 검토 전 자료를 섞을 위험 | 공개 snapshot에서만 사용자 검색 인덱스 생성 |
| 미래 기능이 하나의 거대한 데이터 모델로 결합 | bounded context와 스키마 소유권을 미리 정의 |

## 4. 배포 가능한 구성요소

### `apps/web` — 공개 사용자 웹

- 모바일 우선 반응형 웹과 공개 읽기 API를 제공한다.
- 공개 snapshot 또는 read model만 읽는다.
- 운영자 쓰기 권한과 원문 저장소 접근 권한을 갖지 않는다.
- 선택형 탐색은 서버 렌더링 또는 정적 재검증으로 제공한다.
- AI, 검색 또는 데이터베이스 일부가 실패해도 캐시된 승인 가이드를 제공할 수 있어야 한다.

### `apps/studio` — 비공개 편집·검토 앱

- 출처 등록, 근거 평가, 가이드 작성, 검토와 공개를 제공한다.
- 운영자 인증, 역할 기반 권한, 중요 작업 재확인과 감사 로그를 적용한다.
- 인터넷에 공개해야 한다면 별도 도메인, 접근 정책과 강한 속도 제한을 사용한다.
- 공개 앱과 세션, 쿠키, 데이터베이스 자격증명을 공유하지 않는다.

### `workers/pipeline` — 비동기 처리

- 공식 API 자료 수집 및 변경 확인
- 원문 격리 검사와 구조화
- DOI·PMID·ISBN·내용 해시 중복 제거
- 라이선스, 정정·철회와 재검토 기한 확인
- publication snapshot과 검색 문서 생성
- 선택적인 AI 보조 작업

worker는 데이터베이스 outbox를 소비한다. 초기에는 별도 메시지 브로커를 두지 않고 PostgreSQL 행 잠금, lease와 idempotency key로 안전하게 처리한다.

### `packages/ai-gateway` — AI 단일 경계

- 모델 제공자 SDK는 이 패키지 밖에서 사용할 수 없다.
- 작업별 모델 라우팅, 구조화 입력·출력, 프롬프트 버전과 토큰 예산을 관리한다.
- 개인정보 최소화, 안전 식별자, 타임아웃, 재시도, 회로 차단과 공급자 오류 변환을 담당한다.
- 허용된 가이드 필드와 근거 ID만 모델에 전달한다.
- 모든 결과는 도메인 validator를 통과해야 하며 직접 공개할 수 없다.

### 공유 패키지

```text
apps/web                 공개 사용자 웹과 read API
apps/studio              편집·검토·공개 제어면
workers/pipeline         수집·갱신·snapshot·인덱스 작업
packages/domain          도메인 타입, 상태 전이와 불변 조건
packages/database        스키마, 트랜잭션, outbox와 마이그레이션
packages/evidence        출처·근거 평가와 provenance 규칙
packages/publication     snapshot 생성·검증과 read model
packages/ai-gateway      외부 AI 호출의 유일한 경계
packages/observability   구조화 이벤트, 지표와 개인정보 제거
packages/ui              접근 가능한 공통 UI
```

패키지는 다른 모듈의 테이블을 임의로 수정하지 않는다. 모듈의 application service 또는 명시된 repository 인터페이스를 사용한다.

## 5. 데이터 아키텍처

### 5.1 Authoring model

PostgreSQL의 정규화 모델은 작성과 검토의 기준 데이터다.

```text
Source -> SourceVersion -> EvidenceClaim -> EvidenceAssessment
                                         -> GuideEvidenceLink
Guide -> GuideVersion -> ReviewDecision -> PublicationRequest
                                      \-> AuditEvent
PublicationRequest -> OutboxEvent
```

중요한 불변 조건은 애플리케이션 검사만이 아니라 가능한 범위에서 외래키, unique, check constraint와 transaction으로 강제한다.

### 5.2 Publication snapshot

공개 단위는 여러 편집 테이블의 현재 상태가 아니라 승인 시점의 불변 snapshot이다.

snapshot에는 다음을 포함한다.

- `publication_id`, `guide_id`, `guide_version`
- 사용자에게 표시할 구조화된 행동·금지 행동·대화 예시
- 적용 연령, 지역, 언어와 상황 taxonomy
- 안전 안내와 escalation code
- 공개 가능한 출처와 근거 등급
- 검토자 역할, 승인 시각, 재검토 기한
- schema, renderer, safety policy와 content hash 버전

snapshot은 생성 후 수정하지 않는다. 변경은 새 publication을 만든다. 롤백은 이전 publication을 다시 active로 지정하는 가역적 작업이다.

### 5.3 Public read model

- 사용자 요청에 필요한 형태로 비정규화한다.
- `published`이며 현재 active인 snapshot만 포함한다.
- 공개 앱의 데이터베이스 역할은 read model만 읽을 수 있다.
- CDN 캐시 키는 publication ID, locale과 표현 형식을 포함한다.
- 검색 인덱스는 read model과 동일 publication ID를 사용해 결과와 본문을 일치시킨다.

### 5.4 Raw material quarantine

- 수집된 파일은 공개 파일과 다른 bucket 또는 prefix 및 권한으로 격리한다.
- 형식, 크기, 악성 파일, 라이선스와 수집 출처가 확인되기 전에는 처리 대상에만 머문다.
- 라이선스 미확인 원문은 영구 저장하지 않고 허용된 메타데이터와 링크만 보관한다.
- 원문, 추출 텍스트와 embedding의 삭제·보관 정책을 각각 기록한다.

### 5.5 Transactional outbox

공개 요청과 후속 작업 이벤트를 같은 transaction에 저장한다.

```text
approve publication
  -> insert publication_request
  -> insert outbox_event
  -> commit

worker
  -> lease event
  -> build + validate snapshot
  -> update read model and index idempotently
  -> mark event complete
```

중복 실행은 정상 상황으로 간주한다. 각 handler는 `event_id + handler_version`을 idempotency key로 사용한다.

## 6. 사용자 답변 아키텍처

```text
Request
  -> schema/rate-limit/privacy filter
  -> deterministic safety rules
  -> route decision
      0. exact guide: AI 없음
      1. classify: small-model candidate + schema validation
      2. compose: approved snapshots only + claim allowlist
      3. abstain/escalate: 추가 질문 또는 검증된 안전 안내
  -> output validator
  -> citation/publication binding
  -> response cache
```

### 필수 안전 특성

- 모델의 자체 지식은 사용자용 근거로 인정하지 않는다.
- 위험 판단은 모델 하나에 맡기지 않는다. 결정론 규칙, 모델 후보 판정과 사용자 확인 흐름을 계층화한다.
- 모델 출력은 새로운 행동 문장을 직접 추가하지 못한다. 허용된 claim 또는 content block ID 조합을 우선한다.
- timeout, 공급자 장애, 낮은 검색 점수와 validator 실패는 승인 가이드 또는 제한 응답으로 닫힌다.
- 모델 호출 전후의 raw 자유 문장은 기본 로그에서 제외한다.

OpenAI를 채택할 경우 모델·추론 수준은 작업별 평가로 선택하고, 반복 프롬프트 캐시와 사용량 필드를 측정한다. 구현 시점의 [공식 OpenAI 모델 가이드](https://developers.openai.com/api/docs/guides/latest-model)를 다시 확인한다.

## 7. 검색 전략

### 단계 1

- taxonomy, 연령, locale과 상황 코드의 정확 필터
- PostgreSQL 전문 검색
- 검색 대상은 publication snapshot만 허용

### 단계 2

- 오프라인 평가셋을 만든 뒤 lexical + vector 후보 검색을 비교
- 검색 필터를 먼저 적용하고 publication·안전 상태를 다시 검증
- hybrid 검색이 관련성 목표를 유의미하게 개선할 때만 pgvector 도입

### 단계 3

- PostgreSQL로 목표 지연·품질을 달성하지 못한 것이 재현될 때만 별도 검색 서비스를 검토

검색 점수가 낮거나 상위 결과가 충돌하면 생성으로 보완하지 않고 추가 질문 또는 제한 응답을 사용한다.

## 8. 미래 도메인 경계

장기 통합 플랫폼은 다음 bounded context로 확장한다.

| 도메인 | 소유 데이터 | Evidence와의 관계 |
|---|---|---|
| Evidence & Guidance | 출처, 평가, 가이드, publication | 제품의 신뢰 기준 |
| Identity & Family | 계정, 동의, 최소 프로필 | 가이드 ID만 참조 |
| Community | 게시글, 댓글, 신고, 제재 | 근거로 자동 승격 금지 |
| Learning | 강좌, 진도, 수료 | publication을 읽기 참조 |
| Commerce | 상품, 판매자, 주문, 추천 근거 | 이해상충과 추천 근거 별도 기록 |

각 도메인은 초기에는 같은 PostgreSQL cluster의 별도 schema를 사용할 수 있다. 소유 모듈 외의 직접 쓰기를 금지하고, 추출 필요가 생기면 outbox event와 공개 계약을 사용한다.

## 9. 보안 경계

- public, studio, worker는 서로 다른 런타임 자격증명과 최소 DB 역할을 사용한다.
- 운영자 인증 실패가 공개 가이드 제공에 영향을 주지 않아야 한다.
- publication 승인에는 서버 측 권한, 최신 버전 확인과 감사 이벤트가 필요하다.
- source 파일, 추출 텍스트, 검토 메모와 공개 콘텐츠의 저장·전송 권한을 분리한다.
- 외부 AI·분석·관찰 제공자별 데이터 흐름, 보관 기간과 삭제 방법을 기록한다.
- 계정과 아동 데이터 도입 전 별도 개인정보 영향 및 위협 모델을 승인한다.

## 10. 신뢰성 및 관찰

다음 식별자를 모든 구조화 이벤트에 연결한다.

- request ID
- publication ID와 guide version
- route, prompt와 policy version
- outbox event와 handler version
- model provider와 model configuration ID

관찰 대상은 다음과 같다.

- 공개 가이드 조회 성공률과 지연
- stale 또는 누락 publication
- outbox backlog, 재시도와 dead-letter
- 철회·재검토 알림 처리 시간
- AI 미사용 비율, 토큰, 캐시, 비용과 validator 실패
- 안전 경로 전환 및 답변 제한 비율

원문 사용자 질문, 아동 정보와 검토 전 원문은 기본 관찰 이벤트에 포함하지 않는다.

## 11. 단계별 진화

### Phase 0 — 정적 검증

- 대표 가이드 3개를 검토된 JSON fixture로 관리
- AI, 계정, 외부 수집과 운영자 UI 없음
- snapshot schema와 모바일 사용자 흐름을 먼저 검증

### Phase 1 — Evidence foundation

- PostgreSQL authoring model, 상태 전이, audit와 publication snapshot
- 공개 web과 studio 실행 경계 분리
- DB outbox worker와 공개 read model

### Phase 2 — Controlled discovery

- 공식 API 기반 메타데이터 수집
- 라이선스·철회 갱신과 재검토 알림
- 운영자 검토 UI와 taxonomy 검색

### Phase 3 — Constrained AI

- 평가셋과 AI Gateway 도입
- 분류부터 시작하고 기준 통과 후 제한적 재구성 추가
- 모델·프롬프트 변경 canary와 rollback

### Phase 4 — Platform expansion

- Identity & Family를 별도 동의·보관 정책과 함께 도입
- Community, Learning, Commerce를 독립 모듈로 순차 추가
- 측정된 부하와 조직 경계를 근거로 필요한 모듈만 서비스로 추출

## 12. 도입하지 않는 것과 승격 조건

| 구성요소 | 초기 판단 | 도입 조건 |
|---|---|---|
| 마이크로서비스 | 도입하지 않음 | 독립 팀·배포·장애 격리 요구가 측정됨 |
| 메시지 브로커 | DB outbox 사용 | backlog·처리량·라우팅이 PostgreSQL 한계를 초과 |
| Redis | CDN·DB 캐시 우선 | 분산 rate limit 또는 hot-key 병목이 재현됨 |
| 전용 벡터 DB | 도입하지 않음 | pgvector 평가 후에도 품질·지연 목표 미달 |
| Elasticsearch/OpenSearch | 도입하지 않음 | PostgreSQL 검색 한계가 부하·평가로 입증됨 |
| 실시간 생성형 상담 | 도입하지 않음 | 안전·근거·비용 평가와 전문가 승인이 완료됨 |

## 13. 미정 사항

- 호스팅 제공자, 처리 지역과 데이터 거주성
- ORM 또는 SQL 마이그레이션 도구
- 운영자 인증 제공자와 역할 체계
- 백업 주기, RPO, RTO와 복원 시험 주기
- 사용자 트래픽, publication 수와 비용 한도
- 전문가 승인 역할과 법률 검토 범위

미정 사항은 구현 전에 작업 명세와 `docs/DECISIONS.md`에서 확정한다.
