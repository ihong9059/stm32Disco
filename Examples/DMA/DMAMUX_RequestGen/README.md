# DMAMUX_RequestGen - DMAMUX 요청 생성기를 이용한 BDMA 제어

## 개요

이 예제는 STM32H745의 DMAMUX(DMA Request Multiplexer) 요청 생성기 기능을 사용하여 BDMA(Basic DMA)를 제어하는 방법을 보여줍니다. LPTIM2(Low Power Timer)를 트리거 소스로 사용하여 주기적으로 DMA 전송을 발생시키고, SRAM4에서 GPIO 포트로 데이터를 전송하여 LED를 제어합니다.

## 하드웨어 요구사항

- **STM32H745I-DISCO 보드**
- **LED**: PI12 (LED1) 또는 GPIO 출력 핀
- **메모리**: SRAM4 (D3 도메인, 64KB)
- **타이머**: LPTIM2 (저전력 타이머)

## 주요 기능

### DMAMUX 요청 생성기
- **트리거 소스**: LPTIM2 OUT 신호
- **DMA 요청 생성**: 주기적 또는 이벤트 기반
- **유연한 라우팅**: 임의의 트리거를 DMA 요청으로 변환
- **독립 동작**: CPU 개입 없이 자동 동작

### BDMA (Basic DMA)
- **D3 도메인 전용**: SRAM4 액세스에 최적화
- **저전력 동작**: D3 도메인이 활성화된 상태에서도 동작
- **8채널**: 각 채널 독립 설정 가능
- **간단한 구조**: 기본 메모리-주변장치 전송

## 동작 원리

### 시스템 구성
```
┌──────────┐
│  LPTIM2  │  (저전력 타이머)
│  OUT     │
└────┬─────┘
     │ 트리거 신호
     ▼
┌──────────────┐
│   DMAMUX     │  (요청 생성기)
│  Request Gen │
└──────┬───────┘
       │ DMA 요청
       ▼
┌──────────────┐        ┌──────────┐        ┌──────────┐
│    BDMA      │───────►│  SRAM4   │───────►│  GPIOI   │
│  Channel 0   │        │  Buffer  │        │  ODR     │
└──────────────┘        └──────────┘        └──────────┘
                        [0x00, 0x01,        LED Toggle
                         0x00, 0x01]
```

### 데이터 흐름
```
1. LPTIM2가 주기적으로 OUT 신호 생성 (예: 1Hz)
2. DMAMUX 요청 생성기가 OUT 신호를 DMA 요청으로 변환
3. BDMA가 SRAM4에서 데이터 읽기
4. BDMA가 GPIOI ODR 레지스터에 데이터 쓰기
5. LED 상태 변경 (0x00 = OFF, 0x01 = ON)

Time →
LPTIM2: ─┐  ┌─┐  ┌─┐  ┌─┐  ┌─┐
         └──┘  └──┘  └──┘  └──┘

DMAMUX: ───┬───┬───┬───┬───┬───
Request    │   │   │   │   │

BDMA:   ───┴─┬─┴─┬─┴─┬─┴─┬─┴─
Transfer     └───┴───┴───┴───

LED:    ▁▔▁▔▁▔▁▔▁▔ (깜박임)
```

## 코드 구조

### 1. LPTIM2 초기화 (트리거 소스)

```c
// 전역 변수
LPTIM_HandleTypeDef hlptim2;
DMA_HandleTypeDef hbdma_memtomem_dma2_stream0;

void LPTIM2_Init(void)
{
  LPTIM_ClockConfigTypeDef clock_config = {0};
  LPTIM_ULPClockConfigTypeDef ulp_config = {0};

  // LPTIM2 클럭 활성화
  __HAL_RCC_LPTIM2_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // LPTIM2 기본 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  hlptim2.Instance = LPTIM2;
  hlptim2.Init.Clock.Source = LPTIM_CLOCKSOURCE_APBCLOCK_LPOSC;
  hlptim2.Init.Clock.Prescaler = LPTIM_PRESCALER_DIV128;  // 분주비: 128
  hlptim2.Init.Trigger.Source = LPTIM_TRIGSOURCE_SOFTWARE;
  hlptim2.Init.OutputPolarity = LPTIM_OUTPUTPOLARITY_HIGH;
  hlptim2.Init.UpdateMode = LPTIM_UPDATE_IMMEDIATE;
  hlptim2.Init.CounterSource = LPTIM_COUNTERSOURCE_INTERNAL;
  hlptim2.Init.Input1Source = LPTIM_INPUT1SOURCE_GPIO;
  hlptim2.Init.Input2Source = LPTIM_INPUT2SOURCE_GPIO;

  if (HAL_LPTIM_Init(&hlptim2) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // PWM 모드 설정 (OUT 신호 생성)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // LPTIM2 클럭 = 100MHz (LSE 또는 내부 클럭)
  // Prescaler = 128
  // Period = 781
  // 주파수 = 100MHz / 128 / 781 ≈ 1kHz

  // PWM 시작 (Period: 781, Pulse: 390)
  if (HAL_LPTIM_PWM_Start(&hlptim2, 781, 390) != HAL_OK)
  {
    Error_Handler();
  }
}
```

### 2. DMAMUX 요청 생성기 설정

```c
void DMAMUX_RequestGen_Init(void)
{
  HAL_DMAEx_MuxRequestGeneratorConfigTypeDef pRequestGeneratorConfig = {0};

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // DMAMUX 요청 생성기 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  pRequestGeneratorConfig.SignalID = HAL_DMAMUX2_REQ_GEN_LPTIM2_OUT;  // 트리거: LPTIM2 OUT
  pRequestGeneratorConfig.Polarity = HAL_DMAMUX_REQ_GEN_RISING;       // 상승 엣지
  pRequestGeneratorConfig.RequestNumber = 1;                          // 트리거당 요청 수

  // BDMA 채널 0에 요청 생성기 연결
  if (HAL_DMAEx_ConfigMuxRequestGenerator(&hbdma_memtomem_dma2_stream0,
                                           &pRequestGeneratorConfig) != HAL_OK)
  {
    Error_Handler();
  }

  // 요청 생성기 활성화
  if (HAL_DMAEx_EnableMuxRequestGenerator(&hbdma_memtomem_dma2_stream0) != HAL_OK)
  {
    Error_Handler();
  }
}
```

### 3. BDMA 설정

```c
// LED 패턴 버퍼 (SRAM4에 배치)
uint32_t led_pattern[4] __attribute__((section(".sram4"))) = {
  0x00001000,  // LED OFF (PI12 = 0)
  0x10001000,  // LED ON  (PI12 = 1)
  0x00001000,  // LED OFF
  0x10001000   // LED ON
};

void BDMA_Init(void)
{
  // BDMA 클럭 활성화
  __HAL_RCC_BDMA_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // BDMA 채널 0 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  hbdma_memtomem_dma2_stream0.Instance = BDMA_Channel0;

  // 요청: DMAMUX2 Request Generator 0
  hbdma_memtomem_dma2_stream0.Init.Request = BDMA_REQUEST_GENERATOR0;

  // 방향: 메모리 -> 주변장치 (메모리)
  hbdma_memtomem_dma2_stream0.Init.Direction = DMA_MEMORY_TO_MEMORY;

  // 주변장치 (목적지): GPIOI ODR
  hbdma_memtomem_dma2_stream0.Init.PeriphInc = DMA_PINC_DISABLE;
  hbdma_memtomem_dma2_stream0.Init.PeriphDataAlignment = DMA_PDATAALIGN_WORD;

  // 메모리 (소스): SRAM4 버퍼
  hbdma_memtomem_dma2_stream0.Init.MemInc = DMA_MINC_ENABLE;
  hbdma_memtomem_dma2_stream0.Init.MemDataAlignment = DMA_MDATAALIGN_WORD;

  // 모드: 순환
  hbdma_memtomem_dma2_stream0.Init.Mode = DMA_CIRCULAR;

  // 우선순위
  hbdma_memtomem_dma2_stream0.Init.Priority = DMA_PRIORITY_HIGH;

  if (HAL_DMA_Init(&hbdma_memtomem_dma2_stream0) != HAL_OK)
  {
    Error_Handler();
  }

  // BDMA 인터럽트 활성화 (선택 사항)
  HAL_NVIC_SetPriority(BDMA_Channel0_IRQn, 3, 0);
  HAL_NVIC_EnableIRQ(BDMA_Channel0_IRQn);
}
```

### 4. GPIO 초기화

```c
void GPIO_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  // GPIOI 클럭 활성화
  __HAL_RCC_GPIOI_CLK_ENABLE();

  // PI12 (LED1) 설정
  GPIO_InitStruct.Pin = GPIO_PIN_12;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  HAL_GPIO_Init(GPIOI, &GPIO_InitStruct);

  // 초기 상태: OFF
  HAL_GPIO_WritePin(GPIOI, GPIO_PIN_12, GPIO_PIN_RESET);
}
```

### 5. BDMA 전송 시작

```c
void Start_BDMA_Transfer(void)
{
  // BDMA 전송 시작
  // 소스: led_pattern (SRAM4)
  // 목적지: GPIOI->ODR
  // 크기: 4 워드

  if (HAL_DMA_Start(&hbdma_memtomem_dma2_stream0,
                    (uint32_t)led_pattern,           // 소스
                    (uint32_t)&GPIOI->ODR,           // 목적지
                    4) != HAL_OK)                    // 전송 크기
  {
    Error_Handler();
  }

  printf("BDMA transfer started\n");
  printf("LED will blink at 1Hz\n");
}
```

### 6. 메인 함수

```c
int main(void)
{
  // HAL 초기화
  HAL_Init();

  // 시스템 클럭 설정
  SystemClock_Config();

  // D3 도메인 클럭 활성화
  __HAL_RCC_D3SRAM1_CLK_ENABLE();  // SRAM4

  // 주변장치 초기화
  GPIO_Init();
  LPTIM2_Init();
  BDMA_Init();
  DMAMUX_RequestGen_Init();

  // BDMA 전송 시작
  Start_BDMA_Transfer();

  // 무한 루프
  while (1)
  {
    // BDMA와 DMAMUX가 자동으로 LED 제어
    // CPU는 저전력 모드로 진입 가능

    HAL_Delay(1000);

    // 전송 상태 확인 (선택 사항)
    if (HAL_DMA_GetState(&hbdma_memtomem_dma2_stream0) == HAL_DMA_STATE_ERROR)
    {
      printf("DMA Error!\n");
      Error_Handler();
    }
  }
}
```

## DMAMUX 요청 생성기 상세

### 트리거 소스

STM32H745의 DMAMUX2는 다음 트리거 소스를 지원합니다:

```c
// DMAMUX2 요청 생성기 트리거 소스
#define HAL_DMAMUX2_REQ_GEN_DMAMUX2_CH0_EVT   0   // DMAMUX2 채널 0 이벤트
#define HAL_DMAMUX2_REQ_GEN_DMAMUX2_CH1_EVT   1   // DMAMUX2 채널 1 이벤트
#define HAL_DMAMUX2_REQ_GEN_DMAMUX2_CH2_EVT   2   // DMAMUX2 채널 2 이벤트
#define HAL_DMAMUX2_REQ_GEN_LPTIM2_OUT        3   // LPTIM2 출력
#define HAL_DMAMUX2_REQ_GEN_LPTIM3_OUT        4   // LPTIM3 출력
#define HAL_DMAMUX2_REQ_GEN_LPTIM4_OUT        5   // LPTIM4 출력
#define HAL_DMAMUX2_REQ_GEN_LPTIM5_OUT        6   // LPTIM5 출력
#define HAL_DMAMUX2_REQ_GEN_I2C4_IT_EVT       7   // I2C4 이벤트
#define HAL_DMAMUX2_REQ_GEN_SPI6_IT           8   // SPI6 인터럽트
#define HAL_DMAMUX2_REQ_GEN_LPUART1_RX_WKUP   9   // LPUART1 RX 웨이크업
#define HAL_DMAMUX2_REQ_GEN_LPUART1_TX_WKUP   10  // LPUART1 TX 웨이크업
#define HAL_DMAMUX2_REQ_GEN_LPTIM2_WKUP       11  // LPTIM2 웨이크업
#define HAL_DMAMUX2_REQ_GEN_LPTIM3_WKUP       12  // LPTIM3 웨이크업
#define HAL_DMAMUX2_REQ_GEN_LPTIM4_WKUP       13  // LPTIM4 웨이크업
#define HAL_DMAMUX2_REQ_GEN_LPTIM5_WKUP       14  // LPTIM5 웨이크업
```

### 요청 수 제어

```c
// 단일 트리거당 여러 DMA 요청 생성
pRequestGeneratorConfig.RequestNumber = 4;  // 트리거당 4번 전송

// 예: LPTIM2 OUT 1회 -> BDMA 4회 전송
```

### 극성 설정

```c
// 상승 엣지에서 트리거
pRequestGeneratorConfig.Polarity = HAL_DMAMUX_REQ_GEN_RISING;

// 하강 엣지에서 트리거
pRequestGeneratorConfig.Polarity = HAL_DMAMUX_REQ_GEN_FALLING;

// 양 엣지에서 트리거
pRequestGeneratorConfig.Polarity = HAL_DMAMUX_REQ_GEN_RISING_FALLING;
```

## 메모리 구성

### SRAM4 (D3 도메인)

```
D3 Domain SRAM (SRAM4)
0x38000000  ┌─────────────────────────┐
            │                         │
            │  LED Pattern Buffer     │
            │  (16 bytes)             │
            │                         │
0x38000010  ├─────────────────────────┤
            │                         │
            │  Available SRAM4        │
            │  (~64KB)                │
            │                         │
0x38010000  └─────────────────────────┘

주의: SRAM4는 BDMA만 액세스 가능
```

### 링커 스크립트 설정

```ld
/* STM32H745xx_flash.ld */

MEMORY
{
  /* ... 기존 메모리 영역 ... */

  SRAM4 (xrw)  : ORIGIN = 0x38000000, LENGTH = 64K
}

SECTIONS
{
  .sram4 :
  {
    . = ALIGN(4);
    *(.sram4)
    . = ALIGN(4);
  } >SRAM4
}
```

## 고급 기능

### 1. 다중 요청 생성기 사용

```c
// 요청 생성기 0: LPTIM2 OUT
// 요청 생성기 1: LPTIM3 OUT

DMA_HandleTypeDef hbdma_ch0;
DMA_HandleTypeDef hbdma_ch1;

void Multi_RequestGen_Init(void)
{
  HAL_DMAEx_MuxRequestGeneratorConfigTypeDef gen0_config = {0};
  HAL_DMAEx_MuxRequestGeneratorConfigTypeDef gen1_config = {0};

  // 요청 생성기 0 설정 (LPTIM2, 1Hz)
  gen0_config.SignalID = HAL_DMAMUX2_REQ_GEN_LPTIM2_OUT;
  gen0_config.Polarity = HAL_DMAMUX_REQ_GEN_RISING;
  gen0_config.RequestNumber = 1;
  HAL_DMAEx_ConfigMuxRequestGenerator(&hbdma_ch0, &gen0_config);
  HAL_DMAEx_EnableMuxRequestGenerator(&hbdma_ch0);

  // 요청 생성기 1 설정 (LPTIM3, 10Hz)
  gen1_config.SignalID = HAL_DMAMUX2_REQ_GEN_LPTIM3_OUT;
  gen1_config.Polarity = HAL_DMAMUX_REQ_GEN_RISING;
  gen1_config.RequestNumber = 1;
  HAL_DMAEx_ConfigMuxRequestGenerator(&hbdma_ch1, &gen1_config);
  HAL_DMAEx_EnableMuxRequestGenerator(&hbdma_ch1);
}
```

### 2. 동적 패턴 변경

```c
// 런타임에 LED 패턴 변경
void Change_LED_Pattern(uint8_t pattern_id)
{
  // BDMA 중지
  HAL_DMA_Abort(&hbdma_memtomem_dma2_stream0);

  // 패턴 변경
  switch (pattern_id)
  {
    case 0:  // 빠른 깜박임
      led_pattern[0] = 0x00001000;
      led_pattern[1] = 0x10001000;
      led_pattern[2] = 0x00001000;
      led_pattern[3] = 0x10001000;
      break;

    case 1:  // 느린 깜박임
      led_pattern[0] = 0x00001000;
      led_pattern[1] = 0x00001000;
      led_pattern[2] = 0x10001000;
      led_pattern[3] = 0x10001000;
      break;

    case 2:  // 항상 켜짐
      led_pattern[0] = 0x10001000;
      led_pattern[1] = 0x10001000;
      led_pattern[2] = 0x10001000;
      led_pattern[3] = 0x10001000;
      break;
  }

  // 캐시 클린 (필요시)
  SCB_CleanDCache_by_Addr((uint32_t*)led_pattern, sizeof(led_pattern));

  // BDMA 재시작
  HAL_DMA_Start(&hbdma_memtomem_dma2_stream0,
                (uint32_t)led_pattern,
                (uint32_t)&GPIOI->ODR,
                4);
}
```

### 3. BDMA 콜백 활용

```c
// BDMA 전송 완료 콜백
void HAL_DMA_XferCpltCallback(DMA_HandleTypeDef *hdma)
{
  if (hdma->Instance == BDMA_Channel0)
  {
    // 한 사이클 완료
    static uint32_t cycle_count = 0;
    cycle_count++;

    if (cycle_count % 10 == 0)
    {
      printf("BDMA completed %lu cycles\n", cycle_count);
    }
  }
}

// BDMA 인터럽트 핸들러
void BDMA_Channel0_IRQHandler(void)
{
  HAL_DMA_IRQHandler(&hbdma_memtomem_dma2_stream0);
}
```

## 저전력 모드 활용

### D3 도메인에서 독립 동작

```c
// CM7과 D1/D2 도메인을 저전력 모드로 전환
// D3 도메인만 활성화 상태 유지

void Enter_LowPower_Mode(void)
{
  // D1 도메인을 Stop 모드로 전환
  HAL_PWREx_EnterSTOPMode(PWR_MAINREGULATOR_ON,
                          PWR_STOPENTRY_WFI,
                          PWR_D1_DOMAIN);

  // D3 도메인은 활성 상태 유지
  // BDMA와 LPTIM2는 계속 동작
}

// D3에서 동작하는 주변장치:
// - BDMA
// - LPTIM2/3/4/5
// - LPUART1
// - I2C4
// - SPI6
// - SRAM4
```

## 빌드 및 실행

### 테스트 절차

```
1. 보드 연결 및 플래싱
2. LED1 (PI12) 관찰
3. 1Hz로 깜박임 확인
4. 오실로스코프로 LPTIM2 OUT 신호 확인 (선택)
```

### 예상 동작

```
시간 (초):  0    1    2    3    4
LED 상태:  OFF  ON   OFF  ON   OFF
LPTIM2:    ─┐  ┌─┐  ┌─┐  ┌─┐  ┌─┐
           └──┘  └──┘  └──┘  └──┘
BDMA:      ───┴───┴───┴───┴───┴───
```

## 트러블슈팅

### LED가 깜박이지 않는 경우

```c
// 1. LPTIM2 동작 확인
if (hlptim2.Instance->CR & LPTIM_CR_ENABLE)
{
  printf("LPTIM2 enabled\n");
}
else
{
  printf("LPTIM2 disabled!\n");
  HAL_LPTIM_PWM_Start(&hlptim2, 781, 390);
}

// 2. BDMA 활성화 확인
if (BDMA_Channel0->CCR & DMA_CCR_EN)
{
  printf("BDMA enabled\n");
}
else
{
  printf("BDMA disabled!\n");
}

// 3. DMAMUX 요청 생성기 확인
uint32_t rgcr = DMAMUX2_RequestGenerator0->RGCR;
printf("DMAMUX2 RG0 RGCR: 0x%08lX\n", rgcr);
if (rgcr & DMAMUX_RGxCR_GE)
{
  printf("Request generator enabled\n");
}
```

### SRAM4 액세스 오류

```c
// SRAM4 클럭 활성화 확인
if (!(RCC->AHB2ENR & RCC_AHB2ENR_D2SRAM1EN))
{
  printf("SRAM4 clock disabled!\n");
  __HAL_RCC_D3SRAM1_CLK_ENABLE();
}

// MPU 설정 확인 (필요시)
```

## 참고 자료

- **RM0399**: STM32H745 Reference Manual
  - Chapter 14: BDMA
  - Chapter 16: DMAMUX
  - Chapter 48: LPTIM
- **AN5224**: STM32H7x5/x7 BDMA and DMAMUX
- **예제 코드**: `STM32Cube_FW_H7/Projects/STM32H745I-DISCO/Examples/DMA/DMAMUX_RequestGen`

## 관련 예제

- **DMA_DMAMUX**: 일반 DMA와 DMAMUX
- **LPTIM_PWM**: LPTIM PWM 출력
- **PWR_STANDBY**: 저전력 모드
