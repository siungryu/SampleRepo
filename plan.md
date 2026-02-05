# Lighthouse Flutter 포팅 계획서

> **Codex 리뷰 반영**: 2026-02-04 (consult.md 피드백 적용)

## 개요

**목표**: iOS UIKit + Realm 기반 계층형 할 일 관리 앱을 Flutter로 포팅
**타겟 플랫폼**: iOS, Android, macOS, Windows

### 기술 스택 선택
| 항목 | 선택 |
|------|------|
| UI 디자인 | Material Design 기반 |
| 데이터베이스 | Realm Flutter |
| 상태 관리 | Provider |
| 데이터 마이그레이션 | 필요 (듀얼 필드 방식) |

### 결정 사항 (현재 기준)
| 항목 | 결정 |
|------|------|
| 알람 기능 | **MVP 이후로 연기** |
| 검색 대소문자 | **대소문자 무시** (case-insensitive) |
| DB 관리 범위 | **MVP: 백업/복원/공유만**, 파일 관리 UI는 Post‑MVP |

---

## Phase 0: 프로젝트 설정 (1주차)

### 폴더 구조
```
lighthouse_flutter/
├── lib/
│   ├── main.dart
│   ├── app.dart
│   ├── core/
│   │   ├── constants/       # 색상, 문자열, 치수
│   │   ├── enums/           # FolderType, MapType, TableType, Priority
│   │   ├── extensions/      # Color, String, Date 확장
│   │   ├── utils/           # DateUtils, TreeUtils
│   │   └── themes/          # Material Theme
│   ├── data/
│   │   ├── models/          # Todo, TreeNodeItem
│   │   ├── repositories/    # TodoRepository, SettingsRepository
│   │   └── services/        # RealmService, BackupService (NotificationService는 Post-MVP)
│   ├── providers/           # TodoProvider, NavigationProvider, etc.
│   ├── screens/             # 화면별 폴더
│   └── widgets/             # 재사용 위젯
├── assets/
│   ├── images/
│   └── default.realm
└── test/
```

### 핵심 의존성 (pubspec.yaml)
```yaml
dependencies:
  realm: ^10.8.0                    # DB (iOS와 호환)
  provider: ^6.1.0                  # 상태 관리
  table_calendar: ^3.0.9            # 캘린더
  flutter_fancy_tree_view: ^1.4.0   # 트리 뷰
  intl: ^0.18.0                     # 날짜 포맷
  shared_preferences: ^2.2.0        # 설정 저장
  path_provider: ^2.1.0             # 파일 경로
  share_plus: ^7.2.0                # 백업 공유
```

---

## Phase 1: 데이터 레이어 (1-2주차)

### 1.1 Enum 정의
- `FolderType`: current(0), trash(1), template(2), storage(3)
- `MapType`: goto, showTree, template, templateNodes, moveTo
- `TableType`: favorite, dueDate, template, trash

### 1.2 Todo 모델 (Realm) - 듀얼 필드 마이그레이션 방식

> **Codex 피드백 반영**: String → DateTime 직접 변환은 위험. 듀얼 필드 사용.

```dart
@RealmModel()
class _Todo {
  @PrimaryKey()
  late int todoID;
  late String title, detail;
  late int priority;      // 2-7 (5=Normal, 2=가장 중요)
  late int urgency;       // 0-4
  late int motherID;      // 부모 ID
  late int folderType;    // 폴더 타입

  // 듀얼 필드 마이그레이션 방식
  late String dueDateString;   // iOS 호환용 (기존 "yyyy.MM.dd HH:mm")
  late DateTime? dueDateAt;    // Flutter 신규 필드 (권장)

  late List<int> childIDs;
  // ... 기타 필드
}
```

**마이그레이션 전략**:
1. iOS 데이터 읽을 때: `dueDateString` 파싱 → `dueDateAt` 채움
2. Flutter에서 저장 시: 둘 다 업데이트 (역호환)
3. 완전 마이그레이션 후: `dueDateString` deprecated

### 1.3 TodoRepository - 핵심 데이터 정합성 규칙

| 작업 | 정합성 규칙 |
|------|------------|
| 생성 | 부모의 childIDs에 새 ID 추가 |
| 삭제 | 부모 childIDs에서 제거 → 서브트리 재귀 삭제 |
| 이동 | 자손으로 이동 방지 체크 → 기존 부모에서 제거 → 새 부모에 추가 |
| 폴더 변경 | 자식 포함 재귀적으로 folderType 갱신 |
| 복제 | 서브트리 깊은 복사 + 새 todoID 할당 |

### 참조 파일
- `Todo.swift` - 코어 데이터 모델
- `DataManager.swift` - 트리 작업 로직

---

## Phase 2: 상태 관리 (2-3주차)

### Provider 구조
| Provider | 역할 |
|----------|------|
| TodoProvider | 현재 자식 목록, 선택된 할 일, 네비게이션 |
| NavigationProvider | TableType, MapType, 사이드메뉴 상태 |
| TreeViewProvider | 트리 노드 빌드, 확장/축소 |
| SettingsProvider | 설정값 관리 |

### 주요 이벤트 흐름 (NotificationCenter 대체)
- `TreeNodeChanged` → Provider.notifyListeners()
- 즐겨찾기/완료/우선순위 변경 → 즉시 DB 반영 + notifyListeners()

---

## Phase 3: 핵심 UI 구현 (3-5주차)

### 화면 구현 우선순위

| 순위 | 화면 | 복잡도 | 설명 |
|------|------|--------|------|
| 1 | Main Screen | 높음 | 리스트 + 상세 패널 |
| 2 | New Todo | 중간 | 할 일 생성 다이얼로그 |
| 3 | Side Menu | 중간 | Drawer 네비게이션 |
| 4 | Map View (Tree) | 높음 | 계층 트리 표시 |
| 5 | Calendar | 중간 | table_calendar 연동 |
| 6 | Search | 낮음 | 전문 검색 |
| 7 | Move/Copy | 높음 | 제약조건 포함 이동 |
| 8 | Settings | 중간 | 설정 화면 |
| 9 | Due Date | 낮음 | 날짜 선택기 |
| 10 | Ordering | 중간 | 드래그 정렬 |
| 11 | DB Handling | 중간 | 백업/복원 |

### TodoCell 위젯 구성
- 체크 버튼 (완료 토글)
- 제목 (완료 시 취소선)
- Ax 버튼 (복제 - isQuickAxing에 따라 이미지 변경)
- 우선순위 색상 띠 (2-7)
- 긴급도 색상 띠 (0-4)
- 즐겨찾기 별 아이콘
- 마감일 시계 아이콘
- 자식 개수 배지

### Quick Axing 동작 (Codex 피드백 반영)

> **누락 동작 추가**: 복제 후 새 노드로 자동 이동

| isQuickAxing | 동작 |
|--------------|------|
| false | 아이콘만 변경, 복제 후 현재 위치 유지 |
| **true** | 복제 후 **새로 생성된 노드로 자동 이동** (navigateInto) |

### Clone Flow (Codex 피드백 반영)

> **누락 기능 추가**: MapView에서 복제 시 편집 팝업

**TodoEditViewController 동등 구현 필요**:
1. MapView에서 복제 액션 선택 시
2. **제목/상세 편집 팝업** (BottomSheet 또는 Dialog) 표시
3. 편집 완료 후 copyTree 실행

### 참조 파일
- `ViewController.swift` (65KB) - 메인 화면 로직
- `TodoTableViewCell.swift` - 셀 구성

---

## Phase 4: 기능 구현 (5-7주차)

### 4.1 사이드 메뉴
- 즐겨찾기 (TableType.favorite)
- 마감일 순 (TableType.dueDate)
- 템플릿 (TableType.template)
- 휴지통 (TableType.trash)
- 캘린더
- 검색
- 설정

### 4.2 캘린더 뷰
- `table_calendar` 패키지 사용
- 마감일 있는 날짜에 점 표시
- 날짜 선택 시 필터링 (00:00 ~ 다음날 00:00)
- 선택 날짜에서 새 할 일 생성

### 4.3 검색 (대소문자 무시)

> **결정**: iOS는 case-sensitive였으나, UX 향상을 위해 **case-insensitive** 채택

```dart
// [c] = case-insensitive
realm.query<Todo>('title CONTAINS[c] $0 OR detail CONTAINS[c] $0', [searchText])
```

### 4.4 이동/복사
- 다중 선택 체크박스
- **자손으로 이동 방지** (필수)
- MapView로 목적지 선택
- copyTree: 깊은 복사 + 새 ID

### 4.5 순서 정렬
- `ReorderableListView` 사용
- orderIndex 업데이트
- childIDs 순서 동기화

### 참조 파일
- `MoveViewController.swift` - 이동/복사 로직
- `PBTreeViewDataHandler.swift` - 트리 데이터 변환

---

## Phase 5: DB 관리 - 전체 포팅 (7-8주차)

> **Codex 피드백 반영**: inbox, rename, delete 등 전체 파일 작업 포함

### DBHandlingService 구현 (전체 기능)

| 기능 | 설명 |
|------|------|
| 백업 생성 | `Documents/backup/backup_yyyyMMdd_HHmmss.realm` |
| 백업 목록 | 백업 파일 리스트 표시 |
| 백업 복원 | Realm 닫기 → 파일 교체 → primary_key 동기화 |
| 백업 삭제 | 선택한 백업 파일 삭제 |
| **백업 이름 변경** | 파일명 수정 기능 |
| **Inbox 관리** | 외부에서 가져온 .realm 파일 처리 |
| 백업 공유 | `share_plus` 패키지 (AirDrop 대체) |

### Inbox 처리 흐름
1. 외부 앱에서 .realm 파일 공유 받음
2. 앱 Inbox 디렉토리에 저장
3. Inbox 목록에서 선택하여 복원 또는 삭제

### 참조 파일
- `DBHandlingViewController.swift` - 전체 백업 로직

---

## Phase 6: 데이터 마이그레이션 (8-9주차)

### iOS → Flutter 마이그레이션 (듀얼 필드 방식)

> **Codex 피드백 반영**: in-place 타입 변환 대신 **듀얼 필드** 사용

**마이그레이션 단계**:
1. Schema version: 5 → 6 업그레이드
2. 새 필드 `dueDateAt` (DateTime) 추가
3. 기존 `dueDateString` (String) 유지 (역호환)
4. 첫 실행 시 `dueDateString` 파싱 → `dueDateAt` 채움
5. primary_key 동기화 확인

```dart
final config = Configuration.local(
  [Todo.schema],
  schemaVersion: 6,
  migrationCallback: (migration, oldSchemaVersion) {
    if (oldSchemaVersion < 6) {
      // 새 dueDateAt 필드 추가 (null 허용)
      // dueDateString은 그대로 유지
    }
  },
);

// 첫 실행 시 마이그레이션
void migratedueDateStringToDateTime(Realm realm) {
  final todos = realm.all<Todo>().where((t) =>
    t.dueDateString.isNotEmpty && t.dueDateAt == null);

  realm.write(() {
    for (final todo in todos) {
      todo.dueDateAt = _parseDateString(todo.dueDateString);
    }
  });
}
```

---

## Phase 7: 플랫폼별 최적화 (9-10주차)

### 플랫폼 매트릭스 (상세) - Codex 피드백 반영

> **주의**: 일부 패키지는 데스크톱 지원 제한

| 기능 | iOS | Android | macOS | Windows | 비고 |
|------|-----|---------|-------|---------|------|
| 핵심 기능 | O | O | O | O | |
| Realm DB | O | O | O | O | 전 플랫폼 지원 |
| 백업/복원 | O | O | O | O | |
| 파일 공유 | O | O | O | O | share_plus |
| 키보드 단축키 | - | - | O | O | |
| 로컬 알림 | O | O | △ | △ | 데스크톱 제한 |

**Fallback 전략**:
- macOS/Windows 알림: 인앱 알림으로 대체 (Post-MVP)
- 파일 공유: 데스크톱에서는 "파일 위치 열기" 대체 가능

### 데스크톱 개선
- 키보드 단축키 (Cmd+N: 새 할 일 등)
- 우클릭 컨텍스트 메뉴
- 리사이즈 가능한 상세 패널

### MapType 확장 규칙 (Codex 피드백 반영)

> **누락 UX 추가**: MapType에 따른 노드 초기 확장 동작

| MapType | 초기 확장 동작 |
|---------|---------------|
| goto | 전체 확장 (탐색 용이) |
| showTree | 전체 확장 (계층 표시) |
| template | 1레벨만 확장 |
| templateNodes | 1레벨만 확장 |
| moveTo | 전체 확장 (목적지 선택) |

---

## Phase 8: 테스트 (전체 기간)

### 단위 테스트
- TodoRepository CRUD
- 트리 작업 정합성
- 완료율 계산
- 날짜 유틸리티

### 위젯 테스트
- TodoCell 렌더링
- 트리 뷰 확장/축소
- 사이드 메뉴 네비게이션

### 통합 테스트
- 전체 네비게이션 플로우
- 이동/복사 제약조건 검증
- 백업/복원 사이클

### 필수 테스트 케이스 (Codex 피드백 반영 추가)
1. **삭제 연쇄**: 부모 삭제 시 서브트리 삭제 확인
2. **이동 제약**: 부모를 자손으로 이동 방지
3. **폴더 변경 연쇄**: 모든 자식 folderType 업데이트
4. **childIDs 순서**: 표시 순서 = orderIndex
5. **primary_key 동기화**: 백업 복원 후 확인
6. **copyTree 깊은 복제**: 서브트리 무결성 + 새 ID 검증 *(추가)*
7. **Quick Axing 자동 이동**: 복제 후 새 노드로 이동 확인 *(추가)*
8. **MapType 초기 확장**: 각 MapType별 확장 규칙 검증 *(추가)*
9. **검색 case-insensitive**: 대소문자 무시 검색 확인 *(추가)*
10. **dueDate 마이그레이션**: iOS String → Flutter DateTime 변환 *(추가)*

---

## 일정 요약 (Codex 리뷰 후 수정)

| 주차 | Phase | 산출물 |
|------|-------|--------|
| 1 | 설정 & 아키텍처 | 프로젝트 구조, 의존성, Enum |
| 1-2 | 데이터 레이어 | Todo 모델 (듀얼 필드), Repository, Services |
| 2-3 | 상태 관리 | Providers |
| 3-5 | 핵심 UI | 메인 화면, 셀, 상세 패널, 새 할 일, Clone 편집 팝업 |
| 5-7 | 기능 | 사이드 메뉴, 캘린더, 검색, 이동/복사, 정렬 |
| 7-8 | DB 관리 | 전체 백업 기능 (inbox, rename, delete 포함) |
| 8-9 | 마이그레이션 | iOS 데이터 마이그레이션 (듀얼 필드 방식) |
| 9-10 | 플랫폼 최적화 | 데스크톱 개선, 테스트 |

---

## Post-MVP: 알람 기능

> **결정**: MVP 완료 후 추가 구현

필요 의존성: `flutter_local_notifications`

### NotificationService 구현 (Post-MVP)
```dart
// flutter_local_notifications 사용
Future<void> scheduleDueDateReminder(Todo todo) async {
  if (!todo.isUseAlarm || todo.dueDateAt == null) return;

  final scheduledTime = todo.dueDateAt!.subtract(
    Duration(minutes: todo.alarmBeforeMinute),
  );

  await _notifications.zonedSchedule(
    todo.todoID,
    'Due Date Reminder',
    'Task "${todo.title}" is due soon!',
    // ...
  );
}
```

### UI 추가 (Post-MVP)
- Due Date 설정 화면에 알람 토글 추가
- 알림 시간 선택 (5, 10, 15, 30, 60분, 1일 전 등)
- **데스크톱 Fallback**: 인앱 알림 또는 시스템 알림 대체

---

## 검증 방법

1. **데이터 정합성**: iOS 앱에서 생성한 DB를 Flutter 앱에서 읽고 수정 후 다시 iOS에서 확인
2. **기능 동등성**: iOS 앱과 동일한 시나리오 테스트 체크리스트 작성
3. **크로스 플랫폼**: 4개 플랫폼에서 동일 기능 동작 확인
4. **백업 호환**: iOS 백업 파일을 Flutter에서 복원 테스트

---

## 핵심 정합성 규칙 (ANALYSIS_FOR_FLUTTER_MIGRATION.md 섹션 16 기반)

### 데이터 정합성
- `motherID`와 `childIDs`는 반드시 상호 일치
- `childIDs` 순서 = 정렬 순서 = `orderIndex` 동기화
- 삭제: 부모 childIDs에서 제거 → 서브트리 재귀 삭제
- 이동: 기존 부모 childIDs 제거 → 새 부모 추가 → motherID 변경
- 폴더 이동: 자식 포함 재귀적 folderType 갱신
- 복제(copyTree): 서브트리 깊은 복사 + 새 todoID 할당

### UI/행동 일치
- 메인 화면: **선택된 부모의 직계 자식만** 표시
- `isDisplayTodoDetailTextView` false → 상세 입력 숨기고 제목 최소 2줄
- 완료율: 자식 없음 → 999, 0 또는 999일 때 "Completed" 표기
- 즐겨찾기/완료/우선순위 변경 → 즉시 DB 반영 + 전역 갱신
- `isQuickAxing` → 복제 버튼 이미지 변경 **+ 복제 후 새 노드로 자동 이동** *(Codex 피드백)*
- 검색 결과 선택 → 해당 폴더 전환 + 부모로 이동
- MapView 복제 시 → **제목/상세 편집 팝업 표시** *(Codex 피드백)*

### 날짜/시간 처리
- dueDate: `"yyyy.MM.dd HH:mm"` → Flutter에서는 DateTime 권장
- 캘린더 일별 필터: 해당 날짜 00:00 ~ 다음날 00:00
- Today/Yesterday/Tomorrow 상대 표시 규칙 유지

### 초기 데이터/백업
- 첫 실행: `resource/default.realm` → Document 복사
- `primary_key`: 현재 최대 todoID와 항상 동기화
- 백업/복원: Document/backup 폴더 파일 단위

---

*작성일: 2026-02-04*
*Codex 리뷰 반영: 2026-02-04*
*참조: ANALYSIS_FOR_FLUTTER_MIGRATION.md, app_analysis2.md, consult.md*
