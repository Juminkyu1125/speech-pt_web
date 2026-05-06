# 연습 모드 (Practice Mode) 구현 문서

## 개요

분석이 완료된 후 사용자가 실제로 발표를 연습할 수 있는 모달 기반 인터페이스.
마이크를 통해 실시간으로 목소리를 받아, 분석 결과 점수를 기준으로 현재 발화 상태를 시각적으로 피드백한다.

---

## 동작 흐름

```
분석 페이지 진입
    ↓
"연습모드" 버튼 비활성(disabled) 상태
    ↓
문서 + 음성 파일 업로드 → 분석 실행 → 분석 완료
    ↓
fetchAnalysisResult onComplete 콜백 호출
    ↓
latestAnalysisScores 저장 + 버튼 활성화
    ↓
"연습모드" 버튼 클릭
    ↓
연습 모드 모달 열림 (분석 점수 전달)
    ↓
마이크 버튼 클릭 → getUserMedia 권한 요청
    ↓
실시간 AudioContext 분석 시작 (60fps RAF 루프)
    ↓
웨이브폼 / 볼륨 미터 / 피치 미터 / 링 게이지 업데이트
```

---

## 파일 구성

| 파일 | 역할 |
|------|------|
| `frontend/analysis.html` | 연습 모드 모달 마크업, 링 캔버스, 웨이브 캔버스, 미터, 마이크 버튼 |
| `frontend/js/practice-mode.js` | 연습 모드 전체 로직 (Audio API, 시각화, 링 렌더링) |
| `frontend/js/analysis.js` | practice-mode import, 버튼 활성화 연결, 점수 전달 |
| `frontend/js/analysis-service.js` | `fetchAnalysisResult`에 `onComplete` 콜백 추가 |
| `frontend/css/analysis.css` | 연습 모드 모달 및 내부 컴포넌트 스타일 |

---

## HTML 구조 (`analysis.html`)

```html
<!-- 연습모드 버튼 (분석 완료 전까지 disabled) -->
<button type="button" id="practiceModeButton" disabled>연습모드</button>

<!-- 연습 모드 모달 -->
<div class="practice-modal-backdrop" id="practiceModeModal">
  <div class="practice-modal">
    <!-- 닫기 버튼 -->
    <button id="closePracticeModal">×</button>
    <h2>연습 모드</h2>

    <!-- 분석 점수 링 게이지 (3개) -->
    <div class="practice-score-rings">
      <canvas id="practiceRingContent">  <!-- 내용 커버리지 -->
      <canvas id="practiceRingDelivery"> <!-- 전달 안정성 -->
      <canvas id="practiceRingPacing">   <!-- 발표 속도 -->
    </div>

    <!-- 실시간 웨이브폼 캔버스 -->
    <div class="practice-wave-container">
      <canvas id="practiceWaveCanvas"></canvas>
    </div>

    <!-- 볼륨 / 피치 미터 -->
    <div class="practice-meters">
      <div id="practiceVolumeFill"> <!-- 볼륨 바 -->
      <div id="practicePitchFill">  <!-- 피치 바 -->
    </div>

    <!-- 마이크 토글 버튼 -->
    <button id="practiceMicButton">
  </div>
</div>
```

---

## JS 모듈 상세 (`practice-mode.js`)

### 공개 함수

| 함수 | 설명 |
|------|------|
| `initPracticeMode()` | DOM 이벤트 등록, 링 캔버스 DPR 설정. `initAnalysisPage()`에서 호출 |
| `openPracticeModal(scores)` | 모달 표시, 점수 저장, 아이들 애니메이션 시작, 링 초기 렌더 |
| `closePracticeModal()` | 모달 닫기, 마이크 해제, 애니메이션 정리, UI 초기화 |

### 내부 흐름

#### 아이들 상태 (마이크 OFF)
- `startIdleAnimation()` → RAF 루프에서 `drawIdleWave()` 호출
- 두 개의 사인파를 합산한 부드러운 유동 곡선을 canvas에 렌더
- `"마이크를 시작하면 실시간 음성이 표시됩니다"` 안내 텍스트 오버레이

#### 마이크 활성 상태 (마이크 ON)
- `navigator.mediaDevices.getUserMedia({ audio: true })`
- `AudioContext` → `AnalyserNode` (fftSize: 2048, smoothing: 0.85)
- `MediaStreamAudioSourceNode.connect(analyser)`
- RAF 루프 `drawLiveLoop()` 에서 매 프레임:
  1. `getByteTimeDomainData` → 파형 그리기
  2. `getByteFrequencyData` → 주파수 막대 배경
  3. `getFloatTimeDomainData` → 피치 감지
  4. RMS 계산 → 볼륨 미터 업데이트
  5. 링 게이지 재렌더 (실시간 vs 목표 비교)

### 피치 감지 알고리즘

자기상관(autocorrelation) 방식으로 구현:

```
1. 버퍼 RMS < 0.008 이면 -1 반환 (무음 처리)
2. 크기 1024 슬라이스에 대해 lag별 autocorrelation 계산
3. 초기 감소 구간(d) 이후 최댓값 위치(maxPos) 탐색
4. Parabolic interpolation으로 소수점 정밀도 보정
5. 반환값: sampleRate / truePos (Hz)
```

측정 범위: 80 Hz ~ 1200 Hz (사람 목소리 범위)

### 볼륨 계산

```
RMS = sqrt( mean( ((sample - 128) / 128)^2 ) )
볼륨% = min(RMS × 400, 100)
```

색상 피드백:
- 0~10% → 회색 (너무 조용)
- 10~20% → 노랑 (약함)
- 20~65% → 초록 (적정)
- 65%~ → 빨강 (너무 큼)

### 링 게이지 렌더링

각 링은 두 겹의 원호(arc)로 구성:

| 레이어 | 데이터 | 색상 |
|--------|--------|------|
| 배경 트랙 | 없음 | rgba(흰색, 0.07) |
| 외부 아크 | 분석 목표 점수 | 파랑 그라디언트 (#4a6cff → #5da8ff) + glow |
| 내부 아크 | 실시간 현재 값 | 목표와의 차이에 따라 초록/노랑/빨강 |

내부 아크 색상 기준 (목표 점수와의 차이):
- diff < 15 → `#4aff8c` (초록, 잘 맞음)
- diff < 30 → `#ffda4a` (노랑, 근접)
- diff ≥ 30 → `#ff6b6b` (빨강, 차이 큼)

실시간 점수 매핑:
- `전달 안정성` 링 ← `RMS × 500` (볼륨 기반)
- `발표 속도` 링 ← `(pitch - 80) / 320 × 100` (피치 기반)
- `내용 커버리지` 링 ← 실시간 측정 불가, 목표 점수만 표시

### Canvas DPR 처리

링 캔버스와 웨이브 캔버스 모두 `devicePixelRatio`를 반영해 레티나 디스플레이에서도 선명하게 렌더:

```js
canvas.width  = cssSize * dpr;
canvas.height = cssSize * dpr;
canvas.style.width  = cssSize + "px";
canvas.style.height = cssSize + "px";

// 매 draw call에서:
ctx.save();
ctx.scale(dpr, dpr);
// ... CSS 픽셀 단위로 그리기 ...
ctx.restore();
```

---

## analysis.js 연결 구조

```js
// 1. import
import { initPracticeMode, openPracticeModal } from "./practice-mode.js";

// 2. elements에 버튼 참조 추가
practiceModeButton: document.getElementById("practiceModeButton"),

// 3. 페이지 초기화 시
initPracticeMode();
setButtonDisabled(elements.practiceModeButton, true); // 초기 비활성

elements.practiceModeButton?.addEventListener("click", () => {
  if (latestAnalysisScores) openPracticeModal(latestAnalysisScores);
});

// 4. 분석 완료 콜백
fetchAnalysisResultService({
  ...,
  onComplete: (scores) => {
    setButtonDisabled(elements.practiceModeButton, false); // 버튼 활성화
    latestAnalysisScores = scores; // 점수 저장
  },
});
```

---

## analysis-service.js 변경 사항

`fetchAnalysisResult`에 `onComplete` 옵셔널 파라미터 추가:

```js
export async function fetchAnalysisResult({
  ...,
  onComplete  // 추가됨
}) {
  // ... 기존 렌더링 로직 ...

  if (onComplete) {
    onComplete({
      contentCoverage:    result.scores?.content_coverage   ?? null,
      deliveryStability:  result.scores?.delivery_stability ?? null,
      pacingScore:        result.scores?.pacing_score       ?? null,
    });
  }
}
```

---

## CSS 주요 클래스 (`analysis.css`)

| 클래스 | 설명 |
|--------|------|
| `.practice-modal-backdrop` | 전체 화면 어두운 오버레이, `display: none` → `.active` 시 `display: flex` |
| `.practice-modal` | 다크 그라디언트 배경 모달 컨테이너 (max-width: 880px) |
| `.practice-score-rings` | 3개 링 게이지를 가로로 배치하는 flex 컨테이너 |
| `.practice-wave-container` | 높이 160px 고정, 웨이브 캔버스 wrapper |
| `.practice-meter-track` | 미터 배경 바 |
| `.practice-meter-fill` | 미터 채움 바 (width 트랜지션 0.04s) |
| `.practice-mic-button` | 원형 마이크 버튼 |
| `.practice-mic-button.active` | 활성 상태 — 빨강 테두리 + `micPulse` 애니메이션 |

---

## 브라우저 요구 사항

| 기능 | 필요 API |
|------|----------|
| 마이크 접근 | `navigator.mediaDevices.getUserMedia` |
| 오디오 분석 | `AudioContext`, `AnalyserNode` |
| 캔버스 렌더 | `HTMLCanvasElement` 2D |
| 스무딩 | `requestAnimationFrame` |

> HTTPS 또는 localhost 환경에서만 `getUserMedia` 사용 가능

---

## 알려진 한계

- **내용 커버리지**는 실시간 측정 불가 (STT 없이는 발화 내용 분석이 안 됨). 링에 목표 점수만 표시.
- **발표 속도(pacing)** 링의 실시간 값은 피치(Hz)를 재매핑한 근사치. 실제 발화 속도(WPM)와 다를 수 있음.
- 자기상관 피치 감지는 유성음에서만 정확하고, 무성자음이나 잡음에서는 -1을 반환.
- 피치 연산은 1024 샘플 슬라이스 O(n²) 루프 — 느린 기기에서 RAF 드롭 가능.

---

## 향후 개선 방향

- 피치 감지를 YIN 또는 AMDF 알고리즘으로 교체해 정확도 향상
- 웹 워커(Web Worker)로 피치 연산을 분리해 메인 스레드 부하 감소
- Web Speech API 또는 Whisper API 연동으로 내용 커버리지 실시간 측정
- 연습 세션 녹음 및 전/후 비교 기능 추가
- 목소리 피치 히스토리 그래프 (시간축 기반)

---

## 관련 파일

- [frontend/analysis.html](frontend/analysis.html)
- [frontend/js/practice-mode.js](frontend/js/practice-mode.js)
- [frontend/js/analysis.js](frontend/js/analysis.js)
- [frontend/js/analysis-service.js](frontend/js/analysis-service.js)
- [frontend/css/analysis.css](frontend/css/analysis.css)
