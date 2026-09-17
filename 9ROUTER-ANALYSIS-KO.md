# 9Router 분석 및 활용 정리 (한국어)

> 이 문서는 9Router 코드베이스를 직접 분석해서 정리한 **한국어 요약본**입니다.
> "이게 뭔지 / 언제 쓰는지 / 나한테 어떤 도움이 되는지"를 한 번에 파악하는 것이 목적입니다.

## 📌 저장소 정보

| 항목 | 내용 |
| --- | --- |
| 이 저장소 (포크) | https://github.com/bmshin94/9router |
| 원본 저장소 (upstream) | https://github.com/decolua/9router |
| npm 패키지 (CLI 런처) | https://www.npmjs.com/package/9router |
| Docker Hub | https://hub.docker.com/r/decolua/9router |
| 공식 사이트 | https://9router.com |
| 라이선스 | MIT (Copyright 2024-2026 decolua and contributors) |
| 분석 시점 버전 | `9router-app` v0.5.75 / `cli` v0.5.75 |
| 분석 기준 커밋 | `474f623 Update CLAUDE.md` |

---

## 1. 한 줄 요약

**9Router = 내 컴퓨터에서 돌리는 "AI 통신사 라우터"**

Claude Code, Cursor, Codex 같은 AI 코딩 툴은 원래 각자 자기 회사 서버로만 연결됩니다.
9Router를 중간에 끼우면 **툴 하나로 40개 이상의 AI 제공사(100개 이상 모델)를 자유롭게 갈아끼울 수 있고**, 토큰도 20~40% 절약됩니다.

비유하자면 **배달의민족** 같은 존재입니다. 가게마다 앱을 따로 깔 필요 없이 창구 하나로 전부 주문하고, "여기 품절이면 저기로" 자동 대체까지 해줍니다.

---

## 2. 동작 방식

```
[Claude Code / Cursor / Codex / Cline ...]
            |  http://localhost:20128/v1   (엔드포인트만 이걸로 변경)
            v
  +----------------------------------+
  |        9Router (내 PC)           |
  |  - RTK 토큰 압축                  |
  |  - 포맷 변환 (OpenAI <-> Claude)  |
  |  - 쿼터 추적 / 자동 폴백           |
  |  - OAuth 토큰 자동 갱신            |
  +----------------------------------+
            |
  1순위 구독 -> 2순위 저렴 -> 3순위 무료
 (Claude Pro)  (GLM $0.6/1M)  (Kiro 무료)
```

**핵심 마법**: AI 툴 설정에서 엔드포인트를 `http://localhost:20128/v1`로 바꾸기만 하면 끝.
툴은 자기가 OpenAI와 통신한다고 생각하지만, 실제로는 9Router가 다른 제공사 포맷으로 번역해서 전달합니다.

### 요청 처리 경로 (코드 레벨)

```
src/app/api/v1/*            (next.config.mjs 의 rewrite: /v1/* -> /api/v1/*)
  -> src/sse/handlers/chat.js       (파싱, 콤보 확장, 계정 선택 루프)
  -> open-sse/handlers/chatCore.js  (포맷 감지, 번역, 실행기 디스패치, 재시도/갱신)
  -> open-sse/executors/*           (제공사별 업스트림 호출)
  -> open-sse/translator/*          (클라이언트 포맷 <-> 제공사 포맷)
  -> SSE 스트림으로 클라이언트에 응답
```

---

## 3. 폴더 구조 (실제 확인 내용)

| 폴더 | 역할 | 규모 |
| --- | --- | --- |
| `open-sse/` | **핵심 엔진.** 제공사 무관 라우팅/번역 (독립 사용 가능하게 설계됨) | - |
| `open-sse/providers/registry/` | 제공사 정의 (1파일 = 1제공사) | 약 120개 |
| `open-sse/translator/` | 포맷 번역기. OpenAI를 중간 피벗으로 사용 | - |
| `open-sse/executors/` | 비 OpenAI 호환 제공사 전용 실행기 (kiro, cursor, codex 등) | 30개 |
| `open-sse/rtk/` | **토큰 절약기** (RTK / Caveman / Ponytail / pxpipe / headroom) | - |
| `src/` | Next.js 대시보드 + API 서버 | React 19 + Next 16 |
| `src/app/(dashboard)/` | 대시보드 화면 (providers, combos, quota, usage, token-saver, cli-tools 등) | 17개 |
| `src/app/api/v1/` | OpenAI 호환 API (chat, messages, responses, images, audio, videos, embeddings, search, web) | - |
| `src/lib/db/` | SQLite 계층. 4단 폴백 드라이버 | - |
| `src/lib/mcp/` | MCP stdio <-> SSE 브리지 | - |
| `src/mitm/` | MITM 프록시 + 인증서/DNS 조작 (엔드포인트 설정이 막힌 툴 대응) | - |
| `cli/` | npm에 `9router`로 배포되는 런처 패키지 (독립 버전 관리) | - |
| `skills/` | AI 에이전트용 스킬 문서 | 9개 |
| `tests/` | vitest 테스트 (독립 ESM 패키지) | - |
| `docs/ARCHITECTURE.md` | 시스템 전체 설계 문서 | - |

---

## 4. 돈이 되는 핵심 기능 3가지

### 4-1. RTK 토큰 세이버 (기본 ON)

`git diff`, `grep`, `ls`, `tree` 같은 **툴 출력물**이 프롬프트 예산의 30~50%를 차지합니다.
RTK는 이를 LLM에 전송하기 **전에** 압축합니다.

```
RTK 없이: 47,000 토큰 전송
RTK 사용: 28,000 토큰 전송   -> 약 40% 절감, 답변 품질 동일
```

- 필터: `git-diff`, `git-status`, `grep`, `find`, `ls`, `tree`, `dedup-log`, `smart-truncate`, `read-numbered`, `search-list`
- 자동 감지: 각 `tool_result`의 첫 1KB를 확인해 적절한 필터 선택
- **Fail-open 설계**: 필터가 실패하거나 결과가 더 커지면 원본을 그대로 유지. 절대 요청을 깨뜨리지 않음
- 포맷 변환 **이전**에 동작하므로 모든 포맷(OpenAI/Claude/Gemini/Cursor/Kiro)에서 통용
- 한 요청만 우회하려면 헤더 `X-9Router-Token-Saver: off`

### 4-2. Caveman / Ponytail (출력 토큰 절감)

- **Caveman**: "원시인 말투" 프롬프트 주입 -> 답변이 간결해짐, 출력 토큰 최대 65% 절감
- **Ponytail**: "게으른 시니어 개발자" 프롬프트 -> YAGNI 스타일 최소 코드 (Lite / Full / Ultra 3단계)
  - 입력 검증, 데이터 손실 방지 에러 처리, 보안, 접근성은 절대 희생하지 않음

### 4-3. 3단계 자동 폴백 (콤보)

```
콤보 "my-coding-stack"
  1. cc/claude-opus-4-7     <- 구독을 먼저 소진
  2. glm/glm-5.1            <- 쿼터 소진 시 저렴한 제공사
  3. kr/claude-sonnet-4.5   <- 그 다음 무료 제공사
```

쿼터가 소진되거나 에러가 나면 자동 전환되어 **작업이 끊기지 않습니다.**

---

## 5. 설치 및 사용법

### 방법 A. npm 설치 (가장 쉬움)

```bash
npm install -g 9router
9router
# -> http://localhost:20128 대시보드 자동 오픈
```

`cli/` 폴더가 이 `9router` 명령어의 실체입니다. 서버 설치/실행/트레이를 담당하며,
루트 패키지(`9router-app`)와 **버전이 독립적으로 관리**됩니다.

### 방법 B. 이 저장소 소스로 실행

```bash
cp .env.example .env
npm install
PORT=20128 NEXT_PUBLIC_BASE_URL=http://localhost:20128 npm run dev
```

프로덕션 모드:

```bash
npm run build
PORT=20128 HOSTNAME=0.0.0.0 npm run start
```

> 루트 패키지는 `"private": true`라 npm 배포용이 아닙니다. 소스 실행 또는 Docker가 정식 경로입니다.

### 방법 C. Docker (VPS 배포)

저장소에 `Dockerfile`, `docker-compose.yml`, `DOCKER.md`, `captain-definition`(CapRover)이 모두 포함되어 있습니다.

```bash
docker compose up -d
```

### 실제 사용 흐름

1. 대시보드 -> **Providers** -> 사용할 AI 연결 (OAuth 로그인 또는 API 키)
2. **Combos** -> 폴백 순서 구성
3. 내 AI 툴 설정에서 엔드포인트를 `http://localhost:20128/v1`로 변경

**`cli-tools` 페이지가 특히 유용합니다.** 툴별 전용 카드가 구현되어 있어 클릭 한 번으로 설정 파일을 자동 작성해 줍니다.

```
ClaudeToolCard, CodexToolCard, ClineToolCard, CopilotToolCard,
OpenClawToolCard, OpenCodeToolCard, KiloToolCard, DroidToolCard,
CoworkToolCard, AntigravityToolCard, GrokBuildToolCard, ...
```

### 기본 URL

- 대시보드: `http://localhost:20128/dashboard`
- OpenAI 호환 API: `http://localhost:20128/v1`

---

## 6. 플러그인인가? 스킬인가? MCP인가?

### 정답: 셋 다 아니고 **"서버(게이트웨이/프록시)"** 입니다.

| 구분 | 정체 | 9Router는? |
| --- | --- | --- |
| 플러그인 | 호스트 앱 *안에* 끼워넣는 확장 | 아니오 |
| 스킬 | AI에게 주는 *설명서 문서* | 본체는 아님 (단, 9개 보유) |
| MCP | AI에게 *도구*를 제공하는 프로토콜 | 본체는 아님 (단, 브리지 내장) |
| **API 게이트웨이 / 프록시** | 요청을 받아 다른 곳으로 중계하는 서버 | **예. 이것입니다.** |

쉽게 말해:

- **MCP** = AI에게 "손"을 달아주는 것 (파일 읽기, 브라우저 조작 등의 도구)
- **스킬** = AI에게 "매뉴얼"을 주는 것 (사용법 문서)
- **9Router** = AI의 "전화선을 바꿔치기"하는 것

### 다만 스킬과 MCP 기능도 함께 품고 있음

**① 스킬 제공 (`skills/` 9개)**

```
skills/9router/SKILL.md            <- 진입점 (여기부터 읽으면 됨)
skills/9router-chat/SKILL.md       <- 채팅 / 코드 생성
skills/9router-image/SKILL.md      <- 이미지 생성
skills/9router-video/SKILL.md      <- 영상 생성
skills/9router-tts/SKILL.md        <- 텍스트 -> 음성
skills/9router-stt/SKILL.md        <- 음성 -> 텍스트
skills/9router-embeddings/SKILL.md <- 임베딩
skills/9router-web-search/SKILL.md <- 웹 검색
skills/9router-web-fetch/SKILL.md  <- URL -> 마크다운
```

Claude/Cursor 등에 이 SKILL.md 링크를 주면 "9Router로 이미지 만들어줘"가 동작합니다.

**② MCP 브리지 내장**

```
src/lib/mcp/stdioSseBridge.js              <- stdio MCP를 자식 프로세스로 실행 -> SSE 변환
src/app/api/mcp/[plugin]/sse/route.js      <- MCP SSE 엔드포인트
src/app/api/mcp/[plugin]/message/route.js  <- 클라이언트 -> 서버 JSON-RPC 수신
src/shared/constants/coworkPlugins.js      <- exa, tavily (원격 HTTP) / browsermcp (로컬 stdio)
```

MCP 응답이 너무 길면 `smartFilterText`로 자동 압축(노이즈 제거 + 반복 접기 + 절단)까지 합니다.

> **결론**: 9Router는 MCP와 경쟁 관계가 아니라 **다른 레이어**입니다.
> MCP는 "AI가 무엇을 할 수 있는가", 9Router는 "그 AI를 어디서 데려오는가". 함께 씁니다.

---

## 7. API 토큰이 필요한가?

**2층 구조**로 나눠서 봐야 합니다.

### 1층: 내 툴 -> 9Router (들어오는 방향)

`.env.example` 기준:

```bash
REQUIRE_API_KEY=false   # 기본값
```

코드에서도 확인됨 (`src/sse/handlers/chat.js:71` 등 모든 핸들러 공통):

```js
if (settings.requireApiKey) {   // 꺼져 있으면 검사 자체를 건너뜀
```

- **로컬에서만 쓸 경우: 키 불필요**
- **VPS/외부 공개 시: 반드시 `true`로 설정.** 아니면 누구나 내 AI 계정을 사용할 수 있음
- 켜면 대시보드 -> Keys 에서 `sk-...` 발급

### 2층: 9Router -> 실제 AI 제공사 (나가는 방향)

| 방식 | 예시 | 토큰 필요? |
| --- | --- | --- |
| 무인증 | OpenCode Free | 불필요 |
| **OAuth 로그인** | Claude Code, Codex, GitHub Copilot, Cursor, Kiro, Kimchi, Antigravity | 키 몰라도 됨. 버튼 클릭 로그인 -> 토큰 자동 발급/자동 갱신 |
| API 키 | OpenRouter, GLM, Kimi, DeepSeek 등 40여 개 | 해당 사이트에서 발급 후 입력 |

`src/app/api/oauth/` 아래에 kiro, cursor, codex, iflow, grok-cli, gitlab, xiaomi-mimo 전용 핸들러가 개별 구현되어 있습니다.

### 완전 무료 구성

```
9Router 자체 키 : 불필요 (로컬)
AI 제공사        : Kiro(구글/깃헙 로그인) + OpenCode Free(무인증)
총 비용          : $0, 카드 등록 0회
```

---

## 8. 왜 GitHub에서 유명한가

1. **진짜 아픈 지점을 정확히 해결** — "쿼터 소진으로 작업 중단", "구독료 내고 다 못 씀", "AI 요금 부담". 개발자 전원이 겪는 문제
2. **"무료 AI"라는 강력한 후킹** — README 첫 줄부터 `FREE AI Router`. 바이럴 소재로 최상급
3. **다국어 README 10종** — pt-BR, vi, zh-CN, ja-JP, ru, th, fa_IR, id-ID, es, fr. 영어권 밖 개발자 대거 유입. `scripts/translate-readme.js`로 자동화까지
4. **진입 장벽이 거의 0** — `npm install -g 9router && 9router` 2분이면 실행
5. **실제 완성도가 높음** — 제공사 약 120개, 실행기 30개, 대시보드 17화면, `CHANGELOG.md` 50KB(업데이트 속도), SQLite 4단 폴백으로 설치 실패 방지, 테스트/회귀 베이스라인/아키텍처 문서 완비
6. **MIT 라이선스** — 상업적 이용/수정/재배포 자유. 기업 도입 부담 없음
7. **Trendshift 등재** (`trendshift.io/repositories/22628`) — 트렌딩 진입 시 스타 눈덩이 효과
8. **커뮤니티 콘텐츠** — 베트남어/우르두어/영어 유튜브 튜토리얼이 자발적으로 생성됨

---

## 9. 로컬 에이전트 구축에 도움이 되는가 — 도움됨

### A. "부품"으로 그대로 활용 (가장 실용적)

제공사마다 다른 SDK/포맷/키 관리/재시도 처리가 한 줄로 해결됩니다.

```js
const res = await fetch("http://localhost:20128/v1/chat/completions", {
  method: "POST",
  headers: { Authorization: "Bearer sk-...", "Content-Type": "application/json" },
  body: JSON.stringify({ model: "kr/claude-sonnet-4.5", messages, stream: true }),
});
```

모델 이름만 바꾸면 **약 120개 제공사를 즉시 교체**할 수 있습니다. OpenAI SDK의 `baseURL`만 바꿔도 동작합니다.

게다가 채팅만이 아닙니다:

```
/v1/chat   /v1/messages   /v1/responses   /v1/embeddings
/v1/images /v1/audio      /v1/videos      /v1/search   /v1/web
```

**이미지 / 음성(TTS·STT) / 영상 / 임베딩 / 웹 검색까지 한 서버에서** 제공되므로 멀티모달 에이전트 부품이 통째로 갖춰집니다.

### B. 에이전트의 "생명줄"

로컬 에이전트의 최대 적은 **장시간 실행 중 쿼터 소진으로 인한 중단**입니다.
자동 폴백이 있으면 하나가 막혀도 다음으로 넘어가 계속 살아 있습니다. 자율 실행에는 사실상 필수입니다.

### C. 코드 학습 교보재로 우수

| 배울 주제 | 위치 |
| --- | --- |
| SSE 스트리밍 구현 | `open-sse/handlers/chatCore.js` |
| 포맷 번역기 설계 (OpenAI 피벗 + 직접 라우트) | `open-sse/translator/` |
| 플러그인 아키텍처 (1파일=1제공사) | `open-sse/providers/registry/` |
| 어댑터 폴백 패턴 (SQLite 4단) | `src/lib/db/driver.js` |
| OAuth PKCE + 토큰 자동 갱신 | `src/app/api/oauth/` |
| MCP stdio <-> SSE 브리지 | `src/lib/mcp/stdioSseBridge.js` |
| 프롬프트 압축 기법 | `open-sse/rtk/` |
| 보안: 소켓에서 실제 IP 추출 | `custom-server.js` |

특히 `custom-server.js`는 공격자가 조작한 `X-Forwarded-For`를 버리고 TCP 소켓에서 직접 IP를 추출하며,
루프백 리버스 프록시일 때만 포워딩 헤더를 신뢰합니다. 실무 감각이 드러나는 부분입니다.

> `open-sse/`는 독립 엔진으로 설계되어 있어 이 폴더만 떼어 다른 프로젝트에 넣는 것도 가능합니다.

---

## 10. 수익화 아이디어

### 10-0. 먼저: 코드에서 찾은 "빈 구멍" = 기회 지도

**발견 1 — 멀티유저가 존재하지 않음**

`src/lib/db/schema.js`의 전체 테이블:

```
_meta, settings, providerConnections, providerNodes, proxyPools,
apiKeys, combos, kv, usageHistory, usageDaily, requestDetails
```

`users`, `teams`, `organizations` 테이블이 **없습니다.**
`src/app/api/auth/login/route.js`도 확인:

```js
const storedHash = settings.password;          // 설정에 비밀번호 "하나"
isValid = await bcrypt.compare(password, storedHash);
```

**비밀번호 1개 = 인스턴스 1개 = 사용자 1명.** 완전한 개인용 단일 사용자 전제입니다.
SAML/OIDC(`api/auth/saml`, `api/auth/oidc`)가 있지만 "대시보드 접근 허용 여부"를 판단하는 게이트일 뿐,
로그인한 사람을 구분해 저장하지 않습니다. **기업 인증의 껍데기만 있고 멀티테넌시는 비어 있습니다.**

**발견 2 — 과금 인프라의 절반은 이미 완성**

```
usageHistory   요청별 사용 기록
usageDaily     일별 집계
requestDetails 요청/응답 상세
pricingRepo.js 모델별 단가
apiKeys        API 키 발급/관리
```

"누가 얼마나 썼는지 재는 계량기"가 이미 존재합니다. `user_id` 컬럼과 결제만 붙이면 SaaS 과금이 됩니다.

**발견 3 — 인프라 자동화도 이미 존재**

```
api/proxy-pools/cloudflare-deploy   클라우드플레어 자동 배포
api/proxy-pools/vercel-deploy       버셀 자동 배포
api/proxy-pools/deno-deploy         데노 자동 배포
api/tunnel/tailscale-*              테일스케일 터널
```

**발견 4 — 원저작자가 이미 클라우드를 운영 중**

`.env.example`의 `CLOUD_URL=https://9router.com`. 정면충돌은 피하고 다른 축(지역/업종/유통)으로 가야 합니다.

**기회 지도**

| 영역 | 현재 상태 | 기회 크기 |
| --- | --- | --- |
| 개인 로컬 사용 | 완벽 | 낮음 (무료, 원작자 영역) |
| 글로벌 클라우드 싱크 | 원작자가 운영 | 낮음 (정면충돌) |
| **팀/기업 멀티유저** | **완전히 비어 있음** | **최대** |
| **한국어/한국 제공사** | **README조차 없음** | **최대** |
| 사용량 계량 | 절반 완성 | 높음 (붙이기 쉬움) |
| 비개발자 UX | 없음 | 높음 |

---

### 10-1. (1순위) 팀용 AI 게이트웨이 — 가장 큰 시장

**문제**: 개인은 로컬로 충분하지만 회사는 못 씁니다. 비밀번호 1개로는 누가 썼는지 구분이 안 되어
예산 통제, 감사, 퇴사자 차단이 전부 불가능합니다. **개인 도구와 팀 도구 사이의 이 격차가 곧 가격 차이입니다.**

**구현할 것 (스키마 기준)**

```sql
CREATE TABLE users (id, email, name, role, team_id, created_at);
CREATE TABLE teams (id, name, monthly_budget, created_at);
ALTER TABLE usageHistory ADD COLUMN user_id;   -- 계량기에 이름표 붙이기
ALTER TABLE apiKeys      ADD COLUMN user_id;
```

그 위에:

- 멤버 초대 / 권한 (관리자 · 멤버 · 조회전용)
- 1인당 월 예산 한도 + 80% 도달 시 알림
- 감사 로그 (`requestDetails` 재활용)
- 퇴사자 즉시 차단 (키 폐기)
- 팀 대시보드 (부서별/사람별 지출)

**가격 예시**

```
스타터  5명까지   월  5만원
팀      20명까지  월 19만원
기업    무제한    월 49만원~ (온프레미스 + SAML)
```

**난이도** 중 (SAML/OIDC 기반이 이미 있어 1~2개월 MVP 가능)
**주의** 구독 계정 프록시는 이 상품에서 제외하고 정식 API 키 제공사만 라우팅할 것 (10-6 참고)

---

### 10-2. (2순위) 한국형 특화 버전 — 가장 빨리 시작 가능

README 번역이 10개국어 있는데 **한국어가 없습니다.** 아무도 잡지 않은 자리입니다.

**1단계: 한국어 기여 (비용 0원, 약 2주)**

```
- README.ko-KR.md 번역 후 upstream에 PR
- 대시보드 UI 한국어화 (src/i18n/ 폴더 활용)
- 한국 상황에 맞춘 셋업 가이드 (블로그/유튜브)
```

얻는 것: 기여자 등재 + "한국 9Router 담당자" 포지션. 이후 모든 수익화의 신뢰 자산이 됩니다.

**2단계: 한국 제공사 연동 (약 1개월)**

`open-sse/providers/registry/`에 파일 하나씩 추가하면 됩니다 (`providers/REGISTRY_TEMPLATE.js` 존재).

```
naver-clova.js    네이버 하이퍼클로바X
upstage-solar.js  업스테이지 Solar
kakao.js          카카오
```

> `providers/registry/index.js`는 자동 생성 파일이므로 손으로 고치지 말고
> `scripts/migrate-registry.mjs` / `injectDisplayToRegistry.mjs`로 재생성해야 합니다.

**3단계: 유료 서비스 (3개월~)**

```
"나인라우터 코리아" — 월 9,900원
+ 완전 한국어 대시보드
+ 원화 결제 (토스페이먼츠 / 포트원)
+ 카톡 알림봇 (쿼터 소진, 예산 초과)
+ 한국 제공사 기본 탑재
+ 카톡 오픈채팅 실시간 지원
```

한국 개발자 상당수가 "영어 README + 디스코드 지원"에서 이탈합니다. 그 장벽 제거만으로 상품이 됩니다.
그리고 지역이 다르므로 **원작자와 경쟁하지 않습니다.**

---

### 10-3. (3순위) AI 비용 절감 컨설팅 — 서버 없이 즉시 현금화

```
진단 (무료)        현재 AI 지출 분석 -> 절감 가능액 리포트
구축 (150~400만원)  9Router 세팅 + 콤보 설계 + 팀 온보딩 + 교육
운영 (월 30~80만원) 모니터링, 제공사 정책 변동 대응, 콤보 튜닝
                   또는 절감액의 20% 성과 기반
```

세일즈 멘트: **"귀사 AI 요금, 30% 못 줄이면 비용 받지 않습니다."**
RTK(입력 20~40%) + Caveman(출력 최대 65%) + 콤보 폴백이면 30%는 보수적인 숫자입니다.

**타겟**: AI 도입 후 요금 부담을 겪는 스타트업 / 개발자 10~50명 규모 SI·에이전시 / AI 서비스 운영사

**숨은 이점**: 컨설팅 과정에서 고객의 실제 니즈를 파악하게 되고, 그것이 10-1(팀 상품)의 스펙이 됩니다.
**컨설팅 -> 제품화** 루트가 가장 안전합니다.

---

### 10-4. (4순위) 비개발자용 "AI 요금 반값" 앱

현재 9Router는 개발자용입니다(`npm install -g`, 터미널, 엔드포인트 설정). 일반인은 첫 화면에서 이탈합니다.
그런데 비개발자도 AI 구독을 3~4개씩 하며 월 8~10만원을 지출하고 관리는 못 합니다.

```
설치 파일 더블클릭 (터미널 없음)
설정 마법사 — "어떤 AI 쓰세요?" 클릭만
대시보드 대신 "이번 달 15,400원 절약!" 한 장 화면
알림 — "무료 한도 80% 사용"
```

기존 기능을 숨기고 UI만 새로 작성하면 됩니다. 백엔드는 그대로, React 19이므로 UI 교체가 쉽습니다.

```
무료: 절감액 월 1만원까지
프로: 월 4,900원 — 무제한 + 알림 + 여러 기기
```

시장은 개발자보다 훨씬 크지만 UX 설계가 실제 작업량입니다.

---

### 10-5. (5순위) 콘텐츠 + 제휴 — 리스크 0, 오늘 시작 가능

README에 걸린 유튜브 영상은 베트남어/우르두어/영어뿐이고 **한국어가 없습니다.**

```
수익 경로
1. 유튜브/블로그 광고 수익
2. AI 제공사 리퍼럴 (OpenRouter 등)
3. 세팅 대행 문의 유입 -> 10-3으로 연결
4. 유료 강의 (인프런/클래스101)

콘텐츠 기획
1편 "AI 요금 월 8만원 -> 0원 만들기"   (후킹)
2편 "Claude Code 무료로 쓰는 법"
3편 "회사 AI 비용 통제하는 법"          (B2B 리드 수집)
4편 "9Router 코드 뜯어보기"             (개발자 신뢰)
```

---

### 10-6. 반드시 알아야 할 리스크 3가지

**① 구독 계정 프록시 = 가장 큰 법적 리스크**

| 상황 | 위험도 |
| --- | --- |
| 내 구독을 내 컴퓨터에서 개인 사용 | 회색지대 (약관 위반 소지, 계정 정지 가능) |
| **그것을 상품으로 판매** | **위험** (약관 위반 + 사업 리스크) |
| 남의 구독을 공유/재판매 | **절대 금지** |

대부분 제공사 약관은 구독 계정의 API/프록시 우회 이용을 금지합니다.
개인이 조용히 쓰는 것과 **돈을 받고 파는 것은 법적 무게가 완전히 다릅니다.**

```
판매 금지: 구독 우회, 무료 티어 재판매, 계정 공유
판매 가능: 소프트웨어/서비스(관리·대시보드·팀 기능)
          컨설팅/구축/운영 (사람의 노동)
          정식 API 키 제공사 라우팅 (고객이 자기 키 사용 = BYOK)
          토큰 절감 기술 (RTK/Caveman — 순수 기술이므로 문제 없음)
```

**핵심 전환**: "무료 AI 쓰게 해드립니다"(위험) -> **"AI 비용을 관리하고 줄여드립니다"**(안전하고 더 비싸게 팔림)

**② 무료 티어는 반드시 사라짐** — README가 직접 증언합니다.

```
iFlow      무료 무제한 -> 유료 전환 (2026)
Qwen Code  무료 OAuth 완전 종료 (2026-04-15)
Gemini CLI 구글 서비스 종료 (2026-06-18)
Kiro       무제한 -> 월 50크레딧으로 축소
```

"무료"를 상품 핵심 가치로 걸면 제공사 정책 한 줄에 사업이 무너집니다.
팔아야 할 것은 **안정성 · 관리 · 절감 기술**입니다.

**③ MIT라 누구나 동일하게 가능** — 코드 자체는 해자가 아닙니다.

```
진짜 해자
- 한국어 + 한국 제공사 + 한국 결제 (지역 장벽)
- 고객 관계 / 운영 노하우
- 브랜드
- 팀 데이터 축적 (전환 비용)
```

MIT 조건상 **원저작권 표시(`decolua and contributors`)는 반드시 유지**해야 합니다.

---

### 10-7. 추천 90일 플랜

| 기간 | 할 일 | 예상 수익 |
| --- | --- | --- |
| 1~30일 | README.ko-KR.md PR, 대시보드 한국어화 PR, 유튜브 1편, 블로그 3편 | 거의 0 (자산 축적) |
| 31~60일 | 무료 진단 템플릿, 스타트업 5~10곳 아웃리치, 1~2곳 구축 수주, 한국 제공사 1개 연동 | 200~800만원 |
| 61~90일 | users/teams 스키마, 멤버 초대·예산·감사 로그, 컨설팅 고객을 구독 전환, 결제 연동 | 구독 MRR 시작 |

```
흔한 실패: 제품부터 3개월 만들고 -> 고객 0명
이 순서  : 콘텐츠로 신뢰 -> 컨설팅으로 매출+니즈 확보 -> 제품화
           (각 단계가 다음 단계의 연료)
```

**요약표**

| 아이디어 | 난이도 | 수익 규모 | 시작 시점 | 리스크 |
| --- | --- | --- | --- | --- |
| 콘텐츠 + 제휴 | 하 | 낮~중 | 오늘 | 없음 |
| 컨설팅 | 하~중 | 높음 | 1개월 | 낮음 |
| 한국 특화 | 하~중 | 중~높 | 오늘(기여부터) | 낮음 |
| 비개발자 앱 | 중 | 중~높 | 3개월 | 중간 |
| **팀/기업 상품** | 중 | **최고** | 2~3개월 | 중간 |
| 구독 재판매 | - | - | **하지 말 것** | **높음** |

---

## 11. React나 PHP로 만들 수 있는가

### React — 이미 React입니다

```json
"react": "19.2.4",  "next": "^16.1.6",  "tailwindcss": "^4"
```

대시보드 전체가 React 19 + Next.js 16 + Tailwind 4입니다. 추가로:

- `zustand` — 상태 관리
- `recharts` — 사용량 차트
- `@xyflow/react` — 콤보 플로우 다이어그램 (노드 연결 UI)
- `@dnd-kit` — 드래그앤드롭 (폴백 순서 변경)
- `@monaco-editor/react` — VSCode 엔진 기반 에디터

React를 할 줄 알면 **바로 기여/개조가 가능**합니다.

### PHP — 가능하지만 권장하지 않음

| 항목 | Node.js (현재) | PHP |
| --- | --- | --- |
| SSE 스트리밍 | 네이티브, 쉬움 | `ob_flush()` + 웹서버 버퍼링 대응 필요 |
| 장시간 연결 | 문제 없음 | `max_execution_time` 제약 (Swoole/ReactPHP 필요) |
| 동시 처리 | 이벤트 루프 기본 | FPM은 요청당 프로세스 (무거움) |
| 번역기/제공사 | 이미 완성 | **약 120개 전부 재작성** |
| 생태계 | AI SDK 풍부 | 상대적으로 빈약 |

진짜 문제는 SSE가 아니라 **"제공사 120개 + 번역기 + 실행기 30개를 처음부터 다시 작성"**입니다.
그것이 이 프로젝트 가치의 대부분이며, 몇 달치 작업입니다.

### 권장안

**안 1 (강력 추천): PHP는 껍데기, 9Router는 엔진**

```
[PHP 앱 (내 서비스)] --HTTP--> [9Router (그대로 사용)] --> AI 제공사들
```

```php
$ch = curl_init('http://localhost:20128/v1/chat/completions');
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_HTTPHEADER, [
    'Content-Type: application/json',
    'Authorization: Bearer sk-...',
]);
curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode([
    'model' => 'kr/claude-sonnet-4.5',
    'messages' => $messages,
]));
```

재작성 0줄. 9Router는 별도 프로세스로 두고 PHP는 로그인·결제·회원관리 등 비즈니스 로직만 담당합니다.
라라벨/워드프레스 연동도 쉽습니다.

**안 2: React로 UI만 새로 작성** — 백엔드(`open-sse/`, `src/app/api/`)는 유지하고 대시보드만 교체.
한국어 UI + 초보자용 마법사 화면을 만들면 그 자체로 차별점이 됩니다.

**안 3: PHP 풀 포팅** — 학습 목적이 아니라면 비권장. 투입 시간 대비 효용이 낮습니다.

---

## 12. 주의사항 / 함정

- **보안 환경변수**: `JWT_SECRET`, `INITIAL_PASSWORD`(기본값 `123456` — 반드시 변경), `API_KEY_SECRET`, `MACHINE_ID_SALT`.
  전체 계약은 `.env.example` 및 `docs/ARCHITECTURE.md`의 env 매트릭스 참고
- **외부 공개 시** `REQUIRE_API_KEY=true` 필수
- **`src/mitm/`** 은 시스템 인증서 설치 + DNS 조작까지 수행합니다(관리자 권한 필요).
  엔드포인트 설정이 막힌 툴 대응용이므로, 사용하지 않을 거면 건드리지 마세요
- **테스트는 깨끗한 체크아웃에서도 전부 통과하지 않습니다** (약 938 통과 / 64 실패가 정상).
  회귀 판단은 `tests/__baseline__/verify-no-regression.mjs`로 해야 합니다
- **대시보드의 비용 표시는 실제 청구액이 아닙니다.** "유료 API로 직접 썼다면 이만큼 나왔을 것"이라는
  절감 추정치입니다
- **영속성 계층**: `docs/ARCHITECTURE.md`는 이 부분이 오래된 내용(`db.json`)입니다.
  실제로는 `src/lib/db/` SQLite 계층 (`bun:sqlite` -> `better-sqlite3` -> `node:sqlite` -> `sql.js` 폴백).
  `src/lib/localDb.js`는 하위 호환 shim이며, 새 코드는 `@/lib/db/index.js`에서 import해야 합니다
- **번역기 등록**: 새 translator 파일은 `open-sse/translator/index.js`에서 import하지 않으면 동작하지 않습니다
- **`open-sse/` 수정 전** 반드시 `open-sse/AGENTS.md`를 읽으세요

---

## 13. 최종 요약

| 질문 | 답 |
| --- | --- |
| 설치법 | `npm i -g 9router && 9router` |
| 플러그인/스킬/MCP? | 전부 아님. **서버(게이트웨이)**. 단 스킬 9개 + MCP 브리지 내장 |
| API 토큰 필요? | 로컬은 불필요, OAuth 제공사는 로그인만, 외부 공개 시 필수 |
| 왜 유명? | 실제 문제 해결 + 무료 후킹 + 다국어 + 2분 설치 + 높은 완성도 + MIT |
| 에이전트에 도움? | 부품으로도, 생명줄로도, 학습 교보재로도 매우 유용 |
| 수익화? | 한국 특화 > 컨설팅 > 팀/기업 상품 (구독 프록시 판매는 금지) |
| React / PHP? | React는 이미 사용 중. PHP는 감싸서 호출하는 방식 권장 |

---

## 참고 문서

- `docs/ARCHITECTURE.md` — 시스템 전체 설계 (영속성 섹션은 오래됨)
- `open-sse/AGENTS.md` — 라우팅/번역 엔진 규약, 제공사·실행기·번역기 추가 방법
- `CLAUDE.md` — 이 저장소에서 작업할 때의 지침
- `skills/README.md` — 에이전트용 스킬 인덱스
- `README.md` — 원본 전체 문서 (영어)
