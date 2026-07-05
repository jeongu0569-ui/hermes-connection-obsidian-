# Hermes 100명 사용자용 인증/프로필 라우팅 서버 구상

작성일: 2026-07-05

## 1. 결론

실현 가능합니다.

다만 Obsidian 플러그인이 Hermes dashboard에 직접 붙는 현재 구조를 그대로 100명 서비스로 확장하는 것은 보안과 운영 측면에서 한계가 있습니다.

100명 규모를 생각한다면 권장 구조는 다음과 같습니다.

```text
Obsidian Plugin
  -> 우리 인증/라우팅 서버
  -> Hermes serve
  -> /p/{profile}/... 로 내부 라우팅
```

즉, 사용자는 Hermes dashboard 계정이나 프로필 URL을 직접 알 필요가 없고, 플러그인은 우리 서버에만 로그인합니다. 우리 서버가 로그인한 사용자를 확인한 뒤 해당 사용자에게 허용된 Hermes profile로만 요청을 전달합니다.

## 2. 왜 Hermes에 직접 붙이면 부족한가

현재 플러그인은 대략 이런 방식입니다.

```text
Hermes Server URL: http://100.x.x.x:9119/
Dashboard username: ...
Dashboard password: ...
```

가족 10명 정도가 같은 dashboard ID/password를 공유하는 정도라면 현실적으로 쓸 수 있습니다.

하지만 100명 서비스에서는 문제가 생깁니다.

```text
문제 1. 모든 사용자가 같은 dashboard 계정을 공유
문제 2. 사용자가 /p/다른프로필/ 주소를 알면 접근 가능할 수 있음
문제 3. 프로필별 권한 정책을 플러그인만으로 강제하기 어려움
문제 4. dashboard 비밀번호가 유출되면 전체 Hermes 서버가 위험
문제 5. 사용자 삭제, 정지, 요금제 제한, 사용량 추적을 하기 어려움
```

즉, Hermes profile은 상태 분리에는 좋지만, 그 자체가 SaaS식 사용자 권한 시스템은 아닙니다.

## 3. Hermes에서 활용 가능한 기존 구조

확인한 Hermes 구조상 다음 기능을 활용할 수 있습니다.

### 3.1 Profile

Hermes profile은 별도의 Hermes home directory를 가집니다.

예:

```text
default:
  ~/.hermes

testprofile:
  ~/.hermes/profiles/testprofile
```

프로필별로 나뉠 수 있는 것:

```text
config.yaml
.env
SOUL.md
memories
sessions
skills
cron jobs
state.db
provider/model 설정
MCP 설정
```

주의할 점:

Codex auth처럼 global/root auth 또는 `~/.codex/auth.json`을 fallback으로 읽는 예외가 있습니다. 진짜 사용자별 auth 분리가 필요하면 OS 계정, 컨테이너, CODEX_HOME 분리까지 고려해야 합니다.

### 3.2 multiplex profiles

Hermes에는 하나의 서버에서 여러 프로필을 `/p/{profile}/` prefix로 처리하는 구조가 있습니다.

개념:

```text
http://127.0.0.1:9119/p/father/
http://127.0.0.1:9119/p/mother/
http://127.0.0.1:9119/p/user-001/
```

단, 코드 기준으로 `gateway.multiplex_profiles`가 꺼져 있으면 `/p/{profile}/` prefix가 프로필 선택으로 동작하지 않고 무시될 수 있습니다.

필요 설정:

```bash
hermes config set gateway.multiplex_profiles true
hermes serve --host 0.0.0.0 --port 9119
```

### 3.3 Live WebSocket

Hermes dashboard는 실시간 이벤트를 위해 WebSocket을 씁니다.

확인된 구조:

```text
POST /api/auth/ws-ticket
GET  /api/ws?ticket=...
```

플러그인이 현재 쓰는 핵심 흐름도 이와 유사합니다.

```text
1. dashboard username/password로 로그인
2. /api/auth/ws-ticket 발급
3. /api/ws WebSocket 연결
4. session.create
5. prompt.submit
6. thinking/tool/approval/message 이벤트 수신
```

따라서 우리 인증/라우팅 서버가 이 흐름을 대신 처리하거나, 사용자별로 안전하게 프록시할 수 있습니다.

## 4. 권장 전체 아키텍처

```text
[Obsidian Plugin]
  |
  | 1. user login token
  v
[Hermes Routing Server]
  |
  | 2. user -> profile 매핑 확인
  | 3. 권한/요금제/사용량 확인
  | 4. Hermes 내부 주소로 proxy
  v
[Hermes serve]
  |
  | /p/user-001/api/ws
  | /p/user-001/api/sessions
  | /p/user-001/api/model/options
  v
[Hermes profile: user-001]
```

사용자는 Hermes dashboard 계정을 몰라도 됩니다.

플러그인에는 다음만 저장합니다.

```text
Service URL: https://your-domain.com
User token 또는 로그인 세션
```

Hermes dashboard username/password는 서버 내부 환경변수로만 보관합니다.

## 5. 사용자 요청 처리 흐름

### 5.1 로그인

```text
Obsidian Plugin
  -> POST /auth/login
  <- access_token
```

서버 DB:

```text
users
  id
  email
  password_hash 또는 oauth_subject
  status
  plan

user_profiles
  user_id
  hermes_profile
  allowed_scopes
```

예:

```text
userA -> profile user-a
userB -> profile user-b
```

### 5.2 세션 목록

```text
Plugin
  -> GET /api/sessions
     Authorization: Bearer user-token

Routing Server
  -> user-token 검증
  -> userA는 profile user-a만 허용
  -> Hermes 내부 요청:
     GET http://127.0.0.1:9119/p/user-a/api/sessions
```

### 5.3 새 메시지 전송

WebSocket 방식:

```text
Plugin
  -> WSS /api/ws
     Authorization 또는 ticket

Routing Server
  -> 사용자 인증
  -> userA -> profile user-a 확인
  -> 내부 Hermes WebSocket:
     ws://127.0.0.1:9119/p/user-a/api/ws?ticket=...
```

그 뒤 JSON-RPC 메시지를 중계합니다.

```text
session.create
prompt.submit
approval.respond
projects.tree
projects.project_sessions
```

### 5.4 승인/거절

```text
Plugin
  -> approval.respond

Routing Server
  -> 해당 session_id가 userA/profile user-a 소유인지 확인
  -> Hermes로 전달
```

이 검증이 중요합니다. 단순히 session_id만 받아 Hermes로 넘기면 다른 사람의 세션에 승인/거절을 보낼 위험이 있습니다.

## 6. 플러그인 수정 방향

현재 플러그인 설정:

```text
Hermes Server URL
Dashboard username
Dashboard password
```

100명 서비스용 플러그인 설정:

```text
Service URL
Login email
Login password 또는 OAuth
```

또는 더 단순하게:

```text
Service URL
Personal access token
```

플러그인은 더 이상 Hermes dashboard password를 저장하지 않습니다.

변경되는 통신 대상:

```text
기존:
  plugin -> Hermes dashboard

변경:
  plugin -> Routing Server -> Hermes dashboard
```

플러그인 입장에서는 Hermes 세부 API를 몰라도 됩니다. 우리 서버 API만 맞추면 됩니다.

## 7. 라우팅 서버가 해야 하는 일

### 필수 기능

```text
사용자 로그인
사용자별 Hermes profile 매핑
프로필 URL 강제
세션 소유권 검증
WebSocket 프록시
REST API 프록시
승인/거절 중계
사용량 기록
요청 제한
로그 기록
관리자 페이지 또는 CLI
```

### 선택 기능

```text
사용자별 모델 제한
사용자별 MCP 제한
사용자별 최대 동시 세션 제한
사용자별 월간 토큰/요청 제한
가족/팀 단위 그룹 관리
프로필 자동 생성
프로필 백업/삭제
결제 연동
```

## 8. 서버 구현 방식

추천 스택:

```text
Node.js + Fastify
또는
Python + FastAPI
```

WebSocket 프록시가 중요하므로 둘 다 가능합니다.

개인적으로는 플러그인이 TypeScript/JavaScript 쪽이므로, 유지보수 측면에서 Node.js가 편합니다.

예상 구성:

```text
server/
  src/
    auth.ts
    users.ts
    profiles.ts
    hermesClient.ts
    wsProxy.ts
    restProxy.ts
    usage.ts
    admin.ts
  prisma/
    schema.prisma
  .env
```

DB:

```text
초기: SQLite
운영: PostgreSQL
```

## 9. Hermes profile 생성 전략

### 수동 생성

소규모 베타에서는 수동으로 충분합니다.

```bash
hermes profile create user-001
hermes -p user-001 model
```

서버 DB에 매핑:

```text
user_id: 1
profile: user-001
```

### 자동 생성

가입 시 자동 생성:

```bash
hermes profile create user-001 --clone
```

주의:

`--clone`은 config, .env, skills 등을 복사할 수 있습니다. 사용자별 API key를 완전히 분리하려면 clone 후 `.env`와 auth 설정을 정리해야 합니다.

## 10. 보안 체크포인트

반드시 지켜야 하는 것:

```text
1. 사용자가 profile 이름을 직접 지정하지 못하게 한다.
2. 서버가 user_id -> profile 매핑을 강제한다.
3. session_id가 해당 user/profile 소유인지 확인한다.
4. Hermes dashboard username/password는 클라이언트에 절대 내려주지 않는다.
5. WebSocket ticket도 클라이언트가 Hermes에 직접 쓰는 구조가 아니라 서버가 중계한다.
6. Tailscale 또는 내부망에서만 Hermes serve를 열고, public internet에는 routing server만 노출한다.
7. 사용자별 rate limit을 둔다.
8. mutating tool 승인/거절 이벤트를 감사 로그에 남긴다.
```

위험한 구조:

```text
Plugin이 http://server:9119/p/user-001/ 로 직접 붙음
Plugin에 dashboard password 저장
사용자가 profile URL을 직접 수정 가능
```

이 구조는 가족용으로는 가능하지만 100명 서비스에는 적합하지 않습니다.

## 11. isolated 100개 방식과 비교

### isolated 방식

```text
user-001 -> hermes serve --isolated --port 9201
user-002 -> hermes serve --isolated --port 9202
...
user-100 -> hermes serve --isolated --port 9300
```

장점:

```text
강한 격리
사용자별 dashboard auth 가능
문제 발생 시 해당 사용자 서버만 재시작 가능
```

단점:

```text
포트 100개 관리
프로세스 100개 관리
메모리 낭비
로그/업데이트/장애 대응 복잡
운영 자동화 필수
```

### routing server 방식

```text
사용자 100명
  -> routing server 1개 또는 소수
  -> hermes serve 1개 또는 소수
  -> profile 100개
```

장점:

```text
사용자 인증 통합
프로필 접근 제어 가능
포트 관리 단순
플러그인 설정 단순
사용량/요금제/로그 관리 가능
```

단점:

```text
별도 백엔드 개발 필요
WebSocket proxy 구현 필요
Hermes profile prefix API 동작 검증 필요
Hermes 업데이트에 따른 API 호환성 확인 필요
```

100명 서비스라면 routing server 방식이 더 현실적입니다.

## 12. 단계별 개발 계획

### 1단계: 가족/소규모 검증

```text
Hermes serve 1개
gateway.multiplex_profiles true
프로필 여러 개
플러그인 URL을 /p/{profile}/ 로 직접 입력
```

목표:

```text
프로필별 세션/모델/메모리가 실제로 분리되는지 확인
```

### 2단계: 간단한 routing server

```text
로그인 없음
사용자 토큰 하나당 profile 하나 매핑
REST proxy
WebSocket proxy
```

목표:

```text
플러그인이 Hermes password 없이 동작
```

### 3단계: 인증 추가

```text
email/password 또는 OAuth
JWT/session
사용자별 profile 매핑
세션 소유권 검증
```

목표:

```text
다른 사용자의 profile/session 접근 차단
```

### 4단계: 운영 기능

```text
사용량 측정
rate limit
관리자 페이지
프로필 생성/정지/삭제
감사 로그
백업
```

목표:

```text
가족용을 넘어 소규모 서비스 운영 가능
```

### 5단계: 진짜 서비스화

```text
PostgreSQL
reverse proxy
TLS
monitoring
queue
worker 분리
결제/요금제
사용자별 resource quota
```

## 13. 가장 먼저 테스트해야 할 것

아래가 실제로 동작해야 routing server 방식이 깔끔해집니다.

```text
1. gateway.multiplex_profiles true 설정
2. hermes serve --host 0.0.0.0 --port 9119
3. /p/testprofile/api/model/options 동작 확인
4. /p/testprofile/api/auth/ws-ticket 동작 확인
5. /p/testprofile/api/ws WebSocket 동작 확인
6. session.create 결과가 testprofile 세션 DB에 저장되는지 확인
7. prompt.submit이 testprofile 모델/config로 실행되는지 확인
```

만약 `/p/testprofile/api/ws`가 제대로 동작하지 않으면 routing server가 Hermes 내부 API를 직접 쓰는 대신, 사용자별 isolated 서버 또는 dashboard profile query 방식을 검토해야 합니다.

## 14. 추천 결론

가족 10명:

```text
Tailscale + profile 여러 개 + 공용 dashboard password
또는
간단한 routing server
```

100명 서비스:

```text
Obsidian Plugin
  -> 사용자 인증/프로필 라우팅 서버
  -> Hermes serve multiplex
  -> user별 Hermes profile
```

이 방향이 가장 현실적입니다.

핵심은 Hermes를 다중 사용자 인증 서버로 직접 쓰지 않고, Hermes는 agent runtime으로 두는 것입니다.

```text
우리 서버 = 사용자/권한/라우팅/과금/로그 담당
Hermes = 실제 에이전트 세션/모델/툴 실행 담당
Obsidian 플러그인 = 원격 클라이언트 UI
```

## 15. 참고한 근거

공식 문서:

```text
Hermes Profiles: profile은 config, API keys, memory, sessions, skills, gateway state를 분리
Hermes Web Dashboard: dashboard는 profile switcher를 제공하며, isolated는 프로필별 dashboard/auth 분리에 유용
```

로컬 코드 확인:

```text
gateway.multiplex_profiles 기본값은 false
/p/<profile>/ prefix는 multiplex_profiles가 켜졌을 때 프로필 라우팅으로 의미가 있음
/api/auth/ws-ticket -> /api/ws WebSocket 구조가 존재
projects.tree, projects.project_sessions, session.create, prompt.submit 같은 JSON-RPC 흐름이 존재
```
