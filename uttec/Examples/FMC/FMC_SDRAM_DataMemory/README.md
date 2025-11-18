# FMC SDRAM 데이터 메모리 - IS42S32800G (8MB)

## 개요

이 예제는 STM32H745I-DISCO 보드의 FMC(Flexible Memory Controller)를 사용하여 외부 SDRAM(IS42S32800G, 8MB)을 데이터 메모리로 사용하는 방법을 보여줍니다. SDRAM을 일반 RAM처럼 사용하여 변수 할당, 스택 배치, 힙 메모리 등에 활용할 수 있으며, 내부 RAM보다 훨씬 큰 메모리 공간을 제공합니다.

## 주요 기능

### 1. SDRAM 메모리 활용
- **주소 범위**: 0xD0000000 ~ 0xD07FFFFF (8MB)
- **데이터 메모리**: 변수, 배열, 구조체 저장
- **스택/힙**: 메인 스택 포인터(MSP)를 SDRAM에 배치
- **프레임버퍼**: LCD 디스플레이용 대용량 버퍼

### 2. IS42S32800G SDRAM 사양
- **용량**: 8 Megabytes (64 Megabits)
- **구조**: 4 Banks × 4096 Rows × 256 Columns × 32 bits
- **인터페이스**: 32-bit 데이터 버스
- **클럭**: 최대 133 MHz (STM32H7에서 100 MHz 사용)
- **CAS 지연**: 2 또는 3 클럭
- **리프레시**: 4096 사이클 / 64ms

### 3. FMC 설정
- **메모리 뱅크**: FMC_SDRAM_BANK1
- **데이터 버스**: 32-bit
- **주소 버스**: Row 12비트, Column 8비트
- **리프레시 레이트**: 64ms마다 4096 사이클

## 하드웨어 구성

### SDRAM 메모리 맵
```
┌──────────────────────────────────────────┐
│  STM32H745I SDRAM 메모리 맵              │
├──────────────────────────────────────────┤
│  0xD0000000 - 0xD07FFFFF                 │
│  IS42S32800G SDRAM (8 MB)                │
│                                          │
│  ┌────────────────────────────────┐     │
│  │  0xD0000000: 스택 시작 주소     │     │
│  ├────────────────────────────────┤     │
│  │  0xD0001000: 데이터 영역        │     │
│  ├────────────────────────────────┤     │
│  │  0xD0100000: LCD 프레임버퍼    │     │
│  ├────────────────────────────────┤     │
│  │  0xD0400000: 힙 메모리          │     │
│  ├────────────────────────────────┤     │
│  │  0xD07FFFFF: 끝 주소            │     │
│  └────────────────────────────────┘     │
└──────────────────────────────────────────┘

내부 메모리 비교:
  AXI-SRAM:   512 KB (0x24000000)
  SRAM1:      128 KB (0x30000000)
  SRAM2:      128 KB (0x30020000)
  SRAM3:      32 KB (0x30040000)
  SRAM4:      64 KB (0x38000000)
  -----------------------------------
  총 내부 RAM: 864 KB

  외부 SDRAM: 8192 KB (약 9.5배)
```

### FMC 신호 핀 배치
```
┌────────────────────────────────────────────────┐
│  FMC SDRAM 신호                               │
├────────────────────────────────────────────────┤
│  주소 버스 (A0-A11):                          │
│    FMC_A0  (PF0)  - FMC_A11  (PG1)            │
│                                                │
│  데이터 버스 (D0-D31):                        │
│    FMC_D0  (PD14) - FMC_D15  (PD10)           │
│    FMC_D16 (PH8)  - FMC_D31  (PI10)           │
│                                                │
│  뱅크 어드레스 (BA0-BA1):                     │
│    FMC_BA0 (PG4)  - FMC_BA1  (PG5)            │
│                                                │
│  제어 신호:                                    │
│    FMC_SDNWE (PC0)  - Write Enable            │
│    FMC_SDNRAS (PF11) - Row Address Strobe     │
│    FMC_SDNCAS (PG15) - Column Address Strobe  │
│    FMC_SDNE0 (PH3)  - Chip Enable (Bank 1)    │
│    FMC_SDCKE0 (PH2) - Clock Enable            │
│    FMC_SDCLK (PG8)  - SDRAM Clock (100 MHz)   │
│                                                │
│  바이트 마스크 (NBL0-NBL3):                   │
│    FMC_NBL0 (PE0)  - Byte 0 (D0-D7)           │
│    FMC_NBL1 (PE1)  - Byte 1 (D8-D15)          │
│    FMC_NBL2 (PI4)  - Byte 2 (D16-D23)         │
│    FMC_NBL3 (PI5)  - Byte 3 (D24-D31)         │
└────────────────────────────────────────────────┘
```

### SDRAM 내부 구조
```
IS42S32800G (8 MB = 2M × 32-bit)

4 Banks (BA0-BA1):
  Bank 0: 0xD0000000 - 0xD01FFFFF (2 MB)
  Bank 1: 0xD0200000 - 0xD03FFFFF (2 MB)
  Bank 2: 0xD0400000 - 0xD05FFFFF (2 MB)
  Bank 3: 0xD0600000 - 0xD07FFFFF (2 MB)

각 뱅크:
  4096 Rows (A0-A11)
  256 Columns (A0-A7)
  32-bit word

총 용량:
  4 banks × 4096 rows × 256 columns × 32 bits
  = 4 × 4096 × 256 × 4 bytes
  = 16,777,216 bytes
  = 16 MB (칩 스펙)

실제 사용:
  STM32H745I-DISCO는 8MB 구성 사용
  (뱅크 수 또는 행/열 설정 조정)
```

## 클럭 구성

### FMC 클럭 설정
```c
// FMC 클럭 = HCLK = 200 MHz / 2 = 100 MHz
// SDRAM 클럭 = FMC 클럭 = 100 MHz
// 클럭 주기 = 10 ns

System Clock Configuration:
  SYSCLK (CM7):  400 MHz
  HCLK (AXI):    200 MHz
  FMC_CLK:       100 MHz (HCLK / 2)
  SDRAM_CLK:     100 MHz
```

### 타이밍 계산
```c
// SDRAM IS42S32800G @ 100 MHz
// 클럭 주기 (tCK) = 10 ns

타이밍 파라미터:
  tMRD  = 2 클럭   (Mode Register 설정 시간)
  tXSR  = 7 클럭   (Self-refresh Exit 시간)
  tRAS  = 4 클럭   (Row Active Time: 42ns ≥ 40ns)
  tRC   = 6 클럭   (Row Cycle Time: 60ns ≥ 60ns)
  tWR   = 2 클럭   (Write Recovery Time: 20ns ≥ 14ns)
  tRP   = 2 클럭   (Row Precharge Time: 20ns ≥ 18ns)
  tRCD  = 2 클럭   (RAS to CAS Delay: 20ns ≥ 18ns)

리프레시:
  리프레시 주기: 64 ms
  리프레시 사이클: 4096
  리프레시 간격: 64 ms / 4096 = 15.625 μs
  리프레시 카운트: 15.625 μs / 10 ns = 1562 클럭
  안전 마진: 1562 - 20 = 1542
```

## 코드 상세 분석

### 1. FMC SDRAM 초기화 (HAL MSP)

```c
// stm32h7xx_hal_msp.c

void HAL_SDRAM_MspInit(SDRAM_HandleTypeDef* hsdram)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  // FMC 클럭 활성화
  __HAL_RCC_FMC_CLK_ENABLE();

  // GPIO 클럭 활성화
  __HAL_RCC_GPIOD_CLK_ENABLE();
  __HAL_RCC_GPIOE_CLK_ENABLE();
  __HAL_RCC_GPIOF_CLK_ENABLE();
  __HAL_RCC_GPIOG_CLK_ENABLE();
  __HAL_RCC_GPIOH_CLK_ENABLE();
  __HAL_RCC_GPIOI_CLK_ENABLE();

  // 데이터 버스 (D0-D31) 설정
  // PD14-PD15: D0-D1
  // PD0-PD1: D2-D3
  // PE7-PE15: D4-D12
  // PD8-PD10: D13-D15
  // PH8-PH15: D16-D23
  // PI0-PI3, PI6-PI9: D24-D31
  GPIO_InitStruct.Pin = GPIO_PIN_0 | GPIO_PIN_1 | GPIO_PIN_8 |
                        GPIO_PIN_9 | GPIO_PIN_10 | GPIO_PIN_14 | GPIO_PIN_15;
  GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;  // 최대 속도
  GPIO_InitStruct.Alternate = GPIO_AF12_FMC;
  HAL_GPIO_Init(GPIOD, &GPIO_InitStruct);

  // 주소 버스 (A0-A11) 설정
  GPIO_InitStruct.Pin = GPIO_PIN_0 | GPIO_PIN_1 | GPIO_PIN_2 |
                        GPIO_PIN_3 | GPIO_PIN_4 | GPIO_PIN_5 |
                        GPIO_PIN_12 | GPIO_PIN_13 | GPIO_PIN_14 | GPIO_PIN_15;
  GPIO_InitStruct.Alternate = GPIO_AF12_FMC;
  HAL_GPIO_Init(GPIOF, &GPIO_InitStruct);

  // 뱅크 어드레스 (BA0-BA1) 설정
  GPIO_InitStruct.Pin = GPIO_PIN_4 | GPIO_PIN_5;
  GPIO_InitStruct.Alternate = GPIO_AF12_FMC;
  HAL_GPIO_Init(GPIOG, &GPIO_InitStruct);

  // 제어 신호 설정
  // SDNWE (Write Enable)
  GPIO_InitStruct.Pin = GPIO_PIN_0;
  GPIO_InitStruct.Alternate = GPIO_AF12_FMC;
  HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);

  // SDNE0 (Chip Enable), SDCKE0 (Clock Enable)
  GPIO_InitStruct.Pin = GPIO_PIN_2 | GPIO_PIN_3;
  GPIO_InitStruct.Alternate = GPIO_AF12_FMC;
  HAL_GPIO_Init(GPIOH, &GPIO_InitStruct);

  // SDNRAS (Row Address Strobe)
  GPIO_InitStruct.Pin = GPIO_PIN_11;
  GPIO_InitStruct.Alternate = GPIO_AF12_FMC;
  HAL_GPIO_Init(GPIOF, &GPIO_InitStruct);

  // SDNCAS (Column Address Strobe), SDCLK (Clock)
  GPIO_InitStruct.Pin = GPIO_PIN_8 | GPIO_PIN_15;
  GPIO_InitStruct.Alternate = GPIO_AF12_FMC;
  HAL_GPIO_Init(GPIOG, &GPIO_InitStruct);

  // 바이트 마스크 (NBL0-NBL3) 설정
  GPIO_InitStruct.Pin = GPIO_PIN_0 | GPIO_PIN_1;
  GPIO_InitStruct.Alternate = GPIO_AF12_FMC;
  HAL_GPIO_Init(GPIOE, &GPIO_InitStruct);

  GPIO_InitStruct.Pin = GPIO_PIN_4 | GPIO_PIN_5;
  GPIO_InitStruct.Alternate = GPIO_AF12_FMC;
  HAL_GPIO_Init(GPIOI, &GPIO_InitStruct);
}
```

### 2. BSP SDRAM 초기화

```c
// BSP/STM32H745I-DISCO/stm32h745i_disco_sdram.c

int32_t BSP_SDRAM_Init(uint32_t Instance)
{
  SDRAM_HandleTypeDef hsdram;
  FMC_SDRAM_TimingTypeDef Timing;
  FMC_SDRAM_CommandTypeDef Command;

  // SDRAM 핸들 설정
  hsdram.Instance = FMC_SDRAM_DEVICE;

  // SDRAM 제어 설정
  hsdram.Init.SDBank = FMC_SDRAM_BANK1;              // Bank 1 사용
  hsdram.Init.ColumnBitsNumber = FMC_SDRAM_COLUMN_BITS_NUM_8;  // 8비트 컬럼 (256개)
  hsdram.Init.RowBitsNumber = FMC_SDRAM_ROW_BITS_NUM_12;       // 12비트 행 (4096개)
  hsdram.Init.MemoryDataWidth = FMC_SDRAM_MEM_BUS_WIDTH_32;    // 32비트 데이터 버스
  hsdram.Init.InternalBankNumber = FMC_SDRAM_INTERN_BANKS_NUM_4;  // 4개 내부 뱅크
  hsdram.Init.CASLatency = FMC_SDRAM_CAS_LATENCY_3;            // CAS 지연 3 클럭
  hsdram.Init.WriteProtection = FMC_SDRAM_WRITE_PROTECTION_DISABLE;  // 쓰기 보호 해제
  hsdram.Init.SDClockPeriod = FMC_SDRAM_CLOCK_PERIOD_2;        // HCLK / 2 = 100 MHz
  hsdram.Init.ReadBurst = FMC_SDRAM_RBURST_ENABLE;             // 읽기 버스트 활성화
  hsdram.Init.ReadPipeDelay = FMC_SDRAM_RPIPE_DELAY_0;         // 파이프라인 지연 없음

  // SDRAM 타이밍 설정 (100 MHz 기준)
  Timing.LoadToActiveDelay = 2;       // tMRD: 2 클럭
  Timing.ExitSelfRefreshDelay = 7;    // tXSR: 7 클럭 (70ns)
  Timing.SelfRefreshTime = 4;         // tRAS: 4 클럭 (40ns)
  Timing.RowCycleDelay = 6;           // tRC: 6 클럭 (60ns)
  Timing.WriteRecoveryTime = 2;       // tWR: 2 클럭 (20ns)
  Timing.RPDelay = 2;                 // tRP: 2 클럭 (20ns)
  Timing.RCDDelay = 2;                // tRCD: 2 클럭 (20ns)

  // FMC SDRAM 초기화
  if (HAL_SDRAM_Init(&hsdram, &Timing) != HAL_OK)
  {
    return BSP_ERROR_PERIPH_FAILURE;
  }

  // SDRAM 초기화 시퀀스
  // 1. Clock Enable Command
  Command.CommandMode = FMC_SDRAM_CMD_CLK_ENABLE;
  Command.CommandTarget = FMC_SDRAM_CMD_TARGET_BANK1;
  Command.AutoRefreshNumber = 1;
  Command.ModeRegisterDefinition = 0;
  HAL_SDRAM_SendCommand(&hsdram, &Command, SDRAM_TIMEOUT);

  // 100μs 대기 (전원 안정화)
  HAL_Delay(1);

  // 2. Precharge All Command
  Command.CommandMode = FMC_SDRAM_CMD_PALL;
  Command.CommandTarget = FMC_SDRAM_CMD_TARGET_BANK1;
  Command.AutoRefreshNumber = 1;
  Command.ModeRegisterDefinition = 0;
  HAL_SDRAM_SendCommand(&hsdram, &Command, SDRAM_TIMEOUT);

  // 3. Auto Refresh Command (2회)
  Command.CommandMode = FMC_SDRAM_CMD_AUTOREFRESH_MODE;
  Command.CommandTarget = FMC_SDRAM_CMD_TARGET_BANK1;
  Command.AutoRefreshNumber = 8;  // 8번 리프레시
  Command.ModeRegisterDefinition = 0;
  HAL_SDRAM_SendCommand(&hsdram, &Command, SDRAM_TIMEOUT);

  // 4. Load Mode Register
  // Mode Register:
  //   Burst Length = 1 (0x0000)
  //   Burst Type = Sequential (0x0000)
  //   CAS Latency = 3 (0x0030)
  //   Operating Mode = Standard (0x0000)
  //   Write Burst Mode = Single Write (0x0200)
  uint32_t tmpmrd = 0;
  tmpmrd = (uint32_t)SDRAM_MODEREG_BURST_LENGTH_1          |  // 버스트 길이 1
                     SDRAM_MODEREG_BURST_TYPE_SEQUENTIAL   |  // 순차 버스트
                     SDRAM_MODEREG_CAS_LATENCY_3           |  // CAS 지연 3
                     SDRAM_MODEREG_OPERATING_MODE_STANDARD |  // 표준 모드
                     SDRAM_MODEREG_WRITEBURST_MODE_SINGLE;    // 단일 쓰기

  Command.CommandMode = FMC_SDRAM_CMD_LOAD_MODE;
  Command.CommandTarget = FMC_SDRAM_CMD_TARGET_BANK1;
  Command.AutoRefreshNumber = 1;
  Command.ModeRegisterDefinition = tmpmrd;
  HAL_SDRAM_SendCommand(&hsdram, &Command, SDRAM_TIMEOUT);

  // 5. 리프레시 레이트 설정
  // 리프레시 간격 = 15.625 μs
  // 리프레시 카운트 = 15.625 μs × 100 MHz - 20 = 1542
  HAL_SDRAM_ProgramRefreshRate(&hsdram, 1542);

  return BSP_ERROR_NONE;
}
```

### 3. 메인 프로그램 - SDRAM 테스트

```c
// main.c

#define SDRAM_ADDRESS      0xD0000000
#define BUFFER_SIZE        1024

// SDRAM에 배치할 배열
// 링커 스크립트에서 .sdram 섹션을 0xD0000000에 매핑
uint32_t aTable[1024] __attribute__((section(".sdram")));

int main(void)
{
  uint32_t uwTabAddr;
  uint32_t MSPValue;

  // MPU 설정 (SDRAM 영역을 캐시 가능하게)
  MPU_Config();

  // CPU 캐시 활성화
  CPU_CACHE_Enable();

  // HAL 초기화
  HAL_Init();

  // 시스템 클럭 설정 (400 MHz)
  SystemClock_Config();

  // LED 초기화
  BSP_LED_Init(LED1);  // 성공 표시
  BSP_LED_Init(LED2);  // 실패 표시

  // SDRAM에 데이터 채우기
  // aTable[]은 SDRAM에 위치하므로 SDRAM 쓰기 테스트
  Fill_Buffer(aTable, BUFFER_SIZE, 0);

  // 버퍼 주소 확인
  uwTabAddr = (uint32_t)aTable;

  // 메인 스택 포인터 확인
  // 링커 스크립트에서 스택도 SDRAM에 배치됨
  MSPValue = __get_MSP();

  // SDRAM 영역 확인 (0xD0000000 이상이어야 함)
  if ((uwTabAddr >= SDRAM_ADDRESS) && (MSPValue >= SDRAM_ADDRESS))
  {
    // SDRAM에 성공적으로 배치됨
    BSP_LED_On(LED1);
  }
  else
  {
    // SDRAM 배치 실패
    BSP_LED_On(LED2);
  }

  // 무한 루프
  while (1)
  {
    // SDRAM 읽기/쓰기 테스트
    for (uint32_t i = 0; i < BUFFER_SIZE; i++)
    {
      // SDRAM 읽기
      uint32_t value = aTable[i];

      // 검증
      if (value != i)
      {
        BSP_LED_On(LED2);  // 오류 발생
        Error_Handler();
      }
    }

    HAL_Delay(1000);
  }
}

// 버퍼 채우기 함수
static void Fill_Buffer(uint32_t *pBuffer, uint32_t uwBufferLength, uint16_t uwOffset)
{
  uint16_t tmpIndex = 0;

  // 버퍼에 순차 데이터 쓰기
  for (tmpIndex = 0; tmpIndex < uwBufferLength; tmpIndex++)
  {
    pBuffer[tmpIndex] = tmpIndex + uwOffset;
  }
}
```

### 4. MPU 설정 (SDRAM 캐시 활성화)

```c
static void MPU_Config(void)
{
  MPU_Region_InitTypeDef MPU_InitStruct;

  // MPU 비활성화
  HAL_MPU_Disable();

  // 전체 메모리 영역 기본 설정
  MPU_InitStruct.Enable = MPU_REGION_ENABLE;
  MPU_InitStruct.BaseAddress = 0x00;
  MPU_InitStruct.Size = MPU_REGION_SIZE_4GB;
  MPU_InitStruct.AccessPermission = MPU_REGION_NO_ACCESS;
  MPU_InitStruct.IsBufferable = MPU_ACCESS_NOT_BUFFERABLE;
  MPU_InitStruct.IsCacheable = MPU_ACCESS_NOT_CACHEABLE;
  MPU_InitStruct.IsShareable = MPU_ACCESS_SHAREABLE;
  MPU_InitStruct.Number = MPU_REGION_NUMBER0;
  MPU_InitStruct.TypeExtField = MPU_TEX_LEVEL0;
  MPU_InitStruct.SubRegionDisable = 0x87;
  MPU_InitStruct.DisableExec = MPU_INSTRUCTION_ACCESS_DISABLE;
  HAL_MPU_ConfigRegion(&MPU_InitStruct);

  // SDRAM 영역 설정 (0xD0000000, 16MB)
  // Write-Back 캐시 활성화로 최대 성능 확보
  MPU_InitStruct.Enable = MPU_REGION_ENABLE;
  MPU_InitStruct.BaseAddress = SDRAM_ADDRESS;                  // 0xD0000000
  MPU_InitStruct.Size = MPU_REGION_SIZE_16MB;                  // 16MB 영역
  MPU_InitStruct.AccessPermission = MPU_REGION_FULL_ACCESS;    // 읽기/쓰기 가능
  MPU_InitStruct.IsBufferable = MPU_ACCESS_BUFFERABLE;         // 버퍼 가능 (쓰기 버퍼링)
  MPU_InitStruct.IsCacheable = MPU_ACCESS_CACHEABLE;           // 캐시 가능
  MPU_InitStruct.IsShareable = MPU_ACCESS_NOT_SHAREABLE;       // CM7 전용
  MPU_InitStruct.Number = MPU_REGION_NUMBER1;
  MPU_InitStruct.TypeExtField = MPU_TEX_LEVEL0;                // Write-Back 캐시
  MPU_InitStruct.SubRegionDisable = 0x00;                      // 모든 서브 영역 활성화
  MPU_InitStruct.DisableExec = MPU_INSTRUCTION_ACCESS_ENABLE;  // 코드 실행 가능
  HAL_MPU_ConfigRegion(&MPU_InitStruct);

  // MPU 활성화
  HAL_MPU_Enable(MPU_PRIVILEGED_DEFAULT);
}

/*
 * MPU 캐시 모드:
 *
 * Write-Back (가장 빠름):
 *   - 읽기: 캐시에서 읽음 (10 ns)
 *   - 쓰기: 캐시에만 씀 (10 ns), 나중에 SDRAM에 쓰여짐
 *   - 장점: 최고 성능
 *   - 단점: 캐시 일관성 관리 필요 (DMA 사용 시)
 *
 * Write-Through:
 *   - 읽기: 캐시에서 읽음 (10 ns)
 *   - 쓰기: 캐시와 SDRAM에 동시에 씀 (100 ns)
 *   - 장점: 캐시 일관성 자동 유지
 *   - 단점: 쓰기 성능 저하
 *
 * Non-Cacheable (가장 느림):
 *   - 읽기: SDRAM에서 직접 읽음 (100 ns)
 *   - 쓰기: SDRAM에 직접 씀 (100 ns)
 *   - 장점: 캐시 관리 불필요
 *   - 단점: 최저 성능
 */
```

### 5. 링커 스크립트 설정

```ld
/* STM32H745XIHX_FLASH.ld */

/* 메모리 정의 */
MEMORY
{
  FLASH (rx)      : ORIGIN = 0x08000000, LENGTH = 1024K  /* CM7 Flash */
  DTCMRAM (xrw)   : ORIGIN = 0x20000000, LENGTH = 128K   /* DTCM RAM */
  RAM_D1 (xrw)    : ORIGIN = 0x24000000, LENGTH = 512K   /* AXI SRAM */
  RAM_D2 (xrw)    : ORIGIN = 0x30000000, LENGTH = 288K   /* SRAM1+2+3 */
  RAM_D3 (xrw)    : ORIGIN = 0x38000000, LENGTH = 64K    /* SRAM4 */
  SDRAM (xrw)     : ORIGIN = 0xD0000000, LENGTH = 8192K  /* 외부 SDRAM */
  ITCMRAM (xrw)   : ORIGIN = 0x00000000, LENGTH = 64K    /* ITCM RAM */
}

/* 스택 크기 설정 */
_Min_Heap_Size = 0x8000;   /* 32 KB 힙 (SDRAM에 배치) */
_Min_Stack_Size = 0x8000;  /* 32 KB 스택 (SDRAM에 배치) */

SECTIONS
{
  /* 코드 섹션 (Flash) */
  .text :
  {
    . = ALIGN(4);
    *(.text)
    *(.text*)
    *(.rodata)
    *(.rodata*)
    . = ALIGN(4);
  } >FLASH

  /* 초기화된 데이터 (RAM_D1) */
  .data :
  {
    . = ALIGN(4);
    _sdata = .;
    *(.data)
    *(.data*)
    . = ALIGN(4);
    _edata = .;
  } >RAM_D1 AT> FLASH

  /* SDRAM 섹션 */
  .sdram (NOLOAD) :
  {
    . = ALIGN(4);
    _ssdram = .;
    *(.sdram)
    *(.sdram*)
    . = ALIGN(4);
    _esdram = .;
  } >SDRAM

  /* BSS 섹션 (SDRAM) */
  .bss :
  {
    . = ALIGN(4);
    _sbss = .;
    *(.bss)
    *(.bss*)
    *(COMMON)
    . = ALIGN(4);
    _ebss = .;
  } >SDRAM

  /* 힙 (SDRAM) */
  ._user_heap_stack :
  {
    . = ALIGN(8);
    PROVIDE ( end = . );
    PROVIDE ( _end = . );
    . = . + _Min_Heap_Size;
    . = . + _Min_Stack_Size;
    . = ALIGN(8);
  } >SDRAM

  /* 스택 끝 주소 (SDRAM 끝) */
  _estack = ORIGIN(SDRAM) + LENGTH(SDRAM);
}
```

## 성능 분석

### 메모리 액세스 시간

```
캐시 없음 (MPU: Non-Cacheable):
  읽기: 약 100 ns (10 FMC 클럭)
  쓰기: 약 100 ns (10 FMC 클럭)
  대역폭: 약 40 MB/s (32비트 @ 10 클럭)

캐시 있음 (MPU: Write-Back):
  캐시 히트 읽기: 약 3 ns (1 CPU 클럭)
  캐시 히트 쓰기: 약 3 ns (1 CPU 클럭)
  캐시 미스 읽기: 약 100 ns
  대역폭: 약 1000 MB/s (캐시 히트 시)

내부 RAM (AXI-SRAM):
  읽기: 약 5 ns (2 CPU 클럭)
  쓰기: 약 5 ns (2 CPU 클럭)
  대역폭: 약 800 MB/s

결론:
  - 캐시 활성화 시 SDRAM은 내부 RAM과 유사한 성능
  - 순차 액세스 시 캐시 라인 (32바이트) 단위로 읽으므로 효율적
  - 랜덤 액세스 시 캐시 미스 빈번하여 성능 저하
```

### 벤치마크

```c
// 순차 쓰기 테스트
void SDRAM_BenchmarkWrite(void)
{
  uint32_t *sdram = (uint32_t *)SDRAM_ADDRESS;
  uint32_t start_tick, end_tick;

  start_tick = HAL_GetTick();

  // 1 MB 쓰기
  for (uint32_t i = 0; i < 256 * 1024; i++)
  {
    sdram[i] = i;
  }

  // D-Cache 플러시 (Write-Back 캐시)
  SCB_CleanDCache_by_Addr((uint32_t *)sdram, 1024 * 1024);

  end_tick = HAL_GetTick();

  // 결과:
  // 캐시 없음: 약 25 ms (40 MB/s)
  // 캐시 있음: 약 2 ms (500 MB/s)
  printf("Write 1MB: %lu ms\n", end_tick - start_tick);
}

// 순차 읽기 테스트
void SDRAM_BenchmarkRead(void)
{
  uint32_t *sdram = (uint32_t *)SDRAM_ADDRESS;
  uint32_t start_tick, end_tick;
  uint32_t sum = 0;

  start_tick = HAL_GetTick();

  // 1 MB 읽기
  for (uint32_t i = 0; i < 256 * 1024; i++)
  {
    sum += sdram[i];
  }

  end_tick = HAL_GetTick();

  // 결과:
  // 캐시 없음: 약 25 ms (40 MB/s)
  // 캐시 있음: 약 1 ms (1000 MB/s)
  printf("Read 1MB: %lu ms (sum=%lu)\n", end_tick - start_tick, sum);
}

// 랜덤 액세스 테스트
void SDRAM_BenchmarkRandom(void)
{
  uint32_t *sdram = (uint32_t *)SDRAM_ADDRESS;
  uint32_t start_tick, end_tick;
  uint32_t sum = 0;

  start_tick = HAL_GetTick();

  // 10만 번 랜덤 읽기
  for (uint32_t i = 0; i < 100000; i++)
  {
    uint32_t index = (i * 7919) % (2 * 1024 * 1024);  // 의사 난수
    sum += sdram[index];
  }

  end_tick = HAL_GetTick();

  // 결과:
  // 캐시 없음: 약 25 ms
  // 캐시 있음: 약 15 ms (캐시 미스 빈번)
  printf("Random 100k: %lu ms (sum=%lu)\n", end_tick - start_tick, sum);
}
```

## 고급 사용 예제

### 1. DMA를 사용한 SDRAM 액세스

```c
DMA_HandleTypeDef hdma_memtomem;

void DMA_SDRAM_Init(void)
{
  // DMA 클럭 활성화
  __HAL_RCC_DMA1_CLK_ENABLE();

  // DMA 설정
  hdma_memtomem.Instance = DMA1_Stream0;
  hdma_memtomem.Init.Request = DMA_REQUEST_MEM2MEM;
  hdma_memtomem.Init.Direction = DMA_MEMORY_TO_MEMORY;
  hdma_memtomem.Init.PeriphInc = DMA_PINC_ENABLE;
  hdma_memtomem.Init.MemInc = DMA_MINC_ENABLE;
  hdma_memtomem.Init.PeriphDataAlignment = DMA_PDATAALIGN_WORD;
  hdma_memtomem.Init.MemDataAlignment = DMA_MDATAALIGN_WORD;
  hdma_memtomem.Init.Mode = DMA_NORMAL;
  hdma_memtomem.Init.Priority = DMA_PRIORITY_HIGH;
  hdma_memtomem.Init.FIFOMode = DMA_FIFOMODE_ENABLE;
  hdma_memtomem.Init.FIFOThreshold = DMA_FIFO_THRESHOLD_FULL;
  hdma_memtomem.Init.MemBurst = DMA_MBURST_INC16;
  hdma_memtomem.Init.PeriphBurst = DMA_PBURST_INC16;

  HAL_DMA_Init(&hdma_memtomem);
}

// 내부 RAM → SDRAM 복사
void DMA_CopyToSDRAM(uint32_t *src, uint32_t *dst, uint32_t size)
{
  // D-Cache 클린 (소스 데이터)
  SCB_CleanDCache_by_Addr((uint32_t *)src, size * 4);

  // DMA 전송 시작
  HAL_DMA_Start(&hdma_memtomem, (uint32_t)src, (uint32_t)dst, size);

  // 전송 완료 대기
  HAL_DMA_PollForTransfer(&hdma_memtomem, HAL_DMA_FULL_TRANSFER, HAL_MAX_DELAY);

  // D-Cache 무효화 (목적지 데이터)
  SCB_InvalidateDCache_by_Addr((uint32_t *)dst, size * 4);
}

// SDRAM → 내부 RAM 복사
void DMA_CopyFromSDRAM(uint32_t *src, uint32_t *dst, uint32_t size)
{
  // D-Cache 클린 (SDRAM 소스)
  SCB_CleanDCache_by_Addr((uint32_t *)src, size * 4);

  // DMA 전송
  HAL_DMA_Start(&hdma_memtomem, (uint32_t)src, (uint32_t)dst, size);
  HAL_DMA_PollForTransfer(&hdma_memtomem, HAL_DMA_FULL_TRANSFER, HAL_MAX_DELAY);

  // D-Cache 무효화 (내부 RAM 목적지)
  SCB_InvalidateDCache_by_Addr((uint32_t *)dst, size * 4);
}

// 성능 비교
void Compare_CPU_DMA_Copy(void)
{
  uint32_t src[1024];
  uint32_t *dst = (uint32_t *)SDRAM_ADDRESS;
  uint32_t start, end;

  // CPU 복사
  start = HAL_GetTick();
  memcpy(dst, src, 1024 * 4);
  SCB_CleanDCache_by_Addr((uint32_t *)dst, 1024 * 4);
  end = HAL_GetTick();
  printf("CPU copy: %lu ms\n", end - start);  // 약 0.1 ms

  // DMA 복사
  start = HAL_GetTick();
  DMA_CopyToSDRAM(src, dst, 1024);
  end = HAL_GetTick();
  printf("DMA copy: %lu ms\n", end - start);  // 약 0.2 ms

  // 소량 데이터는 CPU가 빠르지만, 대용량은 DMA가 유리
  // DMA 사용 시 CPU가 다른 작업 가능
}
```

### 2. SDRAM 테스트 패턴

```c
// 다양한 테스트 패턴으로 SDRAM 검증

// Walking 1s 테스트
int SDRAM_Test_Walking1(void)
{
  uint32_t *sdram = (uint32_t *)SDRAM_ADDRESS;

  for (uint8_t bit = 0; bit < 32; bit++)
  {
    uint32_t pattern = 1 << bit;

    // 쓰기
    for (uint32_t i = 0; i < 1024; i++)
    {
      sdram[i] = pattern;
    }

    // D-Cache 클린
    SCB_CleanDCache_by_Addr((uint32_t *)sdram, 1024 * 4);

    // D-Cache 무효화
    SCB_InvalidateDCache_by_Addr((uint32_t *)sdram, 1024 * 4);

    // 읽기 및 검증
    for (uint32_t i = 0; i < 1024; i++)
    {
      if (sdram[i] != pattern)
      {
        printf("Walking 1s failed at bit %d, address 0x%08lX\n", bit, (uint32_t)&sdram[i]);
        return -1;
      }
    }
  }

  printf("Walking 1s test PASSED\n");
  return 0;
}

// Address uniqueness 테스트
int SDRAM_Test_AddressUniqueness(void)
{
  uint32_t *sdram = (uint32_t *)SDRAM_ADDRESS;
  uint32_t size = 2 * 1024 * 1024;  // 8 MB / 4 = 2M words

  // 각 주소에 주소값 쓰기
  for (uint32_t i = 0; i < size; i++)
  {
    sdram[i] = (uint32_t)&sdram[i];
  }

  SCB_CleanDCache();

  // 검증
  for (uint32_t i = 0; i < size; i++)
  {
    if (sdram[i] != (uint32_t)&sdram[i])
    {
      printf("Address uniqueness failed at 0x%08lX\n", (uint32_t)&sdram[i]);
      return -1;
    }
  }

  printf("Address uniqueness test PASSED\n");
  return 0;
}

// Checkerboard 테스트
int SDRAM_Test_Checkerboard(void)
{
  uint32_t *sdram = (uint32_t *)SDRAM_ADDRESS;
  uint32_t pattern1 = 0xAAAAAAAA;
  uint32_t pattern2 = 0x55555555;

  // 패턴 1 쓰기
  for (uint32_t i = 0; i < 1024; i++)
  {
    sdram[i] = (i % 2) ? pattern1 : pattern2;
  }

  SCB_CleanDCache_by_Addr((uint32_t *)sdram, 1024 * 4);
  SCB_InvalidateDCache_by_Addr((uint32_t *)sdram, 1024 * 4);

  // 검증
  for (uint32_t i = 0; i < 1024; i++)
  {
    uint32_t expected = (i % 2) ? pattern1 : pattern2;
    if (sdram[i] != expected)
    {
      printf("Checkerboard failed at 0x%08lX\n", (uint32_t)&sdram[i]);
      return -1;
    }
  }

  // 패턴 반전
  for (uint32_t i = 0; i < 1024; i++)
  {
    sdram[i] = (i % 2) ? pattern2 : pattern1;
  }

  SCB_CleanDCache_by_Addr((uint32_t *)sdram, 1024 * 4);
  SCB_InvalidateDCache_by_Addr((uint32_t *)sdram, 1024 * 4);

  // 검증
  for (uint32_t i = 0; i < 1024; i++)
  {
    uint32_t expected = (i % 2) ? pattern2 : pattern1;
    if (sdram[i] != expected)
    {
      printf("Checkerboard (inverted) failed at 0x%08lX\n", (uint32_t)&sdram[i]);
      return -1;
    }
  }

  printf("Checkerboard test PASSED\n");
  return 0;
}

// 전체 SDRAM 테스트 스위트
void SDRAM_TestSuite(void)
{
  printf("Starting SDRAM test suite...\n");

  if (SDRAM_Test_Walking1() != 0) {
    BSP_LED_On(LED2);
    return;
  }

  if (SDRAM_Test_Checkerboard() != 0) {
    BSP_LED_On(LED2);
    return;
  }

  if (SDRAM_Test_AddressUniqueness() != 0) {
    BSP_LED_On(LED2);
    return;
  }

  printf("All SDRAM tests PASSED!\n");
  BSP_LED_On(LED1);
}
```

### 3. 동적 메모리 할당 (malloc/free)

```c
// SDRAM에서 malloc 사용

#include <stdlib.h>

void SDRAM_Malloc_Example(void)
{
  // 대용량 배열 동적 할당 (2 MB)
  uint32_t *large_buffer = (uint32_t *)malloc(2 * 1024 * 1024);

  if (large_buffer == NULL)
  {
    printf("malloc failed!\n");
    return;
  }

  // 버퍼 주소 확인 (SDRAM 영역이어야 함)
  printf("Buffer address: 0x%08lX\n", (uint32_t)large_buffer);

  if ((uint32_t)large_buffer >= SDRAM_ADDRESS)
  {
    printf("Buffer is in SDRAM\n");
  }

  // 버퍼 사용
  for (uint32_t i = 0; i < 512 * 1024; i++)
  {
    large_buffer[i] = i;
  }

  // 검증
  for (uint32_t i = 0; i < 512 * 1024; i++)
  {
    if (large_buffer[i] != i)
    {
      printf("Buffer verification failed!\n");
      break;
    }
  }

  // 메모리 해제
  free(large_buffer);

  printf("malloc/free test completed\n");
}

// 메모리 풀 구현
typedef struct {
  uint32_t *pool;
  uint32_t size;
  uint32_t used;
} MemoryPool_t;

MemoryPool_t sdram_pool;

void SDRAM_PoolInit(void)
{
  // SDRAM의 특정 영역을 메모리 풀로 사용
  sdram_pool.pool = (uint32_t *)0xD0400000;  // 4 MB 오프셋
  sdram_pool.size = 4 * 1024 * 1024;         // 4 MB
  sdram_pool.used = 0;
}

void* SDRAM_PoolAlloc(uint32_t size)
{
  // 4바이트 정렬
  size = (size + 3) & ~3;

  if (sdram_pool.used + size > sdram_pool.size)
  {
    return NULL;  // 메모리 부족
  }

  void *ptr = (void *)((uint32_t)sdram_pool.pool + sdram_pool.used);
  sdram_pool.used += size;

  return ptr;
}

void SDRAM_PoolReset(void)
{
  sdram_pool.used = 0;  // 풀 리셋
}
```

### 4. LCD 프레임버퍼 (SDRAM 사용)

```c
// LCD 프레임버퍼를 SDRAM에 배치

#define LCD_WIDTH  480
#define LCD_HEIGHT 272
#define LCD_BYTES_PER_PIXEL 4  // ARGB8888

#define LCD_FRAME_BUFFER  0xD0000000

void LCD_InitFrameBuffer(void)
{
  LTDC_LayerCfgTypeDef pLayerCfg;

  // SDRAM 초기화
  BSP_SDRAM_Init(0);

  // LTDC 초기화
  BSP_LCD_Init(0, LCD_ORIENTATION_LANDSCAPE);

  // 레이어 0 설정
  pLayerCfg.WindowX0 = 0;
  pLayerCfg.WindowX1 = LCD_WIDTH;
  pLayerCfg.WindowY0 = 0;
  pLayerCfg.WindowY1 = LCD_HEIGHT;
  pLayerCfg.PixelFormat = LTDC_PIXEL_FORMAT_ARGB8888;
  pLayerCfg.Alpha = 255;
  pLayerCfg.Alpha0 = 0;
  pLayerCfg.BlendingFactor1 = LTDC_BLENDING_FACTOR1_CA;
  pLayerCfg.BlendingFactor2 = LTDC_BLENDING_FACTOR2_CA;
  pLayerCfg.FBStartAdress = LCD_FRAME_BUFFER;  // SDRAM 주소
  pLayerCfg.ImageWidth = LCD_WIDTH;
  pLayerCfg.ImageHeight = LCD_HEIGHT;
  pLayerCfg.Backcolor.Blue = 0;
  pLayerCfg.Backcolor.Green = 0;
  pLayerCfg.Backcolor.Red = 0;

  HAL_LTDC_ConfigLayer(&LtdcHandle, &pLayerCfg, 0);

  // 프레임버퍼 초기화 (검은색)
  uint32_t *fb = (uint32_t *)LCD_FRAME_BUFFER;
  for (uint32_t i = 0; i < LCD_WIDTH * LCD_HEIGHT; i++)
  {
    fb[i] = 0xFF000000;  // 검은색 (알파 = 255)
  }

  // D-Cache 클린
  SCB_CleanDCache_by_Addr((uint32_t *)LCD_FRAME_BUFFER,
                          LCD_WIDTH * LCD_HEIGHT * LCD_BYTES_PER_PIXEL);
}

// 픽셀 그리기
void LCD_DrawPixel(uint16_t x, uint16_t y, uint32_t color)
{
  uint32_t *fb = (uint32_t *)LCD_FRAME_BUFFER;
  fb[y * LCD_WIDTH + x] = color;

  // D-Cache 클린 (해당 픽셀만)
  SCB_CleanDCache_by_Addr((uint32_t *)&fb[y * LCD_WIDTH + x], 4);
}

// 사각형 그리기
void LCD_FillRect(uint16_t x, uint16_t y, uint16_t w, uint16_t h, uint32_t color)
{
  uint32_t *fb = (uint32_t *)LCD_FRAME_BUFFER;

  for (uint16_t j = 0; j < h; j++)
  {
    for (uint16_t i = 0; i < w; i++)
    {
      fb[(y + j) * LCD_WIDTH + (x + i)] = color;
    }
  }

  // D-Cache 클린 (사각형 영역)
  SCB_CleanDCache_by_Addr((uint32_t *)&fb[y * LCD_WIDTH + x],
                          h * LCD_WIDTH * LCD_BYTES_PER_PIXEL);
}

// 프레임버퍼 메모리 사용량:
// 480 × 272 × 4 = 522,240 바이트 (약 510 KB)
```

## 트러블슈팅

### 1. SDRAM 초기화 실패

```c
// 증상: BSP_SDRAM_Init() 실패 또는 데이터 읽기/쓰기 오류

// 원인 1: FMC 클럭 비활성화
// 해결:
__HAL_RCC_FMC_CLK_ENABLE();

// 원인 2: GPIO 설정 오류
// - 핀 번호 확인
// - Alternate Function 확인 (AF12_FMC)
// - 속도 설정 (VERY_HIGH)

// 원인 3: 타이밍 파라미터 오류
// 해결: 데이터시트와 비교하여 타이밍 재계산
Timing.LoadToActiveDelay = 2;     // tMRD
Timing.ExitSelfRefreshDelay = 7;  // tXSR
Timing.SelfRefreshTime = 4;       // tRAS
Timing.RowCycleDelay = 6;         // tRC
Timing.WriteRecoveryTime = 2;     // tWR
Timing.RPDelay = 2;               // tRP
Timing.RCDDelay = 2;              // tRCD

// 원인 4: 초기화 시퀀스 누락
// 해결: 올바른 순서로 명령 실행
// 1) Clock Enable
// 2) Precharge All
// 3) Auto Refresh (8회)
// 4) Load Mode Register
// 5) Refresh Rate 설정
```

### 2. 데이터 손상 (캐시 관련)

```c
// 증상: 쓴 데이터와 읽은 데이터가 다름

// 원인 1: D-Cache 클린 누락
// 해결: 쓰기 후 D-Cache 클린
uint32_t *sdram = (uint32_t *)SDRAM_ADDRESS;
sdram[100] = 0x12345678;
SCB_CleanDCache_by_Addr((uint32_t *)&sdram[100], 4);

// 원인 2: D-Cache 무효화 누락 (DMA 후)
// 해결: DMA 전송 후 D-Cache 무효화
HAL_DMA_Start(&hdma, src, dst, size);
HAL_DMA_PollForTransfer(&hdma, HAL_DMA_FULL_TRANSFER, HAL_MAX_DELAY);
SCB_InvalidateDCache_by_Addr(dst, size * 4);

// 원인 3: MPU 설정 오류
// 해결: SDRAM 영역을 캐시 가능하게 설정
MPU_InitStruct.BaseAddress = SDRAM_ADDRESS;
MPU_InitStruct.IsCacheable = MPU_ACCESS_CACHEABLE;
MPU_InitStruct.IsBufferable = MPU_ACCESS_BUFFERABLE;
HAL_MPU_ConfigRegion(&MPU_InitStruct);
```

### 3. 성능 저하

```c
// 증상: SDRAM 액세스가 매우 느림

// 원인 1: 캐시 비활성화
// 해결: MPU로 SDRAM 영역을 캐시 가능하게 설정

// 원인 2: 읽기 버스트 비활성화
// 해결:
hsdram.Init.ReadBurst = FMC_SDRAM_RBURST_ENABLE;

// 원인 3: CAS 지연이 너무 큼
// 해결: CAS 지연 2 또는 3 사용
hsdram.Init.CASLatency = FMC_SDRAM_CAS_LATENCY_3;

// 원인 4: 클럭 속도가 낮음
// 해결: 최대 100 MHz 사용
hsdram.Init.SDClockPeriod = FMC_SDRAM_CLOCK_PERIOD_2;  // HCLK / 2
```

### 4. 리프레시 오류

```c
// 증상: 데이터가 시간이 지나면 손상됨

// 원인: 리프레시 레이트가 잘못 설정됨
// 해결: 올바른 리프레시 레이트 계산
// 리프레시 간격 = 64 ms / 4096 = 15.625 μs
// 리프레시 카운트 = 15.625 μs × 100 MHz - 20 = 1542
HAL_SDRAM_ProgramRefreshRate(&hsdram, 1542);

// 리프레시 레이트가 너무 낮으면:
// - 데이터 손상 (행 방전)
// 리프레시 레이트가 너무 높으면:
// - 성능 저하 (리프레시에 시간 소비)
```

### 5. 스택 오버플로우

```c
// 증상: 프로그램이 예기치 않게 종료되거나 이상 동작

// 원인: 스택 크기가 부족함
// 해결: 링커 스크립트에서 스택 크기 증가
_Min_Stack_Size = 0x10000;  // 64 KB

// 또는 런타임에 스택 사용량 확인
uint32_t stack_used = _estack - __get_MSP();
printf("Stack used: %lu bytes\n", stack_used);

// 스택 오버플로우 감지
void Check_Stack_Overflow(void)
{
  extern uint32_t _Min_Stack_Size;
  uint32_t stack_remaining = __get_MSP() - (uint32_t)&_Min_Stack_Size;

  if (stack_remaining < 1024)  // 1 KB 미만
  {
    printf("WARNING: Low stack space!\n");
  }
}
```

## 최적화 팁

### 1. 캐시 라인 정렬

```c
// 캐시 라인 크기: 32 바이트
// 32바이트 정렬하면 캐시 효율 향상

// 정렬 속성 사용
__attribute__((aligned(32))) uint32_t buffer[256];

// 또는 동적 할당
void* aligned_malloc(size_t size)
{
  void *ptr = malloc(size + 32);
  uint32_t addr = (uint32_t)ptr;
  uint32_t aligned_addr = (addr + 31) & ~31;
  return (void *)aligned_addr;
}
```

### 2. 버스트 액세스 활용

```c
// 순차 액세스 시 FMC가 자동으로 버스트 모드 사용
// 연속된 주소 액세스 시 성능 향상

// 좋은 예:
for (uint32_t i = 0; i < 1024; i++)
{
  buffer[i] = i;  // 순차 액세스 → 버스트 모드
}

// 나쁜 예:
for (uint32_t i = 0; i < 1024; i += 16)
{
  buffer[i] = i;  // 비순차 액세스 → 버스트 효과 감소
}
```

### 3. 데이터 구조 최적화

```c
// 자주 사용하는 데이터는 내부 RAM에
// 대용량 데이터는 SDRAM에

// 내부 RAM: 자주 액세스하는 작은 변수
uint32_t counter;
uint32_t flags;

// SDRAM: 대용량 버퍼
__attribute__((section(".sdram")))
uint32_t large_buffer[1024 * 1024];

// 또는 동적 할당
uint32_t *small_data = malloc(1024);  // 내부 RAM (빠름)
uint32_t *large_data = (uint32_t *)0xD0000000;  // SDRAM (느리지만 큼)
```

## 참고 자료

### STM32H7 문서
- **RM0399**: STM32H745/755 Reference Manual
  - 16장: FMC (Flexible Memory Controller)
  - SDRAM 컨트롤러, 타이밍 파라미터
- **AN5188**: STM32H7 FMC/QSPI usage
- **AN4936**: Migrating from STM32F7 to STM32H7

### SDRAM 데이터시트
- **IS42S32800G**: ISSI SDRAM 데이터시트
  - 타이밍 파라미터, 전기적 특성
  - 명령 시퀀스, 모드 레지스터

### HAL 드라이버
- `stm32h7xx_hal_sdram.c/h`: SDRAM HAL 드라이버
- `stm32h7xx_ll_fmc.c/h`: FMC 저수준 드라이버
- `STM32H745I_Discovery_sdram.c/h`: BSP SDRAM 드라이버

### 관련 예제
- `STM32Cube_FW_H7/Projects/STM32H745I-DISCO/Examples/FMC/`
  - FMC_SDRAM_DataMemory
  - FMC_SDRAM_ReadWrite_DMA
