# FLASH_CoreConfiguration 예제

## 개요

이 예제는 STM32H745I-DISCO의 듀얼 코어 환경에서 Cortex-M7이 Cortex-M4의 펌웨어를 Flash Bank2에 프로그래밍하고, 부팅 메커니즘을 제어하는 방법을 보여줍니다. 필드 업데이트, 이중 뱅크 부팅, 그리고 안전한 펌웨어 업데이트 기법을 학습할 수 있습니다.

## 하드웨어 요구사항

- **보드**: STM32H745I-DISCO
- **Flash 메모리 구성**:
  - Bank1 (0x08000000 - 0x080FFFFF): CM7 코드 (1MB)
  - Bank2 (0x08100000 - 0x081FFFFF): CM4 코드 (1MB)
- **LED**:
  - LED1 (녹색): 프로그래밍 진행 표시
  - LED2 (주황색): CM4 실행 중
  - LED3 (빨간색): 프로그래밍 오류
  - LED4 (파란색): 작업 완료

## STM32H7 Flash 메모리 구조

### Flash Bank 구성

```
┌─────────────────────────────────────────┐
│  Bank 1 (1 MB)                          │
│  0x08000000 - 0x080FFFFF                │
│  ┌────────────────────────────────┐    │
│  │ CM7 Boot Code                  │    │
│  │ CM7 Application                │    │
│  │ CM7 Data                       │    │
│  └────────────────────────────────┘    │
└─────────────────────────────────────────┘
┌─────────────────────────────────────────┐
│  Bank 2 (1 MB)                          │
│  0x08100000 - 0x081FFFFF                │
│  ┌────────────────────────────────┐    │
│  │ CM4 Boot Code                  │    │
│  │ CM4 Application                │    │
│  │ CM4 Firmware Update Area       │    │
│  └────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

### 섹터 구조 (각 Bank)

- **8KB 섹터**: 128개 섹터 × 8KB = 1024KB (1MB)
- **섹터 0-127**: 각각 8KB

## 주요 기능

### 1. CM7에서 CM4 펌웨어 프로그래밍
- CM7이 마스터 역할 수행
- CM4 코드를 Flash Bank2에 동적으로 프로그래밍
- 프로그래밍 중 CM4를 Hold 상태로 유지

### 2. Hold Boot 메커니즘
- CM4를 리셋 후 대기 상태로 유지
- 펌웨어 업데이트 완료 후 CM4 릴리스
- 안전한 코드 전환

### 3. 이중 뱅크 프로그래밍
- 읽기-동시-쓰기(Read-While-Write) 지원
- Bank1에서 코드 실행 중 Bank2 프로그래밍 가능
- 시스템 중단 없는 업데이트

## 소프트웨어 설명

### Flash 프로그래밍 구조체

```c
/* flash_programmer.h */

/* Flash 주소 정의 */
#define FLASH_BANK1_BASE      0x08000000UL
#define FLASH_BANK2_BASE      0x08100000UL
#define FLASH_BANK_SIZE       0x00100000UL  /* 1 MB */

#define CM7_FLASH_START       FLASH_BANK1_BASE
#define CM7_FLASH_SIZE        FLASH_BANK_SIZE

#define CM4_FLASH_START       FLASH_BANK2_BASE
#define CM4_FLASH_SIZE        FLASH_BANK_SIZE

/* Flash 프로그래밍 설정 */
#define FLASH_SECTOR_SIZE     8192    /* 8 KB */
#define FLASH_WORD_SIZE       32      /* 256-bit (32 bytes) */
#define FLASH_SECTORS_PER_BANK 128

/* 프로그래밍 상태 */
typedef enum
{
    PROG_STATE_IDLE = 0,
    PROG_STATE_ERASING,
    PROG_STATE_PROGRAMMING,
    PROG_STATE_VERIFYING,
    PROG_STATE_COMPLETE,
    PROG_STATE_ERROR
} ProgrammingState_t;

/* 펌웨어 헤더 구조체 */
typedef struct
{
    uint32_t magic;           /* 매직 넘버: 0x46575548 "FWUH" */
    uint32_t version;         /* 펌웨어 버전 */
    uint32_t length;          /* 펌웨어 크기 (바이트) */
    uint32_t crc32;           /* CRC32 체크섬 */
    uint32_t target_core;     /* 대상 코어 (4 또는 7) */
    uint32_t load_address;    /* 로드 주소 */
    uint32_t entry_point;     /* 진입점 주소 */
    uint32_t timestamp;       /* 빌드 타임스탬프 */
    uint8_t  reserved[32];
} FirmwareHeader_t;

/* 프로그래밍 진행 정보 */
typedef struct
{
    ProgrammingState_t state;
    uint32_t total_sectors;
    uint32_t erased_sectors;
    uint32_t programmed_bytes;
    uint32_t total_bytes;
    uint32_t start_time;
    uint32_t end_time;
    uint32_t error_code;
} ProgrammingProgress_t;
```

### Flash 프로그래밍 구현

```c
/* flash_programmer.c */

static ProgrammingProgress_t prog_progress;

/**
  * @brief  Flash 프로그래머 초기화
  */
void FlashProgrammer_Init(void)
{
    memset(&prog_progress, 0, sizeof(ProgrammingProgress_t));
    prog_progress.state = PROG_STATE_IDLE;

    /* Flash 인터페이스 클록 활성화 */
    __HAL_RCC_FLASH_CLK_ENABLE();
}

/**
  * @brief  CM4 Hold 모드 진입
  */
void CM4_HoldBoot(void)
{
    /* CM4 클록 비활성화 */
    __HAL_RCC_C2_FORCE_RESET();

    /* CM4를 리셋 상태로 유지 */
    /* BOOT_CM4_ADD0 레지스터는 유지 */

    printf("CM4 held in reset\n");
}

/**
  * @brief  CM4 부팅 주소 설정
  */
void CM4_SetBootAddress(uint32_t address)
{
    /* CM4 부팅 주소를 Flash Bank2로 설정 */
    HAL_SYSCFG_CM4SetBootAddress0(address);

    printf("CM4 boot address set to 0x%08lX\n", address);
}

/**
  * @brief  CM4 릴리스 및 부팅
  */
void CM4_ReleaseBoot(void)
{
    /* CM4 리셋 해제 */
    __HAL_RCC_C2_RELEASE_RESET();

    /* CM4 클록 활성화 */
    HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);

    printf("CM4 released and booted\n");
}

/**
  * @brief  Flash 섹터 소거
  * @param  bank: Flash 뱅크 (FLASH_BANK_1 또는 FLASH_BANK_2)
  * @param  sector: 시작 섹터 번호
  * @param  nb_sectors: 소거할 섹터 개수
  */
HAL_StatusTypeDef Flash_EraseSectors(uint32_t bank, uint32_t sector,
                                      uint32_t nb_sectors)
{
    FLASH_EraseInitTypeDef EraseInitStruct;
    uint32_t SectorError = 0;
    HAL_StatusTypeDef status;

    prog_progress.state = PROG_STATE_ERASING;
    prog_progress.total_sectors = nb_sectors;
    prog_progress.erased_sectors = 0;

    /* Flash 언락 */
    HAL_FLASH_Unlock();

    /* 소거 파라미터 설정 */
    EraseInitStruct.TypeErase = FLASH_TYPEERASE_SECTORS;
    EraseInitStruct.Banks = bank;
    EraseInitStruct.Sector = sector;
    EraseInitStruct.NbSectors = nb_sectors;
    EraseInitStruct.VoltageRange = FLASH_VOLTAGE_RANGE_3;  /* 2.7V - 3.6V */

    printf("Erasing %lu sectors starting from sector %lu in bank %lu\n",
           nb_sectors, sector, bank);

    /* 섹터 소거 실행 */
    status = HAL_FLASHEx_Erase(&EraseInitStruct, &SectorError);

    if (status != HAL_OK)
    {
        printf("Erase failed at sector %lu, error: 0x%08lX\n",
               SectorError, HAL_FLASH_GetError());
        prog_progress.state = PROG_STATE_ERROR;
        prog_progress.error_code = HAL_FLASH_GetError();
    }
    else
    {
        prog_progress.erased_sectors = nb_sectors;
        printf("Erase completed successfully\n");
    }

    /* Flash 락 */
    HAL_FLASH_Lock();

    return status;
}

/**
  * @brief  Flash 프로그래밍 (256-bit 단위)
  * @param  address: 대상 주소 (32바이트 정렬 필수)
  * @param  data: 프로그래밍할 데이터 (32바이트)
  */
HAL_StatusTypeDef Flash_Program256Bit(uint32_t address, uint8_t *data)
{
    HAL_StatusTypeDef status;

    /* 주소 정렬 확인 */
    if ((address % 32) != 0)
    {
        printf("Error: Address 0x%08lX not aligned to 32 bytes\n", address);
        return HAL_ERROR;
    }

    /* Flash 언락 */
    HAL_FLASH_Unlock();

    /* 256-bit (32바이트) 프로그래밍 */
    status = HAL_FLASH_Program(FLASH_TYPEPROGRAM_FLASHWORD, address,
                               (uint32_t)data);

    if (status != HAL_OK)
    {
        printf("Programming failed at 0x%08lX, error: 0x%08lX\n",
               address, HAL_FLASH_GetError());
        prog_progress.state = PROG_STATE_ERROR;
        prog_progress.error_code = HAL_FLASH_GetError();
    }

    /* Flash 락 */
    HAL_FLASH_Lock();

    return status;
}

/**
  * @brief  펌웨어 이미지 프로그래밍
  * @param  firmware: 펌웨어 이미지 포인터
  * @param  length: 펌웨어 크기
  * @param  dest_address: 대상 Flash 주소
  */
HAL_StatusTypeDef Flash_ProgramFirmware(uint8_t *firmware, uint32_t length,
                                         uint32_t dest_address)
{
    HAL_StatusTypeDef status = HAL_OK;
    uint32_t programmed = 0;
    uint8_t flash_word[32];

    prog_progress.state = PROG_STATE_PROGRAMMING;
    prog_progress.total_bytes = length;
    prog_progress.programmed_bytes = 0;
    prog_progress.start_time = HAL_GetTick();

    printf("Programming %lu bytes to 0x%08lX\n", length, dest_address);

    /* Flash 언락 */
    HAL_FLASH_Unlock();

    /* 32바이트 단위로 프로그래밍 */
    while (programmed < length)
    {
        uint32_t chunk_size = (length - programmed >= 32) ? 32 : (length - programmed);

        /* 데이터 복사 (32바이트 미만인 경우 0xFF로 패딩) */
        memset(flash_word, 0xFF, sizeof(flash_word));
        memcpy(flash_word, firmware + programmed, chunk_size);

        /* 프로그래밍 */
        status = HAL_FLASH_Program(FLASH_TYPEPROGRAM_FLASHWORD,
                                   dest_address + programmed,
                                   (uint32_t)flash_word);

        if (status != HAL_OK)
        {
            printf("Programming failed at offset %lu (addr 0x%08lX)\n",
                   programmed, dest_address + programmed);
            break;
        }

        programmed += 32;
        prog_progress.programmed_bytes = programmed;

        /* 진행 상황 표시 (10% 단위) */
        if ((programmed % (length / 10)) == 0)
        {
            uint32_t percent = (programmed * 100) / length;
            printf("Programming: %lu%%\n", percent);
            BSP_LED_Toggle(LED1);
        }
    }

    /* Flash 락 */
    HAL_FLASH_Lock();

    prog_progress.end_time = HAL_GetTick();

    if (status == HAL_OK)
    {
        printf("Programming completed in %lu ms\n",
               prog_progress.end_time - prog_progress.start_time);
        prog_progress.state = PROG_STATE_COMPLETE;
    }
    else
    {
        prog_progress.state = PROG_STATE_ERROR;
    }

    return status;
}

/**
  * @brief  프로그래밍 검증
  * @param  source: 원본 데이터
  * @param  dest_address: Flash 주소
  * @param  length: 검증할 크기
  */
HAL_StatusTypeDef Flash_VerifyFirmware(uint8_t *source, uint32_t dest_address,
                                        uint32_t length)
{
    prog_progress.state = PROG_STATE_VERIFYING;

    printf("Verifying %lu bytes at 0x%08lX\n", length, dest_address);

    /* 메모리 비교 */
    if (memcmp(source, (void*)dest_address, length) != 0)
    {
        printf("Verification failed!\n");

        /* 불일치 위치 찾기 */
        for (uint32_t i = 0; i < length; i++)
        {
            uint8_t flash_byte = *((volatile uint8_t*)(dest_address + i));
            if (source[i] != flash_byte)
            {
                printf("Mismatch at offset %lu: expected 0x%02X, got 0x%02X\n",
                       i, source[i], flash_byte);

                /* 처음 10개만 출력 */
                static uint32_t mismatch_count = 0;
                if (++mismatch_count >= 10)
                {
                    printf("... (more mismatches)\n");
                    break;
                }
            }
        }

        prog_progress.state = PROG_STATE_ERROR;
        return HAL_ERROR;
    }

    printf("Verification successful!\n");
    prog_progress.state = PROG_STATE_COMPLETE;

    return HAL_OK;
}

/**
  * @brief  CRC32 계산
  */
uint32_t Calculate_CRC32(uint8_t *data, uint32_t length)
{
    CRC_HandleTypeDef hcrc;

    /* CRC 주변장치 초기화 */
    __HAL_RCC_CRC_CLK_ENABLE();

    hcrc.Instance = CRC;
    hcrc.Init.DefaultPolynomialUse = DEFAULT_POLYNOMIAL_ENABLE;
    hcrc.Init.DefaultInitValueUse = DEFAULT_INIT_VALUE_ENABLE;
    hcrc.Init.InputDataInversionMode = CRC_INPUTDATA_INVERSION_NONE;
    hcrc.Init.OutputDataInversionMode = CRC_OUTPUTDATA_INVERSION_DISABLE;
    hcrc.InputDataFormat = CRC_INPUTDATA_FORMAT_BYTES;

    if (HAL_CRC_Init(&hcrc) != HAL_OK)
    {
        Error_Handler();
    }

    /* CRC 계산 */
    uint32_t crc = HAL_CRC_Calculate(&hcrc, (uint32_t*)data, length);

    return crc;
}

/**
  * @brief  펌웨어 헤더 검증
  */
HAL_StatusTypeDef Verify_FirmwareHeader(FirmwareHeader_t *header)
{
    /* 매직 넘버 확인 */
    if (header->magic != 0x46575548)  /* "FWUH" */
    {
        printf("Invalid magic number: 0x%08lX\n", header->magic);
        return HAL_ERROR;
    }

    /* 코어 확인 */
    if (header->target_core != 4 && header->target_core != 7)
    {
        printf("Invalid target core: %lu\n", header->target_core);
        return HAL_ERROR;
    }

    /* 크기 확인 */
    if (header->length > FLASH_BANK_SIZE)
    {
        printf("Firmware too large: %lu bytes\n", header->length);
        return HAL_ERROR;
    }

    /* 주소 확인 */
    if (header->target_core == 4 &&
        header->load_address != CM4_FLASH_START)
    {
        printf("Invalid CM4 load address: 0x%08lX\n", header->load_address);
        return HAL_ERROR;
    }

    printf("Firmware header validated:\n");
    printf("  Version: %lu\n", header->version);
    printf("  Length: %lu bytes\n", header->length);
    printf("  CRC32: 0x%08lX\n", header->crc32);
    printf("  Target: CM%lu\n", header->target_core);
    printf("  Load Address: 0x%08lX\n", header->load_address);

    return HAL_OK;
}
```

### 완전한 업데이트 프로세스

```c
/* firmware_update.c */

/* 임베디드 펌웨어 (링커에서 외부 심볼로 정의) */
extern uint8_t _binary_cm4_firmware_bin_start[];
extern uint8_t _binary_cm4_firmware_bin_end[];
extern uint32_t _binary_cm4_firmware_bin_size;

/**
  * @brief  CM4 펌웨어 업데이트 전체 프로세스
  */
HAL_StatusTypeDef Update_CM4_Firmware(void)
{
    HAL_StatusTypeDef status;
    FirmwareHeader_t *fw_header;
    uint8_t *fw_data;
    uint32_t fw_size;

    printf("\n========================================\n");
    printf("  CM4 Firmware Update Process\n");
    printf("========================================\n\n");

    /* 1. CM4 Hold */
    printf("[Step 1/7] Holding CM4 in reset...\n");
    CM4_HoldBoot();
    HAL_Delay(100);

    /* 2. 펌웨어 이미지 로드 */
    printf("[Step 2/7] Loading firmware image...\n");

    fw_data = _binary_cm4_firmware_bin_start;
    fw_size = (uint32_t)&_binary_cm4_firmware_bin_size;

    printf("  Firmware size: %lu bytes\n", fw_size);

    /* 3. 헤더 검증 */
    printf("[Step 3/7] Verifying firmware header...\n");

    fw_header = (FirmwareHeader_t*)fw_data;

    if (Verify_FirmwareHeader(fw_header) != HAL_OK)
    {
        printf("Header verification failed!\n");
        return HAL_ERROR;
    }

    /* 4. CRC 검증 */
    printf("[Step 4/7] Verifying CRC32...\n");

    uint8_t *fw_payload = fw_data + sizeof(FirmwareHeader_t);
    uint32_t payload_size = fw_header->length;

    uint32_t calculated_crc = Calculate_CRC32(fw_payload, payload_size);

    if (calculated_crc != fw_header->crc32)
    {
        printf("CRC mismatch! Expected: 0x%08lX, Got: 0x%08lX\n",
               fw_header->crc32, calculated_crc);
        BSP_LED_On(LED3);
        return HAL_ERROR;
    }

    printf("  CRC verified: 0x%08lX\n", calculated_crc);

    /* 5. Flash 소거 */
    printf("[Step 5/7] Erasing Flash Bank 2...\n");

    /* 필요한 섹터 개수 계산 */
    uint32_t required_sectors = (payload_size + FLASH_SECTOR_SIZE - 1) /
                                 FLASH_SECTOR_SIZE;

    status = Flash_EraseSectors(FLASH_BANK_2, 0, required_sectors);

    if (status != HAL_OK)
    {
        printf("Erase failed!\n");
        BSP_LED_On(LED3);
        return status;
    }

    /* 6. 프로그래밍 */
    printf("[Step 6/7] Programming firmware...\n");

    status = Flash_ProgramFirmware(fw_payload, payload_size, CM4_FLASH_START);

    if (status != HAL_OK)
    {
        printf("Programming failed!\n");
        BSP_LED_On(LED3);
        return status;
    }

    /* 7. 검증 */
    printf("[Step 7/7] Verifying programmed firmware...\n");

    status = Flash_VerifyFirmware(fw_payload, CM4_FLASH_START, payload_size);

    if (status != HAL_OK)
    {
        printf("Verification failed!\n");
        BSP_LED_On(LED3);
        return status;
    }

    /* 8. CM4 부팅 주소 설정 및 릴리스 */
    printf("\nSetting CM4 boot address and releasing...\n");

    CM4_SetBootAddress(fw_header->entry_point);
    CM4_ReleaseBoot();

    printf("\n========================================\n");
    printf("  CM4 Firmware Update Complete!\n");
    printf("========================================\n\n");

    BSP_LED_On(LED4);

    return HAL_OK;
}
```

## CM7 메인 코드

```c
/* CM7/main.c */

int main(void)
{
    /* HAL 초기화 */
    HAL_Init();

    /* 시스템 클록 설정 (480MHz) */
    SystemClock_Config();

    /* LED 초기화 */
    BSP_LED_Init(LED1);
    BSP_LED_Init(LED2);
    BSP_LED_Init(LED3);
    BSP_LED_Init(LED4);

    /* UART 초기화 (디버그 출력) */
    UART_Init();

    printf("\n\n");
    printf("*******************************************\n");
    printf("*  STM32H745 Dual Core Flash Programming *\n");
    printf("*******************************************\n\n");

    /* Flash 프로그래머 초기화 */
    FlashProgrammer_Init();

    /* CM4 펌웨어 업데이트 */
    if (Update_CM4_Firmware() == HAL_OK)
    {
        printf("CM4 is now running new firmware\n");
    }
    else
    {
        printf("CM4 firmware update failed\n");
        Error_Handler();
    }

    /* 메인 루프 */
    while (1)
    {
        BSP_LED_Toggle(LED1);
        HAL_Delay(1000);

        /* 프로그래밍 통계 출력 */
        static uint32_t last_stats_time = 0;
        if ((HAL_GetTick() - last_stats_time) >= 10000)
        {
            Print_Programming_Statistics();
            last_stats_time = HAL_GetTick();
        }
    }
}

/**
  * @brief  프로그래밍 통계 출력
  */
void Print_Programming_Statistics(void)
{
    printf("\n=== Programming Statistics ===\n");
    printf("State: %d\n", prog_progress.state);
    printf("Total Sectors: %lu\n", prog_progress.total_sectors);
    printf("Erased Sectors: %lu\n", prog_progress.erased_sectors);
    printf("Programmed Bytes: %lu / %lu\n",
           prog_progress.programmed_bytes, prog_progress.total_bytes);

    if (prog_progress.end_time > prog_progress.start_time)
    {
        uint32_t duration = prog_progress.end_time - prog_progress.start_time;
        printf("Duration: %lu ms\n", duration);

        if (prog_progress.programmed_bytes > 0)
        {
            uint32_t speed = (prog_progress.programmed_bytes * 1000) / duration;
            printf("Programming Speed: %lu bytes/sec\n", speed);
        }
    }

    printf("==============================\n");
}
```

## CM4 펌웨어 예제

```c
/* CM4/main.c */

int main(void)
{
    /* HAL 초기화 */
    HAL_Init();

    /* LED 초기화 */
    BSP_LED_Init(LED2);

    /* CM4 실행 중 표시 */
    BSP_LED_On(LED2);

    /* 펌웨어 버전 정보 출력 (UART 공유) */
    printf("[CM4] Firmware Version: 1.0.0\n");
    printf("[CM4] Build Date: %s %s\n", __DATE__, __TIME__);

    /* 메인 루프 */
    while (1)
    {
        /* CM4 작업 */
        BSP_LED_Toggle(LED2);
        HAL_Delay(500);
    }
}
```

## 빌드 스크립트 (펌웨어 임베딩)

### Makefile 수정

```makefile
# CM7 Makefile

# CM4 바이너리를 오브젝트 파일로 변환
CM4_BIN = ../CM4/build/CM4.bin
CM4_OBJ = build/cm4_firmware.o

$(CM4_OBJ): $(CM4_BIN)
	@echo "Embedding CM4 firmware..."
	$(OBJCOPY) -I binary -O elf32-littlearm -B arm \
		--rename-section .data=.cm4_firmware,alloc,load,readonly,data,contents \
		--redefine-sym _binary_CM4_bin_start=_binary_cm4_firmware_bin_start \
		--redefine-sym _binary_CM4_bin_end=_binary_cm4_firmware_bin_end \
		--redefine-sym _binary_CM4_bin_size=_binary_cm4_firmware_bin_size \
		$(CM4_BIN) $(CM4_OBJ)

# CM4 오브젝트를 링크에 추가
OBJECTS += $(CM4_OBJ)

# CM4 빌드 의존성
$(CM4_BIN):
	@echo "Building CM4 firmware first..."
	$(MAKE) -C ../CM4
```

### 링커 스크립트 수정

```ld
/* CM7 링커 스크립트 */

MEMORY
{
    FLASH (rx)      : ORIGIN = 0x08000000, LENGTH = 1024K
    DTCMRAM (xrw)   : ORIGIN = 0x20000000, LENGTH = 128K
    RAM_D1 (xrw)    : ORIGIN = 0x24000000, LENGTH = 512K
}

SECTIONS
{
    /* CM4 펌웨어 임베딩 섹션 */
    .cm4_firmware :
    {
        . = ALIGN(4);
        KEEP(*(.cm4_firmware))
        . = ALIGN(4);
    } > FLASH
}
```

## 안전한 업데이트 메커니즘

### 1. 이중 이미지 방식

```c
/* dual_image.c */

#define FIRMWARE_SLOT_A  CM4_FLASH_START
#define FIRMWARE_SLOT_B  (CM4_FLASH_START + 0x80000)  /* +512KB */

typedef struct
{
    uint32_t active_slot;       /* 0: Slot A, 1: Slot B */
    uint32_t slot_a_version;
    uint32_t slot_b_version;
    uint32_t slot_a_crc;
    uint32_t slot_b_crc;
    uint8_t  slot_a_valid;
    uint8_t  slot_b_valid;
} FirmwareSlotInfo_t;

FirmwareSlotInfo_t slot_info;

/**
  * @brief  안전한 펌웨어 업데이트
  */
HAL_StatusTypeDef Safe_Update_Firmware(uint8_t *new_firmware, uint32_t length)
{
    /* 현재 비활성 슬롯에 새 펌웨어 프로그래밍 */
    uint32_t target_slot = (slot_info.active_slot == 0) ? FIRMWARE_SLOT_B :
                                                            FIRMWARE_SLOT_A;

    printf("Programming new firmware to slot %s\n",
           (target_slot == FIRMWARE_SLOT_A) ? "A" : "B");

    /* 프로그래밍 */
    if (Flash_ProgramFirmware(new_firmware, length, target_slot) != HAL_OK)
    {
        return HAL_ERROR;
    }

    /* 검증 */
    if (Flash_VerifyFirmware(new_firmware, target_slot, length) != HAL_OK)
    {
        return HAL_ERROR;
    }

    /* CRC 저장 */
    uint32_t new_crc = Calculate_CRC32(new_firmware, length);

    if (target_slot == FIRMWARE_SLOT_A)
    {
        slot_info.slot_a_crc = new_crc;
        slot_info.slot_a_valid = 1;
    }
    else
    {
        slot_info.slot_b_crc = new_crc;
        slot_info.slot_b_valid = 1;
    }

    /* 부팅 슬롯 전환 */
    slot_info.active_slot = (target_slot == FIRMWARE_SLOT_A) ? 0 : 1;

    /* 슬롯 정보를 비휘발성 메모리에 저장 */
    Save_SlotInfo(&slot_info);

    /* CM4 리부팅 */
    CM4_HoldBoot();
    CM4_SetBootAddress(target_slot);
    CM4_ReleaseBoot();

    printf("Firmware update completed, CM4 switched to new slot\n");

    return HAL_OK;
}

/**
  * @brief  펌웨어 롤백
  */
HAL_StatusTypeDef Rollback_Firmware(void)
{
    /* 이전 슬롯으로 복귀 */
    uint32_t rollback_slot = (slot_info.active_slot == 0) ? FIRMWARE_SLOT_B :
                                                              FIRMWARE_SLOT_A;

    /* 롤백 슬롯이 유효한지 확인 */
    uint8_t is_valid = (rollback_slot == FIRMWARE_SLOT_A) ?
                        slot_info.slot_a_valid : slot_info.slot_b_valid;

    if (!is_valid)
    {
        printf("Rollback slot is invalid!\n");
        return HAL_ERROR;
    }

    printf("Rolling back to previous firmware\n");

    /* 슬롯 전환 */
    slot_info.active_slot = (rollback_slot == FIRMWARE_SLOT_A) ? 0 : 1;
    Save_SlotInfo(&slot_info);

    /* CM4 재부팅 */
    CM4_HoldBoot();
    CM4_SetBootAddress(rollback_slot);
    CM4_ReleaseBoot();

    return HAL_OK;
}
```

### 2. 부트로더 모드

```c
/* bootloader.c */

#define BOOTLOADER_FLAG_ADDR  0x2007FFF0  /* Backup SRAM */

typedef enum
{
    BOOT_MODE_NORMAL = 0,
    BOOT_MODE_UPDATE,
    BOOT_MODE_RECOVERY
} BootMode_t;

/**
  * @brief  부트로더 모드 설정
  */
void Set_BootMode(BootMode_t mode)
{
    __HAL_RCC_BKPSRAM_CLK_ENABLE();

    /* Backup SRAM에 부팅 플래그 저장 */
    *(volatile uint32_t*)BOOTLOADER_FLAG_ADDR = (uint32_t)mode;
}

/**
  * @brief  부트로더 모드 확인
  */
BootMode_t Get_BootMode(void)
{
    __HAL_RCC_BKPSRAM_CLK_ENABLE();

    uint32_t flag = *(volatile uint32_t*)BOOTLOADER_FLAG_ADDR;

    return (BootMode_t)flag;
}

/**
  * @brief  부트로더 메인 로직
  */
void Bootloader_Main(void)
{
    BootMode_t boot_mode = Get_BootMode();

    switch (boot_mode)
    {
        case BOOT_MODE_UPDATE:
            printf("Entering firmware update mode...\n");
            Enter_UpdateMode();
            break;

        case BOOT_MODE_RECOVERY:
            printf("Entering recovery mode...\n");
            Enter_RecoveryMode();
            break;

        case BOOT_MODE_NORMAL:
        default:
            printf("Booting normal firmware...\n");
            Boot_Application();
            break;
    }
}

/**
  * @brief  애플리케이션 부팅
  */
void Boot_Application(void)
{
    typedef void (*pFunction)(void);
    pFunction JumpToApplication;

    uint32_t app_addr = CM7_FLASH_START + 0x10000;  /* 64KB 오프셋 */

    /* 스택 포인터 설정 */
    uint32_t stack_ptr = *(volatile uint32_t*)app_addr;
    __set_MSP(stack_ptr);

    /* 진입점 주소 가져오기 */
    uint32_t reset_handler = *(volatile uint32_t*)(app_addr + 4);

    JumpToApplication = (pFunction)reset_handler;

    /* 애플리케이션으로 점프 */
    JumpToApplication();
}
```

## 문제 해결

### 1. 프로그래밍 실패

**증상**: HAL_FLASH_Program이 HAL_ERROR 반환

**진단**:
```c
void Debug_FlashError(void)
{
    uint32_t error = HAL_FLASH_GetError();

    printf("Flash Error: 0x%08lX\n", error);

    if (error & HAL_FLASH_ERROR_WRP)
        printf("  - Write protection error\n");
    if (error & HAL_FLASH_ERROR_PGS)
        printf("  - Programming sequence error\n");
    if (error & HAL_FLASH_ERROR_PGA)
        printf("  - Programming alignment error\n");
    if (error & HAL_FLASH_ERROR_WRP)
        printf("  - Write protection error\n");
}
```

### 2. CM4 부팅 실패

**해결**:
```c
void Debug_CM4_Boot(void)
{
    /* 부팅 주소 확인 */
    uint32_t boot_addr = READ_REG(SYSCFG->UR2);
    printf("CM4 Boot Address: 0x%08lX\n", boot_addr);

    /* 리셋 상태 확인 */
    if (__HAL_RCC_GET_FLAG(RCC_FLAG_D2RST))
    {
        printf("CM4 is in reset\n");
    }

    /* 벡터 테이블 확인 */
    uint32_t *vector_table = (uint32_t*)CM4_FLASH_START;
    printf("CM4 SP: 0x%08lX\n", vector_table[0]);
    printf("CM4 Reset Handler: 0x%08lX\n", vector_table[1]);
}
```

## 빌드 및 실행

```bash
# 순서대로 빌드
cd CM4 && make clean && make -j8
cd ../CM7 && make clean && make -j8

# 플래시
openocd -f board/stm32h745i-disco.cfg \
  -c "program CM7/build/CM7.elf verify reset exit"
```

## 참고 자료

- STM32H745/755 Reference Manual (RM0399) - Flash 챕터
- AN4852: STM32 마이크로컨트롤러의 Flash 메모리 프로그래밍
- AN5361: STM32H7 듀얼 코어 시작하기

## 라이선스

BSD 3-Clause License (STMicroelectronics)
