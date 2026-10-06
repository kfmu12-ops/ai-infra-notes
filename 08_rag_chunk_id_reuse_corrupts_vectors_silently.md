# A RAG vector store bug where reused `chunk_id`s silently attach the wrong embedding to the wrong text

## The setup

A RAG system where documents are split into chunks, each chunk gets a sequential `chunk_id`, and a separate `vectors.npy` array stores embeddings indexed by that same `chunk_id` (position in the array == chunk_id).

## The bug

`chunk_id` assignment is **not stable across reindexing** — it's just a running sequence number over however many chunks the current document set happens to produce. Adding or removing even a handful of documents shifts every `chunk_id` after that point. In one measured case, adding just 4 new conversation documents shifted `chunk_id` assignment for **16,606 chunks** starting from id 3,525 onward.

Because `vectors.npy` is stored in `chunk_id` order, a rebuild that reassigns IDs but doesn't fully regenerate every embedding from scratch leaves old embeddings sitting at new ID positions — **attaching the wrong vector to the wrong chunk of text.**

## Why this is worse than a normal bug

The corruption is *silent* at the embedding level: cosine similarity between the swapped vector and its new (wrong) neighbor isn't necessarily low. A later audit resampling 300 reassigned chunks found **96 (32%) were cosine-mismatched** between their stored vector and a freshly recomputed one for the same chunk — mean similarity 0.83, meaning many of these wrong pairings still looked "plausible enough" to a similarity threshold. This isn't a crash or an obviously broken retrieval result; it's a subtle widening of the search-result noise that's easy to attribute to "the model just isn't that accurate" instead of catching as a data-integrity bug.

This also wasn't a one-time incident — the same failure mode reappeared across several reindex events over roughly six weeks before the structural fix below was in place, each time looking like a fresh unrelated bug until the pattern was recognized.

## The fix

Stop keying vector storage on a positional, reassignable `chunk_id`. Instead:

1. Hash the chunk's actual text content and use that content hash as the stable cache key.
2. On reindex, look up each chunk by content hash — if the hash already has a cached vector, reuse it; if not, compute fresh.
3. Rebuild the live vector store atomically: build the new version fully in a staging location, validate it, then swap — never mutate the live store incrementally in place.

With content-hash keying, a `chunk_id` renumbering event can no longer misattach a vector, because the lookup key no longer depends on chunk order at all.

## Why this might be useful to others

If your RAG pipeline's chunk-to-vector mapping is keyed by any form of positional or sequential ID (chunk index, array offset, auto-increment row ID) rather than a hash of the chunk's actual content, you likely have the same latent bug — it just hasn't been triggered by a reindex yet, or has been and you haven't noticed because the symptom looks like ordinary retrieval noise rather than a clear failure.

---

# [한국어] RAG 벡터스토어에서 `chunk_id` 재사용이 조용히 벡터를 엉뚱한 본문에 연결시키는 버그

## 구조

문서를 청크로 나누고, 각 청크에 순차적인 `chunk_id`를 부여하고, 별도의 `vectors.npy` 배열에 그 `chunk_id`를 인덱스로 임베딩을 저장하는(배열 위치 == chunk_id) RAG 시스템입니다.

## 버그

`chunk_id` 배정은 **재인덱싱 사이에 안정적이지 않습니다** — 현재 문서 집합이 만들어낸 청크 개수에 따라 매겨지는 단순 순번일 뿐입니다. 문서 몇 건만 추가/삭제해도 그 이후의 모든 `chunk_id`가 밀립니다. 실측된 한 사례에서는 대화 문서 4건만 추가했는데 `chunk_id` 3,525번부터 **16,606개**의 청크 배정이 전부 밀렸습니다.

`vectors.npy`가 `chunk_id` 순서로 저장되기 때문에, ID는 재배정하면서 모든 임베딩을 처음부터 완전히 재생성하지는 않는 재빌드를 하면, 옛 임베딩이 새 ID 위치에 그대로 남아있게 됩니다 — **엉뚱한 벡터가 엉뚱한 텍스트 청크에 연결**되는 것입니다.

## 왜 평범한 버그보다 더 심각한가

이 오염은 임베딩 레벨에서는 *조용합니다* — 뒤바뀐 벡터와 그 새(틀린) 이웃 사이의 코사인 유사도가 반드시 낮게 나오는 게 아닙니다. 이후 재배정된 청크 300개를 표본감사했을 때 **96개(32%)**가 저장된 벡터와 그 청크를 새로 재계산한 벡터 사이에 코사인 불일치를 보였습니다 — 평균 유사도 0.83, 즉 이 틀린 짝지음 중 다수가 유사도 임계값 기준으로는 여전히 "그럴듯해 보일" 정도였습니다. 이건 크래시도 아니고 명백히 깨진 검색결과도 아닙니다 — 검색결과 노이즈를 미묘하게 넓히는 것뿐이라서, "모델이 그만큼 정확하지 않은가보다"로 오인하기 쉽고 데이터 정합성 버그로 포착하기 어렵습니다.

이것도 1회성 사고가 아니었습니다 — 아래의 구조적 해법이 자리잡기까지 약 6주에 걸쳐 여러 재인덱싱 이벤트에서 같은 실패 양상이 반복됐고, 매번 패턴을 알아차리기 전까지는 전혀 별개의 새 버그처럼 보였습니다.

## 해법

위치 기반·재배정 가능한 `chunk_id`로 벡터 저장소의 키를 삼는 걸 멈춥니다. 대신:

1. 청크의 실제 텍스트 내용을 해시해서 그 내용해시를 안정적인 캐시 키로 삼습니다.
2. 재인덱싱 시 각 청크를 내용해시로 조회합니다 — 그 해시에 이미 캐시된 벡터가 있으면 재사용하고, 없으면 새로 계산합니다.
3. 라이브 벡터스토어는 원자적으로 교체합니다: 스테이징 위치에서 새 버전을 완전히 빌드하고 검증한 뒤에만 교체합니다 — 라이브 스토어를 제자리에서 점진적으로 변경하지 않습니다.

내용해시 키 방식에서는 `chunk_id` 재번호 부여 이벤트가 더 이상 벡터를 잘못 연결시킬 수 없습니다 — 조회 키 자체가 청크 순서에 전혀 의존하지 않기 때문입니다.

## 왜 공유할 가치가 있는가

여러분의 RAG 파이프라인에서 청크-벡터 매핑이 (청크 인덱스, 배열 오프셋, auto-increment row ID 같은) 위치 기반·순차 ID로 키가 걸려 있고 청크 실제 내용의 해시가 아니라면, 아마 같은 잠재 버그를 갖고 있을 가능성이 높습니다 — 아직 재인덱싱으로 트리거되지 않았을 뿐이거나, 이미 트리거됐는데 증상이 명백한 실패가 아니라 그냥 평범한 검색 노이즈처럼 보여서 못 알아차렸을 뿐일 수 있습니다.
