# Lighthouse iOS 앱 분석 보고서
## Flutter 마이그레이션을 위한 종합 분석

> **Note**: 이 문서는 Codex의 보완 분석(`app_analysis2.md`)을 반영하여 수정되었습니다.

---

## 1. 개요 (Executive Summary)

**Lighthouse**는 UIKit 기반의 계층형 할 일 관리 앱입니다. 트리 구조로 할 일을 관리하며, 캘린더 뷰, 검색 기능, 고급 정리 기능을 지원합니다.

| 항목 | 내용 |
|------|------|
| 언어 | Swift |
| UI 프레임워크 | UIKit |
| 데이터베이스 | Realm |
| 최소 iOS 버전 | iOS 15.0 |
| 총 코드 라인 | 약 **9,424줄** (Pods 제외) |

---

## 2. 프로젝트 구조

```
/Lighthouse/
├── 루트 레벨 Swift 파일
│   ├── AppDelegate.swift
│   ├── ViewController.swift (메인 리스트 뷰 - 65KB)
│   ├── CalendarViewController.swift
│   ├── MapViewController.swift (트리 네비게이션)
│   ├── MoveViewController.swift
│   ├── CellMoveViewController.swift (단일 셀 이동)
│   ├── SearchViewController.swift
│   ├── SettingViewController.swift
│   ├── AppInfoViewController.swift
│   ├── AppInfoTableViewCell.swift (앱 정보 셀)
│   ├── DBHandlingViewController.swift
│   ├── DataFixManagerViewController.swift
│   ├── TodoEditViewController.swift
│   ├── DueDateSetViewController.swift
│   ├── WebViewController.swift
│   ├── LicenseViewController.swift
│   ├── DateCell.swift (캘린더 날짜 셀)
│   ├── Todo.swift (핵심 데이터 모델)
│   └── DataManager.swift (데이터 관리자)
│
├── Lighthouse/ (중첩 프로젝트 폴더)
│   ├── SceneDelegate.swift
│   ├── Info.plist
│   ├── classes/
│   │   ├── Common/
│   │   │   └── CommonUtil.swift
│   │   ├── Extensions/
│   │   │   ├── String+Substring.swift
│   │   │   └── UIColor+hexString.swift
│   │   ├── PBTreeView/
│   │   │   ├── PBTreeViewData.swift
│   │   │   ├── PBTreeViewDataHandler.swift
│   │   │   ├── PBTreeViewNodeItem.swift
│   │   │   └── PBTreeViewTableCell.swift
│   │   ├── SideMenu/
│   │   │   └── SideMenuViewController.swift
│   │   ├── NewToDoViewController.swift
│   │   ├── OrderingViewController.swift
│   │   ├── OrderingTableViewCell.swift
│   │   ├── DeleteViewController.swift
│   │   ├── TodoTableViewCell.swift
│   │   ├── IBNextGrowingTextView.swift
│   │   └── Lighthouse.pch
│   ├── Base.lproj/
│   │   └── LaunchScreen.storyboard
│   ├── images/ (56개 이미지)
│   └── Assets.xcassets/ (123개 이미지 세트)
│
├── Base.lproj/
│   └── Main.storyboard
├── TodoEditViewController.xib
├── Todo.swift (핵심 데이터 모델)
├── DataManager.swift (데이터 관리자)
├── Podfile
└── Podfile.lock
```

---

## 3. 데이터 모델

### 3.1 Todo 모델 (핵심 엔티티)

```swift
Todo (Realm Object)
├── todoID: Int (Primary Key)
├── title: String
├── detail: String
├── priority: Int (2-7 스케일, 5=기본/Normal, 2=가장 중요)
├── urgency: Int (0-4 스케일, UI에서 일부 hidden)
├── progress: Float (0.0-1.0, 현재 UI 사용 흔적 적음)
├── isCompleted: Bool
├── isFavorite: Bool
├── isUseDueDate: Bool
├── isUseAlarm: Bool
├── alarmBeforeMinute: Int
├── favoriteOrderIndex: Int
├── orderIndex: Int
├── motherID: Int (부모 ID - 계층 구조)
├── folderType: Int (0=현재, 1=휴지통, 2=템플릿, 3=보관함)
├── createDate: Date
├── updateDate: Date
├── dueDate: String ("yyyy.MM.dd HH:mm")
└── childIDs: List<Int> (자식 할 일 ID 목록)
```

### 3.2 FolderType 열거형

| 값 | 이름 | 설명 |
|----|------|------|
| 0 | folderTypeCurrent | 현재 작업 중인 할 일 |
| 1 | folderTypeTrash | 휴지통 |
| 2 | folderTypeTemplate | 템플릿 |
| 3 | folderTypeStorage | 보관함 |

### 3.3 MapType 열거형 (트리 뷰 모드)

| 값 | 이름 | 설명 |
|----|------|------|
| 0 | mapTypeGoto | 특정 할 일로 이동 |
| 1 | mapTypeShowTree | 트리 구조 표시 |
| 2 | mapTypeTemplate | 템플릿 선택 |
| 3 | mapTypeTemplateNodes | 복수 템플릿 선택 |
| 4 | mapTypeMoveTo | 이동 대상 선택 |

### 3.4 TableType 열거형 (리스트 표시 모드 - SideMenu)

> **수정**: `TableTypeNormal`은 존재하지 않음

| 이름 | 설명 |
|------|------|
| TableTypeFavorate | 즐겨찾기 목록 |
| TableTypeDueDate | 마감일 기준 목록 |
| TableTypeTemplate | 템플릿 목록 |
| TableTypeTrash | 휴지통 목록 |

---

## 4. 화면별 상세 분석

### 4.1 ViewController.swift (메인 화면)

**파일 크기:** 65,652 bytes
**주요 역할:** 앱의 핵심 화면으로 할 일 목록 표시 및 관리

#### UI 구성요소
| 컴포넌트 | 타입 | 설명 |
|----------|------|------|
| todoTableView | UITableView | 메인 할 일 목록 |
| todoTitleTextView | NextGrowingTextView | 제목 편집 (자동 확장) |
| todoDetailTextView | NextGrowingTextView | 상세 내용 편집 |
| priorityView | UIView | 우선순위 색상 표시 |
| dueDateLabel | UILabel | 마감일 표시 |
| completedLabel | UILabel | 자식 완료율 표시 |
| sideMenuCallButton | UIButton | 사이드 메뉴 호출 |

> **Note**: NextGrowingTextView는 Storyboard/XIB에서 직접 사용하지 않고, 컨테이너 UIView에 **코드로 동적 생성**합니다. `IBNextGrowingTextView.swift`는 남아있으나 현재 참조되지 않습니다.

#### 구현된 프로토콜
- UITableViewDataSource / UITableViewDelegate
- UITextViewDelegate
- ToDoCellDelegate
- RyuTodoViewControllerDelegate
- MapViewControllerDelegate
- SideMenuViewControllerDelegate
- DueDateSetViewControllerDelegate

#### 주요 기능
1. 할 일 목록 표시 (테이블 뷰 / 맵 뷰 전환)
2. 선택된 할 일 상세 정보 패널
3. 인라인 편집 (제목, 상세 내용)
4. 계층 탐색 (부모/자식 이동)
5. 완료 상태 토글
6. 즐겨찾기 토글
7. 마감일 설정

---

### 4.2 CalendarViewController.swift

**파일 크기:** 17,833 bytes
**주요 역할:** 월별 캘린더 뷰와 날짜별 할 일 필터링

#### UI 구성요소
| 컴포넌트 | 타입 | 설명 |
|----------|------|------|
| calendarView | JTAppleCalendarView | 월간 캘린더 |
| todoTableView | UITableView | 선택 날짜 할 일 목록 |
| monthLabel | UILabel | 현재 월 표시 |
| prevButton | UIButton | 이전 월 이동 |
| nextButton | UIButton | 다음 월 이동 |

#### 주요 기능
1. 월별 캘린더 표시
2. 할 일 있는 날짜에 점 표시
3. 날짜 선택 시 해당 날짜 할 일 필터링
4. 선택 날짜에 새 할 일 생성
5. 오늘 날짜로 빠른 이동

---

### 4.3 MapViewController.swift (트리 뷰)

**파일 크기:** 44,254 bytes
**주요 역할:** 계층형 트리 구조 표시 및 탐색

#### UI 구성요소
| 컴포넌트 | 타입 | 설명 |
|----------|------|------|
| treeTableView | UITableView | 트리 노드 목록 |
| contentWidthConstraint | NSLayoutConstraint | 깊이에 따른 너비 조절 |

#### 데이터 구조
```swift
treeViewDataSource: [PBTreeViewNodeItem]
dataHandler: PBTreeViewDataHandler
maxLevel: Int (최대 깊이 추적)
```

#### 주요 기능
1. 계층형 트리 표시 (들여쓰기 기반)
2. 노드 확장/축소
3. 노드 선택 및 이동
4. 즐겨찾기 관리
5. 컨텍스트 메뉴 (더보기 옵션)

---

### 4.4 SearchViewController.swift

**파일 크기:** 9,032 bytes
**주요 역할:** 전체 텍스트 검색

#### 주요 기능
1. 제목/상세 내용 실시간 검색
2. Realm 쿼리 기반 필터링
3. 검색 결과에서 바로 해당 폴더로 이동

#### 검색 쿼리
```swift
realm.objects(Todo.self).filter("title CONTAINS '\(searchText)' OR detail CONTAINS '\(searchText)'")
```

---

### 4.5 SettingViewController.swift

**파일 크기:** 10,739 bytes
**주요 역할:** 앱 설정 관리

#### 설정 항목
| 설정 | 타입 | 설명 |
|------|------|------|
| Quick Axing | Bool | 편집 시 복사본 생성 |
| Debug Mode | Bool | 할 일 ID 및 메타데이터 표시 |
| Display Detail | Bool | 상세 텍스트 필드 표시 여부 |
| Theme Color | String (Hex) | 셀 배경색 |
| Select Color | String (Hex) | 선택 셀 배경색 |

---

### 4.6 NewToDoViewController.swift

**파일 크기:** 17,742 bytes
**주요 역할:** 새 할 일 생성

#### 주요 기능
1. 제목/상세 내용 입력
2. 우선순위/긴급도 선택
3. 즐겨찾기 설정
4. 마감일 지정
5. 부모 할 일 지정
6. Save / Save+ (저장 후 계속 생성)

---

### 4.7 DueDateSetViewController.swift

**파일 크기:** 5,967 bytes
**주요 역할:** 마감일 설정

#### 주요 기능
1. UIDatePicker로 날짜/시간 선택
2. 오늘/내일 빠른 선택 버튼
3. 마감일 사용 여부 토글
4. 저장하지 않은 변경사항 경고

---

### 4.8 MoveViewController.swift (다중) + CellMoveViewController.swift (단일)

**파일 크기:** 19,991 bytes (Move), 별도 (CellMove)
**주요 역할:** 할 일 이동/복사

#### 주요 기능
1. 다중 선택 (체크박스) - MoveViewController
2. 단일 셀 이동 - CellMoveViewController
3. 폴더 대상 선택
4. MapView로 목적지 선택
5. **자손으로 이동 방지** (중요한 제약조건)
6. `copyTree()` - 서브트리 복제 기능

---

### 4.9 DeleteViewController.swift

**파일 크기:** 8,861 bytes
**주요 역할:** 할 일 일괄 삭제

#### 주요 기능
1. 확인 다이얼로그
2. 다중 선택 인터페이스
3. 연속 삭제 지원 (자식 포함)
4. 부모-자식 정리

---

### 4.10 OrderingViewController.swift

**파일 크기:** 4,464 bytes
**주요 역할:** 드래그 앤 드롭 재정렬

#### 주요 기능
1. 자식 할 일 순서 변경
2. orderIndex 저장
3. 재정렬 중 인라인 편집 비활성화

---

### 4.11 DBHandlingViewController.swift

**파일 크기:** 16,171 bytes
**주요 역할:** 데이터베이스 관리

#### 주요 기능
1. 데이터베이스 백업/복원
2. Inbox 파일 관리
3. Primary Key 관리
4. AirDrop 공유
5. 파일 작업 (이름 변경, 삭제, 복사)

---

### 4.12 SideMenuViewController.swift

**주요 역할:** 슬라이드 메뉴

#### 메뉴 항목 (모드)
1. 즐겨찾기 목록 (TableTypeFavorate)
2. 마감일 기준 목록 (TableTypeDueDate)
3. 템플릿 목록 (TableTypeTemplate)
4. 휴지통 목록 (TableTypeTrash)
5. 캘린더
6. 검색
7. 설정

#### 추가 기능
- 폴더 전환 다이얼로그
- 즐겨찾기 재정렬 (편집 모드)

---

## 5. 테이블 뷰 셀

### 5.1 TodoTableViewCell.swift

**파일 크기:** 2,301 bytes

#### 구성요소
| 컴포넌트 | 설명 |
|----------|------|
| checkButton | 완료 토글 |
| titleLabel | 할 일 제목 |
| axButton | 복제 버튼 |
| detailLabel | 미리보기 텍스트 |
| priorityLabel | 우선순위 배지 |
| childCountLabel | 자식 개수 배지 |
| urgencyColorView | 긴급도 색상 띠 |
| priorityColorView | 우선순위 색상 띠 |
| favoriteStarIconImageView | 별 표시 |
| clockIconImageView | 마감일 표시 |

#### 델리게이트 메서드
```swift
protocol ToDoCellDelegate {
    func onChecked()   // 완료 토글
    func onAxing()     // 복제 액션
    func onSelect()    // 드릴다운 탐색
}
```

---

### 5.2 PBTreeViewTableCell.swift

#### 트리 전용 기능
- 확장/축소 버튼
- 레벨 기반 들여쓰기 (레벨당 25pt)
- 즐겨찾기 표시
- 마감일 표시
- 더보기 옵션 버튼

---

### 5.3 OrderingTableViewCell.swift

**파일 크기:** 1,189 bytes

#### 구성요소
- 드래그 핸들용 단순화된 UI
- 자식 개수 배지
- 우선순위 색상 띠
- 완료된 항목 취소선

---

## 6. 매니저 및 서비스

### 6.1 DataManager.swift

**파일 크기:** 10,274 bytes
**패턴:** 싱글톤 (`DataManager.shared`)

#### 핵심 책임

**1. Realm 데이터베이스 관리**
```swift
// 기본 Realm 인스턴스 초기화
// 스키마 마이그레이션 처리 (버전 5)
// 번들 리소스 데이터베이스 복사 지원
```

**2. Primary Key 관리**
```swift
func createNewPrimaryKey() -> Int
func changePrimaryKeyIfNeeded()
// UserDefaults에 현재 최대 ID 저장
```

**3. 전역 상태**
```swift
var usingMapType: MapType
var mainViewController: ViewController
var todoCellBackgroundColor: UIColor
var todoCellSelectBackgroundColor: UIColor
```

**4. UserDefaults 속성**
| 키 | 타입 | 설명 |
|----|------|------|
| isDebug | Bool | 디버그 모드 |
| isDueDateDefault | Bool | 마감일 기본 뷰 |
| selectedFolder | Int | 현재 폴더 |
| isQuickAxing | Bool | 빠른 복사 모드 |
| isDisplayTodoDetailTextView | Bool | 상세 필드 표시 |
| hexColorCellBackgroundText | String | 테마 색상 |
| hexColorCellSelectBackgroundText | String | 선택 색상 |

**5. 트리 작업**
```swift
func getTreeNodeWithTodoId(id: Int) -> Todo?
func changFolder(todo: Todo, folderType: FolderType)
func deleteTree(todo: Todo)
func deleteTreeId(todoId: Int)
func getCompletedPercent(todo: Todo) -> Float
```

**6. 데이터베이스 마이그레이션**
```swift
static func realmMigration() {
    Realm.Configuration.defaultConfiguration = Realm.Configuration(
        schemaVersion: 5,
        migrationBlock: { migration, oldSchemaVersion in
            if oldSchemaVersion < 5 {
                migration.enumerateObjects(ofType: Todo.className()) { oldObject, newObject in
                    newObject!["isUseDueDate"] = false
                    newObject!["isUseAlarm"] = false
                    newObject!["alarmBeforeMinute"] = 0
                    newObject!["dueDate"] = ""
                }
            }
        }
    )
}
```

**7. DB 초기화 (Seed)**
- 앱 최초 실행 시 `resource/default.realm`을 Document 디렉토리로 복사
- 초기 데이터/템플릿 제공용

---

### 6.2 PBTreeViewDataHandler.swift

**역할:** 평면 Todo 목록을 계층형 트리 구조로 변환

#### 기능
- 부모-자식 관계에서 트리 노드 생성
- 확장/축소 상태 관리
- 재귀적 트리 빌딩 알고리즘

---

### 6.3 CommonUtil.swift

**파일 크기:** 6,148 bytes

#### 유틸리티 함수

**날짜 포매팅**
```swift
func getDateString() -> String        // "yyyy.MM.dd HH:mm"
func getYearMonthDayString() -> String // "yyyy. M. d"
func getYearMonthString() -> String    // "yyyy. M"
func getDayIndex() -> Int              // 날짜 숫자 추출
```

**마감일 유틸리티**
```swift
func makeDueDateDayZeroSecond() -> Date // 날짜 경계 정규화
func getDateShortDisplayStringFromNow() -> String // "오늘", "내일" 등 상대 표시
```

**UI 유틸리티**
```swift
func labelLineHeightMultiple() // 레이블 줄 간격
func appSafariOpen(url: String) // Safari에서 URL 열기
```

---

## 7. Extensions

### 7.1 UIColor+hexString.swift

#### 색상 유틸리티
```swift
// 초기화 메서드
init(red: Int, green: Int, blue: Int, alpha: CGFloat = 1.0)
init(hex: Int, alpha: CGFloat = 1.0)
init(hexString: String, alpha: CGFloat = 1.0)  // "#RRGGBB" 또는 "RRGGBB"

// 변환
func toHexString() -> String

// 색상 팔레트
static func getTodoAppColor(colorType: ColorType, index: Int) -> UIColor
```

#### 색상 타입
| 타입 | 레벨 | 색상 |
|------|------|------|
| Priority | 6개 | Pink, Orange, Yellow, Clear, Light Blue, Dark Blue |
| Urgency | 5개 | Pink, Orange, Yellow, Clear, Clear |

---

### 7.2 String+Substring.swift

#### 문자열 작업
```swift
func left(_ to: Int) -> String      // 처음 N글자
func right(_ from: Int) -> String   // 마지막 N글자
func mid(_ from: Int, amount: Int) -> String
func mid(_ from: Int) -> String     // 인덱스부터 끝까지
```

---

## 8. 서드파티 의존성

### Podfile 설정

```ruby
platform :ios, '15.0'

target 'Lighthouse' do
  use_frameworks!

  # 데이터베이스
  pod 'RealmSwift'                      # 주 데이터베이스

  # UI 컴포넌트
  pod 'BadgeSwift', '~> 8.0'           # 배지 레이블
  pod 'NextGrowingTextView'            # 자동 확장 텍스트 뷰
  pod 'JTAppleCalendar', '~> 7.1'     # 캘린더 뷰
  pod 'PopupDialog', '~> 1.1'         # 다이얼로그/알림 팝업
  pod 'SideMenu', '~> 6.0'            # 슬라이드 메뉴
  pod 'Colorful', '~> 3.0'            # 색상 선택기

  # UX
  pod 'Toast-Swift', '~> 5.0.1'       # 토스트 알림
  pod 'IQKeyboardManagerSwift'        # 키보드 관리
end
```

### 의존성 분석

| 의존성 | 용도 | 필수 | Flutter 대체 |
|--------|------|------|-------------|
| RealmSwift | 임베디드 DB | YES | sqflite, hive, isar |
| BadgeSwift | 배지 레이블 | NO | badges 패키지 |
| NextGrowingTextView | 자동 크기 텍스트 | YES | TextField (Flutter 기본) |
| JTAppleCalendar | 캘린더 뷰 | YES | table_calendar |
| PopupDialog | 알림 다이얼로그 | YES | AlertDialog (Flutter 기본) |
| SideMenu | 네비게이션 메뉴 | YES | Drawer (Flutter 기본) |
| Colorful | 색상 선택기 | NO | flutter_colorpicker |
| Toast-Swift | 알림 | NO | fluttertoast |
| IQKeyboardManagerSwift | 키보드 처리 | YES | Flutter 기본 지원 |

---

## 9. 에셋 및 리소스

### 이미지 에셋 (Assets.xcassets)

**총 123개 이미지 세트**

| 카테고리 | 예시 |
|----------|------|
| 네비게이션 | arrow_back, arrow_down, arrow_right, arrow_top |
| 액션 | axRyu (복사 아이콘), broom_black, add_nor |
| 앱 아이콘 | AppIcon (여러 크기) |
| UI 컨트롤 | buttons (edit, delete, more) |
| 인디케이터 | star, clock, checkmark 아이콘 |
| 로고 | appstore_lighthouse_final |

### 리소스 디렉토리 (`resource/`)
- 추가 이미지 에셋 (56개)
- 도움말용 PDF 가이드 (Quick guides + synopsis)
- **default.realm** - 시드 데이터베이스 (첫 실행 시 복사)

---

## 10. 네비게이션 및 앱 아키텍처

### 네비게이션 구조

```
메인 진입점 (ViewController)
├── 사이드 메뉴 (SideMenuViewController)
│   ├── 즐겨찾기 (TableType.TableTypeFavorate)
│   ├── 마감일 기준 (TableType.TableTypeDueDate)
│   ├── 캘린더 (CalendarViewController)
│   ├── 검색 (SearchViewController)
│   └── 설정 (SettingViewController)
│
├── 맵/트리 뷰 (MapViewController)
│   ├── 트리 보기 (계층 표시)
│   ├── 이동 (노드로 점프)
│   ├── 템플릿 선택 (템플릿에서 가져오기)
│   └── 이동 대상 (드래그 목적지)
│
├── 할 일 작업
│   ├── 새 할 일 (NewToDoViewController)
│   ├── 할 일 편집 (TodoEditViewController 모달)
│   ├── 할 일 복제 (Axing - 셀 액션)
│   ├── 이동/복사 (MoveViewController)
│   ├── 삭제 (DeleteViewController)
│   ├── 재정렬 (OrderingViewController)
│   └── 마감일 (DueDateSetViewController)
│
├── 앱 정보 (AppInfoViewController)
│   ├── 도움말 가이드 (WebViewController + PDF)
│   └── 설정 (SettingViewController)
│
└── 데이터베이스 관리 (DBHandlingViewController)
    ├── 백업/복원
    ├── Primary Key 관리
    └── AirDrop 공유
```

### 이벤트 통신

**NotificationCenter:**
```swift
"TreeNodeChanged"           // 데이터 갱신, 뷰 새로고침
"TreeNodeButtonClicked"     // 트리 확장/축소
"TreeNodeMoreButtonClicked" // 트리 노드 컨텍스트 메뉴
"CalendarNewTreeNodeChanged" // 캘린더에서 새 할 일
"SideMenuInitUI"            // 사이드 메뉴 UI 새로고침
```

**델리게이트 패턴:**
- RyuTodoViewControllerDelegate
- MapViewControllerDelegate
- DueDateSetViewControllerDelegate
- SideMenuViewControllerDelegate
- OrderingTableViewCellDelegate
- ToDoCellDelegate

---

## 11. 핵심 기능 요약

### 1. 계층형 할 일 관리
- motherID와 childIDs를 통한 부모-자식 관계
- 무제한 중첩 깊이
- 확장/축소 트리 시각화
- 브레드크럼 네비게이션

### 2. 다중 뷰 모드
- **리스트 뷰**: 선택된 할 일 상세 패널이 있는 평면 표시
- **트리 뷰**: 계층 확장 및 탐색
- **캘린더 뷰**: 날짜 기반 필터링

### 3. 정리 기능
- **폴더**: 현재, 휴지통, 템플릿, 보관함
- **즐겨찾기**: 별표 빠른 접근
- **마감일**: 시간 인식 스케줄링
- **우선순위**: 7단계 중요도
- **긴급도**: 5단계 긴급 표시
- **진행률**: 부모 작업 완료율 추적
- **순서 정렬**: 드래그 앤 드롭 재정렬

### 4. 데이터 작업
- **생성**: 단일 또는 일괄 생성
- **조회**: 검색, 필터, 쿼리
- **수정**: 인라인 편집, 일괄 변경
- **삭제**: 단일 또는 연속 일괄 삭제
- **복사**: 선택적 부모와 함께 복제
- **이동**: 계층 재구성

### 5. 고급 기능
- **검색**: 제목/상세 전체 텍스트 검색
- **백업/복원**: 데이터베이스 파일 관리
- **AirDrop**: 데이터베이스 파일 공유
- **테마**: UI용 사용자 정의 색상
- **템플릿**: 재사용 가능한 할 일 구조
- **디버그 모드**: 개발용 ID 표시

---

## 12. Flutter 마이그레이션 고려사항

### 주요 도전 과제

| 영역 | iOS (현재) | Flutter (대체) |
|------|-----------|---------------|
| 데이터베이스 | Realm | sqflite, hive, isar |
| UI 프레임워크 | UIKit | Flutter Widgets |
| 트리 뷰 | PBTreeView (커스텀) | flutter_fancy_tree_view |
| 캘린더 | JTAppleCalendar | table_calendar |
| 사이드 메뉴 | SideMenu | Drawer (기본 제공) |
| 키보드 관리 | IQKeyboardManager | Flutter 기본 지원 |
| 다이얼로그 | PopupDialog | AlertDialog (기본 제공) |
| 토스트 | Toast-Swift | fluttertoast |

### 권장 Flutter 패키지

```yaml
dependencies:
  # 데이터베이스
  sqflite: ^2.3.0
  # 또는
  hive: ^2.2.0
  hive_flutter: ^1.1.0
  # 또는
  isar: ^3.1.0

  # 상태 관리
  provider: ^6.1.0
  # 또는
  riverpod: ^2.4.0
  # 또는
  bloc: ^8.1.0

  # 캘린더
  table_calendar: ^3.0.9

  # 트리 뷰
  flutter_fancy_tree_view: ^1.4.0

  # 날짜/시간
  intl: ^0.18.0

  # 저장소
  shared_preferences: ^2.2.0

  # UI 컴포넌트
  flutter_slidable: ^3.0.0
  fluttertoast: ^8.2.0
  flutter_colorpicker: ^1.0.0
```

### 마이그레이션 단계

**Phase 1: 데이터 레이어**
1. Dart로 Todo 모델 추출
2. 영속성을 위한 SQLite/Hive/Isar 구현
3. 모든 DataManager 작업 복제
4. **권장**: `dueDate`를 String이 아닌 **DateTime**으로 저장

**Phase 2: 핵심 UI**
1. TreeView 위젯으로 메인 리스트 뷰 구현
2. Todo 셀 컴포넌트 생성
3. 상세 패널 UI 빌드

**Phase 3: 기능**
1. 캘린더 통합
2. 검색 기능
3. 이동/복사/삭제 작업
4. 설정 시스템

**Phase 4: 마무리**
1. 테마 시스템 포팅
2. 애니메이션 및 전환
3. 성능 최적화

### 상태 관리 권장사항

- NotificationCenter → **Riverpod / BLoC**로 대체
- 전역 앱 상태 관리 필요:
  - 선택된 폴더
  - 현재 부모 노드
  - 선택 상태
  - 설정값들

### 반드시 보존해야 할 기능

1. 트리 네비게이션
2. 복제/편집 동작 (Axing)
3. 이동/복사 제약조건 (자손으로 이동 방지)
4. 템플릿 + 휴지통
5. 마감일 캘린더
6. DB 백업/복원 (필요시)

---

## 13. 파일 위치 요약

### 루트 레벨 (13개 뷰 컨트롤러)
```
/Users/cst/문서_로컬/ryuSwift/Lighthouse/
├── ViewController.swift
├── AppDelegate.swift
├── CalendarViewController.swift
├── MapViewController.swift
├── MoveViewController.swift
├── SearchViewController.swift
├── SettingViewController.swift
├── AppInfoViewController.swift
├── DBHandlingViewController.swift
├── DataFixManagerViewController.swift
├── TodoEditViewController.swift
├── DueDateSetViewController.swift
└── WebViewController.swift
```

### 중첩 클래스 (13개 파일)
```
/Users/cst/문서_로컬/ryuSwift/Lighthouse/Lighthouse/classes/
├── Common/
│   └── CommonUtil.swift
├── Extensions/
│   ├── UIColor+hexString.swift
│   └── String+Substring.swift
├── PBTreeView/
│   ├── PBTreeViewData.swift
│   ├── PBTreeViewDataHandler.swift
│   ├── PBTreeViewNodeItem.swift
│   └── PBTreeViewTableCell.swift
├── SideMenu/
│   └── SideMenuViewController.swift
├── NewToDoViewController.swift
├── OrderingViewController.swift
├── OrderingTableViewCell.swift
├── DeleteViewController.swift
└── TodoTableViewCell.swift
```

### 핵심 파일
```
/Users/cst/문서_로컬/ryuSwift/Lighthouse/
├── Todo.swift (핵심 데이터 모델)
├── DataManager.swift (데이터 관리자)
├── Podfile (의존성 설정)
└── Base.lproj/Main.storyboard (메인 UI)
```

---

## 14. 결론

**Lighthouse**는 UIKit 기반의 잘 구조화된 계층형 할 일 관리 앱입니다. Realm 데이터베이스 백엔드와 함께 견고한 아키텍처 패턴을 보여줍니다.

Flutter 마이그레이션을 위한 주요 작업:
1. **데이터 레이어 재생성** - 동등한 영속성 구현
2. **UI 컴포넌트 마이그레이션** - UIKit에서 Flutter 위젯으로
3. **네비게이션 재구성** - Flutter 패턴에 맞게
4. **서드파티 의존성 대체** - Flutter 동등 라이브러리로

모듈화된 구조와 명확한 관심사 분리로 적절한 계획을 세우면 비교적 원활한 마이그레이션이 가능합니다.

---

## 15. 다음 단계 권장사항

1. **Flutter 데이터 모델 정의** - Todo + 관계 구조 확정
2. **UI 범위 결정** - 기존 UI 유지 vs 완전 재디자인
3. **DB 스택 선택** - Isar / Hive / Realm 중 결정 + 마이그레이션 포맷
4. **화면별 Flutter 와이어프레임** 작성
5. **데이터 레이어 + 시드 마이그레이션** 우선 구현

---

## 16. 정확한 포팅을 위한 필수 정합성 (추가)

### 16.1 데이터 정합성 규칙
- `motherID`와 `childIDs`는 반드시 상호 일치해야 함.
- `childIDs` 순서가 곧 정렬 순서이며, `orderIndex`와 항상 동기화돼야 함.
- 삭제 시: 부모의 `childIDs`에서 해당 ID 제거 후 서브트리 재귀 삭제.
- 이동 시: 기존 부모의 `childIDs`에서 제거 → 새 부모에 추가 → `motherID` 변경.
- 폴더 이동 시: 자식 포함 재귀적으로 `folderType` 갱신.
- 복제(`copyTree`)는 서브트리 깊은 복사이며 새 `todoID`를 할당해야 함.

### 16.2 UI/행동 일치 요구사항
- 메인 화면은 **선택된 부모의 직계 자식만** 표시한다.
- `isDisplayTodoDetailTextView`가 `false`면 상세 입력 숨기고 제목은 최소 2줄.
- 완료율 표시 로직: 자식 없음 → 999, 0 또는 999일 때 “Completed” 표기.
- 즐겨찾기/완료/우선순위 변경 시 즉시 DB 반영 후 전역 갱신 이벤트 발생.
- `isQuickAxing`에 따라 복제 버튼 이미지가 바뀐다.
- 검색 결과 선택 시 해당 폴더로 전환 후 부모로 이동.

### 16.3 날짜/시간 처리
- dueDate는 `"yyyy.MM.dd HH:mm"` 문자열 기반으로 비교/표시됨.
- 캘린더 일별 필터는 **해당 날짜 00:00~다음날 00:00** 범위로 비교.
- `getDateShortDisplayStringFromNow()`의 Today/Yesterday/Tomorrow 표시 규칙 유지.

### 16.4 글로벌 이벤트 흐름
- `TreeNodeChanged`는 대부분 화면 리프레시 트리거다.
- 캘린더 신규 생성 시 `CalendarNewTreeNodeChanged` 추가 발생.
- 트리맵 확장/축소는 `TreeNodeButtonClicked` → `RelodeTreeView` 흐름.

### 16.5 초기 데이터/백업
- 첫 실행 시 `resource/default.realm`을 Document로 복사.
- `primary_key`는 현재 최대 `todoID`와 항상 동기화.
- 백업/복원은 Document/backup 폴더 파일 단위로 수행.

---

*이 문서는 Flutter 마이그레이션 계획을 위해 작성되었습니다.*
*Codex 보완 분석 반영: 2026-02-04*
*최종 업데이트: 2026-02-04*
