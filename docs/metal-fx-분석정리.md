# metal-fx 완전 분석 정리 💎

> 카리나와 함께한 `metal-fx` 저장소 분석 대화 정리본
> 작성일: 2026-09-18

## 🔗 관련 링크

| 항목 | 주소 |
|---|---|
| **내 저장소 (포크)** | https://github.com/bmshin94/metal-fx |
| **원본 저장소** | https://github.com/Jakubantalik/metal-fx |
| **라이브 데모** | https://metal.jakubantalik.com |
| **npm 패키지** | https://www.npmjs.com/package/metal-fx |
| **이슈 트래커** | https://github.com/Jakubantalik/metal-fx/issues |

---

## 1. 이게 뭐하는 건지

### 한 줄 요약
`metal-fx`는 React 버튼/칩/아이콘을 감싸면 **WebGL 셰이더로 실시간 "액체 금속(liquid metal)" 테두리**를 그려주는 npm 라이브러리다.

### 기본 정보

- 패키지명: `metal-fx` (v1.0.4)
- 원작자: **Jakub Antalik**
- 라이선스: **MIT** (상업적 이용·수정·재배포 자유, 저작권 고지만 필요)
- 언어: TypeScript
- peerDependencies: `react >= 18`, `react-dom >= 18`
- 런타임 의존성: **0개**
- 원본 저장소 지표: ⭐ 478 / 🍴 33 (2026-05-04 생성)

### 폴더 구조 (실측)

```
src/                    ← 배포되는 라이브러리 본체 (약 2,400줄)
├─ MetalFx.tsx          (357줄) 유일한 공개 컴포넌트
├─ types.ts             (149줄) props 타입 정의
├─ styles.ts            (249줄) 런타임 CSS 주입
├─ index.ts             공개 API 진입점
└─ engine/              ← 핵심 엔진
   ├─ shaders.ts        (257줄) GLSL 버텍스 + 프래그먼트 셰이더
   ├─ presets.ts        (183줄) chromatic / silver / gold 색상 프리셋
   ├─ perfConfig.ts     성능 상수 (15fps, DPR 캡 2, GL 캔버스 96px)
   ├─ color.ts          hex → rgb 변환
   ├─ tween.ts          보간
   ├─ renderer/         core.ts / loop.ts / sampling.ts
   ├─ glow/             glow.ts / geometry.ts (SVG 후광)
   └─ reflection/       주변 요소 반사광 모듈 5개

demo/                   ← 배포 안 됨. 플레이그라운드 + 문서 사이트
public/                 _redirects
.github/workflows/      pages.yml (데모 배포) / publish.yml (npm 배포)
CLAUDE.md               카리나 페르소나 파일 (내가 PR #1로 추가)
biome.json              린터/포매터 설정
```

### 동작 원리 (6단계)

1. `<MetalFx>`가 자식 요소 **하나**를 감싸고 `ResizeObserver`로 크기를 측정
2. 페이지 전체에서 **WebGL 컨텍스트를 딱 1개만 공유** (`ensureSharedRenderer`)
3. 그 하나의 오프스크린 GL 캔버스에 GLSL **Plasma 셰이더**를 렌더
   (simplex noise + fbm 옥타브 + warp + 팔레트 보간 + vignette + 9-tap 블러)
4. 각 인스턴스는 결과를 잘라서(`drawImage`) 자기 2D 캔버스에 복사하고,
   가운데를 **둥근 구멍으로 뚫어** 링(테두리)만 남김
5. `gl.readPixels`로 가장 밝은 지점을 찾아 SVG 후광(glow)을 그 위치로 이동
6. 자식 요소는 `pointer-events: none` 오버레이 아래 → **클릭 100% 정상 동작**

### 언제 쓰는가

- "Upgrade to Pro" 같은 결제/프리미엄 유도 버튼
- AI 챗 인터페이스의 전송 버튼 (원형 variant — 데모의 대표 케이스)
- 신규 기능 배지, 한정판 CTA, 히어로 섹션 버튼
- 핵심: **페이지에 1~2개만** 써야 고급스럽고, 남발하면 촌스러워진다

### 나에게 주는 도움

| 관점 | 내용 |
|---|---|
| 바로 쓰기 | `npm i metal-fx` → 3줄로 전환율 올리는 버튼 완성 |
| 학습 교재 | WebGL 공유 컨텍스트, GLSL 노이즈, RAF 단일 루프, IntersectionObserver 최적화, SSR 안전 처리를 2,400줄로 압축한 교과서급 레퍼런스 |
| 포트폴리오 | MIT라서 포크·수정·재배포 자유. 개조해서 내 브랜드 라이브러리로 가능 |
| 오픈소스 경험 | 원본 PR 히스토리(Safari 글로우 버그 수정 등)로 진입 난이도 파악 쉬움 |

---

## 2. 더 쉬운 설명

### 비유
**"버튼에 씌우는 홀로그램 포토카드 케이스"**.
각도를 틀면 무지개색으로 번쩍거리는 포카 케이스인데, 각도를 안 틀어도 **스스로 계속 흐물흐물 움직이는** 버전.

### 코드는 이게 전부

```bash
npm install metal-fx
```

```jsx
import { MetalFx } from 'metal-fx';

<MetalFx>
  <button>결제하기</button>   {/* ← 기존 버튼 그대로 */}
</MetalFx>
```

버튼 코드는 **한 글자도 고칠 필요 없다.**

### 감싸면 벌어지는 일

```
[1] 원래 버튼        [2] metal-fx가 감쌈       [3] 결과
  ┌─────────┐         테두리에 무지개              ╔═══════════╗
  │ 결제하기 │   →    금속 링이 생기고      →     ║ 결제하기  ║ ← 계속 반짝
  └─────────┘         스스로 흐물흐물               ╚═══════════╝    클릭도 정상
```

### 주요 옵션

```jsx
<MetalFx preset="gold" />       {/* 금색 (silver / chromatic 도 있음) */}
<MetalFx variant="circle" />    {/* 동그란 버튼용 */}
<MetalFx strength={0.5} />      {/* 0 = 안 보임, 1 = 풀파워 */}
<MetalFx paused />              {/* 애니메이션 정지 */}
<MetalFx theme="dark" />        {/* 생략하면 OS 다크모드 자동 추종 */}
```

### 잘 만든 이유 (쉬운 버전)

- **배터리 안 잡아먹음**: 버튼 10개를 써도 그림 그리는 엔진은 딱 1개
- **안 보이면 멈춤**: 스크롤로 화면 밖 → 자동 정지 / 탭 숨김 → 완전 정지
- **15fps 고정**: 흐물흐물한 효과는 60fps가 필요 없어서 일부러 낮춰 CPU 절약
- **클릭 안 막힘**: 반짝이는 층이 클릭을 통과시킴
- **다크/라이트 자동**: OS 테마가 바뀌면 즉시 색 전환
- **Next.js OK**: SSR 중에는 투명 placeholder, 하이드레이션 후 브라우저에서만 구동

---

## 3. 세부 질문 답변

### 3-1. 설치 및 사용법

**A. 그냥 쓰기**

```bash
npm install metal-fx
# react, react-dom 18+ 는 내 프로젝트에 이미 있어야 함 (peerDependency)
```

```jsx
import { MetalFx } from 'metal-fx';

export default function Page() {
  return (
    <MetalFx preset="chromatic" strength={1}>
      <button className="rounded-full px-6 h-10">Upgrade to Pro</button>
    </MetalFx>
  );
}
```

CSS import 불필요 — `styles.ts`의 `ensureStylesInjected()`가 모듈 로드 시점에 자동 주입.

**B. 이 포크로 직접 개발하기**

```bash
git clone https://github.com/bmshin94/metal-fx
cd metal-fx
npm install
npm run dev          # 데모 사이트 로컬 실행
npm run typecheck    # tsc --noEmit
npm run build        # dist/ 라이브러리 빌드
npm run build:demo   # dist-demo/ 데모 빌드
```

**전체 Props** (`src/types.ts` 기준)

| Prop | 타입 | 기본값 | 설명 |
|---|---|---|---|
| `children` | ReactNode | 필수 | 감쌀 요소 **하나** |
| `variant` | `'button'` \| `'circle'` | `'button'` | 필(pill) vs 원형 |
| `preset` | `'chromatic'` \| `'silver'` \| `'gold'` | `'chromatic'` | 색상 팔레트 |
| `theme` | `'dark'` \| `'light'` \| `'auto'` | `'auto'` | OS 테마 추종 |
| `strength` | 0~1 | `1` | 효과 세기 |
| `paused` | boolean | `false` | 애니메이션 정지 |
| `borderRadius` | number | 자동 측정 | 라운드 강제 지정 |
| `normalizeHostStyles` | boolean | `true` | 자식의 border/shadow 정리 |
| `reflectionTargets` | `RefObject[]` | — | 주변 요소 반사 (다크모드만) |
| `disableGlow` | boolean | `false` | 후광 끄기 |
| `shaderScale` | number | variant별 | 패턴 확대 배율 |
| `ringCssPx` | number | variant별 | 링 두께(px) |
| `scale` | number | `1` | 모든 px 상수 일괄 배율 |
| `className` / `style` | — | — | 래퍼 전달용 |

**파워유저용 엔진 직접 접근** (`src/index.ts` 노출):

```js
import {
  createInstance, destroyInstance, updateInstance,
  setSharedPreset, pauseShared, resumeShared,
  PRESETS, hexToRgb,
} from 'metal-fx';
```

→ **React 없이 순수 JS로도 구동 가능** (바닐라 JS / Web Component 포팅의 근거)

---

### 3-2. 플러그인? 스킬? MCP?

**셋 다 아니다.** 일반 **npm 프론트엔드 UI 라이브러리**다.

| 종류 | 정체 | metal-fx? |
|---|---|---|
| **npm 라이브러리** | 앱에 import해서 쓰는 코드 패키지 | ✅ **이것** |
| Claude 플러그인 | Claude Code 기능 확장 번들 | ❌ |
| Claude 스킬 | `SKILL.md` 기반 AI 작업 지침서 | ❌ |
| MCP 서버 | AI에 도구/데이터를 연결하는 프로토콜 서버 | ❌ |

**판별 근거**

- `.claude/` 폴더 없음, `SKILL.md` 없음, `plugin.json` 없음, `mcp.json` 없음
- `@modelcontextprotocol/sdk` 의존성 없음
- 존재하는 것: `package.json` + `dist` 진입점 + `peerDependencies: react`

> 헷갈릴 이유: 루트에 `CLAUDE.md`가 있어서. 하지만 그건 내가 PR #1(`6ccf03a docs: created CLAUDE.md persona guide`)로 추가한 **카리나 페르소나 파일**이고, 라이브러리 기능과 무관하다.

---

### 3-3. API 토큰이 필요한가

**라이브러리 사용에는 0개.**

- 런타임 의존성 0개, 외부 API 호출 0개
- 서버 통신 없음 → 100% 브라우저 로컬 연산
- 계정/가입/과금 없음, 오프라인 동작
- 개인정보 외부 전송 없음 → GDPR 이슈 없음

**토큰이 등장하는 곳은 `.github/workflows/` 딱 두 군데**

1. `publish.yml` → `secrets.NPM_TOKEN` : npm 패키지 배포 시에만 필요.
   내 이름으로 재배포하고 싶을 때 npmjs.com에서 Automation/Granular 토큰 발급 필요.
2. `pages.yml` → `id-token: write` : GitHub Pages 배포용 OIDC.
   GitHub가 자동 발급하므로 직접 만들 것 없음.

정리: **사용 = 토큰 0개 / 내 이름으로 재배포 = npm 토큰 1개**

---

### 3-4. 왜 GitHub에서 유명한가

실측: 원본 ⭐ 478 / 🍴 33 — 2026-05-04 생성 후 약 4개월 만의 수치.

1. **타이밍**: 2025~2026 Apple Liquid Glass 디자인 붐 + AI 앱의 "프리미엄 금속 버튼" 트렌드를 정확히 조준
2. **데모가 곧 마케팅**: `demo/`가 단순 예제가 아니라 실시간 플레이그라운드
   (`Playground.tsx` 206줄 + 슬라이더 + 복사 버튼), 커스텀 도메인까지
3. **진입 장벽 제로**: 의존성 0, 코드 3줄, 기존 버튼 무수정, TS 타입 완비
4. **기술적 완성도**: README "Performance" 주장이 실제 코드로 전부 구현됨
   - 공유 WebGL 컨텍스트 1개, 셰이더 컴파일 1회
   - RAF 루프 1개로 전체 인스턴스 구동
   - `IntersectionObserver`로 오프스크린 스킵
   - `ResizeObserver` 콜백을 RAF로 디바운스
   - 마지막 인스턴스 언마운트 시 GL 리소스 해제
   - `visibilitychange`로 탭 숨김 시 루프 정지
   - GL 컨텍스트 로스트 복구 콜백 (`setContextRestoredCallback`)
   - `GLOW_READBACK_INTERVAL_MS = 1500` — 비싼 GPU→CPU 동기화를 1.5초로 쓰로틀
5. **카테고리 킬러**: "React 금속 버튼" 니치에 사실상 대안이 없음
6. **코드 품질 시그널**: `shaders.ts` 주석이 원본 엔진 라인 번호까지 명시
   ("preserved character-for-character") → 신뢰도 상승

---

### 3-5. 로컬 에이전트 구축에 도움이 될까

**직접적으로는 ❌ / 간접적으로는 ⭕**

**직접 관계 없는 이유**: metal-fx는 픽셀을 그리는 UI 라이브러리다. LLM·프롬프트·툴 호출·RAG·벡터DB와 무관하고, `src/` 어디에도 AI 관련 코드가 없다.

**간접적으로 도움되는 3가지**

**(1) 에이전트 UI 껍데기** ← 가장 실용적

```jsx
const chipRef = useRef(null);
<>
  <button ref={chipRef}>Tools</button>
  <MetalFx variant="circle" reflectionTargets={[chipRef]}>
    <button aria-label="Send">↑</button>
  </MetalFx>
</>
```

에이전트 상태 피드백으로도 활용 가능:

```jsx
<MetalFx paused={!isThinking} strength={isThinking ? 1 : 0.4}>
```

**(2) Claude Code 실전 연습장**
`CLAUDE.md` 배치 → 브랜치 → PR → 머지 워크플로를 익히기에 딱 좋은 크기(2,400줄)의 실제 TS 프로젝트.

**(3) 아키텍처 사고 훈련** — "공유 리소스 1개 + 다수 소비자 + 프레임 예산 분배" 패턴이 에이전트 설계와 동형이다.

| metal-fx | 로컬 에이전트 |
|---|---|
| 공유 GL 컨텍스트 1개 | 공유 모델/임베딩 인스턴스 1개 |
| RAF 루프 1개 + 라운드로빈 | 스케줄러 + 태스크 큐 |
| IntersectionObserver로 스킵 | 유휴 에이전트 슬립 |
| `GLOW_READBACK_INTERVAL_MS = 1500` | 비싼 호출 쓰로틀링 |
| GL 컨텍스트 로스트 복구 | 세션 복구 / 리트라이 |

**결론**: 에이전트 **엔진**에는 0%, 에이전트 **얼굴**에는 100%.

---

### 3-6. React나 PHP로 만들 수 있을까

**React → 이미 React다.** `src/MetalFx.tsx`가 `forwardRef` 기반 순수 React 컴포넌트.
직접 만들 수 있는가에 대한 난이도 구분:

| 난이도 | 방법 | 결과 |
|---|---|---|
| 하 | CSS `conic-gradient` + `@keyframes` 회전 | 60% 유사, 1시간 |
| 중 | Canvas 2D + `createConicGradient` + 노이즈 | 75% 유사, 하루 |
| 상 | WebGL + GLSL (현재 방식) | 100%, 수 주 |

핵심 난관은 GLSL 셰이더 (`shaders.ts` 안의 simplex noise, fbm 옥타브 루프, warp,
팔레트 보간, vignette, 9-tap 블러). 단 **MIT 라이선스라 셰이더를 그대로 가져다 개조해도 합법.**

**PHP → 단독으로는 불가능.** PHP는 서버 언어, metal-fx는 브라우저 GPU 실시간 렌더링이라 층이 다르다.

```
❌ PHP 서버 ── HTML 문자열 ──> 브라우저 (움직이는 GPU 효과 표현 불가)
✅ PHP 서버 ── HTML + JS  ──> 브라우저 JS/WebGL이 GPU로 실시간 렌더
```

GD/Imagick으로 프레임을 미리 렌더해 GIF/APNG를 만드는 건 기술적으로 가능하지만,
버튼 크기·다크모드·라운드마다 이미지를 새로 생성해야 하고 품질·서버 CPU·파일 크기가
모두 문제라 실무 불가.

**PHP 프로젝트에서 쓰는 방법**

1. **바닐라 JS로 사용** — `index.ts`가 `createInstance()` 등 엔진 프리미티브를
   노출하므로 React 없이 `<script>` 로드 후 직접 호출 가능. 라라벨/워드프레스에 최적
2. Alpine.js / htmx 같은 경량 JS와 조합
3. PHP는 페이지만 서빙하고 버튼 영역만 React 마운트

정리: **React ⭕ / 바닐라 JS ⭕ / PHP 단독 ❌**

---

## 4. 수익화 아이디어

> 법적 기반: MIT 라이선스 — 상업적 이용·수정·재배포 전부 가능, 조건은 저작권 고지 + 라이선스 사본 포함.
> 단 "metal-fx 코드 자체의 유료 재판매"는 누구나 무료로 구할 수 있어 무의미하므로,
> 아래는 전부 **"위에 무엇을 얹느냐"** 전략이다.

### 1순위: 프리미엄 UI 컴포넌트 키트
**난이도 ⭐⭐ / 수익 💰💰💰 / 2~4주**

- 제품: "Metal UI Kit" — 버튼 12종, 가격표 카드, 프로 배지, 티어 테이블, 토스트, 스피너 + Figma 파일
- 판매처: LemonSqueezy / Gumroad / Paddle (해외 결제 + 세금 자동 처리)
- 가격: Personal $49 / Team $149 / Unlimited $399 (일회성 + 1년 업데이트)
- 근거: 개발자는 "예쁜 걸 조립하는 시간"에 돈을 쓴다 (Tailwind UI, shadcn pro가 증명)
- 핵심: metal-fx는 링 하나만 그린다. 사람들이 원하는 것은 **완성된 프리미엄 화면**. 그 갭이 시장.

### 2순위: 프레임워크 포팅 (독점 니치)
**난이도 ⭐⭐⭐ / 수익 💰💰💰 / 3~6주**

원본은 React 전용이지만 `src/index.ts`가 엔진 프리미티브를 노출하므로 포팅이 구조적으로 쉽다.

| 타겟 | 경쟁자 | 비고 |
|---|---|---|
| Vue 3 | 없음 | 시장 큼 |
| Svelte 5 | 없음 | 얼리어답터 많음 |
| **바닐라 JS / Web Component** | 없음 | 워드프레스·라라벨·Shopify 전부 커버 |
| Flutter / React Native | 없음 | 난이도 높고 단가 높음 |

무료 OSS로 이름을 얻고 → Pro 프리셋 팩 / 상용 라이선스 / 스폰서십으로 수익화.
특히 **Web Component 버전**이 3-6의 PHP 문제를 정확히 해결하므로 시장이 가장 크다.

### 3순위: 워드프레스 / Shopify 플러그인
**난이도 ⭐⭐⭐ / 수익 💰💰💰💰 / 4~8주**

가장 돈이 되는 구간. 개발자는 "무료 라이브러리 있는데?" 하지만
워드프레스/Shopify 운영자는 코드를 못 짜고 매출이 직결되므로 지갑을 연다.

- WP: Elementor / Gutenberg 블록으로 "금속 CTA 버튼"
- Shopify: "Add to Cart" 버튼에 적용 → 전환율 상승이 곧 ROI 증명
- 가격: 월 $9~19 구독 또는 사이트당 $79 일회성
- Shopify 앱스토어는 구독 결제가 기본이라 MRR 축적에 유리

### 4순위: SaaS — 버튼 빌더 웹서비스
**난이도 ⭐⭐⭐⭐ / 수익 💰💰💰 / 2~3개월**

- 브라우저에서 슬라이더로 색·속도·두께 조절 → 코드 / `<script>` 임베드 / CSS 내보내기
- `demo/components/Playground.tsx` (206줄)가 이미 MVP의 약 70%. 저장·계정·내보내기만 추가
- 무료(워터마크) / Pro $12/월 (무제한 + 커스텀 프리셋 + 브랜드 제거)
- 참고 모델: shadcn theme editor, uiverse.io, Haikei

### 5순위: 디자인 에셋 판매 (가장 빠름)
**난이도 ⭐ / 수익 💰💰 / 3~7일**

- 프리셋 팩 판매: "Neon Pack", "Rose Gold Pack", "Cyberpunk Pack" —
  `presets.ts` 구조 그대로 JSON 값만 튜닝. 팩당 $15~29
- metal-fx로 렌더한 버튼 영상/GIF 에셋 → Envato, Creative Market
- Figma 커뮤니티에 "Metal Button Design System" 무료 배포로 리드 확보 후 Pro 유료화
- 코드를 거의 쓰지 않고 디자인 감각만으로 수익화 가능한 유일한 루트

### 6순위: 콘텐츠 / 교육
**난이도 ⭐⭐ / 수익 💰💰 / 지속형**

- 유튜브: "WebGL 셰이더로 애플급 버튼 만들기" — 썸네일 자체가 클릭 유발
- 유료 강의: "React + WebGL 실전" (인프런 / Udemy)
- 기술 블로그 시리즈 → 위 제품들의 유입 채널
- 코드가 주석까지 교과서급이라 강의 자료 제작이 쉽다

### 7순위: 프리랜스 / 외주 특화
**난이도 ⭐⭐ / 수익 💰💰💰💰 / 즉시**

- "프리미엄 랜딩페이지 제작" 패키지의 시그니처 무기로 사용
- 포트폴리오에 이 효과가 있으면 단가가 달라진다 (일반 랜딩 ₩150만 → 프리미엄 ₩400만)
- 타겟: AI 스타트업 / 웹3 / 명품 브랜드
- 투자 0원, 즉시 가능 — 가장 빠르게 현금화하는 경로

### 8순위: 오픈소스 스폰서십 (장기)
**난이도 ⭐⭐⭐⭐ / 수익 💰 / 장기**

GitHub Sponsors, Open Collective, 듀얼 라이선스(AGPL + 상용).
단독 수익 모델로는 약하고, 다른 활동에 따라오는 보너스로 봐야 한다.

### 추천 조합 로드맵

| 단계 | 할 일 | 기간 | 목표 |
|---|---|---|---|
| 지금 당장 | 7순위 프리랜스 무기화 + 5순위 프리셋 팩 | 1주 | 첫 수익 + 검증 |
| 단기 | 2순위 Web Component 포팅 (무료 OSS) | 1개월 | 이름값 + 트래픽 |
| 중기 | 1순위 UI 키트 유료 판매 | 2개월 | 일회성 매출 |
| 장기 | 3순위 WP/Shopify 플러그인 | 3개월 | MRR 구독 수익 |

**한 줄 전략**: 무료로 이름값을 쌓고(2순위) → 그 트래픽을 유료 제품으로 전환(1·3순위).
오픈소스 수익화의 정석 루트.

---

## 5. 핵심 요약 (치트시트)

| 질문 | 답 |
|---|---|
| 뭐하는 거? | React 버튼에 WebGL 액체 금속 테두리를 입히는 npm 라이브러리 |
| 플러그인/스킬/MCP? | 전부 아님. 일반 npm UI 라이브러리 |
| API 토큰? | 사용엔 0개. 재배포 시 npm 토큰만 |
| 설치 | `npm install metal-fx` → `<MetalFx><button/></MetalFx>` |
| 왜 유명? | 타이밍 + 데모 퀄리티 + 의존성 0 + 성능 강박 + 니치 독점 (⭐478) |
| 라이선스 | MIT (상업적 이용 자유) |
| 에이전트에 도움? | 엔진엔 무관, UI(전송 버튼/상태 표현)엔 최적 |
| React로 가능? | 이미 React |
| PHP로 가능? | 단독 불가. 바닐라 JS 포팅으로 우회 |
| 최고 수익화? | 즉시 = 프리랜스 무기화 / 장기 = WP·Shopify 플러그인 구독 |

---

*Generated with Claude Code — 카리나가 정리했어! 💖*
