# Playwright 테스트 실행

Playwright 테스트를 실행하려면 `npx playwright test` 명령이나 패키지 매니저 스크립트를 사용한다. 인터랙티브 html 리포트가 열리지 않게 하려면 `PLAYWRIGHT_HTML_OPEN=never` 환경 변수를 사용한다.

```bash
# Run all tests
PLAYWRIGHT_HTML_OPEN=never npx playwright test

# Run all tests through a custom npm script
PLAYWRIGHT_HTML_OPEN=never npm run special-test-command
```

# Playwright 테스트 디버깅

실패하는 Playwright 테스트를 디버그하려면 `--debug=cli` 옵션을 붙여 실행한다. 이 명령은 테스트 시작 시점에서 일시정지하고 디버깅 지침을 출력한다.

**중요**: 명령을 백그라운드에서 실행하고 "Debugging Instructions"가 출력될 때까지 출력을 확인한다. 끝내고 나서는 반드시 명령을 중지한다.

세션 이름이 담긴 지침이 출력되면 `playwright-cli`로 그 세션에 attach하여 페이지를 탐색한다.

```bash
# Run the test
PLAYWRIGHT_HTML_OPEN=never npx playwright test --debug=cli
# ...
# ... debugging instructions for "tw-abcdef" session ...
# ...

# Attach to the test
playwright-cli attach tw-abcdef
```

탐색하면서 수정을 찾는 동안에는 테스트를 백그라운드에서 계속 실행해 둔다.
테스트는 시작 시점에 일시정지되어 있으므로, 문제가 있을 가능성이 가장 큰 지점에서
스텝을 진행하거나 일시정지하면 된다.

`playwright-cli`로 수행하는 모든 액션은 대응하는 Playwright TypeScript 코드를 생성한다.
이 코드는 출력에 나타나며 테스트에 바로 복사할 수 있다. 대체로 특정 로케이터나 expectation만 고치면 되지만, 앱 자체의 버그일 수도 있다. 재량으로 판단한다.

테스트를 수정한 뒤에는 백그라운드 테스트 실행을 중지한다. 다시 실행해 테스트가 통과하는지 확인한다.
