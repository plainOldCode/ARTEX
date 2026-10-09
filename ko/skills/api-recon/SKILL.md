---
name: api-recon
description: 웹사이트 API 엔드포인트를 수집할 때 이 skill을 호출한다.
---

# API Recon(프론트엔드 인터페이스 정찰, 前端接口侦察)

**승인된** 전제 아래에서 최대한 완전하게 발견한다: **백엔드 API**(경로, 메서드, 파라미터, 응답 본문), **프론트엔드 라우트**, **UI 기능 트리거 지점**(Tab, 팝업, 테이블 작업 등).

---

## 경계와 금지(Agent 필독 · 위반 시 범위 초과)

이 skill은 **API / 파라미터 면 정찰만** 수행하며, 취약점 발굴 또는 침투 공격 단계가 아니다.

### 작업 경계

| 범위 | 허용 | 금지 |
|---|---|---|
| **목표** | path, method, 파라미터, 라우트, UI 트리거 지점 열거 | SQLi/XSS/권한 초과/브루트포스/fuzz 취약점, 요청 변조 공격, 파괴적 작업 |
| **인증** | Hook + stub/mock으로 **클라이언트** 로그인 게이트 우회 | 사용자에게 계정·비밀번호를 요구하거나 추측; 실제 로그인 폼 제출 시도 |
| **런타임** | 자격증명 없이 인터페이스를 hook하고, mock 응답으로 SPA를 로그인 후 셸로 진입 | 실제 백엔드 세션에 의존해야만 진행 가능한 흐름 |

### 자격증명 없는 동적 분석(Phase 3 기본)

1. `preload.js` / `runtime_harvest.js`로 로그인, 권한, 메뉴 등 bootstrap 인터페이스를 **가로채어 stub**한다;
2. 비즈니스 조회 인터페이스에는 **구조가 올바르고, 비즈니스 코드가 성공이며, 데이터는 비어 있어도 되는** mock body를 반환한다;
3. 백엔드가 없거나 401 환경에서도 프론트엔드가 로그인 후 페이지를 렌더링하게 하여 더 많은 XHR/fetch/WebSocket을 유발한다;
4. **빈 데이터, 빈 테이블, 플레이스홀더 UI는 모두 예상된 결과**다 — 이를 이유로 실제 로그인이나 취약점 테스트로 방향을 바꾸지 않는다.

**한 줄 요약**: mock으로 프론트엔드 라우트와 컴포넌트 마운트를 벌려놓고 **outbound 요청만 기록**한다; 백엔드가 무엇을 반환하는지는 중요하지 않고, 중요한 것은 프론트엔드가 **추가로 어떤 인터페이스를 보내는가**다.

### 흐름상 절대 금지

| 금지 | 대안 |
|---|---|
| Phase 1 완료 전에 grep/curl/Read로 main entry `index-*.js`에서 API path 추출 | `OUTDIR/harvest_static.py` 실행 |
| `extract_apis.py` 등 harvest를 대체하는 스크립트를 직접 작성 | `OUTDIR/harvest_static.py`를 수정 후 재실행 |
| 같은 grep/명령이 2회 이상 실패해도 반복 | 전략 교체: tool_logs 읽기, harvest 수정, reference 확인 |
| 게이트 A/B를 건너뛰고 `scripts/` 원본을 바로 실행 | OUTDIR로 복사 후 대상에 맞게 수정 |
| 실제 사용자명/비밀번호, OTP, OAuth 등 인증 | stub/mock(위 참조) |
| 「실제 데이터 확보」를 이유로 stub을 건너뛰고 권한 초과/인젝션 테스트 수행 | outbound만 기록, 정찰(recon) 범위 내 |
| 삭제, 민감 데이터 내보내기, 일괄 쓰기 등 되돌릴 수 없는 작업 | coverage 클릭도 동일 |
| runtime + 동적 열거를 완료하지 않고 모든 페이지와 인터페이스를 확보했다고 주장 | 「완료 정의」 참조 또는 한계 표기 |
| 파라미터 트리거 매트릭스 + diff를 완료하지 않고 모든 파라미터를 파악했다고 주장 | Phase 3b 매트릭스 + Phase 5 diff |
| 단일 runtime 샘플로 필수/선택을 단정 | 다중 샘플 diff 또는 검증 규칙/에러로 역추론 |

---

## 2계층 모델 + 실행 모드

| 계층 | 산출 | 한계 |
|---|---|---|
| **정적**(JS bundle) | 전체 endpoint 경로, 라우트 초안, 페이로드 조립 지점 필드 후보 | HTTP 메서드 없음; 파라미터는 Phase 1b 필요; 런타임에 조립되는 URL 누락 |
| **런타임**(활성 세션) | 메서드 + body + 응답 + 동적 URL + WS/SSE; 다중 샘플 diff로 파라미터 보완 | 페이지가 실제 렌더링되어야 요청이 발생; 단일 샘플로는 필수/선택 판정 불가 |

| 실행 모드 | 엔진 | 적합 |
|---|---|---|
| **depth** | `runtime_harvest.js`(Puppeteer) | API 목록, METHOD/params/응답 본문, WS/SSE, 재현 가능한 일괄 실행 |
| **coverage** | browser + `preload.js` | Tab/팝업/테이블 클릭, 기능 지점 커버리지가 더 깊음 |
| **both** | depth 후 coverage | 가장 완전, 가장 오래 걸림 |

**파라미터 방법론**(범용 스크립트 없음): path는 harvest/정규식; 파라미터는 **앵커 확장 윈도우 + UI 바인딩 체인 + 다중 샘플 diff + 에러 역추론**(grep 레시피는 [reference.md](reference.md) J절).

---

## 완료 정의

모두 충족해야 정찰 완료를 주장할 수 있다:

- [ ] **정적**: Phase 1 harvest가 `api_static.txt`, `routes.txt`, `js/` 산출
- [ ] **런타임**: depth 또는 coverage 중 최소 하나; coverage/both는 **Hook 유효 + 동적 열거 루프** 필요
- [ ] **셸 진입**: 비즈니스 path 접근 시 `/login`이 아님(hash 라우트 주의)
- [ ] **파라미터**: coverage/both로 파라미터 트리거 매트릭스 + `param_samples.json` 완료; Phase 5에서 `params_merged.json` 병합
- [ ] **심층**(모듈 페이지가 빈 경우): Phase 4 권한 트리 복원 후 재실행, **module 수준 API**(locale/bootstrap만 아님)가 나타날 때까지
- [ ] **전달**: Phase 5 산출물 완비(Phase 5 산출물 표 참조); `insert_assets`로 서비스·엔드포인트 자산 기록

---

## 스크립트와 게이트

`scripts/`는 참조 템플릿일 뿐이며, 원본을 그대로 실행해 최종 결과로 삼는 것을 **금지**한다.

**규칙**: 먼저 읽기 → 대상에 맞게 수정 → `OUTDIR`(예: `recon/`)에 기록 → `CHANGES.md` 기록; 맞지 않으면 방법론에 따라 재작성, 구조만 차용.

| 게이트 | 시점 | 참조 스크립트 → OUTDIR 복사본 | 자주 고쳐야 할 항목 |
|---|---|---|---|
| **A(정적)** | Phase 0 후, **처음** harvest/spider 실행 전 | `harvest_static.py` / `spider_mpa.py` | **대부분 사이트는 기본 regex로 바로 실행 가능**; manifest/방언이 맞지 않을 때만 endpoint 정규식, webpack/Vite `publicPath`, MPA exclude/cookie 수정 |
| **B(런타임)** | Phase 2 후, depth/coverage 실행 전 | `runtime_harvest.js` / `preload.js` + `config.json` | Cookie/localStorage 키, neutralize 성공값, stubs, login 정규식, api 접두사, hash/history |

**SPA 강제 순서**(교환 불가; Phase 번호가 「먼저 탐색 후 스크립트」보다 우선):

| 단계 | 필수 | 금지 |
|---|---|---|
| Phase 0 완료 후 | 다음 Bash = `python3 OUTDIR/harvest_static.py <URL> OUTDIR` | curl/grep/Read로 main entry `index-*.js`(통상 >500KB) 접근 |
| 게이트 A | 스크립트 복사 → 필요시 소폭 수정 → **즉시 실행** | 먼저 수동으로 API 추출 후 harvest 여부 결정 |
| Phase 1 완료 전 | `wc -l`로 산출 검증; 404면 harvest 수정 후 재시도 | extract 스크립트 직접 작성; 미다운로드 URL에 반복 grep |
| Phase 1b부터 | grep은 `OUTDIR/js/*.js`에만 | harvest 대신 main bundle 사용 |

- ✅ `harvest_static.py` 복사 → (선택) regex 수정 → **즉시 실행**
- ❌ main bundle curl → grep 반복 → 임시 extract 작성 → 마지막에야 harvest
- **MPA**: Phase 0 후 다음 Bash = `python3 OUTDIR/spider_mpa.py ...`

---

## 도구와 출력 제약

| 제약 | 설명 |
|---|---|
| 대용량 파일 | >100KB인 `index-*.js`는 Read/grep으로 컨텍스트에 넣기 **금지**; OUTDIR 스크립트로 일괄 처리 |
| grep 출력 | 반드시 `\| head -20` 또는 `-m 5`; 대화에는 path 요약만 남기고 bundle 조각을 붙여넣지 않음 |
| 검증 | `wc -l`, `ls \| wc -l` 사용; 디렉터리 전체를 Read하지 않음 |
| regex 초기 탐색 | 선택, ≤1회, ≤50KB 소형 chunk 또는 HTML에 한함; 정식 정적은 harvest 기준 |
| reference | 레시피/템플릿/문제 해결은 [reference.md](reference.md) 참조, 전문을 inline으로 반복하지 않음 |

---

## 실행 로드맵

```
Phase 0 分类 + OUTDIR
  → 门禁 A → Phase 1 harvest（★ 立刻运行 ★）
  → Phase 1b 参数逆向
  → Phase 2 鉴权三道门 → config.json
  → 门禁 B → Phase 3 运行时 + 参数矩阵
  → Phase 4 权限树（必要时）→ 重跑 Phase 3
  → Phase 5 合并报告 + insert_assets批量插入所有发现的服务、端点api资产，无论如何插入时不允许漏掉已发现的资产
```

순서대로 체크; **앞 항목이 미완료면 다음 Phase에 진입 불가**.

1. [ ] **Phase 0**: SPA/MPA 초기 분류; `OUTDIR` 생성 → [Phase 0](#phase-0--분류)
2. [ ] **게이트 A + Phase 1**: 스크립트 복사 → **즉시** harvest → `wc -l` 검증 → [Phase 1](#phase-1--정적)
3. [ ] **Phase 1b**: 앵커 확장 윈도우 + 바인딩 레이어 → `param_candidates.json` → [Phase 1b](#phase-1b--파라미터-역공학)
4. [ ] **Phase 2**: 인증 3관문 → `config.json` → [Phase 2](#phase-2--인증-3관문)
5. [ ] **게이트 B**: runtime 스크립트 조정 → [Phase 3](#phase-3--런타임)
6. [ ] **Phase 3**: depth / coverage / both; 셸 진입 확인; 파라미터 트리거 매트릭스 → `param_samples.json`
7. [ ] **Phase 4**(필요 시): 권한 트리 → stubs patch → Phase 3 재실행 → [Phase 4](#phase-4--권한-트리-복원)
8. [ ] **Phase 5**: 산출물 병합 + 보고 + `insert_assets` → [Phase 5](#phase-5--병합과-보고)

---

## Phase 0 — 분류

엔트리 HTML을 가져오고 **`OUTDIR` 생성**(skill 내 `scripts/`는 수정하지 않음):

- **SPA**: 빈 껍데기 + `<div id=app>` + chunk → Phase 1–5
- **MPA**: SSR + `<form>`, endpoint bundle 없음 → 게이트 A 후:

```bash
python3 recon/spider_mpa.py <BASE_URL> <OUTDIR> [--cookie "session=..."] [--max 300] [--depth 5] [--exclude "logout|delete|destroy"]
```

`forms.txt`, `links.txt`, `api_inline.txt` 산출. SPA에서 forms ≈ 0이면 → Phase 1로 전환.

---

## Phase 1 — 정적

[스크립트와 게이트](#스크립트와-게이트) · [도구와 출력 제약](#도구와-출력-제약)을 준수.

```bash
python3 recon/harvest_static.py <BASE_URL> <OUTDIR>
```

harvest: HTML script 파싱 → webpack/Vite manifest → 전체 lazy chunk 다운로드 → `js/`, `api_static.txt`, `routes.txt`, `chunkmap.txt` 산출.

```bash
wc -l OUTDIR/api_static.txt OUTDIR/routes.txt
ls OUTDIR/js | wc -l
```

- chunk 수 vs manifest: 404면 harvest를 고쳐 재시도, chunk를 수동으로 하나씩 curl하지 않음
- `api_static.txt`가 너무 적으면 → OUTDIR 내 endpoint 정규식을 완화 후 재실행(reference 참조)

### Phase 1b — 파라미터 역공학

path는 Phase 1에서 온다; 파라미터 필드는 별도 정찰 필요. grep 규칙은 [도구와 출력 제약](#도구와-출력-제약).

**완료 기준**: 중요 인터페이스에 대해 답할 수 있어야 한다 — 필드명, 전송 위치, 타입 추론, 필수 여부, 샘플값, 신뢰도.

#### 1b.0 — 전송 형태

| 형태 | 파라미터 위치 | 정적에서 우선 확인 |
|---|---|---|
| REST JSON | body + query | path 앵커 옆 `(params\|data\|body)\s*:\s*\{` |
| GraphQL | `variables` | gql 템플릿, `$page: Int` |
| 전통 form | urlencoded | `<form>`, `FormData` |
| 파일 업로드 | multipart | `FormData.append` |
| 경로 파라미터 | `/user/:id` | 라우트 테이블 + `useParams` / `$route.params` |
| 암호화/서명 | `sign`/`data`로 감쌈 | Hook으로 암호화 함수 인자(reference D절) |

산출: 인터페이스마다 `transport: query|json|form|graphql|encrypted` 표기.

#### 1b.1 — 앵커 확장 윈도우

알려진 path를 앵커로 삼아 창을 넓혀 페이로드 조립 객체를 찾는다:

```bash
grep -n '"/api/user/list"' OUTDIR/js/*.js | head -20
grep -rhoaE '.{0,120}("/api[^"]+").{0,200}' OUTDIR/js/*.js | head -20
grep -rhoaE '(params|data|body|payload)\s*:\s*\{' OUTDIR/js/*.js | head -20
```

| 래퍼 레이어 | 파라미터 단서 |
|---|---|
| axios 인스턴스 | `data` / `params` |
| 통합 request | 인터셉터가 전역 필드 주입 |
| OpenAPI 클라이언트 | 생성된 method 시그니처 |
| React Query / SWR | hook 두 번째 인자 |
| Vue composable | composable 인자 |

타입 잔재: `yup`/`zod`/rules, `Form.Item name=`, 내장 Swagger.

→ `param_candidates.json`: `{ path, fields[], source: "static-callsite", confidence }`

#### 1b.2 — 바인딩 레이어

```
Form field → onFinish/handleSubmit → transform → API payload
```

| 바인딩 소스 | 방법 |
|---|---|
| 폼 submit | submit → transform → API 추적 |
| 테이블 검색 | `getFieldsValue()` → `params` |
| 라우트 | `:id` / `?tab=` |
| 인터셉터 | 전역 `tenantId`, 페이징, sign |
| 열거형 select | `options` → API 열거값 |

DevTools call stack으로 `fetch`/`XHR.send`에서 위로 올라가 페이로드 조립 함수를 추적.

#### 1b.3 — 페이로드 조립 3질문(≠ Phase 2 인증 3관문)

| 질문 | 답할 내용 |
|---|---|
| **조립** | payload가 어디서 build되는지, transform 흔적 |
| **검증** | required, pattern, enum |
| **전송** | path / query / body / multipart / 헤더 |

인터셉터 게이트(Phase 2)에서 전역 주입 필드(Authorization, `X-Tenant-Id`, sign)도 함께 읽는다.

#### 1b.4 — Phase 3와의 연결

후보 필드는 정적/바인딩 레이어에서 온다; **필수/선택/조건 의존**은 Phase 3 파라미터 매트릭스 + diff + Phase 5 에러 역추론으로 확정.

---

## Phase 2 — 인증 3관문

`OUTDIR/js/`에서 grep(`head` 포함)하고 `config.json`에 기록(레시피는 reference):

| 관문 | 질문 | 키워드 |
|---|---|---|
| **렌더 게이트** | 로그인 여부를 어떻게 판단하는가? | `isLogin`, `getToken`, Cookie/localStorage |
| **인터셉터 게이트** | 무엇이 `/login`으로 튕기는가? | `response_code`, `errno`, axios interceptor |
| **콘텐츠 게이트** | 메뉴/권한은 어디서 오는가? | `menu`, `permission`, `role`, `acl`, `routes` |

localStorage 키명을 자격증명으로 취급하지 않는다 — chunk/요청 체인에서 확인해야 한다.

**출구 = 게이트 B**: 결론을 `config.json`에 반영하고, `OUTDIR/runtime_harvest.js` / `preload.js`를 수정.

### Phase 2b — API 관찰(선택)

OUTDIR 내 `preload.js`로 세션 키명, Authorization, 중첩 API URL 확인:

| 설정 | 산출 |
|---|---|
| `recordDetail: true` | `__API_RECON_DETAIL__` |
| `observe.xhrHeaders: true` | headers 관찰 |
| `extractUrlsFromResponse: true` | 응답 내 하위 API |
| `observe.storageReads/cookieReads: true` | config에 역반영 |
| `neutralizeVueRouter: true` | `__API_RECON_ROUTES__` |

coverage 매 라운드 내보내기: `__API_RECON_LOG__`, `__API_RECON_DETAIL__`, `__API_RECON_ROUTES__`, `__API_RECON_OBSERVE__`.

---

## Phase 3 — 런타임

게이트 B를 통과해야 하며, [경계와 금지](#경계와-금지agent-필독--위반-시-범위-초과) 준수 · 자격증명 없는 mock 전략.

`config.json`에 `"runtimeMode": "depth" | "coverage" | "both"` 설정(템플릿은 reference).

### Hook과 stub(depth + coverage 공용)

| 계층 | 범위 | 목적 |
|---|---|---|
| L1 정밀 | auth/권한/bootstrap stub | 첫 화면 인증 통과 |
| L2 네거티브 보정 | 모든 JSON 응답 | 미로그인 코드 → 성공 |
| L3 폴백 | L1에 걸리지 않은 `/api` 등 | 빈 성공 body, UI를 벌려놓음 |

- **depth**: fake auth + `forward`로 비즈니스 코드 변경 + `stubs`; `routes` 순회(hash/history); `runtime_api.json` 산출
- **coverage**: **document-start**에서 `preload.js` 주입(CDP `addScriptToEvaluateOnNewDocument` 또는 Userscript)

검증: `window.__API_RECON_PRELOAD__` 존재; 비즈니스 path가 `/login`으로 돌아가지 않음.

```bash
cd recon && npm install
node runtime_harvest.js config.json
```

### 3b — coverage 동적 열거(필수)

1. 주 내비게이션/사이드바 — 항목마다 클릭, 네트워크 1–3s 대기
2. Tab — `role=tab`, `.ant-tabs-tab`
3. 테이블 — 첫 행의 보기/편집/상세
4. 툴바 — 내보내기, 필터, 신규(**되돌릴 수 없는 삭제는 회피**)
5. 모듈 진입마다 — API/라우트 병합
6. SPA — `routes.txt`가 커버하지 않는 path를 제어된 `pushState`(MPA는 금지)

**파라미터 트리거 매트릭스**(필수): 모듈마다 작업 유형별로 한 번씩 기록, **다중 샘플 diff**:

| 작업 | 보통 추가로 나타나는 파라미터 |
|---|---|
| 목록 첫 화면 | 페이징 + 기본 필터 |
| 검색 클릭 | keyword, filter |
| 고급 필터 | 더 많은 optional |
| 신규/편집 | 완전한 entity |
| 일괄/내보내기/정렬 | `ids[]`, `exportType`, `sortField` |

**stub 하에서도 outbound body/headers는 실제 그대로** — 요청을 기준으로 삼는다. 기록 → `scan_raw.json`, `param_samples.json`, `api_detail.json`.

- **Vue**: `neutralizeVueRouter: true` + document-start preload
- **React**: `routes.txt` + 사이드바 클릭 + `pushState`
- **both**: 먼저 3a depth, 이후 3b coverage

---

## Phase 4 — 권한 트리 복원

**트리거**: 모듈 페이지가 빈 경우 / 라우트마다 bootstrap(locale 등)만 있는 경우 → 콘텐츠 게이트 미통과.

| 현상 | 의미 |
|---|---|
| 셸 진입 성공 | 렌더 게이트 + 인터셉터 게이트 통과 |
| 사이드바 항목 누락/클릭 시 빈 화면 | stub shape 또는 권한 코드 불완전 |
| 라우트마다 API가 동일하고 극소 | `v-if permission` 미통과 |
| `routes.txt`가 bundle보다 훨씬 적음 | auth 모듈에서 보완 필요 |

```bash
grep -rhoaE '"/api[^"]*(permission|perm|role|menu|acl)[^"]*"' OUTDIR/js/*.js | sort -u | head -30
grep -rhoaE 'userRouteAuth|getResultTree|routeMap|routeLink|menuList|authList' OUTDIR/js/*.js | head -20
```

전형적 체인: `role_permissions`(flat codes) + `permissions/all`(tree) → `getResultTree` → `userRouteAuth[CODE].url`.

```bash
python3 recon/extract_route_map.py recon/js recon/
python3 recon/build_perm_tree.py recon/js recon/ --config recon/config.json
```

중간 산출: `route_map.json`, `userRouteAuth.json`, `permissions_tree.json`, `*_stub.json`, `perm_codes_all.txt`.

stub 검증: 외부 `response_code`가 인터셉터 게이트와 일치; flat codes와 tree 정렬 일치; `routes`가 `route_map`의 전체 link를 커버.

`config.json` 갱신 후 **Phase 3 재실행**. 대형 SPA는 `waitUntil`, `routeTimeout`, `perRouteMs` 조정 가능(reference A3/I절).

---

## Phase 5 — 병합과 보고

### 산출물 표

| 파일 | 단계 | 내용 |
|---|---|---|
| `js/`, `api_static.txt`, `routes.txt`, `chunkmap.txt` | 1 | 정적 bundle과 path |
| `param_candidates.json` | 1b | 정적 파라미터 필드 후보 |
| `config.json` | 2 | 3관문 + runtime 설정 |
| `runtime_api.json` | 3a | depth 상세 기록(WS/SSE 포함) |
| `param_samples.json`, `scan_raw.json`, `api_detail.json` | 3b | 다중 샘플, 클릭 로그, detail |
| `route_map.json` 등 | 4 | 권한 트리 중간 파일(실행 시) |
| `params_merged.json` | 5 | 병합된 파라미터 필드 + 신뢰도 |
| `api_merged.txt` | 5 | `METHOD /path [params] [static\|runtime\|both]` |
| `site_map.json` | 5 | 라우트, API, params, 기능 지점, 한계 |
| **insert_assets** | 5 | 모든 서비스, 엔드포인트 자산을 자산 라이브러리에 기록 |

### 5b — 파라미터 병합

`param_samples.json`에서 diff, **범용 병합 스크립트 없음**. 신뢰도 규칙은 reference J7(높음/중간/낮음/트리거 대기).

### 5c — 에러 역추론

승인 범위 내에서 불완전 요청을 보내 400을 읽는 것(**파라미터 정찰이지 취약점 테스트가 아님**): `field 'x' is required`, 열거 오류 등. `data` 래퍼, `variables`, 암호화 전 `bizData`에 주의.

보고서에는 runtimeMode, 정적/런타임 API 수, 파라미터 신뢰도, 미커버 모듈, 참조 스크립트 대비 `CHANGES.md` 요약을 명시해야 한다.

`site_map.json` 권장 구조:

```json
{
  "site": "https://example.com",
  "runtimeMode": "both",
  "appType": "vue-spa",
  "routeGuardStrategy": ["nav-neutralize", "L1-auth", "L2-patch", "forward"],
  "apisFromStatic": [],
  "apisFromRuntime": [],
  "apis": [],
  "params": [{ "method": "POST", "path": "/api/user/list", "transport": "json", "fields": [] }],
  "frontendRoutes": [],
  "routesVerifiedByClick": [],
  "featuresTriggered": [],
  "limitations": ""
}
```

더 많은 필드와 grep 레시피는 [reference.md](reference.md) 참조.

---

## 범용 설명

- **프레임워크 무관**: webpack/Vite/Angular lazy load 방법은 동일
- **전송**: REST/JSON, GraphQL, WebSocket, SSE; gRPC-web은 범위 외
- **SSR**: 클라이언트 fetch는 기록 가능; RSC/Server Actions는 완전 열거 불가
- **사각지대**: JSVMP, WASM, HMAC/mTLS 강검증 → 정적 + 한계 표기
- **파라미터 사각지대**: 조건 연동, hidden params, WASM 페이로드 조립 → 「트리거 대기」/「도달 불가」
- **정적은 안전망**: runtime이 막혀도 정적은 endpoint 열거 가능

---

## 추가 자원

- Grep 레시피, `config.json` 템플릿, 문제 해결, Hook, 파라미터 역공학 J절, site_map 템플릿: **[reference.md](reference.md)**
- 참조 스크립트 경로는 [스크립트와 게이트](#스크립트와-게이트) 표 참조
