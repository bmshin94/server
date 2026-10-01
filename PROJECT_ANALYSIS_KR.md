# Music Assistant Server 전수조사 & 활용 가이드 (한국어 정리본)

> 이 문서는 Claude Code 세션에서 진행한 저장소 전수조사 대화를 정리한 결과물입니다.
> 작성일: 2026-10-01

## 0. 저장소 정보

| 항목 | 값 |
| --- | --- |
| 내 저장소 (fork) | https://github.com/bmshin94/server |
| 원본 (upstream) | https://github.com/music-assistant/server |
| 공식 문서 | https://music-assistant.io |
| 베타 문서 | https://beta.music-assistant.io |
| 이슈 트래커 | https://github.com/music-assistant/support/issues |
| 사용 정책 | https://github.com/music-assistant/.github/blob/main/USAGE_POLICY.md |
| 라이선스 | Apache-2.0 (상업적 사용 허용, 상표는 별도) |
| 소속 재단 | Open Home Foundation |
| 주 개발 브랜치 | `dev` (운영 릴리스는 `stable`) |
| 작업 브랜치 | `claude/eager-lovelace-haubxc` |

## 1. 이게 뭐하는 프로젝트인가

Music Assistant Server는 **"집 전체를 하나의 음악 시스템으로 묶어주는 셀프호스팅 음악 서버"** 입니다.

- 여러 음악 소스(스트리밍 서비스 + 내 NAS의 음원 파일)를 **하나의 통합 라이브러리**로 합칩니다.
- 그 음악을 집 안의 **서로 다른 브랜드 스피커**(Sonos, Chromecast, AirPlay, DLNA, Squeezelite, WiiM, Bluesound 등)로 동시에 또는 개별적으로 재생합니다.
- Home Assistant와 결합해 **자동화**("퇴근하면 거실에 재즈")가 가능합니다.
- 라즈베리파이 / NAS / Intel NUC 같은 **상시 구동 기기**에서 24시간 돌리는 형태를 전제로 설계되어 있습니다.

핵심은 "스트리밍 서비스 + 스피커 + 자동화"를 잇는 **중개 서버(브로커)** 라는 점입니다. 음악을 소유·저장하는 도구가 아니고, **내 스피커로 재생하는 것**이 목적입니다.

## 2. 규모 (실측치)

| 항목 | 수치 |
| --- | --- |
| Python 파일 (패키지 내부) | 905개 |
| 전체 Python 코드 라인 | 약 673,000줄 (테스트 포함) |
| 프로바이더(연동 모듈) | 130개 |
| 코어 컨트롤러 | 13개 |
| 공용 헬퍼 모듈 | 약 50개 |
| 번역 언어 | 32개 (`ko_KR.json` 포함) |
| GitHub Actions 워크플로 | 18개 |
| 커스텀 린트/검증 스크립트 | 20개 이상 |
| 런타임 요구사항 | Python 3.14+, ffmpeg 7.1+ |

## 3. 폴더 구조 해부

```
server/
├── music_assistant/            # 본체 패키지
│   ├── mass.py                 # (72KB) 전체 오케스트레이터 - 모든 컨트롤러를 띄우는 심장
│   ├── __main__.py             # 실행 엔트리포인트
│   ├── constants.py            # (38KB) 설정 키/상수 전체
│   ├── strings.json            # (83KB) UI 문자열 원본
│   ├── controllers/            # 코어 기능 13개
│   ├── providers/              # 외부 연동 모듈 130개
│   ├── models/                 # 프로바이더 베이스 클래스 (상속용 계약서)
│   ├── helpers/                # 재사용 유틸 (ffmpeg, dsp, oauth, jwt, cache...)
│   └── translations/           # 32개 언어 번역본
├── tests/                      # pytest 테스트 + 벤치마크
├── scripts/                    # 자체 제작 린터 / 성능측정 / 코드생성기
├── .claude/skills/review-pr/   # Claude Code 전용 "스킬"
├── .github/                    # CI 워크플로 18개 + AI 에이전트용 지침서
├── CLAUDE.md / AGENTS.md / GEMINI.md  # AI 에이전트용 규칙 문서
├── DEVELOPMENT.md              # 프로바이더 제작 가이드
├── Dockerfile / Dockerfile.base
└── pyproject.toml / requirements_all.txt
```

### 3-1. `controllers/` — 서버의 기능 블록 (13개)

| 컨트롤러 | 역할 |
| --- | --- |
| `music` | 라이브러리 DB, 동기화, 중복 병합, 추천, **DB 마이그레이션** |
| `players` | 스피커 추상화 (전원/볼륨/그룹) |
| `player_queues` | 재생 대기열, 셔플, 반복, crossfade |
| `streams` | 오디오 스트리밍 파이프라인 (ffmpeg, HLS, Ogg, 스마트 페이드, 공지 음성) |
| `webserver` | HTTP + WebSocket API, 인증, 프론트엔드 호스팅, WebRTC 원격접속 |
| `metadata` | 앨범아트 / 가사 / 아티스트 정보 수집 |
| `cache` | API 호출 절약용 캐시 계층 |
| `config` | 사용자 설정 저장/검증 |
| `discovery` | mDNS/zeroconf 기기 자동 탐색 |
| `tasks` | 백그라운드 작업 스케줄러 |
| `dashboard` / `diagnostics` / `translations` | 상태판, 진단 리포트, 번역 로딩 |

### 3-2. `providers/` — 130개 연동 모듈 (이 프로젝트의 진짜 자산)

타입별 분포:

| 타입 | 개수 | 하는 일 | 예시 |
| --- | --- | --- | --- |
| `music` | 61 | 음원 소스 | spotify, apple_music, tidal, qobuz, deezer, ytmusic, plex, jellyfin, emby, subsonic, soundcloud, bandcamp, audible, filesystem_local/smb/nfs/google_drive/onedrive, radiobrowser, podcast_index, neteasecloudmusic, yandex_music, qqmusic, kion_music, zvuk_music |
| `player` | 28 | 스피커/출력 | sonos, sonos_s1, chromecast, airplay, dlna, squeezelite, snapcast, heos, bluesound, wiim, musiccast, bose_soundtouch, mpd, amplipi, roku, samsung_wam, yandex_station, sync_group, universal_group |
| `plugin` | 26 | 부가 기능 | **fastmcp_server**, **ai_radio**, **openai_compatible**, openai_tts, music_quiz, sonic_similarity, smart_fades, smart_playlist, spotify_connect, alexa, hass, profiler, milkdrop_visualizer |
| `metadata` | 10 | 아트/가사/정보 | musicbrainz, fanarttv, theaudiodb, coverartarchive, genius_lyrics, lrclib, wikipedia, itunes_artwork |
| `audio_analysis` | 5 | 음원 분석 | sonic_analysis(CLAP 임베딩), loudness_analysis, acoustid_lookup |

성숙도: stable 76 / beta 24 / alpha 10 / experimental 7 / deprecated 1 / unmaintained 2

`_demo_music_provider`, `_demo_player_provider`, `_demo_plugin_provider`, `_demo_audio_analysis_provider` 는 **주석이 가득한 복붙용 템플릿**입니다. 새 프로바이더를 만들 때 여기서 시작하면 됩니다.

### 3-3. `.claude/skills/review-pr/` — Claude Code 스킬

- `SKILL.md` — PR 리뷰 절차 (worktree 분리 → 체크아웃 → 해시 검증 → diff 분석 → 표준 적용)
- `REVIEW_STANDARDS.md` — 리뷰 기준
- `references/new-provider.md` — 신규 프로바이더 리뷰 체크리스트
- **읽기 전용 리뷰**이고 GitHub에 자동 댓글/승인 금지가 명시되어 있습니다.

### 3-4. AI 에이전트 지침 3종 (이 저장소의 특이점)

| 파일 | 내용 |
| --- | --- |
| `CLAUDE.md` | 업스트림 규칙 + **"카리나" 페르소나**(내가 직접 추가한 부분) |
| `AGENTS.md` | 업스트림 규칙 원본 (Codex 등 범용 에이전트용) |
| `GEMINI.md` | 카리나 페르소나만 (Gemini CLI용) |
| `.github/copilot-instructions.md` | GitHub Copilot용 지침 |
| `.github/instructions/*.instructions.md` | 코드 표준 / PR 설명 / 크로스레포 규칙 3종 |

> 내가 직접 커밋한 변경은 `CLAUDE.md` + `GEMINI.md`에 카리나 페르소나를 붙인 것이고, 코드는 업스트림 `dev`와 동일합니다.

### 3-5. `scripts/` — 자체 제작 품질 게이트

범용 린터로는 못 잡는 이 프로젝트만의 규칙을 직접 코드로 검사합니다.

- `check_blocking_io.py` — async 함수 안에서 동기 I/O 쓰는지 (async 서버의 1급 버그)
- `check_method_order.py` — public 메서드 위 / private 메서드 아래 규칙
- `check_manifests.py` / `check_provider_scope.py` / `check_provider_icons.py`
- `check_datetime_helpers.py` — naive datetime 사용 금지
- `check_config_entries.py` / `check_translatable_labels.py`
- `lint_baseline.py` + `lint_baselines/*.txt` — 레거시 위반은 기준선으로 동결, 신규만 차단
- `perf/` — 벤치마크 서버, WS 클라이언트, 리포트 생성
- `tidal_openapi/generate_models.py` — OpenAPI 스펙에서 모델 코드 자동 생성

## 4. 반드시 지켜야 하는 사용 정책 (매우 중요)

CLAUDE.md와 DEVELOPMENT.md에 공통으로 박혀 있는 레드라인입니다.

금지:
1. 음악 서비스의 **원본 오디오 URL을 서버 외부로 노출**
2. 내 계정 권한을 넘어선 **보호 오디오 디코딩** 또는 디코딩 결과 보관
3. **스로틀링 없는 API 호출**, 캐시가 있는데도 재요청
4. 스트리밍 프로바이더 오디오를 **디스크에 기록**
5. 구독 등급 / 지역 제한 / 동시 스트림 제한(`max_concurrent_streams`) 우회
6. **다운로드 / 내보내기 / 아카이빙** 기능 추가 (어떤 명분이든)

제거하면 안 되는 가드 (지워도 테스트는 통과하므로 특히 주의):
- 스트림 엔드포인트의 **readrate 페이싱**
- 백그라운드 오디오 분석의 **파일시스템 전용 제한**

추가 규칙:
- `helpers/app_vars.py` 의 번들 API 키는 **프로젝트 공용 자산**. 추출/전용 금지 (레이트리밋 공유 → 남용 시 전체 사용자 피해).
- 저장 데이터(설정 키, DB 컬럼)를 건드리면 **반드시 마이그레이션**. 멱등하게, 예외 없이. 실패하면 사용자 라이브러리 DB가 리셋됩니다.

## 5. 설치 및 사용법

### 5-1. 공식 지원 설치 (2가지만)

**① Home Assistant 애드온 (권장)**
- https://music-assistant.io/installation/
- HA 설정 → 애드온 스토어 → 저장소 추가 → Music Assistant 설치 → 시작
- ffmpeg, jemalloc, CIFS/NFS 클라이언트 등 OS 의존성이 전부 번들됩니다.

**② Docker**
```bash
docker run -d --name music-assistant \
  --network host \
  -v /path/to/data:/data \
  ghcr.io/music-assistant/server:latest
```
- `--network host` 는 mDNS 기기 탐색 때문에 필수에 가깝습니다.

> **PyPI 배포 없음.** `pip install music_assistant` 안 됩니다. Python 코드지만 ffmpeg + 네이티브 라이브러리 + 번들 바이너리에 의존하기 때문입니다.

### 5-2. 소스 개발 환경

```bash
git clone https://github.com/bmshin94/server
cd server
scripts/setup.sh                              # venv + 의존성 + pre-commit 훅
python -m music_assistant --log-level debug   # http://localhost:8095
```

```bash
pytest                                  # 전체 테스트
pytest -n auto --dist loadfile          # 병렬
pytest --cov music_assistant            # 커버리지
pre-commit run --all-files              # 린트 전체 (코드 수정 후 필수)
```

데이터 위치: `$HOME/.musicassistant/`
- 로그: `musicassistant.log` (+ `.log.1`, `.log.2` 로테이션)
- DB: `library.db` (sqlite3) — **SELECT만. 라이브 DB에 쓰기 금지**

### 5-3. 사용 흐름

1. `http://<서버IP>:8095` 접속 → 초기 설정(`/setup`)에서 관리자 계정 생성
2. Settings → Music Providers → Spotify/Tidal/NAS 등 추가
3. Settings → Players → 스피커 자동 탐색 확인 (mDNS)
4. 라이브러리 동기화 대기 → 재생
5. (선택) Home Assistant 통합 → 자동화 연결

### 5-4. API

| 엔드포인트 | 설명 |
| --- | --- |
| `/ws` | WebSocket 실시간 양방향 API (프론트엔드가 쓰는 메인 채널) |
| `/api` | HTTP JSON-RPC (단발 요청/응답) |
| `/api-docs` | 자동 생성 API 문서 |
| `/api-docs/openapi.json` | OpenAPI 스펙 |
| `/api-docs/swagger` | Swagger UI |
| `/auth/login`, `/auth/me`, `/auth/providers` | 인증 |
| `/mcp/v1` | MCP 서버 (fastmcp_server 플러그인 활성화 시) |

인증 구조:
- 내장 Provider(username/password, bcrypt + 사용자별·서버별 salt) + Home Assistant OAuth2
- **단기 토큰**: 사용 시 자동 갱신, 30일 슬라이딩 만료 (사용자 세션용)
- **장기 토큰**: 갱신 없음, 10년 만료 (**외부 연동/API 접근용**)
- 로그인 레이트리밋(점진 지연), 토큰 폐기 시 WebSocket 즉시 끊김
- 원격 접속은 WebRTC 기반 (`controllers/webserver/remote_access`)

## 6. 플러그인? 스킬? MCP? — 정답: 전부 다 들어있는 "서버 앱"

| 질문 | 답 |
| --- | --- |
| 이것 자체는? | **독립 실행형 서버 애플리케이션** (Python 패키지 + Docker 이미지) |
| 플러그인인가? | 이 자체는 아니지만, **자체 플러그인 시스템을 가진 호스트**. `type: "plugin"` 프로바이더 26개 |
| 스킬인가? | 저장소 안에 **Claude Code 스킬 1개**(`.claude/skills/review-pr`)가 개발 보조용으로 들어있음. 제품 기능은 아님 |
| MCP인가? | **MCP 서버를 제공할 수 있음**. `providers/fastmcp_server` 플러그인 |

### 6-1. `fastmcp_server` 상세 (AI 연동의 핵심)

- 설명: *"Exposes Music Assistant as a Model Context Protocol (MCP) server for Claude, Codex, and other MCP-aware LLM clients."*
- 런타임: PrefectHQ **FastMCP v3** (`fastmcp==3.4.7`)
- 마운트: MA의 기존 aiohttp 웹서버 **`/mcp/v1`** 에 ASGI 브리지로 장착
  → 별도 uvicorn 없음, 추가 포트 없음, MA 코어 수정 없음
- 단계: `experimental`, `multi_instance: false`
- 노출 도구 모듈 10개: `library`, `media`, `metadata`, `playback`, `players`, `playlists`, `queue`, `volume`, `config`, `debug`
- 인증: 자체 JWT 디코딩을 구현하지 않고 **MA의 `mass.webserver.auth.authenticate_with_token` 에 위임** (단일 진실 공급원 유지)
- 주요 설정: `require_auth`, `mount_path`, `require_confirmation`(쓰기 작업 확인), `enforce_audience`(JWT `aud` 검증), `extra_allowed_origins`, `trust_forwarded_proto`, `meta_tool_discovery`, `lean_admin_schema`
- RFC 9728 준수: 401 응답의 `WWW-Authenticate` 헤더에 `resource_metadata` 채움
- `connect/` 디렉터리 — 클라이언트 연결 온보딩 페이지/핸들러/토큰 폐기

즉, Claude Desktop / Claude Code 에 `/mcp/v1` 을 붙이면 **"거실에 90년대 시티팝 틀어줘"** 를 자연어로 할 수 있습니다.

## 7. API 토큰이 필요한가? — 3개 층으로 나눠 보면 명확

| 층 | 토큰 필요? | 내용 |
| --- | --- | --- |
| **① MA 서버 자체** | 로컬 UI는 로그인만. 외부 연동은 **장기 토큰(10년) 발급** | Settings에서 발급. `/api`, `/ws`, `/mcp/v1` 호출 시 Bearer |
| **② 음악 프로바이더** | 대부분 **불필요** | Spotify/Tidal 등은 `helpers/app_vars.py` 의 **번들 공용 키** + 내 계정 로그인. 별도 발급 없이 바로 동작 |
| **③ AI 기능** | **내 키 필요 (또는 로컬 모델로 0원)** | `openai_compatible` 플러그인에 OpenAI / Groq / OpenRouter / Together 키 입력. **Ollama / LM Studio 로컬 서버면 무료** |

보너스: `local_audio`, `filesystem_*`, `radiobrowser`, `musicbrainz`, `lrclib` 등은 토큰 없이 완전 무료로 동작합니다. 토큰 0개로도 "NAS 음원 + 인터넷 라디오 + 가사 + 앨범아트" 구성이 가능합니다.

## 8. AI 에이전트 구축에 도움이 되는가? — 매우 된다

이 저장소는 **실전 투입된 MCP 서버 + LLM 추상화 레이어의 레퍼런스**입니다. 튜토리얼 코드가 아니라 수천 명이 쓰는 프로덕션 코드라는 점이 핵심 가치입니다.

### 8-1. 그대로 배워 쓸 수 있는 패턴 6가지

1. **MCP 서버를 기존 웹서버에 얹는 법** (`fastmcp_server/http_bridge.py`)
   별도 프로세스 없이 ASGI 브리지로 기존 aiohttp 앱의 서브패스에 마운트.
2. **MCP 인증을 기존 인증 시스템에 위임** (`fastmcp_server/auth.py`)
   JWT를 두 번 구현하지 않는다. `aud` 클레임만 읽어 리소스 검증. RFC 9728 대응.
3. **LLM 프로바이더 추상화** (`providers/openai_compatible`)
   OpenAI / Groq / OpenRouter / Together / Ollama / LM Studio를 하나의 인터페이스로. 벤더 락인 회피 설계의 교본.
4. **도구(tool) 설계 분할** (`fastmcp_server/tools/*.py`)
   도메인별 파일 분리 + `_common.py` 공통화 + `meta_discovery.py`로 도구 수가 많을 때의 **메타 디스커버리**(도구 폭발 문제 해결). 에이전트 설계에서 가장 실용적인 부분.
5. **안전장치**: `require_confirmation` (쓰기 작업 사용자 확인), `origins.py` (CORS 화이트리스트), `middleware.py`, `_revoke.py` (토큰 폐기)
   → 에이전트에게 권한을 주되 폭주를 막는 구조.
6. **벡터 검색 / RAG 유사 구조** (`providers/sonic_similarity`)
   Microsoft **CLAP** 오디오 임베딩 + **usearch** 벡터 인덱스 + `transformers` + HuggingFace Hub.
   `scripts/precompute_clap_prompt_embeddings.py` 로 프롬프트 임베딩 사전계산 → 임베딩 파이프라인 전체를 볼 수 있습니다.

### 8-2. 에이전틱 기능 사례

- `ai_radio` (alpha) — **AI DJ**. 플레이리스트에 AI 진행자 코멘트 섹션 생성 + 동적 라디오 큐 생성. LLM(텍스트) → TTS(음성) → 오디오 파이프라인 삽입까지 엔드투엔드 체인.
- `openai_tts` — OpenAI Speech API 호환. Kokoro-FastAPI / LocalAI / Speaches 등 셀프호스팅 가능.
- `music_quiz` — QR 참여형 멀티플레이어 퀴즈. 게임 상태 머신 + 게스트 접근(`helpers/guest_access.py`).
- `smart_playlist`, `recommendations`, `smart_fades`, `sonic_analysis` — 규칙 기반 + ML 기반 추천 혼합.

### 8-3. "에이전트 친화적 저장소"를 만드는 법 그 자체

`CLAUDE.md` / `AGENTS.md` / `GEMINI.md` / `.github/copilot-instructions.md` / `.github/instructions/*.instructions.md` / `.claude/skills/` 구성은 **AI 에이전트용 컨텍스트 설계의 모범 사례**입니다. 특히:
- 레드라인을 "왜 존재하는지"까지 써둠 (가드를 지우지 않게)
- 데이터 마이그레이션은 **AskUserQuestion 팝업으로 반드시 사람에게 물어라**는 명시적 에스컬레이션 규칙
- 커스텀 린트 스크립트로 AI가 어기기 쉬운 규칙을 **기계적으로 차단**

내 프로젝트에 그대로 이식할 수 있는 패턴입니다.

## 9. React나 PHP로 만들 수 있나?

### 9-1. React — ✅ 프론트엔드/클라이언트는 완전 가능 (추천)

- 현재 공식 프론트엔드는 **Vue 기반 PWA** (`music-assistant-frontend==2.17.312`, npm 패키지로 분리)
- 서버와 프론트는 **`/ws` WebSocket + `/api` JSON-RPC로만 통신** → 완전히 교체 가능한 구조
- `/api-docs/openapi.json` 이 있으므로 **타입 자동 생성** 가능 (`openapi-typescript` 등)
- 추천 스택: React + TypeScript + TanStack Query + 네이티브 WebSocket (또는 React Native로 모바일 리모컨)
- 인증: 장기 토큰 Bearer → 바로 붙습니다.

**결론: "React 음악 리모컨 앱" 은 지금 당장 시작 가능한 현실적인 프로젝트입니다.**

### 9-2. PHP — ⚠️ 조건부

| 할 수 있는 것 | 할 수 없는(비현실적인) 것 |
| --- | --- |
| `/api` JSON-RPC 호출 (cURL/Guzzle) | 서버 코어 재작성 |
| 웹 대시보드 / 관리자 패널 | 오디오 스트리밍 파이프라인 (ffmpeg + 실시간 버퍼링) |
| Laravel로 멀티매장 중앙관리 SaaS | 130개 프로바이더 포팅 |
| 예약 재생 / 리포트 / 결제 연동 | mDNS 기기 탐색, WebRTC 원격접속 |
| Webhook 수신 → MA 제어 | 상시 WebSocket 연결 유지 (PHP-FPM 모델과 안 맞음) |

PHP는 **"MA 위에 올리는 비즈니스 레이어"** 로는 아주 적합합니다. 코어 대체는 비현실적입니다.
(굳이 PHP로 WS를 쓰려면 ReactPHP / Swoole / Laravel Octane 같은 상주 프로세스 런타임이 필요합니다.)

### 9-3. 코어를 직접 건드릴 때

Python입니다. 그리고 쉬운 영역이 분명히 있습니다:
- **새 프로바이더 추가** → `_demo_*_provider` 템플릿 복사 → `__init__.py` + `manifest.json` 2개 파일로 시작
- `DEVELOPMENT.md` 의 "Building a new Music Provider / Player Provider / Metadata Provider / Plugin Provider" 섹션이 단계별 가이드

## 10. 유튜브 강의 영상 제작 가능한가? — ✅ 가능, 소재도 풍부

**라이선스**: Apache-2.0 → 강의/리뷰/튜토리얼 제작 및 수익화 전부 자유.

### 10-1. 지켜야 할 선 4가지

1. **다운로드/우회 방법을 가르치지 않기** — 사용 정책 정면 위반. 영상 신고/채널 리스크.
2. **상표 주의** — 코드는 Apache-2.0이지만 "Music Assistant" 이름/로고로 **공식 채널인 척하면 안 됨**. 제목은 "Music Assistant 설치 가이드 (비공식)" 식으로.
3. **`app_vars` 번들 키를 화면에 노출하거나 추출법을 설명하지 않기**.
4. **내 계정 토큰/비밀번호 블러 처리** — `/auth`, 장기 토큰, 프로바이더 로그인 화면.

### 10-2. 바로 쓸 수 있는 커리큘럼 (한국 시장 공백 큼)

| # | 제목 | 길이 | 타겟 |
| --- | --- | --- | --- |
| 1 | 집에 있는 스피커 전부 하나로 묶기 — Music Assistant 입문 | 10분 | 입문 |
| 2 | 라즈베리파이 + Docker 30분 설치 완벽 가이드 | 20분 | 입문 |
| 3 | NAS 음원 + 스트리밍을 하나의 라이브러리로 | 15분 | 입문 |
| 4 | Sonos / Chromecast / AirPlay 멀티룸 동기화 실전 | 20분 | 중급 |
| 5 | Home Assistant 자동화 — "퇴근하면 음악 켜지는 집" | 25분 | 중급 |
| 6 | **Claude에 MCP 붙여서 말로 음악 틀기** (최고 화제성 🔥) | 20분 | 고급 |
| 7 | **Ollama로 무료 AI DJ 만들기** (ai_radio + 로컬 LLM) | 25분 | 고급 |
| 8 | 음악 서버 플러그인 직접 만들기 (Python 입문자용) | 30분 | 개발 |
| 9 | 130개 프로바이더 코드로 배우는 대규모 async Python 아키텍처 | 40분 | 개발 |
| 10 | 오픈소스 첫 PR 보내기 — 실전 기여 가이드 | 20분 | 개발 |

**6·7번이 차별화 포인트**입니다. "MCP + 실제 하드웨어 제어"는 한국어 콘텐츠가 거의 없고, AI 트렌드와 스마트홈 수요가 동시에 걸립니다.

**부가 수익 경로**: 멤버십(설정 파일/Docker Compose 템플릿 제공), 제휴 링크(라즈베리파이/스피커/NAS), 전자책, 인프런·클래스101 강의 패키지화.

## 11. 수익화 아이디어 (티어별)

> 전제: Apache-2.0은 상업적 사용을 허용합니다. 단 ① 배포 시 LICENSE/NOTICE 포함 ② 상표 미사용 ③ **사용 정책(다운로드 기능 금지)** ④ **한국 상업 공간은 공연권 별도** 를 반드시 지켜야 합니다.

### 티어 1 — 즉시 시작 (자본 0원)

**1. 설치·구축 대행 서비스**
- 타겟: 스마트홈에 관심은 있지만 Docker를 못 다루는 사람 / 오디오 애호가
- 상품: 원격 설치 15~30만원, 방문 설치 30~60만원, 유지보수 월 2~5만원
- 채널: 당근 동네생활, 네이버 카페(오디오/스마트홈), 숨고, 크몽
- 확장: 홈오디오 설치업체·인테리어 업체와 제휴 (그들의 서비스 번들에 포함)

**2. 교육 콘텐츠**
- 유튜브 애드센스 + 멤버십 (위 10-2 커리큘럼)
- 전자책 "셀프호스팅 음악 서버 완전정복" (크몽/부크크)
- 인프런·클래스101 강의 (10~15만원대)
- 노션 기반 유료 템플릿 팩 (Docker Compose + HA 자동화 YAML + 트러블슈팅)

**3. 커미션형 커스텀 개발**
- 한국 특화 프로바이더 제작 (주의: 각 서비스 ToS 확인 필수)
- 고객 전용 플러그인, 대시보드 커스터마이징
- 크몽/위시켓/프리랜서 플랫폼

### 티어 2 — 제품화 (자본 소규모)

**4. 턴키 하드웨어 번들 — 가장 유력** ⭐
- 라즈베리파이 5 / N100 미니PC + SD/SSD 프리셋 이미지 + 케이스 + 설치 매뉴얼
- 1차 세팅 완료 상태 배송 → "전원만 꽂으면 끝"
- 가격: 원가 15~25만원 → 판매 35~50만원 (세팅 + 1년 지원 포함)
- Apache-2.0 준수: LICENSE/NOTICE 동봉, 제품명은 자체 브랜드 (예: "홈사운드 박스")
- 상향 판매: DAC/앰프/스피커 패키지, 설치 서비스

**5. B2B 매장 BGM 시스템** ⭐ 수익성 최상
- 타겟: 카페, 헬스장, 미용실, 요가원, 레스토랑, 호텔, 사무실
- 가치: 멀티존 개별 제어, 시간대별 플레이리스트 자동 전환, 매장 안내방송(announcements 컨트롤러), 지점 중앙관리
- 기존 상업용 BGM 서비스 대비 하드웨어 자유도 + 커스터마이징
- 과금: 구축비 + 월 구독(지점당 3~10만원)
- ⚠️ **필수**: 매장 재생은 **공연권** 영역입니다. 한국음악저작권협회(KOMCA) / 한국음반산업협회 등 공연사용료 신고·납부, 그리고 **B2B 상업용 음원 서비스를 소스로 사용**해야 합니다. 개인 Spotify 계정으로 매장 재생은 ToS 위반입니다. 이 부분을 "합법 구축 컨설팅"으로 상품화하면 오히려 차별점이 됩니다.

**6. React 기반 전용 리모컨 앱**
- WebSocket API 위에 React Native 앱 → 공식 Vue PWA보다 가벼운 UX, 한국어 최적화
- 과금: 무료 + 프리미엄(멀티룸 프리셋, 위젯, 시리 단축어) 월 2~3천원
- 선행 투자 적고(API 이미 완성), 포트폴리오 가치도 큼

### 티어 3 — 스케일 (개발 역량 투입)

**7. 멀티테넌트 중앙관리 SaaS** ⭐ MRR 모델
- 체인 매장/프랜차이즈용 대시보드: 전 지점 상태 모니터링, 원격 플레이리스트 배포, 재생 리포트, 장애 알림
- 스택: React(관리자) + PHP Laravel 또는 Python FastAPI(백엔드) + 각 지점 MA 인스턴스를 장기 토큰으로 제어
- 과금: 지점당 월 2~5만원 → 50지점이면 월 100~250만원
- **PHP로 만들기에 가장 적합한 영역**

**8. AI 음악 비서 (MCP 기반)** ⭐ 차별화 최상
- `fastmcp_server` + `openai_compatible` + `ai_radio` 조합
- "분위기 말하면 알아서 틀어주는 AI DJ", 음성 제어, 상황 인식 자동 재생
- B2B: 매장 컨셉에 맞춘 AI 큐레이션 (사람 MD 대체 → 명확한 비용 절감 세일즈 포인트)
- 로컬 Ollama로 돌리면 **API 비용 0원** → 마진 극대화
- 과금: 프리미엄 애드온 또는 B2B 구독 상위 플랜

**9. 오디오 분석 기반 부가 서비스**
- `sonic_analysis`(CLAP 임베딩) + `sonic_similarity`(usearch) + `smart_fades`(BPM/키) 활용
- 상품: 헬스장 BPM 기반 운동 구간 플레이리스트, 카페 시간대 무드 자동 편성, DJ용 키/BPM 믹스 추천
- 이 기능 자체가 상업용 BGM 서비스에 거의 없는 차별 요소

**10. 전문성 자산화 (간접 수익)**
- 업스트림에 지속 기여 → 커밋 히스토리가 포트폴리오
- `ko_KR.json` 번역 품질 개선, 한국 환경 문서화 → 커뮤니티 평판
- 결과: 스마트홈/오디오 업계 이직·외주 단가 상승, 기술 블로그·강연, 오디오 브랜드 앰배서더

### 추천 실행 순서

```
1단계 (1~2개월)  유튜브 콘텐츠 + 설치 대행   → 시장 수요 검증, 자본 0원
2단계 (3~4개월)  하드웨어 번들 소량 판매      → 현금흐름 확보
3단계 (5~8개월)  B2B 매장 1호점 레퍼런스     → 공연권 프로세스 정립, 단가 확보
4단계 (9개월~)   SaaS 중앙관리 + AI 비서     → MRR 전환, 스케일
```

**핵심 통찰**: 이 프로젝트의 진짜 가치는 코드가 아니라 **130개 프로바이더 = 이미 검증된 통합 자산**입니다. 혼자서는 10년이 걸릴 연동을 공짜로 얻고, 거기에 **"한국 시장 로컬라이징 + 설치/운영 서비스 + AI 레이어"** 를 얹어 파는 구조가 가장 현실적인 수익 모델입니다.

### 리스크 체크리스트

| 리스크 | 대응 |
| --- | --- |
| 저작권/공연권 | B2B는 KOMCA 등 공연사용료 + 상업용 음원 서비스 필수 |
| 서비스 ToS | 개인 계정 상업 사용 금지. 각 서비스 B2B 플랜 확인 |
| 사용 정책 위반 | 다운로드/내보내기 기능 절대 추가 금지 |
| Apache-2.0 | 배포 시 LICENSE + NOTICE 포함 |
| 상표 | "Music Assistant" 이름/로고로 공식 사칭 금지, 자체 브랜드 사용 |
| 업스트림 변경 | 포크 유지보수 부담. 가능하면 플러그인으로 분리해 코어 미수정 |
| 번들 API 키 | `app_vars` 추출/전용 금지. 상업 제품은 자체 키 발급 |

## 12. 한 줄 요약

> **집 안의 모든 스피커와 모든 음악 서비스를 하나로 묶는 셀프호스팅 음악 서버**이고, 동시에 **MCP 서버 + LLM 추상화 + 벡터 검색을 실전 투입한 AI 에이전트 레퍼런스 코드베이스**입니다.
> 수익화는 "코드 판매"가 아니라 **설치·운영 서비스 → 하드웨어 번들 → B2B 매장 BGM → SaaS/AI 레이어** 순서로 쌓는 것이 정답입니다.

---

*정리: Claude Code (카리나 페르소나) · 저장소 https://github.com/bmshin94/server*
