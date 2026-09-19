# Oxc 프로젝트 분석 & 활용 노트 (한국어)

> 저장소: <https://github.com/bmshin94/oxc>
> 원본(Upstream): <https://github.com/oxc-project/oxc>
> 공식 사이트: <https://oxc.rs> · 플레이그라운드: <https://playground.oxc.rs>
> 작성일: 2026-09-19

---

## 1. 프로젝트 개요

**Oxc (Oxidation Compiler)** 는 JavaScript / TypeScript 개발 도구들을 **Rust로 재구현한 고성능 통합 툴체인**이다.

- Evan You(Vue/Vite 창시자)가 설립한 **VoidZero** 의 공식 프로젝트
- **MIT 라이선스** — 상업적 이용·수정·재배포 자유
- 목표 성능: 기존 도구 대비 **10~100배** (`ARCHITECTURE.md` 명시)
- `bmshin94/oxc` 는 원본의 fork이며, `CLAUDE.md` 에 개인 페르소나 가이드가 추가되어 있음

### 사용 중인 대표 프로젝트

| 프로젝트 | 용도 |
|---|---|
| Rolldown (Vite 차세대 번들러) | 파싱 · 변환 · 압축 |
| Nuxt | 파싱 |
| Preact / Shopify / ByteDance / Shopee | 린팅 (oxlint) |
| knip / swc-node / Nova | 모듈 해석 (oxc_resolver) |

---

## 2. 저장소 구조

```
oxc/
├── crates/     41개 Rust 크레이트 (핵심 엔진, .rs 파일 2,151개)
├── apps/       실행 바이너리 (oxlint, oxfmt, shared)
├── napi/       Node.js 바인딩 (parser, minify, transform, transform-react,
│               transform-relay, playground)
├── npm/        npm 배포 패키지 (oxlint, oxfmt, oxc-types,
│               oxlint-plugins, oxlint-plugin-eslint, runtime)
├── tasks/      테스트/자동화 (coverage: test262 · babel · typescript)
├── packages/   워크스페이스 패키지
├── .agents/    AI 에이전트용 Skill 4종
├── AGENTS.md   AI 어시스턴트 기여 가이드
└── CLAUDE.md   @AGENTS.md 참조 + 페르소나 가이드
```

### 2.1 컴파일러 파이프라인 (crates/)

소스코드 한 줄이 처리되는 순서:

| 단계 | 크레이트 | 역할 |
|:--:|---|---|
| 1 | `oxc_lexer` | 소스 → 토큰 분해 |
| 2 | `oxc_parser` | 토큰 → AST (JS + TS) |
| 3 | `oxc_ast` / `oxc_ast_visit` | AST 자료구조 정의 및 순회 |
| 4 | `oxc_semantic` | 스코프 · 심볼 · 참조 · CFG 분석 |
| 5 | `oxc_linter` | oxlint 엔진 |
| 6 | `oxc_formatter` | oxfmt (Prettier 호환) |
| 7 | `oxc_transformer` | TS/JSX → 구형 JS (Babel 역할) |
| 8 | `oxc_minifier` / `oxc_mangler` | 압축 · 변수명 단축 |
| 9 | `oxc_codegen` | AST → 코드 문자열 출력 |

**지원 크레이트**

- `oxc_allocator` — 아레나 메모리 할당기 (성능 핵심)
- `oxc_language_server` — 에디터용 LSP 서버
- `oxc_diagnostics` — 소스 위치 기반 에러 리포팅
- `oxc_formatter_css / json / yaml / graphql` — 다른 언어 포매터
- `oxc_react_compiler` / `oxc_type_checker` — 실험적 영역
- `oxc_isolated_declarations` — TS 선언 파일 생성

### 2.2 린트 규칙 현황

규칙 파일 **946개**, 플러그인 **16종**:

| 플러그인 | 규칙 수 | 플러그인 | 규칙 수 |
|---|---:|---|---:|
| eslint | 208 | vue | 46 |
| unicorn | 139 | jsx_a11y | 36 |
| typescript | 110 | import | 33 |
| react | 85 | oxc (자체) | 27 |
| vitest | 73 | jsdoc | 23 |
| jest | 60 | nextjs | 21 |
| shared | 54 | promise | 16 |
| | | node | 11 |
| | | react_perf | 4 |

### 2.3 배포 패키지 버전 (분석 시점)

- `oxlint` — v1.83.0
- `oxc-parser` — v0.150.0
- `@oxlint/plugins` — v1.83.0

### 2.4 .agents/skills — AI 에이전트용 스킬

| 스킬 | 내용 |
|---|---|
| `migrate-oxlint` | ESLint → Oxlint 마이그레이션 절차 |
| `migrate-oxfmt` | Prettier/Biome → Oxfmt 마이그레이션 절차 |
| `insta-snapshots` | Insta 스냅샷 테스트 비대화형 갱신 |
| `performance-lint-rules` | 린트 규칙 성능 리뷰 체크리스트 |

---

## 3. 왜 빠른가

| 항목 | ESLint 등 기존 도구 | Oxc |
|---|---|---|
| 구현 언어 | JavaScript (인터프리터) | **Rust** (네이티브 컴파일) |
| 파싱 | 도구마다 각각 재파싱 | **AST 1회 생성 후 공유** |
| 메모리 | GC 의존 | `oxc_allocator` 아레나 할당 |
| 병렬성 | 제한적 | **전체 코어 병렬 처리** |

> 핵심: 파싱을 한 번만 수행하고 린트·포맷·변환이 그 결과를 공유하는 **통합 파이프라인** 구조.

---

## 4. 설치 및 사용법

### 4.1 도구로 사용 (가장 간단)

```bash
npx oxlint@latest              # 린트
npx oxlint@latest src/         # 특정 경로
npx oxlint@latest --fix        # 자동 수정
npx oxfmt@latest               # 포맷

pnpm add -D oxlint oxfmt       # 프로젝트 설치
```

### 4.2 기존 도구에서 마이그레이션

```bash
npx @oxlint/migrate                    # eslint.config.js → .oxlintrc.json
npx @oxlint/migrate --type-aware       # 타입 인식 규칙 포함
npx oxfmt@latest --migrate prettier    # Prettier 설정 변환
npx oxfmt@latest --migrate biome       # Biome 설정 변환
```

> 동적 JS/TS 설정(환경 분기 등)이나 디렉터리별 중첩 설정은 자동 변환이 값 스냅샷만 남기므로 수동 이관 필요.

### 4.3 라이브러리로 사용 (Node.js)

```bash
npm i oxc-parser oxc-transform oxc-minify
```

```javascript
import { parseSync } from 'oxc-parser';

const { program, errors, comments } = parseSync('app.tsx', code);
console.log(program.body);   // ESTree 표준 AST
```

### 4.4 에디터 연동

VSCode 마켓플레이스에서 **Oxc** 확장 설치 → `oxc_language_server` 가 LSP로 연결되어 실시간 진단 제공.

### 4.5 저장소 자체 개발

사전 요구: Rust 1.96+, Node.js, pnpm, just

```bash
just submodules      # test262 / babel / typescript 테스트 스위트 클론
just fmt             # 코드 포맷
just test            # 단위/통합 테스트
just conformance     # 적합성 테스트
just ready           # 커밋 후 최종 검사 (PR 전 필수)
just new-rule <name> # 새 린트 규칙 템플릿 생성
cargo lintgen        # 규칙 추가/수정 후 생성 코드 갱신

# 예제 직접 실행
cargo run -p oxc_parser --example parser -- test.js
cargo run -p oxc_linter --example linter -- src/
```

> 테스트 스위트(`tasks/coverage/*`)는 gitignore 대상이므로 검색 시 `rg --no-ignore` 사용.

---

## 5. 정체성 정리 — 플러그인? 스킬? MCP?

**Oxc 본체는 독립 실행 CLI 도구 + Rust/Node 라이브러리다.** 셋 중 어느 것도 아니다.

| 구분 | 해당 여부 | 위치 | 설명 |
|---|:--:|---|---|
| 본체 | ✅ | `crates/`, `apps/` | CLI + 라이브러리 |
| 플러그인 | 🟡 | `npm/oxlint-plugins` | oxlint가 **플러그인을 받는 쪽** |
| 스킬 | 🟡 | `.agents/skills/` | AI 에이전트용 스킬 4종 부록 |
| MCP | ❌ | 없음 | **MCP 서버 미제공** (직접 구축 여지 있음) |

### JS 플러그인 작성 예시

```typescript
import { definePlugin, defineRule } from "@oxlint/plugins";

const rule = defineRule({
  create(context) {
    return {
      Program(node) { /* 규칙 로직 */ }
    };
  },
});

export default definePlugin({
  meta: { name: "oxlint-plugin-amazing" },
  rules: { amazing: rule },
});
```

`eslintCompatPlugin` 으로 감싸면 동일 플러그인이 ESLint에서도 동작한다.

---

## 6. API 토큰 필요 여부

**필요 없음.**

- MIT 라이선스 — 상업적 이용 자유
- 완전 로컬 실행, 네트워크 미사용
- 회원가입 · 로그인 · 과금 없음
- 소스코드 외부 전송 없음 → 보안 감사 통과 용이, 에어갭 환경 사용 가능

---

## 7. GitHub에서 유명한 이유

1. **압도적 속도** — 목표 10~100배, 체감 차이가 큼
2. **VoidZero 배경** — Evan You의 회사, Vite 생태계 수렴
3. **대형 채택 사례** — Rolldown, Nuxt, Preact, Shopify, ByteDance, Shopee
4. **낮은 진입장벽** — `npx oxlint@latest` 한 줄
5. **높은 테스트 신뢰도** — test262(ECMAScript 공식) + Babel + TypeScript 스위트 전수 검증
6. **트렌드 대표성** — esbuild(Go) → SWC(Rust) → Oxc 로 이어지는 흐름
7. **기여 친화적 구조** — `just new-rule` 템플릿, good first issue 관리

---

## 8. 로컬 AI 에이전트 구축 활용법

### 8.1 AST 기반 컨텍스트 압축

파일 전문을 프롬프트에 넣는 대신 AST에서 구조(함수 목록, import 관계, 타입 정의)만 추출해 전달 → **토큰 대폭 절감 + 정확도 향상**. 정규식 기반 코드 파싱의 한계를 근본적으로 해결.

### 8.2 AI 생성 코드 자가수정 루프

```
LLM 코드 생성
   ↓
oxc-parser  → 문법 오류 즉시 감지
   ↓
oxlint      → 946개 규칙으로 버그 패턴 검사
   ↓
oxfmt       → 포맷 통일
   ↓
실패 시 에러 메시지를 LLM에 피드백 → 재생성 (반복)
```

Rust 기반이라 루프를 수십 회 돌려도 지연이 거의 없다.

### 8.3 문서화 모범 사례

`AGENTS.md` 와 `.agents/skills/*/SKILL.md` 는 **AI가 읽을 수 있는 프로젝트 문서**의 실전 레퍼런스. 자체 프로젝트에 구조를 그대로 차용 가능.

### 8.4 프라이버시

네트워크 미사용 · 토큰 불필요 → 사내망/폐쇄망 환경 에이전트 구성 가능.

### 8.5 주의사항

Oxc는 LLM이 아니라 **구문 분석 부품**이다. 추론은 여전히 언어 모델의 역할이며, `oxc_type_checker` 는 실험 단계이므로 완전한 타입 추론에는 `tsc` 또는 `oxlint-tsgolint` 가 필요하다.

---

## 9. React / PHP 로 만들 수 있는가

### 결론

| | Oxc 엔진 재구현 | Oxc를 활용한 제품 |
|---|:--:|:--:|
| React | ❌ 불가능 | ✅ 완전 가능 |
| PHP | ❌ 불가능 | 🟡 간접 가능 |

### React (권장)

```javascript
// 브라우저: WASM 빌드 (napi/playground 가 실제 사례)
import initWasm, { parse } from 'oxc-parser/wasm';

// 서버: Next.js API Route
export async function POST(req) {
  const { parseSync } = await import('oxc-parser');
  const { program, errors } = parseSync('input.ts', await req.text());
  return Response.json({ program, errors });
}
```

구현 가능한 제품: AST 뷰어/플레이그라운드, 코드 품질 대시보드, 린트 결과 시각화, 코드 변환 실습 사이트, AI 코드 리뷰 웹앱

### PHP

공식 NAPI 바인딩이 없으므로 우회 필요:

```php
// 방법 1: CLI 호출
exec('npx oxlint --format=json src/', $output);
$result = json_decode(implode($output), true);

// 방법 2: Node 마이크로서비스 + HTTP 통신
```

Laravel 백엔드가 관리/대시보드를 담당하고 분석은 oxc CLI에 위임하는 구조가 현실적.

### 엔진 재구현이 불가능한 이유

- JS/PHP는 GC + 인터프리터 구조 → 아레나 할당기의 메모리 최적화 재현 불가
- 진정한 멀티스레드 병렬 처리 불가
- 재구현해도 기존 도구와 동일한 성능 → 존재 의의 소멸

---

## 10. 수익화 아이디어

> 전제: Oxc 자체는 무료 오픈소스이므로 **재판매는 무의미**하다. 수익은 **Oxc 위에 얹는 부가가치**에서 발생한다.

### 10.1 Oxc 기반 MCP 서버 (최우선 추천)

저장소에 MCP 서버가 **부재**하며, AI 코딩 도구의 MCP 수요는 급증 중 → 공백 시장.

제공할 도구: `parse_file`, `find_symbol`, `lint_check`, `validate_syntax`, `impact_analysis`, `format_code`

| 플랜 | 가격 | 내용 |
|---|---|---|
| Free | $0 | 로컬 단일 사용자, OSS |
| Pro | $15/월 | 대규모 repo 인덱싱, 캐시 서버 |
| Team | $49/인/월 | 팀 공유 인덱스, 코드베이스 지식 그래프 |
| Enterprise | 협의 | 온프레미스, SSO, 지원 |

- 난이도: 낮음 (MCP SDK + `oxc-parser`)
- MVP 2~3주 · OSS 선공개 후 유료 기능 부가 전략

### 10.2 커스텀 린트 규칙 팩 (구독형)

`@oxlint/plugins` 로 JS 작성 가능 → Rust 지식 불필요.

| 팩 | 내용 | 타겟 |
|---|---|---|
| Security Pack | XSS, dangerouslySetInnerHTML, eval, 하드코딩 시크릿 | 핀테크/커머스 |
| React Perf Pack | 불필요한 리렌더, useEffect 오용, key 누락 | 프론트엔드 팀 |
| A11y Plus | 한국 웹접근성 인증(KWCAG) 기준 | 공공/대기업 |
| Cloud Cost Pack | 서버리스 콜드스타트 유발 패턴 | 스타트업 |
| K-Convention | 국내 기업 코딩 컨벤션 | SI/대기업 |

- 팩당 $29~99/월(팀), 맞춤 제작 300~1,000만원, 번들 $149/월
- 난이도: 중간 (도메인 전문성 필요)

### 10.3 마이그레이션 컨설팅

`.agents/skills/migrate-*` 절차를 서비스화. CI 시간 단축은 직접적 비용 절감이라 결재 설득이 쉽다.

| 규모 | 가격 |
|---|---|
| 소규모 (~100 파일) | 100만원 / 3일 |
| 중규모 (~1,000 파일) | 500만원 / 2주 |
| 대기업 (10,000+) | 2,000만원~ / 1~2개월 |
| 유지보수 리테이너 | 월 100만원 |

- 난이도: 낮음 · **초기 자본 0원, 즉시 시작 가능**
- 단점: 시간 = 매출 구조라 확장성 낮음 → 초기 현금흐름용

### 10.4 코드 품질 SaaS 대시보드

GitHub 연동 → PR마다 자동 분석 → 품질 점수 · 트렌드 시각화. SonarQube의 고속·저가 대안.

| | SonarQube | 대안 제품 |
|---|---|---|
| 속도 | 수십 분 | 수 초 |
| 가격 | $150+/월 | $29/월 |
| 설치 | 무겁고 복잡 | GitHub 앱 연동 |
| AI 리뷰 | 약함 | LLM 연동 강화 |

- 프론트: React/Next.js · 백엔드: Node 또는 PHP/Laravel + oxlint CLI
- Free(공개 repo) / Pro $29 / Team $99
- 난이도: 높음 (인증·결제·웹훅·워커 전부 필요) — 단 React 역량이 직접 무기가 됨

### 10.5 AI 코드 검증 API (B2B)

```javascript
POST /v1/validate  { code, lang }
→ { valid, errors, fixed, score }
```

타겟: AI 코딩 도구 스타트업, 노코드 플랫폼, **LLM 파인튜닝 데이터 품질 필터링**, 코딩 교육 플랫폼

- 종량제 1,000 요청당 $1 / 월정액 $99(10만 요청) / 온프레미스 연 $10,000+
- 난이도: 중간 (기술은 단순, B2B 영업이 관건)

### 10.6 비교 및 추천 로드맵

| 아이디어 | 난이도 | 초기비용 | 수익시점 | 확장성 |
|---|:--:|:--:|:--:|:--:|
| MCP 서버 | 낮음 | 낮음 | 2~3개월 | 높음 |
| 규칙 팩 | 중간 | 낮음 | 3~4개월 | 중간 |
| 컨설팅 | 낮음 | 0원 | 즉시 | 낮음 |
| SaaS | 높음 | 높음 | 6개월+ | 최고 |
| 검증 API | 중간 | 중간 | 4개월 | 높음 |

**단계별 전략**

1. **0~1개월** — 마이그레이션 컨설팅으로 현금흐름 + 실전 경험 확보
2. **1~4개월** — MCP 서버를 OSS로 공개, 스타/커뮤니티/브랜딩 확보
3. **4개월~** — SaaS 대시보드 구축, 앞 단계 고객을 그대로 유입

---

## 11. 참고 링크

| 항목 | 주소 |
|---|---|
| 이 저장소 | <https://github.com/bmshin94/oxc> |
| 원본 저장소 | <https://github.com/oxc-project/oxc> |
| 공식 문서 | <https://oxc.rs> |
| 플레이그라운드 | <https://playground.oxc.rs> |
| Oxlint 가이드 | <https://oxc.rs/docs/guide/usage/linter> |
| Oxfmt 가이드 | <https://oxc.rs/docs/guide/usage/formatter> |
| JS 플러그인 가이드 | <https://oxc.rs/docs/guide/usage/linter/js-plugins> |
| 마이그레이션 도구 | <https://github.com/oxc-project/oxlint-migrate> |
| VoidZero | <https://voidzero.dev> |
| Discord | <https://discord.gg/9uXCAwqQZW> |
