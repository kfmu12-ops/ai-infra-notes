# A silent `<script>`-killing bug in Python triple-quoted HTML templates — and why your API tests will never catch it

If you serve a single-file Python web app where the whole frontend lives in a triple-quoted string (`INDEX_HTML = """...<script>...</script>..."""`), there's a specific escaping mistake that kills **the entire script block** while leaving every API endpoint working perfectly — which makes it unusually hard to notice.

## The bug

Inside a Python triple-quoted string, `\"` and `\n` are **Python** escape sequences, not literal characters that get passed through to the browser:

- `\"` is unescaped by Python into a literal `"` before the string ever reaches the template
- `\n` becomes an actual newline character, not the two characters `\` and `n`

If your JS inside that block was written assuming those sequences would survive into the browser's JS parser literally (e.g. JS string escapes inside a nested string literal), Python has already mangled them by the time the HTML is served. The browser receives syntactically broken JavaScript, the `<script>` tag throws a `SyntaxError`, and the entire script block dies — so **every button on the page silently stops responding**.

## Why it's so easy to miss

- `curl`/API testing shows everything is fine, because the backend endpoints are completely unaffected — only the client-side JS is broken.
- Grepping the *source* Python file for the escape sequences looks fine too, because the problem only appears after Python evaluates the string — the source text and the evaluated text are different, and the bug only exists in the latter.
- We confirmed this during an audit across six single-file web apps we'd built this way: **five of the six had been silently broken** for over a week before anyone noticed, because nothing in normal testing surfaces it.

## Detection that actually works

Don't grep the source. Actually `import` the module, pull out the evaluated `INDEX_HTML` string (post-Python-processing), extract the `<script>` contents, and run a real JS syntax check against *that*:

```python
import importlib
mod = importlib.import_module("app")
html = mod.INDEX_HTML  # the *evaluated* string, escapes already resolved
# extract the <script>...</script> body, write to a temp .js file
```
```bash
node --check extracted_script.js
```

This is now a standing step every time the embedded JS changes in any of these apps.

## Fix

Use single quotes for JS string literals inside the template (sidesteps the whole `\"` collision), or double-escape deliberately (`\\n`, `\\"`) when you do need a literal backslash-n or backslash-quote to survive into the browser.

## A related gotcha in the same family

`onclick="fn(${JSON.stringify(x)})"` has a similar failure mode: `JSON.stringify` emits double quotes, which collide with the HTML attribute's own double-quote delimiters and break the tag — even when the *content* being stringified doesn't itself contain quotes (the collision is with `JSON.stringify`'s own output quoting, not your data). Routing through a small helper that HTML-entity-escapes the quotes first (`JSON.stringify(s).replace(/"/g, "&quot;")`) avoids it.

## Why this might be useful to others

Single-file Python-served web apps (Flask/FastAPI with the whole frontend as an inline template string) are a common quick-prototyping pattern. If you've ever had a frontend go unresponsive with a perfectly healthy backend and no obvious cause, this is worth checking before assuming it's a logic bug — it leaves zero trace in normal API-level testing.

---

# [한국어] Python 삼중인용부호 HTML 템플릿에서 `<script>`를 조용히 죽이는 버그 — API 테스트로는 절대 못 잡는 이유

프론트엔드 전체가 삼중인용부호 문자열(`INDEX_HTML = """...<script>...</script>..."""`) 안에 들어있는 단일파일 Python 웹앱을 서빙하고 있다면, **스크립트 블록 전체**를 죽이면서도 모든 API 엔드포인트는 멀쩡하게 남겨두는 특정 이스케이핑 실수가 있습니다 — 그래서 알아차리기가 유난히 어렵습니다.

## 버그

Python 삼중인용부호 문자열 안에서 `\"`와 `\n`은 브라우저로 그대로 전달되는 리터럴 문자가 아니라 **Python의** 이스케이프 시퀀스입니다:

- `\"`는 템플릿이 브라우저에 도달하기도 전에 Python이 먼저 리터럴 `"`로 풀어버립니다
- `\n`은 문자 `\`와 `n` 두 개가 아니라 실제 개행 문자가 됩니다

그 블록 안의 JS가 이 시퀀스들이 브라우저의 JS 파서에 리터럴로 그대로 도달할 거라는 전제로 작성돼 있었다면(예: 중첩된 문자열 리터럴 안의 JS 이스케이프), HTML이 서빙되는 시점엔 이미 Python이 그걸 망가뜨린 뒤입니다. 브라우저는 문법이 깨진 JavaScript를 받고, `<script>` 태그가 `SyntaxError`를 던지며 스크립트 블록 전체가 죽습니다 — 그러면 **페이지의 모든 버튼이 조용히 반응을 멈춥니다**.

## 왜 이렇게 놓치기 쉬운가

- `curl`/API 테스트는 전부 정상으로 보입니다 — 백엔드 엔드포인트는 전혀 영향받지 않고, 클라이언트 쪽 JS만 깨지기 때문입니다.
- *소스* Python 파일을 그대로 grep해도 괜찮아 보입니다 — 문제는 Python이 문자열을 평가(evaluate)한 뒤에만 나타나는데, 소스 텍스트와 평가된 텍스트가 다르고 버그는 후자에만 존재하기 때문입니다.
- 이런 방식으로 만든 단일파일 웹앱 6개를 전수 점검하면서 이걸 확인했는데, **6개 중 5개가 일주일 넘게 조용히 고장난 상태**였습니다 — 일반적인 테스트로는 아무것도 드러나지 않았기 때문입니다.

## 실제로 작동하는 탐지법

소스를 grep하지 마세요. 모듈을 실제로 `import`해서 평가된 `INDEX_HTML` 문자열(Python 처리가 끝난 뒤)을 꺼내고, 그 안의 `<script>` 내용을 추출해 *그것*에 대해 실제 JS 문법 검사를 돌리세요:

```python
import importlib
mod = importlib.import_module("app")
html = mod.INDEX_HTML  # 평가된 문자열 — 이스케이프가 이미 풀린 상태
# <script>...</script> 본문을 추출해 임시 .js 파일로 저장
```
```bash
node --check extracted_script.js
```

이제 이 앱들에서 내장 JS를 고칠 때마다 상시로 거치는 단계가 됐습니다.

## 수정 방법

템플릿 안의 JS 문자열 리터럴에는 작은따옴표를 쓰거나(`\"` 충돌을 통째로 피함), 브라우저에 리터럴 백슬래시-n이나 백슬래시-인용부호가 정말로 필요할 때는 의도적으로 이중 이스케이프(`\\n`, `\\"`)하세요.

## 같은 계열의 관련 함정

`onclick="fn(${JSON.stringify(x)})"`도 비슷한 실패 양상이 있습니다: `JSON.stringify`가 내놓는 큰따옴표가 HTML 속성 자체의 큰따옴표 구분자와 충돌해 태그가 깨집니다 — 문자열화되는 *내용*에 따옴표가 전혀 없어도 발생합니다(충돌은 데이터가 아니라 `JSON.stringify`의 출력 자체 인용부호와 일어납니다). 따옴표를 먼저 HTML 엔티티로 이스케이프하는 작은 헬퍼(`JSON.stringify(s).replace(/"/g, "&quot;")`)를 거치면 피할 수 있습니다.

## 왜 공유할 가치가 있는가

프론트엔드 전체를 인라인 템플릿 문자열로 넣는 단일파일 Python 웹앱(Flask/FastAPI)은 흔한 빠른 프로토타이핑 패턴입니다. 백엔드는 멀쩡한데 프론트엔드가 뚜렷한 원인 없이 반응을 멈춘 적이 있다면, 로직 버그로 단정하기 전에 이걸 먼저 확인해볼 가치가 있습니다 — 일반적인 API 레벨 테스트에는 흔적이 전혀 남지 않기 때문입니다.
