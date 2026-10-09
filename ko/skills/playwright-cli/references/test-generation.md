# 테스트 생성(plan → generate → heal)

`playwright-cli`로 Playwright 테스트를 작성하고 유지보수하는 엔드 투 엔드 워크플로. `playwright-cli`의 모든 액션은 동등한 Playwright TypeScript를 출력하며, 이 생성된 코드가 모든 테스트의 원료가 된다. 아래 절들은 독립적으로 사용할 수 있다:

- **생성 방식** — 다른 모든 것이 의존하는 핵심 메커니즘: 액션이 TypeScript가 되는 방식과 어설션 추가 방법.
- **Plan(계획)** — 앱을 탐색하고 무엇을 테스트할지 기술하는 스펙 파일을 만든다.
- **Generate(생성)** — 스펙을 Playwright 테스트 파일로 바꾼다. 스펙이 모호하거나 낡았으면 갱신한다.
- **Heal(치유)** — 실패한 테스트를 진단하고 코드를 고치며, 스펙을 실제와 대조한다.

Plan / generate / heal은 같은 메커니즘에 의존한다: `npx playwright test --debug=cli`를 백그라운드에서 실행한 뒤 `playwright-cli attach tw-XXXX`로 일시정지된 페이지를 인터랙티브하게 구동한다. debug/attach 동작은 [playwright-tests.md](playwright-tests.md) 참조.

---

## 0. 생성 방식

`playwright-cli`로 수행하는 모든 액션은 대응하는 Playwright TypeScript 코드를 생성한다. 이 코드는 출력에 나타나며 테스트 파일에 바로 복사할 수 있다.

```bash
# Start a session
playwright-cli open https://example.com/login

# Take a snapshot to see elements
playwright-cli snapshot
# Output shows: e1 [textbox "Email"], e2 [textbox "Password"], e3 [button "Sign In"]

# Fill form fields - generates code automatically
playwright-cli fill e1 "user@example.com"
# Ran Playwright code:
# await page.getByRole('textbox', { name: 'Email' }).fill('user@example.com');

playwright-cli fill e2 "password123"
# Ran Playwright code:
# await page.getByRole('textbox', { name: 'Password' }).fill('password123');

playwright-cli click e3
# Ran Playwright code:
# await page.getByRole('button', { name: 'Sign In' }).click();
```

### 테스트 파일 만들기

생성된 코드를 모아 Playwright 테스트로 만든다:

```typescript
import { test, expect } from '@playwright/test';

test('login flow', async ({ page }) => {
  // Generated code from playwright-cli session:
  await page.goto('https://example.com/login');
  await page.getByRole('textbox', { name: 'Email' }).fill('user@example.com');
  await page.getByRole('textbox', { name: 'Password' }).fill('password123');
  await page.getByRole('button', { name: 'Sign In' }).click();

  // Add assertions
  await expect(page).toHaveURL(/.*dashboard/);
});
```

### 시맨틱 로케이터 사용

생성된 코드는 가능한 한 role 기반 로케이터를 사용하며, 이쪽이 더 견고하다:

```typescript
// Generated (good - semantic)
await page.getByRole('button', { name: 'Submit' }).click();

// Avoid (fragile - CSS selectors)
await page.locator('#submit-btn').click();
```

### 녹화 전에 탐색

액션을 녹화하기 전에 스냅샷으로 페이지 구조를 파악한다:

```bash
playwright-cli open https://example.com
playwright-cli snapshot
# Review the element structure
playwright-cli click e5
```

### 어설션 수동 추가

생성된 코드는 액션은 담지만 어설션은 담지 않는다. 권장 매처 중 하나로 테스트에 expectation을 추가한다:

- `toBeVisible()` — 요소가 렌더링되고 보인다
- `toHaveText(text)` — 요소 텍스트 콘텐츠가 일치한다
- `toHaveValue(value) / toBeEmpty()` — input/select 값이 일치한다
- `toBeChecked() / toBeUnchecked()` — 체크박스 상태가 일치한다
- `toMatchAriaSnapshot(snapshot)` — 페이지(또는 로케이터)가 부분 접근성 스냅샷과 일치한다

어설션에 쓸 로케이터 식은 `playwright-cli generate-locator <target>`으로 만들고, 기대값은 snapshot/eval 명령으로 수집한다.

텍스트 콘텐츠를 어설션할 때는 생성된 로케이터에 요소 본문의 텍스트가 포함되지 않게 한다. `getByTestId()`나 `getByLabel()`은 보통 텍스트 어설션과 잘 맞는다. 로케이터가 텍스트 기반일 때는 대신 `toBeVisible()`을 선호한다.

매칭할 스냅샷이 모든 정보를 담을 필요는 없다 — 어설션에 필요한 것만 캡처한다. 불안정한 값에는 정규식을 쓸 수 있다.

```bash
# Get a stable locator for an element ref to use in the assertion
playwright-cli --raw generate-locator e5
# getByRole('button', { name: 'Submit' })

# Capture expected text content for toHaveText
playwright-cli --raw eval "el => el.textContent" e5

# Capture expected input value for toHaveValue/toBeEmpty
playwright-cli --raw eval "el => el.value" e5

# Capture expected aria snapshot for toMatchAriaSnapshot/toBeChecked
# (whole page, or use a ref to scope to a region)
playwright-cli --raw snapshot
playwright-cli --raw snapshot e5
```

```typescript
// Generated action
await page.getByRole('button', { name: 'Submit' }).click();

// Manual assertions using the outputs above:
await expect(page.getByRole('alert', { name: 'Success' })).toBeVisible();
await expect(page.getByTestId('main-header')).toHaveText('Welcome, user');
await expect(page.getByRole('textbox', { name: 'Email' })).toHaveValue('user@example.com');
await expect(page.getByRole('checkbox', { name: 'Enable notifications' })).toBeChecked();

// toMatchAriaSnapshot on the whole page, finds a matching region
await expect(page).toMatchAriaSnapshot(`
  - heading "Welcome, user"
  - link /\\d+ new messages?/
  - button "Sign out"
`);

// toMatchAriaSnapshot scoped to a region
await expect(page.getByRole('navigation')).toMatchAriaSnapshot(`
  - link "Home"
  - link /\\d+ new messages?/
  - link "Profile"
`);
```

---

## 1. 계획(Planning)

목표: 테스트할 시나리오를 나열하는 스펙 파일(예: `specs/<feature>.plan.md`)을 만든다. 스펙은 **반드시** 파일로 기록한다.

### 1.1 사전 조건: 워크스페이스

무엇보다 먼저 워크스페이스에 Playwright가 설치돼 있는지 확인한다:

```bash
# Either of these confirms a workspace:
test -f playwright.config.ts || test -f playwright.config.js
npx --no-install playwright --version
```

Playwright 설치가 없으면 부트스트랩하고 기본값 선택은 사용자에게 맡긴다:

```bash
npm init playwright@latest
```

### 1.2 사전 조건: 시드 테스트

**시드 테스트(seed test)**는 모든 시나리오가 시작하는 상태로 페이지를 진입시키는 최소 테스트다: 앱으로의 탐색, 필요한 로그인, 기능 플래그 등. 시나리오는 시드 *이후*의 새 출발을 가정한다. `--debug=cli`는 이 테스트 *내부*에서 일시정지하므로, 시드가 모든 계획·생성 세션의 시작점이다.

최소 시드:

```ts
// tests/seed.spec.ts
import { test } from '@playwright/test';

test('seed', async ({ page }) => {
  await page.goto('https://example.com/');
});
```

권장 — 탐색을 픽스처에 넣어 시나리오 테스트가 재사용하게 한다:

```ts
// tests/fixtures.ts
import { test as baseTest } from '@playwright/test';
export { expect } from '@playwright/test';

export const test = baseTest.extend({
  page: async ({ page }, use) => {
    await page.goto('https://example.com/');
    await use(page);
  },
});
```

```ts
// tests/seed.spec.ts
import { test } from './fixtures';

test('seed', async ({ page }) => {
  // Fixture already navigates. This empty body tells agents where to start.
});
```

시드가 없으면 최소한 앱으로 탐색하는 시드를 만든다.

### 1.3 앱 탐색

시드로 앱을 백그라운드에서 띄우고 attach한다:

```bash
PLAYWRIGHT_HTML_OPEN=never npx playwright test tests/seed.spec.ts --debug=cli
# wait for "Debugging Instructions" and the session name tw-XXXX
playwright-cli attach tw-XXXX
```

resume으로 시드를 통과시킨 뒤 앱을 탐색한다:

```bash
playwright-cli resume                   # resume so that seed test runs fully
playwright-cli snapshot                 # inventory of interactive elements
playwright-cli click e5                 # follow a flow
playwright-cli eval "location.href"     # read URL / state
playwright-cli show --annotate          # ask the user to point at something
```

파악할 것:

- 인터랙티브 표면(폼, 버튼, 목록, 필터, 모달).
- 주요 사용자 여정의 엔드 투 엔드 흐름.
- 엣지 케이스: 빈 상태, 유효성 에러, 매우 긴 입력, 경계값.
- 영속성: 리로드, local/session 스토리지, URL 프래그먼트.
- 탐색: 어떤 컨트롤이 URL을 바꾸는지, 뒤로/앞으로 동작.

**중요**: 앱 url을 playwright-cli로 바로 열지 말고, 항상 테스트를 거쳐 거기서 수행되는 커스텀 셋업을 캡처한다.
**중요**: 탐색이 끝나면 백그라운드 테스트를 중지한다.

### 1.4 스펙 파일 작성

`specs/<feature>.plan.md` 아래에 저장한다. 이 구조를 사용한다:

```markdown
# <Feature> Test Plan

## Application Overview

<One paragraph describing what the feature does and why it matters.>

## Test Scenarios

### 1. <Group Name>

**Seed:** `tests/seed.spec.ts`

#### 1.1. <kebab-case-scenario-name>

**File:** `tests/<group>/<kebab-case-scenario-name>.spec.ts`

**Steps:**
  1. <Concrete user step>
    - expect: <observable outcome>
    - expect: <another observable outcome>
  2. <Next step>
    - expect: <outcome>

#### 1.2. <next-scenario>
...

### 2. <Next Group>

**Seed:** `tests/seed.spec.ts`
...
```

가이드라인:

- 각 시나리오는 독립적이며 시드의 새 상태에서 시작한다 — 시나리오를 체이닝하지 않는다.
- 시나리오 이름은 kebab-case이고 테스트 파일명과 일치한다(`should-add-single-todo` → `should-add-single-todo.spec.ts`).
- 핵심 성공 경로, 엣지 케이스, 유효성 검증, 네거티브 흐름, 영속성을 커버한다.
- 단계는 API 수준("`fill` 호출")이 아니라 사용자 수준("입력창에 '우유 사기' 입력")으로 적는다.
- 관찰 가능한 결과는 `- expect:` 불릿에 넣는다. 각 불릿이 생성 중 어설션이 된다.

---

## 2. 생성(Generate)

목표: 스펙 파일을 받아 Playwright 테스트 파일을 만든다. 스펙이 실제와 어긋났으면 선택적으로 갱신한다.

### 2.1 입력

- **스펙 파일**, 예: `specs/basic-operations.plan.md`.
- **대상**: 단일 시나리오(예: `1.2`), 그룹 전체(`1`), 또는 전부.
- **시드 파일**, 시나리오 그룹의 `**Seed:**` 줄에서 읽는다.

### 2.2 시나리오 하나 생성

대상 시나리오마다 순서대로(절대 병렬로 — 시나리오는 시드 세션을 공유한다):

```bash
PLAYWRIGHT_HTML_OPEN=never npx playwright test <seed-file> --debug=cli   # background
playwright-cli attach tw-XXXX
# resume
```

**playwright-cli로 앱 url을 바로 열지 않는다.** 항상 테스트를 거쳐 거기서 수행되는 커스텀 셋업을 캡처한다.

시나리오의 `Steps:`를 `playwright-cli`로 하나씩 수행하되, 스펙을 계획으로, 라이브 앱을 원천으로 삼는다. 단계가 모호하거나("click the button" — 어떤 버튼?), 더 이상 존재하지 않는 요소를 참조하거나, 앱의 실제 동작과 모순되면 재량으로 판단한다: 앱이 실제로 하는 대로 스펙을 갱신하고 계속 진행한다. 생성 중 스펙을 고치는 것은 예상된 일이다.

모든 액션은 동등한 Playwright TypeScript를 출력한다([생성 방식](#0-생성-방식) 참조):

```bash
playwright-cli snapshot                         # find refs
playwright-cli fill e3 "John Doe"               # -> page.getByRole('textbox', {...}).fill(...)
playwright-cli press Enter
playwright-cli click e7
```

각 `- expect:` 불릿마다 명시적 어설션을 추가한다. 자세한 내용은 [생성 방식](#0-생성-방식) 참조.

생성된 코드를 모아 스펙에 명시된 경로에 테스트 파일을 쓴다:

```ts
// spec: specs/basic-operations.plan.md
// seed: tests/seed.spec.ts
import { test, expect } from './fixtures';   // or '@playwright/test' if no fixtures file

test.describe('Signing in and out', () => {
  test('should sign in', async ({ page }) => {
    // 1. Navigate to the application
    // (handled by the seed fixture)

    // 2. Type 'John Doe' into the username field
    await page.getByRole('textbox', { name: 'username' }).fill('John Doe');

    // 3. Type password
    await page.getByRole('textbox', { name: 'password' }).fill('TestPassword');

    // 4. Press Enter to submit
    await page.getByRole('textbox', { name: 'password' }).press('Enter');

    await expect(page.getByRole('heading')).toContainText('Welcome, John Doe!');
  });
});
```

규칙:

- **파일당 테스트 하나.** 파일 경로, describe 이름, 테스트 이름은 스펙에서 그대로 온다(서수는 제외).
- 번호가 매겨진 각 단계 앞에 `// N. <step text>` 주석을 붙인다.
- describe 그룹 이름은 스펙에서 그대로 쓴다(`1.` 서수 없이).
- 프로젝트에 `./fixtures`가 있으면 거기서 import하고, 없으면 `@playwright/test`.
- **중요**: 다음 시나리오로 넘어가기 전에 CLI 세션을 닫고 백그라운드 테스트를 중지한다.

### 2.3 여러 시나리오 생성

2.2를 대상 시나리오마다 하나씩 반복하되, 각 사이에 시드를 재시작해 모든 테스트가 깨끗한 페이지에서 시작하게 한다. 생성되는 세션 이름이 고유하므로 병렬화해도 안전하다 — 각 테스트 실행을 반드시 중지하는 것만 확인한다.

### 2.4 생성된 테스트 실행

생성 후 새 테스트를 한 번 실행한다:

```bash
PLAYWRIGHT_HTML_OPEN=never npx playwright test tests/<group>/<scenario>.spec.ts
```

실패가 있으면 3절로 간다.

---

## 3. 치유(Heal)

목표: 실패한 테스트를 고치고, 앱의 의도된 동작이 바뀌었다면 스펙을 갱신한다.

### 3.1 실패한 테스트 찾기

```bash
PLAYWRIGHT_HTML_OPEN=never npx playwright test
```

실패한 `<file>:<line>` 목록을 기록하고 하나씩 처리한다. 병렬 수정을 시도하지 않는다 — 공유 상태와 단일 CLI 세션 때문에 깨지기 쉽다.

### 3.2 실패 하나 디버그

실패한 테스트 하나를 백그라운드에서 디버그 모드로 실행하고 attach한다:

```bash
PLAYWRIGHT_HTML_OPEN=never npx playwright test tests/<group>/<scenario>.spec.ts:<line> --debug=cli
# wait for "Debugging Instructions" and the tw-XXXX session name
playwright-cli attach tw-XXXX
```

테스트는 시작 시점에 일시정지되어 있다. 실패한 액션이나 어설션 직전까지 스텝을 진행하거나 실행한 뒤 진단한다:

```bash
playwright-cli snapshot                # did the element change / move / rename?
playwright-cli console                 # app-side errors?
playwright-cli requests                # failed request? wrong payload?
playwright-cli show --annotate         # ask the user to point somewhere
```

흔한 원인: 셀렉터 변화, 새 wrapper 요소, 라벨/ARIA 이름 변경, 타이밍(전환, 비동기 로드), 앱에서 갱신된 어설션 텍스트, 실행 간 테스트 데이터 누수.

수정된 상호작용을 `playwright-cli`로 리허설한다 — 출력에 나타난 생성 코드가 테스트에 붙여넣을 그 코드다.

### 3.3 수정 적용

테스트 파일을 편집한다: 로케이터, 어설션, 단계 순서, 입력을 수정된 동작에 맞게 갱신한다. 백그라운드 디버그 실행을 중지한다. 단일 테스트를 다시 실행해 그린을 확인한다.

훅을 건너뛰거나 sleep을 추가해 해결하지 않는다. `networkidle`을 사용하지 않는다.

### 3.4 스펙과 대조

테스트 파일의 `// spec:` 헤더가 참조하는 스펙을 열어 테스트에 해당하는 시나리오를 찾는다.

- **수정이 순수하게 기술적**이었다면(로케이터 변화, 더 나은 어설션 형태) 스펙의 사용자 수준 동작이 여전히 앱과 일치하므로 → 스펙은 그대로 둔다.
- **수정이 스펙이 기술하는 사용자에게 보이는 단계, 입력, 순서, 기대 결과를 바꿨다면** → 스펙을 실제에 맞게 갱신한다. 시나리오 id와 파일 경로는 유지하고, step / expect 줄만 바꾼다.
- **앱 변화가 의도적인 것**(스펙이 낡음) **인지 회귀인지**(테스트가 맞고 앱이 틀림) 불명확하면 → **멈추고 사용자에게 묻는다**. 다음을 제공한다:
  - 시나리오 id(예: `2.3`),
  - 더 이상 일치하지 않는 스펙 줄,
  - 관찰된 앱 동작(스냅샷 발췌나 구체적 결과를 인용).

사용자가 답한 후에야 스펙을 갱신하거나(의도적 변화) 테스트를 버그 커버로 기록/표시한다(회귀).

### 3.5 반복과 포기

- 실패는 한 번에 하나씩 고치고, 각각 다시 실행한다.
- 철저히 조사한 뒤 테스트가 맞고 앱이 틀렸다는 확신이 들고 *사용자가 버그를 확인했으면*: 테스트를 `test.fixme(...)`로 표시하고 사용자의 결정이나 이슈 링크를 가리키는 주석을 단다. 조용히 건너뛰지 않는다.

---

## 교차 참조

| 항목 | 참조 |
|---|---|
| `--debug=cli` / attach 동작 | [playwright-tests.md](playwright-tests.md) |
| 탐색/생성 중 요청 Mock | [request-mocking.md](request-mocking.md) |
| CLI 브라우저 세션 관리 | [session-management.md](session-management.md) |
