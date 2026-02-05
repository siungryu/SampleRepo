# Flutter 포팅 TODO (실행 백로그)

## 0) 결정 사항 확정
- [x] 타깃 플랫폼 확정 (iOS/Android/macOS/Windows) — 완료 기준: plan.md에 플랫폼 범위 명시
- [x] 검색 대소문자 정책 확정 (case‑insensitive) — 완료 기준: 결정 기록 + 쿼리 반영
- [x] DBHandling 범위 확정 (MVP: 백업/복원/공유만) — 완료 기준: 범위 문서화

## 1) 프로젝트 셋업
- [x] Flutter 프로젝트 생성 및 폴더 구조 반영 — 완료 기준: plan.md 구조와 일치
- [x] MVP 의존성 추가 (Post‑MVP 제외) — 완료 기준: pubspec에 핵심 패키지만
- [x] assets 등록 (images + default.realm) — 완료 기준: 런타임 로딩 확인

## 2) 데이터 레이어
- [x] Todo 모델 정의 (dueDateString + dueDateAt 듀얼 필드) — 완료 기준: 스키마 빌드 성공
- [x] primary_key 동기화 전략 구현 — 완료 기준: 최대 todoID와 일치
- [x] 트리 연산 구현 (create/delete/move/copyTree) — 완료 기준: 단위 테스트 통과
- [x] 정합성 규칙 적용 (motherID ↔ childIDs, orderIndex 동기화) — 완료 기준: 검증 테스트 통과
- [x] 폴더 변경 재귀 처리 — 완료 기준: 하위 모두 folderType 변경 확인

## 3) 마이그레이션
- [x] iOS Realm 파일 읽기 — 완료 기준: Flutter에서 iOS DB 열기 성공
- [x] dueDateString → dueDateAt 변환 — 완료 기준: 데이터 손실 없이 변환
- [x] 최초 실행 시 시드 DB 복사 — 완료 기준: default.realm 복사 확인

## 4) 상태 관리
- [x] Settings 상태 구현 (isDebug, isQuickAxing, isDisplayTodoDetailTextView, isDueDateDefault, 테마 색상) — 완료 기준: 저장/로드 확인
- [x] Navigation 상태 구현 (selectedFolder, selectedMotherId, current todo) — 완료 기준: UI 반영 확인
- [x] NotificationCenter 대체 흐름 구현 — 완료 기준: 변경 시 전역 리프레시 정상

## 5) 핵심 UI
- [x] 메인 화면: 리스트 + 상세 패널 — 완료 기준: 현재 부모의 자식만 표시
- [x] TodoCell 위젯 parity — 완료 기준: 체크/복제/배지/아이콘 동작 동일
- [x] Quick Axing 자동 이동 — 완료 기준: 복제 후 새 노드로 이동
- [x] 클론 편집 팝업 — 완료 기준: 제목/상세 수정 후 복제 가능
- [x] 사이드 메뉴 탭 — 완료 기준: Favorite/DueDate/Template/Trash 정상 목록

## 6) 기능 구현
- [ ] 트리맵(Map) 뷰 — 완료 기준: MapType별 확장 정책 반영
- [x] 검색 화면 — 완료 기준: 결정된 대소문자 정책 동일
- [ ] 새 Todo 플로우 — 완료 기준: 캘린더 생성 시 dueDate 자동 세팅
- [x] 이동/복사 (단일 + 다중) — 완료 기준: 자손 이동 방지 + copyTree 검증
- [x] 정렬 (ReorderableListView) — 완료 기준: childIDs + orderIndex 동기화
- [x] DueDate 설정 화면 — 완료 기준: 오늘/내일 단축 규칙 동일

## 7) DBHandling (범위 확정 후)
- [x] 백업 생성 — 완료 기준: Documents/backup 파일 생성
- [x] 복원 — 완료 기준: DB 교체 + primary_key 동기화
- [x] 공유 — 완료 기준: 공유 시트 열림

## 8) 테스트
- [ ] 단위 테스트: 트리 정합성, copyTree, delete subtree, move 제약 — 완료 기준: 전부 통과
- [ ] 위젯 테스트: TodoCell, TreeView 확장/축소 — 완료 기준: 안정 실행
- [ ] 마이그레이션 테스트: iOS Realm 열기 + dueDate 변환 — 완료 기준: 데이터 손실 없음

## 9) Post‑MVP
- [ ] 알람/알림 기능 — 완료 기준: 스케줄링 + UI 토글 완료
