  // 복잡한 통신은 serial-helper.js가 처리합니다.
let serial, connectButton;

let handPose, video, slider;
let hands = [];
let connections;

let sliderValue = 50;
let lastPWM = -1;

let statusText = 'Arduino를 연결해 주세요.';

async function setup() {
  createCanvas(640, 480);
  textFont('sans-serif');

  // Serial 연결
  serial = new SerialHelper(
    9600,
    receiveLine,
    message => statusText = message
  );

  connectButton = createButton('Arduino 연결 / 해제');
  connectButton.mousePressed(() => serial.toggle());

  // 슬라이더
  slider = createSlider(0, 100, 50, 1);
  slider.position(10, 20);
  slider.style('writing-mode', 'vertical-rl');
  slider.style('height', '440px');
  slider.style('width', '20px');
  slider.style('transform', 'rotate(180deg)');

  slider.input(() => {
    sliderValue = Number(slider.value());
  });

  // 카메라
  video = createCapture(VIDEO, { flipped: true });
  video.size(640, 480);
  video.hide();

  // HandPose
  handPose = await ml5.handPose({ flipped: true });
  handPose.detectStart(video, gotHands);

  connections = handPose.getConnections();
}


function draw() {
  image(video, 0, 0, width, height);

  drawSkeletons();
  drawKeyPoints();

  // 손이 인식되었을 때
  if (hands.length > 0) {
    let hand = hands[0];

    let thumb = hand.keypoints[4];   // 엄지 끝
    let index = hand.keypoints[8];   // 검지 끝
    let wrist = hand.keypoints[0];
    let palm = hand.keypoints[9];

    // 엄지와 검지 사이 거리
    let fingerDistance = dist(
      thumb.x, thumb.y,
      index.x, index.y
    );

    // 손 크기 기준
    let handSize = dist(
      wrist.x, wrist.y,
      palm.x, palm.y
    );

    // 0~100 슬라이더 값으로 변환
    let target = map(
      fingerDistance / handSize,
      0.2, 1.4,
      0, 100
    );

    target = constrain(target, 0, 100);

    // 부드럽게 이동
    sliderValue = lerp(sliderValue, target, 0.1);
    slider.value(sliderValue);

    // 엄지-검지 연결선
    stroke(255, 140, 0);
    strokeWeight(4);
    line(
      thumb.x, thumb.y,
      index.x, index.y
    );
  }

  // Arduino로 PWM 값 전송
  sendPWM();

  // 화면 표시
  fill(0, 150);
  noStroke();
  rect(390, 20, 230, 85, 10);

  fill(255);
  textAlign(CENTER);
  textSize(16);

  text(
    'Slider : ' + round(sliderValue),
    505, 50
  );

  let pwm = round(
    map(sliderValue, 0, 100, 0, 255)
  );

  text(
    'PWM : ' + pwm + ' / 255',
    505, 75
  );

  textSize(12);
  text(statusText, 505, 95);
}


// Arduino로 PWM 값 보내기
async function sendPWM() {
  let pwm = round(
    map(sliderValue, 0, 100, 0, 255)
  );

  pwm = constrain(pwm, 0, 255);

  // 값이 변했을 때만 전송
  if (pwm === lastPWM) return;

  if (await serial.send('PWM:' + pwm)) {
    lastPWM = pwm;
  }
}


// 손 인식 결과
function gotHands(results) {
  hands = results;
}


// 손가락 점
function drawKeyPoints() {
  for (let hand of hands) {
    for (let point of hand.keypoints) {
      fill(255, 0, 0);
      noStroke();
      circle(point.x, point.y, 8);
    }
  }
}


// 손 뼈대
function drawSkeletons() {
  for (let hand of hands) {
    stroke(0, 255, 0);
    strokeWeight(2);

    for (let [a, b] of connections) {
      let p1 = hand.keypoints[a];
      let p2 = hand.keypoints[b];

      if (p1 && p2) {
        line(p1.x, p1.y, p2.x, p2.y);
      }
    }
  }
}


// Arduino에서 받은 메시지
function receiveLine(line) {
  console.log('Arduino:', line);
}
