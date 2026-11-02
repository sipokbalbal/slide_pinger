# LED 밝기 조절 코드 및 설명

## 1. 아두이노 코드

파일: `led_pwm_slider/led_pwm_slider.ino`

```cpp
/*
 * 사진 회로 그대로: Arduino Uno R3(ATmega328P) 전용
 * LED: D13 / 버튼: D2-GND / 가변저항 가운데: A0
 * I2C LCD: SDA=A4, SCL=A5, VCC=5V, GND=GND
 * 웹에서 0~255 + 줄바꿈을 받습니다. 통신 속도: 9600 baud
 *
 * D13에는 일반 analogWrite PWM이 없으므로 Timer1 인터럽트로
 * 소프트웨어 PWM을 만듭니다. D9로 배선을 옮길 필요가 없습니다.
 * Timer1을 사용하는 Servo 등의 라이브러리와 함께 사용하지 마세요.
 * 라이브러리 관리자: LiquidCrystal I2C (Frank de Brabander) 설치
 */

#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <avr/interrupt.h>
#include <stdio.h>

#if !defined(__AVR_ATmega328P__) || F_CPU != 16000000UL
#error "Select Arduino Uno R3 (ATmega328P, 16 MHz)."
#endif

const byte LED_PIN = 13;
const byte BUTTON_PIN = 2;
const byte POT_PIN = A0;
const byte LCD_ADDRESS = 0x27;  // LCD가 0x3F 제품이면 0x3F로 변경
LiquidCrystal_I2C lcd(LCD_ADDRESS, 16, 2);
bool lcdFound = false;

volatile uint8_t ledBrightness = 0;
int potValue = 0;
char inputLine[16];
byte inputLength = 0;
bool lineTooLong = false;

// 16 MHz / 512 = 31,250회/초. 256단계 PWM 주파수는 약 122 Hz입니다.
// Uno의 D13은 PB5입니다. LCD 통신 중에도 LED 밝기를 유지합니다.
ISR(TIMER1_COMPA_vect) {
  static uint8_t phase = 0;
  const uint8_t value = ledBrightness;

  if (value == 0) {
    PORTB &= ~_BV(PB5);
  } else if (value == 255) {
    PORTB |= _BV(PB5);
  } else if (phase == 0) {
    PORTB |= _BV(PB5);
  } else if (phase == value) {
    PORTB &= ~_BV(PB5);
  }
  phase++;
}

void startLedPwm() {
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);

  const uint8_t savedSREG = SREG;
  cli();
  TCCR1A = 0;
  TCCR1B = 0;
  TCNT1 = 0;
  OCR1A = 511;
  TIFR1 = _BV(OCF1A);  // 남아 있는 비교 일치 플래그 지우기
  TIMSK1 = _BV(OCIE1A);
  TCCR1B = _BV(WGM12) | _BV(CS10);  // CTC 모드 / 분주 없음
  SREG = savedSREG;
}

void setBrightness(byte value) {
  ledBrightness = value;
  Serial.print("LED:");
  Serial.println(value);
}

void processLine() {
  if (inputLength == 0) return;
  unsigned int value = 0;
  for (byte i = 0; i < inputLength; i++) {
    if (inputLine[i] < '0' || inputLine[i] > '9') return;
    value = value * 10 + (inputLine[i] - '0');
    if (value > 255) return;
  }
  setBrightness((byte)value);
}

void receiveSerial() {
  while (Serial.available() > 0) {
    const char ch = Serial.read();
    if (ch == '\r') continue;
    if (ch == '\n') {
      if (!lineTooLong) processLine();
      inputLength = 0;
      lineTooLong = false;
    } else if (!lineTooLong) {
      if (inputLength < sizeof(inputLine) - 1) {
        inputLine[inputLength++] = ch;
      } else {
        lineTooLong = true;
      }
    }
  }
}

void readButton() {
  static byte lastReading = HIGH;
  static byte stableState = HIGH;
  static unsigned long changedAt = 0;
  const byte reading = digitalRead(BUTTON_PIN);

  if (reading != lastReading) changedAt = millis();
  if (millis() - changedAt >= 30 && reading != stableState) {
    stableState = reading;
    if (stableState == LOW) {
      // 버튼은 LED 끄기 기능입니다. 슬라이더를 다시 움직이면 켜집니다.
      Serial.println("BUTTON:0");
      setBrightness(0);
    }
  }
  lastReading = reading;
}

void refreshDisplay() {
  potValue = analogRead(POT_PIN);
  Serial.print("POT:");
  Serial.println(potValue);
  if (!lcdFound) return;

  char row[17];
  snprintf(row, sizeof(row), "LED:%3u / 255   ", (unsigned int)ledBrightness);
  lcd.setCursor(0, 0);
  lcd.print(row);
  snprintf(row, sizeof(row), "POT:%4d BTN:%d  ", potValue,
           digitalRead(BUTTON_PIN) == LOW ? 1 : 0);
  lcd.setCursor(0, 1);
  lcd.print(row);
}

void setup() {
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  Serial.begin(9600);
  startLedPwm();
  Wire.begin();
#if defined(WIRE_HAS_TIMEOUT)
  Wire.setWireTimeout(3000, true);
#endif

  Wire.beginTransmission(LCD_ADDRESS);
  lcdFound = Wire.endTransmission() == 0;
  if (lcdFound) {
    lcd.init();
    lcd.backlight();
    lcd.clear();
  }

  Serial.println("READY");
  setBrightness(0);
  refreshDisplay();
}

void loop() {
  receiveSerial();
  readButton();

  static unsigned long lastRefresh = 0;
  if (millis() - lastRefresh >= 200) {
    lastRefresh = millis();
    refreshDisplay();
  }
}
```

### 코드 설명

- **D13 LED:** 웹에서 받은 0~255 값으로 밝기를 조절합니다. 0은 꺼짐, 255는 최대 밝기입니다.
- **소프트웨어 PWM:** Uno R3의 D13은 일반 PWM 핀이 아니므로 Timer1 인터럽트로 빠르게 켜고 끄는 비율을 조절합니다. 주파수는 약 122Hz입니다.
- **시리얼 수신:** 줄바꿈까지 받은 뒤 숫자를 확인합니다. 범위를 벗어나거나 숫자가 아닌 입력은 무시합니다.
- **D2 버튼:** 버튼을 누르면 LED 밝기를 0으로 만듭니다. `INPUT_PULLUP`을 사용하므로 버튼은 D2와 GND 사이에 연결합니다. 접점 떨림을 줄이기 위해 30ms 동안 상태를 확인합니다.
- **A0 가변저항:** 값을 읽어 웹과 LCD에 표시합니다. LED 밝기를 직접 바꾸지는 않습니다.
- **I2C LCD:** 첫째 줄에 LED 밝기, 둘째 줄에 가변저항 값과 버튼 상태를 표시합니다. SDA는 A4, SCL은 A5입니다.
- **200ms 갱신:** 가변저항 전송과 LCD 표시를 200ms마다 갱신합니다.

이 코드는 **Arduino Uno R3(ATmega328P, 16MHz) 전용**이며, `LiquidCrystal I2C`(Frank de Brabander) 라이브러리가 필요합니다. LCD 주소는 기본 `0x27`이고 제품에 따라 `0x3F` 등 실제 주소로 바꿔야 합니다. Timer1을 사용하는 Servo 등의 라이브러리와 함께 사용하지 않습니다. LED 직렬 저항을 유지합니다.

## 2. p5.js 코드

파일: `p5/sketch.js`

```javascript
// 슬라이더의 0~255 값을 아두이노로 보냅니다.
let serial, brightnessSlider, connectButton;
let statusText = 'USB 케이블을 연결한 뒤 Arduino 연결을 눌러 주세요.';
let actualBrightness = null;
let potValue = null;
let connecting = false;
let sendTimer = null;

function setup() {
  createCanvas(600, 320).parent('canvas-container');
  textFont('sans-serif');

  serial = new SerialHelper(9600, receiveLine, message => statusText = message);

  brightnessSlider = createSlider(0, 255, 0, 1);
  brightnessSlider.parent('slider-container');
  brightnessSlider.attribute('aria-label', 'LED 밝기 0부터 255');
  brightnessSlider.input(() => {
    // 드래그 중에도 약 40ms 간격으로 최신 값을 보냅니다.
    if (sendTimer === null) {
      sendTimer = setTimeout(() => {
        sendTimer = null;
        sendBrightness();
      }, 40);
    }
  });
  brightnessSlider.changed(sendBrightness);

  connectButton = createButton('Arduino 연결');
  connectButton.parent('button-container');
  connectButton.mousePressed(toggleConnection);

  if (!window.isSecureContext || !('serial' in navigator)) {
    statusText = 'PC용 Chrome/Edge에서 HTTPS 주소로 열어 주세요.';
    connectButton.attribute('disabled', '');
  }
}

function draw() {
  const value = Number(brightnessSlider.value());
  background(244, 247, 251);
  noStroke();
  textAlign(CENTER, CENTER);

  fill(25, 40, 60);
  textSize(22);
  text('p5.js 슬라이더 · LED 밝기 조절', width / 2, 34);

  // 화면의 원은 슬라이더 값에 대한 미리보기입니다.
  fill(lerpColor(color('#475569'), color('#fbbf24'), value / 255));
  circle(145, 154, 116);
  fill(value > 100 ? '#422006' : '#ffffff');
  textSize(25);
  text(round(value / 255 * 100) + '%', 145, 154);

  fill(25, 40, 60);
  textSize(17);
  text('슬라이더 값', 405, 110);
  textSize(38);
  text(value + ' / 255', 405, 154);
  textSize(14);
  const response = serial.connected && actualBrightness !== null
    ? actualBrightness + ' / 255' : '확인 전';
  text('아두이노 응답: ' + response, 405, 201);

  textSize(14);
  text('LED 미리보기', 145, 239);
  text('가변저항 A0: ' + (potValue === null ? '확인 전' : potValue + ' / 1023'), 405, 239);
  fill(serial.connected ? '#15803d' : '#64748b');
  textSize(13);
  text(serial.connected ? '● 연결됨 · D13 소프트웨어 PWM · 9600 baud' : '● 연결 안 됨 · 화면 미리보기만 동작', 300, 287);

  document.getElementById('status').textContent = statusText;
  if (!connecting) {
    connectButton.html(serial.connected ? 'Arduino 연결 해제' : 'Arduino 연결');
  }
}

async function toggleConnection() {
  if (connecting) return;
  connecting = true;
  connectButton.attribute('disabled', '');
  connectButton.html('처리 중…');

  try {
    if (serial.connected) {
      // 정상적으로 연결을 해제할 때 LED를 먼저 끕니다.
      await serial.send('0');
      await serial.disconnect();
      actualBrightness = null;
      brightnessSlider.value(0);
    } else {
      actualBrightness = null;
      if (await serial.connect()) await sendBrightness();
    }
  } finally {
    connecting = false;
    connectButton.removeAttribute('disabled');
  }
}

async function sendBrightness() {
  if (serial && serial.connected) {
    await serial.send(String(brightnessSlider.value()));
  }
}

function receiveLine(line) {
  const ledMatch = line.match(/^LED:(\d+)$/);
  const potMatch = line.match(/^POT:(\d+)$/);

  if (ledMatch) {
    const value = Number(ledMatch[1]);
    if (value >= 0 && value <= 255) actualBrightness = value;
  } else if (potMatch) {
    const value = Number(potMatch[1]);
    if (value >= 0 && value <= 1023) potValue = value;
  } else if (line === 'BUTTON:0') {
    // 회로의 버튼을 누르면 화면 슬라이더도 0으로 맞춥니다.
    if (sendTimer !== null) clearTimeout(sendTimer);
    sendTimer = null;
    brightnessSlider.value(0);
    actualBrightness = 0;
    // 대기 중인 이전 전송이 있더라도 마지막 명령을 0으로 만듭니다.
    sendBrightness();
    statusText = '회로의 버튼을 눌러 LED를 껐습니다. 슬라이더로 다시 조절하세요.';
  }
}
```

### 코드 설명

- **`createSlider(0, 255, 0, 1)`**: 0~255 범위의 슬라이더를 만듭니다. 처음 값은 0, 간격은 1입니다.
- **`sendBrightness()`**: 슬라이더 값을 문자열로 바꾸어 아두이노에 전송합니다.
- **전송 간격:** 드래그할 때 약 40ms마다 최신 값을 전송해 통신량을 줄입니다.
- **`receiveLine()`**: `LED:값`으로 LED 응답을, `POT:값`으로 가변저항 값을 받습니다.
- **버튼 연동:** `BUTTON:0`을 받으면 웹 슬라이더도 0으로 되돌립니다.
- **화면 표시:** 원의 색과 숫자는 슬라이더 미리보기이고, '아두이노 응답'은 보드가 되돌려 준 값입니다. 실제 밝기를 센서로 측정한 값은 아닙니다.
- **연결 해제:** 정상 해제 시 LED를 끄는 0 명령을 먼저 보냅니다. 통신이 갑자기 끊기면 전원이 남아 있는 동안 마지막 밝기가 유지될 수 있습니다.

이 파일은 아래의 `serial-helper.js`와 `index.html`을 함께 사용합니다.

## 3. 시리얼 통신 코드

파일: `p5/serial-helper.js`

```javascript
// Web Serial 연결, 줄 단위 수신, 연결 해제를 담당합니다.
class SerialHelper {
  constructor(baudRate, onLine, onStatus) {
    this.baudRate = baudRate;
    this.onLine = onLine;
    this.onStatus = onStatus;
    this.port = null;
    this.reader = null;
    this.writer = null;
    this.readTask = null;
    this.writeQueue = Promise.resolve();
    this.connected = false;
    this.disconnecting = false;
  }

  async connect() {
    if (this.connected) return true;
    if (!window.isSecureContext || !('serial' in navigator)) {
      this.onStatus('PC용 Chrome/Edge에서 HTTPS 주소로 열어 주세요.');
      return false;
    }

    // 이전 연결이 비정상 종료되었다면 먼저 정리합니다.
    if (this.port) await this.disconnect();

    try {
      this.onStatus('목록에서 Arduino의 USB 포트를 선택해 주세요.');
      this.port = await navigator.serial.requestPort();
      await this.port.open({ baudRate: this.baudRate });
      this.writer = this.port.writable.getWriter();
      this.reader = this.port.readable.getReader();
      this.readTask = this.readLines();

      // Uno는 포트를 열면 재시작할 수 있어 잠시 기다립니다.
      this.onStatus('Arduino가 준비되는 중입니다. 잠시 기다려 주세요.');
      await new Promise(resolve => setTimeout(resolve, 2000));
      if (!this.port || !this.reader || !this.writer) {
        throw new Error('USB 연결을 확인해 주세요.');
      }

      this.connected = true;
      this.onStatus('연결되었습니다. 슬라이더로 밝기를 조절하세요.');
      return true;
    } catch (error) {
      await this.disconnect();
      this.onStatus(error.name === 'NotFoundError'
        ? '포트 선택을 취소했습니다. 연결 버튼을 다시 눌러 주세요.'
        : '연결 실패: ' + error.message + ' (시리얼 모니터를 닫아 주세요.)');
      return false;
    }
  }

  async send(line) {
    if (!this.connected || !this.writer) return false;

    // 여러 명령이 동시에 들어와도 순서대로 전송합니다.
    this.writeQueue = this.writeQueue.catch(() => false).then(async () => {
      if (!this.connected || !this.writer) return false;
      await this.writer.write(new TextEncoder().encode(String(line) + '\n'));
      return true;
    });

    try {
      return await this.writeQueue;
    } catch (error) {
      this.onStatus('전송 실패: ' + error.message);
      return false;
    }
  }

  async readLines() {
    const decoder = new TextDecoder();
    let buffer = '';
    try {
      while (this.reader && !this.disconnecting) {
        const { value, done } = await this.reader.read();
        if (done) break;
        buffer += decoder.decode(value, { stream: true });
        let newline;
        while ((newline = buffer.indexOf('\n')) !== -1) {
          const line = buffer.slice(0, newline).trim();
          buffer = buffer.slice(newline + 1);
          if (line) this.onLine(line);
        }
        // 비정상 입력으로 버퍼가 무한히 커지는 것을 방지합니다.
        if (buffer.length > 512) buffer = '';
      }
    } catch (error) {
      if (!this.disconnecting) this.onStatus('수신 중 연결이 끊겼습니다.');
    } finally {
      if (this.reader) {
        this.reader.releaseLock();
        this.reader = null;
      }
      if (!this.disconnecting) {
        this.connected = false;
        // 읽기 작업이 끝난 다음 포트와 쓰기 잠금을 정리합니다.
        setTimeout(async () => {
          if (this.port && !this.disconnecting) {
            await this.disconnect();
            this.onStatus('연결이 끊겼습니다. USB를 확인한 뒤 다시 연결하세요.');
          }
        }, 0);
      }
    }
  }

  async disconnect() {
    if (this.disconnecting) return;
    this.disconnecting = true;
    this.connected = false;

    await this.writeQueue.catch(() => false);
    if (this.reader) {
      try { await this.reader.cancel(); } catch (_) {}
    }
    if (this.readTask) {
      try { await this.readTask; } catch (_) {}
      this.readTask = null;
    }
    if (this.writer) {
      try { this.writer.releaseLock(); } catch (_) {}
      this.writer = null;
    }
    if (this.port) {
      try { await this.port.close(); } catch (_) {}
      this.port = null;
    }
    this.disconnecting = false;
    this.onStatus('연결이 해제되었습니다.');
  }
}
```

### 코드 설명

- **`connect()`**: 브라우저에서 사용자가 USB 포트를 선택하면 9600 baud로 연결합니다. Uno가 재시작할 수 있어 약 2초 기다립니다.
- **`send()`**: 명령 끝에 줄바꿈을 붙여 순서대로 보냅니다.
- **`readLines()`**: 나뉘어 들어온 데이터를 모아 줄바꿈 단위로 처리하고 p5.js에 전달합니다.
- **`disconnect()`**: 읽기와 쓰기 잠금을 해제한 뒤 포트를 닫습니다.

Web Serial을 지원하는 **PC용 Chrome 또는 Edge**에서 HTTPS 주소로 실행합니다. 연결 전 Arduino IDE의 시리얼 모니터와 시리얼 플로터를 닫습니다.

## 4. 웹 화면 코드

파일: `p5/index.html`

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>p5.js LED 밝기 조절</title>
  <script src="https://cdn.jsdelivr.net/npm/p5@1.11.3/lib/p5.min.js"></script>
  <style>
    * { box-sizing: border-box; }
    body { margin: 0; padding: 32px 16px; background: #e8eef5; color: #19283c;
      font-family: system-ui, -apple-system, "Malgun Gothic", sans-serif; }
    main { max-width: 650px; margin: auto; padding: 24px; background: white;
      border-radius: 18px; box-shadow: 0 8px 32px #19283c12; }
    h1 { margin: 0 0 8px; font-size: 22px; }
    p { line-height: 1.7; }
    .intro { margin: 0 0 20px; color: #526277; font-size: 14px; }
    canvas { display: block; max-width: 100%; height: auto !important; border-radius: 12px; }
    #slider-container { padding: 24px 8px 8px; }
    input[type="range"] { width: 100%; accent-color: #2563eb; cursor: pointer; }
    .range-labels { display: flex; justify-content: space-between; padding: 0 8px;
      color: #526277; font-size: 13px; }
    #button-container { margin-top: 24px; }
    button { width: 100%; padding: 13px; border: 0; border-radius: 10px;
      background: #2563eb; color: white; font-size: 16px; cursor: pointer; }
    button:disabled { background: #94a3b8; cursor: wait; }
    #status { min-height: 48px; margin: 14px 0; font-size: 14px; overflow-wrap: anywhere; }
    .note { margin: 0; padding-top: 14px; border-top: 1px solid #e2e8f0;
      color: #526277; font-size: 12px; }
  </style>
</head>
<body>
  <main>
    <h1>Arduino LED 밝기 조절</h1>
    <p class="intro">슬라이더를 움직이면 USB로 연결된 LED의 밝기가 바뀝니다.</p>
    <div id="canvas-container"></div>
    <div id="slider-container"></div>
    <div class="range-labels"><span>0 · 꺼짐</span><span>255 · 최대 밝기</span></div>
    <div id="button-container"></div>
    <p id="status" role="status" aria-live="polite">프로그램을 불러오는 중입니다.</p>
    <p class="note">PC용 Chrome/Edge · Arduino Uno R3 · LED 핀 D13<br>
      Arduino IDE의 시리얼 모니터와 시리얼 플로터를 닫은 뒤 연결하세요.<br>
      회로의 D2 버튼: LED 끄기 · A0 가변저항: 값 표시<br>
      p5.js를 불러오기 위해 인터넷 연결이 필요합니다.</p>
    <noscript>이 프로그램을 실행하려면 JavaScript를 켜 주세요.</noscript>
  </main>
  <script src="serial-helper.js"></script>
  <script src="sketch.js"></script>
</body>
</html>
```

### 코드 설명

- p5.js 라이브러리를 불러옵니다.
- 캔버스, 슬라이더, 연결 버튼, 상태 문구를 표시할 공간을 만듭니다.
- `serial-helper.js`를 먼저 불러온 뒤 `sketch.js`를 실행합니다.
- CSS로 화면 크기, 색상, 버튼과 슬라이더 모양을 지정합니다.

README에 코드를 적는 것만으로 프로그램이 실행되지는 않습니다. 실행하려면 각 코드 블록을 표시된 경로의 실제 파일에도 저장해야 합니다. 실제 보드와 LCD에서의 동작 확인은 별도로 필요합니다.

