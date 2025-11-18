# QSPI 메모리 맵 듀얼 모드 - MX25LM51245G (512 Mbit)

## 개요

이 예제는 STM32H745I-DISCO 보드의 QSPI(Quad-SPI) Flash 메모리(MX25LM51245G, 64MB)를 메모리 맵 모드로 사용하는 방법을 보여줍니다. 메모리 맵 모드에서는 QSPI Flash가 CPU 주소 공간(0x90000000)에 매핑되어 일반 메모리처럼 직접 읽기가 가능하며, XIP(Execute In Place) 코드 실행도 지원합니다.

## 주요 기능

### 1. MX25LM51245G QSPI Flash 사양
- **용량**: 512 Mbit (64 MB)
- **전압**: 1.8V
- **인터페이스**: Octal SPI (최대 8 라인) 또는 Quad SPI (4 라인)
- **최대 클럭**: 133 MHz (STR), 66 MHz (DTR)
- **메모리 맵 주소**: 0x90000000 ~ 0x93FFFFFF

### 2. 메모리 맵 모드
- **직접 읽기**: 포인터로 Flash 데이터 직접 액세스
- **XIP**: Flash에서 코드 실행 가능
- **캐시 지원**: MPU 설정으로 캐시 활성화 가능
- **쓰기**: 명령 모드로 전환 필요

### 3. 듀얼 모드 (Dual-Flash)
- **2개 Flash 칩**: 병렬 연결로 16-bit 데이터 버스
- **용량 배가**: 64 MB × 2 = 128 MB
- **대역폭 2배**: 동시 읽기로 성능 향상
- **주소 범위**: 0x90000000 ~ 0x97FFFFFF (128 MB)

## 하드웨어 구성

### QUADSPI 신호 핀
```
┌──────────────────────────────────────────┐
│  QUADSPI Flash 인터페이스                │
├──────────────────────────────────────────┤
│  QSPI_CLK   (PB2)   - 클럭               │
│  QSPI_NCS   (PG6)   - 칩 선택            │
│                                          │
│  Flash 1 (낮은 바이트):                  │
│  QSPI_BK1_IO0 (PD11) - Data 0            │
│  QSPI_BK1_IO1 (PF9)  - Data 1            │
│  QSPI_BK1_IO2 (PF7)  - Data 2            │
│  QSPI_BK1_IO3 (PF6)  - Data 3            │
│                                          │
│  Flash 2 (높은 바이트):                  │
│  QSPI_BK2_IO0 (PH2)  - Data 4            │
│  QSPI_BK2_IO1 (PH3)  - Data 5            │
│  QSPI_BK2_IO2 (PG9)  - Data 6            │
│  QSPI_BK2_IO3 (PG14) - Data 7            │
│                                          │
│  듀얼 모드 대역폭:                       │
│  133 MHz × 8 bits = 1064 Mbps = 133 MB/s│
└──────────────────────────────────────────┘

메모리 맵:
  0x90000000: QSPI 메모리 맵 시작
  0x93FFFFFF: 단일 Flash 끝 (64 MB)
  0x97FFFFFF: 듀얼 Flash 끝 (128 MB)
```

## 코드 상세 분석

### 1. QSPI 초기화

```c
QSPI_HandleTypeDef QSPIHandle;

void QSPI_Init(void)
{
  // QSPI 핸들 설정
  QSPIHandle.Instance = QUADSPI;

  // QSPI 클럭: 200 MHz / (1+1) = 100 MHz
  QSPIHandle.Init.ClockPrescaler = 1;
  QSPIHandle.Init.FifoThreshold = 4;
  QSPIHandle.Init.SampleShifting = QSPI_SAMPLE_SHIFTING_HALFCYCLE;
  QSPIHandle.Init.FlashSize = 25;  // 2^(25+1) = 64 MB
  QSPIHandle.Init.ChipSelectHighTime = QSPI_CS_HIGH_TIME_1_CYCLE;
  QSPIHandle.Init.ClockMode = QSPI_CLOCK_MODE_0;
  QSPIHandle.Init.FlashID = QSPI_FLASH_ID_1;  // 단일 Flash
  QSPIHandle.Init.DualFlash = QSPI_DUALFLASH_ENABLE;  // 듀얼 모드

  if (HAL_QSPI_Init(&QSPIHandle) != HAL_OK)
  {
    Error_Handler();
  }

  // Flash 초기화 시퀀스
  QSPI_ResetMemory();
  QSPI_DummyCyclesCfg();
  QSPI_WriteEnable();
}

// Flash 리셋
void QSPI_ResetMemory(void)
{
  QSPI_CommandTypeDef sCommand;

  // Reset Enable 명령
  sCommand.InstructionMode = QSPI_INSTRUCTION_1_LINE;
  sCommand.Instruction = 0x66;  // RSTEN
  sCommand.AddressMode = QSPI_ADDRESS_NONE;
  sCommand.AlternateByteMode = QSPI_ALTERNATE_BYTES_NONE;
  sCommand.DataMode = QSPI_DATA_NONE;
  sCommand.DummyCycles = 0;
  sCommand.DdrMode = QSPI_DDR_MODE_DISABLE;
  sCommand.DdrHoldHalfCycle = QSPI_DDR_HHC_ANALOG_DELAY;
  sCommand.SIOOMode = QSPI_SIOO_INST_EVERY_CMD;

  if (HAL_QSPI_Command(&QSPIHandle, &sCommand, HAL_QPSI_TIMEOUT_DEFAULT_VALUE) != HAL_OK)
  {
    Error_Handler();
  }

  // Reset 명령
  sCommand.Instruction = 0x99;  // RST
  if (HAL_QSPI_Command(&QSPIHandle, &sCommand, HAL_QPSI_TIMEOUT_DEFAULT_VALUE) != HAL_OK)
  {
    Error_Handler();
  }

  HAL_Delay(1);  // tRST = 30 μs
}
```

### 2. 메모리 맵 모드 활성화

```c
void QSPI_EnableMemoryMappedMode(void)
{
  QSPI_CommandTypeDef sCommand;
  QSPI_MemoryMappedTypeDef sMemMappedCfg;

  // Quad Output Fast Read 명령 (0x6B)
  sCommand.InstructionMode = QSPI_INSTRUCTION_1_LINE;
  sCommand.Instruction = 0x6B;  // Fast Read Quad Output
  sCommand.AddressMode = QSPI_ADDRESS_1_LINE;
  sCommand.AddressSize = QSPI_ADDRESS_24_BITS;
  sCommand.Address = 0;  // 메모리 맵 모드에서는 자동 설정
  sCommand.AlternateByteMode = QSPI_ALTERNATE_BYTES_NONE;
  sCommand.DataMode = QSPI_DATA_4_LINES;  // Quad (4-bit) 데이터
  sCommand.DummyCycles = 8;  // Dummy cycles (Flash 사양)
  sCommand.DdrMode = QSPI_DDR_MODE_DISABLE;
  sCommand.DdrHoldHalfCycle = QSPI_DDR_HHC_ANALOG_DELAY;
  sCommand.SIOOMode = QSPI_SIOO_INST_EVERY_CMD;

  // 메모리 맵 설정
  sMemMappedCfg.TimeOutActivation = QSPI_TIMEOUT_COUNTER_DISABLE;
  sMemMappedCfg.TimeOutPeriod = 0;

  // 메모리 맵 모드 활성화
  if (HAL_QSPI_MemoryMapped(&QSPIHandle, &sCommand, &sMemMappedCfg) != HAL_OK)
  {
    Error_Handler();
  }

  printf("Memory-mapped mode enabled at 0x90000000\n");
}
```

### 3. 메모리 맵 읽기

```c
#define QSPI_BASE_ADDRESS  0x90000000

// 포인터로 직접 읽기
void QSPI_ReadMemoryMapped(uint32_t address, uint8_t *pData, uint32_t size)
{
  uint8_t *qspi_ptr = (uint8_t *)(QSPI_BASE_ADDRESS + address);

  // 메모리 맵 모드에서는 일반 메모리처럼 읽기
  memcpy(pData, qspi_ptr, size);

  // 또는 직접 액세스
  for (uint32_t i = 0; i < size; i++)
  {
    pData[i] = qspi_ptr[i];
  }
}

// 32비트 워드 읽기
uint32_t QSPI_ReadWord(uint32_t address)
{
  uint32_t *qspi_ptr = (uint32_t *)(QSPI_BASE_ADDRESS + address);
  return *qspi_ptr;
}

// 구조체 읽기
typedef struct {
  uint32_t magic;
  uint32_t version;
  uint8_t  data[256];
} FlashData_t;

void QSPI_ReadStruct(void)
{
  FlashData_t *flash_data = (FlashData_t *)QSPI_BASE_ADDRESS;

  printf("Magic: 0x%08lX\n", flash_data->magic);
  printf("Version: %lu\n", flash_data->version);

  for (int i = 0; i < 256; i++)
  {
    printf("%02X ", flash_data->data[i]);
  }
}
```

### 4. 명령 모드 쓰기 (메모리 맵 종료 필요)

```c
// 메모리 맵 모드 종료
void QSPI_DisableMemoryMappedMode(void)
{
  // HAL_QSPI_Abort() 호출하여 메모리 맵 모드 종료
  if (HAL_QSPI_Abort(&QSPIHandle) != HAL_OK)
  {
    Error_Handler();
  }

  printf("Memory-mapped mode disabled\n");
}

// 섹터 지우기
void QSPI_EraseSector(uint32_t address)
{
  QSPI_CommandTypeDef sCommand;

  // 메모리 맵 모드 종료
  QSPI_DisableMemoryMappedMode();

  // Write Enable
  QSPI_WriteEnable();

  // Sector Erase 명령 (4 KB)
  sCommand.InstructionMode = QSPI_INSTRUCTION_1_LINE;
  sCommand.Instruction = 0x20;  // Sector Erase (4 KB)
  sCommand.AddressMode = QSPI_ADDRESS_1_LINE;
  sCommand.AddressSize = QSPI_ADDRESS_24_BITS;
  sCommand.Address = address;
  sCommand.AlternateByteMode = QSPI_ALTERNATE_BYTES_NONE;
  sCommand.DataMode = QSPI_DATA_NONE;
  sCommand.DummyCycles = 0;
  sCommand.DdrMode = QSPI_DDR_MODE_DISABLE;
  sCommand.DdrHoldHalfCycle = QSPI_DDR_HHC_ANALOG_DELAY;
  sCommand.SIOOMode = QSPI_SIOO_INST_EVERY_CMD;

  if (HAL_QSPI_Command(&QSPIHandle, &sCommand, HAL_QPSI_TIMEOUT_DEFAULT_VALUE) != HAL_OK)
  {
    Error_Handler();
  }

  // 지우기 완료 대기
  QSPI_AutoPollingMemReady(HAL_QPSI_TIMEOUT_DEFAULT_VALUE);

  // 메모리 맵 모드 재활성화
  QSPI_EnableMemoryMappedMode();
}

// 페이지 프로그램 (최대 256 바이트)
void QSPI_WritePage(uint32_t address, uint8_t *pData, uint32_t size)
{
  QSPI_CommandTypeDef sCommand;

  // 메모리 맵 모드 종료
  QSPI_DisableMemoryMappedMode();

  // Write Enable
  QSPI_WriteEnable();

  // Quad Page Program 명령
  sCommand.InstructionMode = QSPI_INSTRUCTION_1_LINE;
  sCommand.Instruction = 0x32;  // Quad Page Program
  sCommand.AddressMode = QSPI_ADDRESS_1_LINE;
  sCommand.AddressSize = QSPI_ADDRESS_24_BITS;
  sCommand.Address = address;
  sCommand.AlternateByteMode = QSPI_ALTERNATE_BYTES_NONE;
  sCommand.DataMode = QSPI_DATA_4_LINES;
  sCommand.NbData = size;
  sCommand.DummyCycles = 0;
  sCommand.DdrMode = QSPI_DDR_MODE_DISABLE;
  sCommand.DdrHoldHalfCycle = QSPI_DDR_HHC_ANALOG_DELAY;
  sCommand.SIOOMode = QSPI_SIOO_INST_EVERY_CMD;

  if (HAL_QSPI_Command(&QSPIHandle, &sCommand, HAL_QPSI_TIMEOUT_DEFAULT_VALUE) != HAL_OK)
  {
    Error_Handler();
  }

  // 데이터 전송
  if (HAL_QSPI_Transmit(&QSPIHandle, pData, HAL_QPSI_TIMEOUT_DEFAULT_VALUE) != HAL_OK)
  {
    Error_Handler();
  }

  // 프로그램 완료 대기
  QSPI_AutoPollingMemReady(HAL_QPSI_TIMEOUT_DEFAULT_VALUE);

  // 메모리 맵 모드 재활성화
  QSPI_EnableMemoryMappedMode();
}

// 대용량 쓰기
void QSPI_WriteData(uint32_t address, uint8_t *pData, uint32_t size)
{
  uint32_t end_addr = address + size;
  uint32_t current_size;
  uint32_t current_addr = address;

  // 첫 페이지 (256바이트 경계까지)
  current_size = 256 - (current_addr % 256);
  if (current_size > size)
    current_size = size;

  QSPI_WritePage(current_addr, pData, current_size);

  current_addr += current_size;
  pData += current_size;

  // 나머지 페이지 (256바이트씩)
  while (current_addr < end_addr)
  {
    current_size = ((end_addr - current_addr) >= 256) ? 256 : (end_addr - current_addr);
    QSPI_WritePage(current_addr, pData, current_size);

    current_addr += current_size;
    pData += current_size;
  }
}
```

### 5. XIP (Execute In Place) - Flash에서 코드 실행

```c
// 함수를 QSPI Flash에 배치
__attribute__((section(".qspi_code"))) void FlashFunction(void)
{
  printf("This function runs from QSPI Flash!\n");
  BSP_LED_Toggle(LED_GREEN);
}

// 링커 스크립트 (.ld 파일)
/*
MEMORY
{
  QSPI (rx)  : ORIGIN = 0x90000000, LENGTH = 64M
}

SECTIONS
{
  .qspi_code :
  {
    *(.qspi_code)
  } >QSPI
}
*/

// 메인 코드
int main(void)
{
  // ... 초기화 ...

  // QSPI 메모리 맵 모드 활성화
  QSPI_EnableMemoryMappedMode();

  // MPU 설정 (QSPI 영역을 실행 가능하게)
  MPU_Config_QSPI_XIP();

  // Flash에 저장된 함수 호출
  FlashFunction();  // 0x90000000 주소에서 실행됨

  while (1);
}

// MPU 설정 (XIP 지원)
void MPU_Config_QSPI_XIP(void)
{
  MPU_Region_InitTypeDef MPU_InitStruct;

  HAL_MPU_Disable();

  // QSPI 영역: 캐시 가능, 실행 가능
  MPU_InitStruct.Enable = MPU_REGION_ENABLE;
  MPU_InitStruct.BaseAddress = 0x90000000;
  MPU_InitStruct.Size = MPU_REGION_SIZE_64MB;
  MPU_InitStruct.AccessPermission = MPU_REGION_FULL_ACCESS;
  MPU_InitStruct.IsBufferable = MPU_ACCESS_BUFFERABLE;
  MPU_InitStruct.IsCacheable = MPU_ACCESS_CACHEABLE;
  MPU_InitStruct.IsShareable = MPU_ACCESS_NOT_SHAREABLE;
  MPU_InitStruct.Number = MPU_REGION_NUMBER2;
  MPU_InitStruct.TypeExtField = MPU_TEX_LEVEL0;
  MPU_InitStruct.SubRegionDisable = 0x00;
  MPU_InitStruct.DisableExec = MPU_INSTRUCTION_ACCESS_ENABLE;  // 실행 가능

  HAL_MPU_ConfigRegion(&MPU_InitStruct);

  HAL_MPU_Enable(MPU_PRIVILEGED_DEFAULT);
}
```

## 성능 분석

### 읽기 속도
```
명령 모드 (HAL_QSPI_Receive):
  속도: 20-30 MB/s
  오버헤드: 명령 전송, 폴링

메모리 맵 모드 (캐시 없음):
  속도: 40-50 MB/s
  직접 읽기, 명령 오버헤드 없음

메모리 맵 모드 (캐시 있음):
  캐시 히트: 400 MB/s (CPU 속도)
  캐시 미스: 40-50 MB/s
  순차 읽기: 80-100 MB/s (캐시 라인 프리페치)

듀얼 Flash 모드:
  속도: 80-100 MB/s (2배 향상)
  대역폭: 두 Flash 동시 읽기
```

## 고급 사용 예제

### 1. 대용량 데이터 저장 (이미지, 폰트)

```c
// QSPI Flash에 이미지 저장
const uint8_t image_480x272_rgb565[] __attribute__((section(".qspi_data"))) = {
  // 이미지 데이터 (261,120 바이트)
};

// 메모리 맵 모드에서 LCD에 직접 표시
void Display_Image_From_QSPI(void)
{
  uint16_t *lcd_fb = (uint16_t *)LCD_FRAME_BUFFER;
  uint16_t *qspi_image = (uint16_t *)0x90000000;

  // DMA2D로 고속 복사
  DMA2D_CopyBuffer(qspi_image, lcd_fb, 480, 272);
}
```

### 2. 파일 시스템 (LittleFS)

```c
#include "lfs.h"

// LittleFS 구성
const struct lfs_config cfg = {
  .read  = qspi_read,
  .prog  = qspi_prog,
  .erase = qspi_erase,
  .sync  = qspi_sync,

  .read_size = 256,
  .prog_size = 256,
  .block_size = 4096,
  .block_count = 16384,  // 64 MB / 4 KB
  .cache_size = 256,
  .lookahead_size = 16,
  .block_cycles = 500,
};

lfs_t lfs;
lfs_file_t file;

void QSPI_LittleFS_Example(void)
{
  // 마운트
  int err = lfs_mount(&lfs, &cfg);
  if (err)
  {
    lfs_format(&lfs, &cfg);
    lfs_mount(&lfs, &cfg);
  }

  // 파일 쓰기
  lfs_file_open(&lfs, &file, "hello.txt", LFS_O_WRONLY | LFS_O_CREAT);
  lfs_file_write(&lfs, &file, "Hello STM32H7!", 14);
  lfs_file_close(&lfs, &file);

  // 파일 읽기
  char buffer[32];
  lfs_file_open(&lfs, &file, "hello.txt", LFS_O_RDONLY);
  lfs_file_read(&lfs, &file, buffer, 32);
  lfs_file_close(&lfs, &file);

  printf("Read: %s\n", buffer);

  // 언마운트
  lfs_unmount(&lfs);
}
```

## 트러블슈팅

### 1. 메모리 맵 모드에서 읽기 오류
- **Dummy cycles 확인**: Flash 사양에 맞게 설정
- **MPU 설정**: QSPI 영역을 액세스 가능하게 설정

### 2. XIP 실행 실패
- **MPU DisableExec**: MPU_INSTRUCTION_ACCESS_ENABLE로 설정
- **캐시 활성화**: 성능 향상을 위해 캐시 필요

### 3. 쓰기 후 읽기 오류
- **Write Enable**: 쓰기 전 반드시 WREN 명령
- **Busy 대기**: 쓰기 완료까지 대기
- **메모리 맵 재활성화**: 쓰기 후 메모리 맵 모드 재시작

## 참고 자료
- **RM0399**: STM32H7 Reference Manual (QUADSPI)
- **MX25LM51245G**: Macronix Flash 데이터시트
- HAL 드라이버: `stm32h7xx_hal_qspi.c/h`
