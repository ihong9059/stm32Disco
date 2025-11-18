# FatFs_Shared_Device - 듀얼 코어 FatFs 공유 접근

## 개요

이 애플리케이션은 STM32H745의 Cortex-M7과 Cortex-M4에서 동시에 FatFs 파일 시스템에 접근하는 방법을 보여줍니다. HSEM (Hardware Semaphore)을 사용하여 eMMC 또는 SD 카드의 동시 접근을 안전하게 동기화합니다.

## 공유 저장 장치 접근의 필요성

듀얼 코어 시스템에서 파일 시스템 공유는 다음과 같은 경우에 필요합니다:

- **데이터 로깅**: CM4가 센서 데이터를 기록하고 CM7이 읽음
- **설정 파일**: 양쪽 코어가 공통 설정 파일 접근
- **펌웨어 업데이트**: CM7이 다운로드하고 CM4가 처리

## 시스템 아키텍처

### 공유 FatFs 구조
```
┌──────────────────┐              ┌──────────────────┐
│   Cortex-M7      │              │   Cortex-M4      │
│                  │              │                  │
│  ┌────────────┐  │              │  ┌────────────┐  │
│  │ Application │  │              │  │ Application │  │
│  └──────┬─────┘  │              │  └──────┬─────┘  │
│         │        │              │         │        │
│  ┌──────▼─────┐  │              │  ┌──────▼─────┐  │
│  │   FatFs    │  │              │  │   FatFs    │  │
│  └──────┬─────┘  │              │  └──────┬─────┘  │
│         │        │              │         │        │
│  ┌──────▼─────┐  │              │  ┌──────▼─────┐  │
│  │ Disk I/O   │  │              │  │ Disk I/O   │  │
│  │ (HSEM Lock)│  │              │  │ (HSEM Lock)│  │
│  └──────┬─────┘  │              │  └──────┬─────┘  │
│         │        │              │         │        │
└─────────┼────────┘              └─────────┼────────┘
          │                                 │
          │         ┌─────────┐             │
          └────────►│  HSEM   │◄────────────┘
                    │ (동기화)│
                    └────┬────┘
                         │
                    ┌────▼────┐
                    │  eMMC / │
                    │ SD Card │
                    └─────────┘
```

## 메모리 맵

### 공유 데이터 구조
```c
// D3 SRAM에 공유 데이터 배치
#define SHARED_BUFFER_ADDRESS  0x38000000

// 공유 구조체
typedef struct {
  uint8_t write_buffer[512];    // 쓰기 버퍼
  uint8_t read_buffer[512];     // 읽기 버퍼
  volatile uint32_t status;     // 상태 플래그
  volatile uint32_t file_size;  // 파일 크기
} SharedData_TypeDef;

__attribute__((section(".shared_data")))
SharedData_TypeDef shared_data;
```

## HSEM 기반 동기화

### HSEM ID 할당
```c
// 하드웨어 세마포어 ID
#define HSEM_ID_DISK_IO    0   // Disk I/O 접근 제어
#define HSEM_ID_FAT_TABLE  1   // FAT 테이블 접근
#define HSEM_ID_DIRECTORY  2   // 디렉토리 접근
```

### 잠금/해제 함수
```c
// Disk I/O 잠금
void FatFs_Lock(void)
{
  // HSEM 획득 시도 (블로킹)
  while (HAL_HSEM_FastTake(HSEM_ID_DISK_IO) != HAL_OK)
  {
    // 다른 코어가 사용 중이면 대기
    __NOP();
  }
}

// Disk I/O 해제
void FatFs_Unlock(void)
{
  HAL_HSEM_Release(HSEM_ID_DISK_IO, 0);
}
```

## 코드 구현

### 1. CM7 메인 (Master)
```c
#include "ff.h"
#include "ff_gen_drv.h"

FATFS SDFatFs;
char SDPath[4];

int main(void)
{
  // 시스템 초기화
  HAL_Init();
  SystemClock_Config();

  // LED 초기화
  BSP_LED_Init(LED1);
  BSP_LED_Init(LED2);

  // HSEM 초기화
  __HAL_RCC_HSEM_CLK_ENABLE();

  // FatFs 드라이버 링크
  FATFS_LinkDriver(&SD_Driver, SDPath);

  // 파일 시스템 마운트
  if (f_mount(&SDFatFs, SDPath, 0) != FR_OK)
  {
    Error_Handler();
  }

  // CM4 부팅
  HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);

  // CM4 준비 대기
  while (shared_data.status != CM4_READY);

  while (1)
  {
    // CM7: 파일 쓰기 작업
    CM7_WriteFile();

    BSP_LED_Toggle(LED1);
    HAL_Delay(1000);
  }
}
```

### 2. CM7 파일 쓰기 (동기화 적용)
```c
FRESULT CM7_WriteFile(void)
{
  FIL file;
  FRESULT res;
  UINT bytes_written;
  char text[64];
  static uint32_t counter = 0;

  // 타임스탬프가 있는 메시지 생성
  sprintf(text, "[CM7] Counter: %lu\r\n", counter++);

  // HSEM 잠금 획득
  FatFs_Lock();

  // 파일 열기 (추가 모드)
  res = f_open(&file, "0:/shared_log.txt", FA_OPEN_APPEND | FA_WRITE);
  if (res == FR_OK)
  {
    // 데이터 쓰기
    res = f_write(&file, text, strlen(text), &bytes_written);
    f_close(&file);

    if (res == FR_OK)
    {
      printf("CM7: Written %u bytes\n", bytes_written);
    }
  }

  // HSEM 잠금 해제
  FatFs_Unlock();

  return res;
}
```

### 3. CM4 메인 (Remote)
```c
int main(void)
{
  // HSEM 대기 (CM7이 초기화 완료할 때까지)
  __HAL_RCC_HSEM_CLK_ENABLE();

  // CM7이 먼저 마운트할 때까지 대기
  HAL_Delay(100);

  // HAL 초기화
  HAL_Init();

  // LED 초기화
  BSP_LED_Init(LED3);
  BSP_LED_Init(LED4);

  // 준비 완료 신호
  shared_data.status = CM4_READY;

  while (1)
  {
    // CM4: 파일 읽기 작업
    CM4_ReadFile();

    BSP_LED_Toggle(LED3);
    HAL_Delay(2000);
  }
}
```

### 4. CM4 파일 읽기 (동기화 적용)
```c
FRESULT CM4_ReadFile(void)
{
  FIL file;
  FRESULT res;
  UINT bytes_read;
  char buffer[256];

  // HSEM 잠금 획득
  FatFs_Lock();

  // 파일 열기
  res = f_open(&file, "0:/shared_log.txt", FA_READ);
  if (res == FR_OK)
  {
    // 파일 끝에서 읽기 (최근 데이터)
    if (f_size(&file) > sizeof(buffer))
    {
      f_lseek(&file, f_size(&file) - sizeof(buffer));
    }

    // 데이터 읽기
    res = f_read(&file, buffer, sizeof(buffer) - 1, &bytes_read);
    if (res == FR_OK)
    {
      buffer[bytes_read] = '\0';
      printf("CM4: Read data:\n%s\n", buffer);
    }

    f_close(&file);
  }

  // HSEM 잠금 해제
  FatFs_Unlock();

  return res;
}
```

## 동기화된 Disk I/O 드라이버

### ff_sd_diskio.c 수정
```c
// 읽기 작업
DRESULT SD_read(BYTE pdrv, BYTE *buff, DWORD sector, UINT count)
{
  DRESULT res = RES_ERROR;

  // HSEM 잠금
  FatFs_Lock();

  // SD 카드 읽기
  if (BSP_SD_ReadBlocks((uint32_t *)buff, sector, count, SD_TIMEOUT) == HAL_OK)
  {
    // DMA 완료 대기
    while (BSP_SD_GetCardState() != SD_TRANSFER_OK);
    res = RES_OK;
  }

  // HSEM 해제
  FatFs_Unlock();

  return res;
}

// 쓰기 작업
DRESULT SD_write(BYTE pdrv, const BYTE *buff, DWORD sector, UINT count)
{
  DRESULT res = RES_ERROR;

  // HSEM 잠금
  FatFs_Lock();

  // SD 카드 쓰기
  if (BSP_SD_WriteBlocks((uint32_t *)buff, sector, count, SD_TIMEOUT) == HAL_OK)
  {
    // DMA 완료 대기
    while (BSP_SD_GetCardState() != SD_TRANSFER_OK);
    res = RES_OK;
  }

  // HSEM 해제
  FatFs_Unlock();

  return res;
}
```

## FatFs 재진입 설정

### ffconf.h
```c
// 재진입 (Thread-safe) 설정
#define FF_FS_REENTRANT    1
#define FF_FS_TIMEOUT      1000
#define FF_SYNC_t          osSemaphoreId_t

// 또는 HSEM 기반 동기화
#define FF_FS_REENTRANT    0  // FatFs 내부 동기화 비활성화
// 대신 Disk I/O 레벨에서 HSEM으로 동기화
```

### 동기화 함수 (FatFs 사용 시)
```c
// ff_lock.c
#if FF_FS_REENTRANT

int ff_cre_syncobj(BYTE vol, FF_SYNC_t *sobj)
{
  // HSEM 초기화
  *sobj = (FF_SYNC_t)(HSEM_ID_DISK_IO + vol);
  return 1;
}

int ff_req_grant(FF_SYNC_t sobj)
{
  // HSEM 획득
  while (HAL_HSEM_FastTake((uint32_t)sobj) != HAL_OK)
  {
    HAL_Delay(1);
  }
  return 1;
}

void ff_rel_grant(FF_SYNC_t sobj)
{
  // HSEM 해제
  HAL_HSEM_Release((uint32_t)sobj, 0);
}

int ff_del_syncobj(FF_SYNC_t sobj)
{
  return 1;
}

#endif
```

## 고급 기능

### 1. 파일 잠금 (File Locking)
```c
// 특정 파일에 대한 배타적 접근
#define HSEM_FILE_CONFIG   3
#define HSEM_FILE_LOG      4

void LockConfigFile(void)
{
  while (HAL_HSEM_FastTake(HSEM_FILE_CONFIG) != HAL_OK);
}

void UnlockConfigFile(void)
{
  HAL_HSEM_Release(HSEM_FILE_CONFIG, 0);
}
```

### 2. 비동기 알림
```c
// CM7: CM4에게 파일 업데이트 알림
void NotifyFileUpdated(void)
{
  shared_data.status = FILE_UPDATED;
  HAL_HSEM_ActivateNotification(__HAL_HSEM_SEMID_TO_MASK(HSEM_ID_NOTIFICATION));
}

// CM4: HSEM 인터럽트 핸들러
void HSEM1_IRQHandler(void)
{
  HAL_HSEM_IRQHandler();

  if (shared_data.status == FILE_UPDATED)
  {
    // 파일 읽기 요청 처리
    file_update_pending = 1;
  }
}
```

### 3. 버퍼링된 쓰기
```c
#define WRITE_BUFFER_SIZE  4096

typedef struct {
  uint8_t data[WRITE_BUFFER_SIZE];
  uint32_t count;
} WriteBuffer_TypeDef;

// CM4: 버퍼에 데이터 축적
void BufferedWrite(const char *data, uint32_t len)
{
  // 버퍼에 추가
  if (write_buffer.count + len < WRITE_BUFFER_SIZE)
  {
    memcpy(&write_buffer.data[write_buffer.count], data, len);
    write_buffer.count += len;
  }
  else
  {
    // 버퍼 가득 참 - 플러시
    FlushWriteBuffer();
    memcpy(write_buffer.data, data, len);
    write_buffer.count = len;
  }
}

// 버퍼를 파일에 플러시
void FlushWriteBuffer(void)
{
  if (write_buffer.count > 0)
  {
    FatFs_Lock();

    FIL file;
    UINT written;

    if (f_open(&file, "0:/log.txt", FA_OPEN_APPEND | FA_WRITE) == FR_OK)
    {
      f_write(&file, write_buffer.data, write_buffer.count, &written);
      f_close(&file);
    }

    FatFs_Unlock();

    write_buffer.count = 0;
  }
}
```

### 4. 우선순위 기반 접근
```c
// CM7 (고우선순위) - 즉시 획득 시도
FRESULT CM7_PriorityWrite(void)
{
  // 빠른 획득 시도
  if (HAL_HSEM_FastTake(HSEM_ID_DISK_IO) == HAL_OK)
  {
    // 즉시 쓰기
    // ...
    HAL_HSEM_Release(HSEM_ID_DISK_IO, 0);
  }
  else
  {
    // 나중에 재시도
    return FR_DENIED;
  }
  return FR_OK;
}

// CM4 (저우선순위) - 무한 대기
FRESULT CM4_BackgroundRead(void)
{
  // 대기하며 획득
  while (HAL_HSEM_FastTake(HSEM_ID_DISK_IO) != HAL_OK)
  {
    HAL_Delay(10);  // 다른 작업에 양보
  }

  // 읽기
  // ...

  HAL_HSEM_Release(HSEM_ID_DISK_IO, 0);
  return FR_OK;
}
```

## 응용 예제

### 1. 데이터 로깅 시스템
```c
// CM4: 센서 데이터 수집 및 저장
void SensorLogging_Task(void)
{
  SensorData_TypeDef data;

  while (1)
  {
    // 센서 데이터 읽기
    ReadSensors(&data);

    // 버퍼에 추가
    char line[64];
    sprintf(line, "%lu,%d,%d,%d\r\n",
            HAL_GetTick(), data.temp, data.humidity, data.pressure);
    BufferedWrite(line, strlen(line));

    HAL_Delay(100);  // 10 Hz 샘플링
  }
}

// CM7: 데이터 분석 및 표시
void DataAnalysis_Task(void)
{
  while (1)
  {
    // 로그 파일 읽기
    FatFs_Lock();

    // 최근 데이터 분석
    AnalyzeRecentData("0:/sensor_log.csv");

    FatFs_Unlock();

    // 결과 LCD에 표시
    DisplayAnalysisResult();

    HAL_Delay(5000);  // 5초마다 업데이트
  }
}
```

### 2. 설정 파일 공유
```c
// 공유 설정 구조체
typedef struct {
  uint32_t sample_rate;
  uint32_t threshold;
  uint8_t mode;
} Config_TypeDef;

// CM7: 설정 저장
void SaveConfig(Config_TypeDef *config)
{
  LockConfigFile();

  FIL file;
  if (f_open(&file, "0:/config.bin", FA_CREATE_ALWAYS | FA_WRITE) == FR_OK)
  {
    UINT written;
    f_write(&file, config, sizeof(Config_TypeDef), &written);
    f_close(&file);
  }

  UnlockConfigFile();

  // CM4에게 설정 업데이트 알림
  NotifyConfigUpdated();
}

// CM4: 설정 로드
void LoadConfig(Config_TypeDef *config)
{
  LockConfigFile();

  FIL file;
  if (f_open(&file, "0:/config.bin", FA_READ) == FR_OK)
  {
    UINT read;
    f_read(&file, config, sizeof(Config_TypeDef), &read);
    f_close(&file);
  }

  UnlockConfigFile();
}
```

## 트러블슈팅

### 데드락 발생

1. **잠금 순서 준수**: 항상 같은 순서로 HSEM 획득
2. **타임아웃 설정**: 무한 대기 방지
   ```c
   uint32_t timeout = 1000;
   while (HAL_HSEM_FastTake(HSEM_ID) != HAL_OK)
   {
     if (--timeout == 0)
     {
       // 타임아웃 처리
       return FR_TIMEOUT;
     }
     HAL_Delay(1);
   }
   ```

3. **중첩 잠금 방지**: 이미 획득한 세마포어를 다시 획득하지 않음

### 데이터 손상

1. **캐시 관리**: 공유 버퍼는 non-cacheable 또는 캐시 무효화
   ```c
   SCB_InvalidateDCache_by_Addr((uint32_t*)shared_data, sizeof(shared_data));
   ```

2. **완전한 쓰기 보장**: f_sync() 사용
   ```c
   f_write(&file, data, len, &written);
   f_sync(&file);  // 데이터를 미디어에 플러시
   ```

### 성능 문제

1. **잠금 범위 최소화**: 필요한 작업만 잠금 내에서 수행
2. **버퍼링 사용**: 작은 쓰기를 모아서 처리
3. **분리된 파일 사용**: 가능하면 각 코어가 다른 파일 사용

## 성능 고려사항

### 동기화 오버헤드
- **HSEM 획득**: 약 10-100 사이클
- **잠금 대기**: 다른 코어의 작업 시간 의존
- **최적화**: 잠금 범위 최소화

### 권장 사항
- 중요하지 않은 데이터는 별도 파일 사용
- 대용량 전송 시 DMA + 인터럽트 활용
- 주기적 작업은 타이밍 조절로 충돌 최소화

## 참고 자료

- FatFs 공식 사이트: http://elm-chan.org/fsw/ff/
- AN4838: STM32 hardware semaphores (HSEM)
- STM32H7 참조 매뉴얼: RM0399
- 예제: `STM32H745I-DISCO/Applications/FatFs/FatFs_Shared_Device`
