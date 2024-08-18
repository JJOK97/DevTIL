# WebRTC와 MediaPipe 성능 최적화


WebRTC(OpenVidu)를 활용해 비디오 연결까지는 무난하게 진행됐다. 그러나 실제 사용 중 다음과 같은 문제들이 발생했다:

-   퍼블리셔와 서브스크라이버 모든 영상에 MediaPipe를 활용해 배경 제거 기능을 추가함
-   여러 명이 연결되면서 성능이 급격히 저하됨
-   GPU가 과부하로 동작하게 됨

처음에는 단순히 비디오를 출력하는 것만으로 문제가 없었으나, 배경 제거 기능을 추가하면서 상황이 심각해졌다.

<br>

## 문제 분석: MediaPipe의 GPU 자원 소비

실시간으로 MediaPipe를 적용하니 GPU 자원을 지나치게 많이 소모하게 됐다. 특히 다음과 같은 점들이 문제였다:

-   퍼블리셔와 서브스크라이버 모든 비디오 스트림에 배경 제거 기능을 적용
-   MediaPipe를 싱글톤으로 구현하여 병렬 처리가 불가능
-   실시간으로 계속 동작하니 GPU가 과부하 상태에 도달

특히 로컬 환경에서 여러 개의 브라우저를 통해 테스트를 하게 되면, 한 로컬 시스템에서 중복된 많은 비디오를 처리하게 되어 GPU 과부화가 더욱 심해졌다.

<br>

## 해결책 모색: 다중 스레드와 병렬 처리

이 문제를 해결하기 위해 여러 가지 방법을 시도했다

-   WebWorker를 활용한 다중 스레드 적용
-   MediaPipe 인스턴스를 여러 개 생성하여 병렬 처리
-   GPU 사용 최적화 시도

<br>

### 1. WebWorker를 통한 다중 스레드 구현

WebWorker를 사용하면 무거운 작업을 메인 스레드가 아닌 별도의 스레드에서 처리할 수 있다. WebWorker가 메인 스레드와 메세지 통신을 통해 작업 결과를 비동기적으로 받아 처리를 하게 됨으로써 UI가 버벅거리는 현상을 줄일 수 있을 것으로 기대했다.

```javascript
// WebWorker 생성
const worker = new Worker('mediapipe-worker.js');

// 비디오 프레임 전송
worker.postMessage({
    type: 'processFrame',
    frame: videoFrame,
    streamId: streamId,
});

// 결과 수신 후 처리
worker.onmessage = function (e) {
    if (e.data.type === 'processedFrame') {
        // 처리된 프레임을 화면에 그리기
        applyProcessedFrame(e.data.streamId, e.data.processedFrame);
    }
};
```

<br>

### 2. MediaPipe 인스턴스 다중화

모든 스트림을 하나의 MediaPipe 인스턴스로 처리할 경우 성능 저하가 발생할 수 있다고 판단하였고고, 각 스트림마다 별도의 MediaPipe 인스턴스를 할당해 병렬로 처리하도록 하여 성능을 최적화하려고 하였다.

```javascript
class MediaPipeManager {
    constructor() {
        this.instances = new Map();
    }

    getOrCreateInstance(streamId) {
        if (!this.instances.has(streamId)) {
            this.instances.set(streamId, new MediaPipeInstance());
        }
        return this.instances.get(streamId);
    }

    processFrame(streamId, frame) {
        const instance = this.getOrCreateInstance(streamId);
        return instance.processFrame(frame);
    }
}
```

<br>

### 3. GPU 사용 최적화

GPU를 보다 효율적으로 사용하기 위해 다음과 같은 방법들을 시도했다:

-   해상도를 낮춰 배경 제거 후 다시 해상도를 높이기
-   프레임 수를 조절하기

<br>

### 다양한 기기 성능 고려의 중요성

이런 시도들 덕분에 개발용 컴퓨터(GPU 성능이 뛰어난 기기)에서는 성능이 개선되었으나, 다른 기기들(노트북, 태블릿, 스마트폰 등)에서 테스트해보니 여전히 문제가 있었다. 특히 구형 기기들은 성능이 현저하게 떨어졌다. 이로부터 얻은 교훈은 다음과 같다:

-   하나의 해결책으로 모든 상황을 커버할 수 없다.
-   사용자 기기 성능에 따라 동적으로 최적화가 필요하다.
-   때로는 핵심 기능이 제대로 동작하도록 하기 위해 일부 기능을 포기해야 할 수도 있다.

<br>

## 앞으로의 계획

-   기기 성능을 체크
-   성능 등급별 최적화 전략 수립 (좋음, 보통, 나쁨)
-   사용자가 성능을 직접 조절할 수 있는 옵션 제공
-   계속해서 성능을 모니터링하고 피드백을 받는 시스템 개발

```arduino
// TODO: 성능 테스트 자동화 스크립트 작성하기
// FIXME: MediaPipe 인스턴스에서 메모리 누수가 발생할 가능성 있음, 확인 필요
// NOTE: 저사양 기기에서 배경 제거 기능을 완전히 끄는 옵션 고려하기
```
