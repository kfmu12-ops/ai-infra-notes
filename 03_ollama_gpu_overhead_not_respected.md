# Confirming `OLLAMA_GPU_OVERHEAD` doesn't actually reserve VRAM (and what worked instead)

Related upstream issue: [ollama/ollama#12223 "OLLAMA_GPU_OVERHEAD is not respected"](https://github.com/ollama/ollama/issues/12223)

## What we tried

We wanted a VRAM safety margin so a model running in Ollama wouldn't contend with normal desktop GPU usage (browser tabs, etc.) on the same card. Ollama exposes `OLLAMA_GPU_OVERHEAD` for exactly this — reserve N bytes of VRAM that the model allocator won't touch.

## What we measured

Raising `OLLAMA_GPU_OVERHEAD` from 2GB to 8GB made **no difference** — the model still filled the card to 23.77GB/24GB. Ollama's own logs show it *computing* the reduced available-VRAM figure correctly ("overhead applied, available X GB"), but the actual allocation at model-load time ignores that figure and fills the card anyway. This matches the symptom in #12223 exactly — we're just adding a second independent confirmation with specific before/after numbers (2GB→8GB, no change, 23.77/24GB observed either way).

## What we used instead

Since the env var doesn't give you a reliable reservation, we solved the actual problem (desktop apps competing with the model for VRAM) from the other side — stop desktop apps from touching the GPU at all:

**Chrome** — `/etc/opt/chrome/policy/managed/*.json`:
```json
{"HardwareAccelerationModeEnabled": false}
```

**Firefox (snap)** — `/etc/firefox/policies/policies.json`:
```json
{
  "policies": {
    "Preferences": {
      "layers.acceleration.disabled": { "Value": true, "Status": "locked" }
    }
  }
}
```

Measured VRAM usage with these applied (idle tabs, no model running): Chrome ~3GB → 0, Firefox 350MB → 85MB. With both browsers GPU-accelerated off, a model loaded at 100% GPU alongside normal multi-tab browsing without contention.

If you specifically need deterministic VRAM headroom *within* a model's own footprint (not from other processes), the more reliable lever is forcing some layers onto CPU via a Modelfile:
```
PARAMETER num_gpu <N>
```
This is deterministic (you're choosing the split directly) at the cost of some inference speed, rather than relying on an overhead reservation that silently doesn't apply.

## Why this might be useful to others

If you've set `OLLAMA_GPU_OVERHEAD` and are still seeing VRAM-contention crashes, it's not your configuration — the setting doesn't do what its name says, as of this testing. Browser hardware-acceleration policies are a more reliable way to get your VRAM headroom back if the contention is coming from desktop apps rather than another model.

---

# [한국어] `OLLAMA_GPU_OVERHEAD`가 실제로 VRAM을 예약하지 않는다는 재현 확인 + 대안

관련 업스트림 이슈: [ollama/ollama#12223 "OLLAMA_GPU_OVERHEAD is not respected"](https://github.com/ollama/ollama/issues/12223)

## 시도한 것

Ollama로 모델을 돌릴 때 같은 카드의 일반 데스크톱 GPU 사용(브라우저 탭 등)과 경합하지 않도록 VRAM 안전마진을 두고 싶었습니다. Ollama는 바로 이 용도로 `OLLAMA_GPU_OVERHEAD`를 제공합니다 — 모델 할당기가 건드리지 않을 N바이트의 VRAM을 예약하는 설정입니다.

## 실측 결과

`OLLAMA_GPU_OVERHEAD`를 2GB→8GB로 올려도 **아무 차이가 없었습니다** — 모델은 여전히 카드를 23.77GB/24GB까지 채웠습니다. Ollama 자체 로그는 줄어든 가용 VRAM 수치를 정확히 *계산*은 하고 있었지만("오버헤드 적용, 가용 X GB"), 실제 모델 로드 시점의 할당은 그 수치를 무시하고 카드를 그대로 채워버렸습니다. 이건 #12223의 증상과 정확히 일치합니다 — 저희는 구체적인 전후 수치(2GB→8GB, 변화 없음, 양쪽 다 23.77/24GB 관측)로 독립적인 재현 확인을 하나 더 추가하는 셈입니다.

## 대신 사용한 방법

환경변수가 믿을 만한 예약을 주지 못하니, 실제 문제(데스크톱 앱이 모델과 VRAM을 다투는 것)를 반대쪽에서 풀었습니다 — 데스크톱 앱이 아예 GPU를 건드리지 못하게 막는 방식입니다:

**Chrome** — `/etc/opt/chrome/policy/managed/*.json`:
```json
{"HardwareAccelerationModeEnabled": false}
```

**Firefox(snap)** — `/etc/firefox/policies/policies.json`:
```json
{
  "policies": {
    "Preferences": {
      "layers.acceleration.disabled": { "Value": true, "Status": "locked" }
    }
  }
}
```

이 설정을 적용한 뒤 실측한 VRAM 사용량(유휴 탭, 모델 미실행 상태): Chrome 약 3GB→0, Firefox 350MB→85MB. 두 브라우저의 GPU 가속을 모두 끈 뒤에는, 모델이 GPU를 100% 점유한 채로 평소처럼 여러 탭을 켜놓고 브라우징해도 경합이 없었습니다.

다른 프로세스가 아니라 **모델 자신의 점유량 안에서** 결정론적인 VRAM 여유가 필요하다면, 더 믿을 만한 방법은 Modelfile로 일부 레이어를 CPU로 강제하는 것입니다:
```
PARAMETER num_gpu <N>
```
이건 결정론적(분배를 직접 선택)이라는 장점이 있지만 추론 속도를 일부 희생합니다 — 조용히 적용 안 되는 오버헤드 예약에 기대는 것보다는 확실합니다.

## 왜 공유할 가치가 있는가

`OLLAMA_GPU_OVERHEAD`를 설정했는데도 여전히 VRAM 경합 크래시가 난다면, 설정이 잘못된 게 아닙니다 — 이번 테스트 기준으로 이 설정은 이름이 말하는 대로 작동하지 않습니다. 경합의 원인이 다른 모델이 아니라 데스크톱 앱이라면, 브라우저의 하드웨어 가속 정책을 끄는 쪽이 VRAM 여유를 되찾는 더 확실한 방법입니다.
