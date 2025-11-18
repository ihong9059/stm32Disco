# RCC_ClockConfig - 동적 클럭 설정 및 전환

## 개요

이 예제는 STM32H745의 RCC(Reset and Clock Control)를 사용하여 런타임에 시스템 클럭 소스를 동적으로 전환하는 방법을 보여줍니다. HSE, HSI, CSI 간 PLL 소스를 전환하고, SYSCLK을 MCO2 핀으로 출력하여 클럭 변경을 실시간으로 확인할 수 있습니다.

## 하드웨어 요구사항

- **STM32H745I-DISCO 보드**
- **LED1**: PI12 - 클럭 소스 표시
- **MCO2 출력**: PC9 - SYSCLK 출력
- **오실로스코프**: 클럭 주파수 측정용
- **버튼**: Wakeup 버튼 (PC13) - 클럭 전환 트리거

## 주요 기능

### 클럭 소스 전환
- **HSE (High Speed External)**: 25MHz 외부 크리스탈
- **HSI (High Speed Internal)**: 64MHz 내부 RC 오실레이터
- **CSI (Low Power Internal)**: 4MHz 저전력 내부 오실레이터
- **PLL**: 각 소스로부터 400MHz SYSCLK 생성

### MCO (Microcontroller Clock Output)
- **MCO1**: PA8 - 다양한 클럭 소스 출력
- **MCO2**: PC9 - SYSCLK 출력 (분주 가능)
- **실시간 모니터링**: 오실로스코프로 클럭 확인

## 클럭 트리 구조

```
                    ┌──────────┐
                    │   HSE    │  25MHz (외부 크리스탈)
                    │  25 MHz  │
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │   HSI    │  64MHz (내부 RC)
                    │  64 MHz  │
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │   CSI    │  4MHz (저전력 내부)
                    │  4 MHz   │
                    └────┬─────┘
                         │
                    ┌────▼─────┐
           ┌────────┤   PLL1   ├────────┐
           │        │          │        │
           │        └──────────┘        │
           │                            │
           ▼                            ▼
    ┌──────────┐                 ┌──────────┐
    │ SYSCLK   │  400MHz         │   MCO2   │  PC9
    │  (CM7)   │                 │ (SYSCLK/4)│
    └──────────┘                 └──────────┘
```

## 동작 원리

### 클럭 전환 시퀀스
```
1. 현재 클럭 소스 확인
2. 새로운 클럭 소스 활성화
3. PLL 재설정 (새 소스 기준)
4. SYSCLK 소스 전환
5. 플래시 대기 상태 조정
6. 주변장치 클럭 재설정
7. 이전 클럭 소스 비활성화 (선택)
```

### 상태 머신
```
     ┌─────────┐
     │   HSE   │◄──────┐
     │ (25MHz) │       │
     └────┬────┘       │
          │ Button    │
          ▼            │
     ┌─────────┐       │
     │   HSI   │       │ Button
     │ (64MHz) │       │
     └────┬────┘       │
          │ Button    │
          ▼            │
     ┌─────────┐       │
     │   CSI   │───────┘
     │ (4MHz)  │
     └─────────┘
```

## 코드 구조

### 1. 클럭 소스 열거형

```c
typedef enum {
  CLOCK_SOURCE_HSE = 0,
  CLOCK_SOURCE_HSI,
  CLOCK_SOURCE_CSI,
  CLOCK_SOURCE_MAX
} ClockSource_t;

// 현재 클럭 소스
ClockSource_t current_clock_source = CLOCK_SOURCE_HSE;

// 클럭 소스 이름
const char* clock_names[] = {
  "HSE (25 MHz)",
  "HSI (64 MHz)",
  "CSI (4 MHz)"
};
```

### 2. HSE 클럭 설정 (기본)

```c
void SystemClock_Config_HSE(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  // PWR 클럭 활성화
  __HAL_RCC_PWR_CLK_ENABLE();

  // 전압 스케일링: VOS1 (최고 성능)
  __HAL_PWR_VOLTAGESCALING_CONFIG(PWR_REGULATOR_VOLTAGE_SCALE1);

  while (!__HAL_PWR_GET_FLAG(PWR_FLAG_VOSRDY)) {}

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 오실레이터 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
  RCC_OscInitStruct.HSEState = RCC_HSE_ON;
  RCC_OscInitStruct.HSIState = RCC_HSI_OFF;
  RCC_OscInitStruct.CSIState = RCC_CSI_OFF;

  // PLL1 설정 (SYSCLK = 400MHz)
  // VCO = HSE × (N / M) = 25MHz × (160 / 5) = 800MHz
  // SYSCLK = VCO / P = 800MHz / 2 = 400MHz
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
  RCC_OscInitStruct.PLL.PLLM = 5;       // 25MHz / 5 = 5MHz
  RCC_OscInitStruct.PLL.PLLN = 160;     // 5MHz × 160 = 800MHz (VCO)
  RCC_OscInitStruct.PLL.PLLP = 2;       // 800MHz / 2 = 400MHz (SYSCLK)
  RCC_OscInitStruct.PLL.PLLQ = 4;       // 800MHz / 4 = 200MHz
  RCC_OscInitStruct.PLL.PLLR = 2;       // 800MHz / 2 = 400MHz
  RCC_OscInitStruct.PLL.PLLRGE = RCC_PLL1VCIRANGE_2;  // 4-8 MHz
  RCC_OscInitStruct.PLL.PLLVCOSEL = RCC_PLL1VCOWIDE;  // Wide VCO
  RCC_OscInitStruct.PLL.PLLFRACN = 0;

  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 클럭 분배 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK |
                                 RCC_CLOCKTYPE_SYSCLK |
                                 RCC_CLOCKTYPE_PCLK1 |
                                 RCC_CLOCKTYPE_PCLK2 |
                                 RCC_CLOCKTYPE_D3PCLK1 |
                                 RCC_CLOCKTYPE_D1PCLK1;

  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;  // PLL1
  RCC_ClkInitStruct.SYSCLKDivider = RCC_SYSCLK_DIV1;         // 400MHz
  RCC_ClkInitStruct.AHBCLKDivider = RCC_HCLK_DIV2;           // 200MHz
  RCC_ClkInitStruct.APB3CLKDivider = RCC_APB3_DIV2;          // 100MHz
  RCC_ClkInitStruct.APB1CLKDivider = RCC_APB1_DIV2;          // 100MHz
  RCC_ClkInitStruct.APB2CLKDivider = RCC_APB2_DIV2;          // 100MHz
  RCC_ClkInitStruct.APB4CLKDivider = RCC_APB4_DIV2;          // 100MHz

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_4) != HAL_OK)
  {
    Error_Handler();
  }

  // D3 도메인 클럭 설정
  __HAL_RCC_D3PCLK1_CONFIG(RCC_APB4_DIV2);

  printf("SYSCLK configured: HSE -> PLL1 -> 400 MHz\n");
}
```

### 3. HSI 클럭 설정

```c
void SystemClock_Config_HSI(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // HSI 오실레이터 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSI;
  RCC_OscInitStruct.HSIState = RCC_HSI_ON;
  RCC_OscInitStruct.HSICalibrationValue = RCC_HSICALIBRATION_DEFAULT;
  RCC_OscInitStruct.HSEState = RCC_HSE_OFF;  // HSE 비활성화
  RCC_OscInitStruct.CSIState = RCC_CSI_OFF;

  // PLL1 설정 (SYSCLK = 400MHz)
  // VCO = HSI × (N / M) = 64MHz × (50 / 4) = 800MHz
  // SYSCLK = VCO / P = 800MHz / 2 = 400MHz
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSI;
  RCC_OscInitStruct.PLL.PLLM = 4;       // 64MHz / 4 = 16MHz
  RCC_OscInitStruct.PLL.PLLN = 50;      // 16MHz × 50 = 800MHz (VCO)
  RCC_OscInitStruct.PLL.PLLP = 2;       // 800MHz / 2 = 400MHz
  RCC_OscInitStruct.PLL.PLLQ = 4;
  RCC_OscInitStruct.PLL.PLLR = 2;
  RCC_OscInitStruct.PLL.PLLRGE = RCC_PLL1VCIRANGE_3;  // 8-16 MHz
  RCC_OscInitStruct.PLL.PLLVCOSEL = RCC_PLL1VCOWIDE;
  RCC_OscInitStruct.PLL.PLLFRACN = 0;

  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  // 클럭 분배 (HSE와 동일)
  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK |
                                 RCC_CLOCKTYPE_SYSCLK |
                                 RCC_CLOCKTYPE_PCLK1 |
                                 RCC_CLOCKTYPE_PCLK2 |
                                 RCC_CLOCKTYPE_D3PCLK1 |
                                 RCC_CLOCKTYPE_D1PCLK1;

  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
  RCC_ClkInitStruct.SYSCLKDivider = RCC_SYSCLK_DIV1;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_HCLK_DIV2;
  RCC_ClkInitStruct.APB3CLKDivider = RCC_APB3_DIV2;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_APB1_DIV2;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_APB2_DIV2;
  RCC_ClkInitStruct.APB4CLKDivider = RCC_APB4_DIV2;

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_4) != HAL_OK)
  {
    Error_Handler();
  }

  printf("SYSCLK configured: HSI -> PLL1 -> 400 MHz\n");
}
```

### 4. CSI 클럭 설정

```c
void SystemClock_Config_CSI(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // CSI 오실레이터 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_CSI;
  RCC_OscInitStruct.CSIState = RCC_CSI_ON;
  RCC_OscInitStruct.CSICalibrationValue = RCC_CSICALIBRATION_DEFAULT;
  RCC_OscInitStruct.HSEState = RCC_HSE_OFF;
  RCC_OscInitStruct.HSIState = RCC_HSI_OFF;

  // PLL1 설정 (SYSCLK = 400MHz)
  // VCO = CSI × (N / M) = 4MHz × (200 / 1) = 800MHz
  // SYSCLK = VCO / P = 800MHz / 2 = 400MHz
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_CSI;
  RCC_OscInitStruct.PLL.PLLM = 1;       // 4MHz / 1 = 4MHz
  RCC_OscInitStruct.PLL.PLLN = 200;     // 4MHz × 200 = 800MHz (VCO)
  RCC_OscInitStruct.PLL.PLLP = 2;       // 800MHz / 2 = 400MHz
  RCC_OscInitStruct.PLL.PLLQ = 4;
  RCC_OscInitStruct.PLL.PLLR = 2;
  RCC_OscInitStruct.PLL.PLLRGE = RCC_PLL1VCIRANGE_1;  // 2-4 MHz
  RCC_OscInitStruct.PLL.PLLVCOSEL = RCC_PLL1VCOWIDE;
  RCC_OscInitStruct.PLL.PLLFRACN = 0;

  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  // 클럭 분배
  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK |
                                 RCC_CLOCKTYPE_SYSCLK |
                                 RCC_CLOCKTYPE_PCLK1 |
                                 RCC_CLOCKTYPE_PCLK2 |
                                 RCC_CLOCKTYPE_D3PCLK1 |
                                 RCC_CLOCKTYPE_D1PCLK1;

  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
  RCC_ClkInitStruct.SYSCLKDivider = RCC_SYSCLK_DIV1;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_HCLK_DIV2;
  RCC_ClkInitStruct.APB3CLKDivider = RCC_APB3_DIV2;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_APB1_DIV2;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_APB2_DIV2;
  RCC_ClkInitStruct.APB4CLKDivider = RCC_APB4_DIV2;

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_4) != HAL_OK)
  {
    Error_Handler();
  }

  printf("SYSCLK configured: CSI -> PLL1 -> 400 MHz\n");
}
```

### 5. MCO2 출력 설정 (SYSCLK 모니터링)

```c
void MCO2_Config(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  // GPIOC 클럭 활성화
  __HAL_RCC_GPIOC_CLK_ENABLE();

  // PC9를 MCO2 Alternate Function으로 설정
  GPIO_InitStruct.Pin = GPIO_PIN_9;
  GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;
  GPIO_InitStruct.Alternate = GPIO_AF0_MCO;  // AF0 = MCO

  HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // MCO2 설정: SYSCLK / 4
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // SYSCLK = 400MHz
  // MCO2 출력 = 400MHz / 4 = 100MHz
  HAL_RCC_MCOConfig(RCC_MCO2,
                    RCC_MCO2SOURCE_SYSCLK,
                    RCC_MCODIV_4);

  printf("MCO2 configured: PC9 outputs SYSCLK/4 (100 MHz)\n");
}
```

### 6. 동적 클럭 전환

```c
void Switch_Clock_Source(ClockSource_t new_source)
{
  if (new_source == current_clock_source)
  {
    printf("Already using %s\n", clock_names[new_source]);
    return;
  }

  printf("Switching from %s to %s...\n",
         clock_names[current_clock_source],
         clock_names[new_source]);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 클럭 전환
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  switch (new_source)
  {
    case CLOCK_SOURCE_HSE:
      SystemClock_Config_HSE();
      break;

    case CLOCK_SOURCE_HSI:
      SystemClock_Config_HSI();
      break;

    case CLOCK_SOURCE_CSI:
      SystemClock_Config_CSI();
      break;

    default:
      printf("Invalid clock source\n");
      return;
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // SystemCoreClock 변수 업데이트
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  SystemCoreClockUpdate();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // SysTick 재설정 (HAL_Delay 정확도 유지)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  HAL_InitTick(TICK_INT_PRIORITY);

  current_clock_source = new_source;

  printf("Clock switch complete. SYSCLK = %lu MHz\n",
         SystemCoreClock / 1000000);
}
```

### 7. 클럭 정보 출력

```c
void Print_Clock_Info(void)
{
  RCC_ClkInitTypeDef clk_init;
  uint32_t flash_latency;

  HAL_RCC_GetClockConfig(&clk_init, &flash_latency);

  printf("\n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n");
  printf("Clock Configuration:\n");
  printf("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n");

  // 클럭 소스
  printf("Current source: %s\n", clock_names[current_clock_source]);

  // SYSCLK
  printf("SYSCLK:    %3lu MHz\n", HAL_RCC_GetSysClockFreq() / 1000000);

  // HCLK (AHB)
  printf("HCLK:      %3lu MHz\n", HAL_RCC_GetHCLKFreq() / 1000000);

  // PCLK1 (APB1)
  printf("PCLK1:     %3lu MHz\n", HAL_RCC_GetPCLK1Freq() / 1000000);

  // PCLK2 (APB2)
  printf("PCLK2:     %3lu MHz\n", HAL_RCC_GetPCLK2Freq() / 1000000);

  // Flash 대기 상태
  printf("Flash WS:  %lu\n", flash_latency);

  // MCO2 출력
  printf("MCO2:      SYSCLK/4 = %lu MHz\n",
         HAL_RCC_GetSysClockFreq() / 4 / 1000000);

  printf("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n\n");
}
```

### 8. 메인 함수

```c
int main(void)
{
  // HAL 초기화
  HAL_Init();

  // 기본 클럭 설정 (HSE)
  SystemClock_Config_HSE();

  // GPIO 초기화
  GPIO_Init();

  // MCO2 출력 설정
  MCO2_Config();

  // 초기 클럭 정보 출력
  Print_Clock_Info();

  printf("Press USER button to cycle through clock sources\n");
  printf("Monitor PC9 with oscilloscope to see SYSCLK/4\n\n");

  // 무한 루프
  while (1)
  {
    // LED 토글 (현재 클럭 소스 표시)
    HAL_GPIO_TogglePin(GPIOI, GPIO_PIN_12);
    HAL_Delay(500);

    // 버튼 입력 확인
    if (HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_RESET)
    {
      // 디바운싱
      HAL_Delay(50);

      if (HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_RESET)
      {
        // 다음 클럭 소스로 전환
        ClockSource_t next_source = (current_clock_source + 1) % CLOCK_SOURCE_MAX;
        Switch_Clock_Source(next_source);

        // 클럭 정보 출력
        Print_Clock_Info();

        // 버튼 릴리스 대기
        while (HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_RESET);
        HAL_Delay(50);
      }
    }
  }
}
```

## PLL 계산 상세

### PLL 공식

```
VCO_IN = CLK_SRC / PLLM
VCO_OUT = VCO_IN × PLLN
SYSCLK = VCO_OUT / PLLP

제약 조건:
- VCO_IN: 2-16 MHz (PLLRGE 설정)
- VCO_OUT: 192-836 MHz (Wide VCO)
- SYSCLK: 최대 480 MHz (H745는 400 MHz)
```

### HSE 예제 (25MHz -> 400MHz)

```c
// HSE = 25MHz
// 목표 SYSCLK = 400MHz

PLLM = 5   // VCO_IN = 25MHz / 5 = 5MHz ✓ (2-16 MHz)
PLLN = 160 // VCO_OUT = 5MHz × 160 = 800MHz ✓ (192-836 MHz)
PLLP = 2   // SYSCLK = 800MHz / 2 = 400MHz ✓

PLLRGE = RCC_PLL1VCIRANGE_2  // 4-8 MHz (5MHz ✓)
```

### HSI 예제 (64MHz -> 400MHz)

```c
// HSI = 64MHz
// 목표 SYSCLK = 400MHz

PLLM = 4   // VCO_IN = 64MHz / 4 = 16MHz ✓
PLLN = 50  // VCO_OUT = 16MHz × 50 = 800MHz ✓
PLLP = 2   // SYSCLK = 800MHz / 2 = 400MHz ✓

PLLRGE = RCC_PLL1VCIRANGE_3  // 8-16 MHz (16MHz ✓)
```

### CSI 예제 (4MHz -> 400MHz)

```c
// CSI = 4MHz
// 목표 SYSCLK = 400MHz

PLLM = 1    // VCO_IN = 4MHz / 1 = 4MHz ✓
PLLN = 200  // VCO_OUT = 4MHz × 200 = 800MHz ✓
PLLP = 2    // SYSCLK = 800MHz / 2 = 400MHz ✓

PLLRGE = RCC_PLL1VCIRANGE_1  // 2-4 MHz (4MHz ✓)
```

## 고급 기능

### 1. CSS (Clock Security System)

```c
void Enable_CSS(void)
{
  // HSE 장애 감지 시 자동으로 HSI로 전환
  __HAL_RCC_CSS_ENABLE();

  // CSS 인터럽트 활성화
  HAL_NVIC_SetPriority(RCC_IRQn, 0, 0);
  HAL_NVIC_EnableIRQ(RCC_IRQn);

  printf("Clock Security System enabled\n");
}

// CSS 콜백
void HAL_RCC_CSSCallback(void)
{
  printf("HSE failure detected! Switched to HSI\n");

  // LED 경고
  HAL_GPIO_WritePin(GPIOI, GPIO_PIN_13, GPIO_PIN_SET);
}
```

### 2. MCO1 출력 (다양한 소스)

```c
void MCO1_Config(uint32_t source, uint32_t div)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  __HAL_RCC_GPIOA_CLK_ENABLE();

  // PA8 = MCO1
  GPIO_InitStruct.Pin = GPIO_PIN_8;
  GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;
  GPIO_InitStruct.Alternate = GPIO_AF0_MCO;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  // MCO1 소스 선택
  // RCC_MCO1SOURCE_HSI, HSE, CSI, PLL1, etc.
  HAL_RCC_MCOConfig(RCC_MCO1, source, div);

  printf("MCO1 configured on PA8\n");
}
```

### 3. 저전력 클럭 모드

```c
void LowPower_Clock_Config(void)
{
  // CSI로 전환 (저전력)
  Switch_Clock_Source(CLOCK_SOURCE_CSI);

  // SYSCLK 분주 (추가 전력 절약)
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};
  uint32_t flash_latency;

  HAL_RCC_GetClockConfig(&RCC_ClkInitStruct, &flash_latency);

  RCC_ClkInitStruct.SYSCLKDivider = RCC_SYSCLK_DIV2;  // 400MHz -> 200MHz
  HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_2);

  SystemCoreClockUpdate();

  printf("Low power mode: SYSCLK = %lu MHz\n",
         SystemCoreClock / 1000000);
}
```

## 클럭 측정

### 오실로스코프 설정

```
프로브 연결:
- PC9 (MCO2): SYSCLK/4 출력
- GND

예상 주파수:
- HSE: 100 MHz (400MHz / 4)
- HSI: 100 MHz (400MHz / 4)
- CSI: 100 MHz (400MHz / 4)

오실로스코프 설정:
- 주파수 측정 모드
- AC 커플링
- 1:10 프로브
```

## 빌드 및 실행

### 테스트 시나리오

```
1. 플래싱 및 리셋
2. 시리얼 터미널: "SYSCLK: HSE -> 400 MHz"
3. 오실로스코프: PC9에서 100 MHz 확인
4. USER 버튼 누름
5. 터미널: "Switching to HSI..."
6. 오실로스코프: 여전히 100 MHz (PLL 조정)
7. USER 버튼 누름
8. 터미널: "Switching to CSI..."
9. 오실로스코프: 여전히 100 MHz
10. USER 버튼 누름 (HSE로 복귀)
```

## 트러블슈팅

### PLL 락 실패

```c
// PLL이 락되지 않는 경우
if (!__HAL_RCC_GET_FLAG(RCC_FLAG_PLLRDY))
{
  printf("PLL failed to lock!\n");

  // PLL 파라미터 확인
  // VCO_IN 범위: 2-16 MHz
  // VCO_OUT 범위: 192-836 MHz
}
```

### CSS 오동작

```c
// HSE가 없는 보드에서 CSS 활성화 시 계속 인터럽트 발생
// 해결: HSE 사용 시에만 CSS 활성화
if (current_clock_source == CLOCK_SOURCE_HSE)
{
  __HAL_RCC_CSS_ENABLE();
}
```

### MCO 출력 없음

```c
// GPIO Alternate Function 확인
uint32_t afr = GPIOC->AFR[1];  // PC9 = AFR[1]
printf("PC9 AFR: 0x%08lX\n", (afr >> 4) & 0xF);  // AF0

// MCO 활성화 확인
uint32_t cfgr = RCC->CFGR;
printf("RCC_CFGR: 0x%08lX\n", cfgr);
```

## 참고 자료

- **RM0399**: STM32H745 Reference Manual, Chapter 8 (RCC)
- **AN4891**: Clock configuration on STM32H7 MCUs
- **DS12110**: STM32H745 Datasheet (클럭 사양)
- **예제 코드**: `STM32Cube_FW_H7/Projects/STM32H745I-DISCO/Examples/RCC/RCC_ClockConfig`

## 관련 예제

- **RCC_LSEConfig**: LSE 외부 저속 클럭
- **RCC_CRS_Synchronization**: HSI48 자동 교정
- **PWR_VoltageScaling**: 전압 스케일링 및 오버드라이브
