# 프로토타입 체인 원정대 퀴즈앱

우아한테크코스 프론트엔드 크루가 다른 크루원들에게 JS 프로토타입 체인을 설명하기 위해 만든 퀴즈앱.

## 프로젝트 구조

```
/
├── index.html              # 앱 전체 (HTML + CSS + JS 단일 파일)
│   ├── <style>             # CSS 변수, 레이아웃, 컴포넌트 스타일
│   ├── <body>
│   │   ├── #intro-page     # 인트로 화면
│   │   ├── #quiz-page      # 퀴즈 화면
│   │   │   ├── .quiz-header          # 진행 바 + 문제 번호
│   │   │   ├── .main-content         # 문제 렌더링 영역 (#question-content)
│   │   │   └── aside#sidebar         # 해설 사이드바 (정답 확인 후 슬라이드인)
│   │   └── #result-page    # 결과 화면
│   └── <script>
│       ├── characters[]    # 캐릭터 이미지 파일명 목록
│       ├── quizData[]      # 퀴즈 데이터 배열 (문제/정답/해설)
│       ├── highlightJS()   # 인라인 JS 구문 강조기
│       └── 퀴즈 흐름 함수들
├── images/                 # 캐릭터 이미지 (행성이 시리즈)
│   ├── planet-coding.png   # 인트로 메인 이미지
│   ├── planet-default.png  # 사이드바 기본 캐릭터
│   ├── planet-coffee.png
│   ├── planet-leg.png
│   ├── planet-cheer.png
│   └── planet-star.png     # 결과 페이지 이미지
├── package.json            # vite 개발 서버
├── CLAUDE.md
└── .gitignore
```

**모든 로직은 `index.html` 한 파일**에 인라인으로 작성되어 있다. 별도의 빌드 파이프라인 없음.

## 개발 서버

```bash
npm run dev      # http://localhost:5173 에서 개발 서버 실행 (파일 변경 시 자동 새로고침)
npm run build    # 프로덕션 빌드 → dist/
npm run preview  # 빌드 결과물 미리보기
```

## 페이지 구성 및 흐름

```
[#intro-page] → startQuiz() → [#quiz-page] → showResult() → [#result-page]
                                    ↑                               |
                                    └──────── resetQuiz() ──────────┘
```

앱은 세 개의 `div`로 구성되며, JS로 `display` 속성을 전환해 페이지를 교체한다.

| 페이지 ID | 설명 | 진입 함수 |
|---|---|---|
| `#intro-page` | 팀 정보, 시작 버튼. 기본 표시 상태 | — |
| `#quiz-page` | 문제 + 사이드바 해설. 두 컬럼 레이아웃 | `startQuiz()` |
| `#result-page` | 원정 완료 메시지 | `showResult()` |

### 퀴즈 화면 내부 구조

```
#quiz-page
├── .quiz-header
│   ├── .progress-bar > .progress-fill   # 진행률 바 (width %로 제어)
│   └── #question-number                 # "문제 N / 5" 텍스트
├── .quiz-layout (flex)
│   ├── .main-content
│   │   ├── #question-content            # renderQuestion()이 innerHTML로 교체
│   │   └── .nav-buttons
│   │       ├── #prev-btn                # 이전 문제
│   │       ├── #check-btn               # 정답 확인 (클릭 시 사이드바 열림)
│   │       └── #next-btn                # 다음 문제 (정답 확인 후에만 표시)
│   └── aside#sidebar (.sidebar)
│       ├── .character-img-box > #side-character-img
│       ├── #answer-status               # 정답 코드 블록
│       ├── #exp-text                    # 해설 텍스트
│       └── #ref-text (.reflection-box)  # 고민해볼 점 (노란 박스)
```

사이드바는 기본 `width: 0; opacity: 0` 상태이며, `.active` 클래스 추가 시 `width: 420px`로 슬라이드인된다.

## 퀴즈 데이터 구조

퀴즈는 `index.html` 내 `quizData` 배열로 관리된다.

```js
{
  id: 1,                    // 문제 번호 (현재 미사용, 표시용)
  question: '문제 설명',    // 문제 안내 텍스트
  code: `JS 코드 스니펫`,   // 코드 블록 (백틱 템플릿 리터럴, highlightJS에 전달됨)
  answer: "예상 결과값",    // 정답 텍스트 (highlightJS로 강조 적용됨)
  explanation: "해설 내용", // #exp-text에 innerText로 출력 (\n 줄바꿈 지원)
  reflection: "고민해볼 점" // #ref-text에 innerText로 출력
}
```

### 퀴즈 추가 방법

`quizData` 배열에 위 구조의 객체를 추가하면 된다. 문제 수(`quizData.length`)는 진행 바와 문제 번호에 자동 반영된다.

퀴즈 코드에 새로운 클래스명/함수명이 등장하면 `highlightJS`의 `rules` 배열도 함께 업데이트해야 구문 강조가 적용된다 (아래 섹션 참고).

## 주요 함수

| 함수 | 역할 |
|---|---|
| `startQuiz()` | 인트로 숨김 → 퀴즈 표시 → `renderQuestion()` 호출 |
| `renderQuestion()` | `currentQuestionIndex` 기준으로 `#question-content` innerHTML 교체, 사이드바 닫기, 버튼 상태 초기화 |
| `checkAnswer()` | 사이드바에 정답/해설/reflection 삽입 후 `.active` 추가, check-btn 숨김/next-btn 표시, localStorage 저장 |
| `goToNextQuestion()` | 인덱스 증가 후 `renderQuestion()`, 마지막이면 `showResult()` |
| `goToPrevQuestion()` | 인덱스 감소 후 `renderQuestion()` |
| `showResult()` | 퀴즈 숨김 → 결과 표시 → localStorage 삭제 |
| `resetQuiz()` | 결과 숨김 → 인트로 표시 → 인덱스 0 초기화 |
| `highlightJS(code)` | 코드 문자열을 받아 `<span class="hl-*">` 태그로 감싼 HTML 문자열 반환 |

## 상태 관리

| 상태 | 위치 | 설명 |
|---|---|---|
| `currentQuestionIndex` | JS 전역 변수 | 현재 문제 인덱스 (0-based) |
| `proto_expedition_step` | localStorage | 마지막으로 정답 확인한 문제 인덱스. 재방문 시 이어서 시작. 완료/리셋 시 삭제 |

## 이미지

`images/` 폴더의 캐릭터 이미지를 사용한다. 파일명은 영문으로 유지한다 (한글 파일명은 Vercel CDN에서 URL 인코딩 문제로 로드 실패함).

| 파일명 | 사용 위치 |
|---|---|
| `planet-coding.png` | 인트로 페이지 메인 이미지 |
| `planet-default.png` | 퀴즈 사이드바 기본 캐릭터 |
| `planet-star.png` | 결과 페이지 이미지 |

## 코딩 컨벤션

### JavaScript

- 변수/함수: `camelCase`
- 상수: `UPPER_SNAKE_CASE`
- 문자열: 작은따옴표 `'` 사용, 템플릿 리터럴은 보간이 필요할 때만
- 변수 선언: `const` 우선, 재할당이 필요한 경우만 `let`, `var` 사용 금지
- 함수: 함수 선언식(`function foo()`) 사용
- 비교 연산자: 엄격한 동등 `===` 사용
- 세미콜론: 항상 붙임

```js
// good
const MAX_QUESTIONS = 5;
function renderQuestion() { ... }
if (value === null) { ... }

// bad
var count = 0;
const fn = () => { ... }  // 최상위 함수는 선언식으로
if (value == null) { ... }
```

### HTML / CSS

- 들여쓰기: 스페이스 2칸
- CSS 클래스명: `kebab-case`
- ID: `kebab-case`, JS에서 접근하는 요소에만 사용
- 인라인 스타일: 동적으로 값이 바뀌는 경우에만 허용, 정적 스타일은 `<style>`에 작성

### 기타

- 주석: 로직이 명확하지 않은 경우에만 작성, 자명한 코드에는 생략
- 커밋 메시지: 한국어 또는 영어 중 일관되게, 동사로 시작 (예: `퀴즈 데이터 추가`, `Fix image path`)

## 코드 구문 강조 (`highlightJS`)

별도 라이브러리 없이 직접 구현된 간단한 하이라이터. `<span class="hl-*">` 태그를 삽입하며, 이미 처리된 span은 다시 처리하지 않는다.

지원 CSS 클래스:

| 클래스 | 색상 | 대상 |
|---|---|---|
| `hl-keyword` | 보라 | `const`, `let`, `class`, `new` 등 |
| `hl-string` | 주황 | 문자열 리터럴 |
| `hl-comment` | 초록 | `//` 주석 |
| `hl-func` | 노랑 | 함수/메서드명 |
| `hl-const` | 하늘 | `true`, `false` |
| `hl-class` | 민트 | 클래스/생성자명 |
| `hl-number` | 연두 | 숫자 리터럴 |

퀴즈 코드에 새로운 클래스명이나 함수명이 등장하면 `rules` 배열에 추가한다:

```js
// highlightJS 내부 rules 배열
{ reg: /\b(Car|Game|Shape|Circle|...추가할 이름)\b/g, cls: 'hl-class' }
{ reg: /\b(console\.log|push|...추가할 함수)\b/g,    cls: 'hl-func'  }
```
