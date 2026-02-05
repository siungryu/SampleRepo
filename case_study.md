# 멀티플랫폼 포팅 실전 기록: iOS 앱 → Flutter 전환과 AI 협업
> 여행자 사이먼 / 2026-02-05 / 안드로이드 앱 개발

## 📝 3줄 요약
1. UIKit + Realm 기반 iOS 앱을 Flutter로 포팅해 iOS/Android/macOS 빌드까지 성공했다. 다만 UI는 크게 깨져 재설계가 필요했다.
2. 분석/설계는 Claude CLI, 구현/테스트는 Codex Extension이 주도하는 분업 구조로 진행했다.
3. 작업기간은 2026-02-04 09:53:41 ~ 15:37:00(총 5시간 43분 19초), **빌드는 성공했지만 UI가 많이 깨져 체감 79점**이었다.

## 이런 분들께 도움돼요
- iOS 앱을 Flutter로 멀티플랫폼 포팅하려는 분
- AI를 “분석/설계”와 “실행”으로 역할 분리해 협업하고 싶은 분
- Realm 기반 앱의 데이터 정합성과 마이그레이션을 안전하게 다루고 싶은 분

## 이 글을 쓰게 된 계기
기존 앱은 UIKit + Realm 기반의 계층형 할 일 관리 앱이었다. iOS 전용 구조를 Flutter로 옮겨 iOS/Android/macOS까지 확장하려 했고, 포팅 규모가 커서 **AI 협업을 구조화**해 생산성을 높이는 방식으로 접근했다.
Rose’s 플러그인으로 Claude 응답 시점마다 자동 스크린샷을 남겼고, 일부 잘못 찍힌 컷은 삭제했지만 남아 있는 54장으로 전체 맥락을 복원했다.

## 준비물
- Claude CLI
- Codex Extension
- Flutter SDK
- Realm Flutter
- 기존 iOS UIKit + Realm 코드베이스
- 산출물 문서: `Ryu-write-guide/did/ANALYSIS_FOR_FLUTTER_MIGRATION.md`, `Ryu-write-guide/did/plan.md`, `Ryu-write-guide/did/todo.md`
- Rose’s 플러그인(스크린샷 자동 후킹)

## Step-by-Step 과정 (4단계 요약)
1. **분석(09:53–11:35)**
빌드 에러 원인부터 추적했다. NextGrowingTextView 사용 위치, iOS deployment target, Podfile/Podspec/Xcode 설정을 조사했고, 이후 전체 코드 구조를 전수 분석해 `ANALYSIS_FOR_FLUTTER_MIGRATION.md`에 정리했다. 이 단계에서 데이터 정합성 규칙과 핵심 동작 요구사항을 확정했다.

2. **계획(12:02–12:14)**
`plan.md`와 `todo.md`를 확정했다. 듀얼 필드 마이그레이션(dueDateString → dueDateAt), Quick Axing 자동 이동, 복제 시 편집 팝업, case-insensitive 검색, Post‑MVP 알람 등을 명시해 구현 기준을 고정했다.

3. **구현/테스트(14:19–14:35)**
Codex가 MapType 확장 규칙, 이동/복사, 정렬, repository 쿼리 수정 등을 수행했다. in-memory Realm 테스트를 추가하고 `flutter test` 통과로 정합성 규칙을 “테스트로 고정”했다.

4. **빌드/디버깅(14:40–15:37)**
Android 빌드와 iPhone 실기기 실행을 확인했다. Android 빈 화면 문제는 schemaVersion 불일치와 seed DB 로드 흐름으로 추적했고, iOS는 CocoaPods 설치 누락과 Runner.xcworkspace 오픈 문제로 정리했다.

## 삽질 기록 & 해결 방법 (결과 포함)
- **iOS 빌드 실패**: Podfile/Deployment Target 충돌 → Podfile post_install + pbxproj 타깃 정리 → iOS 빌드 재시도 경로 확보
- **Android 빈 화면**: schemaVersion 불일치 + seed DB 로드 누락 → schemaVersion 6 통일 + Reset DB 도입 → 초기 데이터 표시 가능
- **CocoaPods 미설치**: `pod install` 누락 → `pod install` 실행 + `Runner.xcworkspace`로 열기 → iOS 의존성 정상 연결
- **Realm 임시파일 증가**: `.management`, `.lock` 생성 → `flutter clean` + build 폴더 정리 → 빌드 캐시 정상화

## 핵심 인사이트
- **분석/설계는 Claude, 실행은 Codex**가 효율적이었다.
- “문서 → 구현 → 테스트” 파이프라인이 명확할수록 협업 품질이 안정됐다.
- 포팅 성공 여부는 UI보다 **데이터 정합성과 마이그레이션 안정성**이 먼저다.

## 마무리
iOS/Android/macOS 빌드까지 성공했고, UI는 재작업 중이다.
Rose’s 플러그인으로 남긴 스크린샷 덕분에, 포팅 과정의 중요한 프롬프트와 AI 응답을 역으로 추적해 정리할 수 있었다.
전체 작업시간은 5시간 43분 19초, 체감 성과는 79점. “분석은 Claude, 구현은 Codex” 분업이 가장 큰 생산성 포인트였다.

## Q&A
Q. UI는 완성됐나요?
A. 빌드는 성공했지만 UIKit → Flutter 전환 과정에서 레이아웃/폰트/컴포넌트가 재해석되어 디자인이 깨졌고, UI 재설계를 진행 중이다.

Q. 왜 Claude CLI와 Codex를 분리해서 썼나요?
A. Claude는 분석/설계·문서화에 강했고, Codex는 구현·수정·테스트 자동화에 강했다. 무엇보다도 Codex는 pro요금제라 토큰이 넘쳐나서 주로 모든 잡일을 시키고 혼자하는것 보다 다른 agent와 협업을 시키면 빈틈을 많이 줄여준다.

Q. Windows 빌드는 어떻게 하나요?
A. 일반적으로 `flutter build windows`로 빌드하면 `build/windows/runner/Release`에 exe가 생성된다. 안타깝게도 내 노트북이 맥북이라 윈도우 빌드는 못해보았음.

## 이미지(대표 컷 + 설명)
- `session_20260204_095332/screenshot_20260204_095341_480.png` — 초기 빌드 요청과 CocoaPods 상태 확인
- `session_20260204_105816_462.png` — 분석 범위·결과 파일 결정 장면
- `session_20260204_111410_531.png` — 정합성 규칙과 핵심 요구사항 보강
- `session_20260204_120248_912.png` — plan.md로 포팅 로드맵 확정
- `session_20260204_121227_450.png` — todo.md 백로그 한국어 정리
- `session_20260204_143508_904.png` — 테스트 통과 및 남은 Post‑MVP 정리
- `session_20260204_144035_307.png` — Android 빌드 실행 화면
- `session_20260204_150050_611.png` — Android 빈 화면 원인 분석과 seed DB 대응
- `session_20260204_152112_061.png` — iOS Pod install 및 Runner.xcworkspace 안내
- `session_20260204_153700_811.png` — Realm 임시파일 정리 및 Windows 빌드 질문
