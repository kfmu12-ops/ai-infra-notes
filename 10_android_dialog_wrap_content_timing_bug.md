# Android `Dialog` with `WRAP_CONTENT` leaves dead whitespace below the content — a measure-timing bug, not a layout bug

## The symptom

A custom `Dialog` in portrait mode, with its window height set to `WRAP_CONTENT`, shows the first few lines of content correctly but then a large block of blank white space fills the rest of the screen below it — as if the dialog were sized to something much taller than its actual content.

## The actual cause (two compounding issues)

1. **Timing**: `dialog.window?.setLayout(width, WRAP_CONTENT)` was being called immediately after `dialog.show()`. At that exact point, the dialog's view hierarchy hasn't finished its measure/layout pass yet, so `WRAP_CONTENT` gets resolved against content that doesn't have its final size yet — the height ends up locked in before there's anything correct to wrap around.
2. **A `MATCH_PARENT` child inside a `WRAP_CONTENT` parent**: a touch-overlay view inside the dialog's content frame was set to `MATCH_PARENT` height, while its parent frame was `WRAP_CONTENT`. This is a layout-resolution cycle — the parent is trying to size itself to its content, while that content is simultaneously trying to size itself to the parent. Android doesn't resolve this sanely; it tends to fall back to a large default.

## The fix

Don't call `setLayout` immediately after `show()`. Wait for measurement to actually complete using a `ViewTreeObserver.OnGlobalLayoutListener`, then set the layout and immediately remove the listener (it fires on every subsequent layout pass otherwise):

```kotlin
root.viewTreeObserver.addOnGlobalLayoutListener(object : ViewTreeObserver.OnGlobalLayoutListener {
    override fun onGlobalLayout() {
        root.viewTreeObserver.removeOnGlobalLayoutListener(this)
        dialog.window?.setLayout(targetWidth, WindowManager.LayoutParams.WRAP_CONTENT)
    }
})
```

And separately: change the touch-overlay's height from `MATCH_PARENT` to `WRAP_CONTENT` so it's no longer in a sizing cycle with its own parent.

## The general rule worth keeping

**Never put a `MATCH_PARENT` child inside a `WRAP_CONTENT` parent.** It's not reliably resolvable, and the Android layout system doesn't error on it — it just picks something, and that something is usually wrong in a way that's easy to misdiagnose as "my content is too tall" rather than "my sizing constraints are circular."

## Why this might be useful to others

This exact symptom (dialog sized way bigger than its content, extra whitespace at the bottom, only in some orientations) is a recurring report in Android dialog implementations and usually gets attributed to the wrong layer (content height, padding, margins) rather than the actual root cause, which is almost always one of these two timing/cycle issues.

---

# [한국어] Android `Dialog`의 `WRAP_CONTENT`가 콘텐츠 아래에 빈 공백을 남기는 이유 — 레이아웃 문제가 아니라 측정 타이밍 문제

## 증상

세로모드의 커스텀 `Dialog`에서 창 높이를 `WRAP_CONTENT`로 설정했는데, 콘텐츠 앞 몇 줄은 정상적으로 보이지만 그 아래 화면 나머지를 채우는 큰 흰 공백 블록이 남습니다 — 다이얼로그가 실제 콘텐츠보다 훨씬 크게 잡힌 것처럼 보입니다.

## 실제 원인(두 가지가 겹침)

1. **타이밍**: `dialog.show()` 직후에 `dialog.window?.setLayout(width, WRAP_CONTENT)`를 호출하고 있었습니다. 바로 그 시점에는 다이얼로그의 뷰 계층이 아직 measure/layout 패스를 끝내지 않은 상태라, `WRAP_CONTENT`가 아직 최종 크기가 정해지지 않은 콘텐츠를 기준으로 해석됩니다 — 제대로 감쌀 대상이 생기기도 전에 높이가 고정돼버립니다.
2. **`WRAP_CONTENT` 부모 안의 `MATCH_PARENT` 자식**: 다이얼로그의 콘텐츠 프레임 안에 있던 터치 오버레이 뷰가 높이 `MATCH_PARENT`로 설정돼 있었는데, 그 부모 프레임은 `WRAP_CONTENT`였습니다. 이건 레이아웃 해석 순환 구조입니다 — 부모는 자기 콘텐츠에 맞춰 크기를 정하려 하는데, 그 콘텐츠는 동시에 부모에 맞춰 크기를 정하려 합니다. Android는 이걸 깔끔하게 풀어내지 못하고, 보통 크게 잡힌 기본값으로 떨어집니다.

## 수정

`show()` 직후에 바로 `setLayout`을 호출하지 마세요. `ViewTreeObserver.OnGlobalLayoutListener`로 측정이 실제로 끝나는 시점을 기다린 뒤 레이아웃을 설정하고, 리스너를 즉시 제거하세요(안 그러면 이후 모든 레이아웃 패스마다 계속 호출됩니다):

```kotlin
root.viewTreeObserver.addOnGlobalLayoutListener(object : ViewTreeObserver.OnGlobalLayoutListener {
    override fun onGlobalLayout() {
        root.viewTreeObserver.removeOnGlobalLayoutListener(this)
        dialog.window?.setLayout(targetWidth, WindowManager.LayoutParams.WRAP_CONTENT)
    }
})
```

그리고 별도로: 터치 오버레이의 높이를 `MATCH_PARENT`에서 `WRAP_CONTENT`로 바꿔서 자기 부모와의 크기 결정 순환에서 빠져나오게 합니다.

## 기억해둘 일반 규칙

**`WRAP_CONTENT` 부모 안에 `MATCH_PARENT` 자식을 절대 넣지 마세요.** 믿을 만하게 해석되지 않고, Android 레이아웃 시스템이 이걸 오류로 처리하지도 않습니다 — 그냥 뭔가를 골라서 적용하는데, 그 결과가 보통 틀려서 "내 콘텐츠가 너무 길다"처럼 엉뚱한 레이어의 문제로 오진하기 쉽습니다.

## 왜 공유할 가치가 있는가

이 정확한 증상(콘텐츠보다 훨씬 크게 잡힌 다이얼로그, 아래쪽 여분 공백, 특정 화면방향에서만 발생)은 Android 다이얼로그 구현에서 반복적으로 보고되는 문제인데, 보통 진짜 원인(이 두 가지 타이밍/순환 이슈 중 하나)이 아니라 엉뚱한 레이어(콘텐츠 높이, 패딩, 마진)의 문제로 오인되곤 합니다.
