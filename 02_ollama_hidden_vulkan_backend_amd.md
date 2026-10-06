# Ollama has an undocumented Vulkan backend that fixed our AMD RDNA4 hard-freezes (and was ~2x faster than ROCm)

**Environment**: AMD RX 9060 XT (RDNA4, gfx1200), Ollama, Linux.

## The problem

Running a 30B-class vision model (`qwen3-vl:30b`) via Ollama's default ROCm backend caused **silent hard freezes** — no OOM message, no kernel panic, no log entry at all. The machine just died mid-inference. This wasn't a one-off; it was a recurring category of crash on this specific RDNA4 card.

## The discovery

Running `strings` against the Ollama binary turned up two things that don't appear in Ollama's public docs or `--help` output:

- An env var: `OLLAMA_VULKAN`
- A linked library: `libggml-vulkan.so`

Ollama ships with a Vulkan backend baked in — it's just not surfaced anywhere you'd normally look for backend selection.

### Getting it to actually activate

Setting `OLLAMA_VULKAN=1` alone did **nothing** — ROCm still took priority whenever it was available. The backend only switches over when ROCm is explicitly hidden from the process:

```bash
HIP_VISIBLE_DEVICES=-1
ROCR_VISIBLE_DEVICES=-1
```

With both of those set, Ollama's logs show it picking up the Vulkan backend as `RADV GFX1200` (Mesa's RADV driver).

## Measured results (same model, same prompt, same machine)

| Metric | ROCm | Vulkan |
|---|---|---|
| Speed | 121–145s | **71s** (~2x faster) |
| Stability | Hard freeze (1 occurrence in the test run) | No freeze |
| Accuracy (Korean place-name OCR) | Misread ("경산 전량을") | Correct ("경산 진량읍") |

All three axes improved simultaneously — this wasn't a speed/stability tradeoff.

## Rollout

After confirming reproducibility across three different models, we moved this to a permanent systemd override rather than per-invocation env vars. One scope note: **ComfyUI has no Vulkan backend** (confirmed against ComfyUI's own repo/issues), so this only applies to Ollama — the ComfyUI pipeline on the same machine stays on ROCm.

We also later tested mixing an old GTX 1060 into the same Vulkan pool (`CUDA_VISIBLE_DEVICES=-1` to force it into Vulkan too) — Ollama recognized it as a second device (`Vulkan0` = 9060 XT, `Vulkan1` = 1060) and pooled ~22GB of combined VRAM, letting a 21GB model run that didn't fit on the 9060 XT alone. The 1060 is old enough that it didn't add meaningful *speed*, but it did let a bigger model fit.

## Why this might be useful to others

AMD RDNA4 + ROCm is still a thin-coverage combination — community validation and troubleshooting threads are much sparser than for NVIDIA/CUDA. If you're hitting unexplained hard freezes on a newer AMD card with Ollama's default ROCm path, it's worth trying the Vulkan backend before assuming it's a hardware or driver-stability issue — in our case it wasn't.

---

# [한국어] Ollama의 문서화되지 않은 Vulkan 백엔드 — AMD RDNA4 하드프리즈를 해결하고 ROCm보다 약 2배 빨랐던 사례

**환경**: AMD RX 9060 XT(RDNA4, gfx1200), Ollama, Linux.

## 문제

30B급 비전 모델(`qwen3-vl:30b`)을 Ollama의 기본 ROCm 백엔드로 돌리면 **무기록 하드프리즈**가 발생했습니다 — OOM 메시지도, 커널패닉도, 로그 한 줄도 없이 그냥 추론 중에 죽었습니다. 한 번뿐이 아니라, 이 특정 RDNA4 카드에서 반복적으로 재현되는 크래시 유형이었습니다.

## 발견 경위

Ollama 바이너리에 `strings`를 돌려보니 공식 문서나 `--help`에는 전혀 안 나오는 두 가지가 나왔습니다:

- 환경변수 `OLLAMA_VULKAN`
- 링크된 라이브러리 `libggml-vulkan.so`

Ollama에는 Vulkan 백엔드가 이미 내장돼 있었습니다 — 다만 백엔드 선택과 관련해 보통 찾아볼 만한 어디에도 드러나 있지 않았을 뿐입니다.

### 실제로 활성화시키기

`OLLAMA_VULKAN=1`만 설정하면 **아무 변화가 없었습니다** — ROCm이 사용 가능하면 항상 우선권을 가져갔습니다. ROCm을 프로세스에서 명시적으로 숨겨야만 백엔드가 전환됐습니다:

```bash
HIP_VISIBLE_DEVICES=-1
ROCR_VISIBLE_DEVICES=-1
```

이 둘을 설정하면 Ollama 로그에 Vulkan 백엔드가 `RADV GFX1200`(Mesa의 RADV 드라이버)으로 잡힌 게 보입니다.

## 실측 결과(같은 모델, 같은 프롬프트, 같은 머신)

| 지표 | ROCm | Vulkan |
|---|---|---|
| 속도 | 121~145초 | **71초** (약 2배 빠름) |
| 안정성 | 하드프리즈(테스트 중 1회 발생) | 프리즈 없음 |
| 정확도(한국어 지명 OCR) | 오독("경산 전량을") | 정확("경산 진량읍") |

세 지표가 동시에 전부 개선됐습니다 — 속도와 안정성 사이의 트레이드오프가 아니었습니다.

## 적용

세 가지 다른 모델로 재현성을 확인한 뒤, 매번 환경변수를 넣는 대신 영구 systemd override로 전환했습니다. 참고할 범위 제한 하나: **ComfyUI에는 Vulkan 백엔드가 없습니다**(ComfyUI 자체 저장소/이슈로 확인) — 그래서 이건 Ollama에만 적용되고, 같은 머신의 ComfyUI 파이프라인은 그대로 ROCm을 씁니다.

이후 오래된 GTX 1060을 같은 Vulkan 풀에 섞어보기도 했습니다(`CUDA_VISIBLE_DEVICES=-1`로 1060도 Vulkan에 편입) — Ollama가 이를 별도 장치(`Vulkan0`=9060 XT, `Vulkan1`=1060)로 인식해 약 22GB의 통합 VRAM 풀로 묶었고, 9060 XT 단독으로는 못 돌리던 21GB 모델을 완주시켰습니다. 1060이 구형이라 의미 있는 *속도* 향상은 없었지만, 더 큰 모델을 돌릴 수 있게는 해줬습니다.

## 왜 공유할 가치가 있는가

AMD RDNA4 + ROCm은 아직 커뮤니티 검증 사례가 얕은 조합입니다 — NVIDIA/CUDA에 비해 트러블슈팅 스레드 자체가 훨씬 적습니다. 신형 AMD 카드에서 Ollama 기본 ROCm 경로로 원인을 알 수 없는 하드프리즈를 겪고 있다면, 하드웨어나 드라이버 안정성 문제로 단정하기 전에 Vulkan 백엔드를 먼저 시도해볼 만합니다 — 저희 경우엔 하드웨어 문제가 아니었습니다.
