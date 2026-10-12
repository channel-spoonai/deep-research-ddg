---
name: deep-research-ddg
description: 로컬·비Anthropic 모델용 딥리서치. ddgs(DuckDuckGo 우선) 웹 검색으로 여러 각도 검색 → 출처 본문 추출 → 주장별 3표 적대적 검증 → 출처 인용 보고서를 만든다. 내장 /deep-research(Anthropic WebSearch·Workflow 필요)의 대체품으로, 현재 Claude Code 세션이 쓰는 모델 엔드포인트를 그대로 사용한다.
argument-hint: <연구 질문>
disable-model-invocation: true
allowed-tools: Bash(uv run --quiet --script ${CLAUDE_SKILL_DIR}/scripts/deep_research_ddg.py *), Read, AskUserQuestion
---

# deep-research-ddg

내장 `/deep-research`와 같은 파이프라인을 **파이썬 오케스트레이터**가 실행한다. 이 세션(메인 모델)은 연구를 직접 하지 않는다. 질문을 다듬어 스크립트를 실행하고, 결과 보고서를 사용자에게 전달하는 일만 한다.

```
Scope(질문→검색 각도 N개) → Search(ddgs + LLM 랭킹) → URL 중복·예산 필터
 → Fetch(본문·PDF 추출 + 주장 추출, 원문에 없는 인용 폐기) → Verify(주장마다 독립 검색 3개 + 3표, 2표 반박이면 폐기)
 → Synthesize(살아남은 주장만 병합·신뢰도·출처)
```

- **LLM**: 스크립트가 이 세션의 `ANTHROPIC_BASE_URL`·`ANTHROPIC_AUTH_TOKEN`/`ANTHROPIC_API_KEY`·`ANTHROPIC_MODEL` 환경변수를 읽어 같은 엔드포인트(`/v1/messages`)를 호출한다. ccx 같은 런처가 띄운 로컬 프록시도 그대로 쓴다. URL을 따로 설정하지 않는다.
- **검색**: DuckDuckGo를 우선 쓴다. DDG는 연속 요청을 봇으로 보고 HTTP 202 CAPTCHA로 막기 때문에 최소 12초 간격을 지키고, 그 사이 요청과 차단 중 요청은 다른 엔진(yahoo 등)이 맡는다.

연구 질문: $ARGUMENTS

## 절차

1. **질문 점검.** 질문이 너무 넓으면(예: "무슨 차 살까" — 예산·용도·지역 없음) AskUserQuestion으로 2~3개만 물어 범위를 좁히고, 답을 질문 문장에 녹여 넣는다. 충분히 구체적이면 바로 진행한다. 질문이 비어 있으면 무엇을 조사할지 묻는다.

2. **엔드포인트 확인.** 아래를 실행한다(수 초).

   ```bash
   uv run --quiet --script ${CLAUDE_SKILL_DIR}/scripts/deep_research_ddg.py --check-endpoint
   ```

   `uv: command not found`가 나오면 중단하고 uv 설치를 안내한다(`brew install uv` 또는 `curl -LsSf https://astral.sh/uv/install.sh | sh`). 첫 실행은 uv가 ddgs·pypdf를 받느라 수십 초 걸릴 수 있다.
   `structured call: OK`가 아니면 중단하고 출력의 `ERROR:` 문장을 그대로 사용자에게 전한다. 특히 "환경변수에서 찾지 못했습니다"는 Claude 구독 로그인으로 Anthropic 모델을 쓰는 경우이므로 내장 `/deep-research`를 권한다.

3. **실행.** Bash를 **`run_in_background: true`** 로 실행한다(포그라운드 타임아웃을 넘는다). 질문은 따옴표 문제를 피하려고 heredoc으로 넘긴다.

   ```bash
   uv run --quiet --script ${CLAUDE_SKILL_DIR}/scripts/deep_research_ddg.py --depth standard --question-file - <<'EOF'
   <다듬은 연구 질문>
   EOF
   ```

   - `--depth`: `quick`(각도 3·출처 8·검증 10) / `standard`(5·15·25, 원본과 동일, 기본값) / `deep`(6·24·40). 사용자가 "빠르게"라고 하면 quick, "철저히"라고 하면 deep.
   - 소요 시간은 모델 속도와 검색 차단 상황에 따라 크게 달라진다. 로컬 모델 기준 quick 15분 안팎, standard는 그 2~3배를 예상한다.
   - 그 외 옵션: `--backend`(기본 `auto` = duckduckgo 우선 + yahoo·brave·mojeek·google 보조. `duckduckgo`만 주면 DDG만 사용), `--ddg-interval 12`, `--proxy socks5h://…`(검색 IP 차단 회피), `--concurrency 3`(LLM 동시 요청 수), `--out-dir research`.
   - 시작하면 사용자에게 한 줄로 알린다: depth, 예상 소요 시간, 로그 파일 위치(stdout 첫 줄 `LOG=...`).

4. **대기.** 백그라운드 작업이 끝나면 알림이 온다. 그 사이에 폴링하거나 같은 조사를 직접 하지 않는다. 사용자가 진행 상황을 물으면 `LOG=` 파일의 마지막 20줄을 읽어 요약한다.

5. **결과 전달.** stdout의 `REPORT_MD=` 경로를 Read로 읽고 다음 순서로 답한다.
   - 요약(보고서 요약을 그대로 또는 압축)
   - 핵심 발견: 각 항목에 신뢰도·투표(예: 3-0)·출처 URL
   - 주의사항, 그리고 반박된 주장이 의미 있으면 1~2줄
   - 보고서 파일 경로(.md / .json)
   보고서에 없는 사실을 덧붙이지 않는다. 종료 코드가 0이 아니거나 `summary`가 인프라 실패(검증 패널 전부 실패 등)를 말하면 연구 결과처럼 포장하지 말고 실패라고 알린 뒤 재시도를 제안한다.

## 참고
- 보고서는 작업 디렉터리의 `research/<날짜시각>-<질문>.{md,json,log}`에 쌓인다.
- 엔드포인트를 직접 지정하려면 환경변수 `DRD_BASE_URL`·`DRD_API_KEY`·`DRD_MODEL`(`DRD_API=openai`면 `/v1/chat/completions`)을 쓴다. 지정하면 세션 환경변수보다 우선한다.
- `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1`이 켜져 있으면 Claude Code가 Bash 환경에서 자격 증명을 지워 자동 감지가 실패한다. 이때는 `DRD_*`로 직접 지정한다.
- 검색이 계속 실패하면 로그의 `search [엔진] blocked(상태코드)` 줄로 원인을 본다. 202=DDG 봇 차단, 403=IP 차단, 429=요청 과다.
