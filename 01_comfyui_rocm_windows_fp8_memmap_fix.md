# Fixing a C++ crash (`THPStorage_assertNotNull`) when loading large fp8 safetensors in ComfyUI on Windows + ROCm

**Environment**: AMD RX 9060 XT (16GB VRAM), 32GB system RAM, Windows, ROCm 7.2.1, PyTorch 2.9.1+rocm7.2.1, ComfyUI 0.21.1.

## The problem

Loading large fp8 checkpoints (FLUX.2 dev, fp8mixed UNET at 35.5GB + a 12.3GB fp4_mixed CLIP) crashed ComfyUI with a native C++ access violation:

```
THPStorage_assertNotNull → NULL storage → access violation
```

Two things were stacking up against a 32GB machine:

1. The combined model size (~48GB across UNET + CLIP) doesn't fit in system RAM even before considering VRAM, so the loader was trying to materialize the full fp8 tensors in RAM before any offload logic kicked in.
2. On Windows, memory-mapping safetensors files above a certain size with the standard `mmap` approach was failing outright for 35GB+ files, which ComfyUI's existing fp8-safe loader path depended on.

## The fix

Rewrote ComfyUI's `_load_safetensors_fp8_safe()` (in `comfy/utils.py`, ~line 108–172) to load every tensor as a **read-only `numpy.memmap`** instead of reading it into a materialized torch storage:

```python
mm  = np.memmap(ckpt, dtype=np_dt, mode='r', offset=offset, shape=(n_elem,))
arr = mm.reshape(tuple(shape)) if shape else mm.reshape(())
t   = torch.from_numpy(arr)
if view_dt is not None:
    t = t.view(view_dt)
```

Key points:

- `mode='r'` memmap means pages are loaded **on demand** rather than materializing the whole tensor up front — no RAM spike for the full checkpoint.
- fp8 tensors are mapped as `uint8` and then `.view(torch.float8_e4m3fn)` — this reinterprets the bytes without allocating a new fp8-typed storage (which is where the original NULL-storage crash was happening).
- BF16 tensors follow the same pattern: `uint16` memmap → `.view(torch.bfloat16)`.
- The standard safetensors-library mmap path is bypassed entirely, since that's exactly what was failing on Windows for files this large.

## Result

With this patch, FLUX.2 dev fp8mixed (35.5GB UNET) + a 12.3GB fp4_mixed CLIP loaded and ran successfully on a 32GB-RAM machine with 16GB VRAM, using `--lowvram --bf16-unet`. A 17-image batch generation run completed end-to-end with **0 failures**, averaging 8–13 minutes per image over ~2h41m total.

## Why this might be useful to others

If you're running ComfyUI with large fp8/fp4 checkpoints on **Windows + ROCm** (as opposed to the much more common Linux + CUDA setup), you're in a sparsely-documented corner of the ecosystem. The combination of (a) Windows' mmap limitations on very large files and (b) fp8 storage handling quirks isn't something I found written up anywhere else, so sharing it here in case it saves someone a debugging session.

*(Originally found while generating weather-condition background art for a personal Android clock app; the patch itself is general-purpose and applies to any large fp8/bf16 safetensors checkpoint under similar constraints.)*

---

# [한국어] Windows + ROCm에서 ComfyUI의 대형 fp8 safetensors 로딩 시 C++ 크래시(`THPStorage_assertNotNull`) 해결

**환경**: AMD RX 9060 XT(16GB VRAM), 시스템 RAM 32GB, Windows, ROCm 7.2.1, PyTorch 2.9.1+rocm7.2.1, ComfyUI 0.21.1.

## 문제

대형 fp8 체크포인트(FLUX.2 dev, fp8mixed UNET 35.5GB + fp4_mixed CLIP 12.3GB)를 로드하면 ComfyUI가 네이티브 C++ 액세스 바이올레이션으로 크래시했습니다:

```
THPStorage_assertNotNull → NULL storage → access violation
```

32GB 메모리 환경에서 두 가지 문제가 겹쳤습니다:

1. UNET+CLIP 합계 약 48GB가 VRAM을 따지기 전부터 이미 시스템 RAM에 다 안 들어가는데, 로더가 오프로드 로직이 작동하기 전에 fp8 텐서 전체를 RAM에 올리려 시도했습니다.
2. Windows에서는 일정 크기 이상의 safetensors 파일을 표준 `mmap` 방식으로 매핑하는 것 자체가 35GB+ 파일에서 실패했는데, ComfyUI의 기존 fp8-safe 로더가 바로 이 방식에 의존하고 있었습니다.

## 해결 방법

ComfyUI의 `_load_safetensors_fp8_safe()`(`comfy/utils.py`, 약 108~172번째 줄)를 모든 텐서를 **읽기 전용 `numpy.memmap`**으로 로드하도록 재작성했습니다(완성된 torch storage로 읽어들이는 대신):

```python
mm  = np.memmap(ckpt, dtype=np_dt, mode='r', offset=offset, shape=(n_elem,))
arr = mm.reshape(tuple(shape)) if shape else mm.reshape(())
t   = torch.from_numpy(arr)
if view_dt is not None:
    t = t.view(view_dt)
```

핵심 포인트:

- `mode='r'` memmap은 텐서 전체를 미리 메모리에 올리는 대신 **필요할 때마다(on demand)** 페이지를 로드합니다 — 체크포인트 전체에 대한 RAM 스파이크가 없습니다.
- fp8 텐서는 `uint8`로 매핑한 뒤 `.view(torch.float8_e4m3fn)`으로 처리합니다 — 새 fp8 타입 storage를 할당하지 않고 바이트를 재해석만 하는 방식이라, 원래 NULL-storage 크래시가 나던 바로 그 지점을 피해갑니다.
- BF16 텐서도 같은 패턴입니다: `uint16` memmap → `.view(torch.bfloat16)`.
- 표준 safetensors 라이브러리의 mmap 경로는 완전히 건너뜁니다 — Windows에서 이 크기의 파일에 대해 바로 그 경로가 실패하고 있었기 때문입니다.

## 결과

이 패치 적용 후, FLUX.2 dev fp8mixed(35.5GB UNET) + fp4_mixed CLIP(12.3GB)이 RAM 32GB·VRAM 16GB 머신에서 `--lowvram --bf16-unet` 옵션으로 정상 로드·실행됐습니다. 17장 배치 생성이 **실패 0건**으로 완주했고, 장당 평균 8~13분(총 약 2시간 41분)이 걸렸습니다.

## 왜 공유할 가치가 있는가

ComfyUI에서 대형 fp8/fp4 체크포인트를 **Windows + ROCm**(훨씬 흔한 Linux + CUDA 조합이 아니라)으로 돌리고 있다면, 문서가 희박한 생태계의 구석에 있는 셈입니다. (a) Windows의 대형 파일 mmap 제약과 (b) fp8 스토리지 처리의 특이점이 겹친 이 조합은 다른 곳에서 정리된 걸 찾지 못해서, 누군가의 디버깅 시간을 아껴줄 수 있길 바라며 공유합니다.

*(개인용 안드로이드 시계 앱의 날씨별 배경 이미지를 생성하던 중 발견한 문제였지만, 패치 자체는 비슷한 제약을 가진 모든 대형 fp8/bf16 safetensors 체크포인트에 범용적으로 적용됩니다.)*
