# 브라우저 세션 관리

상태 영속성을 갖춘 격리된 브라우저 세션 여러 개를 동시에 실행한다.

## 이름 있는 브라우저 세션

브라우저 컨텍스트를 격리하려면 `-s` 플래그를 사용한다:

```bash
# Browser 1: Authentication flow
playwright-cli -s=auth open https://app.example.com/login

# Browser 2: Public browsing (separate cookies, storage)
playwright-cli -s=public open https://example.com

# Commands are isolated by browser session
playwright-cli -s=auth fill e1 "user@example.com"
playwright-cli -s=public snapshot
```

## 브라우저 세션 격리 속성

브라우저 세션마다 다음이 독립적이다:
- Cookies
- LocalStorage / SessionStorage
- IndexedDB
- Cache
- 브라우징 히스토리
- 열린 탭

## 브라우저 세션 명령

```bash
# List all browser sessions
playwright-cli list

# Stop a browser session (close the browser)
playwright-cli close                # stop the default browser
playwright-cli -s=mysession close   # stop a named browser

# Stop all browser sessions
playwright-cli close-all

# Forcefully kill all daemon processes (for stale/zombie processes)
playwright-cli kill-all

# Delete browser session user data (profile directory)
playwright-cli delete-data                # delete default browser data
playwright-cli -s=mysession delete-data   # delete named browser data
```

## 환경 변수

환경 변수로 기본 브라우저 세션 이름을 지정한다:

```bash
export PLAYWRIGHT_CLI_SESSION="mysession"
playwright-cli open example.com  # Uses "mysession" automatically
```

## 일반적인 패턴

### 동시 스크래핑

```bash
#!/bin/bash
# Scrape multiple sites concurrently

# Start all browsers
playwright-cli -s=site1 open https://site1.com &
playwright-cli -s=site2 open https://site2.com &
playwright-cli -s=site3 open https://site3.com &
wait

# Take snapshots from each
playwright-cli -s=site1 snapshot
playwright-cli -s=site2 snapshot
playwright-cli -s=site3 snapshot

# Cleanup
playwright-cli close-all
```

### A/B 테스트 세션

```bash
# Test different user experiences
playwright-cli -s=variant-a open "https://app.com?variant=a"
playwright-cli -s=variant-b open "https://app.com?variant=b"

# Compare
playwright-cli -s=variant-a screenshot
playwright-cli -s=variant-b screenshot
```

### 영속 프로파일

기본적으로 브라우저 프로파일은 메모리에만 유지된다. 브라우저 프로파일을 디스크에 영속하려면 `open`에 `--persistent` 플래그를 사용한다:

```bash
# Use persistent profile (auto-generated location)
playwright-cli open https://example.com --persistent

# Use persistent profile with custom directory
playwright-cli open https://example.com --profile=/path/to/profile
```

## 실행 중인 브라우저에 연결(attach)

새로 띄우는 대신 이미 실행 중인 브라우저에 연결할 때는 `attach`를 사용한다.

### 채널 이름으로 attach

채널 이름으로 실행 중인 Chrome 또는 Edge 인스턴스에 연결한다. 브라우저에는 원격 디버깅이 활성화되어 있어야 한다 — 대상 브라우저에서 `chrome://inspect/#remote-debugging`으로 이동해 "Allow remote debugging for this browser instance"을 체크한다.

```bash
# Attach to Chrome
playwright-cli attach --cdp=chrome

# Attach to Chrome Canary
playwright-cli attach --cdp=chrome-canary

# Attach to Microsoft Edge
playwright-cli attach --cdp=msedge

# Attach to Edge Dev
playwright-cli attach --cdp=msedge-dev
```

지원 채널: `chrome`, `chrome-beta`, `chrome-dev`, `chrome-canary`, `msedge`, `msedge-beta`, `msedge-dev`, `msedge-canary`.

`--session`을 지정하지 않으면 세션 이름은 채널 이름을 따른다(예: `--cdp=msedge`는 `msedge`라는 세션을 만든다). 따라서 Chrome과 Edge에 동시에 attach해도 `default`에서 충돌하지 않는다. 다른 이름을 쓰려면 `--session=<name>`을 전달한다.

### CDP 엔드포인트로 attach

Chrome DevTools Protocol 엔드포인트를 노출하는 브라우저에 연결한다:

```bash
playwright-cli attach --cdp=http://localhost:9222
```

### 브라우저 확장으로 attach

Playwright 확장이 설치된 브라우저에 연결한다:

```bash
playwright-cli attach --extension
```

### detach

외부 브라우저에 영향을 주지 않고 attach된 세션을 정리한다:

```bash
# Detach the default attached session
playwright-cli detach

# Detach a specific attached session
playwright-cli -s=msedge detach
```

`detach`는 `attach`로 만든 세션에만 동작한다. `open`으로 만든 세션은 `close`를 사용한다.

## 기본 브라우저 세션

`-s`를 생략하면 명령은 기본 브라우저 세션을 사용한다:

```bash
# These use the same default browser session
playwright-cli open https://example.com
playwright-cli snapshot
playwright-cli close  # Stops default browser
```

## 브라우저 세션 설정

열 때 특정 설정으로 브라우저 세션을 구성한다:

```bash
# Open with config file
playwright-cli open https://example.com --config=.playwright/my-cli.json

# Open with specific browser
playwright-cli open https://example.com --browser=firefox

# Open in headed mode
playwright-cli open https://example.com --headed

# Open with persistent profile
playwright-cli open https://example.com --persistent
```

## 모범 사례

### 1. 브라우저 세션에 의미 있는 이름 붙이기

```bash
# GOOD: Clear purpose
playwright-cli -s=github-auth open https://github.com
playwright-cli -s=docs-scrape open https://docs.example.com

# AVOID: Generic names
playwright-cli -s=s1 open https://github.com
```

### 2. 항상 정리하기

```bash
# Stop browsers when done
playwright-cli -s=auth close
playwright-cli -s=scrape close

# Or stop all at once
playwright-cli close-all

# If browsers become unresponsive or zombie processes remain
playwright-cli kill-all
```

### 3. 오래된 브라우저 데이터 삭제

```bash
# Remove old browser data to free disk space
playwright-cli -s=oldsession delete-data
```
