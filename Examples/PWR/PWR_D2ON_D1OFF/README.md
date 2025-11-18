# PWR_D2ON_D1OFF - D2 도메인 ON, D1 도메인 OFF

## 개요

이 예제는 STM32H745 듀얼 코어 MCU에서 D2 도메인(Cortex-M4)만 활성화하고 D1 도메인(Cortex-M7)을 STANDBY 모드로 유지하는 방법을 보여줍니다. M4의 중간 성능으로 충분하고 저전력이 중요한 애플리케이션에 적합하며, D1에 비해 약 70mA의 전력을 절감할 수 있습니다.

## 하드웨어 요구사항

- **STM32H745I-DISCO 보드**
- **LED1**: PI12 - D2/M4 활성 상태
- **LED2**: PI13 - 작업 표시
- **USER 버튼**: PC13 - 인터럽트
- **전류계**: 전력 측정용
- **UART3**: 디버깅 (D2 도메인)

## STM32H745 도메인 아키텍처

### 도메인 구조 (D2 활성)

```
┌─────────────────────────────────────────────────────────┐
│              STM32H745 Dual Core MCU                     │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────────────────────────────────────┐  ★ OFF  │
│  │         D1 Domain (CPU Domain)             │         │
│  ├────────────────────────────────────────────┤         │
│  │  • Cortex-M7 @ 480 MHz (STANDBY)           │         │
│  │  • AXI SRAM: 512 KB (Retention optional)   │         │
│  │  • DTCM/ITCM: 192 KB (OFF)                 │         │
│  │  • Peripherals (All OFF)                   │         │
│  │    - DMA1, DMA2, MDMA                      │         │
│  │    - Ethernet, USB OTG HS                  │         │
│  │    - DCMI, JPEG                            │         │
│  └────────────────────────────────────────────┘         │
│                                                          │
│  ┌────────────────────────────────────────────┐  ★ ON   │
│  │         D2 Domain (Peripheral Domain)      │         │
│  ├────────────────────────────────────────────┤         │
│  │  • Cortex-M4 @ 240 MHz                     │         │
│  │    - 32-bit RISC processor                 │         │
│  │    - Single precision FPU                  │         │
│  │    - MPU with 8 regions                    │         │
│  │    - No cache                              │         │
│  │                                            │         │
│  │  • Memory                                  │         │
│  │    - SRAM1: 128 KB                         │         │
│  │    - SRAM2: 128 KB                         │         │
│  │    - SRAM3: 32 KB                          │         │
│  │    - Total: 288 KB                         │         │
│  │                                            │         │
│  │  • Peripherals (Rich Set)                  │         │
│  │    - ADC1, ADC2 (16-bit, dual mode)        │         │
│  │    - DAC1 (dual channel)                   │         │
│  │    - DFSDM (4 filters, 8 channels)         │         │
│  │    - TIM2-7 (General purpose timers)       │         │
│  │    - TIM12-17 (Advanced/basic timers)      │         │
│  │    - SPI1-5 (up to 150 MHz)                │         │
│  │    - I2C1-3 (1 MHz Fast mode+)             │         │
│  │    - USART1-6, UART7-8                     │         │
│  │    - FDCAN1-2 (CAN FD)                     │         │
│  │    - SDMMC1-2 (SD/MMC card)                │         │
│  │    - SAI1-2 (Serial Audio)                 │         │
│  │    - BDMA (8 channels)                     │         │
│  └────────────────────────────────────────────┘         │
│                                                          │
│  ┌────────────────────────────────────────────┐         │
│  │      D3 Domain (Autonomous Domain)         │         │
│  │  • RTC, IWDG, LPTIM, LPUART                │         │
│  │  • I2C4, SPI6, BDMA                        │         │
│  └────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────┘
```

### 전력 모드 비교

```
구성                  │ D1 (M7)  │ D2 (M4)  │ 전력 소비  │ 성능
────────────────────┼─────────┼─────────┼───────────┼──────────
양쪽 코어 RUN        │ RUN      │ RUN      │ ~250 mA    │ 최대
D1 ON, D2 OFF       │ RUN      │ STANDBY  │ ~170 mA    │ M7 고성능
D2 ON, D1 OFF ★     │ STANDBY  │ RUN      │ ~100 mA    │ M4 중성능
양쪽 코어 STOP       │ STOP     │ STOP     │ ~100 μA    │ 저전력
양쪽 코어 STANDBY    │ STANDBY  │ STANDBY  │ ~2.4 μA    │ 최저 전력
```

## 주요 기능

### D2 도메인만 사용하는 이유

**장점**

1. **저전력**
   - M7 대비 ~70mA 절감 (42% 감소)
   - 배터리 수명 2배 연장
   - 발열 감소

2. **충분한 성능**
   - M4 @ 240MHz
   - 대부분의 제어 작업에 충분
   - 실시간 OS 실행 가능

3. **풍부한 주변장치**
   - ADC, DAC, DFSDM
   - 모든 통신 인터페이스
   - 타이머 다수

4. **간단한 구조**
   - 캐시 없음 (예측 가능한 타이밍)
   - 단일 코어 디버깅
   - 메모리 관리 단순

### 사용 사례

1. **센서 허브**
   - 다중 센서 읽기
   - 데이터 필터링
   - 주기적 전송

2. **모터 제어**
   - PWM 생성
   - 엔코더 읽기
   - PID 제어

3. **통신 게이트웨이**
   - UART-CAN 변환
   - ModBus RTU/TCP
   - LoRaWAN

4. **배터리 구동 장치**
   - 스마트 미터
   - 휴대용 기기
   - IoT 노드

## 동작 원리

### 부팅 시퀀스 (M4 전용)

```
┌─────────────────┐
│  Power-On Reset │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Boot CM7 First  │  M7 항상 먼저 부팅
│ (Default)       │  Flash: 0x08000000
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ M7 Boot Code    │  최소 초기화만 수행
│ - Enable CM4    │  - 클럭 설정
│ - Jump to M4    │  - M4 부팅
│ - Enter STANDBY │  - M7 STANDBY
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Boot CM4        │  M4 메인 코드 실행
│ Flash: 0x0810   │  0000 (Sector 8)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ M4 main()       │
│                 │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ M4 Running      │  ★ D2 도메인만 활성
│ M7 Stopped      │    D1 도메인 STANDBY
│                 │
│ D1: STANDBY     │    전력: ~100 mA
│ D2: RUN         │    성능: 240 MHz
└─────────────────┘
```

### 메모리 맵 (M4 관점)

```
┌─────────────────────────────────────────────┐
│           Memory Map (M4 Active)            │
├─────────────────────────────────────────────┤
│                                             │
│  M4 접근 가능:                               │
│  ┌───────────────────────────────────┐     │
│  │ M4 Flash:      0x0810_0000 (1MB)  │ ✓   │
│  │ M7 Flash:      0x0800_0000 (1MB)  │ ✓*  │
│  │                                   │     │
│  │ SRAM1:         0x3000_0000 (128KB)│ ✓   │
│  │ SRAM2:         0x3002_0000 (128KB)│ ✓   │
│  │ SRAM3:         0x3004_0000 (32KB) │ ✓   │
│  │                                   │     │
│  │ AXI SRAM:      0x2400_0000 (512KB)│ ✓** │
│  │ SRAM4 (D3):    0x3800_0000 (64KB) │ ✓   │
│  └───────────────────────────────────┘     │
│                                             │
│  * M7 Flash는 읽기 전용으로 접근 가능       │
│  ** AXI SRAM은 D1 전원 ON 시만 접근 가능   │
│                                             │
│  M4 접근 불가:                               │
│  ┌───────────────────────────────────┐     │
│  │ DTCM:          0x2000_0000        │ ✗   │
│  │ ITCM:          0x0000_0000        │ ✗   │
│  └───────────────────────────────────┘     │
└─────────────────────────────────────────────┘
```

## 코드 구현

### 1. M7 부트 코드 (최소 초기화)

```c
// M7 Flash: 0x08000000
// M7은 M4를 부팅하고 STANDBY로 진입

#include "stm32h7xx_hal.h"

#define CM4_BOOT_ADDRESS  0x08100000  // M4 Flash 시작 주소

int main(void)
{
  // HAL 초기화 (최소)
  HAL_Init();

  // 시스템 클럭 설정
  SystemClock_Config();

  printf("\n========================================\n");
  printf("STM32H745 D2 Only Mode (M4 Active)\n");
  printf("M7 Boot Loader\n");
  printf("========================================\n\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // M4 부팅 준비
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  printf("Enabling CM4...\n");

  // M4 리셋 해제
  __HAL_RCC_CM4_RELEASE_RESET();

  // M4 부팅 주소 설정
  HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);

  // M4 시작
  __HAL_SYSCFG_CM4_BOOT_ADDR(CM4_BOOT_ADDRESS >> 16);

  printf("CM4 started at 0x%08X\n", CM4_BOOT_ADDRESS);

  // 짧은 지연 (M4 부팅 대기)
  HAL_Delay(100);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // D1 도메인 STANDBY 진입
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  printf("M7 entering STANDBY...\n");
  HAL_Delay(100);  // 메시지 출력 완료 대기

  // D1 주변장치 클럭 비활성화
  RCC->AHB3ENR = 0;
  RCC->AHB4ENR = 0;

  // PWR 클럭 활성화
  __HAL_RCC_PWR_CLK_ENABLE();

  // 플래그 클리어
  __HAL_PWR_CLEAR_FLAG(PWR_FLAG_STOP);
  __HAL_PWR_CLEAR_FLAG(PWR_FLAG_SB);

  // D1 도메인 STANDBY 진입
  HAL_PWREx_EnterSTANDBYMode(PWR_D1_DOMAIN);

  // 여기는 실행되지 않음
  while (1);
}
```

### 2. M4 메인 코드

```c
// M4 Flash: 0x08100000 (Sector 8)
// M4 메인 애플리케이션

#include "stm32h7xx_hal.h"

// M4 전용 메모리 (D2 SRAM)
uint8_t sensor_buffer[1024] __attribute__((section(".sram2")));
uint32_t m4_counter = 0;

int main(void)
{
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // M4 HAL 초기화
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  HAL_Init();

  // M4 클럭 설정 (240 MHz)
  SystemClock_Config_M4();

  printf("\n========================================\n");
  printf("STM32H745 Cortex-M4 Running\n");
  printf("========================================\n\n");

  printf("System Configuration:\n");
  printf("  • M4 Core: RUNNING @ 240 MHz\n");
  printf("  • M7 Core: STANDBY\n");
  printf("  • D1 Domain: STANDBY\n");
  printf("  • D2 Domain: ACTIVE\n");
  printf("  • Power: ~100 mA\n\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // D2 주변장치 초기화
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  GPIO_Init_M4();
  UART_Init_M4();
  ADC_Init_M4();
  TIM_Init_M4();

  printf("M4 peripherals initialized\n\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // M4 메인 루프
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  while (1)
  {
    // M4 작업 수행
    Perform_M4_Tasks();

    // LED 토글
    HAL_GPIO_TogglePin(GPIOI, GPIO_PIN_12);

    HAL_Delay(1000);

    m4_counter++;
    printf("[M4] Running... count=%lu\n", m4_counter);
  }
}
```

### 3. M4 클럭 설정

```c
void SystemClock_Config_M4(void)
{
  RCC_ClkInitTypeDef RCC_ClkInitStruct;
  RCC_OscInitTypeDef RCC_OscInitStruct;

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // PLL 설정 (M4 최적화)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // HSE 25 MHz → PLL → 240 MHz

  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
  RCC_OscInitStruct.HSEState = RCC_HSE_ON;
  RCC_OscInitStruct.HSIState = RCC_HSI_OFF;
  RCC_OscInitStruct.CSIState = RCC_CSI_OFF;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;

  // PLL1: M4용 (240 MHz)
  // VCO = 25 MHz / 5 * 96 = 480 MHz
  // SYSCLK = 480 MHz / 2 = 240 MHz
  RCC_OscInitStruct.PLL.PLLM = 5;
  RCC_OscInitStruct.PLL.PLLN = 96;
  RCC_OscInitStruct.PLL.PLLP = 2;  // M4 클럭
  RCC_OscInitStruct.PLL.PLLQ = 4;
  RCC_OscInitStruct.PLL.PLLR = 2;
  RCC_OscInitStruct.PLL.PLLRGE = RCC_PLL1VCIRANGE_2;
  RCC_OscInitStruct.PLL.PLLVCOSEL = RCC_PLL1VCOWIDE;
  RCC_OscInitStruct.PLL.PLLFRACN = 0;

  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 도메인 클럭 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  RCC_ClkInitStruct.ClockType = (RCC_CLOCKTYPE_SYSCLK |
                                 RCC_CLOCKTYPE_HCLK |
                                 RCC_CLOCKTYPE_D1PCLK1 |
                                 RCC_CLOCKTYPE_PCLK1 |
                                 RCC_CLOCKTYPE_PCLK2 |
                                 RCC_CLOCKTYPE_D3PCLK1);

  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
  RCC_ClkInitStruct.SYSCLKDivider = RCC_SYSCLK_DIV1;

  // D2 Domain (M4)
  RCC_ClkInitStruct.AHBCLKDivider = RCC_HCLK_DIV1;       // 240 MHz
  RCC_ClkInitStruct.APB1CLKDivider = RCC_APB1_DIV2;      // 120 MHz
  RCC_ClkInitStruct.APB2CLKDivider = RCC_APB2_DIV2;      // 120 MHz

  // D1 Domain (OFF이지만 설정 필요)
  RCC_ClkInitStruct.D1CPREDivider = RCC_D1CPRE_DIV1;

  // D3 Domain
  RCC_ClkInitStruct.D3PPREDivider = RCC_D3PPRE_DIV2;

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_2) != HAL_OK)
  {
    Error_Handler();
  }

  printf("System Clock configured:\n");
  printf("  • SYSCLK: %lu Hz\n", HAL_RCC_GetSysClockFreq());
  printf("  • HCLK (AHB): %lu Hz\n", HAL_RCC_GetHCLKFreq());
  printf("  • APB1: %lu Hz\n", HAL_RCC_GetPCLK1Freq());
  printf("  • APB2: %lu Hz\n", HAL_RCC_GetPCLK2Freq());
}
```

### 4. D2 주변장치 초기화

```c
// ADC 초기화 (D2 domain)
ADC_HandleTypeDef hadc1;

void ADC_Init_M4(void)
{
  printf("Initializing ADC1 (D2 domain)...\n");

  __HAL_RCC_ADC12_CLK_ENABLE();

  hadc1.Instance = ADC1;
  hadc1.Init.ClockPrescaler = ADC_CLOCK_ASYNC_DIV4;
  hadc1.Init.Resolution = ADC_RESOLUTION_16B;
  hadc1.Init.ScanConvMode = ADC_SCAN_DISABLE;
  hadc1.Init.EOCSelection = ADC_EOC_SINGLE_CONV;
  hadc1.Init.LowPowerAutoWait = DISABLE;
  hadc1.Init.ContinuousConvMode = DISABLE;
  hadc1.Init.NbrOfConversion = 1;
  hadc1.Init.DiscontinuousConvMode = DISABLE;
  hadc1.Init.ExternalTrigConv = ADC_SOFTWARE_START;
  hadc1.Init.ConversionDataManagement = ADC_CONVERSIONDATA_DR;
  hadc1.Init.Overrun = ADC_OVR_DATA_OVERWRITTEN;
  hadc1.Init.OversamplingMode = DISABLE;

  if (HAL_ADC_Init(&hadc1) != HAL_OK)
  {
    Error_Handler();
  }

  // ADC 캘리브레이션
  HAL_ADCEx_Calibration_Start(&hadc1, ADC_CALIB_OFFSET, ADC_SINGLE_ENDED);

  printf("  • ADC1 initialized (16-bit)\n");
}

// Timer 초기화 (PWM)
TIM_HandleTypeDef htim2;

void TIM_Init_M4(void)
{
  printf("Initializing TIM2 (PWM)...\n");

  __HAL_RCC_TIM2_CLK_ENABLE();

  // TIM2: 1kHz PWM
  htim2.Instance = TIM2;
  htim2.Init.Prescaler = 120 - 1;  // 120 MHz / 120 = 1 MHz
  htim2.Init.CounterMode = TIM_COUNTERMODE_UP;
  htim2.Init.Period = 1000 - 1;    // 1 MHz / 1000 = 1 kHz
  htim2.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
  htim2.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_ENABLE;

  if (HAL_TIM_PWM_Init(&htim2) != HAL_OK)
  {
    Error_Handler();
  }

  // PWM 채널 설정
  TIM_OC_InitTypeDef sConfigOC = {0};
  sConfigOC.OCMode = TIM_OCMODE_PWM1;
  sConfigOC.Pulse = 500;  // 50% duty
  sConfigOC.OCPolarity = TIM_OCPOLARITY_HIGH;
  sConfigOC.OCFastMode = TIM_OCFAST_DISABLE;

  HAL_TIM_PWM_ConfigChannel(&htim2, &sConfigOC, TIM_CHANNEL_1);

  // PWM 시작
  HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1);

  printf("  • TIM2 PWM started (1 kHz, 50%%)\n");
}

// UART 초기화 (D2 domain)
UART_HandleTypeDef huart3;

void UART_Init_M4(void)
{
  printf("Initializing UART3 (D2 domain)...\n");

  __HAL_RCC_USART3_CLK_ENABLE();

  // GPIO 설정 (PD8: TX, PD9: RX)
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  __HAL_RCC_GPIOD_CLK_ENABLE();

  GPIO_InitStruct.Pin = GPIO_PIN_8 | GPIO_PIN_9;
  GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  GPIO_InitStruct.Alternate = GPIO_AF7_USART3;
  HAL_GPIO_Init(GPIOD, &GPIO_InitStruct);

  // UART3 설정
  huart3.Instance = USART3;
  huart3.Init.BaudRate = 115200;
  huart3.Init.WordLength = UART_WORDLENGTH_8B;
  huart3.Init.StopBits = UART_STOPBITS_1;
  huart3.Init.Parity = UART_PARITY_NONE;
  huart3.Init.Mode = UART_MODE_TX_RX;
  huart3.Init.HwFlowCtl = UART_HWCONTROL_NONE;
  huart3.Init.OverSampling = UART_OVERSAMPLING_16;

  if (HAL_UART_Init(&huart3) != HAL_OK)
  {
    Error_Handler();
  }

  printf("  • UART3 initialized (115200 baud)\n");
}
```

### 5. DFSDM 사용 (M4 전용)

```c
// Digital Filter for Sigma-Delta Modulators
// M4에서만 사용 가능한 고급 ADC 기능

DFSDM_Filter_HandleTypeDef hdfsdm1_filter0;
DFSDM_Channel_HandleTypeDef hdfsdm1_channel0;

void DFSDM_Init_M4(void)
{
  printf("Initializing DFSDM (D2 domain)...\n");

  __HAL_RCC_DFSDM1_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // DFSDM 채널 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  hdfsdm1_channel0.Instance = DFSDM1_Channel0;
  hdfsdm1_channel0.Init.OutputClock.Activation = ENABLE;
  hdfsdm1_channel0.Init.OutputClock.Selection = DFSDM_CHANNEL_OUTPUT_CLOCK_SYSTEM;
  hdfsdm1_channel0.Init.OutputClock.Divider = 2;
  hdfsdm1_channel0.Init.Input.Multiplexer = DFSDM_CHANNEL_EXTERNAL_INPUTS;
  hdfsdm1_channel0.Init.Input.DataPacking = DFSDM_CHANNEL_STANDARD_MODE;
  hdfsdm1_channel0.Init.Input.Pins = DFSDM_CHANNEL_SAME_CHANNEL_PINS;
  hdfsdm1_channel0.Init.SerialInterface.Type = DFSDM_CHANNEL_SPI_RISING;
  hdfsdm1_channel0.Init.SerialInterface.SpiClock = DFSDM_CHANNEL_SPI_CLOCK_INTERNAL;
  hdfsdm1_channel0.Init.Awd.FilterOrder = DFSDM_CHANNEL_FASTSINC_ORDER;
  hdfsdm1_channel0.Init.Awd.Oversampling = 1;
  hdfsdm1_channel0.Init.Offset = 0;
  hdfsdm1_channel0.Init.RightBitShift = 0;

  if (HAL_DFSDM_ChannelInit(&hdfsdm1_channel0) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // DFSDM 필터 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  hdfsdm1_filter0.Instance = DFSDM1_Filter0;
  hdfsdm1_filter0.Init.RegularParam.Trigger = DFSDM_FILTER_SW_TRIGGER;
  hdfsdm1_filter0.Init.RegularParam.FastMode = ENABLE;
  hdfsdm1_filter0.Init.RegularParam.DmaMode = ENABLE;
  hdfsdm1_filter0.Init.FilterParam.SincOrder = DFSDM_FILTER_SINC3_ORDER;
  hdfsdm1_filter0.Init.FilterParam.Oversampling = 256;
  hdfsdm1_filter0.Init.FilterParam.IntOversampling = 1;

  if (HAL_DFSDM_FilterInit(&hdfsdm1_filter0) != HAL_OK)
  {
    Error_Handler();
  }

  // 채널을 필터에 연결
  HAL_DFSDM_FilterConfigRegChannel(&hdfsdm1_filter0,
                                    DFSDM_CHANNEL_0,
                                    DFSDM_CONTINUOUS_CONV_ON);

  printf("  • DFSDM initialized (Sigma-Delta ADC)\n");
}

// DFSDM 데이터 읽기
int32_t Read_DFSDM_Value(void)
{
  int32_t value = 0;

  // Regular conversion 시작
  HAL_DFSDM_FilterRegularStart(&hdfsdm1_filter0);

  // 변환 완료 대기
  HAL_DFSDM_FilterPollForRegConversion(&hdfsdm1_filter0, 1000);

  // 값 읽기
  value = HAL_DFSDM_FilterGetRegularValue(&hdfsdm1_filter0, NULL);

  // 정지
  HAL_DFSDM_FilterRegularStop(&hdfsdm1_filter0);

  return value;
}
```

## 고급 기능

### 1. 다중 센서 데이터 수집

```c
// ADC1, ADC2 듀얼 모드로 동시 샘플링

#define SAMPLE_SIZE  1024

uint16_t adc1_buffer[SAMPLE_SIZE];
uint16_t adc2_buffer[SAMPLE_SIZE];

void Dual_ADC_Sampling(void)
{
  ADC_HandleTypeDef hadc1, hadc2;
  DMA_HandleTypeDef hdma_adc1;

  printf("Dual ADC sampling...\n");

  // ADC1, ADC2 초기화
  __HAL_RCC_ADC12_CLK_ENABLE();

  // ADC1 설정
  hadc1.Instance = ADC1;
  hadc1.Init.ClockPrescaler = ADC_CLOCK_ASYNC_DIV4;
  hadc1.Init.Resolution = ADC_RESOLUTION_16B;
  hadc1.Init.ContinuousConvMode = ENABLE;
  hadc1.Init.ConversionDataManagement = ADC_CONVERSIONDATA_DMA_CIRCULAR;
  HAL_ADC_Init(&hadc1);

  // ADC2 설정 (동일)
  hadc2.Instance = ADC2;
  hadc2.Init = hadc1.Init;
  HAL_ADC_Init(&hadc2);

  // 듀얼 모드 설정
  ADC_MultiModeTypeDef multimode = {0};
  multimode.Mode = ADC_DUALMODE_REGSIMULT;  // 동시 샘플링
  multimode.DualModeData = ADC_DUALMODEDATAFORMAT_32_10_BITS;
  HAL_ADCEx_MultiModeConfigChannel(&hadc1, &multimode);

  // DMA 설정
  __HAL_RCC_BDMA_CLK_ENABLE();
  hdma_adc1.Instance = BDMA_Channel0;
  hdma_adc1.Init.Request = BDMA_REQUEST_ADC1;
  hdma_adc1.Init.Direction = DMA_PERIPH_TO_MEMORY;
  hdma_adc1.Init.PeriphInc = DMA_PINC_DISABLE;
  hdma_adc1.Init.MemInc = DMA_MINC_ENABLE;
  hdma_adc1.Init.Mode = DMA_CIRCULAR;
  HAL_DMA_Init(&hdma_adc1);

  __HAL_LINKDMA(&hadc1, DMA_Handle, hdma_adc1);

  // ADC + DMA 시작
  HAL_ADCEx_MultiModeStart_DMA(&hadc1,
                                (uint32_t*)adc1_buffer,
                                SAMPLE_SIZE);

  printf("  • Dual ADC started (simultaneous sampling)\n");
  printf("  • Sample rate: %.0f kHz\n",
         (float)HAL_RCC_GetPCLK2Freq() / 4 / 256 / 1000);
}
```

### 2. CAN FD 통신

```c
// FDCAN (CAN with Flexible Data-rate)
// M4의 D2 도메인에서만 사용 가능

FDCAN_HandleTypeDef hfdcan1;

void FDCAN_Init_M4(void)
{
  printf("Initializing FDCAN1...\n");

  __HAL_RCC_FDCAN_CLK_ENABLE();

  // GPIO 설정 (PD0: RX, PD1: TX)
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  __HAL_RCC_GPIOD_CLK_ENABLE();

  GPIO_InitStruct.Pin = GPIO_PIN_0 | GPIO_PIN_1;
  GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
  GPIO_InitStruct.Alternate = GPIO_AF9_FDCAN1;
  HAL_GPIO_Init(GPIOD, &GPIO_InitStruct);

  // FDCAN 초기화
  hfdcan1.Instance = FDCAN1;
  hfdcan1.Init.FrameFormat = FDCAN_FRAME_FD_BRS;  // CAN FD with BRS
  hfdcan1.Init.Mode = FDCAN_MODE_NORMAL;
  hfdcan1.Init.AutoRetransmission = ENABLE;
  hfdcan1.Init.TransmitPause = DISABLE;
  hfdcan1.Init.ProtocolException = ENABLE;

  // Nominal bit rate: 500 kbps
  hfdcan1.Init.NominalPrescaler = 12;
  hfdcan1.Init.NominalSyncJumpWidth = 1;
  hfdcan1.Init.NominalTimeSeg1 = 13;
  hfdcan1.Init.NominalTimeSeg2 = 2;

  // Data bit rate: 2 Mbps
  hfdcan1.Init.DataPrescaler = 3;
  hfdcan1.Init.DataSyncJumpWidth = 1;
  hfdcan1.Init.DataTimeSeg1 = 13;
  hfdcan1.Init.DataTimeSeg2 = 2;

  hfdcan1.Init.MessageRAMOffset = 0;
  hfdcan1.Init.StdFiltersNbr = 1;
  hfdcan1.Init.ExtFiltersNbr = 0;
  hfdcan1.Init.RxFifo0ElmtsNbr = 2;
  hfdcan1.Init.RxFifo0ElmtSize = FDCAN_DATA_BYTES_64;
  hfdcan1.Init.RxFifo1ElmtsNbr = 0;
  hfdcan1.Init.TxEventsNbr = 0;
  hfdcan1.Init.TxBuffersNbr = 0;
  hfdcan1.Init.TxFifoQueueElmtsNbr = 2;
  hfdcan1.Init.TxFifoQueueMode = FDCAN_TX_FIFO_OPERATION;
  hfdcan1.Init.TxElmtSize = FDCAN_DATA_BYTES_64;

  if (HAL_FDCAN_Init(&hfdcan1) != HAL_OK)
  {
    Error_Handler();
  }

  // 필터 설정
  FDCAN_FilterTypeDef sFilterConfig;
  sFilterConfig.IdType = FDCAN_STANDARD_ID;
  sFilterConfig.FilterIndex = 0;
  sFilterConfig.FilterType = FDCAN_FILTER_MASK;
  sFilterConfig.FilterConfig = FDCAN_FILTER_TO_RXFIFO0;
  sFilterConfig.FilterID1 = 0x000;
  sFilterConfig.FilterID2 = 0x000;  // 모든 ID 수신

  HAL_FDCAN_ConfigFilter(&hfdcan1, &sFilterConfig);

  // 글로벌 필터 설정
  HAL_FDCAN_ConfigGlobalFilter(&hfdcan1,
                                FDCAN_ACCEPT_IN_RX_FIFO0,
                                FDCAN_ACCEPT_IN_RX_FIFO0,
                                FDCAN_REJECT_REMOTE_FRAMES,
                                FDCAN_REJECT_REMOTE_FRAMES);

  // FDCAN 시작
  HAL_FDCAN_Start(&hfdcan1);

  // RX FIFO 0 알림 활성화
  HAL_FDCAN_ActivateNotification(&hfdcan1,
                                  FDCAN_IT_RX_FIFO0_NEW_MESSAGE,
                                  0);

  printf("  • FDCAN1 initialized\n");
  printf("  • Nominal: 500 kbps\n");
  printf("  • Data: 2 Mbps\n");
}

// CAN FD 메시지 전송
void FDCAN_Transmit_Message(uint32_t id, uint8_t *data, uint32_t len)
{
  FDCAN_TxHeaderTypeDef TxHeader;

  TxHeader.Identifier = id;
  TxHeader.IdType = FDCAN_STANDARD_ID;
  TxHeader.TxFrameType = FDCAN_DATA_FRAME;
  TxHeader.DataLength = len << 16;  // DLC
  TxHeader.ErrorStateIndicator = FDCAN_ESI_ACTIVE;
  TxHeader.BitRateSwitch = FDCAN_BRS_ON;
  TxHeader.FDFormat = FDCAN_FD_CAN;
  TxHeader.TxEventFifoControl = FDCAN_NO_TX_EVENTS;
  TxHeader.MessageMarker = 0;

  HAL_FDCAN_AddMessageToTxFifoQ(&hfdcan1, &TxHeader, data);
}
```

### 3. SD 카드 로깅

```c
// SDMMC를 사용한 데이터 로깅
// M4의 D2 도메인에서 사용 가능

SD_HandleTypeDef hsd1;

void SD_Card_Init_M4(void)
{
  printf("Initializing SD card (SDMMC1)...\n");

  __HAL_RCC_SDMMC1_CLK_ENABLE();

  // GPIO 설정 (4-bit mode)
  // PC8: D0, PC9: D1, PC10: D2, PC11: D3
  // PC12: CLK, PD2: CMD

  // ... GPIO 초기화 코드 ...

  // SDMMC1 초기화
  hsd1.Instance = SDMMC1;
  hsd1.Init.ClockEdge = SDMMC_CLOCK_EDGE_RISING;
  hsd1.Init.ClockPowerSave = SDMMC_CLOCK_POWER_SAVE_DISABLE;
  hsd1.Init.BusWide = SDMMC_BUS_WIDE_4B;
  hsd1.Init.HardwareFlowControl = SDMMC_HARDWARE_FLOW_CONTROL_DISABLE;
  hsd1.Init.ClockDiv = 2;  // 120MHz / 2 = 60MHz

  if (HAL_SD_Init(&hsd1) != HAL_OK)
  {
    Error_Handler();
  }

  // SD 카드 정보 읽기
  HAL_SD_CardInfoTypeDef CardInfo;
  HAL_SD_GetCardInfo(&hsd1, &CardInfo);

  printf("  • SD Card initialized\n");
  printf("  • Type: %s\n",
         CardInfo.CardType == CARD_SDSC ? "SDSC" :
         CardInfo.CardType == CARD_SDHC_SDXC ? "SDHC/SDXC" : "Unknown");
  printf("  • Capacity: %lu MB\n",
         (CardInfo.BlockNbr * CardInfo.BlockSize) / 1024 / 1024);
}

// 데이터 로깅
void Log_To_SD_Card(uint8_t *data, uint32_t size)
{
  uint32_t block_addr = 0;  // 시작 블록
  uint32_t num_blocks = (size + 511) / 512;  // 블록 수

  // SD 카드에 쓰기
  if (HAL_SD_WriteBlocks(&hsd1, data, block_addr, num_blocks, 1000) != HAL_OK)
  {
    printf("ERROR: SD write failed\n");
    return;
  }

  // 쓰기 완료 대기
  while (HAL_SD_GetCardState(&hsd1) != HAL_SD_CARD_TRANSFER);

  printf("Logged %lu bytes to SD card\n", size);
}
```

## 전력 소비 최적화

### 동적 주파수 조정

```c
// M4 주파수를 동적으로 조정하여 전력 절감

typedef enum {
  FREQ_60MHZ = 0,   // 저전력 모드
  FREQ_120MHZ,      // 중간 성능
  FREQ_240MHZ       // 최대 성능
} M4_Frequency_t;

void Set_M4_Frequency(M4_Frequency_t freq)
{
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};
  uint32_t pllm, plln, pllp;

  switch (freq)
  {
    case FREQ_60MHZ:
      pllm = 5;
      plln = 96;
      pllp = 8;  // 480 / 8 = 60 MHz
      printf("Setting M4 frequency to 60 MHz...\n");
      break;

    case FREQ_120MHZ:
      pllm = 5;
      plln = 96;
      pllp = 4;  // 480 / 4 = 120 MHz
      printf("Setting M4 frequency to 120 MHz...\n");
      break;

    case FREQ_240MHZ:
    default:
      pllm = 5;
      plln = 96;
      pllp = 2;  // 480 / 2 = 240 MHz
      printf("Setting M4 frequency to 240 MHz...\n");
      break;
  }

  // PLL 재설정
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_NONE;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
  RCC_OscInitStruct.PLL.PLLM = pllm;
  RCC_OscInitStruct.PLL.PLLN = plln;
  RCC_OscInitStruct.PLL.PLLP = pllp;

  HAL_RCC_OscConfig(&RCC_OscInitStruct);

  printf("M4 frequency changed\n");
  printf("  • SYSCLK: %lu Hz\n", HAL_RCC_GetSysClockFreq());
}

// 작업 부하에 따라 주파수 조정
void Adaptive_Frequency_Control(void)
{
  static uint32_t idle_counter = 0;
  static M4_Frequency_t current_freq = FREQ_240MHZ;

  // CPU 사용률 측정 (간단한 방법)
  uint32_t busy_count = Get_Busy_Count();

  if (busy_count < 10)  // 낮은 부하
  {
    idle_counter++;
    if (idle_counter > 100 && current_freq != FREQ_60MHZ)
    {
      Set_M4_Frequency(FREQ_60MHZ);
      current_freq = FREQ_60MHZ;
      idle_counter = 0;
    }
  }
  else if (busy_count > 80)  // 높은 부하
  {
    if (current_freq != FREQ_240MHZ)
    {
      Set_M4_Frequency(FREQ_240MHZ);
      current_freq = FREQ_240MHZ;
    }
    idle_counter = 0;
  }
  else  // 중간 부하
  {
    if (current_freq != FREQ_120MHZ)
    {
      Set_M4_Frequency(FREQ_120MHZ);
      current_freq = FREQ_120MHZ;
    }
    idle_counter = 0;
  }
}
```

### 전력 측정 및 분석

```c
void Measure_Power_Consumption_M4(void)
{
  printf("\n=== M4 Power Consumption Analysis ===\n");

  printf("\n1. M4 @ 240 MHz:\n");
  printf("   • Core:              ~80 mA\n");
  printf("   • Peripherals:       ~20 mA\n");
  printf("   • Total:             ~100 mA\n");

  printf("\n2. M4 @ 120 MHz:\n");
  printf("   • Core:              ~45 mA\n");
  printf("   • Peripherals:       ~15 mA\n");
  printf("   • Total:             ~60 mA\n");
  printf("   • Savings:           40%\n");

  printf("\n3. M4 @ 60 MHz:\n");
  printf("   • Core:              ~25 mA\n");
  printf("   • Peripherals:       ~10 mA\n");
  printf("   • Total:             ~35 mA\n");
  printf("   • Savings:           65%\n");

  printf("\n4. Battery Life (2000 mAh):\n");
  printf("   • 240 MHz:           20 hours\n");
  printf("   • 120 MHz:           33 hours (+65%%)\n");
  printf("   • 60 MHz:            57 hours (+185%%)\n");

  printf("\n=====================================\n\n");
}
```

## 실전 응용 예제

### 예제: 산업용 센서 노드

```c
// M4로 다중 센서 모니터링 및 CAN 통신

#define SENSOR_COUNT  4

typedef struct {
  uint8_t id;
  float value;
  uint32_t timestamp;
} SensorData_t;

SensorData_t sensor_data[SENSOR_COUNT];

void Industrial_Sensor_Node(void)
{
  printf("Industrial Sensor Node (M4)\n");

  // 주변장치 초기화
  ADC_Init_M4();
  FDCAN_Init_M4();
  TIM_Init_M4();

  uint32_t sample_count = 0;

  while (1)
  {
    // 센서 읽기 (ADC1, ADC2)
    sensor_data[0].value = Read_Temperature_Sensor();
    sensor_data[1].value = Read_Pressure_Sensor();
    sensor_data[2].value = Read_Vibration_Sensor();
    sensor_data[3].value = Read_Current_Sensor();

    // 타임스탬프
    uint32_t timestamp = HAL_GetTick();
    for (int i = 0; i < SENSOR_COUNT; i++)
    {
      sensor_data[i].timestamp = timestamp;
    }

    // CAN FD로 전송
    uint8_t can_data[64];
    memcpy(can_data, sensor_data, sizeof(sensor_data));
    FDCAN_Transmit_Message(0x100, can_data, sizeof(sensor_data));

    sample_count++;
    printf("[%lu] Sensors read and transmitted\n", sample_count);

    HAL_Delay(100);  // 10 Hz
  }
}
```

## 트러블슈팅

### 문제 1: M4가 부팅되지 않음

```c
void Debug_M4_Boot(void)
{
  printf("=== M4 Boot Debug ===\n");

  // 1. M4 리셋 상태 확인
  if (RCC->GCR & RCC_GCR_CM4_RSTN)
  {
    printf("M4 is in reset state\n");
    printf("Releasing M4 reset...\n");
    __HAL_RCC_CM4_RELEASE_RESET();
  }

  // 2. M4 부팅 주소 확인
  uint32_t boot_addr = (SYSCFG->UR2 & SYSCFG_UR2_BOOT_ADD0) << 16;
  printf("M4 boot address: 0x%08lX\n", boot_addr);

  if (boot_addr != 0x08100000)
  {
    printf("WARNING: Incorrect M4 boot address!\n");
    printf("Setting to 0x08100000...\n");
    __HAL_SYSCFG_CM4_BOOT_ADDR(0x08100000 >> 16);
  }

  // 3. M4 Flash 확인
  uint32_t *m4_vector_table = (uint32_t*)0x08100000;
  printf("M4 Stack pointer: 0x%08lX\n", m4_vector_table[0]);
  printf("M4 Reset handler: 0x%08lX\n", m4_vector_table[1]);

  if (m4_vector_table[0] == 0xFFFFFFFF)
  {
    printf("ERROR: M4 Flash is empty!\n");
    printf("Please program M4 firmware to 0x08100000\n");
  }

  printf("=====================\n\n");
}
```

## 참고 자료

- **RM0399**: STM32H745 Reference Manual (D2 peripherals)
- **PM0214**: STM32H7 Cortex-M4 programming manual
- **AN5361**: Getting started with STM32H7 dual-core
- **DS12110**: STM32H745 datasheet (power characteristics)

## 관련 예제

- **PWR_D1ON_D2OFF**: D1만 활성화 (M7 전용)
- **PWR_Domain3SystemControl**: D3 자율 동작
- **PWR_Hold_Mechanism**: 듀얼 코어 부팅 제어
