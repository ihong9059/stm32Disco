# USB_Host MSC_Standalone - USB 대용량 저장 장치 호스트

## 개요

이 애플리케이션은 STM32H745I-DISCO 보드를 USB MSC (Mass Storage Class) 호스트로 구현하여 USB 플래시 드라이브, 외장 하드 디스크 등을 연결하고 파일 작업을 수행하는 예제입니다. FatFs 파일 시스템을 통해 파일 읽기/쓰기를 지원합니다.

## USB MSC 호스트란?

**USB MSC (Mass Storage Class) 호스트**는 USB 대용량 저장 장치를 연결하여 파일 시스템에 접근할 수 있는 호스트 시스템입니다.

### 주요 특징
- **표준 프로토콜**: Bulk-Only Transport (BOT)
- **SCSI 명령**: READ, WRITE, INQUIRY 등
- **파일 시스템**: FAT16, FAT32, exFAT

## 시스템 아키텍처

### USB MSC 호스트 구조
```
┌──────────────────────────────────────────────────────────────┐
│                      STM32H745I-DISCO                        │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                    Application                       │    │
│  │       (파일 읽기/쓰기, 디렉토리 탐색)                │    │
│  └────────────────────┬─────────────────────────────────┘    │
│                       │                                      │
│  ┌────────────────────▼─────────────────────────────────┐    │
│  │                      FatFs                           │    │
│  │              (파일 시스템 레이어)                    │    │
│  └────────────────────┬─────────────────────────────────┘    │
│                       │                                      │
│  ┌────────────────────▼─────────────────────────────────┐    │
│  │                   USBH MSC                           │    │
│  │              (Mass Storage Class)                    │    │
│  │                                                      │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │    │
│  │  │  BOT Proto  │  │ SCSI Cmd   │  │  LUN Mgmt   │   │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘   │    │
│  │                                                      │    │
│  └────────────────────┬─────────────────────────────────┘    │
│                       │                                      │
│  ┌────────────────────▼─────────────────────────────────┐    │
│  │                 USB Host Core                        │    │
│  └────────────────────┬─────────────────────────────────┘    │
│                       │                                      │
│  ┌────────────────────▼─────────────────────────────────┐    │
│  │              USB OTG FS Controller                   │    │
│  │                   (CN13)                             │    │
│  └────────────────────┬─────────────────────────────────┘    │
│                       │                                      │
└───────────────────────┼──────────────────────────────────────┘
                        │
                   USB Cable
                        │
               ┌────────▼────────┐
               │  USB Flash      │
               │  Drive          │
               └─────────────────┘
```

## 하드웨어 연결

### USB 커넥터
| 커넥터 | 위치 | 용도 |
|--------|------|------|
| CN13 | USB-C OTG | USB Host (저장 장치 연결) |
| CN14 | Micro-USB | ST-Link (디버깅) |

### 전원 요구사항
- **VBUS**: 5V 공급
- **최대 전류**: 500mA (일부 외장 HDD는 추가 전원 필요)

### LED 상태
| LED | 상태 | 의미 |
|-----|------|------|
| LED1 (녹색) | 켜짐 | USB 드라이브 연결됨 |
| LED2 (주황) | 점멸 | 읽기/쓰기 동작 중 |
| LED3 (빨강) | 켜짐 | 오류 발생 |

## 코드 구현

### 1. USB Host 및 FatFs 초기화
```c
#include "usbh_core.h"
#include "usbh_msc.h"
#include "ff.h"
#include "ff_gen_drv.h"
#include "usbh_diskio.h"

USBH_HandleTypeDef hUsbHostFS;
FATFS USBDISKFatFs;
char USBDISKPath[4];  // "0:/"
FIL MyFile;

int main(void)
{
  // 시스템 초기화
  HAL_Init();
  SystemClock_Config();

  // LED 초기화
  BSP_LED_Init(LED1);
  BSP_LED_Init(LED2);
  BSP_LED_Init(LED3);

  // FatFs 드라이버 링크
  FATFS_LinkDriver(&USBH_Driver, USBDISKPath);

  // USB Host 초기화
  USBH_Init(&hUsbHostFS, USBH_UserProcess, HOST_FS);

  // MSC 클래스 등록
  USBH_RegisterClass(&hUsbHostFS, USBH_MSC_CLASS);

  // USB Host 시작
  USBH_Start(&hUsbHostFS);

  while (1)
  {
    // USB Host 상태 머신 처리
    USBH_Process(&hUsbHostFS);
  }
}
```

### 2. USB Host 콜백 처리
```c
typedef enum {
  APPLICATION_IDLE,
  APPLICATION_START,
  APPLICATION_READY,
  APPLICATION_DISCONNECT
} ApplicationState;

ApplicationState app_state = APPLICATION_IDLE;

void USBH_UserProcess(USBH_HandleTypeDef *phost, uint8_t id)
{
  switch (id)
  {
    case HOST_USER_SELECT_CONFIGURATION:
      break;

    case HOST_USER_DISCONNECTION:
      app_state = APPLICATION_DISCONNECT;
      BSP_LED_Off(LED1);

      // 파일 시스템 마운트 해제
      f_mount(NULL, USBDISKPath, 0);
      printf("USB Drive Disconnected\n");
      break;

    case HOST_USER_CLASS_ACTIVE:
      app_state = APPLICATION_READY;
      BSP_LED_On(LED1);

      // 드라이브 정보 출력
      MSC_PrintDriveInfo();

      // 파일 시스템 마운트
      if (f_mount(&USBDISKFatFs, USBDISKPath, 0) == FR_OK)
      {
        printf("USB Drive Mounted\n");

        // 테스트 파일 작업
        MSC_FileOperationTest();
      }
      break;

    case HOST_USER_CONNECTION:
      app_state = APPLICATION_START;
      printf("USB Drive Connected\n");
      break;

    default:
      break;
  }
}
```

### 3. 드라이브 정보 출력
```c
void MSC_PrintDriveInfo(void)
{
  MSC_LUNTypeDef lun_info;

  // LUN 정보 가져오기
  USBH_MSC_GetLUNInfo(&hUsbHostFS, 0, &lun_info);

  printf("\n--- USB Drive Information ---\n");
  printf("Capacity: %lu MB\n",
         (lun_info.capacity.block_nbr / 2048));
  printf("Block Size: %d bytes\n",
         lun_info.capacity.block_size);
  printf("Vendor: %.8s\n", lun_info.inquiry.vendor_id);
  printf("Product: %.16s\n", lun_info.inquiry.product_id);
  printf("Revision: %.4s\n", lun_info.inquiry.revision_id);
  printf("-----------------------------\n");
}
```

### 4. 파일 작업 테스트
```c
FRESULT MSC_FileOperationTest(void)
{
  FRESULT res;
  uint32_t bytesWritten, bytesRead;
  uint8_t wtext[] = "STM32H745 USB MSC Host Test\r\n";
  uint8_t rtext[100];

  printf("\n--- File Operation Test ---\n");

  // 파일 생성 및 쓰기
  res = f_open(&MyFile, "0:/test.txt", FA_CREATE_ALWAYS | FA_WRITE);
  if (res != FR_OK)
  {
    printf("Error: Cannot create file (0x%02X)\n", res);
    return res;
  }

  res = f_write(&MyFile, wtext, sizeof(wtext)-1, (void*)&bytesWritten);
  if (res != FR_OK || bytesWritten != sizeof(wtext)-1)
  {
    printf("Error: Write failed\n");
    f_close(&MyFile);
    return res;
  }

  printf("Written %lu bytes to test.txt\n", bytesWritten);
  f_close(&MyFile);
  BSP_LED_Toggle(LED2);

  // 파일 읽기
  res = f_open(&MyFile, "0:/test.txt", FA_READ);
  if (res != FR_OK)
  {
    printf("Error: Cannot open file for reading\n");
    return res;
  }

  res = f_read(&MyFile, rtext, sizeof(rtext), (void*)&bytesRead);
  if (res != FR_OK)
  {
    printf("Error: Read failed\n");
    f_close(&MyFile);
    return res;
  }

  rtext[bytesRead] = '\0';
  printf("Read %lu bytes: %s\n", bytesRead, rtext);
  f_close(&MyFile);
  BSP_LED_Toggle(LED2);

  printf("--- Test Complete ---\n");

  return FR_OK;
}
```

### 5. 디렉토리 탐색
```c
FRESULT MSC_ScanDirectory(char *path)
{
  FRESULT res;
  DIR dir;
  FILINFO fno;

  res = f_opendir(&dir, path);
  if (res != FR_OK)
  {
    printf("Error: Cannot open directory\n");
    return res;
  }

  printf("\nContents of %s:\n", path);
  printf("%-30s %10s %s\n", "Name", "Size", "Attr");
  printf("--------------------------------------------\n");

  for (;;)
  {
    res = f_readdir(&dir, &fno);
    if (res != FR_OK || fno.fname[0] == 0)
      break;  // 오류 또는 디렉토리 끝

    if (fno.fattrib & AM_DIR)
    {
      // 디렉토리
      printf("%-30s %10s <DIR>\n", fno.fname, "");
    }
    else
    {
      // 파일
      printf("%-30s %10lu\n", fno.fname, fno.fsize);
    }
  }

  f_closedir(&dir);
  return res;
}
```

### 6. 대용량 파일 복사
```c
#define COPY_BUFFER_SIZE  4096

FRESULT MSC_CopyFile(const char *src_path, const char *dst_path)
{
  FIL src_file, dst_file;
  FRESULT res;
  uint32_t bytes_read, bytes_written;
  uint8_t buffer[COPY_BUFFER_SIZE];
  uint32_t total_bytes = 0;

  // 원본 파일 열기
  res = f_open(&src_file, src_path, FA_READ);
  if (res != FR_OK) return res;

  // 대상 파일 생성
  res = f_open(&dst_file, dst_path, FA_CREATE_ALWAYS | FA_WRITE);
  if (res != FR_OK)
  {
    f_close(&src_file);
    return res;
  }

  // 복사 루프
  uint32_t file_size = f_size(&src_file);
  printf("Copying %s (%lu bytes)...\n", src_path, file_size);

  while (1)
  {
    // 읽기
    res = f_read(&src_file, buffer, COPY_BUFFER_SIZE, (void*)&bytes_read);
    if (res != FR_OK || bytes_read == 0)
      break;

    // 쓰기
    res = f_write(&dst_file, buffer, bytes_read, (void*)&bytes_written);
    if (res != FR_OK || bytes_written != bytes_read)
      break;

    total_bytes += bytes_written;

    // 진행률 표시
    printf("\rProgress: %lu%%", (total_bytes * 100) / file_size);
    BSP_LED_Toggle(LED2);
  }

  printf("\nCopied %lu bytes\n", total_bytes);

  f_close(&src_file);
  f_close(&dst_file);

  return res;
}
```

## FatFs 설정

### ffconf.h 주요 설정
```c
// 파일 시스템 기능
#define FF_FS_READONLY     0   // 읽기/쓰기 모두 지원
#define FF_FS_MINIMIZE     0   // 모든 기능 사용
#define FF_USE_STRFUNC     1   // f_gets, f_putc 등 문자열 함수
#define FF_USE_FIND        1   // f_findfirst, f_findnext
#define FF_USE_MKFS        1   // f_mkfs 포맷 기능
#define FF_USE_LABEL       1   // 볼륨 레이블 지원

// 로케일 및 코드 페이지
#define FF_CODE_PAGE       949 // 한국어 (CP949)
#define FF_USE_LFN         2   // 긴 파일 이름 지원
#define FF_LFN_UNICODE     0   // ANSI/OEM 문자열
#define FF_MAX_LFN         255 // 최대 파일 이름 길이

// 볼륨 설정
#define FF_VOLUMES         1   // 논리 드라이브 수
#define FF_MULTI_PARTITION 0   // 단일 파티션

// 성능 설정
#define FF_FS_TINY         0   // 일반 모드
#define FF_FS_EXFAT        1   // exFAT 지원
```

## 고급 기능

### 1. 드라이브 포맷
```c
FRESULT MSC_FormatDrive(void)
{
  FRESULT res;
  uint8_t work_buffer[FF_MAX_SS];

  printf("Formatting USB Drive...\n");

  // FAT32로 포맷
  res = f_mkfs(USBDISKPath, FM_FAT32, 0, work_buffer, sizeof(work_buffer));

  if (res == FR_OK)
  {
    printf("Format Complete!\n");

    // 볼륨 레이블 설정
    f_setlabel("STM32USB");
  }
  else
  {
    printf("Format Failed (0x%02X)\n", res);
  }

  return res;
}
```

### 2. 드라이브 용량 확인
```c
void MSC_GetDriveSpace(void)
{
  FATFS *fs;
  DWORD fre_clust, fre_sect, tot_sect;

  // 클러스터 정보 가져오기
  FRESULT res = f_getfree(USBDISKPath, &fre_clust, &fs);

  if (res == FR_OK)
  {
    // 전체 및 빈 섹터 수 계산
    tot_sect = (fs->n_fatent - 2) * fs->csize;
    fre_sect = fre_clust * fs->csize;

    // KB 단위로 변환 (섹터 크기 512 바이트 가정)
    printf("Total Space: %lu KB\n", tot_sect / 2);
    printf("Free Space:  %lu KB\n", fre_sect / 2);
    printf("Used Space:  %lu KB\n", (tot_sect - fre_sect) / 2);
  }
}
```

### 3. 파일 검색
```c
FRESULT MSC_FindFiles(const char *path, const char *pattern)
{
  FRESULT res;
  DIR dir;
  FILINFO fno;

  // 패턴으로 검색 시작
  res = f_findfirst(&dir, &fno, path, pattern);

  printf("Searching for '%s' in %s:\n", pattern, path);

  while (res == FR_OK && fno.fname[0])
  {
    printf("  Found: %s (%lu bytes)\n", fno.fname, fno.fsize);
    res = f_findnext(&dir, &fno);
  }

  f_closedir(&dir);
  return res;
}

// 사용 예: 모든 텍스트 파일 찾기
// MSC_FindFiles("0:/", "*.txt");
```

### 4. 바이너리 파일 읽기 (펌웨어 등)
```c
FRESULT MSC_ReadBinaryFile(const char *path, uint8_t *buffer, uint32_t max_size, uint32_t *actual_size)
{
  FIL file;
  FRESULT res;

  res = f_open(&file, path, FA_READ);
  if (res != FR_OK) return res;

  // 파일 크기 확인
  uint32_t file_size = f_size(&file);
  if (file_size > max_size)
  {
    f_close(&file);
    return FR_INVALID_PARAMETER;
  }

  // 전체 파일 읽기
  res = f_read(&file, buffer, file_size, (void*)actual_size);

  f_close(&file);
  return res;
}
```

### 5. 로그 파일 추가 쓰기
```c
void MSC_AppendLog(const char *message)
{
  FIL log_file;
  UINT bytes_written;
  char timestamp[32];

  // 로그 파일 열기 (추가 모드)
  if (f_open(&log_file, "0:/system.log", FA_OPEN_APPEND | FA_WRITE) != FR_OK)
  {
    // 파일이 없으면 생성
    f_open(&log_file, "0:/system.log", FA_CREATE_ALWAYS | FA_WRITE);
  }

  // 타임스탬프 추가
  RTC_TimeTypeDef time;
  RTC_DateTypeDef date;
  HAL_RTC_GetTime(&hrtc, &time, RTC_FORMAT_BIN);
  HAL_RTC_GetDate(&hrtc, &date, RTC_FORMAT_BIN);

  sprintf(timestamp, "[%04d-%02d-%02d %02d:%02d:%02d] ",
          2000 + date.Year, date.Month, date.Date,
          time.Hours, time.Minutes, time.Seconds);

  // 로그 쓰기
  f_write(&log_file, timestamp, strlen(timestamp), &bytes_written);
  f_write(&log_file, message, strlen(message), &bytes_written);
  f_write(&log_file, "\r\n", 2, &bytes_written);

  f_close(&log_file);
}
```

## USB MSC 설정

### usbh_msc.h 설정
```c
// MSC 설정
#define USBH_MSC_MAX_LUN     2     // 최대 LUN 수
#define MSC_BOT_CBW_TAG      0x20304050
```

### SCSI 명령 타임아웃
```c
// 장시간 작업을 위한 타임아웃 증가
#define USBH_MSC_TIMEOUT     10000  // 10초
```

## 응용 예제

### 1. 데이터 로거
```c
void DataLogger_Task(void)
{
  FIL data_file;
  char filename[32];

  // 날짜별 파일 생성
  sprintf(filename, "0:/log_%04d%02d%02d.csv", year, month, day);

  if (f_open(&data_file, filename, FA_OPEN_APPEND | FA_WRITE) == FR_OK)
  {
    char line[128];
    sprintf(line, "%02d:%02d:%02d,%d,%d,%d\r\n",
            time.Hours, time.Minutes, time.Seconds,
            temperature, humidity, pressure);

    UINT written;
    f_write(&data_file, line, strlen(line), &written);
    f_close(&data_file);
  }
}
```

### 2. 펌웨어 업데이트
```c
void FirmwareUpdate_FromUSB(void)
{
  uint8_t *fw_buffer = (uint8_t*)0x24000000;  // AXI SRAM 사용
  uint32_t fw_size;

  // USB에서 펌웨어 파일 읽기
  if (MSC_ReadBinaryFile("0:/firmware.bin", fw_buffer, 512*1024, &fw_size) == FR_OK)
  {
    printf("Firmware loaded: %lu bytes\n", fw_size);

    // CRC 검증
    uint32_t crc = Calculate_CRC32(fw_buffer, fw_size - 4);
    uint32_t stored_crc = *(uint32_t*)(fw_buffer + fw_size - 4);

    if (crc == stored_crc)
    {
      printf("CRC OK, updating firmware...\n");
      // Flash 프로그래밍...
    }
    else
    {
      printf("CRC Error!\n");
    }
  }
}
```

### 3. 설정 파일 로드
```c
void LoadConfiguration(void)
{
  FIL config_file;
  char line[128];

  if (f_open(&config_file, "0:/config.ini", FA_READ) == FR_OK)
  {
    while (f_gets(line, sizeof(line), &config_file))
    {
      // 간단한 INI 파서
      if (strncmp(line, "baudrate=", 9) == 0)
      {
        config.baudrate = atoi(line + 9);
      }
      else if (strncmp(line, "timeout=", 8) == 0)
      {
        config.timeout = atoi(line + 8);
      }
    }

    f_close(&config_file);
    printf("Configuration loaded\n");
  }
}
```

## 트러블슈팅

### USB 드라이브가 인식되지 않는 경우

1. **전원 확인**:
   - VBUS 5V 공급 확인
   - 외장 HDD는 별도 전원 필요

2. **파일 시스템 확인**:
   - FAT16, FAT32, exFAT 지원
   - NTFS는 지원하지 않음

3. **드라이브 상태 확인**:
   ```c
   USBH_MSC_UnitIsReady(&hUsbHostFS, 0);
   ```

### 마운트 실패

1. **FatFs 드라이버 링크 확인**:
   ```c
   FATFS_LinkDriver(&USBH_Driver, USBDISKPath);
   ```

2. **드라이브 경로 확인**: "0:/" 형식 사용

3. **파티션 테이블**: MBR 또는 GPT 확인

### 읽기/쓰기 오류

1. **쓰기 보호 확인**: 물리적 스위치 확인

2. **디스크 공간 확인**: 충분한 여유 공간

3. **파일 핸들 닫기**: 작업 완료 후 반드시 f_close()

### 속도가 느린 경우

1. **클러스터 크기**: 큰 클러스터로 포맷
2. **버퍼 크기**: 더 큰 전송 버퍼 사용
3. **USB 허브**: 직접 연결 권장

## 성능 고려사항

### 전송 속도
- **USB Full Speed**: 최대 12 Mbps
- **실제 속도**: 약 1 MB/s
- **대용량 전송**: 큰 버퍼 사용 권장

### 메모리 사용
```c
// FatFs 작업 버퍼
#define FF_MAX_SS  4096  // 섹터 크기

// 파일 버퍼
static uint8_t file_buffer[4096];
```

### 캐시 활성화
```c
// DMA 캐시 관리
SCB_CleanDCache_by_Addr((uint32_t*)buffer, size);
SCB_InvalidateDCache_by_Addr((uint32_t*)buffer, size);
```

## 참고 자료

- USB Mass Storage Class 사양: https://www.usb.org/documents
- FatFs 공식 사이트: http://elm-chan.org/fsw/ff/
- STM32 USB Host Library: UM1720
- 예제: `STM32H745I-DISCO/Applications/USB_Host/MSC_Standalone`
