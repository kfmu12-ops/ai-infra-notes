# Fixing ACE-Step VRAM OOM on repaint — the real fix was one env var, but it took four wrong turns to find

**Context**: ACE-Step (local music generation model) running via a standalone `api_server.py` (switched over from the Gradio demo server, which doesn't even expose the `repaint` — section regeneration — parameter at all; Gradio's bundled API is text2music-only).

## The symptom

`repaint` (regenerating a specific section of an already-generated track) reliably triggered a VRAM OOM, while plain text2music generation didn't.

## The four things tried, in order

1. **Disable the LM component** — reduces footprint, but OOM persisted on repaint specifically.
2. **Found and fixed a missing quantization patch** — the DiT model was loading at full float32 precision instead of the intended int8 quantization (a patch had silently not been applied). Fixing this helped overall footprint but did not fully resolve the repaint-specific OOM.
3. **Tried the `PYTORCH_*_ALLOC_CONF` family of allocator-tuning env vars** — no measurable effect in this case.
4. **The actual fix**: `ACESTEP_OFFLOAD_DIT_TO_CPU=true`. This offloads the DiT model to CPU between the steps where it isn't actively needed, and resolved the repaint OOM.

## The actual working configuration (all four pieces, together)

Repaint specifically needs **all** of: LM disabled + the quantization patch applied + `ACESTEP_OFFLOAD_DIT_TO_CPU=true`. The `PYTORCH_*_ALLOC_CONF` variables turned out not to matter for this particular failure, but are harmless to leave set.

Two separate, smaller findings from the same project, worth noting on their own:
- The default vision-LLM choice for a related image-description step was switched from `qwen3-vl:8b` to `gemma4:12b` after measuring that `qwen3-vl:8b` ignores a `think:false` request and reasons at length anyway, burning its token budget and returning empty responses. `gemma4:12b` is stable but runs VRAM close to the limit (15.9/16GB) when co-loaded with ACE-Step.
- A separate, harder bug (a browser input field silently not accepting typed text) was never root-caused — code review and focus-stealing were both ruled out, but no definitive cause was found. It was worked around by replacing the form-POST architecture with fetch+streaming entirely, which happened to make the symptom disappear along with the architecture that exhibited it.

## Why this might be useful to others

If you're running ACE-Step (or likely any diffusion-transformer music/audio model with a similar DiT+LM split) and hitting OOM specifically on a regeneration/inpainting-style operation rather than plain generation, `ACESTEP_OFFLOAD_DIT_TO_CPU=true` is worth trying directly rather than going through the same allocator-tuning detour first — in this case that detour produced no measurable benefit.

---

# [한국어] ACE-Step repaint의 VRAM OOM 해결 — 진짜 해법은 환경변수 하나였지만 거기까지 네 번의 삽질이 있었다

**배경**: Gradio 데모 서버(여기엔 `repaint`—구간 재생성—파라미터 자체가 없음, Gradio 내장 API는 text2music 전용)에서 standalone `api_server.py`로 전환한 ACE-Step(로컬 음악 생성 모델)입니다.

## 증상

`repaint`(이미 생성된 곡의 특정 구간을 재생성)는 거의 매번 VRAM OOM을 냈지만, 일반 text2music 생성은 멀쩡했습니다.

## 순서대로 시도한 네 가지

1. **LM 컴포넌트 끄기** — 점유량은 줄었지만 repaint 특유의 OOM은 그대로였습니다.
2. **빠져있던 양자화 패치 발견·수정** — DiT 모델이 의도된 int8 양자화가 아니라 full float32 정밀도로 로드되고 있었습니다(패치가 조용히 적용 안 돼 있었음). 이걸 고치니 전체 점유량은 줄었지만 repaint 특유의 OOM은 완전히 해결되지 않았습니다.
3. **`PYTORCH_*_ALLOC_CONF` 계열 할당기 튜닝 환경변수들 시도** — 이 경우엔 측정 가능한 효과가 없었습니다.
4. **실제로 통한 해법**: `ACESTEP_OFFLOAD_DIT_TO_CPU=true`. DiT 모델이 당장 필요 없는 단계 사이에 CPU로 오프로드되도록 하는 설정이고, 이게 repaint OOM을 해결했습니다.

## 실제로 작동하는 구성(네 가지를 전부 함께)

repaint는 구체적으로 **다음 전부**가 필요합니다: LM 끄기 + 양자화 패치 적용 + `ACESTEP_OFFLOAD_DIT_TO_CPU=true`. `PYTORCH_*_ALLOC_CONF` 계열 변수는 이 특정 실패엔 영향이 없었던 것으로 확인됐지만, 그대로 설정해둬도 해는 없습니다.

같은 프로젝트에서 나온, 따로 적어둘 만한 더 작은 발견 두 가지:
- 관련 이미지-설명 단계의 기본 비전 LLM을 `qwen3-vl:8b`에서 `gemma4:12b`로 바꿨습니다 — `qwen3-vl:8b`는 `think:false` 요청을 무시하고 계속 장황하게 "생각"하다 토큰예산을 다 쓰고 빈 응답을 내는 게 실측됐습니다. `gemma4:12b`는 안정적이지만 ACE-Step과 동시 로드 시 VRAM이 한계(15.9/16GB)에 거의 다다릅니다.
- 별개로 더 어려웠던 버그(브라우저 입력 필드가 조용히 타이핑을 안 받는 증상)는 끝내 근본원인을 못 찾았습니다 — 코드와 포커스 탈취 둘 다 무죄로 확인됐지만 확정적인 원인은 못 찾았습니다. 폼POST 아키텍처를 fetch+스트리밍으로 전면 교체하는 걸로 우회했는데, 마침 그 증상이 보이던 아키텍처 자체가 없어지면서 증상도 함께 사라졌습니다.

## 왜 공유할 가치가 있는가

ACE-Step(또는 DiT+LM 구조가 비슷한 다른 디퓨전-트랜스포머 음악/오디오 모델)을 돌리면서 일반 생성이 아니라 재생성/인페인팅류 작업에서만 유독 OOM이 난다면, 같은 할당기 튜닝 삽질을 먼저 거치지 말고 `ACESTEP_OFFLOAD_DIT_TO_CPU=true`를 바로 시도해볼 가치가 있습니다 — 이 경우엔 그 삽질이 측정 가능한 효과가 전혀 없었습니다.
