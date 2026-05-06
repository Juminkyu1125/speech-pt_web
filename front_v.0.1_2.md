# Front v0.1.2 Summary

## 개요

이번 프론트엔드 변경은 기존 바닐라 JS 구조를 유지하면서, 분석 화면과 연습 모드에 더 밝고 테크놀로지한 분위기를 입히는 데 초점을 맞췄다. 전체 톤은 화이트 기반의 밝은 화면을 유지하되, 블루 네온 계열 하이라이트와 유리 패널 느낌을 추가해 기존보다 더 현대적인 인상을 주도록 조정했다.

## 현재 구조

- 프론트엔드는 React가 아닌 정적 HTML + 바닐라 JavaScript 모듈 구조를 유지한다.
- 주요 화면은 `frontend/analysis.html` 중심으로 동작한다.
- 상태 변화와 상호작용은 `frontend/js/analysis.js`, `frontend/js/practice-mode.js` 등에서 직접 DOM을 제어하는 방식이다.

## 새로 반영된 핵심 요소

### 1. 전체 비주얼 톤 개선

- `frontend/css/common.css`
  - 기본 폰트를 `Segoe UI`, `Pretendard`, `Noto Sans KR` 계열로 조정
  - 배경을 단색 회색에서 밝은 블루 계열 그라데이션 배경으로 변경
  - 전체적으로 더 가볍고 시원한 화면 인상으로 조정

### 2. 분석 페이지 분위기 개편

- `frontend/css/analysis.css`
  - 분석 페이지 전체에 은은한 광원 느낌의 배경 레이어 추가
  - 카드 컴포넌트(`feedback-box`, `chat-header`, `chat-body`, `score-card`)를 반투명 유리 패널 스타일로 재구성
  - 카드 경계선, 그림자, blur 효과를 활용해 밝은 테크 UI 느낌 강화

### 3. 문서 미리보기 영역 강화

- 좌측 문서 미리보기 박스를 기존 단순 어두운 박스에서
  - 블루 계열 그라데이션
  - 은은한 그리드 패턴
  - 미래적인 패널 느낌
  으로 개선
- PDF는 `iframe` 기반 미리보기 유지
- PPT/PPTX는 파일 정보 카드 형태로 노출되도록 스타일 보강

### 4. 채팅/분석 인터페이스 개선

- 중앙 분석 헤더에 `LIVE ANALYSIS` 배지 추가
- 채팅 말풍선을 흰색/블루 그라데이션 기반으로 재설계
- 사용자 메시지는 블루 네온 계열 하이라이트로 강조
- 상태 메시지도 일반 메시지와 구분되도록 별도 배경과 보더 스타일 적용
- 업로드된 첨부 파일 칩도 pill 형태의 밝은 카드 스타일로 개선

### 5. 입력창 및 버튼 인터랙션 보강

- 입력창 포커스 시 블루 glow가 생기도록 처리
- 전송 버튼/첨부 버튼에 그라데이션 및 hover 애니메이션 추가
- 버튼 hover 시 살짝 떠오르는 느낌이 나도록 전환 효과 반영

### 6. 연습 모드 UI 연결

- `frontend/analysis.html`
  - 연습모드 버튼에 `id="practiceModeButton"` 추가
  - 연습 모드 모달 마크업 추가
  - 점수 링, 웨이브 캔버스, 볼륨/피치 미터, 마이크 버튼 영역 추가

- `frontend/js/analysis.js`
  - `practice-mode.js`와 연결
  - 분석 완료 후 연습 모드 버튼 활성화
  - 최신 분석 점수를 연습 모드에 전달하도록 처리

- `frontend/js/practice-mode.js`
  - 연습 모드 모달 열기/닫기
  - 마이크 권한 요청 및 실시간 오디오 분석
  - 웨이브폼, 볼륨, 피치, 실시간 점수 링 렌더링
  - 대기 상태 애니메이션과 마이크 활성 상태 애니메이션 처리

### 7. 연습 모드 스타일 고도화

- `frontend/css/analysis.css`
  - 연습 모드는 메인 화면보다 더 진한 다크 테크 분위기로 유지
  - 모달 내부에 블루 glow, 격자 오버레이, 반투명 버튼 효과 추가
  - 마이크 활성화 시 pulse 애니메이션 적용
  - 메인 화면과는 다른 집중형 UX를 주되, 같은 블루 계열 브랜드 톤을 공유

## 디자인 방향 요약

현재 프론트 디자인 방향은 다음과 같이 정리할 수 있다.

- 밝은 배경 기반
- 차가운 블루/시안 포인트 컬러
- 유리 패널 느낌의 카드 UI
- 과하지 않은 glow와 shadow
- 연습 모드에서 더 강한 테크 감성
- 작은 hover/transition으로 동적 인상 강화

## 현재 상태에서의 장점

- React로 전환하지 않고도 UI 완성도를 빠르게 높일 수 있음
- 기존 구조를 크게 깨지 않아 기능과 스타일 작업을 병행하기 쉬움
- 분석 화면과 연습 모드의 톤이 한 방향으로 정리되기 시작함
- 추후 `login`, `note` 화면도 같은 디자인 시스템으로 확장 가능

## 다음 추천 작업

### 우선순위 높은 것

- `note.css` 화면도 같은 디자인 시스템으로 통일
- `login.html` / `common.css` 로그인 카드도 현재 톤에 맞춰 리디자인
- 버튼/카드/배지 스타일을 공통 규칙으로 정리

### 이후 확장 가능

- 카드 등장 애니메이션 강화
- 연습 모드 웨이브와 점수 변화에 더 많은 시각 효과 추가
- 분석 결과 섹션별 강조 UI 추가
- 색상, radius, shadow를 CSS 변수로 분리해 디자인 시스템화

## 관련 파일

- [frontend/css/common.css](/Users/alsrb1125/Desktop/파란학기/SpeechPt/frontend/css/common.css)
- [frontend/css/analysis.css](/Users/alsrb1125/Desktop/파란학기/SpeechPt/frontend/css/analysis.css)
- [frontend/analysis.html](/Users/alsrb1125/Desktop/파란학기/SpeechPt/frontend/analysis.html)
- [frontend/js/analysis.js](/Users/alsrb1125/Desktop/파란학기/SpeechPt/frontend/js/analysis.js)
- [frontend/js/practice-mode.js](/Users/alsrb1125/Desktop/파란학기/SpeechPt/frontend/js/practice-mode.js)
