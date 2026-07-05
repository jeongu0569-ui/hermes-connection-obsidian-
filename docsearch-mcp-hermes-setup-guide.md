# docsearch-mcp + Hermes + Obsidian Vault 연동 강의서

## 0. 목표

이 문서는 Obsidian Vault 내용을 Hermes가 RAG 검색으로 참고할 수 있도록 `docsearch-mcp`를 설치하고 연결하는 전체 과정을 정리합니다.

최종 구조는 다음과 같습니다.

```text
Obsidian Vault
  ↓
docsearch-mcp watcher
  ↓
SQLite + sqlite-vec 인덱스
  ↓
docsearch-mcp MCP server
  ↓
Hermes MCP tool
  ↓
Hermes Agent가 Vault 내용 검색
```

이번 구성의 핵심은 `docsearch-mcp`를 두 가지 역할로 나누어 쓰는 것입니다.

```text
1. watcher 프로세스
   파일 변경을 감지하고 인덱스를 최신 상태로 유지

2. MCP server 프로세스
   Hermes가 검색 도구로 호출
```

`docsearch-mcp`의 MCP server 자체는 자동 watch를 하지 않습니다. 그래서 자동 인덱싱을 원하면 별도 watcher 프로세스를 계속 실행해야 합니다.

---

## 1. 준비물

필요한 구성 요소는 다음과 같습니다.

```text
Node.js
npm
Ollama OpenAI-compatible API
bge-m3 임베딩 모델
Hermes
Obsidian Vault 폴더
```

이번 환경에서는 다음 상태였습니다.

```bash
node --version
npm --version
```

확인 결과:

```text
Node.js v22.23.1
npm 10.9.8
```

Ollama의 OpenAI 호환 API는 다음 주소에서 동작 중이었습니다.

```text
http://localhost:11434/v1
```

모델 확인:

```bash
curl -sS http://localhost:11434/v1/models
```

사용한 임베딩 모델:

```text
bge-m3:latest
embedding dimension: 1024
```

---

## 2. docsearch-mcp 설치

전역 npm 패키지로 설치합니다.

```bash
npm install -g docsearch-mcp
```

설치된 버전 확인:

```bash
npm view docsearch-mcp version
```

이번에 설치한 버전:

```text
0.4.0
```

---

## 3. npm 패키지 bin 경로 문제

설치 후 바로 다음 명령이 동작해야 합니다.

```bash
docsearch-mcp --help
```

하지만 이번 환경에서는 다음 문제가 있었습니다.

```text
docsearch-mcp not found
```

원인은 `docsearch-mcp` 패키지의 `package.json`에 들어 있는 bin 경로가 실제 파일 구조와 맞지 않았기 때문입니다.

패키지에는 이렇게 되어 있었습니다.

```json
"bin": {
  "docsearch-mcp": "dist/src/cli/main.js"
}
```

하지만 실제 파일은 여기에 있었습니다.

```text
/Users/user/.local/lib/node_modules/docsearch-mcp/dist/cli/main.js
```

그래서 실행 wrapper를 직접 만들었습니다.

파일:

```text
/Users/user/.local/bin/docsearch-mcp
```

내용:

```sh
#!/bin/sh
exec /Users/user/.local/bin/node /Users/user/.local/lib/node_modules/docsearch-mcp/dist/cli/main.js "$@"
```

실행 권한 부여:

```bash
chmod +x /Users/user/.local/bin/docsearch-mcp
```

이후 확인:

```bash
/Users/user/.local/bin/docsearch-mcp --version
```

결과:

```text
0.4.0
```

---

## 4. docsearch-mcp 설정 파일 만들기

운영용 설정 폴더를 만들었습니다.

```bash
mkdir -p /Users/user/.hermes/docsearch-mcp
```

설정 파일:

```text
/Users/user/.hermes/docsearch-mcp/.env
```

내용:

```env
DOTENV_CONFIG_QUIET=true
EMBEDDINGS_PROVIDER=openai
OPENAI_API_KEY=ollama
OPENAI_BASE_URL=http://localhost:11434/v1
OPENAI_EMBED_MODEL=bge-m3:latest
OPENAI_EMBED_DIM=1024
DB_TYPE=sqlite
DB_PATH=/Users/user/.hermes/docsearch-mcp/index.db
FILE_ROOTS=/Users/user/Library/Mobile Documents/iCloud~md~obsidian/Documents/컴퓨터
FILE_INCLUDE_GLOBS=**/*.md,**/*.txt,**/*.pdf,**/*.js,**/*.ts,**/*.py,**/*.json,**/*.yaml,**/*.yml,**/*.html,**/*.css,**/*.rs,**/*.go,**/*.java,**/*.cpp,**/*.c,**/*.h
FILE_EXCLUDE_GLOBS=**/.git/**,**/node_modules/**,**/dist/**,**/build/**,**/target/**,**/.obsidian/plugins/**,**/.trash/**
```

각 항목의 의미는 다음과 같습니다.

```text
DOTENV_CONFIG_QUIET=true
stdout에 dotenv 안내 문구가 섞이지 않게 함.
MCP stdio 연결에서는 매우 중요합니다.

EMBEDDINGS_PROVIDER=openai
OpenAI 호환 임베딩 API를 사용한다는 뜻입니다.

OPENAI_API_KEY=ollama
Ollama는 실제 API key가 필요 없지만, docsearch-mcp가 빈 값을 거부하므로 더미 값을 넣습니다.

OPENAI_BASE_URL=http://localhost:11434/v1
Ollama OpenAI-compatible endpoint입니다.

OPENAI_EMBED_MODEL=bge-m3:latest
사용할 로컬 임베딩 모델입니다.

OPENAI_EMBED_DIM=1024
bge-m3의 임베딩 차원입니다.

DB_TYPE=sqlite
로컬 SQLite DB를 사용합니다.

DB_PATH=/Users/user/.hermes/docsearch-mcp/index.db
검색 인덱스 DB 파일 위치입니다.

FILE_ROOTS=...
감시하고 색인할 Obsidian Vault root입니다.

FILE_INCLUDE_GLOBS=...
색인할 파일 확장자 목록입니다.

FILE_EXCLUDE_GLOBS=...
제외할 폴더 목록입니다.
```

주의할 점:

`FILE_INCLUDE_GLOBS`에는 brace glob를 쓰면 안 됩니다.

나쁜 예:

```env
FILE_INCLUDE_GLOBS=**/*.{md,js,pdf}
```

이 값은 내부에서 comma split되면서 깨질 수 있습니다.

좋은 예:

```env
FILE_INCLUDE_GLOBS=**/*.md,**/*.js,**/*.pdf
```

---

## 5. 초기 인덱싱

설정 폴더에서 초기 인덱싱을 실행합니다.

```bash
cd /Users/user/.hermes/docsearch-mcp
/Users/user/.local/bin/docsearch-mcp ingest files
```

성공하면 다음과 비슷하게 나옵니다.

```text
Ingesting files...
Files processed, generating embeddings...
Files ingested.
Ingestion completed successfully
```

이번 실제 결과:

```text
49 documents
108 chunks
```

SQLite로 직접 확인할 수 있습니다.

```bash
sqlite3 /Users/user/.hermes/docsearch-mcp/index.db \
  'select count(*) documents from documents; select count(*) chunks from chunks;'
```

---

## 6. 검색 테스트

예를 들어 `mcp-loco-setup.md` 관련 내용을 검색합니다.

```bash
cd /Users/user/.hermes/docsearch-mcp
/Users/user/.local/bin/docsearch-mcp search "Hermes mcp-loco obsidian rag 연결" \
  --mode vector \
  --top-k 5 \
  --output json
```

실제 테스트에서 `mcp-loco-setup.md`가 정상 검색되었습니다.

검색 결과에 포함된 파일:

```text
hermes continue(obsidian plugin)/mcp-loco-setup.md
```

검색 모드는 세 가지가 있습니다.

```text
keyword
정확한 단어/문구 검색에 강함

vector
의미 기반 검색에 강함

auto
keyword + vector 혼합
```

주의:

이번 테스트에서 `auto` 모드는 exact marker 검색 순위가 가끔 이상했습니다. 정확한 문자열 검증에는 `keyword`, 의미 검색에는 `vector`가 더 안정적이었습니다.

---

## 7. watcher 실행 방식

자동 인덱싱을 위해 watcher를 실행합니다.

중요:

`docsearch-mcp`의 `start` MCP 서버는 자동 watch를 하지 않습니다.

자동 watch는 다음 명령으로 별도 실행해야 합니다.

```bash
docsearch-mcp ingest files --watch
```

이번에 만든 watcher 스크립트:

```text
/Users/user/.hermes/docsearch-mcp/watch.sh
```

내용:

```sh
#!/bin/sh
set -eu

ROOT="/Users/user/Library/Mobile Documents/iCloud~md~obsidian/Documents/컴퓨터"

cd "$ROOT"

export DOTENV_CONFIG_QUIET=true
export EMBEDDINGS_PROVIDER=openai
export OPENAI_API_KEY=ollama
export OPENAI_BASE_URL=http://localhost:11434/v1
export OPENAI_EMBED_MODEL=bge-m3:latest
export OPENAI_EMBED_DIM=1024
export DB_TYPE=sqlite
export DB_PATH=/Users/user/.hermes/docsearch-mcp/index.db
export FILE_ROOTS="$ROOT"
export FILE_INCLUDE_GLOBS="**/*.md,**/*.txt,**/*.pdf,**/*.js,**/*.ts,**/*.py,**/*.json,**/*.yaml,**/*.yml,**/*.html,**/*.css,**/*.rs,**/*.go,**/*.java,**/*.cpp,**/*.c,**/*.h"
export FILE_EXCLUDE_GLOBS="**/.git/**,**/node_modules/**,**/dist/**,**/build/**,**/target/**,**/.obsidian/plugins/**,**/.trash/**"

exec /Users/user/.local/bin/docsearch-mcp ingest files --watch
```

실행 권한:

```bash
chmod +x /Users/user/.hermes/docsearch-mcp/watch.sh
```

직접 실행 테스트:

```bash
/Users/user/.hermes/docsearch-mcp/watch.sh
```

정상 출력:

```text
Watching for changes...
Ingesting files...
Files processed, generating embeddings...
Files ingested.
```

---

## 8. watcher 자동 실행 등록

Mac에서 로그인 후 자동으로 watcher가 실행되도록 LaunchAgent를 등록했습니다.

파일:

```text
/Users/user/Library/LaunchAgents/com.user.docsearch-mcp.watch.plist
```

내용:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.user.docsearch-mcp.watch</string>
  <key>ProgramArguments</key>
  <array>
    <string>/Users/user/.hermes/docsearch-mcp/watch.sh</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>StandardOutPath</key>
  <string>/Users/user/.hermes/docsearch-mcp/watch.log</string>
  <key>StandardErrorPath</key>
  <string>/Users/user/.hermes/docsearch-mcp/watch.err.log</string>
  <key>WorkingDirectory</key>
  <string>/Users/user/.hermes/docsearch-mcp</string>
  <key>EnvironmentVariables</key>
  <dict>
    <key>PATH</key>
    <string>/Users/user/.local/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
  </dict>
</dict>
</plist>
```

등록:

```bash
launchctl bootstrap gui/$(id -u) /Users/user/Library/LaunchAgents/com.user.docsearch-mcp.watch.plist
```

재등록할 때는 기존 것을 먼저 내립니다.

```bash
launchctl bootout gui/$(id -u) /Users/user/Library/LaunchAgents/com.user.docsearch-mcp.watch.plist 2>/dev/null || true
launchctl bootstrap gui/$(id -u) /Users/user/Library/LaunchAgents/com.user.docsearch-mcp.watch.plist
```

상태 확인:

```bash
launchctl print gui/$(id -u)/com.user.docsearch-mcp.watch
```

실제 확인 결과:

```text
state = running
pid = 61025
last exit code = (never exited)
```

프로세스 확인:

```bash
ps -axo pid,ppid,command | rg 'docsearch-mcp|watch.sh|PID'
```

---

## 9. 자동 인덱싱 테스트

### 9.1 watcher가 켜진 상태에서 새 파일 추가

테스트 파일을 만들었습니다.

```text
docsearch-watch-live-test.md
```

내용에는 고유 marker를 넣었습니다.

```text
DOCSEARCH_WATCH_LIVE_DELTA_20260702
```

검색 루프:

```bash
cd /Users/user/.hermes/docsearch-mcp
TOKEN='DOCSEARCH_WATCH_LIVE_DELTA_20260702'

for i in $(seq 0 60); do
  OUT=$(DOTENV_CONFIG_QUIET=true /Users/user/.local/bin/docsearch-mcp search "$TOKEN" --mode keyword --top-k 3 --output json 2>&1)
  if printf '%s' "$OUT" | rg -q 'docsearch-watch-live-test.md'; then
    echo "FOUND"
    printf '%s\n' "$OUT"
    exit 0
  fi
  sleep 1
done
```

결과:

```text
FOUND after 0.21s
```

즉 watcher가 켜진 상태에서는 새 Markdown 파일이 거의 즉시 검색에 반영되었습니다.

### 9.2 watcher가 꺼진 동안 파일 추가 후 재시작

watcher를 끄고 파일을 추가했습니다.

```text
docsearch-watch-restart-test.md
```

고유 marker:

```text
DOCSEARCH_WATCH_RESTART_ECHO_20260702
```

watcher를 다시 켠 뒤 검색했습니다.

결과:

```text
FOUND after 0.19s
```

즉 watcher가 꺼진 동안 추가된 파일도 다시 켜면 초기 ingest 단계에서 반영됩니다.

### 9.3 LaunchAgent 상태에서 자동 인덱싱

LaunchAgent로 watcher를 실행한 상태에서 새 파일을 추가했습니다.

```text
docsearch-launchagent-test.md
```

고유 marker:

```text
DOCSEARCH_LAUNCH_AGENT_FOXTROT_20260702
```

결과:

```text
FOUND after 0.21s
```

LaunchAgent 기반 자동 인덱싱도 정상 동작했습니다.

---

## 10. 테스트 파일 정리

테스트 후 만든 임시 파일들은 삭제했습니다.

```text
docsearch-watch-live-test.md
docsearch-watch-restart-test.md
docsearch-launchagent-test.md
```

삭제 후 인덱스를 현재 Vault 상태와 맞추기 위해 수동 ingest를 한 번 더 실행했습니다.

```bash
cd /Users/user/.hermes/docsearch-mcp
/Users/user/.local/bin/docsearch-mcp ingest files
```

최종 인덱스 상태:

```text
49 documents
108 chunks
```

---

## 11. Hermes MCP 연결

Hermes 설정 파일:

```text
/Users/user/.hermes/config.yaml
```

`mcp_servers` 아래에 다음 항목을 추가했습니다.

```yaml
mcp_servers:
  obsidian-docsearch:
    command: /Users/user/.local/bin/docsearch-mcp
    args:
      - start
    connect_timeout: 90
    timeout: 90
    env:
      DOTENV_CONFIG_QUIET: "true"
      EMBEDDINGS_PROVIDER: openai
      OPENAI_API_KEY: ollama
      OPENAI_BASE_URL: http://localhost:11434/v1
      OPENAI_EMBED_MODEL: bge-m3:latest
      OPENAI_EMBED_DIM: "1024"
      DB_TYPE: sqlite
      DB_PATH: /Users/user/.hermes/docsearch-mcp/index.db
      FILE_ROOTS: /Users/user/Library/Mobile Documents/iCloud~md~obsidian/Documents/컴퓨터
      FILE_INCLUDE_GLOBS: "**/*.md,**/*.txt,**/*.pdf,**/*.js,**/*.ts,**/*.py,**/*.json,**/*.yaml,**/*.yml,**/*.html,**/*.css,**/*.rs,**/*.go,**/*.java,**/*.cpp,**/*.c,**/*.h"
      FILE_EXCLUDE_GLOBS: "**/.git/**,**/node_modules/**,**/dist/**,**/build/**,**/target/**,**/.obsidian/plugins/**,**/.trash/**"
```

중요:

```text
args:
  - start
```

Hermes가 연결하는 프로세스는 watcher가 아니라 MCP server입니다. 그래서 `docsearch-mcp start`를 실행해야 합니다.

반대로 자동 인덱싱 watcher는 다음 별도 프로세스입니다.

```bash
docsearch-mcp ingest files --watch
```

---

## 12. Hermes MCP 연결 테스트

Hermes에서 MCP 서버 연결을 테스트했습니다.

```bash
hermes mcp test obsidian-docsearch
```

결과:

```text
Testing 'obsidian-docsearch'...
Transport: stdio → /Users/user/.local/bin/docsearch-mcp
Auth: none
✓ Connected
✓ Tools discovered: 3

doc-ingest
doc-ingest-status
doc-search
```

즉 Hermes가 `docsearch-mcp`를 MCP 서버로 정상 인식했습니다.

---

## 13. Hermes 도구 활성화 확인

도구 목록 확인:

```bash
hermes tools list
```

결과 중 MCP 서버 항목:

```text
MCP servers:
  obsidian-rag  all tools enabled
  obsidian-docsearch  all tools enabled
```

---

## 14. Hermes Agent 실제 검색 테스트

Hermes Agent에게 실제로 MCP 도구를 쓰게 했습니다.

명령:

```bash
hermes -z 'Use the obsidian-docsearch MCP doc-search tool to search for "mcp-loco BGE-M3 Hermes 연결" in the local document index. Reply with the top result title and path only.' \
  --provider nous \
  -m stepfun/step-3.7-flash:free
```

결과:

```text
Top result:

Title: mcp-loco-setup.md
Path: hermes continue(obsidian plugin)/mcp-loco-setup.md
```

이 테스트는 Hermes가 실제로 `obsidian-docsearch` MCP 도구를 통해 Vault 인덱스를 검색할 수 있음을 확인한 것입니다.

---

## 15. PDF와 코드 파일 검색에 대하여

`docsearch-mcp`는 기본적으로 다음 파일들을 처리할 수 있습니다.

```text
Markdown
Text
PDF
JavaScript
TypeScript
Python
JSON
YAML
HTML
CSS
Rust
Go
Java
C/C++
```

이번 실제 Vault root에는 PDF가 없었습니다.

확인:

```bash
find "/Users/user/Library/Mobile Documents/iCloud~md~obsidian/Documents/컴퓨터" -type f -iname '*.pdf'
```

결과:

```text
PDF 없음
```

그래서 별도 테스트 corpus에서 PDF와 JS 검색을 검증했습니다.

테스트 결과:

```text
Markdown 검색 성공
JavaScript 코드 검색 성공
PDF 텍스트 검색 성공
```

---

## 16. 여러 폴더를 함께 등록하는 방법

여러 프로젝트 폴더를 같이 검색하고 싶다면 `FILE_ROOTS`를 comma-separated 값으로 바꿉니다.

예:

```env
FILE_ROOTS=/Users/user/Library/Mobile Documents/iCloud~md~obsidian/Documents/컴퓨터,/Users/user/Documents/Projects,/Users/user/Documents/Research
```

같은 변경을 두 곳에 반영해야 합니다.

```text
/Users/user/.hermes/docsearch-mcp/.env
/Users/user/.hermes/docsearch-mcp/watch.sh
```

그리고 Hermes MCP 설정의 env에도 반영하면 좋습니다.

```text
/Users/user/.hermes/config.yaml
```

변경 후 다시 인덱싱합니다.

```bash
cd /Users/user/.hermes/docsearch-mcp
/Users/user/.local/bin/docsearch-mcp ingest files
```

watcher도 재시작합니다.

```bash
launchctl bootout gui/$(id -u) /Users/user/Library/LaunchAgents/com.user.docsearch-mcp.watch.plist
launchctl bootstrap gui/$(id -u) /Users/user/Library/LaunchAgents/com.user.docsearch-mcp.watch.plist
```

---

## 17. 운영 시 자주 쓸 명령어

### MCP 연결 테스트

```bash
hermes mcp test obsidian-docsearch
```

### Hermes 도구 활성화 확인

```bash
hermes tools list
```

### 인덱스 수동 갱신

```bash
cd /Users/user/.hermes/docsearch-mcp
/Users/user/.local/bin/docsearch-mcp ingest files
```

### 검색 테스트

```bash
cd /Users/user/.hermes/docsearch-mcp
/Users/user/.local/bin/docsearch-mcp search "검색어" --mode vector --top-k 5 --output json
```

### 정확한 문자열 검색

```bash
cd /Users/user/.hermes/docsearch-mcp
/Users/user/.local/bin/docsearch-mcp search "정확한 문자열" --mode keyword --top-k 5 --output json
```

### watcher 상태 확인

```bash
launchctl print gui/$(id -u)/com.user.docsearch-mcp.watch
```

### watcher 로그 확인

```bash
tail -100 /Users/user/.hermes/docsearch-mcp/watch.log
tail -100 /Users/user/.hermes/docsearch-mcp/watch.err.log
```

### watcher 재시작

```bash
launchctl bootout gui/$(id -u) /Users/user/Library/LaunchAgents/com.user.docsearch-mcp.watch.plist
launchctl bootstrap gui/$(id -u) /Users/user/Library/LaunchAgents/com.user.docsearch-mcp.watch.plist
```

---

## 18. mcp-loco와 비교한 결론

`mcp-loco`는 테스트 결과 파일 이벤트를 감지하긴 했지만, 현재 설치된 버전에서는 watcher가 실제 reindex를 호출하지 않았습니다.

반면 `docsearch-mcp`는 다음 테스트를 통과했습니다.

```text
새 Markdown 파일 생성 후 자동 검색 반영: 0.21초
watcher 꺼진 동안 파일 추가 후 재시작 반영: 0.19초
LaunchAgent 상태에서 자동 검색 반영: 0.21초
Hermes MCP 연결 성공
Hermes Agent 실제 검색 성공
```

그래서 현재 목적에는 `docsearch-mcp`가 더 적합합니다.

---

## 19. 최종 상태 요약

현재 구성:

```text
docsearch-mcp version: 0.4.0
embedding provider: Ollama OpenAI-compatible API
embedding model: bge-m3:latest
embedding dimension: 1024
database: /Users/user/.hermes/docsearch-mcp/index.db
watcher: LaunchAgent 실행 중
Hermes MCP server name: obsidian-docsearch
indexed documents: 49
indexed chunks: 108
```

현재 실행 중인 watcher:

```text
LaunchAgent: com.user.docsearch-mcp.watch
state: running
```

Hermes에서 사용 가능한 MCP 도구:

```text
doc-ingest
doc-ingest-status
doc-search
```

---

## 20. 추천 운영 방식

Obsidian 플러그인과 Hermes 연동에서는 다음 방식이 좋습니다.

```text
1. 일반 검색
   Hermes가 obsidian-docsearch의 doc-search 사용

2. 새 파일/수정 파일 반영
   LaunchAgent watcher가 자동 반영

3. 큰 변경 후 안정화
   수동으로 docsearch-mcp ingest files 실행

4. 정확한 파일명/문자열 검색
   keyword 모드 사용

5. 의미 기반 질문
   vector 모드 사용
```

한 줄 요약:

```text
docsearch-mcp = Obsidian Vault + PDF + 코드 파일까지 검색 가능한 로컬 RAG 인덱스
Hermes = docsearch-mcp MCP 도구를 호출해서 필요한 문맥을 가져오는 에이전트
```

---

## 21. 자주 헷갈리는 질문

### 21.1 컴퓨터를 켤 때마다 터미널에서 watch 명령을 직접 입력해야 하나?

아닙니다.

직접 입력해도 동작은 합니다.

```bash
docsearch-mcp ingest files --watch
```

하지만 이번 구성에서는 이미 LaunchAgent로 자동 실행 등록을 해두었습니다.

등록된 파일:

```text
/Users/user/Library/LaunchAgents/com.user.docsearch-mcp.watch.plist
```

그래서 Mac에 로그인하면 watcher가 자동으로 실행됩니다.

상태 확인:

```bash
launchctl print gui/$(id -u)/com.user.docsearch-mcp.watch
```

정상 상태 예:

```text
state = running
pid = 61025
last exit code = (never exited)
```

즉 평소에는 터미널을 열어서 직접 `docsearch-mcp ingest files --watch`를 입력할 필요가 없습니다.

수동으로 끄고 싶을 때:

```bash
launchctl bootout gui/$(id -u) /Users/user/Library/LaunchAgents/com.user.docsearch-mcp.watch.plist
```

수동으로 다시 켜고 싶을 때:

```bash
launchctl bootstrap gui/$(id -u) /Users/user/Library/LaunchAgents/com.user.docsearch-mcp.watch.plist
```

### 21.2 지금 `docsearch-mcp start` 서버가 백그라운드에 계속 떠 있는 건가?

아닙니다.

현재 계속 떠 있는 것은 MCP 서버가 아니라 watcher입니다.

계속 실행되는 프로세스:

```text
docsearch-mcp ingest files --watch
```

역할:

```text
파일 변화 감지
자동 인덱싱
index.db 갱신
```

반면 `docsearch-mcp start`는 Hermes가 MCP 도구를 사용할 때 stdio 방식으로 필요할 때 실행합니다.

```text
Hermes
  ↓ 필요할 때 실행
docsearch-mcp start
  ↓
index.db를 읽어서 검색 도구 제공
```

따라서 사용자가 따로 터미널에서 `docsearch-mcp start`를 계속 열어둘 필요는 없습니다.

### 21.3 mcp-loco는 daemon을 계속 켜야 했는데 docsearch-mcp는 왜 다른가?

두 도구의 구조가 다르기 때문입니다.

`mcp-loco` 구조:

```text
Hermes
  ↓ stdio mcp-loco
mcp-loco bridge
  ↓
mcp-loco daemon
  ↓
index DB / watcher / search
```

`mcp-loco`는 실제 검색 엔진이 daemon 쪽에 있어서 daemon을 계속 켜두는 구조에 가깝습니다.

`docsearch-mcp` 구조:

```text
Hermes
  ↓ stdio
docsearch-mcp start
  ↓
SQLite index.db 직접 검색
```

`docsearch-mcp`는 검색용 MCP 서버를 미리 켜둘 필요가 없습니다. Hermes가 필요할 때 실행하고, SQLite DB를 직접 읽습니다.

대신 최신 파일 상태를 유지하려면 watcher만 계속 켜두면 됩니다.

```text
docsearch-mcp ingest files --watch
  ↓
파일 변화 감지
  ↓
index.db 갱신
```

정리:

```text
mcp-loco
- daemon을 계속 켜두는 구조
- MCP 프로세스가 daemon에 의존

docsearch-mcp
- MCP 서버는 Hermes가 필요할 때 실행
- 검색 DB는 SQLite 파일
- 계속 켜둘 것은 watcher뿐
```
