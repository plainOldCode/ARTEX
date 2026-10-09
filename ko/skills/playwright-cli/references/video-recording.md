# 비디오 녹화

디버깅, 문서화, 검증을 위해 브라우저 자동화 세션을 비디오로 수집한다. WebM(VP8/VP9 코덱)을 생성한다.

## 기본 녹화

```bash
# Open browser first
playwright-cli open

# Start recording
playwright-cli video-start demo.webm

# Add a chapter marker for section transitions
playwright-cli video-chapter "Getting Started" --description="Opening the homepage" --duration=2000

# Navigate and perform actions
playwright-cli goto https://example.com
playwright-cli snapshot
playwright-cli click e1

# Add another chapter
playwright-cli video-chapter "Filling Form" --description="Entering test data" --duration=2000
playwright-cli fill e2 "test input"

# Stop and save
playwright-cli video-stop
```

## 모범 사례

### 1. 설명이 담긴 파일명 사용

```bash
# Include context in filename
playwright-cli video-start recordings/login-flow-2024-01-15.webm
playwright-cli video-start recordings/checkout-test-run-42.webm
```

### 2. 전체 데모 스크립트 녹화

사용자에게 보여주거나 작업 증거로 삼을 영상을 녹화할 때는 코드 스니펫을 만들어 run-code로 실행하는 것이 가장 좋다.
액션 사이에 적절한 일시정지를 넣고 영상에 어노테이션을 달 수 있다. 이를 위한 새로운 Playwright API가 있다.

1) CLI로 시나리오를 수행하며 모든 로케이터와 액션을 적어둔다. 하이라이트를 위해 바운딩 박스를 요청할 때 이 로케이터들이 필요하다.
2) 영상용 스크립트를 담은 파일을 만든다(아래). 보기 좋은 타이핑을 위해 pressSequentially에 delay를 쓰고, 적절히 일시정지를 넣는다.
3) playwright-cli run-code --filename your-script.js 를 실행한다

**중요**: 오버레이는 `pointer-events: none` — 페이지 상호작용에 간섭하지 않는다. 클릭, 입력 등 페이지에서 어떤 액션을 수행하는 중에도 스티키 오버레이를 안전하게 띄워 둘 수 있다.

```js
async page => {
  await page.screencast.start({ path: 'video.webm', size: { width: 1280, height: 800 } });
  await page.goto('https://demo.playwright.dev/todomvc');

  // Show a chapter card — blurs the page and shows a dialog.
  // Blocks until duration expires, then auto-removes.
  // Use this for simple use cases, but always feel free to hand-craft your own beautiful
  // overlay via await page.screencast.showOverlay().
  await page.screencast.showChapter('Adding Todo Items', {
    description: 'We will add several items to the todo list.',
    duration: 2000,
  });

  // Perform action
  await page.getByRole('textbox', { name: 'What needs to be done?' }).pressSequentially('Walk the dog', { delay: 60 });
  await page.getByRole('textbox', { name: 'What needs to be done?' }).press('Enter');
  await page.waitForTimeout(1000);

  // Show next chapter
  await page.screencast.showChapter('Verifying Results', {
    description: 'Checking the item appeared in the list.',
    duration: 2000,
  });

  // Add a sticky annotation that stays while you perform actions.
  // Overlays are pointer-events: none, so they won't block clicks.
  const annotation = await page.screencast.showOverlay(`
    <div style="position: absolute; top: 8px; right: 8px;
      padding: 6px 12px; background: rgba(0,0,0,0.7);
      border-radius: 8px; font-size: 13px; color: white;">
      ✓ Item added successfully
    </div>
  `);

  // Perform more actions while the annotation is visible
  await page.getByRole('textbox', { name: 'What needs to be done?' }).pressSequentially('Buy groceries', { delay: 60 });
  await page.getByRole('textbox', { name: 'What needs to be done?' }).press('Enter');
  await page.waitForTimeout(1500);

  // Remove the annotation when done
  await annotation.dispose();

  // You can also highlight relevant locators and provide contextual annotations.
  const bounds = await page.getByText('Walk the dog').boundingBox();
  await page.screencast.showOverlay(`
    <div style="position: absolute;
      top: ${bounds.y}px;
      left: ${bounds.x}px;
      width: ${bounds.width}px;
      height: ${bounds.height}px;
      border: 1px solid red;">
    </div>
    <div style="position: absolute;
      top: ${bounds.y + bounds.height + 5}px;
      left: ${bounds.x + bounds.width / 2}px;
      transform: translateX(-50%);
      padding: 6px;
      background: #808080;
      border-radius: 10px;
      font-size: 14px;
      color: white;">Check it out, it is right above this text
    </div>
  `, { duration: 2000 });

  await page.screencast.stop();
}
```

창의성을 발휘하라, 오버레이는 강력하다.

### 오버레이 API 요약

| 메서드 | 용도 |
|--------|----------|
| `page.screencast.showChapter(title, { description?, duration?, styleSheet? })` | 흐려진 배경의 전체 화면 챕터 카드 — 섹션 전환에 적합 |
| `page.screencast.showOverlay(html, { duration? })` | 커스텀 HTML 오버레이 — 콜아웃, 라벨, 하이라이트에 사용 |
| `disposable.dispose()` | duration 없이 추가한 스티키 오버레이 제거 |
| `page.screencast.hideOverlays()` / `page.screencast.showOverlays()` | 모든 오버레이를 임시로 숨김/표시 |

## 트레이싱 vs 비디오

| 기능 | 비디오 | 트레이싱 |
|---------|-------|---------|
| 출력 | WebM 파일 | 트레이스 파일(Trace Viewer에서 볼 수 있음) |
| 보여주는 것 | 시각적 녹화 | DOM 스냅샷, 네트워크, 콘솔, 액션 |
| 용도 | 데모, 문서화 | 디버깅, 분석 |
| 크기 | 큼 | 작음 |

## 제약

- 녹화는 자동화에 약간의 오버헤드를 더한다
- 큰 녹화물은 디스크 공간을 상당히 소비할 수 있다
