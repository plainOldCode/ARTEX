# `/btw` 검증 기록

날짜: 2026-09-10. 브랜치: `codex/btw-side-question`. 기준선: `8dae851b9b622f2ff2631f332fde9719d0b16fba`.

독립 PostgreSQL 테스트 DB와 데이터 디렉터리를 사용했다. 실제 모델 자격증명은 독립 테스트 환경에만 주입했으며 코드나 이 기록에 기록하지 않았고, 제품 기본 모델도 바꾸지 않았다. Go 1.26.3, norma v0.3.6, Next.js 16.2.9.

실제 모델 대화, 반환 객체, 엔지니어링 단정문, Qwen 원본 검토 텍스트는 [validation-2026-09-10.json](../../sidequestion/validation-2026-09-10.json)에 저장되어 있으며 API 자격증명은 없다.

## 엔지니어링 검사

| 범위 | 결과 | 증거 |
| --- | --- | --- |
| 구조화 메시지, 도구 파라미터 딥 카피 | 통과 | `TestCheckpointDeepCopyAndBoundaries` |
| 요약 / 압축 요청이 덮지 않음, 완전한 응답과 종료 상태 발행, 반쪽 응답 제외 | 통과 | `TestCheckpointDeepCopyAndBoundaries`, `TestSnapshotExcludesPartialStreamAndSelectsPoolMember` |
| 실제 모델 풀 멤버십 | 통과 | `TestSnapshotExcludesPartialStreamAndSelectsPoolMember` |
| 도구 쌍, 20조 재생, 예산 절단과 초과 오류 | 통과 | `TestBuildRequestCompactionToolPairingAndBudget` |
| 메인·사이드 병행, 양방향 취소 격리 | 통과 | 블로킹 Provider, `TestMainSideConcurrencyAndIndependentCancellation` |
| 도구 실행 없음, 스트리밍 / 비스트리밍, 실패 시 기존 사용량 | 통과 | `TestServiceNoToolsAndUsageOnFailure` |
| 실제 norma ChatAgent + 로컬 Read 도구, 메인 transcript / 활동 격리 | 통과 | `TestSideActualChatCheckpointToolResultAndTranscriptIsolation`, 스트리밍·비스트리밍 하위 케이스 |
| 영속화, 페이지네이션, 멱등, 재시작 시 부분 답변 보존 | 통과 | `TestSideHistoryIdempotencyPagingAndRecovery` |
| 비우기와 늦은 기록 경합, 부모 리소스 삭제, 버전 비교 | 통과 | `TestSideClearLateWritersAndDeletedParent` |
| MainAgent / Worker 아카이브와 복구, v1/v2/v3 | 통과 | `TestSideTaskArchiveVersions` |
| 세 부모 인터페이스, 인증, 리소스 귀속, Worker 논리 삭제 | 통과 | `TestSideHTTPGlobalLimitTaskWorkerAndDeletion`, `TestSideCheckpointPersistsBeforeAdmissionAndRestart` |
| 사용 중인 메인 세션에서 사이드 가능, 독립 SSE 재접속 / 끊김, 취소, 비우기 | 통과 | `TestSideHTTPBusyIsolationClearAndReconnect` |
| 부모 세션당 1 / 전역 4 동시성 | 통과 | `TestSideHTTP…` 두 케이스 |
| 접수 전 스냅샷 DB 반영, 재시작 후 이어 질문, 구(舊) 세션의 스냅샷 위조 불가 | 통과 | `TestSideCheckpointPersistsBeforeAdmissionAndRestart` |
| 캐시의 설정 삭제 또는 모델 변경 후 계속 거부 | 통과 | `TestSideRejectsDeletedOrChangedCachedProfile` |
| 아카이브 전 취소 및 최종 답변·사용량의 DB 반영 대기 | 통과 | `TestSideTaskDrainPersistsBeforeArchive` |
| 스트리밍 소비자 조기 취소 시 사용량 1회 기록 및 사이드 귀속 | 통과 | `TestSideUsageRecordedOnceOnConsumerCancellation` |
| 재시작 자동 복구된 Worker / deadline 실행 컨텍스트가 새 스냅샷을 계속 발행 | 통과 | `TestSideRestoredWorkerRuntimePublishesNewCheckpoint` |
| 관련 패키지 race 검사 | 통과 | 아래 명령 |
| TypeScript와 프로덕션 빌드 | 통과 | `npx tsc --noEmit`, `npm run build` |
| 신규 프런트엔드 모듈의 Biome | 통과 | `biome check`, 신규 모듈 3개 |

별도의 버릴 수 있는 DB에 `ARTEX_PG_DSN`을 설정하면 자동화 검사를 재현할 수 있다(운영 DB를 가리키지 말 것):

```sh
go test -race ./agent ./db ./server ./sidequestion ./llmrec ./llmpool \
  -run 'Test(Side|Checkpoint|Snapshot|BuildRequest|Service|MainSide|CaptureRun|TaskArchive|CompleteForwards|StopIntent|CancelIntent)' -count=1
cd web
npx tsc --noEmit
npx biome check src/lib/side-questions.ts src/hooks/use-side-questions.ts src/components/side-question-workspace.tsx
npm run build
```

전체 Go 회귀가 전부 그린은 아니다: `server` 패키지에 임시 디렉터리 정리 단계에서 실패하는 기존 테스트 둘이 있으며, 모두 `TempDir RemoveAll … directory not empty`를 보고한다:

- `TestInheritedActivityDetailAndRelationDeletion`
- `TestTaskMetadataPatchReturnsRenameAndPin`

위에서 수정하지 않은 기준선에서 소스를 내보내 같은 격리 환경에서 `server` 패키지를 다시 돌려도 이 두 정리 실패는 재현된다. 기준선 실행에서는 또한 `TestCoreTaskLifecyclePG`의 대상 노드 수 단정 실패가 있었는데, 최종 수정 후의 `server` 회귀에는 이 단정 실패가 없었다. 다른 패키지는 통과했고, 이번 사이드 관련 케이스와 race 검사는 통과했다. 기준선의 문제를 이번 인수 통과로 표시하지 않았고, 문제를 숨기기 위해 기존 단정문을 고치지도 않았다.

Next.js 빌드 출력에는 기존의 다중 lockfile / workspace root 추론 경고가 있다. 빌드는 완료되었고 모든 페이지가 성공적으로 생성되었다.

## 브라우저 검사

Codex In-app Browser를 사용해 독립 로컬 Go 서비스와 Next.js 개발 서버에 연결했다. 데스크톱과 390 × 844 좁은 화면에서 다음의 수동·자동화 동작을 수행하고 스크린샷과 브라우저 로그를 확인했다:

- 일반 대화 실행 중 `/btw` 입력 시 메인 내용과 사이드가 동시에 표시된다. 데스크톱 사이드바가 정상이다.
- 연속 후속 질문. 사이드 중지 후에도 이미 생성된 부분이 유지된다. 메인 흐름은 계속된다.
- 패널을 닫아도 요청은 계속되고, 다시 열면 완료된 답변이 복원된다. 페이지 새로고침 후 빈 `/btw`로 히스토리가 복원된다.
- 좁은 화면 Drawer의 입력, 버튼, 히스토리, 닫기 동작이 정상이고 가로 오버플로가 없다.
- 비우기는 확인 팝업을 사용하며, 비운 뒤 히스토리가 사라지고 메인 transcript와 스냅샷은 유지된다.
- 작업 MainAgent와 두 Worker에서 각각 질문하고 전환했을 때 Agent 탭과 히스토리가 섞이지 않는다.
- 블로킹 로컬 모델 픽스처로 Worker를 실행 상태로 유지한 채 Worker 메인 입력창에서 `/btw`를 제출하면, 사이드를 중지한 뒤에도 Worker는 실시간 실행과 자신의 일시정지 버튼을 그대로 표시하며 사이드는 부분 답변을 저장한다.
- 브라우저 오류 / 경고 로그가 비어 있다.

통제된 픽스처는 정확한 동시성 타이밍 검증을 위한 것이며 실제 모델의 출력 속도에 의존하지 않는다. 디버깅 중 두 Worker 런타임 검사에서 유효한 동시성 창이 만들어지지 않았고(작업이 이미 끝남 / 답변이 조기 종료), 픽스처를 고쳐 다시 수행해 통과했다. 이 초기 시도들을 유효한 통과로 기록하지 않았다.

## 실제 모델 대화

먼저 `grok-4.6`을 탐지했다. OpenAI 호환 인터페이스 `http://127.0.0.1:12580/tingly/openai`. 탐지는 HTTP 200이고, 모델명 `grok-4.6`과 `READY`를 반환했으며 소요 2.82초. 첫 번째 후보가 가용했기 때문에 Tingly `glm`이나 지푸(智谱) `glm-5.3` 예비 체인은 활성화하지 않았다. 이 두 예비 서비스는 이번에 검증하지 않았다.

| 시나리오 | 실제 결과 |
| --- | --- |
| 메인 세션 실행 중 자산, 목표, 표시 질문 | `redhaze.top`, 홈페이지 읽기와 요약 목표, `BTW-REAL-0910`을 반환. 사이드 완료, 16.97초 |
| 메인 세션이 홈페이지 읽기를 끝낸 뒤 도구 근거 질문 | WebFetch 200, curl 리다이렉트 301 → 302 → 200, 페이지 제목을 정확히 인용. 7.24초 |
| 사이드에서 Bash로 테스트 파일 생성 요구 | 실행을 거부했고 대상 파일은 생성되지 않음. 7.74초 |
| 완료 후의 사이드가 메인 컨텍스트를 바꾸지 않음 | 메인 transcript SHA-256이 메인 활동 기록과 일치. 사이드의 도구 실행 횟수 0 |
| 실제로 중지 / 재시작한 Go 서비스에서 후속 질문 | 이전 사이드 히스토리 3건을 유지하고, 영속화된 스냅샷에서 자산·표시·제목을 바로 답하며 메인 Agent를 재실행하지 않음 |
| 새 세션에서 Grok 비스트리밍 설정 사용 | 자산과 `ATOMIC-0910`을 정확히 답변. 사용량 반환 및 저장: input 11734, output 138, cache_read 11520 |

자산 케이스의 메인 세션은 WebFetch와 Bash/curl로 공개 홈페이지를 읽었으며, 랜딩 페이지는 `https://id.redhaze.top/home`, 제목은 “훙무테크 RedHaze Group · 글로벌 종합 그룹 포털”(红幕科技 RedHaze Group · 全球综合集团门户)이었다. Bash는 응답을 로컬 테스트 파일에 임시 저장했고 원격에는 쓰기를 실행하지 않았다. 이 사실은 「사이드가 도구를 실행하지 않았다」는 것과 별도로 검증했다.

메인 transcript 검증값: `e7e61f135a4a120954b539f357e8c4205d7d5cd7460dcaf3dc0fd066463e1d00`.

**사용량 제한:** Tingly의 Grok 스트리밍 응답은 usage를 반환하지 않았다. 별도로 `stream_options.include_usage=true`를 직접 보내 검증했을 때 HTTP 200, 12개 데이터 프레임, usage 프레임 0개였다. 따라서 스트리밍 테스트에서의 0은 엔드포인트가 사용량을 제공하지 않았다는 뜻이지 과금이 없다는 뜻이 아니다. 비스트리밍 사용량과 픽스처의 실패 / 취소 사용량은 정확히 저장되었다.

## Qwen 검토

검토 모델 `qwen-flash`, OpenAI 호환 인터페이스 `https://dashscope.aliyuncs.com/compatible-mode/v1`, HTTP 200. 앞의 세 가지 실제 사이드 대화, 메인 세션의 도구 근거, 엔지니어링 단정문을 제공했다. `verdict: accept`, `concerns: []`를 반환했고, 답변이 자산·표시·페이지 읽기 증거와 일치하며 사이드의 도구 거부가 제약에 부합한다고 판단했다. 검토 사용량: prompt 6625, completion 312, total 6937.

이번 Qwen 검토 범위에는 이후 추가된 서비스 재시작과 비스트리밍 테스트가 포함되지 않았다. Qwen의 「쓰기 없음」 요약은 지나치게 넓었다. 메인 세션의 curl이 실제로 로컬 응답 임시 파일을 만들었다는 점은 위에 명확히 기록되어 있다. 동시성, 도구 실행 0회, transcript 격리는 엔지니어링 단정문으로 판정하며, 모델 검토는 답변 품질 평가의 보조일 뿐이다.
