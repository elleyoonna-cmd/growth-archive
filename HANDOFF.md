# 우리집 성장 기록 (Growth Archive) — 기술 인수 문서

아이의 말·생각·탐구 과정·작품·관심사·공식서류를 **가족이 함께** 쌓는 비공개 웹앱입니다.
“찍고 한 줄 적고 태그만 고르면, 나중에 주제별·시기별로 저절로 묶이는 타임라인”이 목표입니다.

- 라이브: `https://elleyoonna-cmd.github.io/growth-archive/` (저장소·Pages 준비 후)
- 저장소: `elleyoonna-cmd/growth-archive`
- 한 파일 앱: `index.html` 하나(HTML+CSS+JS 전부) + PWA 파일(`manifest.webmanifest`, 아이콘)
- 검색엔진 차단: `<meta name="robots" content="noindex, nofollow">`
- **실제 가족 정보는 코드·저장소에 넣지 않습니다.** 이름·생년월일은 기기 저장소(localStorage)와 가족 방(Firebase) 안에만 존재합니다.
- **`reading-garden`(독서기록장)과는 완전히 독립**입니다. 메뉴·연동·가져오기·로그인 통합이 전혀 없고, 같은 Firebase 프로젝트의 **다른 방**만 씁니다.

---

## 1. 화면 구조 (4탭)

| 탭 | 내용 |
|---|---|
| 🏠 홈 | 통합 타임라인 + `＋ 지금 남기기`(1분 기록) + 이어쓰기 + 검색·필터 |
| 🗺️ 모아보기 | 주제 지도(태그 자동 집계·이름 정리) / 돌아보기 타임라인(월·학기·연도) / 모음집(선택 PDF·인쇄) |
| 📁 서류함 | 학교·기관 공식서류 전용 공간(일상 기록과 분리). PIN 잠금 옵션 |
| ⚙️ 설정 | 아이 이름·생년월일, 가족 방 코드·동기화 상태, 서류함 PIN, ZIP/JSON 내보내기·복원, 전체 지우기 |

PC(≥900px)에서는 홈 타임라인이 2열, 하단 탭이 떠 있는 도크 형태가 됩니다. 모바일 우선 설계.

---

## 2. 폴더 구성

```
work/growth-archive/
  index.html               빌드 결과(배포 파일)
  manifest.webmanifest     PWA 매니페스트
  icon-192.png, icon-512.png
  build.py                 src 조각 → index.html 한 파일로 합침
  deploy.sh                빌드 + main/gh-pages 푸시
  HANDOFF.md               이 문서
  README.md                사용자용 안내
  src/
    index.template.html    템플릿 ( /*__CSS__*/ , /*__JS__*/ 자리 )
    styles.css             전체 CSS(모바일·PC·인쇄)
    app1_core.js           상태·저장·sanitize·라우팅·필터·공통 UI
    app2_sync.js           Firebase RTDB 실시간 동기화 + 덮어쓰기 방지 + 보호 방
    app3_media.js          사진 압축·브라우저 녹음·파일·ZIP 빌더/파서
    app4_views.js          홈·모아보기·서류함·설정 화면
    app5_actions.js        모든 동작(기록/이어쓰기/서류/모음집/백업·복원/설정)
    app6_boot.js           시작 코드 + 디버그 손잡이 window.__ga
  tests/
    serve.py               임시 정적 서버(테스트용)
    make_fixtures.py       가상 사진·PDF·큰 파일 생성(실제 가족 정보 아님)
    test_app.py            기능 E2E 132항목
    test_sync_guard.py     동기화·덮어쓰기 방지 E2E 39항목
    fixtures/              테스트용 가상 파일
  shots/                   검증 스크린샷
```

수정 후:

```sh
python3 work/growth-archive/build.py     # index.html 재생성
sh work/growth-archive/deploy.sh         # main + gh-pages 푸시
```

---

## 3. 데이터 모델 (state 하나)

`localStorage` 키: `growth-archive-v1` · 방 코드: `growth-archive-room` · 기기 ID: `growth-archive-device`

```js
state = {
  v: 1,
  profile: { childName: '', birthDate: 'YYYY-MM-DD' },   // 기기·가족 방에만
  settings: { docPin: '1234'|'', docsLocked: bool, collectionNote: '' },
  records: [{
    id, title,                     // 비우면 저장 시 "11월 15일 · 질문·탐구" 자동
    date: 'YYYY-MM-DD',
    type: '생각'|'질문·탐구'|'작품'|'협업',
    topics: ['로봇', ...],          // 자유 태그, 최대 10개, 24자
    childSay,                      // 아이 말 그대로
    parentNote,                    // 부모 메모(해석·도움) — 화면에서 항상 분리 표시
    helped: 'alone'|'helped'|'unknown',
    media: [Media],                // 사진/파일/녹음 최대 8개
    versions: [{ id, date, childSay, note, media, at }],   // 이어쓰기(다음 시도·달라진 생각)
    fav: bool, later: bool,        // 즐겨찾기 / 나중에 정리
    createdAt, updatedAt
  }],
  docs: [{
    id, name, issuer, period, issuedAt, location,
    status: '보유중'|'재발급필요'|'완료',
    note, media: [Media], createdAt, updatedAt
  }],
  deleted: { records: {id: ts}, docs: {id: ts} },   // 지운 항목 표식(되살아남 방지)
  updatedAt: 0                     // 동기화 충돌 판정 기준
}

Media = {
  id, kind: 'image'|'file'|'audio',
  name, mime, size, w, h, dur,
  data: 'data:image/webp;base64,...'   // 실제 파일(ZIP 내보내기에서는 path로 대체)
}
```

`sanitize()`가 외부/원격/복원 데이터를 화이트리스트 방식으로 검증합니다(길이 제한·타입 제한·최대 개수).
`loadLocal()`·`persist()`는 localStorage, `save()`는 저장 + 동기화 예약까지 담당합니다.

---

## 4. 가족 실시간 동기화 (Firebase RTDB)

- 프로젝트: `carbot-morning` (카봇 모닝 대시보드·독서기록장과 같은 프로젝트, **방은 분리**)
  - `databaseURL: https://carbot-morning-default-rtdb.asia-southeast1.firebasedatabase.app`
  - 경로: `rooms/<가족 방 코드>`
- **기본 방 코드는 없습니다.** 설정에서 코드를 정해야 연결됩니다(로컬 모드 기본). 코드는 `영문·숫자·-·_ 3~48자`, `?room=코드` URL로도 지정할 수 있습니다.
- 저장 형식: `{ v:1, updatedAt, by:<기기ID>, at, data:{profile, settings, records, docs} }`
- SDK: Firebase compat(`firebase-app-compat` + `firebase-database-compat`) → 실패 시 REST 폴링(7초).
- 충돌: `updatedAt`이 큰 쪽이 이깁니다. 수신 데이터는 `sanitize()` 후 적용.

### 덮어쓰기 방지 3중 안전장치 (`test_sync_guard.py`로 검증)

1. **빈 초기 상태 자동 업로드 차단** — 기록·서류·이름·생년월일이 하나도 없는 기기는 아무것도 올리지 않습니다(빈 방 유지).
2. **원격 빈 값 보호** — 방이 비었거나 초기값뿐이면 이 기기 기록을 지우지 않습니다. 기록이 있는 기기는 자기 기록을 다시 올립니다.
3. **`updatedAt` 비교 + 방 삭제 금지** — 더 새로운 원격만 적용하고, 방을 지우거나 임의로 되살리지 않습니다(`DELETE` 호출 없음).

4. **동시에 남긴 기록 보존(합집합 병합)** — 원격 상태를 받을 때 ID 기준으로 합칩니다. 같은 ID는 `updatedAt`이 더 새로운 쪽을 쓰고, 이 기기에만 있던 기록·서류는 지워지지 않습니다. 합쳐서 늘어났으면 이 기기가 그 합친 상태를 다시 올려 가족 방도 맞춥니다(`mergeById()`).
5. **삭제 전파** — 지운 기록·서류는 `state.deleted`에 표식(타임스탬프)을 남깁니다. 다른 기기는 그 표식보다 나중에 고쳐지지 않은 항목을 되살리지 않으므로, 한쪽에서 지우면 가족 모두에게서 사라집니다(1년 지난 표식은 정리).
6. **프로필·PIN은 이 기기 우선** — 이름·생년월일·서류함 PIN은 이 기기 값이 있으면 유지하고, 비어 있을 때만 원격 값을 받습니다.

추가 안전장치:

- **보호된 실사용 방**: `connor-family-main`, `reading-garden-main`, `connor-family`는 URL·설정 어디로 들어와도 연결을 거부합니다(`PROTECTED_ROOMS`). 테스트는 반드시 `pw-growth-*` 임시 방에서 하고 끝나면 REST DELETE로 지웁니다.
- **첨부 압축**: 사진은 긴 변 800px 이하 WebP(미지원 시 JPEG q0.72→0.5)로 줄여 업로드. 파일은 2.5MB, 녹음은 약 2.2MB(90초) 제한, 한 기록당 첨부 8개.
- **용량 안내**: 설정에 가족 방 데이터 크기(`payloadBytes`)와 4.2MB 권장 게이지를 표시하고, 초과 시 ZIP 백업을 권합니다(RTDB는 큰 사진을 계속 쌓기엔 적합하지 않음).
- 수동 **⬆ 지금 올리기 / ⬇ 가족 방에서 받기** 제공(받기는 확인 후 강제 적용).

---

## 5. 백업(ZIP)과 복원

`설정 → 전체 내보내기(ZIP)`은 **브라우저 안에서** 만듭니다(외부 라이브러리·만료 링크 없음, 순수 JS ZIP 빌더 `zipBuild`).

ZIP 구조(모두 `stored`, UTF-8 파일명):

```
growth-archive-YYYYMMDD/
  data.json        전체 데이터. 첨부는 data 대신 path만 담아 용량을 줄임
  records.csv      엑셀용(BOM 포함)
  documents.csv
  archive.html     앱 없이 읽는 사본(상대 경로로 attachments/·docs/ 참조)
  archive.md       같은 내용의 Markdown 사본(사진은 상대 경로 링크)
  README.md        복원 안내
  attachments/<recordId>/...   이어쓰기는 v1-1-... 형태
  docs/<docId>/...
```

- `data.json`의 Media는 `{kind,name,mime,size,path}` — 실제 파일은 같은 ZIP 안에 원본 바이트로 들어 있습니다.
- **복원(JSON/ZIP)**: `data.json`(또는 JSON 백업)을 읽고 ZIP이면 경로로 첨부를 되살립니다. 먼저 **미리보기 시트**에서 새 항목/중복 ID 수를 보여 주고, ① 새 항목만 병합(중복 ID 건너뜀) ② 전체 덮어쓰기(2차 확인) 중 선택합니다.
- JSON 백업은 첨부를 `data`로 그대로 담습니다(단일 파일 복원용).
- ZIP 파서는 `stored`와 `deflate-raw`(브라우저 `DecompressionStream`)를 지원합니다.

---

## 6. 모음집(선택 PDF/인쇄)

`모아보기 → 모음집` 또는 홈의 `📄 모음집 만들기`:

1. 기간(시작·끝), 주제, 유형 선택
2. 담을 내용 체크: 사진·아이 원문·부모 메모·이어쓰기 과정·파일 목록
3. `미리보기` → `#printRoot`에 인쇄용 문서 렌더 + `인쇄 / PDF 저장`

- 출력물 첫머리에 **“가족 작성 자료 · 평가·점수·공식 증빙이 아닙니다”** 를 고정 표기합니다.
- 인쇄 CSS(`@media print`)가 앱 UI를 숨기고 `#printRoot`만 인쇄합니다.
- 테스트는 미리보기 DOM을 검증하며 실제 `window.print()`는 호출하지 않습니다.

---

## 7. 서류함 PIN — 범위를 정확히 알 것

- PIN은 **화면 잠금**입니다. 데이터는 localStorage/가족 방에 그대로 있고, 개발자 도구나 기기 접근 앞에서 암호화되지 않습니다.
- PIN은 `state.settings.docPin`(4자리)로 저장되고 가족 방으로 동기화됩니다(가족 공용 잠금). 잊으면 설정에서 다시 정할 수 있습니다.
- 새로 열 때마다 잠깁니다(`UI.locked = !!docPin`).
- 진짜 암호화가 필요하면 WebCrypto 기반 암호화(비밀번호에서 키 유도)로 확장해야 합니다 — 아래 “다음 단계”.

---

## 8. 주요 함수 지도

| 함수 | 역할 |
|---|---|
| `sanitize()` / `sanitizeRecord()` / `sanitizeDoc()` | 외부 데이터 검증 |
| `save()` / `persist()` | 저장 + 동기화 예약 |
| `ViewHome/ViewMap/ViewDocs/ViewSettings` | 4탭 렌더 |
| `matchRecord()` / `filteredRecords()` | 키워드·유형·주제·기간·나이·첨부·즐겨찾기·나중 필터 |
| `allTopics()` / `groupPeriods()` | 주제 자동 집계 / 월·학기·연도 그룹 |
| `openQuick()` / `saveRecord()` | 1분 기록 |
| `openVersion()` / `saveVersion()` | 이어쓰기(버전) |
| `openRecord()` / `openCompare()` | 상세·나란히 비교 |
| `compressImage()` / `startRecording()` | 사진 800px 압축 / MediaRecorder 녹음 |
| `zipBuild()` / `zipParse()` / `exportZip()` / `restoreFrom()` | 백업·복원 |
| `collectionHtml()` / `previewCollection()` | 모음집 |
| `window.GASync` | `status, room, connect, uploadNow, pushNow, downloadNow, payloadBytes, restGet/restPut/restDelete` |
| `window.__ga` | 테스트용 손잡이(state, save, render, openQuick, exportZip, sanitize, …) |

이벤트는 `document` 위임(`data-act`) 하나로 처리합니다. 새 동작은 `ACT` 표에 추가하고, 버튼에 `data-act="이름"`을 답니다.

---

## 9. 테스트

```sh
export LD_LIBRARY_PATH=/workspace/.deblibs/usr/usr/lib/x86_64-linux-gnu:$LD_LIBRARY_PATH
python3 work/growth-archive/tests/make_fixtures.py     # 가상 사진·PDF 생성(필요 시)
python3 work/growth-archive/tests/test_app.py          # 132항목
python3 work/growth-archive/tests/test_sync_guard.py   # 54항목 (임시 방 pw-growth-*)
```

검증 범위:

- 1분 기록(사진만/글만), 자동 제목, 사진 800px·용량 확인
- 아이 말·부모 메모 분리, 유형·주제·혼자/도움
- 이어쓰기 추가·타임라인·나란히 비교
- 검색·필터(키워드/유형/기간/첨부/즐겨찾기/나중에 정리)
- 서류함 분리·상태·첨부·PIN 잠금/해제/새로 열기
- 주제 지도 집계·이름 정리(병합), 월·학기·연도 타임라인, 기간 모음집
- 모음집 미리보기(가족 작성 자료 표기·사진·원문·부모 메모, 옵션 제외)
- ZIP 내보내기 구조(data.json 경로 방식·CSV·HTML·README·원본 첨부) / ZIP·JSON 복원·중복 건너뛰기
- 2기기 실시간 동기화, 빈 기기 덮어쓰기 방지, 방 삭제 보호, 초기값 방 재업로드, updatedAt 충돌
- 보호 방 차단, 큰 파일 안내, 녹음 미지원 대체 안내, 가짜 마이크 녹음(시작/표시/미리듣기/다시 녹음)
- PC 반응형, 콘솔·페이지 에러 0건, 코드 내 실제 가족 정보 없음

최근 결과: **test_app 136/136 · test_sync_guard 54/54 통과**, 콘솔 에러 0건.

---

## 10. 알려진 한계 · 다음 단계

1. **인증 없음** — 방 코드를 아는 사람이 곧 가족입니다. RTDB 규칙이 공개 읽기/쓰기라면 코드는 충분히 길고 추측 불가능해야 합니다(예: `growth-family-x7k2m9`). 코드를 바꾸면 기존 방 데이터는 그대로 남습니다.
2. **PIN은 암호화가 아님**(7절 참조). 필요 시 WebCrypto AES-GCM + PBKDF2로 `docs`만 암호화하는 확장을 권합니다.
3. **첨부 용량** — RTDB 동기화는 사진 수십 장 규모에 적합합니다. 장기 보관은 ZIP 내보내기와 함께 운용하고, 4.2MB 게이지가 넘으면 정리를 안내합니다. 대용량화하려면 Firebase Storage/Supabase Storage로 옮기는 개편이 필요합니다.
4. **음성** — `MediaRecorder` 지원 브라우저에서 최대 90초. iOS Safari는 webm 대신 mp4로 녹음될 수 있어 `mime`을 그대로 저장합니다. 녹음 도중 시트를 닫거나 화면을 벗어나면 마이크를 끄고 그 녹음은 저장하지 않습니다.
5. **ZIP** — `stored` 방식이라 압축률은 없습니다(사진은 이미 압축됨). 대용량이면 `CompressionStream('deflate-raw')`를 빌더에도 붙일 수 있습니다.
6. **아이 계정 확장** — 지금은 부모 공용. 아이가 자라 직접 쓰게 하려면 방 코드 대신 계정·권한(읽기/쓰기/서류함) 계층이 필요합니다.
7. **다중 아이** — 현재 state는 한 아이 기준. 둘째를 위해 `children[]`로 확장하려면 `records.childId`를 추가하고 필터에 아이 선택을 넣습니다(마이그레이션: 기존 기록은 첫 아이).
8. **PWA 오프라인** — 서비스 워커를 넣지 않았습니다(캐시 갱신 문제 회피). 필요 시 network-first SW를 추가하세요.
9. **충돌 해결 범위** — 기록·서류는 항목 단위 합집합으로 보존되지만, 같은 기록을 두 기기에서 동시에 고친 경우에는 `updatedAt`이 더 새로운 쪽이 이깁니다(필드 단위 병합 아님). 프로필·설정은 이 기기 우선입니다.

---

## 11. 개인정보·보안 원칙 (반드시 유지)

- 실제 가족 정보(이름·생년월일·동네·기관)를 코드·저장소·테스트 픽스처·커밋 메시지에 넣지 않습니다.
- 테스트 데이터는 `tests/make_fixtures.py`의 가상 파일만 사용합니다.
- 실사용 방(`connor-family-main`, `reading-garden-main`)은 절대 읽거나 쓰지 않습니다.
- `noindex, nofollow` 유지. 공개 Pages 주소이므로 방 코드·백업 파일을 공개하지 않도록 안내합니다.
- ZIP 백업에는 아이 사진·음성·서류가 들어 있으므로 공유 링크·공개 저장소 업로드를 금지합니다.
