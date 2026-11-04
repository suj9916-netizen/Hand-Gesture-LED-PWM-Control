# Hand-Gesture-LED-PWM-Control
# Hand Gesture LED PWM Control

## 1. 프로젝트 개요

p5.js와 ml5.js의 HandPose 기능을 이용하여 엄지와 검지 사이의 거리를 측정하고, 그 값을 슬라이더에 반영하여 Arduino에 연결된 LED의 밝기를 조절하는 프로젝트이다.

손가락 사이가 가까워지면 LED가 어두워지고, 멀어지면 LED가 밝아지도록 구현하였다.

---

## 2. 동작 방식

전체 동작 과정은 다음과 같다.

```text
엄지와 검지 사이 거리 측정
        ↓
0 ~ 100 범위의 슬라이더 값으로 변환
        ↓
0 ~ 255 범위의 PWM 값으로 변환
        ↓
Serial 통신으로 Arduino에 전송
        ↓
LED 밝기 조절
```

예를 들어 슬라이더 값이 50이라면 약 128 정도의 PWM 값으로 변환된다.

```text
Slider 0   → PWM 0   → LED OFF
Slider 50  → PWM 128 → 중간 밝기
Slider 100 → PWM 255 → 최대 밝기
```

---

## 3. HandPose

ml5.js의 HandPose를 사용하여 카메라에서 손의 주요 위치를 인식하였다.

엄지 끝은 `keypoints[4]`, 검지 끝은 `keypoints[8]`을 사용하였다.

```javascript
let thumb = hand.keypoints[4];
let index = hand.keypoints[8];
```

두 점 사이의 거리를 `dist()` 함수로 계산하였다.

```javascript
let fingerDistance = dist(
  thumb.x, thumb.y,
  index.x, index.y
);
```

손을 카메라에 가까이하거나 멀리했을 때 값이 크게 변하는 것을 줄이기 위해 손목과 손바닥 사이의 거리도 함께 사용하였다.

---

## 4. 슬라이더 제어

엄지와 검지의 거리를 `0 ~ 100` 범위로 변환하여 슬라이더 값으로 사용하였다.

```javascript
let target = map(
  fingerDistance / handSize,
  0.2, 1.4,
  0, 100
);
```

슬라이더가 갑자기 움직이지 않도록 `lerp()`를 이용하여 현재 값에서 목표 값으로 조금씩 이동하도록 하였다.

```javascript
sliderValue = lerp(sliderValue, target, 0.1);
```

이렇게 하여 손가락을 움직였을 때 슬라이더가 조금 더 부드럽게 움직이도록 만들었다.

---

## 5. Arduino와 Serial 통신

p5.js에서는 슬라이더 값을 `0 ~ 255` 범위의 PWM 값으로 변환하였다.

```javascript
let pwm = round(
  map(sliderValue, 0, 100, 0, 255)
);
```

변환된 값은 다음과 같은 형식으로 Arduino에 전송한다.

```text
PWM:128
```

Arduino는 `PWM:` 뒤에 있는 숫자를 읽은 뒤 `analogWrite()`를 사용하여 LED의 밝기를 변경한다.

---

## 6. 회로 연결

LED는 Arduino의 PWM 출력이 가능한 11번 핀에 연결하였다.

<img width="739" height="499" alt="xxxxxxxxxxxxxx" src="https://github.com/user-attachments/assets/b8393d36-469b-4048-a212-39cc8f18ab79" />


```text
Arduino D11
    ↓
저항
    ↓
LED (+)

LED (-)
    ↓
GND
```

---

## 7. 결과

https://github.com/user-attachments/assets/b03f0688-772e-4ff9-a413-4a27ea189256



## 8. 주요 기능

- 웹캠을 이용한 실시간 손 인식
- 엄지와 검지 사이 거리 측정
- 손가락 간격에 따른 슬라이더 자동 이동
- 마우스로 슬라이더 직접 조절 가능
- 슬라이더 값을 PWM 값으로 변환
- Web Serial을 이용한 Arduino 통신
- Arduino LED 밝기 실시간 조절

---

## 9. 고찰

처음에는 손가락 사이의 거리를 바로 슬라이더 값으로 사용했는데, 손을 카메라에 가까이하거나 멀리하면 같은 손동작이어도 값이 크게 달라지는 문제가 있었다.

그래서 손목과 손바닥 사이의 길이를 기준으로 사용하여 손가락 거리를 비율로 계산하였다. 이렇게 수정하니 손의 위치가 조금 달라져도 이전보다 일정하게 값을 얻을 수 있었다.

또 손 인식 값이 계속 조금씩 변하기 때문에 슬라이더가 떨리는 것처럼 보였다. 이를 줄이기 위해 `lerp()`를 사용하여 슬라이더가 목표 값으로 천천히 이동하도록 하였다.

이번 실습을 통해 카메라에서 얻은 손동작 정보를 단순히 화면에 표시하는 것뿐만 아니라, Serial 통신을 이용하여 실제 Arduino 장치까지 제어할 수 있다는 점이 재미있었다.
