# Make → n8n 이전 (2026-09-20)

## 결론부터: 이번 주 안에 끝내는 것은 가능하다

인계문서(`inseoul______0920.md`) §13이 이미 n8n 이전을 권장하는 판단까지 내려놓았고,
막혀 있던 건 "언제·누가 배선을 다시 만드나"뿐이었다. 이 폴더가 그 배선이다 —
Make 시나리오 8개를 n8n 워크플로 8개로 옮겨서 바로 임포트할 수 있게 만들어 두었다.

**이전 가능한 이유(§13-5의 표 그대로):**
- 프롬프트 전문(사실조사·KR 원고·EN 원고) — 원본 Make Body JSON을 그대로 파싱해서 옮김. 재입력 아님.
- 카드 HTML/CSS + 확정 수치 — 그대로 이식.
- 시트 구조·상태값·로직 흐름 — 손대지 않음.
- **배선만 새로 만들면 됐고, 그 배선을 이번에 다 만들었다.**

남은 일은 "만들기"가 아니라 "연결하고 실측하기"다. 아래 체크리스트대로 하면
반나절~하루 안에 1건을 전 구간 통과시킬 수 있고, 그러면 이번 주 안에 끝난다.

---

## 이 폴더의 구조

```
automation/
├── README.md                  ← 이 파일
├── n8n-workflows/              8개 워크플로 JSON (n8n에 바로 임포트)
│   ├── 01-batch-submit.json    inseoul_배치제출  대체
│   ├── 02-facts-receive.json   inseoul_사실수령  대체
│   ├── 03-draft-receive.json   inseoul_원고수령  대체
│   ├── 04-photo-collect.json   inseoul_사진수집  대체
│   ├── 05-render.json          inseoul_렌더      대체
│   ├── 06-publish.json         inseoul_발행      대체
│   ├── 07-en-submit.json       inseoul_EN제출    대체
│   └── 08-en-receive.json      inseoul_EN수령    대체
├── prompts/                    프롬프트 전문 (사람이 읽는 원본 — 워크플로 JSON 안에도
│                                동일한 텍스트가 들어 있지만, 여기 따로도 보관한다)
└── templates/
    └── card-template.html      카드 HTML/CSS 원본 (05-render.json의 Code 노드가 이 구조를 품고 있음)
```

**중요한 차이 — Make와 달리 이 8개는 지금 이 저장소 안에 있다.** 인계문서 §6이
경고했던 "Body가 유일한 원본이 되는 상황"이 이제는 안 생긴다. 프롬프트·템플릿·
워크플로가 모두 git으로 버전 관리된다.

---

## Make → n8n, 무엇이 그대로고 무엇이 바뀌었나

| 항목 | Make | n8n (이 폴더) |
|---|---|---|
| 시트 값 참조 | 배지 클릭 필수, 타이핑하면 무음 실패 | `row['사진URL']` 처럼 그냥 코드로 참조 |
| `map`/`join`/`split` | 함수 패널 클릭 삽입, 따옴표 규칙이 반대로 알려져 있었음 | 표준 JavaScript (`Array.map/join/split`) |
| 컬럼 레인지 A1:Z1 | 26열 넘으면 모듈 12개 전부 수정 | 해당 개념 없음 |
| 모듈 번호 추적 (`20.photos`) | 사람이 직접 추적 | 노드 **이름**으로 참조 (`$('노드이름')`) |
| 카드 HTML 조립 | 1행 압축, 큰따옴표 금지, JSON 이스케이프 수작업 | Code 노드가 JS 객체로 만들고 HTTP 노드가 자동 직렬화 — 제약 없음 |
| 발행 폴링 | Repeater 4회를 무조건 다 기다림(조기 종료 불가) | If 노드로 FINISHED면 즉시 진행, 아니면만 재시도 |
| 오류 없는 실패 | 필터에서 0건이어도 Success로 표시 | 실행 로그에서 각 노드 입출력이 그대로 보임 |
| 실행 크레딧 | 모듈 1개 실행 = 1 크레딧, Free 월 1,000 | 자체 호스팅이면 무제한 |
| Meta 토큰 하드코딩 (§10-C 미해결 과제였음) | 시나리오 3곳에 직접 입력 | `$env.META_ACCESS_TOKEN` 한 곳만 — **이 이전으로 자동 해결됨** |

바뀌지 않는 것도 명확히 적어둔다 (§13-4 그대로): **사진 조달은 여전히 사람 손**이고,
Anthropic 배치 대기(20~40분)도 그대로다. 도구를 바꿔도 이 둘은 안 바뀐다.

---

## 시작하기 전 준비물

### 1. n8n 인스턴스
문서 §13-7 1단계 그대로 — 우선 로컬에서 시험:
```
docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n n8nio/n8n
```
`http://localhost:5678` 접속. 시험이 끝나고 상시 운영으로 넘어갈 때는 VPS(Hetzner·Vultr
등 월 $5~7)로 옮긴다 — 로컬 Docker는 노트북이 켜져 있을 때만 돈다.

### 2. Google Sheets 자격증명
n8n **Credentials → New → Google Sheets OAuth2 API** 로 하나 만들고 이름을
`Google Sheets - inseoul` 로 붙인다(워크플로 JSON에 이 이름이 참조로 박혀 있어서
찾기 쉽다). 워크플로를 임포트한 뒤 Google Sheets 노드마다 이 자격증명을 한 번씩
선택해줘야 한다 — 계정이 다르면 자격증명 ID가 안 맞기 때문에 n8n이 자동으로
매칭해주지 않는다. 8개 워크플로 × 평균 2~3개 노드, 총 10분 안팎.

### 3. 환경변수 (n8n Settings → Variables, 또는 `.env`)

| 변수 | 값 | 용도 |
|---|---|---|
| `ANTHROPIC_API_KEY` | `sk-ant-...` | 사실조사·원고 배치 API |
| `HTML2IMG_API_KEY` | `htim_...` | 카드 렌더 |
| `META_ACCESS_TOKEN` | `IGAA...` | `@inseoul_curation` 발행 (KR) |

⚠️ **인계문서 09-16에 화면 캡처로 Anthropic 키가 노출됐다고 적혀 있다.**
아직 재발급 전이면 `console.anthropic.com/settings/keys` 에서 지금 재발급하고
새 키를 여기 넣을 것 — Make 쪽 3~4곳을 일일이 고칠 필요 없이 여기 한 곳만 바꾸면
된다는 게 이 이전의 실질적 이득 중 하나다 (§10-C 문제가 자동으로 풀린다).

`META_ACCESS_TOKEN_EN` 은 EN 계정(`@inseoul_curation_en`) 발행 워크플로를 만들 때
추가하면 된다 (§10-6, 아직 미착수 — 이번 이전 범위 밖).

### 4. GitHub
공개 리포(`inseoulcuration/inseoul_assets`)라 `04-photo-collect.json`의 GitHub 노드는
인증 없이 그대로 동작한다. 리포를 비공개로 바꾸면 그 노드에 PAT 헤더 인증을 추가해야
한다 — 현재는 필요 없음.

---

## 임포트 순서

1. n8n 왼쪽 메뉴 **Workflows → Import from File** 로 `n8n-workflows/*.json` 8개를 각각 불러온다.
2. 각 워크플로에서 **Google Sheets 노드**(보통 2~3개)를 열어 자격증명을 선택하고,
   `documentId`/`sheetName` 드롭다운을 한 번 다시 선택한다 — 임포트 직후에는
   "찾을 수 없음"으로 보이는 게 정상이다(다른 계정에서 만든 리소스 참조라 그렇다).
   시트 URL: `docs.google.com/spreadsheets/d/1e3ym-2I7ju60aVtuCqFCRgNzT4LIvXaZRfe6ZjYx_eo`
3. `02-facts-receive.json`과 `03-draft-receive.json`, `08-en-receive.json`의
   **"Anthropic: 결과 다운로드 (raw)"** HTTP 노드를 열어 Options → Response → Response
   Format이 **Text**(또는 File)로 되어 있는지 확인한다. 배치 결과가 JSONL이라
   자동 JSON 파싱을 켜두면 깨진다 — Make의 "Parse response: No"와 같은 이유다.
   노드 위에 스티키 노트로 이 내용을 적어 두었다.
4. 각 워크플로 안의 스티키 노트(노란 메모)를 한 번씩 읽는다 — Make에서 겪었던
   함정과 그걸 n8n에서 어떻게 다르게 처리했는지 그 자리에 적어 두었다.

---

## 워크플로별 확인 포인트

### 01-batch-submit / 07-en-submit
가장 단순하다. 시트 읽기 → 대상 행 찾기(Code) → 프롬프트 조립(Code) → Anthropic
배치 제출(HTTP) → 시트 갱신. 임포트 후 자격증명만 연결하면 바로 실행 가능.

### 02-facts-receive / 03-draft-receive / 08-en-receive
구조가 같다: 배치 상태 조회 → `ended`인지 If → raw 텍스트 다운로드 → Code로
`<<<`/`>>>` 사이 JSON 추출. **02는 추가로 게이트A(facts≥6, sources≥2, 첫
entity의 fact_ids≥1) 판정까지 하고, 통과하면 곧바로 원고 배치를 제출한다** —
Make의 Router 12 한 개를 그대로 옮긴 것.

### 04-photo-collect
Make에서 Iterator+필터+Text aggregator 3개 모듈이 하던 일을 Code 노드 하나가
한다 (`.gitkeep` 제외, 파일명으로 정렬, `download_url` 콤마 조인). 가장 단순하니
**맨 처음 시험용으로 이 워크플로를 추천한다** — §13-7의 원래 계획과 같다.

### 05-render
가장 손이 많이 간 워크플로다. `card-template.html`의 CSS를 Code 노드 안 JS 함수
(`buildCardHtml`)로 그대로 옮겼다. Make 때는 HTML을 한 줄로 압축하고 큰따옴표를
빼야 했지만, 여기서는 HTTP 노드가 JS 객체를 알아서 JSON으로 직렬화하므로 그런
제약이 전혀 없다 — 원본 HTML을 거의 그대로 읽을 수 있는 형태로 넣었다.

### 06-publish
가장 복잡하다. 이미지 컨테이너 9개 생성 → 캐러셀 컨테이너 생성 → **5초 대기 →
상태 조회 → FINISHED면 발행, 아니면 최대 4회까지 반복 → 초과하면 실패 처리**로
루프를 구성했다. Make의 "Repeater 안에서는 뒤 모듈을 참조할 수 없어 조기 종료가
불가능하다"는 제약(§9)이 여기서는 없다 — 실측으로 대개 첫 조회에 FINISHED였다는
문서 §4-5의 기록대로라면, 이 워크플로는 대부분 5초 만에 끝난다.

---

## 이번 주 진행 순서 (제안)

| 요일 | 할 일 |
|---|---|
| **1일차** | Docker로 n8n 로컬 기동 → Google Sheets 자격증명 1개 생성 → `04-photo-collect.json` 임포트 후 실제 행 하나로 시험 (가장 단순, 실패해도 손실 적음) |
| **2일차** | `01-batch-submit.json`, `02-facts-receive.json`, `03-draft-receive.json` 임포트·시험 — 여기서 Anthropic 배치 왕복(20~40분)이 걸리니 다른 일과 병행 |
| **3일차** | `05-render.json` 시험 — 렌더된 카드 9장이 Make 때와 시각적으로 동일한지 눈으로 확인 (html2img 크레딧 소모, 여유 있음: 월 1,000 중 200/월 사용 중) |
| **4일차** | `06-publish.json` 시험 — **테스트 계정이나 임시 게시물로 먼저 확인**, 실제 발행 전에 캐러셀 구성이 맞는지 재차 확인 |
| **5일차** | 신규 행 1건으로 **01→06 전 구간 완주** (문서의 "10행·11행 실렌더 검증" 같은 방식). 통과하면 VPS로 옮겨 상시 구동 준비. EN(07/08)은 KR 파이프라인이 안정된 뒤 이어서 |

문서 §10 A-3의 교훈 그대로: **재보기 전에 설계하지 말 것.** 각 단계에서 실제 값으로
한 번 돌려보고 걸리는 시간과 실패 지점을 재고 나서 다음 단계로 넘어간다.

---

## 알아두어야 할 한계 (거짓말하지 않기)

- **`01-facts-research-user-prompt.md`(사실조사 프롬프트)는 원본 Make Body 파일이
  없어 인계문서 마크다운에서 재구성한 것이다.** KR/EN 원고 프롬프트 둘은 실제
  Make Body JSON을 그대로 파싱해 옮겨서 글자 단위로 확실하지만, 이것만은 실제
  Make 시나리오(`inseoul_배치제출`의 HTTP 4)의 Body와 한 번 나란히 대조해보길 권한다.
- Google Sheets 노드의 리소스 선택 UI(`documentId`/`sheetName`)는 계정이 바뀌면
  임포트 후 반드시 재선택해야 한다 — n8n 자체의 동작이라 피할 수 없다.
- HTTP 응답을 raw text로 받는 옵션의 정확한 토글 위치는 n8n 버전에 따라 조금씩
  달라질 수 있다 — 위 "임포트 순서" 3번을 꼭 확인할 것.
- `06-publish.json`의 대기 시간(5초×최대4회)은 Make와 동일한 값을 썼다. 문서
  §4-5의 실측("첫 조회에 이미 FINISHED")이 계속 맞다면 넉넉하지만, 인스타 쪽
  응답이 느려지면 이 값을 늘려야 할 수 있다.
- 사진 조달, Threads/블로그/릴스 채널, EN 렌더·발행 워크플로(§12-6 미착수)는
  이번 이전 범위에 포함하지 않았다 — Make에도 아직 없던 것들이라 이전할 대상이
  없었다. n8n 파이프라인이 안정되면 그 위에 이어 만들면 된다(문서 §13-7 3단계).
