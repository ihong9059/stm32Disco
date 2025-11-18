# STM32H745I-DISCO Demonstration Firmware

## 개요

STM32H745I-DISCO의 공식 데모 펌웨어는 보드의 듀얼 코어 성능과 아날로그/디지털 기능을 보여주는 종합 애플리케이션입니다. 오실로스코프/신호 발생기, CoreMark 벤치마크, 시스템 정보를 포함한 3가지 주요 데모로 구성되어 있습니다.

## 주요 애플리케이션

### 1. Oscilloscope / Signal Generator (오실로스코프 / 신호 발생기)
듀얼 코어를 활용한 실시간 아날로그 신호 처리 데모

### 2. EEMBC CoreMark
CM7과 CM4의 성능 벤치마크

### 3. System Information
보드 정보 및 펌웨어 버전 표시

## 시스템 아키텍처

### 듀얼 코어 역할 분담
```
┌──────────────────────────────────────────────────┐
│                Demonstration                     │
├──────────────────────┬───────────────────────────┤
│    Cortex-M7         │       Cortex-M4           │
│    (400 MHz)         │       (200 MHz)           │
├──────────────────────┼───────────────────────────┤
│                      │                           │
│  - User Interface    │  - Digital Oscilloscope   │
│  - LCD/Touch         │  - ADC Sampling (4 MSPS)  │
│  - Signal Generator  │  - FFT Analysis           │
│  - DAC Control       │  - UART Communication     │
│  - Menu System       │  - Data Streaming         │
│                      │                           │
└──────────────────────┴───────────────────────────┘
```

## 1. Oscilloscope / Signal Generator

### 개요
CM7은 신호 발생기로 동작하고, CM4는 디지털 오실로스코프로 동작하는 듀얼 코어 아날로그 애플리케이션입니다.

### CM7: Signal Generator (신호 발생기)

#### 지원 파형
1. **DC**: 직류 전압 출력
2. **Square**: 구형파 (50% duty cycle)
3. **Sinus**: 정현파
4. **Triangle**: 삼각파
5. **Escalator**: 계단형 파형
6. **Noise**: 랜덤 노이즈

#### 설정 가능 파라미터
```
주파수 (Frequency):
  - 범위: 5 Hz ~ 130 kHz
  - 조정: 터치스크린 슬라이더

진폭 (Amplitude):
  - 범위: 0.1V ~ 3.3V
  - 조정: 터치스크린 슬라이더

오프셋 (Offset):
  - 범위: 0V ~ 3.3V
  - DC 레벨 조정
```

#### DAC 구현
```c
// DAC 설정
#define DAC_CHANNEL  DAC_CHANNEL_1  // PA4 핀

// 신호 발생기 초기화
void SignalGenerator_Init(void)
{
  DAC_HandleTypeDef hdac;

  // DAC 초기화
  hdac.Instance = DAC1;
  HAL_DAC_Init(&hdac);

  // DAC 채널 설정
  DAC_ChannelConfTypeDef sConfig;
  sConfig.DAC_Trigger = DAC_TRIGGER_T6_TRGO;  // Timer 6 트리거
  sConfig.DAC_OutputBuffer = DAC_OUTPUTBUFFER_ENABLE;
  HAL_DAC_ConfigChannel(&hdac, &sConfig, DAC_CHANNEL);

  // DMA로 파형 데이터 출력
  HAL_DAC_Start_DMA(&hdac, DAC_CHANNEL,
                    (uint32_t*)waveform_buffer,
                    WAVEFORM_SAMPLES,
                    DAC_ALIGN_12B_R);
}

// 정현파 생성
void Generate_Sine_Wave(uint32_t frequency, float amplitude)
{
  uint32_t timer_period = (SystemCoreClock / frequency) / WAVEFORM_SAMPLES;

  for (int i = 0; i < WAVEFORM_SAMPLES; i++)
  {
    float angle = (2.0f * M_PI * i) / WAVEFORM_SAMPLES;
    float value = (sinf(angle) + 1.0f) * 2048.0f * amplitude / 3.3f;
    waveform_buffer[i] = (uint32_t)value;
  }

  // 타이머 주파수 업데이트
  __HAL_TIM_SET_AUTORELOAD(&htim6, timer_period - 1);
}

// 구형파 생성
void Generate_Square_Wave(uint32_t frequency, float amplitude)
{
  uint32_t timer_period = (SystemCoreClock / frequency) / WAVEFORM_SAMPLES;
  uint32_t half_samples = WAVEFORM_SAMPLES / 2;

  for (int i = 0; i < WAVEFORM_SAMPLES; i++)
  {
    if (i < half_samples)
      waveform_buffer[i] = (uint32_t)(4095 * amplitude / 3.3f);
    else
      waveform_buffer[i] = 0;
  }

  __HAL_TIM_SET_AUTORELOAD(&htim6, timer_period - 1);
}

// 삼각파 생성
void Generate_Triangle_Wave(uint32_t frequency, float amplitude)
{
  uint32_t timer_period = (SystemCoreClock / frequency) / WAVEFORM_SAMPLES;
  uint32_t half_samples = WAVEFORM_SAMPLES / 2;

  for (int i = 0; i < WAVEFORM_SAMPLES; i++)
  {
    float value;
    if (i < half_samples)
      value = (float)i / half_samples;  // 상승
    else
      value = 2.0f - (float)i / half_samples;  // 하강

    waveform_buffer[i] = (uint32_t)(value * 4095 * amplitude / 3.3f);
  }

  __HAL_TIM_SET_AUTORELOAD(&htim6, timer_period - 1);
}
```

### CM4: Digital Oscilloscope (디지털 오실로스코프)

#### 사양
```
샘플링 속도 (Sampling Rate):
  - 최대: 4 MSPS (Mega Samples Per Second)
  - ADC 클럭: 36 MHz
  - 해상도: 14-bit

입력 채널:
  - 채널 1: PA0 (ADC1_INP0)
  - 입력 범위: 0 ~ 3.3V

데이터 전송:
  - UART: 115200 baud, 8N1
  - PC GUI와 실시간 통신
```

#### ADC 구현
```c
#define ADC_BUFFER_SIZE  1024

uint16_t adc_buffer[ADC_BUFFER_SIZE];

void Oscilloscope_Init(void)
{
  ADC_HandleTypeDef hadc1;

  // ADC1 초기화
  hadc1.Instance = ADC1;
  hadc1.Init.ClockPrescaler = ADC_CLOCK_ASYNC_DIV2;
  hadc1.Init.Resolution = ADC_RESOLUTION_14B;
  hadc1.Init.ScanConvMode = ADC_SCAN_DISABLE;
  hadc1.Init.ContinuousConvMode = ENABLE;
  hadc1.Init.DiscontinuousConvMode = DISABLE;
  hadc1.Init.ExternalTrigConv = ADC_SOFTWARE_START;
  hadc1.Init.DataAlign = ADC_DATAALIGN_RIGHT;
  hadc1.Init.NbrOfConversion = 1;
  hadc1.Init.DMAContinuousRequests = ENABLE;
  hadc1.Init.EOCSelection = ADC_EOC_SINGLE_CONV;
  hadc1.Init.Overrun = ADC_OVR_DATA_OVERWRITTEN;

  HAL_ADC_Init(&hadc1);

  // ADC 채널 설정
  ADC_ChannelConfTypeDef sConfig;
  sConfig.Channel = ADC_CHANNEL_0;  // PA0
  sConfig.Rank = ADC_REGULAR_RANK_1;
  sConfig.SamplingTime = ADC_SAMPLETIME_1CYCLE_5;
  sConfig.SingleDiff = ADC_SINGLE_ENDED;
  sConfig.OffsetNumber = ADC_OFFSET_NONE;
  sConfig.Offset = 0;

  HAL_ADC_ConfigChannel(&hadc1, &sConfig);

  // ADC 캘리브레이션
  HAL_ADCEx_Calibration_Start(&hadc1, ADC_CALIB_OFFSET, ADC_SINGLE_ENDED);

  // DMA로 연속 샘플링 시작
  HAL_ADC_Start_DMA(&hadc1, (uint32_t*)adc_buffer, ADC_BUFFER_SIZE);
}

// DMA 완료 콜백
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)
{
  // 버퍼가 꽉 찼음 - UART로 PC에 전송
  Send_Data_To_PC(adc_buffer, ADC_BUFFER_SIZE);
}
```

#### UART 통신 프로토콜
```c
// PC GUI와 통신 프로토콜
typedef struct {
  uint8_t  header[4];      // "OSCI"
  uint32_t sample_rate;    // Hz
  uint16_t num_samples;    // 샘플 개수
  uint16_t resolution;     // 비트 (14)
  uint16_t samples[];      // ADC 데이터
} OsciData_t;

void Send_Data_To_PC(uint16_t *samples, uint16_t count)
{
  OsciData_t packet;

  packet.header[0] = 'O';
  packet.header[1] = 'S';
  packet.header[2] = 'C';
  packet.header[3] = 'I';
  packet.sample_rate = 4000000;  // 4 MSPS
  packet.num_samples = count;
  packet.resolution = 14;

  // 헤더 전송
  HAL_UART_Transmit(&huart3, (uint8_t*)&packet, 12, HAL_MAX_DELAY);

  // 샘플 데이터 전송
  HAL_UART_Transmit(&huart3, (uint8_t*)samples, count * 2, HAL_MAX_DELAY);
}
```

### PC GUI 소프트웨어

데모에는 PC용 GUI 소프트웨어가 포함되어 있습니다.

#### 위치
```
Demonstrations/PC_Software/
├── STM32H745_Oscilloscope.exe  (Windows)
├── README.txt
└── Source/                      (소스 코드)
```

#### 기능
- 실시간 파형 디스플레이
- 트리거 설정 (레벨, 에지)
- 시간축/전압축 스케일 조정
- 데이터 저장 (CSV, 이미지)
- FFT 분석 (주파수 도메인)

#### 연결 방법
1. STM32H745I-DISCO를 USB로 PC에 연결
2. Virtual COM Port 드라이버 설치 (ST-Link)
3. GUI 실행
4. COM 포트 선택 (115200 baud)
5. Connect 클릭

### Auto-Test 모드

신호 발생기와 오실로스코프를 직접 연결하여 자동 테스트 가능:

```
하드웨어 연결:
  PA4 (DAC OUT) ───────► PA0 (ADC IN)
```

이렇게 연결하면 생성된 신호를 바로 측정하여 파형 확인 가능

## 2. EEMBC CoreMark 벤치마크

### 개요
CoreMark는 임베디드 시스템의 CPU 성능을 측정하는 표준 벤치마크입니다.

### 벤치마크 결과 표시
```
┌─────────────────────────────┐
│  CoreMark Benchmark         │
├─────────────────────────────┤
│                             │
│  Cortex-M7 @ 400MHz:        │
│    Score: 2020 CoreMark     │
│    CoreMark/MHz: 5.05       │
│                             │
│  Cortex-M4 @ 200MHz:        │
│    Score: 600 CoreMark      │
│    CoreMark/MHz: 3.00       │
│                             │
└─────────────────────────────┘
```

### 실행 방법
- 메인 메뉴에서 "CoreMark" 선택
- 벤치마크 자동 실행 (약 10초 소요)
- 결과 화면에 점수 표시

## 3. System Information

### 표시 정보
```
┌─────────────────────────────┐
│  System Information         │
├─────────────────────────────┤
│  Board: STM32H745I-DISCO    │
│  MCU:   STM32H745XIH6       │
│                             │
│  Firmware Version: 1.0.0    │
│  Build Date: 2024-01-15     │
│                             │
│  CM7 Core: 400 MHz          │
│  CM4 Core: 200 MHz          │
│                             │
│  Flash:  2048 KB            │
│  RAM:    1024 KB            │
│  SDRAM:  8192 KB            │
│                             │
│  LCD:    480 x 272          │
│  Touch:  Enabled            │
└─────────────────────────────┘
```

## 메뉴 시스템

### 메인 메뉴
```c
typedef enum {
  MENU_OSCILLOSCOPE = 0,
  MENU_COREMARK,
  MENU_SYSINFO,
  MENU_STANDBY
} MenuState_t;

void Display_Main_Menu(void)
{
  // 배경 이미지 표시
  BSP_LCD_DrawBitmap(0, 0, 0, (uint8_t*)background_image);

  // 메뉴 항목
  const char *menu_items[] = {
    "Oscilloscope / Signal Generator",
    "CoreMark Benchmark",
    "System Information",
    "Enter Standby Mode"
  };

  for (int i = 0; i < 4; i++)
  {
    Draw_Menu_Button(50, 60 + i * 50, 380, 40, menu_items[i]);
  }
}

void Process_Touch_Menu(TS_State_t *ts)
{
  for (int i = 0; i < 4; i++)
  {
    if (Touch_In_Region(ts->TouchX[0], ts->TouchY[0],
                        50, 60 + i * 50, 380, 40))
    {
      switch (i)
      {
        case MENU_OSCILLOSCOPE:
          Start_Oscilloscope_App();
          break;
        case MENU_COREMARK:
          Start_CoreMark_Benchmark();
          break;
        case MENU_SYSINFO:
          Display_System_Info();
          break;
        case MENU_STANDBY:
          Enter_Standby_Mode();
          break;
      }
    }
  }
}
```

## Standby 모드

### 저전력 모드 진입
```c
void Enter_Standby_Mode(void)
{
  // 사용자 확인
  Display_Confirmation("Enter Standby Mode?");

  if (Wait_For_Confirmation())
  {
    // 주변장치 비활성화
    BSP_LCD_DisplayOff(0);
    BSP_LED_DeInit(LED1);
    BSP_LED_DeInit(LED2);

    // Wakeup 핀 설정 (Tamper 버튼)
    HAL_PWR_EnableWakeUpPin(PWR_WAKEUP_PIN1);

    // Standby 모드 진입
    HAL_PWR_EnterSTANDBYMode();
  }
}
```

### Wakeup
- Tamper/Wakeup 버튼 누름
- RTC 알람 (설정 시)
- 외부 리셋

## 빌드 및 프로그래밍

### 바이너리 파일
미리 빌드된 바이너리 파일 사용 가능:

```
Demonstrations/Binaries/
├── STM32H745I_DISCO_Demo_V1.0.0.hex
└── README.txt
```

### STM32CubeProgrammer로 플래시
```bash
# HEX 파일 프로그램
STM32_Programmer_CLI -c port=SWD -w STM32H745I_DISCO_Demo_V1.0.0.hex -v -rst
```

### 소스에서 빌드

#### STM32CubeIDE
1. 프로젝트 임포트
   - `Demonstrations/STM32CubeIDE/CM7`
   - `Demonstrations/STM32CubeIDE/CM4`

2. 빌드 순서
   - CM4 프로젝트 먼저 빌드
   - CM7 프로젝트 빌드

3. Debug Configuration
   - Multi-core debug 활성화
   - CM7, CM4 동시 디버그 가능

#### MDK-ARM / IAR
해당 폴더의 프로젝트 파일 사용

## 메모리 맵

### Flash 레이아웃
```
0x08000000  ┌─────────────────────┐
            │  CM7 Code           │
            │  - Main Menu        │
            │  - Signal Gen       │
            │  - GUI              │
0x08040000  ├─────────────────────┤
            │  CM4 Code           │
            │  - Oscilloscope     │
            │  - CoreMark         │
0x08080000  ├─────────────────────┤
            │  Resources          │
            │  - Images           │
            │  - Fonts            │
0x08100000  └─────────────────────┘
```

### RAM 사용
```
ITCM:    CM7 코드 캐시
DTCM:    CM7 데이터 (스택, 힙)
AXI:     공유 버퍼
D2 SRAM: CM4 데이터
D3 SRAM: 코어 간 통신
SDRAM:   LCD 프레임버퍼, 대용량 버퍼
```

## 성능 최적화

### CM7 최적화
- I-Cache, D-Cache 활성화
- ITCM에 크리티컬 코드 배치
- DTCM에 자주 사용하는 데이터 배치

### CM4 최적화
- ADC DMA를 D2 SRAM에 직접 저장
- 인터럽트 우선순위 최적화
- UART DMA 사용

## 트러블슈팅

### 오실로스코프가 동작하지 않는 경우
1. CM4가 정상 부팅되었는지 확인
2. UART 연결 확인 (Virtual COM Port)
3. PC GUI와 보드 연결 확인
4. ADC/DAC 핀 연결 확인

### CoreMark 점수가 낮은 경우
1. 클럭 설정 확인 (CM7: 400MHz, CM4: 200MHz)
2. 캐시 활성화 확인
3. Flash wait state 설정 확인

### LCD 터치가 반응하지 않는 경우
1. 터치 캘리브레이션 실행
2. I2C 버스 확인
3. 터치 컨트롤러 초기화 확인

## PC GUI 사용법

### 연결
1. USB 케이블로 보드와 PC 연결
2. ST-Link Virtual COM Port 드라이버 설치
3. 장치 관리자에서 COM 포트 번호 확인
4. GUI에서 해당 포트 선택 후 Connect

### 오실로스코프 제어
- **Trigger**: Level, Edge 설정
- **Timebase**: μs ~ s 범위 조정
- **Voltage**: mV ~ V 범위 조정
- **Run/Stop**: 샘플링 시작/중지
- **Single**: 단일 트리거 캡처
- **Save**: 데이터 저장

### FFT 분석
- FFT 버튼 클릭
- 주파수 스펙트럼 표시
- 피크 주파수 자동 감지
- THD (Total Harmonic Distortion) 계산

## 추가 리소스

### 문서
- UM2411: STM32H745I-DISCO User Manual
- AN5361: Dual-core STM32H7 Getting Started
- UM2609: STM32CubeH7 HAL Description

### 다운로드
- 최신 펌웨어: st.com
- PC GUI 업데이트: st.com
- 소스 코드: STM32CubeH7 패키지

## 라이선스

- 데모 펌웨어: ST SLA0044 라이선스
- CoreMark: EEMBC 라이선스
- PC GUI: ST 소프트웨어 라이선스

---

**이 데모 펌웨어는 STM32H745I-DISCO 보드의 모든 기능을 체험할 수 있는 최고의 출발점입니다!**
