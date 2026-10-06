# "Hybrid models can't use prompt caching" was wrong — a two-session debugging arc on Ollama/llama.cpp prompt cache behavior

**Model**: a Qwen3.6 hybrid/linear-attention variant served through Ollama (llama.cpp backend).

## Session 1: the (incorrect) conclusion

Every request was taking 50+ seconds regardless of how short the conversation was. The server log explained why:

```
common_init_: KV cache shifting is not supported for this context, disabling KV cache shifting
slot: forcing full prompt re-processing due to lack of cache data
      (likely due to SWA or hybrid/recurrent memory, see llama.cpp#13194)
```

This reproduced even at 1,479 tokens (9% of the 16,384-token context limit) — not a context-length problem. The conclusion at the time: *this model's hybrid/linear attention architecture is structurally incompatible with llama.cpp's standard context-shift caching* — a real, open upstream limitation (llama.cpp PR #13194 landed SWA cache support, but Qwen's hybrid variants are specifically called out as still problematic). The documented workaround (`--no-kv-unified`) was already in effect and didn't help.

**What we did instead**, since the caching mechanism itself seemed unusable: upgraded Ollama (0.32.14 → 0.34.2). This didn't fix the caching bug (identical log lines reproduced on the new version) but genuinely helped throughput — prompt reprocessing went from ~46 to ~105 tok/s (an 8,190-token reprocess dropped from an estimated ~178s to 78s), because the reprocessing itself got faster even though it was still happening every turn.

## Session 2 (the next day): the conclusion gets revised

Re-testing with closer attention to the logs surfaced something session 1 had missed: Ollama 0.34.2's llama-server, despite having standard context-shift disabled, runs a **separate "context checkpoint" cache** (`context checkpoints enabled, max=32, min spacing=8192`, `prompt cache is enabled, size limit: 8192 MiB`).

- Turn 1 (cold, 8,194 tokens): full reprocess, 118.97s (68.9 tok/s) — matches session 1's finding.
- Turn 2, sent as an exact continuation of turn 1 with the **same session-id**: `prompt_eval_cached_count: 8190/8194`, reprocess time **0.322 seconds**. Log line confirms: `checking checkpoint ... → restored context checkpoint`.
- Any request that doesn't match a stored checkpoint prefix exactly falls straight back to full reprocessing.

**Revised, corrected understanding**: it was never "hybrid architecture ⇒ caching impossible." It's "caching works, but only via checkpoint-exact-prefix-match rather than the standard incremental context-shift mechanism" — a different, narrower caveat than the one originally concluded.

Confirmed in real usage the same night with 3 sequential turns on a fixed session-id: turn 1 (cold, 74s, `forcing`) → turn 2 (plain follow-up, 26s, `restored`) → turn 3 (included an actual tool call, both the tool-call decision step and the response-generation step showed `restored`). **Tool calls mixed into the conversation did not break the cache** — ruling out an earlier plan to switch to a non-hybrid model as unnecessary.

**One real precondition this surfaced**: the cache hit only happens because the caller explicitly pinned and reused the same `--session-id`. A separate piece of our own tooling had changed its default behavior (as of an earlier, unrelated fix) to auto-generate a fresh session-id on every call specifically to avoid unbounded prompt growth from long-lived sessions — so callers that don't deliberately override that default are still cold-starting every single call, even with this caching mechanism fully working.

## Why this might be useful to others

Two things worth taking away, independent of the specific model:

1. If you see `"forcing full prompt re-processing due to lack of cache data (likely due to SWA or hybrid/recurrent memory...)"` in llama.cpp/Ollama logs, that message is accurate about *context-shift* specifically being unavailable — it is not proof that *no* caching mechanism is available for that model. Check for a separate checkpoint-based cache before concluding the architecture is incompatible with caching in general.
2. The checkpoint cache's hit condition (exact-prefix match on a pinned session-id) is a meaningfully different contract from classic incremental context-shift (which tolerates appending to a growing conversation more loosely). If your calling code auto-rotates session IDs for unrelated reasons, you can have a fully-working cache mechanism sitting unused.

---

# [한국어] "하이브리드 모델은 프롬프트캐시를 못 쓴다"는 틀렸다 — Ollama/llama.cpp 프롬프트캐시를 둘러싼 이틀간의 디버깅과 자기정정

**모델**: Ollama(llama.cpp 백엔드)로 서빙되는 Qwen3.6 하이브리드/선형어텐션 변형 모델.

## 1일차: (틀렸던) 결론

대화가 얼마나 짧은지와 무관하게 모든 요청이 50초 이상 걸렸습니다. 서버 로그가 이유를 설명했습니다:

```
common_init_: KV cache shifting is not supported for this context, disabling KV cache shifting
slot: forcing full prompt re-processing due to lack of cache data
      (likely due to SWA or hybrid/recurrent memory, see llama.cpp#13194)
```

이건 1,479토큰(16,384토큰 한도의 9%)에서도 재현됐습니다 — 컨텍스트 길이 문제가 아니었습니다. 당시 결론: *이 모델의 하이브리드/선형 어텐션 아키텍처가 llama.cpp의 표준 컨텍스트시프트 캐싱과 구조적으로 호환되지 않는다* — 실제로 존재하는, 아직 해결 안 된 업스트림 한계였습니다(llama.cpp PR #13194이 SWA 캐시 지원을 머지했지만, Qwen의 하이브리드 변형은 여전히 문제로 따로 언급돼 있었음). 공식 우회법(`--no-kv-unified`)은 이미 적용돼 있었고 도움이 안 됐습니다.

캐싱 메커니즘 자체가 못 쓰는 것처럼 보였기에 **대신 한 것**: Ollama를 업그레이드(0.32.14→0.34.2)했습니다. 이건 캐싱 버그 자체를 고치진 못했지만(새 버전에서도 동일한 로그가 재현됨), 처리량 자체는 실제로 개선됐습니다 — 프롬프트 재처리가 ~46→~105 tok/s로(8,190토큰 재처리가 약 178초 추정에서 78초로) 빨라졌습니다. 매 턴 재처리가 여전히 일어나긴 했지만, 그 재처리 자체가 더 빨라진 것입니다.

## 2일차: 결론이 정정됨

로그를 더 꼼꼼히 보며 재테스트하다가 1일차가 놓쳤던 걸 발견했습니다: Ollama 0.34.2의 llama-server는 표준 컨텍스트시프트가 꺼져 있음에도 **별도의 "컨텍스트 체크포인트" 캐시**를 돌리고 있었습니다(`context checkpoints enabled, max=32, min spacing=8192`, `prompt cache is enabled, size limit: 8192 MiB`).

- 턴1(콜드, 8,194토큰): 전체 재처리, 118.97초(68.9 tok/s) — 1일차 발견과 일치.
- 턴2, 턴1에 정확히 이어붙여 **같은 session-id**로 전송: `prompt_eval_cached_count: 8190/8194`, 재처리 시간 **0.322초**. 로그에 `checking checkpoint ... → restored context checkpoint`로 명시.
- 저장된 체크포인트 접두사와 조금이라도 안 맞는 요청은 즉시 전체 재처리로 폴백합니다.

**정정된 올바른 이해**: "하이브리드 아키텍처라서 캐싱이 불가능하다"가 아니었습니다. "캐싱은 작동하지만, 표준적인 점진적 컨텍스트시프트 메커니즘이 아니라 체크포인트-정확접두사일치 방식으로만 작동한다"는, 처음 내렸던 결론과는 다르고 더 좁은 전제조건이었습니다.

같은 날 밤 고정된 session-id로 3턴 연속 실사용 검증: 턴1(콜드, 74초, `forcing`) → 턴2(순수 후속질문, 26초, `restored`) → 턴3(실제 툴콜 포함, 툴콜판단 단계와 응답생성 단계 둘 다 `restored`). **대화에 툴콜이 섞여도 캐시가 깨지지 않음**을 확인 — 비하이브리드 모델로 교체하려던 기존 계획이 불필요했음이 드러났습니다.

**이번에 새로 드러난 진짜 전제조건**: 이 캐시 적중은 호출하는 쪽이 명시적으로 같은 `--session-id`를 고정해서 재사용했기 때문에만 일어납니다. 저희 자체 도구 중 하나가 (이와 무관한 이전 수정에서) 장시간 세션이 프롬프트를 무한정 불리는 걸 막으려고 매 호출마다 새 session-id를 자동발급하도록 기본값을 바꿔놨던 상태였습니다 — 그래서 이 기본값을 의도적으로 오버라이드하지 않는 호출자는, 이 캐싱 메커니즘이 완전히 정상 작동하는데도 매 호출마다 여전히 콜드스타트를 겪습니다.

## 왜 공유할 가치가 있는가

특정 모델과 무관하게 챙길 만한 두 가지:

1. llama.cpp/Ollama 로그에서 `"forcing full prompt re-processing due to lack of cache data (likely due to SWA or hybrid/recurrent memory...)"`를 보셨다면, 이 메시지는 *컨텍스트시프트*가 구체적으로 안 된다는 것에 대해서는 정확하지만, 그 모델에 캐싱 메커니즘이 *아예* 없다는 증거는 아닙니다. 아키텍처가 캐싱과 전반적으로 호환 안 된다고 결론 내리기 전에 별도의 체크포인트 기반 캐시가 있는지 먼저 확인하세요.
2. 체크포인트 캐시의 적중 조건(고정된 session-id에 대한 정확한 접두사 일치)은 고전적인 점진적 컨텍스트시프트(불어나는 대화에 이어붙이는 걸 더 느슨하게 허용)와는 의미있게 다른 계약입니다. 호출하는 코드가 무관한 이유로 session ID를 자동으로 돌려쓰고 있다면, 완전히 정상 작동하는 캐시 메커니즘을 그냥 못 쓰고 있을 수 있습니다.
