# UART_WakeUpFromStopUsingFIFO - UART FIFO를 이용한 Stop 모드 웨이크업

## 개요

이 예제는 STM32H745의 UART FIFO(First-In-First-Out) 기능을 사용하여 Stop 모드에서 효율적으로 웨이크업하는 방법을 보여줍니다. FIFO 임계값 또는 FIFO Full 이벤트를 트리거로 사용하여 불필요한 웨이크업을 최소화하고 전력 소비를 줄입니다.

## 하드웨어 요구사항

- **STM32H745I-DISCO 보드**
- **UART**: USART3 (가상 COM 포트)
  - TX: PB10
  - RX: PB11
- **USB-UART 브리지**: ST-LINK 내장
- **LED1**: PI12 - 웨이크업 표시
- **LED2**: PI13 - Stop 모드 표시

## 주요 기능

### UART FIFO
- **RX FIFO 크기**: 8바이트 (깊이 설정 가능)
- **TX FIFO 크기**: 8바이트
- **임계값 설정**: 1/8, 1/4, 1/2, 3/4, 7/8, Full
- **웨이크업 소스**: FIFO 임계값 또는 Full

### 웨이크업 모드
1. **FIFO 임계값 모드**: FIFO가 설정된 레벨에 도달하면 웨이크업
2. **FIFO Full 모드**: FIFO가 완전히 차면 웨이크업

### Stop 모드 이점
- **전력 소비**: ~100 μA (Stop 모드)
- **SRAM 보존**: 모든 SRAM 내용 유지
- **빠른 복귀**: 몇 μs 이내
- **주변장치 동작**: 일부 주변장치 계속 동작 (UART, RTC 등)

## 동작 원리

### FIFO 웨이크업 메커니즘
```
UART RX 데이터 수신:

Byte 1 → FIFO[0]  ─────► (계속 Stop 모드)
Byte 2 → FIFO[1]  ─────► (계속 Stop 모드)
Byte 3 → FIFO[2]  ─────► (계속 Stop 모드)
Byte 4 → FIFO[3]  ─────► (임계값 도달!)
         │
         ▼
    ┌─────────┐
    │ Wakeup  │  Stop 모드에서 복귀
    │  Event  │
    └────┬────┘
         │
         ▼
    ┌─────────┐
    │ Process │  수신된 데이터 처리
    │  Data   │
    └─────────┘
```

### 시스템 상태 흐름
```
┌─────────────┐
│  Run Mode   │  ─┐
│  (100 mA)   │   │ 아이들 상태
└──────┬──────┘   │ (1초)
       │          │
       │ Enter    │
       │ Stop     │
       ▼          │
┌─────────────┐   │
│  Stop Mode  │   │
│  (100 μA)   │   │
└──────┬──────┘   │
       │          │
       │ UART RX  │
       │ FIFO     │
       ▼          │
┌─────────────┐   │
│  Wakeup     │   │
│  Process    │   │
└──────┬──────┘   │
       │          │
       └──────────┘
```

## 코드 구조

### 1. UART FIFO 초기화

```c
UART_HandleTypeDef huart3;

// FIFO 웨이크업 모드
typedef enum {
  FIFO_WAKEUP_THRESHOLD,  // 임계값 모드
  FIFO_WAKEUP_FULL        // Full 모드
} FIFO_WakeupMode_t;

FIFO_WakeupMode_t wakeup_mode = FIFO_WAKEUP_THRESHOLD;

void UART3_FIFO_Init(void)
{
  UART_AdvFeatureInitTypeDef advfeature = {0};

  // UART3 클럭 활성화
  __HAL_RCC_USART3_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // UART 기본 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  huart3.Instance = USART3;
  huart3.Init.BaudRate = 9600;
  huart3.Init.WordLength = UART_WORDLENGTH_8B;
  huart3.Init.StopBits = UART_STOPBITS_1;
  huart3.Init.Parity = UART_PARITY_ODD;
  huart3.Init.Mode = UART_MODE_TX_RX;
  huart3.Init.HwFlowCtl = UART_HWCONTROL_NONE;
  huart3.Init.OverSampling = UART_OVERSAMPLING_16;
  huart3.Init.OneBitSampling = UART_ONE_BIT_SAMPLE_DISABLE;
  huart3.Init.ClockPrescaler = UART_PRESCALER_DIV1;

  if (HAL_UART_Init(&huart3) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // FIFO 모드 활성화
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  if (HAL_UARTEx_EnableFifoMode(&huart3) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // RX FIFO 임계값 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // FIFO 임계값: 1/2 (4바이트)
  if (HAL_UARTEx_SetRxFifoThreshold(&huart3, UART_RXFIFO_THRESHOLD_1_2) != HAL_OK)
  {
    Error_Handler();
  }

  // TX FIFO 임계값 (선택 사항)
  HAL_UARTEx_SetTxFifoThreshold(&huart3, UART_TXFIFO_THRESHOLD_1_2);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 고급 기능 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  advfeature.AdvFeatureInit = UART_ADVFEATURE_NO_INIT;

  // Stop 모드에서 UART 활성화
  advfeature.AdvFeatureInit |= UART_ADVFEATURE_WAKEUP_INIT;

  if (wakeup_mode == FIFO_WAKEUP_THRESHOLD)
  {
    // 모드 1: FIFO 임계값 웨이크업
    advfeature.WakeUpMethod = UART_WAKEUPMETHOD_RXFIFOTHRESHOLD;
    printf("FIFO Wakeup Mode: THRESHOLD (4 bytes)\n");
  }
  else
  {
    // 모드 2: FIFO Full 웨이크업
    advfeature.WakeUpMethod = UART_WAKEUPMETHOD_RXFIFOFULL;
    printf("FIFO Wakeup Mode: FULL (8 bytes)\n");
  }

  if (HAL_UART_AdvFeatureConfig(&huart3, &advfeature) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // UART 인터럽트 활성화
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // RXFIFO 임계값 인터럽트
  __HAL_UART_ENABLE_IT(&huart3, UART_IT_RXFT);

  // RXFIFO Full 인터럽트 (Full 모드 사용 시)
  if (wakeup_mode == FIFO_WAKEUP_FULL)
  {
    __HAL_UART_ENABLE_IT(&huart3, UART_IT_RXFF);
  }

  HAL_NVIC_SetPriority(USART3_IRQn, 0, 0);
  HAL_NVIC_EnableIRQ(USART3_IRQn);

  printf("UART3 FIFO initialized\n");
}
```

### 2. GPIO 초기화

```c
void GPIO_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  // GPIO 클럭 활성화
  __HAL_RCC_GPIOB_CLK_ENABLE();
  __HAL_RCC_GPIOI_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // UART3 핀 설정 (PB10=TX, PB11=RX)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  GPIO_InitStruct.Pin = GPIO_PIN_10 | GPIO_PIN_11;
  GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
  GPIO_InitStruct.Pull = GPIO_PULLUP;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;
  GPIO_InitStruct.Alternate = GPIO_AF7_USART3;
  HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // LED 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  GPIO_InitStruct.Pin = GPIO_PIN_12 | GPIO_PIN_13;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  HAL_GPIO_Init(GPIOI, &GPIO_InitStruct);

  // 초기 상태: OFF
  HAL_GPIO_WritePin(GPIOI, GPIO_PIN_12 | GPIO_PIN_13, GPIO_PIN_RESET);
}
```

### 3. Stop 모드 진입

```c
void Enter_Stop_Mode(void)
{
  printf("Entering STOP mode...\n");
  printf("Send 4+ characters to wake up\n");
  HAL_Delay(100);  // 메시지 전송 대기

  // LED2 켜기 (Stop 모드 표시)
  HAL_GPIO_WritePin(GPIOI, GPIO_PIN_13, GPIO_PIN_SET);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // UART 웨이크업 활성화
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  HAL_UARTEx_EnableStopMode(&huart3);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // Stop 모드 진입
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // D1 도메인 Stop 모드
  HAL_PWREx_EnterSTOPMode(PWR_MAINREGULATOR_ON,
                          PWR_STOPENTRY_WFI,
                          PWR_D1_DOMAIN);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 웨이크업 후 처리
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // LED2 끄기
  HAL_GPIO_WritePin(GPIOI, GPIO_PIN_13, GPIO_PIN_RESET);

  // LED1 켜기 (웨이크업 표시)
  HAL_GPIO_WritePin(GPIOI, GPIO_PIN_12, GPIO_PIN_SET);

  // 시스템 클럭 재설정 (Stop에서 복귀 시 HSI로 전환됨)
  SystemClock_Config();

  // UART Stop 모드 비활성화
  HAL_UARTEx_DisableStopMode(&huart3);

  printf("\nWoke up from STOP mode!\n");
}
```

### 4. UART 인터럽트 핸들러

```c
// 수신 버퍼
#define RX_BUFFER_SIZE  32
uint8_t rx_buffer[RX_BUFFER_SIZE];
uint8_t rx_index = 0;

void USART3_IRQHandler(void)
{
  // RXFIFO 임계값 인터럽트
  if (__HAL_UART_GET_FLAG(&huart3, UART_FLAG_RXFT) != RESET)
  {
    // FIFO에서 데이터 읽기
    while (__HAL_UART_GET_FLAG(&huart3, UART_FLAG_RXFNE) != RESET)
    {
      // FIFO가 비어있지 않음
      uint8_t data;
      HAL_UART_Receive(&huart3, &data, 1, 0);

      if (rx_index < RX_BUFFER_SIZE)
      {
        rx_buffer[rx_index++] = data;
      }
    }

    // 플래그 클리어
    __HAL_UART_CLEAR_FLAG(&huart3, UART_CLEAR_RTOF);
  }

  // RXFIFO Full 인터럽트
  if (__HAL_UART_GET_FLAG(&huart3, UART_FLAG_RXFF) != RESET)
  {
    // FIFO에서 모든 데이터 읽기
    while (__HAL_UART_GET_FLAG(&huart3, UART_FLAG_RXFNE) != RESET)
    {
      uint8_t data;
      HAL_UART_Receive(&huart3, &data, 1, 0);

      if (rx_index < RX_BUFFER_SIZE)
      {
        rx_buffer[rx_index++] = data;
      }
    }

    // 플래그 클리어
    __HAL_UART_CLEAR_FLAG(&huart3, UART_CLEAR_RXFFCF);
  }

  // 일반 HAL 인터럽트 처리
  HAL_UART_IRQHandler(&huart3);
}
```

### 5. 수신 데이터 처리

```c
void Process_Received_Data(void)
{
  if (rx_index > 0)
  {
    printf("Received %d bytes: ", rx_index);

    for (uint8_t i = 0; i < rx_index; i++)
    {
      printf("%c", rx_buffer[i]);
    }

    printf("\n");

    // 버퍼 클리어
    rx_index = 0;
    memset(rx_buffer, 0, sizeof(rx_buffer));
  }
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

  // GPIO 초기화
  GPIO_Init();

  // UART FIFO 초기화
  UART3_FIFO_Init();

  printf("\n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n");
  printf("UART FIFO Stop Mode Wakeup Example\n");
  printf("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n\n");

  // 무한 루프
  while (1)
  {
    // Run 모드에서 1초 대기
    printf("Running for 1 second...\n");
    HAL_GPIO_TogglePin(GPIOI, GPIO_PIN_12);
    HAL_Delay(1000);

    // Stop 모드 진입
    Enter_Stop_Mode();

    // 웨이크업 후 수신 데이터 처리
    Process_Received_Data();

    // LED1 끄기
    HAL_GPIO_WritePin(GPIOI, GPIO_PIN_12, GPIO_PIN_RESET);

    HAL_Delay(500);
  }
}
```

## FIFO 임계값 설정

### 사용 가능한 임계값

```c
// RX FIFO 임계값 옵션
UART_RXFIFO_THRESHOLD_1_8   // 1바이트
UART_RXFIFO_THRESHOLD_1_4   // 2바이트
UART_RXFIFO_THRESHOLD_1_2   // 4바이트 (권장)
UART_RXFIFO_THRESHOLD_3_4   // 6바이트
UART_RXFIFO_THRESHOLD_7_8   // 7바이트
UART_RXFIFO_THRESHOLD_8_8   // 8바이트 (Full)

// TX FIFO 임계값 옵션
UART_TXFIFO_THRESHOLD_1_8   // 1바이트
UART_TXFIFO_THRESHOLD_1_4   // 2바이트
UART_TXFIFO_THRESHOLD_1_2   // 4바이트
UART_TXFIFO_THRESHOLD_3_4   // 6바이트
UART_TXFIFO_THRESHOLD_7_8   // 7바이트
UART_TXFIFO_THRESHOLD_8_8   // 8바이트 (Empty)
```

### 임계값 선택 가이드

```c
용도                      │ 권장 임계값      │ 이유
────────────────────────┼─────────────────┼──────────────────
단일 문자 입력          │ 1/8 (1 byte)    │ 즉시 웨이크업
짧은 명령어 (2-3자)     │ 1/4 (2 bytes)   │ 명령어 시작 감지
일반 메시지 (4-7자)     │ 1/2 (4 bytes)   │ 균형잡힌 선택
긴 메시지 (8자 이상)    │ 7/8 또는 Full   │ 웨이크업 최소화
```

## 전력 소비 분석

### 모드별 전류 소비

```
모드               │ 전류 (전형)  │ 웨이크업 시간
──────────────────┼─────────────┼──────────────
Run (400 MHz)     │ ~100 mA     │ -
Stop (HSI ON)     │ ~100 μA     │ ~5 μs
Stop (HSI OFF)    │ ~40 μA      │ ~50 μs
Standby           │ ~2.4 μA     │ ~ms
```

### 사이클 분석

```
사이클 시간:
- Run: 1초 (100 mA)
- Stop: 가변 (100 μA)
- Wakeup: 10ms (100 mA)

시나리오 1: 10초마다 웨이크업
평균 전류 = (100mA × 1s + 0.1mA × 9s + 100mA × 0.01s) / 10s
          = (100 + 0.9 + 1) / 10
          ≈ 10.2 mA

시나리오 2: 1시간마다 웨이크업
평균 전류 = (100mA × 1s + 0.1mA × 3599s + 100mA × 0.01s) / 3600s
          ≈ 0.13 mA
```

## 고급 기능

### 1. 다중 웨이크업 소스

```c
void Configure_Multiple_Wakeup_Sources(void)
{
  // UART FIFO 웨이크업
  HAL_UARTEx_EnableStopMode(&huart3);

  // RTC 알람 웨이크업
  RTC_AlarmTypeDef sAlarm = {0};
  sAlarm.Alarm = RTC_ALARM_A;
  sAlarm.AlarmTime.Seconds = 10;  // 10초 후
  HAL_RTC_SetAlarm_IT(&hrtc, &sAlarm, RTC_FORMAT_BIN);

  // 두 웨이크업 소스 중 하나라도 발생하면 깨어남
  Enter_Stop_Mode();

  // 웨이크업 소스 확인
  if (__HAL_UART_GET_FLAG(&huart3, UART_FLAG_WUF))
  {
    printf("Woken up by UART\n");
    Process_Received_Data();
  }

  if (__HAL_RTC_ALARM_GET_FLAG(&hrtc, RTC_FLAG_ALRAF))
  {
    printf("Woken up by RTC Alarm\n");
    __HAL_RTC_ALARM_CLEAR_FLAG(&hrtc, RTC_FLAG_ALRAF);
  }
}
```

### 2. FIFO 상태 확인

```c
void Check_FIFO_Status(void)
{
  uint32_t isr = huart3.Instance->ISR;

  printf("\n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n");
  printf("UART FIFO Status:\n");
  printf("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n");

  // RX FIFO
  if (isr & USART_ISR_RXFF)
    printf("RX FIFO: Full\n");
  else if (isr & USART_ISR_RXFT)
    printf("RX FIFO: Above threshold\n");
  else if (isr & USART_ISR_RXFNE)
    printf("RX FIFO: Not empty\n");
  else
    printf("RX FIFO: Empty\n");

  // TX FIFO
  if (isr & USART_ISR_TXFE)
    printf("TX FIFO: Empty\n");
  else if (isr & USART_ISR_TXFT)
    printf("TX FIFO: Below threshold\n");
  else if (isr & USART_ISR_TXFNF)
    printf("TX FIFO: Not full\n");
  else
    printf("TX FIFO: Full\n");

  printf("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n\n");
}
```

### 3. DMA와 FIFO 조합

```c
DMA_HandleTypeDef hdma_usart3_rx;

void UART_DMA_FIFO_Init(void)
{
  // DMA 클럭 활성화
  __HAL_RCC_DMA1_CLK_ENABLE();

  // DMA 설정
  hdma_usart3_rx.Instance = DMA1_Stream0;
  hdma_usart3_rx.Init.Request = DMA_REQUEST_USART3_RX;
  hdma_usart3_rx.Init.Direction = DMA_PERIPH_TO_MEMORY;
  hdma_usart3_rx.Init.PeriphInc = DMA_PINC_DISABLE;
  hdma_usart3_rx.Init.MemInc = DMA_MINC_ENABLE;
  hdma_usart3_rx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
  hdma_usart3_rx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
  hdma_usart3_rx.Init.Mode = DMA_CIRCULAR;
  hdma_usart3_rx.Init.Priority = DMA_PRIORITY_HIGH;

  HAL_DMA_Init(&hdma_usart3_rx);

  // UART와 DMA 연결
  __HAL_LINKDMA(&huart3, hdmarx, hdma_usart3_rx);

  // DMA 수신 시작
  HAL_UART_Receive_DMA(&huart3, rx_buffer, RX_BUFFER_SIZE);

  printf("UART DMA + FIFO mode enabled\n");
}
```

### 4. 타임아웃 기능

```c
void UART_FIFO_With_Timeout(void)
{
  // RX 타임아웃 활성화
  // 마지막 바이트 수신 후 일정 시간 경과 시 인터럽트 발생
  HAL_UART_ReceiverTimeout_Config(&huart3, 100);  // 100 비트 시간
  HAL_UART_EnableReceiverTimeout(&huart3);

  // 타임아웃 인터럽트 활성화
  __HAL_UART_ENABLE_IT(&huart3, UART_IT_RTO);

  printf("RX Timeout enabled (100 bit times)\n");
}

// 타임아웃 인터럽트 처리
void USART3_IRQHandler(void)
{
  // ... (기존 코드)

  // RX 타임아웃
  if (__HAL_UART_GET_FLAG(&huart3, UART_FLAG_RTOF) != RESET)
  {
    printf("RX Timeout occurred\n");

    // FIFO에 남은 데이터 읽기
    while (__HAL_UART_GET_FLAG(&huart3, UART_FLAG_RXFNE) != RESET)
    {
      uint8_t data;
      HAL_UART_Receive(&huart3, &data, 1, 0);
      rx_buffer[rx_index++] = data;
    }

    // 타임아웃 플래그 클리어
    __HAL_UART_CLEAR_FLAG(&huart3, UART_CLEAR_RTOF);
  }
}
```

## 빌드 및 실행

### 테스트 시나리오

```
1. 보드 플래싱 및 리셋
2. 시리얼 터미널 연결 (9600 baud)
3. 메시지 확인: "Entering STOP mode..."
4. LED2 켜짐 (Stop 모드)
5. 시리얼 터미널에서 4자 이상 입력 (예: "test")
6. LED1 켜짐 (웨이크업)
7. 메시지: "Woke up from STOP mode!"
8. 수신 데이터 출력: "Received 4 bytes: test"
9. 2-8 반복
```

### 전류 측정

```
IDD 점퍼를 제거하고 전류계 연결

예상 전류:
- Run 모드 (1초): ~100 mA
- Stop 모드 (대기): ~100 μA
- Wakeup (10ms): ~100 mA
```

## 트러블슈팅

### FIFO가 작동하지 않는 경우

```c
// FIFO 활성화 확인
if (huart3.Instance->CR1 & USART_CR1_FIFOEN)
{
  printf("FIFO enabled\n");
}
else
{
  printf("FIFO disabled!\n");
  HAL_UARTEx_EnableFifoMode(&huart3);
}
```

### Stop 모드에서 웨이크업되지 않는 경우

```c
// UART Stop 모드 확인
if (huart3.Instance->CR3 & USART_CR3_UESM)
{
  printf("UART Stop mode enabled\n");
}
else
{
  printf("UART Stop mode disabled!\n");
  HAL_UARTEx_EnableStopMode(&huart3);
}

// 웨이크업 플래그 확인
if (__HAL_UART_GET_FLAG(&huart3, UART_FLAG_WUF))
{
  printf("Wakeup flag set\n");
  __HAL_UART_CLEAR_FLAG(&huart3, UART_CLEAR_WUF);
}
```

### 데이터 손실

```c
// FIFO 오버런 확인
if (__HAL_UART_GET_FLAG(&huart3, UART_FLAG_ORE))
{
  printf("FIFO Overrun detected!\n");

  // 오버런 플래그 클리어
  __HAL_UART_CLEAR_FLAG(&huart3, UART_CLEAR_OREF);

  // 해결: FIFO 임계값 낮추기 또는 DMA 사용
}
```

## 참고 자료

- **RM0399**: STM32H745 Reference Manual
  - Chapter 55: USART
  - Chapter 7: PWR
- **AN5224**: STM32H7 power management
- **예제 코드**: `STM32Cube_FW_H7/Projects/STM32H745I-DISCO/Examples/UART/UART_WakeUpFromStopUsingFIFO`

## 관련 예제

- **UART_HyperTerminal_IT**: 기본 UART 인터럽트
- **UART_HyperTerminal_DMA**: UART DMA 전송
- **PWR_STOP_RTC**: Stop 모드 기본
- **PWR_STANDBY**: Standby 모드
