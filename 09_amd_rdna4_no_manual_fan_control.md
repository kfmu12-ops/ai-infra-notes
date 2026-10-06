# A brand-new AMD RDNA4 card with no working manual fan control in the driver

**Environment**: AMD RX 9060 XT (RDNA4, SMU firmware v14.0.0 — current at time of testing), Linux, amdgpu driver.

## The assumption that turned out wrong

"It's a current-generation card, manual fan control is a basic feature, this should just work." It didn't.

## What we tried

1. `fan1_enable` manual switch via the standard hwmon sysfs interface → failed immediately with `Invalid argument`.
2. Investigated further and found `amdgpu.ppfeaturemask` was masking out `PP_OVERDRIVE_MASK` — Overdrive (the feature gate for manual fan/clock control) wasn't enabled at the driver level at all.
3. Added the unmask to the GRUB kernel command line. It didn't take effect on the first reboot — turned out there were two EFI boot entries (one from the current NVMe install, one a leftover from an old SATA drive), and boot order was defaulting to the stale one. Fixed boot order with `efibootmgr -o`, rebooted again, confirmed the new kernel parameter was actually active this time.
4. With Overdrive now genuinely enabled: **manual fan control is still not implemented in the driver for this card.** Overdrive unlocks other things (clock/power limit overrides), but fan curve control specifically isn't there.

## Resolution

Gave up on software fan control for this card and went physical instead — a case-side-panel-mounted fan blowing directly on it.

## Why this might be useful to others

Two separable lessons:

- If `fan1_enable` fails with `Invalid argument` on a recent AMD card, don't assume you've misconfigured something — check whether Overdrive itself is masked (`amdgpu.ppfeaturemask`) before assuming the fan subsystem is broken, *and* separately verify your GRUB change actually took effect by checking `/proc/cmdline` after reboot, since a stale secondary EFI boot entry can silently make a correct kernel parameter never apply.
- Even after that's all correctly in place, manual fan curve control simply isn't guaranteed to be implemented for every current-generation AMD GPU — "it's new hardware" is not evidence it'll have this specific feature. Worth confirming before you sink time into the driver-level path at all.

---

# [한국어] 최신 AMD RDNA4 카드인데 드라이버에 수동 팬제어가 아예 구현이 안 되어 있던 사례

**환경**: AMD RX 9060 XT(RDNA4, SMU 펌웨어 v14.0.0 — 테스트 당시 최신), Linux, amdgpu 드라이버.

## 틀렸던 전제

"최신 세대 카드니까 수동 팬제어 정도는 기본 기능일 테니 그냥 될 것이다." 안 됐습니다.

## 시도한 것

1. 표준 hwmon sysfs 인터페이스로 `fan1_enable` 수동 전환 → 즉시 `Invalid argument`로 실패.
2. 더 조사해보니 `amdgpu.ppfeaturemask`가 `PP_OVERDRIVE_MASK`를 꺼놓고 있었습니다 — 수동 팬/클럭 제어의 기능 게이트인 오버드라이브가 드라이버 레벨에서부터 애초에 활성화돼 있지 않았습니다.
3. GRUB 커널 커맨드라인에 마스크 해제를 추가했습니다. 첫 재부팅에서는 반영이 안 됐는데 — 알고보니 EFI 부팅 항목이 2개(현재 NVMe 설치분/옛 SATA 드라이브 잔재)였고, 부팅 순서가 기본적으로 옛 쪽을 우선했습니다. `efibootmgr -o`로 부팅 순서를 정정하고 다시 재부팅해서야 새 커널 파라미터가 실제로 적용된 걸 확인했습니다.
4. 오버드라이브가 이제 정말로 활성화된 상태에서: **이 카드는 여전히 드라이버에 수동 팬제어가 구현이 안 되어 있었습니다.** 오버드라이브는 다른 것들(클럭/전력제한 오버라이드)은 풀어주지만, 팬커브 제어 자체는 그 안에 없었습니다.

## 결론

이 카드에 대한 소프트웨어 팬제어는 포기하고 물리적 방법으로 넘어갔습니다 — 케이스 옆판에 직결한 팬으로 직접 바람을 쏘는 방식입니다.

## 왜 공유할 가치가 있는가

따로 떼어낼 수 있는 교훈 두 가지:

- 최신 AMD 카드에서 `fan1_enable`이 `Invalid argument`로 실패한다면, 설정을 잘못했다고 바로 단정하지 마세요 — 팬 서브시스템이 고장났다고 생각하기 전에 오버드라이브 자체가 마스크돼 있는지(`amdgpu.ppfeaturemask`) 확인하고, **별도로** GRUB 변경이 실제로 적용됐는지 재부팅 후 `/proc/cmdline`으로 직접 확인하세요 — 낡은 보조 EFI 부팅 항목이 있으면 올바른 커널 파라미터가 조용히 전혀 적용 안 될 수 있습니다.
- 그 모든 게 제대로 갖춰진 뒤에도, 수동 팬커브 제어가 모든 최신 세대 AMD GPU에 반드시 구현돼 있다는 보장은 없습니다 — "신형 하드웨어니까"는 이 특정 기능이 있다는 증거가 아닙니다. 드라이버 레벨 경로에 시간을 쏟기 전에 먼저 확인해볼 가치가 있습니다.
