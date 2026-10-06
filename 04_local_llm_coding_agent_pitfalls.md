# 20 failure patterns from running a small local LLM (gpt-oss:20b) as a tool-calling coding agent

**Setup**: a local open-weight model (gpt-oss:20b, later also qwen3.6:35b-a3b) wired up as a delegate "junior agent" that a larger orchestrating model hands off mechanical tasks to — file edits, remote SSH investigation, log analysis — over several months of real usage. These are the failure modes that showed up repeatedly, with what actually fixed each one.

The biggest meta-lesson first: **small/mid local models are reliable executors of well-specified, narrow tasks, and unreliable at judgment calls.** Every pitfall below stems from asking the model to do a little more inference about *how* to do something than it could reliably handle, or from trusting a self-report instead of checking the actual artifact.

| # | Pattern | Symptom | Fix |
|---|---|---|---|
| 0 | No backup before delegating a file edit | Model edits wrong, original is gone | Caller backs up (`cp`/git commit) *before* handing off, always |
| 1 | Vague command intent ("check GPU status") | Model runs only `nvidia-smi`, misses AMD GPU entirely | Give the exact command(s) to run, not a goal |
| 2 | No output format specified | Free-form prose instead of parseable output | Specify JSON/table/list explicitly |
| 3 | No access precheck | Fails deep into a remote task with "permission denied" | Have it verify access (`ls`, `whoami`, path) *first*, as its own step |
| 4 | Absolute paths outside its sandbox root | Silent read failures ("Path escapes sandbox root") | Copy needed files *into* its working dir; give only relative paths |
| 5 | Delegating a find/replace with regex or heavy escaping in the literal text | Model retypes the text and gets escaping wrong → edit rejected → retry loop → context blown | Describe the target by function/block name instead of literal text with special chars |
| 6 | Large (tens of KB) payload needs to be inspected | Repeated re-reads of the same big blob blow up its own context (we saw input tokens hit 1.3M) | Do purely mechanical large-payload work yourself; if delegating, explicitly say "don't dump full contents, just report size/fields" |
| 7 | Told to write output to an absolute path containing a run-id-looking UUID segment | Model "corrects" the UUID to its own run id while reconstructing the path → write fails | Only ever give it a bare filename, never an absolute path, for output targets |
| 8 | A single remote command is slow (cold start, big model reload) | Model backgrounds it and polls repeatedly, each poll re-ingesting cumulative output → runaway token growth, no actual progress | If a single call will take minutes, don't delegate the waiting — do it yourself |
| 9 | Follow-up "fix that" instruction after a prior task | Model replies "done: X=8" but never actually called the edit/write tool — file unchanged | Don't trust "done" prose; diff/grep the file. If unchanged, re-ask demanding an explicit tool call |
| 10 | Long instruction file (`--message-file`) | Model echoes the instructions back instead of executing | Short, imperative phrasing: "execute the following now, do not repeat it back" |
| 11 | 4+ independent investigation topics in one task | Each topic's command output accumulates until context limit is exceeded, failing the whole batch | One topic (or tightly related subset) per delegation, sequential |
| 12 | Model misjudges its own session as "not found" mid-task | Silently re-runs the same long-running job a 2nd/3rd time in parallel, causing real resource contention | After any long remote job, check the process list for duplicates yourself — don't trust exit 0 |
| 13 | Reports "SSH connection failed" | Connection is actually fine; model fabricated the failure | Before retrying through the model, verify connectivity yourself with one direct call. Two fabricated failures in a row = stop delegating that task to it |
| 14 | Asked to "output verbatim" a long/repetitive block (service unit lists, Korean log text) | Silently paraphrases/corrupts characters (e.g. `.` → `-` in service names) despite exit 0 | Don't trust verbatim reproduction of anything where every character matters; re-fetch important identifiers with a narrower, unambiguous query |
| 15 | `find` reports a file doesn't exist | File actually exists (confirmed moments later via `grep -rl`) | Treat negative existence results as unconfirmed until cross-checked a second way |
| 16 | Nested `$(...)` command substitution inside a delegated shell command | Variable silently resolves to empty string, command "succeeds" against the wrong path | Split into two calls: resolve the value first, then hardcode it literally into the second call |
| 17 | Delegated calls routinely take 1–2 minutes | Looks like a failure because it hit the harness's default foreground timeout, when the model was just slow | Default to background execution with a generous timeout (5 min+) for any local-model call |
| 18 | Shell field-reference syntax (`$2`, `$18` — e.g. `awk -F,`) inside a one-line delegated command | An intermediate shell layer re-evaluates the string and strips/mangles `$`+digit patterns as positional parameters | Avoid `$N` syntax in anything passed through a delegation wrapper; write scripts to a file and execute the file instead of inlining them |
| 19 | **False refusal**: model replies "I can't run that command / no SSH configuration available" for a read-only, idempotent command | The command actually works fine — this is hallucinated unwillingness, not a real restriction (temperature=1 nondeterminism, not a permission issue) | For read-only commands, retry unmodified once or twice before assuming it's really blocked. Never auto-retry this way for state-changing commands |
| 20 | Multi-step stateful task (e.g., "create a new systemd service") | Model returns a shell script as *text* for a human to paste, instead of executing it itself | Break multi-step stateful operations into separate delegated steps (write → reload → enable → verify status), each requiring confirmation of an actual tool call |

## The one rule that mattered most

**Diagnosis and design decisions never get delegated to the small model — only mechanical execution of an already-known fix does.** When we broke this rule (asking it to "figure out why this failed and fix it" for a genuinely unknown bug), it failed every time, slowly, in confusing ways. When we kept diagnosis on the orchestrating side and gave the small model "run exactly this script," it succeeded reliably even across five consecutive different failure causes in the same debugging session.

## Why this might be useful to others

Most public discussion of local-LLM agents focuses on benchmark scores or single-shot task success rates. This is the accumulated, unglamorous operational layer — the actual failure taxonomy you hit once you're running a small open-weight model as a *standing* delegate over weeks of real tasks rather than a one-off eval.

---

# [한국어] 로컬 소형 LLM(gpt-oss:20b)을 툴콜 코딩 에이전트로 쓰면서 겪은 실패 패턴 20가지

**환경**: 로컬 오픈웨이트 모델(gpt-oss:20b, 이후 qwen3.6:35b-a3b도 병행)을, 더 큰 오케스트레이팅 모델이 파일수정·원격 SSH 조사·로그분석 같은 기계적 작업을 맡기는 "하위 에이전트"로 몇 달간 실사용했습니다. 아래는 반복적으로 나타난 실패 유형과, 각각 실제로 통했던 해법입니다.

가장 중요한 메타 교훈부터: **소형~중형 로컬 모델은 명확히 명시된 좁은 작업의 실행자로는 신뢰할 만하지만, 판단이 필요한 일에는 신뢰할 수 없습니다.** 아래 함정들은 전부 모델에게 "어떻게 할지"에 대한 추론을 조금이라도 더 맡겼을 때, 또는 실제 결과물을 확인하지 않고 자기보고를 믿었을 때 생겼습니다.

| # | 패턴 | 증상 | 해법 |
|---|---|---|---|
| 0 | 파일수정 위임 전 백업 없음 | 모델이 잘못 고치면 원본이 사라짐 | 위임하는 쪽이 작업 전에 반드시 먼저 백업(`cp`/git commit) |
| 1 | 모호한 명령 의도("GPU 상태 확인해줘") | `nvidia-smi`만 돌려서 AMD GPU를 통째로 놓침 | 목표가 아니라 실행할 정확한 명령어를 그대로 지정 |
| 2 | 출력 형식 미지정 | 파싱 불가능한 산문체 응답 | JSON/표/목록 등 형식을 명시적으로 지정 |
| 3 | 접근권한 사전확인 없음 | 원격 작업 한참 진행하다 "permission denied"로 실패 | 접근 가능 여부(`ls`, `whoami`, 경로)를 먼저 별도 단계로 확인시킴 |
| 4 | 샌드박스 루트 밖의 절대경로 | 조용한 읽기 실패("Path escapes sandbox root") | 필요한 파일을 작업 디렉토리 안으로 복사하고, 상대경로만 지정 |
| 5 | 정규식·이스케이프가 많은 텍스트를 찾아바꾸기로 위임 | 모델이 텍스트를 재입력하며 이스케이프를 틀림 → 수정 거부 → 재시도 루프 → 컨텍스트 소진 | 특수문자 섞인 리터럴 텍스트 대신 함수/블록 이름으로 대상 지정 |
| 6 | 수십KB급 대용량 페이로드 검토 필요 | 같은 거대 블록을 반복 재읽기하며 자기 컨텍스트 폭주(입력토큰 130만까지 관측) | 순수 기계적 대용량 작업은 직접 처리. 위임해야 하면 "전체 내용 덤프 말고 크기/필드만 보고" 명시 |
| 7 | 출력 경로로 run-id처럼 보이는 UUID가 든 절대경로를 지시 | 경로를 재구성하며 UUID를 자기 run id로 "교정"해버려 쓰기 실패 | 출력 대상은 항상 파일명만, 절대경로는 절대 지시하지 않음 |
| 8 | 단일 원격 명령이 느림(콜드스타트, 대형모델 재적재) | 백그라운드로 돌리고 반복 폴링하며 매번 누적 출력을 재흡수 → 토큰 폭주, 실제 진행은 0 | 한 호출이 몇 분 걸릴 것 같으면 대기 자체를 위임하지 말고 직접 처리 |
| 9 | 이전 작업에 대한 후속 "그거 고쳐줘" 지시 | "완료: X=8"이라 답하지만 실제로는 edit/write 툴을 호출 안 함 — 파일 그대로 | "완료" 서술을 믿지 말고 diff/grep으로 확인. 안 바뀌었으면 "도구를 실제로 호출하라" 명시해 재요청 |
| 10 | 긴 지시문 파일(`--message-file`) | 실행 대신 지시문을 그대로 되풀이함 | "지금 즉시 실행하라, 되풀이하지 말라"처럼 짧고 명령형으로 작성 |
| 11 | 하나의 작업에 독립된 조사항목 4개 이상을 묶음 | 각 항목의 명령 출력이 누적되며 컨텍스트 한도를 넘어 전체 배치가 실패 | 항목 하나(또는 밀접하게 연관된 소수)씩 순차 위임 |
| 12 | 모델이 자기 세션을 "없음"으로 오판 | 같은 장시간 작업을 2·3차로 조용히 중복 실행 → 실제 리소스 경합 발생 | 장시간 원격 작업 뒤엔 직접 프로세스 목록으로 중복 확인 — exit 0을 믿지 말 것 |
| 13 | "SSH 연결 실패"로 보고 | 실제 연결은 정상 — 모델이 실패를 지어냄 | 모델을 통해 재시도하기 전에 직접 호출 1회로 연결성 확인. 같은 날조가 2회 연속이면 그 작업은 더 이상 위임하지 않음 |
| 14 | 길고 반복적인 블록(서비스 유닛 목록, 한글 로그)을 "그대로 출력"하라고 지시 | exit 0임에도 조용히 문자를 바꿔치기(예: 서비스명의 `.`→`-`) | 글자 하나하나가 중요한 결과물은 신뢰하지 말고, 중요한 식별자는 더 좁고 명확한 질의로 재조회 |
| 15 | `find`가 파일이 없다고 보고 | 잠시 뒤 `grep -rl`로 확인하니 실제로는 존재 | 부정(없음) 결과는 다른 방법으로 교차확인하기 전까지 미확정으로 취급 |
| 16 | 위임하는 셸 명령 안의 중첩 `$(...)` 명령치환 | 변수가 조용히 빈 문자열로 치환돼 엉뚱한 경로로 "성공"함 | 값을 먼저 알아내는 호출과, 그 값을 리터럴로 박아 넣는 호출을 분리 |
| 17 | 위임 호출이 평소 1~2분씩 걸림 | 하네스의 기본 foreground timeout에 걸려 실패처럼 보임(실은 그냥 느린 것) | 로컬모델 호출은 기본적으로 백그라운드+넉넉한 타임아웃(5분 이상)으로 |
| 18 | 한 줄 위임 명령 안의 셸 필드참조 문법(`$2`, `$18` 등, 예: `awk -F,`) | 중간 셸 레이어가 문자열을 재평가하며 `$`+숫자 패턴을 포지셔널 파라미터로 오인해 날려먹음 | 위임 래퍼를 거치는 명령에는 `$N` 문법을 쓰지 말고, 스크립트를 파일로 작성해 그 파일을 실행 |
| 19 | **거짓 거부**: 읽기전용·멱등 명령인데 "실행 불가/SSH 설정 없음"이라 응답 | 실제로는 명령이 정상 작동 — 권한 문제가 아니라 환각에 의한 거부(temperature=1 비결정성) | 읽기전용 명령이면 수정 없이 1~2회 재시도부터. 상태를 바꾸는 명령에는 이 자동재시도를 절대 적용하지 말 것 |
| 20 | 다단계 상태변경 작업(예: "새 systemd 서비스 생성") | 직접 실행하는 대신 사람이 붙여넣을 셸 스크립트를 텍스트로만 반환 | 다단계 상태변경 작업은 (작성→재적재→활성화→상태확인) 각각 별도로 쪼개 위임하고, 매 단계 실제 도구 호출 여부를 확인 |

## 가장 중요했던 단 하나의 규칙

**진단과 설계 결정은 절대 소형 모델에게 위임하지 않는다 — 이미 원인이 밝혀진 수정의 기계적 실행만 위임한다.** 이 규칙을 어겼을 때(진짜 원인을 모르는 버그를 "원인 찾아서 고쳐봐"로 맡겼을 때)는 매번 느리고 혼란스럽게 실패했습니다. 진단은 오케스트레이팅 쪽이 맡고 소형 모델에게는 "정확히 이 스크립트를 실행하라"만 준 경우엔, 같은 디버깅 세션에서 연속으로 다른 원인의 실패 5건을 겪었어도 매번 신뢰성 있게 성공했습니다.

## 왜 공유할 가치가 있는가

로컬 LLM 에이전트에 대한 공개된 논의는 대부분 벤치마크 점수나 단발성 작업 성공률에 초점을 맞춥니다. 이건 그보다 덜 화려하지만 실제로 누적된 운영 레이어입니다 — 소형 오픈웨이트 모델을 한 번의 평가가 아니라 몇 주에 걸친 **상시 위임처**로 실제로 돌릴 때 부딛히게 되는 진짜 실패 분류표입니다.
