# eMMC 읽기/쓰기 (DMA 모드)

## 개요

이 예제는 STM32H745I-DISCO 보드의 eMMC(embedded Multi-Media Card) 스토리지에 DMA(Direct Memory Access)를 사용하여 대용량 데이터를 고속으로 읽고 쓰는 방법을 보여줍니다. DMA를 사용하면 CPU 개입 없이 메모리와 eMMC 간 데이터 전송이 가능하여 최대 성능을 달성할 수 있습니다.

## 주요 기능

### 1. eMMC 사양 (STM32H745I-DISCO)
- **용량**: 가변 (일반적으로 4GB ~ 16GB)
- **인터페이스**: SDMMC1 (8-bit 버스)
- **최대 속도**: 50 MHz (HS200 모드에서 200 MHz 가능)
- **블록 크기**: 512 바이트
- **전송 모드**: DMA, 폴링, 인터럽트

### 2. DMA 전송
- **DMA 컨트롤러**: SDMMC1 전용 DMA
- **버퍼 크기**: 256 KB (512 블록)
- **전체 테스트 크기**: 100 MB
- **전송 속도**: 최대 40 MB/s (쓰기), 50 MB/s (읽기)

### 3. 버퍼 배치
- **TX 버퍼**: 0x24000000 (AXI-SRAM)
- **RX 버퍼**: 0x24040000 (AXI-SRAM)
- **D-Cache 관리**: 필수 (일관성 유지)

## 하드웨어 구성

### SDMMC1 신호 핀
```
┌────────────────────────────────────┐
│  SDMMC1 eMMC 인터페이스            │
├────────────────────────────────────┤
│  SDMMC1_CK   (PC12) - 클럭         │
│  SDMMC1_CMD  (PD2)  - 명령         │
│                                    │
│  데이터 라인 (8-bit):              │
│  SDMMC1_D0   (PC8)                 │
│  SDMMC1_D1   (PC9)                 │
│  SDMMC1_D2   (PC10)                │
│  SDMMC1_D3   (PC11)                │
│  SDMMC1_D4   (PB8)                 │
│  SDMMC1_D5   (PB9)                 │
│  SDMMC1_D6   (PC6)                 │
│  SDMMC1_D7   (PC7)                 │
│                                    │
│  전송 속도: 최대 50 MHz × 8 bits   │
│           = 400 Mbps = 50 MB/s     │
└────────────────────────────────────┘
```

## 코드 상세 분석

### 1. eMMC 초기화

```c
MMC_HandleTypeDef MMCHandle;

// eMMC 초기화
MMCHandle.Instance = SDMMC1;

// SDMMC 클럭: 200 MHz / (2 * 2) = 50 MHz
MMCHandle.Init.ClockEdge = SDMMC_CLOCK_EDGE_RISING;
MMCHandle.Init.ClockPowerSave = SDMMC_CLOCK_POWER_SAVE_DISABLE;
MMCHandle.Init.BusWide = SDMMC_BUS_WIDE_8B;  // 8-bit 버스
MMCHandle.Init.HardwareFlowControl = SDMMC_HARDWARE_FLOW_CONTROL_ENABLE;
MMCHandle.Init.ClockDiv = 2;  // 분주비

if (HAL_MMC_Init(&MMCHandle) != HAL_OK)
{
  Error_Handler();
}

// 8-bit 버스 모드 설정
if (HAL_MMC_ConfigWideBusOperation(&MMCHandle, SDMMC_BUS_WIDE_8B) != HAL_OK)
{
  Error_Handler();
}

// 카드 정보 읽기
HAL_MMC_CardCIDTypeDef pCID;
HAL_MMC_CardCSDTypeDef pCSD;
HAL_MMC_GetCardCID(&MMCHandle, &pCID);
HAL_MMC_GetCardCSD(&MMCHandle, &pCSD);

printf("MMC Manufacturer ID: 0x%02X\n", pCID.ManufacturerID);
printf("MMC Capacity: %lu MB\n", (pCSD.DeviceSize * pCSD.DeviceSizeMul) >> 10);
```

### 2. DMA 쓰기 (100 MB)

```c
#define BUFFER_SIZE    0x00040000U  // 256 KB
#define DATA_SIZE      0x06400000U  // 100 MB
#define NB_BUFFER      (DATA_SIZE / BUFFER_SIZE)  // 400 버퍼
#define NB_BLOCK_BUFFER (BUFFER_SIZE / 512)       // 512 블록

// AXI-SRAM에 버퍼 배치
__attribute__((section(".RAM_D1")))
uint8_t aTxBuffer[BUFFER_SIZE];

void MMC_Write_Test(void)
{
  uint32_t index = 0;
  uint32_t start_time, stop_time;

  // 전송 버퍼 초기화
  for (uint32_t i = 0; i < BUFFER_SIZE; i++)
  {
    aTxBuffer[i] = 0xB5 + i;
  }

  // D-Cache 클린 (DMA가 최신 데이터를 읽도록)
  SCB_CleanDCache_by_Addr((uint32_t*)aTxBuffer, BUFFER_SIZE);

  printf("Write test: 100 MB\n");
  start_time = HAL_GetTick();

  // 400개 버퍼 전송 (각 256 KB)
  for (index = 0; index < NB_BUFFER; index++)
  {
    // eMMC 준비 대기
    if (Wait_MMC_Ready() != HAL_OK)
    {
      Error_Handler();
    }

    // DMA 쓰기 시작 (512 블록 = 256 KB)
    TxCplt = 0;
    if (HAL_MMC_WriteBlocks_DMA(&MMCHandle, aTxBuffer,
                                 ADDRESS + index * NB_BLOCK_BUFFER,
                                 NB_BLOCK_BUFFER) != HAL_OK)
    {
      Error_Handler();
    }

    // 전송 완료 대기
    while (TxCplt == 0)
    {
      // DMA 전송 중 다른 작업 가능
    }
  }

  stop_time = HAL_GetTick();
  float speed = (float)(DATA_SIZE >> 10) / (float)(stop_time - start_time);
  printf("Write: %lu ms, Speed: %.2f MB/s\n", stop_time - start_time, speed);
}

// DMA 쓰기 완료 콜백
void HAL_MMC_TxCpltCallback(MMC_HandleTypeDef *hmmc)
{
  TxCplt = 1;
}
```

### 3. DMA 읽기 (100 MB)

```c
__attribute__((section(".RAM_D1")))
uint8_t aRxBuffer[BUFFER_SIZE];

void MMC_Read_Test(void)
{
  uint32_t index = 0;
  uint32_t start_time, stop_time;

  // 수신 버퍼 초기화
  memset(aRxBuffer, 0, BUFFER_SIZE);
  SCB_CleanDCache_by_Addr((uint32_t*)aRxBuffer, BUFFER_SIZE);

  printf("Read test: 100 MB\n");
  start_time = HAL_GetTick();

  for (index = 0; index < NB_BUFFER; index++)
  {
    if (Wait_MMC_Ready() != HAL_OK)
    {
      Error_Handler();
    }

    // DMA 읽기 시작
    RxCplt = 0;
    if (HAL_MMC_ReadBlocks_DMA(&MMCHandle, aRxBuffer,
                                ADDRESS + index * NB_BLOCK_BUFFER,
                                NB_BLOCK_BUFFER) != HAL_OK)
    {
      Error_Handler();
    }

    // 전송 완료 대기
    while (RxCplt == 0);

    // D-Cache 무효화 (DMA가 쓴 최신 데이터 읽기)
    SCB_InvalidateDCache_by_Addr((uint32_t*)aRxBuffer, BUFFER_SIZE);

    // 데이터 검증 (옵션)
    for (uint32_t i = 0; i < BUFFER_SIZE; i++)
    {
      if (aRxBuffer[i] != (uint8_t)(0xB5 + i))
      {
        printf("Verification failed at offset %lu\n", i);
        Error_Handler();
      }
    }
  }

  stop_time = HAL_GetTick();
  float speed = (float)(DATA_SIZE >> 10) / (float)(stop_time - start_time);
  printf("Read: %lu ms, Speed: %.2f MB/s\n", stop_time - start_time, speed);
}

// DMA 읽기 완료 콜백
void HAL_MMC_RxCpltCallback(MMC_HandleTypeDef *hmmc)
{
  RxCplt = 1;
}
```

## 성능 분석

### 전송 속도
```
이론적 최대 속도:
  50 MHz × 8 bits = 400 Mbps = 50 MB/s

실제 측정 속도:
  쓰기: 35-40 MB/s (이론치의 80%)
  읽기: 45-50 MB/s (이론치의 95%)

DMA 오버헤드:
  버퍼 전환 시간: 약 50 μs
  전체 영향: < 1%
```

## 고급 사용 예제

### 1. 파일 시스템 (FatFs)

```c
#include "ff.h"

FATFS MMCFatFs;
FIL MyFile;

void MMC_FatFs_Test(void)
{
  FRESULT res;
  uint32_t byteswritten, bytesread;
  uint8_t wtext[] = "Hello STM32H7!";
  uint8_t rtext[100];

  // FatFs 등록
  res = f_mount(&MMCFatFs, "0:", 1);
  if (res != FR_OK)
  {
    printf("Mount failed: %d\n", res);
    return;
  }

  // 파일 생성 및 쓰기
  res = f_open(&MyFile, "0:test.txt", FA_CREATE_ALWAYS | FA_WRITE);
  if (res == FR_OK)
  {
    res = f_write(&MyFile, wtext, sizeof(wtext), &byteswritten);
    if (res == FR_OK)
    {
      printf("Written: %lu bytes\n", byteswritten);
    }
    f_close(&MyFile);
  }

  // 파일 읽기
  res = f_open(&MyFile, "0:test.txt", FA_READ);
  if (res == FR_OK)
  {
    res = f_read(&MyFile, rtext, sizeof(rtext), &bytesread);
    if (res == FR_OK)
    {
      printf("Read: %s\n", rtext);
    }
    f_close(&MyFile);
  }

  // 언마운트
  f_mount(NULL, "0:", 1);
}
```

### 2. 블록 단위 액세스

```c
// 단일 블록 쓰기
uint8_t block_data[512];
if (HAL_MMC_WriteBlocks_DMA(&MMCHandle, block_data, 0, 1) == HAL_OK)
{
  while (TxCplt == 0);
  printf("Single block written\n");
}

// 다중 블록 쓰기 (10 블록 = 5 KB)
if (HAL_MMC_WriteBlocks_DMA(&MMCHandle, buffer, 100, 10) == HAL_OK)
{
  while (TxCplt == 0);
  printf("10 blocks written\n");
}
```

## 트러블슈팅

### 1. DMA 전송 실패
- **D-Cache 클린/무효화 확인**
- **버퍼 주소 4바이트 정렬**
- **SDMMC 클럭 설정 확인**

### 2. 데이터 손상
- **캐시 일관성 문제**: SCB_CleanDCache / SCB_InvalidateDCache 사용
- **전송 중 버퍼 수정 금지**

### 3. 느린 속도
- **8-bit 버스 모드 확인**: SDMMC_BUS_WIDE_8B
- **하드웨어 흐름 제어 활성화**: SDMMC_HARDWARE_FLOW_CONTROL_ENABLE
- **클럭 분주비 최적화**: ClockDiv = 2 (50 MHz)

## 참고 자료
- **RM0399**: STM32H7 Reference Manual (SDMMC)
- **UM2411**: STM32H745I-DISCO User Manual
- HAL 드라이버: `stm32h7xx_hal_mmc.c/h`
