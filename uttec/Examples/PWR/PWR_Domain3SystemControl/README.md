# PWR_Domain3SystemControl - D3 도메인 자율 동작

## 개요

이 예제는 STM32H745의 D3 도메인(Autonomous Domain)을 독립적으로 동작시키는 방법을 보여줍니다. D1과 D2 도메인이 STANDBY 모드에 있는 동안에도 D3 도메인만 활성 상태로 유지하여 초저전력으로 백그라운드 작업을 수행할 수 있습니다. LPTIM, LPUART, BDMA 등을 사용하여 주변장치를 자율적으로 제어하는 배터리 구동 장치에 이상적입니다.

## 하드웨어 요구사항

- **STM32H745I-DISCO 보드**
- **LED1**: PI12 (주황색) - D1 도메인 활성 상태
- **LED2**: PI13 (초록색) - D3 도메인 활성 상태
- **외부 센서**: I2C 또는 SPI (선택 사항)
- **전류계**: 전력 소비 측정용
- **오실로스코프**: LPTIM 출력 확인용 (선택 사항)

## STM32H745 도메인 아키텍처

### 3개 전력 도메인

```
┌─────────────────────────────────────────────────────────────┐
│                    STM32H745 Power Domains                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────┐         │
│  │         D1 Domain (CPU Domain)                 │         │
│  ├────────────────────────────────────────────────┤         │
│  │  • Cortex-M7 @ 480 MHz                         │         │
│  │  • L1 Cache (16KB I + 16KB D)                  │         │
│  │  • AXI SRAM (512KB)                            │         │
│  │  • DTCM (128KB), ITCM (64KB)                   │         │
│  │  • DMA1, DMA2, MDMA                            │         │
│  │  • Ethernet, USB OTG HS, DCMI                  │         │
│  │  • TIM1, TIM8                                  │         │
│  │                                                │         │
│  │  전력 모드: RUN / STOP / STANDBY               │         │
│  └────────────────────────────────────────────────┘         │
│                                                              │
│  ┌────────────────────────────────────────────────┐         │
│  │         D2 Domain (Peripheral Domain)          │         │
│  ├────────────────────────────────────────────────┤         │
│  │  • Cortex-M4 @ 240 MHz                         │         │
│  │  • AHB SRAM1, SRAM2, SRAM3 (288KB)            │         │
│  │  • ADC1, ADC2, DAC1                            │         │
│  │  • TIM2-7, TIM12-17                            │         │
│  │  • SPI1-5, I2C1-3                              │         │
│  │  • USART1-6, UART7-8                           │         │
│  │  • BDMA (D2 peripherals)                       │         │
│  │                                                │         │
│  │  전력 모드: RUN / STOP / STANDBY               │         │
│  └────────────────────────────────────────────────┘         │
│                                                              │
│  ┌────────────────────────────────────────────────┐         │
│  │      D3 Domain (Autonomous Domain) ★          │         │
│  ├────────────────────────────────────────────────┤         │
│  │  • SRD SRAM (64KB) - Backup domain             │         │
│  │  • RTC (Real-Time Clock)                       │         │
│  │  • IWDG (Independent Watchdog)                 │         │
│  │  • LPTIM1-5 (Low-Power Timers)                 │         │
│  │  • LPUART1 (Low-Power UART)                    │         │
│  │  • I2C4 (Autonomous I2C)                       │         │
│  │  • SPI6 (Autonomous SPI)                       │         │
│  │  • BDMA (D3 peripherals)                       │         │
│  │  • COMParator, OpAmps                          │         │
│  │  • SAI4 (Serial Audio Interface)               │         │
│  │  • ADC3 (16-bit ADC)                           │         │
│  │                                                │         │
│  │  전력 모드: RUN / STOP / STANDBY               │         │
│  │  ★ D1/D2 OFF 상태에서도 독립 동작 가능        │         │
│  └────────────────────────────────────────────────┘         │
│                                                              │
│  ┌────────────────────────────────────────────────┐         │
│  │         Backup Domain (VBAT)                   │         │
│  │  • RTC registers                               │         │
│  │  • Backup registers (32 × 32-bit)              │         │
│  │  • LSE oscillator                              │         │
│  │                                                │         │
│  │  전원: VBAT (배터리) 또는 VDD                  │         │
│  └────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

### 도메인별 전력 모드 조합

```
시나리오           │ D1      │ D2      │ D3      │ 전력    │ 용도
─────────────────┼────────┼────────┼────────┼────────┼──────────────
정상 동작          │ RUN     │ RUN     │ RUN     │ ~250mA  │ 최대 성능
D1 저전력         │ STOP    │ RUN     │ RUN     │ ~150mA  │ M7 휴면
D2 저전력         │ RUN     │ STOP    │ RUN     │ ~180mA  │ M4 휴면
D1/D2 저전력      │ STOP    │ STOP    │ RUN     │ ~100μA  │ 주변장치만
D3 자율 동작 ★    │ STBY    │ STBY    │ RUN     │ ~10μA   │ 초저전력 모니터링
전체 대기         │ STBY    │ STBY    │ STBY    │ ~2.4μA  │ 최저 전력
```

## 주요 기능

### D3 도메인의 특징

**독립 동작 능력**
- D1, D2가 STANDBY 상태에서도 D3만 RUN 모드 유지
- CPU 없이 하드웨어만으로 작업 수행
- BDMA로 메모리 전송 자동화
- 초저전력 소비 (~10 μA)

**사용 가능한 주변장치**
1. **LPTIM (Low-Power Timer)**
   - 타이머 기반 주기적 이벤트
   - PWM 출력 생성
   - 펄스 카운팅

2. **LPUART (Low-Power UART)**
   - 비동기 시리얼 통신
   - 웨이크업 기능
   - LSE 클럭으로 동작 가능

3. **I2C4 (Autonomous I2C)**
   - 센서 데이터 읽기
   - EEPROM 접근
   - BDMA와 연동

4. **SPI6 (Autonomous SPI)**
   - Flash 메모리 접근
   - 센서 통신
   - BDMA와 연동

5. **BDMA (Basic DMA)**
   - D3 주변장치와 SRD SRAM 간 데이터 전송
   - CPU 개입 없이 자동 동작

6. **ADC3 (16-bit ADC)**
   - 아날로그 신호 측정
   - 배터리 전압 모니터링
   - 온도 센서 읽기

### 응용 분야

1. **초저전력 센서 모니터링**
   - 주기적 센서 데이터 수집
   - 임계값 초과 시 D1 웨이크업
   - 배터리 수명 극대화

2. **백그라운드 데이터 로깅**
   - LPTIM으로 주기적 트리거
   - BDMA로 센서 데이터 수집
   - SRD SRAM에 버퍼링

3. **실시간 모니터링**
   - ADC로 지속적 측정
   - 비정상 조건 감지
   - 긴급 웨이크업

## 동작 원리

### D3 자율 동작 시퀀스

```
┌───────────────────┐
│   System Boot     │
│   (D1, D2, D3)    │
└─────────┬─────────┘
          │
          │ 초기화
          ▼
┌───────────────────┐
│ D3 Peripherals    │
│ Configuration     │
│ • LPTIM           │  주기적 트리거 설정
│ • I2C4            │  센서 주소 설정
│ • BDMA            │  전송 설정
│ • SRD SRAM        │  버퍼 준비
└─────────┬─────────┘
          │
          │ D1/D2 Standby 진입
          ▼
┌───────────────────┐
│ D1: STANDBY       │  전력: 거의 0
│ D2: STANDBY       │  전력: 거의 0
└───────────────────┘

┌───────────────────┐
│ D3: RUN (자율)    │  전력: ~10 μA
│                   │
│  ┌──────────┐     │
│  │  LPTIM   │────┐│
│  │ (1초마다) │    ││  트리거
│  └──────────┘    ││
│       │          ││
│       ▼          ││
│  ┌──────────┐    ││
│  │  BDMA    │◄───┘│
│  │  Start   │     │
│  └────┬─────┘     │
│       │           │
│       ▼           │
│  ┌──────────┐     │
│  │  I2C4    │     │  센서 읽기
│  │  Read    │─────┼──► Sensor
│  └────┬─────┘     │
│       │           │
│       ▼           │
│  ┌──────────┐     │
│  │  BDMA    │     │
│  │ Transfer │     │  데이터 → SRAM
│  └────┬─────┘     │
│       │           │
│       ▼           │
│  ┌──────────┐     │
│  │ SRD SRAM │     │  버퍼링
│  │  [data]  │     │
│  └────┬─────┘     │
│       │           │
│       │ 버퍼 가득? │
│       ▼           │
│  ┌──────────┐     │
│  │ Wakeup   │     │  임계값 도달 시
│  │ D1/D2    │     │  인터럽트 발생
│  └──────────┘     │
└───────────────────┘
          │
          │ 웨이크업 신호
          ▼
┌───────────────────┐
│ D1: RUN           │
│ D2: RUN           │  데이터 처리
│ • 데이터 읽기     │  • 무선 전송
│ • 분석            │  • 저장
│ • 판단            │  • 표시
└─────────┬─────────┘
          │
          │ 완료 후
          ▼
┌───────────────────┐
│ D1/D2: STANDBY    │  다시 저전력 모드
│ D3: RUN (자율)    │
└───────────────────┘
```

## 코드 구현

### 1. 시스템 초기화

```c
#include "stm32h7xx_hal.h"

// D3 도메인 주변장치 핸들
LPTIM_HandleTypeDef hlptim1;
I2C_HandleTypeDef hi2c4;
DMA_HandleTypeDef hbdma;

// SRD SRAM 버퍼 (D3 도메인)
#define BUFFER_SIZE  128
__attribute__((section(".srd_sram"))) uint8_t sensor_buffer[BUFFER_SIZE];
volatile uint32_t buffer_index = 0;

int main(void)
{
  // HAL 초기화
  HAL_Init();

  // 시스템 클럭 설정
  SystemClock_Config();

  // D3 도메인 초기화
  D3_Domain_Init();

  printf("\n========================================\n");
  printf("STM32H745 D3 Autonomous Operation Demo\n");
  printf("========================================\n\n");

  // D3 자율 동작 설정
  Configure_D3_Autonomous_Mode();

  // D1, D2 도메인 STANDBY 진입
  printf("Putting D1 and D2 domains to STANDBY...\n");
  printf("D3 domain will continue autonomous operation.\n");
  HAL_Delay(1000);

  Enter_D1_D2_Standby_Mode();

  // 여기는 실행되지 않음 (D1 STANDBY)
  while (1);
}
```

### 2. D3 도메인 초기화

```c
void D3_Domain_Init(void)
{
  RCC_PeriphCLKInitTypeDef PeriphClkInit = {0};

  printf("Initializing D3 Domain...\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 1. D3 클럭 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // LSE 활성화 (32.768kHz, 저전력)
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_LSE;
  RCC_OscInitStruct.LSEState = RCC_LSE_ON;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_NONE;

  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  // D3 주변장치 클럭 소스: LSE
  PeriphClkInit.PeriphClockSelection = RCC_PERIPHCLK_LPTIM1 |
                                       RCC_PERIPHCLK_LPUART1 |
                                       RCC_PERIPHCLK_I2C4;
  PeriphClkInit.Lptim1ClockSelection = RCC_LPTIM1CLKSOURCE_LSE;
  PeriphClkInit.Lpuart1ClockSelection = RCC_LPUART1CLKSOURCE_LSE;
  PeriphClkInit.I2c4ClockSelection = RCC_I2C4CLKSOURCE_D3PCLK1;

  if (HAL_RCCEx_PeriphCLKConfig(&PeriphClkInit) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 2. SRD SRAM 클럭 활성화
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  __HAL_RCC_D3SRAM1_CLK_ENABLE();

  // SRD SRAM 초기화
  memset(sensor_buffer, 0, BUFFER_SIZE);

  printf("  • LSE clock: 32.768 kHz\n");
  printf("  • SRD SRAM: 64 KB\n");
  printf("D3 Domain initialized.\n\n");
}
```

### 3. LPTIM 설정 (주기적 트리거)

```c
void LPTIM_Init(void)
{
  LPTIM_HandleTypeDef hlptim1;

  printf("Configuring LPTIM1...\n");

  // LPTIM1 클럭 활성화
  __HAL_RCC_LPTIM1_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // LPTIM1 초기화
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  hlptim1.Instance = LPTIM1;
  hlptim1.Init.Clock.Source = LPTIM_CLOCKSOURCE_APBCLOCK_LPOSC;
  hlptim1.Init.Clock.Prescaler = LPTIM_PRESCALER_DIV128;  // 32768/128 = 256Hz
  hlptim1.Init.Trigger.Source = LPTIM_TRIGSOURCE_SOFTWARE;
  hlptim1.Init.OutputPolarity = LPTIM_OUTPUTPOLARITY_HIGH;
  hlptim1.Init.UpdateMode = LPTIM_UPDATE_IMMEDIATE;
  hlptim1.Init.CounterSource = LPTIM_COUNTERSOURCE_INTERNAL;
  hlptim1.Init.Input1Source = LPTIM_INPUT1SOURCE_GPIO;
  hlptim1.Init.Input2Source = LPTIM_INPUT2SOURCE_GPIO;

  if (HAL_LPTIM_Init(&hlptim1) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 주기적 트리거 설정 (1초마다)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // ARR = 256 (256 / 256Hz = 1초)
  if (HAL_LPTIM_TimeOut_Start_IT(&hlptim1, 0xFFFF, 256) != HAL_OK)
  {
    Error_Handler();
  }

  printf("  • Period: 1 second\n");
  printf("  • Trigger: BDMA transfer\n");
  printf("LPTIM1 configured.\n\n");
}

// LPTIM 인터럽트 핸들러
void LPTIM1_IRQHandler(void)
{
  HAL_LPTIM_IRQHandler(&hlptim1);
}

void HAL_LPTIM_CompareMatchCallback(LPTIM_HandleTypeDef *hlptim)
{
  if (hlptim->Instance == LPTIM1)
  {
    // 1초마다 호출됨
    // BDMA 트리거 또는 다른 작업 수행
    Trigger_BDMA_Transfer();
  }
}
```

### 4. I2C4 설정 (자율 센서 읽기)

```c
void I2C4_Init(void)
{
  I2C_HandleTypeDef hi2c4;

  printf("Configuring I2C4...\n");

  // I2C4 클럭 활성화
  __HAL_RCC_I2C4_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // GPIO 설정 (PD12: SCL, PD13: SDA)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  GPIO_InitTypeDef GPIO_InitStruct = {0};
  __HAL_RCC_GPIOD_CLK_ENABLE();

  GPIO_InitStruct.Pin = GPIO_PIN_12 | GPIO_PIN_13;
  GPIO_InitStruct.Mode = GPIO_MODE_AF_OD;
  GPIO_InitStruct.Pull = GPIO_PULLUP;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  GPIO_InitStruct.Alternate = GPIO_AF4_I2C4;
  HAL_GPIO_Init(GPIOD, &GPIO_InitStruct);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // I2C4 초기화
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  hi2c4.Instance = I2C4;
  hi2c4.Init.Timing = 0x00000E14;  // 100kHz @ D3PCLK1
  hi2c4.Init.OwnAddress1 = 0;
  hi2c4.Init.AddressingMode = I2C_ADDRESSINGMODE_7BIT;
  hi2c4.Init.DualAddressMode = I2C_DUALADDRESS_DISABLE;
  hi2c4.Init.OwnAddress2 = 0;
  hi2c4.Init.OwnAddress2Masks = I2C_OA2_NOMASK;
  hi2c4.Init.GeneralCallMode = I2C_GENERALCALL_DISABLE;
  hi2c4.Init.NoStretchMode = I2C_NOSTRETCH_DISABLE;

  if (HAL_I2C_Init(&hi2c4) != HAL_OK)
  {
    Error_Handler();
  }

  // 아날로그 필터 활성화
  HAL_I2CEx_ConfigAnalogFilter(&hi2c4, I2C_ANALOGFILTER_ENABLE);

  printf("  • Speed: 100 kHz\n");
  printf("  • Pins: PD12 (SCL), PD13 (SDA)\n");
  printf("I2C4 configured.\n\n");
}

// I2C4로 센서 읽기 (예: BME280 온습도 센서)
#define SENSOR_I2C_ADDR  0x76 << 1  // BME280 주소

HAL_StatusTypeDef Read_Sensor_Data(uint8_t *data, uint16_t size)
{
  HAL_StatusTypeDef status;

  // 센서 레지스터 읽기 (0xF7부터 8바이트)
  status = HAL_I2C_Mem_Read(&hi2c4,
                            SENSOR_I2C_ADDR,
                            0xF7,
                            I2C_MEMADD_SIZE_8BIT,
                            data,
                            size,
                            1000);

  if (status != HAL_OK)
  {
    printf("ERROR: I2C read failed\n");
  }

  return status;
}
```

### 5. BDMA 설정 (자동 데이터 전송)

```c
void BDMA_Init(void)
{
  DMA_HandleTypeDef hbdma;

  printf("Configuring BDMA...\n");

  // BDMA 클럭 활성화
  __HAL_RCC_BDMA_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // BDMA 채널 0 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  hbdma.Instance = BDMA_Channel0;
  hbdma.Init.Request = BDMA_REQUEST_I2C4_RX;  // I2C4 RX 트리거
  hbdma.Init.Direction = DMA_PERIPH_TO_MEMORY;
  hbdma.Init.PeriphInc = DMA_PINC_DISABLE;
  hbdma.Init.MemInc = DMA_MINC_ENABLE;
  hbdma.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
  hbdma.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
  hbdma.Init.Mode = DMA_CIRCULAR;  // 순환 모드
  hbdma.Init.Priority = DMA_PRIORITY_HIGH;
  hbdma.Init.FIFOMode = DMA_FIFOMODE_DISABLE;

  if (HAL_DMA_Init(&hbdma) != HAL_OK)
  {
    Error_Handler();
  }

  // I2C4와 DMA 연결
  __HAL_LINKDMA(&hi2c4, hdmarx, hbdma);

  // BDMA 인터럽트 설정
  HAL_NVIC_SetPriority(BDMA_Channel0_IRQn, 0, 0);
  HAL_NVIC_EnableIRQ(BDMA_Channel0_IRQn);

  printf("  • Channel: BDMA_Channel0\n");
  printf("  • Trigger: I2C4 RX\n");
  printf("  • Destination: SRD SRAM\n");
  printf("BDMA configured.\n\n");
}

// BDMA 전송 시작
void Start_BDMA_Sensor_Collection(void)
{
  // I2C4 + BDMA로 센서 데이터 자동 수집
  HAL_I2C_Mem_Read_DMA(&hi2c4,
                       SENSOR_I2C_ADDR,
                       0xF7,
                       I2C_MEMADD_SIZE_8BIT,
                       &sensor_buffer[buffer_index],
                       8);  // 8바이트 읽기

  buffer_index += 8;

  if (buffer_index >= BUFFER_SIZE)
  {
    // 버퍼 가득 참 - D1 웨이크업 트리거
    printf("Buffer full! Waking up D1...\n");
    Trigger_D1_Wakeup();
    buffer_index = 0;
  }
}

// BDMA 인터럽트 핸들러
void BDMA_Channel0_IRQHandler(void)
{
  HAL_DMA_IRQHandler(&hbdma);
}

void HAL_I2C_MemRxCpltCallback(I2C_HandleTypeDef *hi2c)
{
  if (hi2c->Instance == I2C4)
  {
    // BDMA 전송 완료
    // 다음 LPTIM 트리거까지 대기
  }
}
```

### 6. D3 자율 모드 설정

```c
void Configure_D3_Autonomous_Mode(void)
{
  printf("Configuring D3 Autonomous Mode...\n\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 1. D3 주변장치 초기화
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  LPTIM_Init();   // 주기적 트리거
  I2C4_Init();    // 센서 통신
  BDMA_Init();    // 자동 데이터 전송

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 2. D3 도메인 전력 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // D3 도메인을 RUN 모드로 유지
  __HAL_RCC_D3SRAM1_CLK_ENABLE();

  // D3 주변장치 저전력 클럭 활성화
  __HAL_RCC_LPTIM1_CLK_ENABLE();
  __HAL_RCC_I2C4_CLK_ENABLE();
  __HAL_RCC_LPUART1_CLK_ENABLE();
  __HAL_RCC_BDMA_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 3. D1 웨이크업 소스 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // LPTIM1 인터럽트로 D1 웨이크업 가능
  __HAL_RCC_LPTIM1_CLK_ENABLE();
  HAL_NVIC_SetPriority(LPTIM1_IRQn, 0, 0);
  HAL_NVIC_EnableIRQ(LPTIM1_IRQn);

  // EXTI 라인 설정
  __HAL_RCC_SYSCFG_CLK_ENABLE();

  printf("D3 Autonomous Mode configured.\n");
  printf("Ready for D1/D2 standby.\n\n");
}
```

### 7. D1/D2 STANDBY 진입

```c
void Enter_D1_D2_Standby_Mode(void)
{
  printf("===========================================\n");
  printf("Entering D1/D2 STANDBY, D3 RUN mode...\n");
  printf("===========================================\n\n");

  HAL_Delay(1000);  // 메시지 출력 대기

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 1. D1 도메인 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // 플래그 클리어
  __HAL_PWR_CLEAR_FLAG(PWR_FLAG_STOP);
  __HAL_PWR_CLEAR_FLAG(PWR_FLAG_SB);

  // D1 STANDBY 진입
  HAL_PWREx_EnterSTANDBYMode(PWR_D1_DOMAIN);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 2. D2 도메인 설정 (듀얼 코어인 경우)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // D2 STANDBY 진입
  HAL_PWREx_EnterSTANDBYMode(PWR_D2_DOMAIN);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 3. D3 도메인은 RUN 상태 유지
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // D3는 계속 동작:
  // - LPTIM1: 1초마다 트리거
  // - BDMA: I2C4 데이터 → SRD SRAM
  // - I2C4: 센서 읽기

  // 이 코드는 실행되지 않음 (D1 STANDBY)
}
```

### 8. D1 웨이크업 트리거

```c
void Trigger_D1_Wakeup(void)
{
  // D3에서 D1으로 웨이크업 신호 보내기

  // 방법 1: EXTI 인터럽트
  // GPIO를 사용하여 인터럽트 생성
  // (하드웨어 연결 필요)

  // 방법 2: LPTIM 인터럽트
  // LPTIM1에서 인터럽트 발생
  HAL_LPTIM_TimeOut_Start_IT(&hlptim1, 0xFFFF, 1);

  // 방법 3: 특정 조건 플래그
  // SRD SRAM에 플래그 설정
  // D1 부팅 후 플래그 확인
}

// D1 웨이크업 후 처리
void D1_Wakeup_Handler(void)
{
  // 시스템 클럭 재설정
  SystemClock_Config();

  printf("\n===========================================\n");
  printf("D1 woken up from STANDBY!\n");
  printf("===========================================\n\n");

  // SRD SRAM 데이터 읽기
  printf("Reading sensor data from SRD SRAM...\n");

  for (uint32_t i = 0; i < buffer_index; i += 8)
  {
    // 센서 데이터 파싱 (BME280 예제)
    uint32_t raw_temp = (sensor_buffer[i+3] << 12) |
                        (sensor_buffer[i+4] << 4) |
                        (sensor_buffer[i+5] >> 4);

    uint32_t raw_press = (sensor_buffer[i+0] << 12) |
                         (sensor_buffer[i+1] << 4) |
                         (sensor_buffer[i+2] >> 4);

    uint32_t raw_hum = (sensor_buffer[i+6] << 8) |
                       sensor_buffer[i+7];

    printf("  Sample %lu: T=%lu P=%lu H=%lu\n",
           i/8, raw_temp, raw_press, raw_hum);
  }

  // 데이터 처리 및 전송
  Process_Sensor_Data();
  Transmit_Data_Over_LoRa();

  // 다시 D1/D2 STANDBY 진입
  printf("Data processed. Going back to standby...\n\n");
  HAL_Delay(1000);

  buffer_index = 0;  // 버퍼 리셋
  Enter_D1_D2_Standby_Mode();
}
```

## 고급 기능

### 1. 임계값 기반 웨이크업

```c
// ADC3로 센서 값 모니터링
ADC_HandleTypeDef hadc3;

void ADC3_Init(void)
{
  __HAL_RCC_ADC3_CLK_ENABLE();

  hadc3.Instance = ADC3;
  hadc3.Init.ClockPrescaler = ADC_CLOCK_ASYNC_DIV4;
  hadc3.Init.Resolution = ADC_RESOLUTION_16B;
  hadc3.Init.ScanConvMode = ADC_SCAN_DISABLE;
  hadc3.Init.EOCSelection = ADC_EOC_SINGLE_CONV;
  hadc3.Init.LowPowerAutoWait = ENABLE;
  hadc3.Init.ContinuousConvMode = DISABLE;
  hadc3.Init.NbrOfConversion = 1;

  if (HAL_ADC_Init(&hadc3) != HAL_OK)
  {
    Error_Handler();
  }

  // ADC 채널 설정 (온도 센서 예제)
  ADC_ChannelConfTypeDef sConfig = {0};
  sConfig.Channel = ADC_CHANNEL_TEMPSENSOR;
  sConfig.Rank = ADC_REGULAR_RANK_1;
  sConfig.SamplingTime = ADC_SAMPLETIME_810CYCLES_5;
  sConfig.SingleDiff = ADC_SINGLE_ENDED;
  sConfig.OffsetNumber = ADC_OFFSET_NONE;
  sConfig.Offset = 0;

  HAL_ADC_ConfigChannel(&hadc3, &sConfig);

  // 아날로그 워치독 설정 (임계값 모니터링)
  ADC_AnalogWDGConfTypeDef AnalogWDGConfig = {0};
  AnalogWDGConfig.WatchdogNumber = ADC_ANALOGWATCHDOG_1;
  AnalogWDGConfig.WatchdogMode = ADC_ANALOGWATCHDOG_SINGLE_REG;
  AnalogWDGConfig.Channel = ADC_CHANNEL_TEMPSENSOR;
  AnalogWDGConfig.ITMode = ENABLE;
  AnalogWDGConfig.HighThreshold = 2500;  // 상한
  AnalogWDGConfig.LowThreshold = 500;    // 하한

  HAL_ADC_AnalogWDGConfig(&hadc3, &AnalogWDGConfig);

  printf("ADC3 Watchdog configured\n");
  printf("  • Low threshold: 500\n");
  printf("  • High threshold: 2500\n");
}

// ADC 워치독 인터럽트
void ADC3_IRQHandler(void)
{
  HAL_ADC_IRQHandler(&hadc3);
}

void HAL_ADC_LevelOutOfWindowCallback(ADC_HandleTypeDef* hadc)
{
  if (hadc->Instance == ADC3)
  {
    // 임계값 초과!
    printf("ALERT: Temperature out of range!\n");

    // D1 웨이크업
    Trigger_D1_Wakeup();
  }
}
```

### 2. LPUART 데이터 로깅

```c
UART_HandleTypeDef hlpuart1;

void LPUART_Init(void)
{
  __HAL_RCC_LPUART1_CLK_ENABLE();

  // GPIO 설정 (PA9: TX, PA10: RX)
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  __HAL_RCC_GPIOA_CLK_ENABLE();

  GPIO_InitStruct.Pin = GPIO_PIN_9 | GPIO_PIN_10;
  GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  GPIO_InitStruct.Alternate = GPIO_AF3_LPUART;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  // LPUART1 초기화
  hlpuart1.Instance = LPUART1;
  hlpuart1.Init.BaudRate = 9600;
  hlpuart1.Init.WordLength = UART_WORDLENGTH_8B;
  hlpuart1.Init.StopBits = UART_STOPBITS_1;
  hlpuart1.Init.Parity = UART_PARITY_NONE;
  hlpuart1.Init.Mode = UART_MODE_TX_RX;
  hlpuart1.Init.HwFlowCtl = UART_HWCONTROL_NONE;

  if (HAL_UART_Init(&hlpuart1) != HAL_OK)
  {
    Error_Handler();
  }

  printf("LPUART1 initialized (9600 baud)\n");
}

// D3 자율 모드에서 로깅
void Log_Sensor_Data_Via_LPUART(uint8_t *data, uint16_t size)
{
  char log_msg[64];

  // 타임스탬프 (LPTIM 카운터 사용)
  uint32_t timestamp = HAL_LPTIM_ReadCounter(&hlptim1);

  // 로그 메시지 생성
  snprintf(log_msg, sizeof(log_msg),
           "[%lu] Data: %02X %02X %02X %02X\r\n",
           timestamp, data[0], data[1], data[2], data[3]);

  // LPUART로 전송
  HAL_UART_Transmit(&hlpuart1, (uint8_t*)log_msg, strlen(log_msg), 1000);
}
```

### 3. 다중 센서 관리

```c
typedef enum {
  SENSOR_BME280 = 0,  // 온습도 센서
  SENSOR_LIS3DH,      // 가속도계
  SENSOR_MAX30102,    // 심박 센서
  SENSOR_COUNT
} SensorType_t;

typedef struct {
  SensorType_t type;
  uint8_t i2c_addr;
  uint8_t reg_addr;
  uint8_t data_size;
  uint32_t sample_interval;  // LPTIM 틱 단위
} SensorConfig_t;

SensorConfig_t sensors[SENSOR_COUNT] = {
  {SENSOR_BME280,   0x76 << 1, 0xF7, 8, 256},   // 1초마다
  {SENSOR_LIS3DH,   0x18 << 1, 0x28, 6, 128},   // 0.5초마다
  {SENSOR_MAX30102, 0x57 << 1, 0x05, 6, 64}     // 0.25초마다
};

uint32_t sensor_counters[SENSOR_COUNT] = {0};

void Multi_Sensor_Task(void)
{
  static uint32_t lptim_tick = 0;

  lptim_tick++;

  // 각 센서의 샘플링 간격 확인
  for (int i = 0; i < SENSOR_COUNT; i++)
  {
    sensor_counters[i]++;

    if (sensor_counters[i] >= sensors[i].sample_interval)
    {
      // 센서 읽기 (BDMA 사용)
      Read_Sensor_With_BDMA(i);

      sensor_counters[i] = 0;
    }
  }
}

void Read_Sensor_With_BDMA(uint8_t sensor_index)
{
  SensorConfig_t *sensor = &sensors[sensor_index];

  // BDMA로 센서 데이터 읽기
  HAL_I2C_Mem_Read_DMA(&hi2c4,
                       sensor->i2c_addr,
                       sensor->reg_addr,
                       I2C_MEMADD_SIZE_8BIT,
                       &sensor_buffer[buffer_index],
                       sensor->data_size);

  buffer_index += sensor->data_size;

  // 센서 타입 표시 (디버깅)
  sensor_buffer[buffer_index++] = sensor->type;
}
```

### 4. 에너지 하베스팅 연동

```c
// 태양광 패널 전압 모니터링
#define SOLAR_THRESHOLD  2500  // 2.5V

void Energy_Harvesting_Monitor(void)
{
  // ADC3로 태양광 전압 측정
  HAL_ADC_Start(&hadc3);
  HAL_ADC_PollForConversion(&hadc3, 100);
  uint32_t solar_voltage = HAL_ADC_GetValue(&hadc3);
  HAL_ADC_Stop(&hadc3);

  if (solar_voltage > SOLAR_THRESHOLD)
  {
    // 충분한 전력 - 샘플링 빈도 증가
    printf("Solar power available: %lu mV\n", solar_voltage * 3300 / 65535);

    // LPTIM 주기 단축 (더 자주 센서 읽기)
    HAL_LPTIM_TimeOut_Start_IT(&hlptim1, 0xFFFF, 128);  // 0.5초마다

    // 추가 센서 활성화
    Enable_Additional_Sensors();
  }
  else
  {
    // 배터리만 사용 - 샘플링 빈도 감소
    printf("Battery power only: %lu mV\n", solar_voltage * 3300 / 65535);

    // LPTIM 주기 연장 (덜 자주 센서 읽기)
    HAL_LPTIM_TimeOut_Start_IT(&hlptim1, 0xFFFF, 512);  // 2초마다

    // 불필요한 센서 비활성화
    Disable_Non_Critical_Sensors();
  }
}
```

## 전력 소비 분석

### 도메인별 전력 소비

```
┌──────────────────────────────────────────────────────┐
│               전력 소비 분석                          │
├──────────────────────────────────────────────────────┤
│                                                       │
│  전체 RUN 모드:                      ~250 mA          │
│  ┌────────────────────────────────────────────┐      │
│  │ D1: Cortex-M7 @ 480MHz      ~150 mA        │      │
│  │ D2: Cortex-M4 @ 240MHz      ~80 mA         │      │
│  │ D3: Peripherals             ~20 mA         │      │
│  └────────────────────────────────────────────┘      │
│                                                       │
│  D1/D2 STANDBY, D3 RUN:              ~10 μA           │
│  ┌────────────────────────────────────────────┐      │
│  │ D1: STANDBY                 ~1 μA          │      │
│  │ D2: STANDBY                 ~1 μA          │      │
│  │ D3: LPTIM + I2C4 + BDMA     ~8 μA          │      │
│  │   - LPTIM1                  ~2 μA          │      │
│  │   - I2C4 (idle)             ~1 μA          │      │
│  │   - I2C4 (active)           ~3 μA          │      │
│  │   - BDMA (active)           ~2 μA          │      │
│  └────────────────────────────────────────────┘      │
│                                                       │
│  전체 STANDBY:                       ~2.4 μA          │
│  ┌────────────────────────────────────────────┐      │
│  │ D1: STANDBY                 ~0.8 μA        │      │
│  │ D2: STANDBY                 ~0.8 μA        │      │
│  │ D3: STANDBY                 ~0.8 μA        │      │
│  └────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────┘
```

### 배터리 수명 계산

**시나리오: 환경 모니터링 스테이션**

```
동작 패턴:
- D3 자율 동작: 59분 (10 μA)
- D1 데이터 처리: 1분 (100 mA)
- 주기: 60분

평균 전력:
= (0.01mA × 59min + 100mA × 1min) / 60min
= (0.59 + 100) / 60
= 100.59 / 60
≈ 1.68 mA

배터리 수명 (2000 mAh):
= 2000 mAh / 1.68 mA
≈ 1190 시간
≈ 49.6 일
≈ 1.7 개월

만약 D3만 사용 (D1 웨이크업 없음):
= 2000 mAh / 0.01 mA
= 200,000 시간
≈ 22.8 년!
```

## 실전 응용 예제

### 예제 1: 스마트 농업 모니터링

```c
// 토양 수분, 온도, 조도 센서 모니터링
// D3 자율 모드로 1시간마다 센서 읽기
// 임계값 초과 시 D1 웨이크업하여 알림 전송

#define SOIL_MOISTURE_THRESHOLD  30  // 30% 이하 시 알림

typedef struct {
  uint16_t soil_moisture;
  int16_t temperature;
  uint16_t light_level;
  uint32_t timestamp;
} AgriData_t;

__attribute__((section(".srd_sram"))) AgriData_t agri_buffer[24];  // 24시간 분량

void Smart_Agriculture_Task(void)
{
  // D3 자율 모드 설정
  Configure_D3_Autonomous_Mode();

  // 센서 초기화
  Init_Soil_Moisture_Sensor();  // ADC3
  Init_Temperature_Sensor();     // I2C4
  Init_Light_Sensor();           // I2C4

  // LPTIM: 1시간마다 트리거
  HAL_LPTIM_TimeOut_Start_IT(&hlptim1, 0xFFFF, 3600);

  while (1)
  {
    // 센서 읽기 (BDMA 사용)
    agri_buffer[buffer_index].soil_moisture = Read_Soil_Moisture();
    agri_buffer[buffer_index].temperature = Read_Temperature_I2C();
    agri_buffer[buffer_index].light_level = Read_Light_Level();
    agri_buffer[buffer_index].timestamp = Get_RTC_Timestamp();

    // 임계값 확인
    if (agri_buffer[buffer_index].soil_moisture < SOIL_MOISTURE_THRESHOLD)
    {
      printf("ALERT: Soil moisture low (%d%%)\n",
             agri_buffer[buffer_index].soil_moisture);

      // 즉시 D1 웨이크업하여 알림 전송
      Trigger_D1_Wakeup();
    }

    buffer_index++;

    if (buffer_index >= 24)
    {
      // 24시간 데이터 전송
      Trigger_D1_Wakeup();
      buffer_index = 0;
    }

    // D1/D2 STANDBY, D3 RUN
    Enter_D1_D2_Standby_Mode();
  }
}
```

### 예제 2: 산업용 진동 모니터링

```c
// 가속도계로 기계 진동 모니터링
// D3 자율 모드로 고속 샘플링 (100Hz)
// FFT 분석은 D1 웨이크업 후 수행

#define SAMPLE_RATE      100  // 100 Hz
#define FFT_SIZE         256

__attribute__((section(".srd_sram"))) int16_t accel_buffer[FFT_SIZE * 3];  // X, Y, Z

void Vibration_Monitoring_Task(void)
{
  // 가속도계 초기화 (I2C4)
  Init_Accelerometer_LIS3DH();

  // LPTIM: 10ms마다 트리거 (100Hz)
  HAL_LPTIM_TimeOut_Start_IT(&hlptim1, 0xFFFF, 10);

  // BDMA: 가속도계 → SRD SRAM
  Setup_BDMA_Accelerometer();

  uint32_t sample_count = 0;

  while (1)
  {
    // LPTIM 트리거로 BDMA 시작
    // 가속도계 6바이트 (X, Y, Z) → SRD SRAM

    sample_count++;

    if (sample_count >= FFT_SIZE)
    {
      // 충분한 샘플 수집
      printf("Collected %d samples for FFT\n", FFT_SIZE);

      // D1 웨이크업하여 FFT 분석
      Trigger_D1_Wakeup();

      sample_count = 0;
    }

    // D1/D2 STANDBY, D3 RUN
    Enter_D1_D2_Standby_Mode();
  }
}

// D1 웨이크업 후 FFT 분석
void D1_FFT_Analysis(void)
{
  printf("Performing FFT analysis...\n");

  // SRD SRAM 데이터 읽기
  float fft_input[FFT_SIZE];
  for (int i = 0; i < FFT_SIZE; i++)
  {
    // X축 가속도
    fft_input[i] = (float)accel_buffer[i * 3];
  }

  // FFT 수행 (CMSIS-DSP 라이브러리 사용)
  arm_rfft_fast_f32(&fft_instance, fft_input, fft_output, 0);

  // 주파수 분석
  float peak_freq = Find_Peak_Frequency(fft_output, FFT_SIZE);

  printf("Peak frequency: %.2f Hz\n", peak_freq);

  // 비정상 진동 감지
  if (peak_freq > 50.0f)
  {
    printf("WARNING: Abnormal vibration detected!\n");
    Send_Alert();
  }

  // 다시 STANDBY
  Enter_D1_D2_Standby_Mode();
}
```

## 트러블슈팅

### 문제 1: D3 도메인이 동작하지 않음

```c
void Debug_D3_Domain(void)
{
  printf("=== D3 Domain Debug ===\n");

  // 1. D3 클럭 확인
  if (!(RCC->D3CCIPR & RCC_D3CCIPR_LPTIM1SEL))
  {
    printf("ERROR: LPTIM1 clock not configured\n");
  }

  // 2. LSE 상태 확인
  if (!(RCC->BDCR & RCC_BDCR_LSERDY))
  {
    printf("ERROR: LSE not ready\n");
    printf("Enabling LSE...\n");

    __HAL_RCC_LSE_CONFIG(RCC_LSE_ON);

    // LSE ready 대기
    uint32_t timeout = 5000;
    while (!(RCC->BDCR & RCC_BDCR_LSERDY) && timeout--)
    {
      HAL_Delay(1);
    }

    if (timeout == 0)
    {
      printf("ERROR: LSE failed to start\n");
    }
  }

  // 3. SRD SRAM 클럭 확인
  if (!(RCC->AHB4ENR & RCC_AHB4ENR_SRDSMEN))
  {
    printf("WARNING: SRD SRAM clock not enabled\n");
    __HAL_RCC_D3SRAM1_CLK_ENABLE();
  }

  printf("D3 Domain debug completed.\n");
}
```

### 문제 2: BDMA가 동작하지 않음

```c
void Debug_BDMA(void)
{
  printf("=== BDMA Debug ===\n");

  // 1. BDMA 클럭 확인
  if (!(RCC->AHB4ENR & RCC_AHB4ENR_BDMAEN))
  {
    printf("ERROR: BDMA clock not enabled\n");
    __HAL_RCC_BDMA_CLK_ENABLE();
  }

  // 2. DMA 상태 확인
  printf("BDMA_Channel0 CR: 0x%08lX\n", BDMA_Channel0->CCR);
  printf("BDMA_Channel0 CNDTR: %lu\n", BDMA_Channel0->CNDTR);
  printf("BDMA_Channel0 CPAR: 0x%08lX\n", BDMA_Channel0->CPAR);
  printf("BDMA_Channel0 CM0AR: 0x%08lX\n", BDMA_Channel0->CM0AR);

  // 3. 메모리 주소 확인
  // SRD SRAM 주소: 0x38000000 - 0x3800FFFF
  if (BDMA_Channel0->CM0AR < 0x38000000 ||
      BDMA_Channel0->CM0AR > 0x3800FFFF)
  {
    printf("ERROR: BDMA destination not in SRD SRAM!\n");
    printf("  SRD SRAM range: 0x38000000 - 0x3800FFFF\n");
  }

  printf("BDMA debug completed.\n");
}
```

### 문제 3: D1 웨이크업이 안 됨

```c
void Debug_D1_Wakeup(void)
{
  printf("=== D1 Wakeup Debug ===\n");

  // 1. 웨이크업 소스 확인
  printf("PWR_WKUPFR: 0x%08lX\n", PWR->WKUPFR);
  printf("PWR_WKUPCR: 0x%08lX\n", PWR->WKUPCR);

  // 2. EXTI 라인 확인
  printf("EXTI_IMR1: 0x%08lX\n", EXTI->IMR1);
  printf("EXTI_PR1: 0x%08lX\n", EXTI->PR1);

  // 3. LPTIM 인터럽트 확인
  if (!(LPTIM1->IER & LPTIM_IER_CMPMIE))
  {
    printf("WARNING: LPTIM1 compare interrupt not enabled\n");
  }

  // 4. NVIC 확인
  printf("NVIC_ISER[LPTIM1_IRQn]: 0x%08lX\n",
         NVIC->ISER[LPTIM1_IRQn >> 5]);

  printf("D1 Wakeup debug completed.\n");
}
```

## 참고 자료

- **RM0399**: STM32H745 Reference Manual
  - Chapter 6: Power Controller (PWR)
  - Chapter 7: Reset and Clock Control (RCC)
  - Chapter 16: Basic DMA (BDMA)
  - Chapter 46: Low-Power Timer (LPTIM)

- **AN5293**: Building low-power applications with STM32H7 series

- **AN4749**: Managing low-power modes in STM32H7 series

- **DS12110**: STM32H745/755 datasheet (전력 특성)

## 관련 예제

- **PWR_STOP_RTC**: STOP 모드 기본 사용법
- **PWR_D1ON_D2OFF**: D1만 활성화
- **PWR_D2ON_D1OFF**: D2만 활성화
- **LPTIM_PWM**: LPTIM PWM 출력
- **I2C_TwoBoards_ComDMA**: I2C DMA 통신
