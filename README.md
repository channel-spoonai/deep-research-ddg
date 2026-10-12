# deep-research-ddg

로컬 모델(mlx-serve, LM Studio, Ollama 등)이나 Anthropic 이외의 엔드포인트로 Claude Code를 쓸 때도 딥리서치를 할 수 있게 해 주는 Claude Code 플러그인입니다.

Claude Code 내장 `/deep-research`는 Anthropic 서버 도구(WebSearch)와 Workflow 기능이 필요해서 로컬 모델에서는 동작하지 않습니다. 이 플러그인은 같은 파이프라인을 파이썬 스크립트로 옮기고, 검색은 [ddgs](https://github.com/deedy5/ddgs)(DuckDuckGo 우선)로 합니다.

```
Scope   질문 → 서로 다른 검색 각도 N개
Search  각도별 웹 검색 + LLM이 관련 결과 선별
Fetch   URL 중복·예산 필터 → 본문(PDF 포함) 추출 → 반증 가능한 주장 + 원문 인용 추출
        (원문에 실제로 없는 인용은 폐기)
Verify  주장마다 독립 검색 3개 + 반박 시도 3표, 2표 이상 반박이면 폐기
Synth   살아남은 주장만 병합해 신뢰도·투표·출처가 달린 보고서 작성
```

## 요구 사항

- Claude Code
- [uv](https://docs.astral.sh/uv/) — 스크립트 의존성(ddgs, pypdf)을 자동으로 설치합니다.
- Anthropic Messages API(`/v1/messages`) 호환 엔드포인트로 Claude Code를 쓰는 환경. 예: ccx 같은 런처, `ANTHROPIC_BASE_URL`을 지정한 LiteLLM·로컬 서버, `ANTHROPIC_API_KEY`.

## 설치

```bash
# GitHub에 올린 경우
claude plugin marketplace add <owner>/deep-research-ddg
claude plugin install deep-research-ddg@deep-research-ddg

# 로컬 폴더에서
claude plugin marketplace add /path/to/deep-research-ddg
claude plugin install deep-research-ddg@deep-research-ddg --scope local   # 현재 프로젝트에서만
```

`--scope`: `user`(모든 프로젝트, 기본값) / `project`(저장소에 공유) / `local`(이 프로젝트에서 나만).

## 사용

```
/deep-research-ddg Qwen3.8-Flash-Next를 M4 Max에서 가장 빠르게 돌리는 설정은?
```

- 깊이: 메시지에 "빠르게"나 "철저히"를 넣으면 `quick`/`deep`으로 실행합니다. 기본은 원본과 같은 `standard`(각도 5 · 출처 15 · 검증 25)입니다.
- 결과: 작업 디렉터리의 `research/<날짜시각>-<질문>.md`(보고서), `.json`(원자료), `.log`(진행 로그).

## LLM 엔드포인트를 정하는 방식

설정할 것이 없습니다. 스크립트는 Claude Code의 Bash 도구로 실행되고, Claude Code는 자기 환경변수를 그대로 물려줍니다. 스크립트는 그중 다음 값을 읽어 **지금 세션이 쓰는 모델과 엔드포인트**를 그대로 호출합니다.

| 환경변수 | 용도 |
|---|---|
| `ANTHROPIC_BASE_URL` | 엔드포인트. 없고 API 키만 있으면 `https://api.anthropic.com` |
| `ANTHROPIC_AUTH_TOKEN` / `ANTHROPIC_API_KEY` | 인증 (`Authorization: Bearer` / `x-api-key`) |
| `ANTHROPIC_MODEL` | 모델. `opus`·`sonnet` 같은 별칭이면 `ANTHROPIC_DEFAULT_*_MODEL`로 풀고, `[256k]` 같은 접미사는 뗍니다. 없으면 `/v1/models`의 첫 모델 |
| `ANTHROPIC_CUSTOM_HEADERS` | 그대로 전달. 이름에 `session`이 들어간 헤더는 값 끝에 `-deep-research`를 붙여 메인 세션의 프롬프트 캐시를 밀어내지 않게 합니다 |

예외:
- **Claude 구독 로그인(OAuth)**으로 Anthropic 모델을 쓰는 세션에는 이 환경변수가 없습니다. 그런 경우에는 내장 `/deep-research`를 쓰세요.
- **Bedrock / Vertex AI / Foundry**는 지원하지 않습니다.
- `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1`이면 Claude Code가 Bash 환경에서 자격 증명을 지우므로 자동 감지가 실패합니다.

이런 경우에는 직접 지정합니다(세션 환경변수보다 우선).

```bash
export DRD_BASE_URL=http://127.0.0.1:1234   # Anthropic 호환 서버
export DRD_API_KEY=...                      # 필요하면
export DRD_MODEL=...                        # 생략하면 /v1/models에서 자동
export DRD_API=openai                       # OpenAI 호환(/v1/chat/completions)만 있는 서버라면
```

연결 확인:

```bash
uv run --script skills/deep-research-ddg/scripts/deep_research_ddg.py --check-endpoint
```

구조화 출력은 Anthropic 경로에서는 `output` 도구 강제 호출(`tool_choice`)로, OpenAI 경로에서는 `response_format: json_schema`로 받습니다. 두 경우 모두 thinking은 끄고, 스키마 맨 앞의 `analysis` 필드에서 먼저 추론하게 합니다.

## 검색 엔진과 DuckDuckGo 차단

DuckDuckGo를 우선 엔진으로 씁니다. 다만 DuckDuckGo는 같은 IP에서 몇 초 간격으로 요청이 이어지면 **HTTP 202와 함께 "bots use DuckDuckGo too" 이미지 CAPTCHA 페이지**를 돌려주고, 이 차단은 수십 초에서 몇 분 동안 이어집니다. ddgs는 이 응답을 "No results found"로 보여 주기 때문에 원인이 드러나지 않습니다.

이 플러그인은 다음처럼 대응합니다.
- 엔진을 직접 호출해 HTTP 상태와 본문을 보고 **차단과 진짜 무결과를 구분**합니다.
- DuckDuckGo는 **최소 12초 간격**으로만 씁니다. 실측에서 13초 간격 15회 연속 요청이 모두 성공했습니다. 간격이 아직 안 찼거나 차단 중인 동안에는 보조 엔진(yahoo, brave, mojeek, google)이 요청을 받습니다.
- 차단된 엔진은 60초부터 시작해 최대 600초까지 늘려 가며 쉬게 합니다.
- 두 엔진에서 결과가 없으면 쿼리를 줄여 다시 검색합니다.
- ddgs 9.16에서 bing·yandex는 비활성 상태입니다.

옵션: `--ddg-interval 12`, `--backend duckduckgo`(DDG만), `--proxy socks5h://127.0.0.1:9150`(Tor 등으로 IP 차단 회피).

## 스크립트 직접 실행

```bash
uv run --script skills/deep-research-ddg/scripts/deep_research_ddg.py --depth quick "질문"
uv run --script skills/deep-research-ddg/scripts/deep_research_ddg.py --help
```
