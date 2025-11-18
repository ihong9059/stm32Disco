# eMMC 읽기/쓰기 (인터럽트 모드)

## 개요

이 예제는 STM32H745I-DISCO 보드의 eMMC 스토리지에 인터럽트 모드를 사용하여 데이터를 읽고 쓰는 방법을 보여줍니다. 인터럽트 모드는 DMA 없이 CPU가 데이터 전송을 처리하며, 전송 완료 시 인터럽트를 통해 알림을 받습니다.

## 주요 기능

### 1. 인터럽트 모드 특징
- **CPU 기반 전송**: FIFO를 통한 데이터 송수신
- **비동기 동작**: 전송 중 다른 작업 가능
- **인터럽트 알림**: 전송 완료/오류 시 즉시 통지
- **메모리 절약**: DMA 버퍼 불필요

### 2. DMA 모드와의 비교
```
인터럽트 모드:
  - 장점: 구현 간단, DMA 채널 불필요
  - 단점: CPU 부하 높음, 속도 느림 (20-30 MB/s)
  - 용도: 소량 데이터, DMA 채널 부족 시

DMA 모드:
  - 장점: CPU 부하 없음, 고속 (40-50 MB/s)
  - 단점: 캐시 관리 복잡, DMA 채널 필요
  - 용도: 대용량 데이터, 고성능 요구 시
```

## 코드 상세 분석

### 1. eMMC 초기화 (인터럽트 모드)

```c
MMC_HandleTypeDef MMCHandle;

void MMC_Init_IT(void)
{
  MMCHandle.Instance = SDMMC1;
  MMCHandle.Init.ClockEdge = SDMMC_CLOCK_EDGE_RISING;
  MMCHandle.Init.ClockPowerSave = SDMMC_CLOCK_POWER_SAVE_DISABLE;
  MMCHandle.Init.BusWide = SDMMC_BUS_WIDE_8B;
  MMCHandle.Init.HardwareFlowControl = SDMMC_HARDWARE_FLOW_CONTROL_ENABLE;
  MMCHandle.Init.ClockDiv = 2;  // 50 MHz

  if (HAL_MMC_Init(&MMCHandle) != HAL_OK)
  {
    Error_Handler();
  }

  // 8-bit 버스 설정
  if (HAL_MMC_ConfigWideBusOperation(&MMCHandle, SDMMC_BUS_WIDE_8B) != HAL_OK)
  {
    Error_Handler();
  }

  // SDMMC 인터럽트 활성화
  HAL_NVIC_SetPriority(SDMMC1_IRQn, 5, 0);
  HAL_NVIC_EnableIRQ(SDMMC1_IRQn);
}
```

### 2. 인터럽트 기반 쓰기

```c
uint8_t aTxBuffer[512];
volatile uint8_t TxComplete = 0;

void MMC_Write_IT_Example(void)
{
  // 버퍼 준비
  for (uint16_t i = 0; i < 512; i++)
  {
    aTxBuffer[i] = i & 0xFF;
  }

  // 인터럽트 쓰기 시작 (단일 블록)
  TxComplete = 0;
  if (HAL_MMC_WriteBlocks_IT(&MMCHandle, aTxBuffer, 0, 1) != HAL_OK)
  {
    Error_Handler();
  }

  // 전송 완료 대기 (인터럽트 방식)
  while (TxComplete == 0)
  {
    // 여기서 다른 작업 수행 가능
    // 예: LED 토글, 센서 읽기 등
  }

  printf("Block written successfully\n");
}

// 다중 블록 쓰기 (10 블록)
void MMC_Write_MultiBlock_IT(void)
{
  uint8_t buffer[5120];  // 10 블록 = 5120 바이트

  for (uint16_t i = 0; i < 5120; i++)
  {
    buffer[i] = i & 0xFF;
  }

  TxComplete = 0;
  if (HAL_MMC_WriteBlocks_IT(&MMCHandle, buffer, 100, 10) != HAL_OK)
  {
    Error_Handler();
  }

  while (TxComplete == 0);

  printf("10 blocks written\n");
}

// 쓰기 완료 콜백
void HAL_MMC_TxCpltCallback(MMC_HandleTypeDef *hmmc)
{
  TxComplete = 1;
  BSP_LED_On(LED_GREEN);
}
```

### 3. 인터럽트 기반 읽기

```c
uint8_t aRxBuffer[512];
volatile uint8_t RxComplete = 0;

void MMC_Read_IT_Example(void)
{
  // 수신 버퍼 초기화
  memset(aRxBuffer, 0, 512);

  // 인터럽트 읽기 시작
  RxComplete = 0;
  if (HAL_MMC_ReadBlocks_IT(&MMCHandle, aRxBuffer, 0, 1) != HAL_OK)
  {
    Error_Handler();
  }

  // 전송 완료 대기
  while (RxComplete == 0)
  {
    // 다른 작업 가능
  }

  // 데이터 검증
  for (uint16_t i = 0; i < 512; i++)
  {
    if (aRxBuffer[i] != (i & 0xFF))
    {
      printf("Verification failed at byte %d\n", i);
      return;
    }
  }

  printf("Block read and verified successfully\n");
}

// 읽기 완료 콜백
void HAL_MMC_RxCpltCallback(MMC_HandleTypeDef *hmmc)
{
  RxComplete = 1;
  BSP_LED_On(LED_GREEN);
}
```

### 4. 오류 처리

```c
// 오류 콜백
void HAL_MMC_ErrorCallback(MMC_HandleTypeDef *hmmc)
{
  uint32_t error_code = HAL_MMC_GetError(hmmc);

  if (error_code & HAL_MMC_ERROR_CMD_CRC_FAIL)
  {
    printf("Command CRC failed\n");
  }
  else if (error_code & HAL_MMC_ERROR_DATA_CRC_FAIL)
  {
    printf("Data CRC failed\n");
  }
  else if (error_code & HAL_MMC_ERROR_CMD_RSP_TIMEOUT)
  {
    printf("Command response timeout\n");
  }
  else if (error_code & HAL_MMC_ERROR_DATA_TIMEOUT)
  {
    printf("Data timeout\n");
  }

  BSP_LED_On(LED_RED);
  Error_Handler();
}

// SDMMC 인터럽트 핸들러
void SDMMC1_IRQHandler(void)
{
  HAL_MMC_IRQHandler(&MMCHandle);
}
```

### 5. 상태 모니터링

```c
void MMC_MonitorStatus(void)
{
  HAL_MMC_CardStateTypeDef card_state;

  card_state = HAL_MMC_GetCardState(&MMCHandle);

  switch (card_state)
  {
    case HAL_MMC_CARD_READY:
      printf("Card ready\n");
      break;

    case HAL_MMC_CARD_IDENTIFICATION:
      printf("Card identification\n");
      break;

    case HAL_MMC_CARD_STANDBY:
      printf("Card standby\n");
      break;

    case HAL_MMC_CARD_TRANSFER:
      printf("Card transfer mode\n");
      break;

    case HAL_MMC_CARD_SENDING:
      printf("Card sending data\n");
      break;

    case HAL_MMC_CARD_RECEIVING:
      printf("Card receiving data\n");
      break;

    case HAL_MMC_CARD_PROGRAMMING:
      printf("Card programming\n");
      break;

    case HAL_MMC_CARD_ERROR:
      printf("Card error!\n");
      break;
  }
}
```

## 성능 분석

### 전송 속도 비교
```
인터럽트 모드:
  쓰기: 20-25 MB/s
  읽기: 25-30 MB/s
  CPU 사용률: 40-60%

DMA 모드:
  쓰기: 35-40 MB/s
  읽기: 45-50 MB/s
  CPU 사용률: 5-10%

결론:
  - 소량 데이터 (< 10 KB): 인터럽트 모드
  - 대량 데이터 (> 1 MB): DMA 모드
```

## 고급 사용 예제

### 1. 비동기 읽기/쓰기

```c
typedef enum {
  MMC_IDLE,
  MMC_WRITING,
  MMC_READING,
  MMC_COMPLETE,
  MMC_ERROR
} MMC_State_t;

volatile MMC_State_t mmc_state = MMC_IDLE;

void MMC_AsyncWrite(uint8_t *data, uint32_t address, uint32_t blocks)
{
  if (mmc_state != MMC_IDLE)
  {
    printf("MMC busy\n");
    return;
  }

  mmc_state = MMC_WRITING;

  if (HAL_MMC_WriteBlocks_IT(&MMCHandle, data, address, blocks) != HAL_OK)
  {
    mmc_state = MMC_ERROR;
  }
}

void MMC_AsyncRead(uint8_t *data, uint32_t address, uint32_t blocks)
{
  if (mmc_state != MMC_IDLE)
  {
    printf("MMC busy\n");
    return;
  }

  mmc_state = MMC_READING;

  if (HAL_MMC_ReadBlocks_IT(&MMCHandle, data, address, blocks) != HAL_OK)
  {
    mmc_state = MMC_ERROR;
  }
}

void HAL_MMC_TxCpltCallback(MMC_HandleTypeDef *hmmc)
{
  mmc_state = MMC_COMPLETE;
}

void HAL_MMC_RxCpltCallback(MMC_HandleTypeDef *hmmc)
{
  mmc_state = MMC_COMPLETE;
}

void HAL_MMC_ErrorCallback(MMC_HandleTypeDef *hmmc)
{
  mmc_state = MMC_ERROR;
}

// 메인 루프
void MainLoop(void)
{
  while (1)
  {
    switch (mmc_state)
    {
      case MMC_IDLE:
        // MMC가 유휴 상태, 새 작업 시작 가능
        break;

      case MMC_WRITING:
      case MMC_READING:
        // 전송 진행 중, 다른 작업 수행
        ProcessSensorData();
        UpdateDisplay();
        break;

      case MMC_COMPLETE:
        printf("Transfer complete\n");
        mmc_state = MMC_IDLE;
        break;

      case MMC_ERROR:
        printf("Transfer error\n");
        mmc_state = MMC_IDLE;
        break;
    }
  }
}
```

### 2. 진행률 모니터링

```c
typedef struct {
  uint32_t total_blocks;
  uint32_t current_block;
  uint8_t  progress_percent;
} MMC_Progress_t;

MMC_Progress_t mmc_progress;

void MMC_Write_WithProgress(uint8_t *data, uint32_t address, uint32_t blocks)
{
  mmc_progress.total_blocks = blocks;
  mmc_progress.current_block = 0;
  mmc_progress.progress_percent = 0;

  for (uint32_t i = 0; i < blocks; i++)
  {
    TxComplete = 0;

    if (HAL_MMC_WriteBlocks_IT(&MMCHandle, &data[i * 512], address + i, 1) != HAL_OK)
    {
      Error_Handler();
    }

    while (TxComplete == 0);

    // 진행률 업데이트
    mmc_progress.current_block = i + 1;
    mmc_progress.progress_percent = ((i + 1) * 100) / blocks;

    // LCD에 진행률 표시
    LCD_DisplayProgress(mmc_progress.progress_percent);

    printf("Progress: %d%%\r", mmc_progress.progress_percent);
  }

  printf("\nWrite complete\n");
}
```

## 트러블슈팅

### 1. 인터럽트가 호출되지 않음
```c
// NVIC 설정 확인
HAL_NVIC_SetPriority(SDMMC1_IRQn, 5, 0);
HAL_NVIC_EnableIRQ(SDMMC1_IRQn);

// 인터럽트 핸들러 확인
void SDMMC1_IRQHandler(void)
{
  HAL_MMC_IRQHandler(&MMCHandle);
}
```

### 2. 전송 타임아웃
```c
// 카드 상태 확인
if (HAL_MMC_GetCardState(&MMCHandle) != HAL_MMC_CARD_TRANSFER)
{
  printf("Card not in transfer state\n");
}

// 타임아웃 증가
#define MMC_TIMEOUT  10000  // 10초
```

### 3. 데이터 손상
```c
// 버퍼 정렬 확인 (4바이트 경계)
__attribute__((aligned(4))) uint8_t buffer[512];

// 전송 완료 대기 시간 충분히 설정
while (TxComplete == 0)
{
  HAL_Delay(1);
}
```

## 참고 자료
- **RM0399**: STM32H7 Reference Manual (SDMMC 인터럽트)
- HAL 드라이버: `stm32h7xx_hal_mmc.c/h`
- eMMC 표준: JEDEC JESD84-B51
