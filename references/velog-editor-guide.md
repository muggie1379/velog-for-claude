# Velog 에디터 구조 & 조작 가이드 (실전 확인, 2026-09-13)

`mcp__claude-in-chrome__*`로 velog.io를 직접 조작할 때 필요한 정보를 모아둔다.
매번 새로 발견하지 않도록, 여기 없는 새로운 사실을 알아내면 이 파일에 추가한다.

## URL 패턴

- 글 목록: `https://velog.io/@<username>/posts`
- 시리즈 목록: `https://velog.io/@<username>/series`
- 특정 시리즈: `https://velog.io/@<username>/series/<시리즈명>` (예: `.../series/Workflow`)
- 글 보기: `https://velog.io/@<username>/<url-slug>`
- 새 글 작성: `https://velog.io/write`
- 기존 글 수정: `https://velog.io/write?id=<post-uuid>`
  - `id`는 글 읽기 페이지에서 "수정" 버튼을 눌러야 알 수 있다. 한 번 알아내면
    그 이후로는 이 URL로 바로 navigate해서 들어갈 수 있다 (수정 버튼을 다시
    찾아 클릭할 필요 없음 — 클릭이 씹히는 경우가 종종 있어서 이쪽이 더 안정적).

## 에디터는 CodeMirror 기반

왼쪽 마크다운 소스 창은 CodeMirror다. 전체 텍스트를 읽거나 쓸 때는 DOM 텍스트
추출(`get_page_text`/`read_page`) 대신 CodeMirror API를 직접 쓴다:

```js
const cm = document.querySelector('.CodeMirror').CodeMirror;
cm.getValue();          // 전체 마크다운 읽기
cm.setValue(newText);   // 전체 마크다운 통째로 교체
```

**왜**: CodeMirror는 가상 스크롤이라 화면에 안 보이는 줄은 `get_page_text`/
`read_page`로 못 읽는다. 오른쪽 미리보기 창은 가상 스크롤이 아니라서 전체가
잡히지만, 그건 렌더링된 HTML이지 원본 마크다운이 아니다.

**주의**: `cm.getValue()` 결과를 그대로 텍스트로 반환하면, 본문에 있는
쿼리스트링 붙은 URL(예: 유튜브 `?v=...`)이 "쿠키/쿼리스트링 데이터"로 오인돼
도구 응답 자체가 차단될 수 있다. 필요한 구간만 `.slice(start, end)`로 좁혀
읽거나, 반환 전에 `.replace(/\?[^\s)\]]*/g, '?REDACTED')`로 마스킹한다.

## 이미지 업로드: 클립보드 붙여넣기만 쓴다

툴바의 이미지 버튼(🖼️)을 클릭하지 않는다 — 자동화가 보지도 닫지도 못하는
네이티브 "파일 열기" 대화상자가 뜨고, `Escape`도 안 먹는다 (OS 레벨 창이라서).
한 번 열리면 사용자가 직접 "취소"를 눌러줘야 없어진다.

대신:

```powershell
Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing
$img = [System.Drawing.Image]::FromFile("<이미지 경로>")
[System.Windows.Forms.Clipboard]::SetImage($img)
```

으로 로컬 이미지를 클립보드에 올린 뒤, 에디터 안 아무 곳이나 클릭 →
(필요하면 `Ctrl+End`로 커서를 문서 끝으로) → `Ctrl+V`. 업로드는 몇 초 걸리니
2~3초 기다린 뒤 `cm.getValue()`로 방금 삽입된 `![](https://velog.velcdn.com/...)`
URL을 정규식으로 뽑아낸다.

여러 장을 특정 위치에 넣어야 하면: 매번 정밀 클릭으로 커서를 맞추려 하지 말고,
(1) 문서 끝에 순서대로 붙여넣기 → (2) 그때그때 URL만 기록 → (3) 마지막에
`cm.getValue()`로 전체를 가져와 JS 문자열 치환으로 원하는 위치에 배치한 뒤
`cm.setValue()`로 한 번에 반영한다.

## 본문 전체를 한 번에 바꿀 때는 클립보드 대신 `cm.setValue()`를 쓴다

긴 텍스트(예: SKILL.md 전문을 게시글에 통째로 반영)를 넣을 때 "PowerShell로
클립보드에 텍스트를 올리고 에디터에서 Ctrl+A → Ctrl+V" 방식을 시도했는데,
**클립보드에 새 내용을 확실히 세팅했다고 검증까지 했는데도, 붙여넣기 결과가
훨씬 이전의 오래된 내용으로 나온 사고가 있었다** (원인 불명 — 다른 프로세스가
그 사이에 클립보드를 건드렸거나, 자동화가 보내는 `Ctrl+V` 키 이벤트가 실제
OS 클립보드에서 읽어오는 게 아니라 페이지/브라우저의 다른 캐시된 값을 쓰는
것으로 추정된다). 여러 번 재시도해도 같은 문제가 재현됐다.

**결론**: 본문을 통째로 교체해야 하면 클립보드/Ctrl+V에 의존하지 말고,
`javascript_tool`로 `cm.setValue(text)`를 직접 호출한다. 이 방법은 매번
정확하게 동작했다. 텍스트에 작은따옴표가 섞여 있어서 JS 문자열 리터럴로
옮기기 까다로우면, 아래처럼 Node로 먼저 JSON 인코딩해서 이스케이프 문제를
피한다 (JSON 문자열 리터럴은 그대로 JS 문자열 리터럴로도 유효하다):

```bash
node -e "
const fs = require('fs');
const content = fs.readFileSync('원본.txt', 'utf8');
fs.writeFileSync('encoded.txt', JSON.stringify(content));
"
```

그 다음 `encoded.txt`의 내용(따옴표 포함 전체)을 그대로
`const body = "...";` 자리에 붙여넣고 `cm.setValue(body)`를 호출하면 된다.
클립보드 이미지 붙여넣기(위 "이미지 업로드" 절)는 이 문제를 겪지 않았다 —
문제는 유독 **큰 텍스트를 클립보드 텍스트로 올릴 때**만 재현됐다.

## `<details>` 접이식 섹션은 안 먹는다

에디터 미리보기(오른쪽 창)에서는 `<details><summary>...</summary><div>...내용...
</div></details>` 형태가 실제로 접이식으로 렌더링되는 것처럼 보인다. **하지만
실제 발행된 페이지는 `<details>`/`<summary>`/`</details>` 태그 자체를 렌더링
시 제거한다** (`<div>`는 남기고, 안의 텍스트는 평범한 문단으로 노출됨). 즉
에디터 미리보기와 발행 후 실제 렌더링이 다르므로, 미리보기만 보고 판단하면 안
된다 — 반드시 발행 후 실제 글 URL에서
`document.querySelectorAll('details').length`로 확인한다.

**결론**: velog 마크다운으로는 접이식 섹션을 구현할 방법이 없다. 상세 내용은
`####` 소제목으로 구분하는 것으로 대체한다 (항상 펼쳐진 채로 보이지만,
소제목 덕분에 관심 없는 독자는 훑고 지나갈 수 있다).

## 저장/발행 흐름 (2단계 확인 필요)

"수정하기"(기존 글) / "출간하기"(새 글) 버튼은 **누른다고 바로 저장되지
않는다**. 흐름:

1. 버튼 클릭 → 임시저장되면서 "공개 설정 / URL 설정 / 시리즈 설정"을 확인하는
   패널이 뜬다 (같은 페이지 안에서, 새 팝업 아님).
2. 시리즈가 다르면 "시리즈에 추가하기" → 목록에서 선택 → "선택하기".
3. 그 패널 안에 있는 **두 번째** "수정하기"/"출간하기" 버튼을 눌러야 실제로
   확정된다. `find`로 버튼을 찾으면 같은 텍스트의 버튼이 2개(ref) 나오는데,
   나중에 나온(더 큰 ref 번호) 쪽이 이 확정 버튼이다.
4. 확정되면 글 읽기 페이지(`/@username/slug`)로 자동 이동한다 — 이게 저장
   완료의 신호다.

**주의**: 새로고침/재-find 없이 오래된 `ref`로 클릭하면 조용히 씹히는
경우가 있었다 (에디터가 리렌더링되면서 ref가 무효화됨). 클릭이 안 먹은 것
같으면 `find`를 다시 호출해서 새 ref를 받아온다.

## 시리즈 설정 UI

"시리즈 설정" 영역 클릭 → 기존 시리즈 목록(카드 형태)이 뜬다 → 원하는 시리즈
카드를 클릭해서 선택(하이라이트됨) → "선택하기" 버튼으로 확정. 새 시리즈를
만들 땐 같은 패널 위쪽의 "새로운 시리즈 이름을 입력하세요" 입력창에 타이핑.

## 제목/태그 입력 시 주의

새 글 작성 페이지(`/write`, id 없음) 진입 직후 `find`로 얻은 title/tag
textbox의 `ref`가 실제로는 반응하지 않는 경우가 있었다(페이지가 완전히
자리잡기 전에 얻은 stale ref로 추정). 타이핑했는데 스크린샷에서 placeholder가
그대로 보이면, 좌표 클릭(`computer` action `left_click` with `coordinate`)으로
다시 시도한다. 제목 입력창은 대략 화면 상단 좌측 큰 텍스트 자리, 태그
입력창은 그 바로 아래 줄이다 — 정확한 좌표는 스크린샷으로 그때그때 확인한다.
