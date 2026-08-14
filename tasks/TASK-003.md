# TASK-003 프로젝트 기반과 실행 경계 초기화

## 목적

공개 web, 비공개 studio와 pipeline worker를 독립적으로 실행·검증할 수 있는 최소 TypeScript 모노레포 기반을 만들고 이후 데이터 모델 및 화면 작업의 안전한 경계를 제공한다.

## 사용자

- 사용자 역할: 개발자와 운영자
- 시작 조건: `docs/ARCHITECTURE.md`, `docs/SECURITY.md`, `docs/DEPLOYMENT.md`를 확인한다.

## 범위

- 포함:
  - TypeScript workspace와 재현 가능한 패키지 잠금 파일
  - `apps/web`, `apps/studio`, `workers/pipeline` entrypoint
  - `packages/domain`, `database`, `publication`, `ai-gateway`, `observability`, `ui` 경계
  - 공통 포매팅, lint, 타입 검사와 테스트 실행 명령
  - 환경변수 schema와 값이 없는 `.env.example`
  - 각 실행 단위의 health check와 구조화 로그 기반
  - CI에서 설치, 정적 검사, 테스트와 빌드
- 제외:
  - 실제 데이터베이스 스키마와 마이그레이션
  - 외부 AI 또는 자료 API 연결
  - 인증 및 운영자 기능
  - 실제 육아 콘텐츠와 사용자 화면 완성
  - 클라우드 배포

## 사용자 흐름

1. 개발자가 문서의 단일 명령으로 의존성을 설치한다.
2. web, studio와 pipeline을 각각 실행한다.
3. 각 entrypoint의 health check와 최소 화면 또는 프로세스 상태를 확인한다.
4. 정적 검사, 단위 테스트와 빌드를 한 명령으로 실행한다.
5. CI가 같은 잠금 파일과 명령으로 결과를 재현한다.

## 완료 조건

- [ ] 지원 런타임, 패키지 관리자와 버전 정책이 문서화된다.
- [ ] 세 실행 단위가 독립 entrypoint와 환경 설정을 갖는다.
- [ ] 공개 web이 studio 또는 pipeline 전용 패키지에 의존하지 않는 규칙이 자동 검사된다.
- [ ] 외부 AI SDK를 `packages/ai-gateway` 밖에서 import할 수 없는 규칙이 마련된다.
- [ ] 비밀 값 없이 필요한 환경변수 이름과 목적을 검증할 수 있다.
- [ ] 개발·테스트·운영 설정이 분리된다.
- [ ] 설치, lint, typecheck, test와 build가 CI 및 로컬에서 통과한다.
- [ ] health check가 내부 경로와 비밀정보를 노출하지 않는다.

## 예외 및 경계 상황

- 필수 환경변수가 없거나 형식이 잘못된 경우
- web만 실행하고 studio 또는 pipeline이 중단된 경우
- 서로 다른 Node 또는 패키지 관리자 버전을 사용하는 경우
- CI의 깨끗한 환경에서 잠금 파일과 설치 결과가 다른 경우
- pipeline이 종료 신호를 받거나 작업 중 비정상 종료되는 경우

## 데이터·보안 영향

- 추가·변경·삭제되는 데이터: 없음
- 개인정보 여부: 없음
- 인증 및 권한 조건: 실제 인증은 제외하지만 실행 단위별 자격증명 분리 지점을 마련한다.
- 로그 및 외부 전송 영향: 외부 로그 전송은 없으며 개인정보를 받지 않는 구조화 logger 인터페이스만 정의한다.

## 검증 계획

- 단위 테스트: 환경 설정과 모듈 경계 검사
- 통합 테스트: 각 entrypoint 실행 및 health check
- 사용자 흐름 테스트: 깨끗한 환경에서 설치부터 전체 검증 명령까지 실행
- 수동 확인: README와 실제 명령의 일치 여부

## 구현 전 확정 사항

- Node.js와 패키지 관리자 버전
- workspace 및 빌드 도구
- lint, formatter와 테스트 도구
- CI 제공자

## 작업 기록

- 상태: 대기
- 결정 사항: 한 저장소에서 세 실행 경계를 분리하고 공유 패키지는 방향성 의존 규칙을 갖는다.
- 미검증 사항: 실제 호스팅 환경과 개발 장비별 실행 결과
- 남은 위험: 초기 scaffold가 실제 기능보다 복잡해지거나 경계 검사가 형식적인 디렉터리 분리에 그칠 가능성
