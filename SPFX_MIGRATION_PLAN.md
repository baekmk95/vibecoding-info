# SharePoint SPFx 전환 계획

> 대상: `index.html`(입문 가이드), `internal.html`(사내망 반입 가이드), `practice.html`(HR 대시보드 실습)
> 현재 배포: Cloudflare Pages 정적 사이트 (`vibecoding-info.pages.dev`), 빌드 도구 없음, 순수 HTML/CSS/Vanilla JS 3파일 구조

## 1. 현황 분석

| 항목 | 현재 구조 |
|---|---|
| 구성 | 정적 HTML 3개(각 950~1930줄), 파일당 `<style>`/`<script>` 인라인, 공통 컴포넌트/빌드 파이프라인 없음 |
| 외부 리소스 | Google 폰트 대신 jsDelivr의 SUITE Variable 웹폰트, cdnjs의 Font Awesome 6 CSS |
| 상호작용 | 순수 JS(IIFE)로 구현: 모달, 복사 버튼, 슬라이드 모드, 스크롤 리빌(IntersectionObserver), back-to-top, 스와이프, CSV 다운로드(Blob) |
| 페이지 간 이동 | `<a href="index.html">` 형태의 상대경로 링크 |
| 데이터 | 서버/API 없음. `practice.html`의 HR 표본 데이터는 JS 배열에 하드코딩 |
| 디자인 | 풀블리드 히어로(`clip-path` 사용), 최대 폭 1000~1100px 컨테이너, 카드형 레이아웃 |

이 사이트는 SPA도 아니고 백엔드 의존성도 없어서, SPFx로 옮길 때 "기능을 새로 만드는" 작업이 아니라 **동일한 마크업/스타일/JS 로직을 SPFx 웹파트 구조로 포장하는 작업**에 가깝습니다. 다만 SPFx 특유의 제약(웹파트 캔버스 폭, 테마 CSS 충돌, 사내망 CDN 차단 이슈 — 공교롭게도 `internal.html` 본문이 다루는 바로 그 문제) 때문에 손봐야 할 지점이 있습니다.

## 2. 핵심 아키텍처 결정

### 2.1 웹파트 분할 전략
페이지 3개 → **SPFx 웹파트 3개**로 1:1 매핑하고, SharePoint 최신 페이지 3개(예: `/SitePages/vibecoding-guide.aspx`, `internal-guide.aspx`, `practice.aspx`)에 각각 배치합니다.
- 장점: 현재 정보구조(라우팅 없이 파일 단위 이동) 그대로 유지, 각 웹파트가 독립적으로 가볍게 빌드/배포됨.
- 대안(단일 웹파트 + 내부 탭 전환)은 페이지당 URL이 사라져 북마크/공유가 불가능해지므로 채택하지 않음.

### 2.2 프레임워크: React + TypeScript (SPFx 표준 구성)
`No JavaScript Framework` 템플릿으로 마크업만 옮기는 대신 React를 채택합니다.
- 모달 열림/닫힘, 슬라이드 모드, 복사 버튼 상태 등은 React state로 옮기는 편이 SPFx 표준 라이프사이클(프로퍼티 변경, 테마 변경 이벤트)과 자연스럽게 맞물림.
- 장기 유지보수 시 SPFx 생태계(PnP React 컨트롤 등) 재사용 가능.
- 순수 DOM 조작 코드(IntersectionObserver, touch 이벤트)는 `useEffect` 안에서 기존 로직을 거의 그대로 재사용 가능 — 전면 재작성이 아니라 감싸는 작업.

### 2.3 스타일: 페이지별 SCSS 모듈
각 페이지의 `:root` CSS 변수와 클래스 전체를 웹파트별 `*.module.scss`로 이전. 클래스명은 SPFx CSS 모듈이 자동으로 스코프 처리하므로 SharePoint 페이지의 전역 스타일과 충돌하지 않음. `* { margin:0; padding:0 }` 같은 전역 리셋은 웹파트 루트 엘리먼트 하위로 스코프를 좁혀야 함(SharePoint 페이지의 다른 웹파트에 영향 금지).

### 2.4 외부 리소스(폰트/아이콘) — 사내망 이슈 정면 대응
`internal.html`이 스스로 경고하는 문제(사내망에서 임의 CDN 차단)를 SPFx 버전에서 그대로 남겨두면 모순입니다.
- **SUITE Variable 폰트**: jsDelivr 대신 `.woff2` 파일을 웹파트 `assets/` 폴더에 넣고 `require()`로 로컬 번들에 포함 (webpack이 해시 포함 정적 자산으로 처리).
- **Font Awesome**: cdnjs CSS 전체 대신 (a) 사내망에서 이미 허용된 SharePoint 기본 아이콘 세트인 **Fluent UI(Office UI Fabric) 아이콘**으로 대체하거나, (b) 꼭 Font Awesome이 필요하면 `@fortawesome/fontawesome-free`를 npm 의존성으로 설치해 로컬 번들에 포함. 둘 다 외부 네트워크 요청 자체를 없애 사내망/오프라인 환경에서도 100% 동작.
- **og-image.png**: 소셜 공유 메타태그는 웹파트가 아니라 페이지 속성 영역이므로 SPFx 코드 밖(페이지 설정 또는 사이트 홈페이지 설정)에서 처리.

### 2.5 페이지 간 링크
`href="internal.html"` 같은 상대경로는 SharePoint 사이트 페이지 URL로 교체하고, `this.context.pageContext.web.absoluteUrl` 기반으로 동적 생성(사이트 URL이 환경마다 달라도 하드코딩 없이 동작).

## 3. 단계별 실행 계획

| Phase | 내용 | 주요 산출물 |
|---|---|---|
| 0. 준비 | SPFx 버전 확정(테넌트 SharePoint Online 기준 최신 LTS, Node 18.x), 테넌트/사이트 앱 카탈로그 권한 확인, `yo @microsoft/sharepoint` 개발환경 구성 | 개발 환경 세팅 |
| 1. 스캐폴딩 | SPFx 솔루션 1개 생성, 웹파트 3개 스캐폴딩: `VibeCodingGuideWebPart`, `InternalNetworkGuideWebPart`, `PracticeDashboardWebPart` | 빈 웹파트 3종 (로컬 워크벤치에서 렌더 확인) |
| 2. 스타일 이전 | 인라인 `<style>` → 웹파트별 `.module.scss`, CSS 변수 → SCSS 변수/커스텀 프로퍼티, 전역 리셋 스코프 조정 | 정적 마크업이 원본과 시각적으로 동일하게 렌더 |
| 3. 마크업→컴포넌트 변환 | 섹션 단위로 React 컴포넌트 분해 (Hero, FlowDiagram, DetailCard, PromptBox, Modal, SchemaTable, MockDashboard 등), 반복 구조는 데이터 배열 + `.map()`으로 정리 | 컴포넌트 트리 |
| 4. 인터랙션 이전 | 기존 IIFE JS → React hooks: 모달(`useState`), 복사 버튼(`navigator.clipboard` 그대로 재사용), 슬라이드 모드(`useState` + `useEffect` 키보드 리스너), 스크롤 리빌(`useRef` + `IntersectionObserver`), CSV 다운로드(Blob 로직 그대로 재사용) | 원본과 동일한 UX |
| 5. 자산 로컬화 | 폰트 `.woff2`, 아이콘 세트를 npm 패키지/로컬 파일로 전환 (2.4 참조) | 외부 네트워크 요청 0건 확인 (사내망 오프라인 테스트 통과) |
| 6. 페이지 구성 | SharePoint 최신 페이지 3개 생성, 웹파트 배치(풀폭 섹션 사용 검토), 페이지 간 링크를 SharePoint URL로 교체 | 사이트에서 3개 페이지 탐색 가능 |
| 7. 접근성/반응형 QA | 좁은 웹파트 존/모바일 뷰에서 레이아웃 검증, 포커스 트랩/키보드 내비게이션 재검증, `prefers-reduced-motion` 유지 확인 | QA 체크리스트 통과 |
| 8. 배포 | `gulp bundle --ship && gulp package-solution --ship`, `.sppkg`를 테넌트(또는 사이트) 앱 카탈로그에 업로드, 사이트에 앱 추가 | 프로덕션 배포 완료 |
| 9. 운영 전환 | 기존 Cloudflare Pages 사이트는 리다이렉트 안내 또는 유지, README에 SPFx 빌드/배포 절차 문서화 | 신규 유지보수 프로세스 |

## 4. 리스크 및 결정이 필요한 사항

- **웹파트 캔버스 폭 제약**: 현재 디자인은 `clip-path` 풀블리드 히어로 + 최대폭 1000~1100px 컨테이너 조합입니다. SharePoint 최신 페이지의 "전체 너비 섹션"을 쓰면 유사하게 재현 가능하지만, 일반 섹션에서는 히어로가 잘려 보일 수 있어 QA 단계에서 반드시 확인이 필요합니다.
- **테마 충돌**: SharePoint 사이트 테마(파랑 계열 기본 테마 등)가 카드 그림자/버튼 색상과 부딪힐 수 있음 — 웹파트 루트에 `data-theme` 스코프를 주고 `!important` 없이 우선순위로 해결.
- **온라인 vs 온프레미스**: SharePoint Online인지 SharePoint Server(온프레미스)인지에 따라 SPFx 버전과 앱 카탈로그 배포 절차가 달라짐 — 확인 필요.
- **단일 웹파트 통합 여부**: 지금은 페이지 3개=웹파트 3개로 계획했으나, 만약 "허브사이트 내 탐색 메뉴로 3개를 묶고 싶다"는 요구가 있으면 SharePoint 사이트 탐색(사이트 내비게이션) 설정으로 대체 가능(코드 변경 없음).
- **실습 페이지의 실데이터화**: `practice.html`은 하드코딩된 샘플 데이터입니다. SPFx 전환을 기회로 실제 SharePoint 리스트/PnPjs 연동까지 원하는지 여부는 범위 외 옵션으로, 필요 시 별도 단계로 분리 권장.

## 5. 예상 리포지토리 구조 (Phase 1 이후)

```
vibecoding-info-spfx/
├─ config/                     # SPFx 빌드 설정
├─ src/
│  └─ webparts/
│     ├─ vibeCodingGuide/
│     │  ├─ components/        # Hero, FlowDiagram, DetailCard, PromptBox, Modal ...
│     │  ├─ assets/            # SUITE Variable woff2 등 로컬 폰트
│     │  └─ VibeCodingGuideWebPart.ts
│     ├─ internalNetworkGuide/
│     └─ practiceDashboard/
├─ sharepoint/assets/           # 배포용 .sppkg 산출물
├─ gulpfile.js
└─ package.json
```

원본 `index.html`/`internal.html`/`practice.html`은 참고용 원본으로 리포지토리에 유지하거나, 마이그레이션 완료 후 `legacy/` 폴더로 이동해 이력만 보존하는 방식을 권장합니다.

## 6. 다음 액션

이 계획에 동의하시면 이어서 진행할 수 있는 작업:
1. SPFx 개발 환경 세팅(Node/SPFx 버전, `yo`/`gulp` 설치) 및 솔루션 스캐폴딩
2. 웹파트 1개(가장 단순한 `index.html`)부터 시범 전환 후 나머지 2개에 패턴 적용
3. 사내망 CDN 차단 여부를 실제로 테스트할 수 있는 환경이 있는지 확인

궁금하신 세부 사항(SharePoint Online/온프레미스 여부, 웹파트 분할 방식 등)이 있으면 알려주시면 계획을 조정하겠습니다.
