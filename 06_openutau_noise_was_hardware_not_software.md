# A full night of debugging "OpenUtau audio noise" — it was the monitor's audio jack, not software

## The setup

Rendering Korean vocal synthesis through OpenUtau (sheet music → OMR → `.ustx` project → OpenUtau render, run under a headless Linux desktop with Xvfb for the GUI-only parts). Output audio consistently had audible noise.

## The wrong trail

Every reasonable software suspect got investigated: Xvfb's virtual display pipeline, OpenUtau's own audio rendering, PipeWire sample-rate handling, GPU audio routing. None of it panned out cleanly, and the noise kept reappearing.

## The actual cause

After an extended investigation, the noise turned out to have nothing to do with any of the software in the pipeline: **it was the machine's monitor DP/HDMI audio jack itself** — a hardware defect in that specific output path. Confirmed by playing the exact same file through two different outputs: noise present on the monitor's audio jack, completely clean through the motherboard's own speaker jack.

**Rule now in place: always audition rendered audio through the motherboard speaker jack, never through the monitor's.**

One secondary, actually-useful config decision that came out of the same investigation: PipeWire's multi-rate support (`10-rates.conf`) turned out to be a generally better default regardless of the noise issue, so it was kept; forcing a fixed GPU clock, by contrast, gave negligible benefit and was reverted.

## A few separable, concrete OpenUtau + Korean tips from the same project (useful on their own)

- **Phonemizer**: the default phonemizer silently produces silent output for Korean text. You need `DiffSingerKoreanG2PPhonemizer` explicitly.
- **Korean liaison rules (연음법칙) are not applied automatically** — OpenUtau's note-to-syllable mapping is strictly 1 syllable per note, so you have to pre-transform the lyric text into its liaison-resolved form yourself (e.g. "졸업하고" → "조러파고"). Five such patterns showed up in practice.
- **Gender/pitch parameter**: the slider named `gen` is not what actually changes the voice — the curve type `genc` is. Pushing `gen` up produced the *opposite* of the intended effect (thinner voice) in testing, so verify by ear in small increments rather than trusting the parameter name.
- **Key/pitch mismatches**: resolved reliably using `librosa`'s `chroma_cqt` cross-correlation to find the exact semitone shift needed, rather than guessing.

## Why this might be useful to others

The "spent hours debugging software that turned out to be a hardware fault" story is a classic for a reason — it's always worth testing the dumb hypothesis (swap the output device) early rather than last. If you're doing Korean vocal synthesis in OpenUtau specifically, the phonemizer/liaison/`genc` notes above will save you a few of the same dead ends.

---

# [한국어] "OpenUtau 노이즈" 밤샘 디버깅의 결말 — 소프트웨어가 아니라 모니터 오디오잭이었다

## 상황

악보(OMR로 인식) → `.ustx` 프로젝트 생성 → OpenUtau 렌더링으로 한국어 보컬 합성을 하는 파이프라인(GUI 전용 부분은 헤드리스 리눅스에서 Xvfb로 구동)이었습니다. 출력 오디오에 계속 들리는 노이즈가 있었습니다.

## 잘못된 길

합리적으로 의심할 만한 소프트웨어 요소는 전부 조사했습니다: Xvfb의 가상 디스플레이 파이프라인, OpenUtau 자체의 오디오 렌더링, PipeWire의 샘플레이트 처리, GPU 오디오 라우팅. 어느 것도 명확히 들어맞지 않았고 노이즈는 계속 재현됐습니다.

## 진짜 원인

긴 조사 끝에, 노이즈는 파이프라인의 어떤 소프트웨어와도 무관했습니다 — **이 PC 모니터의 DP/HDMI 오디오잭 자체의 하드웨어 결함**이었습니다. 같은 파일을 두 개의 다른 출력으로 재생해서 확인했습니다: 모니터 오디오잭에서는 노이즈가 있었고, 메인보드 자체 스피커잭으로는 완전히 깨끗했습니다.

**이제 정해진 규칙: 렌더링된 오디오를 들어볼 땐 항상 메인보드 스피커잭으로, 절대 모니터 잭으로는 하지 않는다.**

같은 조사에서 나온 부수적이지만 실제로 유용한 설정 결정 하나: PipeWire의 다중 샘플레이트 지원(`10-rates.conf`)은 이 노이즈 문제와 무관하게도 일반적으로 더 나은 기본값이라 유지했고, 반대로 GPU 클럭을 강제 고정하는 건 이득이 미미해서 원복했습니다.

## 같은 프로젝트에서 나온, 따로 떼어낼 수 있는 구체적인 OpenUtau + 한국어 팁 몇 가지

- **포네마이저**: 기본 포네마이저는 한국어 텍스트에서 조용히 무음 출력을 냅니다. `DiffSingerKoreanG2PPhonemizer`를 명시적으로 지정해야 합니다.
- **한국어 연음법칙은 자동 적용되지 않습니다** — OpenUtau의 음표-음절 매핑은 음표 하나당 음절 하나로 고정돼 있어서, 가사 텍스트 자체를 연음 결과로 미리 바꿔둬야 합니다(예: "졸업하고"→"조러파고"). 실전에서 이런 패턴이 5개 확인됐습니다.
- **성별/음높이 파라미터**: `gen`이라는 슬라이더가 실제로 목소리를 바꾸는 게 아니라, 커브 타입 `genc`가 그 역할을 합니다. 테스트에서 `gen`을 올렸더니 의도와 **반대로**(더 얇은 목소리) 나왔습니다 — 파라미터 이름을 믿지 말고 소량씩 조정하며 직접 들어보고 확인해야 합니다.
- **조성(키) 불일치**: 추측 대신 `librosa`의 `chroma_cqt` 상관분석으로 정확한 반음 이동량을 찾아 안정적으로 해결했습니다.

## 왜 공유할 가치가 있는가

"몇 시간 동안 소프트웨어를 디버깅했는데 알고보니 하드웨어 결함이었다"는 이야기는 괜히 클리셰가 된 게 아닙니다 — 마지막이 아니라 초반에 가장 단순한 가설(출력 장치를 바꿔보기)부터 시험해볼 가치가 늘 있습니다. OpenUtau로 한국어 보컬 합성을 하고 있다면, 위의 포네마이저/연음법칙/`genc` 팁들이 똑같은 막다른 길 몇 개를 건너뛰게 해줄 겁니다.
