# HSEM_CoreSync 예제

## 개요

이 예제는 STM32H745I-DISCO 보드의 하드웨어 세마포어(HSEM)를 사용하여 Cortex-M7과 Cortex-M4 코어 간의 동기화를 구현하는 방법을 보여줍니다. 듀얼 코어 시스템에서 부팅 시퀀스 조정과 코어 간 동기화가 어떻게 이루어지는지 학습할 수 있습니다.

## 하드웨어 요구사항

- **보드**: STM32H745I-DISCO
- **LED**:
  - LED1 (녹색): CM7 동기화 완료 표시
  - LED2 (주황색): CM4 동기화 완료 표시
  - LED3 (빨간색): 동기화 오류 표시
  - LED4 (파란색): 전체 시스템 준비 완료

## 주요 기능

### 1. 하드웨어 세마포어(HSEM)
- **32개의 세마포어**: STM32H7 시리즈는 32개의 독립적인 하드웨어 세마포어 제공
- **원자적 연산**: 하드웨어 레벨에서 락 획득/해제 보장
- **듀얼 코어 동기화**: 두 코어가 안전하게 리소스를 공유하고 동기화

### 2. 부팅 시퀀스 조정
- CM7이 먼저 부팅하여 시스템 초기화 수행
- CM4는 CM7의 준비 신호를 대기
- 순차적이고 안전한 부팅 프로세스

## 소프트웨어 설명

### 세마포어 ID 정의

```c
/* 세마포어 ID 정의 */
#define HSEM_ID_0     0U  /* CM7 부팅 완료 신호 */
#define HSEM_ID_1     1U  /* CM4 부팅 완료 신호 */
#define HSEM_ID_2     2U  /* 동기화 포인트 1 */
#define HSEM_ID_3     3U  /* 동기화 포인트 2 */

/* 프로세스 ID (코어 식별) */
#define HSEM_PROCESSID_CM7  0U
#define HSEM_PROCESSID_CM4  1U
```

### CM7 코어 구현

```c
/* CM7/main.c */

/* 변수 선언 */
volatile uint32_t cm7_ready = 0;
volatile uint32_t cm4_ready = 0;

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

    /* CM7이 부팅 완료되었음을 표시 */
    BSP_LED_On(LED1);

    /* HSEM 클록 활성화 */
    __HAL_RCC_HSEM_CLK_ENABLE();

    /* 세마포어 0을 사용하여 CM7 준비 완료 신호 */
    if (HAL_HSEM_FastTake(HSEM_ID_0) == HAL_OK)
    {
        cm7_ready = 1;

        /* CM4 활성화 */
        HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);
    }
    else
    {
        /* 세마포어 획득 실패 */
        BSP_LED_On(LED3);
        Error_Handler();
    }

    /* CM4가 준비될 때까지 대기 */
    while (HAL_HSEM_IsSemTaken(HSEM_ID_1) == 0)
    {
        HAL_Delay(10);
    }

    cm4_ready = 1;
    BSP_LED_On(LED2);

    /* 동기화 포인트 1: 두 코어가 준비됨 */
    Sync_Point_1();

    /* 메인 루프 */
    while (1)
    {
        /* 주기적인 동기화 작업 */
        if (Perform_Sync_Operation() == HAL_OK)
        {
            BSP_LED_Toggle(LED4);
            HAL_Delay(500);
        }
    }
}

/**
  * @brief  첫 번째 동기화 포인트
  * @retval None
  */
void Sync_Point_1(void)
{
    /* 세마포어 2 획득 시도 (타임아웃: 1초) */
    uint32_t timeout = HAL_GetTick() + 1000;

    while (HAL_GetTick() < timeout)
    {
        if (HAL_HSEM_FastTake(HSEM_ID_2) == HAL_OK)
        {
            /* CM7이 동기화 포인트 통과 */
            BSP_LED_Toggle(LED1);

            /* 작업 수행 */
            HAL_Delay(100);

            /* 세마포어 해제 */
            HAL_HSEM_Release(HSEM_ID_2, HSEM_PROCESSID_CM7);
            return;
        }
        HAL_Delay(1);
    }

    /* 타임아웃 오류 */
    BSP_LED_On(LED3);
    Error_Handler();
}

/**
  * @brief  동기화된 작업 수행
  * @retval HAL status
  */
HAL_StatusTypeDef Perform_Sync_Operation(void)
{
    /* 세마포어 3 획득 */
    if (HAL_HSEM_Take(HSEM_ID_3, HSEM_PROCESSID_CM7) != HAL_OK)
    {
        return HAL_ERROR;
    }

    /* 임계 영역 - 공유 리소스 접근 */
    /* 예: 공유 메모리 업데이트 */
    shared_data.counter_cm7++;
    shared_data.timestamp = HAL_GetTick();

    /* 세마포어 해제 */
    HAL_HSEM_Release(HSEM_ID_3, HSEM_PROCESSID_CM7);

    return HAL_OK;
}

/**
  * @brief  시스템 클록 설정
  *         CM7: 480MHz, CM4: 240MHz
  */
void SystemClock_Config(void)
{
    RCC_ClkInitTypeDef RCC_ClkInitStruct;
    RCC_OscInitTypeDef RCC_OscInitStruct;

    /* 전압 스케일 1로 설정 (최대 성능) */
    __HAL_PWR_VOLTAGESCALING_CONFIG(PWR_REGULATOR_VOLTAGE_SCALE1);

    while (!__HAL_PWR_GET_FLAG(PWR_FLAG_VOSRDY)) {}

    /* HSE 및 PLL 설정 */
    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
    RCC_OscInitStruct.HSEState = RCC_HSE_ON;
    RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
    RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
    RCC_OscInitStruct.PLL.PLLM = 5;
    RCC_OscInitStruct.PLL.PLLN = 192;
    RCC_OscInitStruct.PLL.PLLP = 2;
    RCC_OscInitStruct.PLL.PLLQ = 4;
    RCC_OscInitStruct.PLL.PLLR = 2;
    RCC_OscInitStruct.PLL.PLLRGE = RCC_PLL1VCIRANGE_2;
    RCC_OscInitStruct.PLL.PLLVCOSEL = RCC_PLL1VCOWIDE;
    RCC_OscInitStruct.PLL.PLLFRACN = 0;

    if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
    {
        Error_Handler();
    }

    /* 시스템 클록, AHB, APB 설정 */
    RCC_ClkInitStruct.ClockType = (RCC_CLOCKTYPE_SYSCLK | RCC_CLOCKTYPE_HCLK |
                                    RCC_CLOCKTYPE_D1PCLK1 | RCC_CLOCKTYPE_PCLK1 |
                                    RCC_CLOCKTYPE_PCLK2  | RCC_CLOCKTYPE_D3PCLK1);
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
}
```

### CM4 코어 구현

```c
/* CM4/main.c */

volatile uint32_t cm4_initialized = 0;

int main(void)
{
    /* HAL 초기화 */
    HAL_Init();

    /* HSEM 클록 활성화 */
    __HAL_RCC_HSEM_CLK_ENABLE();

    /* CM7이 준비될 때까지 대기 */
    while (HAL_HSEM_IsSemTaken(HSEM_ID_0) == 0)
    {
        HAL_Delay(10);
    }

    /* CM4 준비 완료 신호 */
    if (HAL_HSEM_FastTake(HSEM_ID_1) == HAL_OK)
    {
        cm4_initialized = 1;
        BSP_LED_On(LED2);
    }
    else
    {
        BSP_LED_On(LED3);
        Error_Handler();
    }

    /* 동기화 포인트 1 대기 */
    Sync_Point_1();

    /* 메인 루프 */
    while (1)
    {
        /* CM4의 동기화된 작업 */
        if (Perform_Sync_Operation() == HAL_OK)
        {
            BSP_LED_Toggle(LED2);
            HAL_Delay(500);
        }
    }
}

/**
  * @brief  CM4 첫 번째 동기화 포인트
  */
void Sync_Point_1(void)
{
    uint32_t timeout = HAL_GetTick() + 1000;

    while (HAL_GetTick() < timeout)
    {
        if (HAL_HSEM_FastTake(HSEM_ID_2) == HAL_OK)
        {
            BSP_LED_Toggle(LED2);
            HAL_Delay(100);
            HAL_HSEM_Release(HSEM_ID_2, HSEM_PROCESSID_CM4);
            return;
        }
        HAL_Delay(1);
    }

    BSP_LED_On(LED3);
    Error_Handler();
}

/**
  * @brief  CM4 동기화된 작업
  */
HAL_StatusTypeDef Perform_Sync_Operation(void)
{
    if (HAL_HSEM_Take(HSEM_ID_3, HSEM_PROCESSID_CM4) != HAL_OK)
    {
        return HAL_ERROR;
    }

    /* 공유 리소스 접근 */
    shared_data.counter_cm4++;
    shared_data.last_core = CORE_CM4;

    HAL_HSEM_Release(HSEM_ID_3, HSEM_PROCESSID_CM4);

    return HAL_OK;
}
```

### 공유 메모리 구조

```c
/* common.h - 두 코어에서 공유 */

/* 공유 메모리 영역 정의 (D3 SRAM) */
#define SHARED_MEM_BASE    0x38000000

/* 공유 데이터 구조체 */
typedef struct
{
    volatile uint32_t counter_cm7;
    volatile uint32_t counter_cm4;
    volatile uint32_t timestamp;
    volatile uint32_t last_core;
    volatile uint32_t sync_flag;
} SharedData_t;

/* 공유 메모리에 배치 */
#if defined ( __ICCARM__ ) /* IAR */
#pragma location = SHARED_MEM_BASE
__no_init SharedData_t shared_data;

#elif defined ( __CC_ARM ) /* Keil */
SharedData_t shared_data __attribute__((at(SHARED_MEM_BASE)));

#elif defined ( __GNUC__ ) /* GCC */
SharedData_t shared_data __attribute__((section(".shared_data")));
#endif

/* 코어 식별자 */
#define CORE_CM7    0x7
#define CORE_CM4    0x4
```

## 링커 스크립트 설정

### CM7 링커 스크립트 (STM32H745XIHX_CM7.ld)

```ld
/* 공유 메모리 영역 정의 */
MEMORY
{
    FLASH (rx)      : ORIGIN = 0x08000000, LENGTH = 1024K
    DTCMRAM (xrw)   : ORIGIN = 0x20000000, LENGTH = 128K
    RAM_D1 (xrw)    : ORIGIN = 0x24000000, LENGTH = 512K
    RAM_D2 (xrw)    : ORIGIN = 0x30000000, LENGTH = 288K
    RAM_D3 (xrw)    : ORIGIN = 0x38000000, LENGTH = 64K  /* 공유 메모리 */
}

SECTIONS
{
    /* 공유 데이터 섹션 */
    .shared_data (NOLOAD) :
    {
        . = ALIGN(4);
        *(.shared_data)
        . = ALIGN(4);
    } > RAM_D3
}
```

### CM4 링커 스크립트 (STM32H745XIHX_CM4.ld)

```ld
MEMORY
{
    FLASH (rx)      : ORIGIN = 0x08100000, LENGTH = 1024K
    RAM (xrw)       : ORIGIN = 0x10000000, LENGTH = 288K
    RAM_D3 (xrw)    : ORIGIN = 0x38000000, LENGTH = 64K  /* 공유 메모리 */
}

SECTIONS
{
    .shared_data (NOLOAD) :
    {
        . = ALIGN(4);
        *(.shared_data)
        . = ALIGN(4);
    } > RAM_D3
}
```

## 빌드 및 실행

### 1. 프로젝트 빌드

```bash
# CM7 프로젝트 빌드
cd CM7
make clean
make -j8

# CM4 프로젝트 빌드
cd ../CM4
make clean
make -j8
```

### 2. 플래시 프로그래밍

```bash
# OpenOCD 사용
openocd -f board/stm32h745i-disco.cfg -c "program CM7/build/CM7.elf verify reset" -c "program CM4/build/CM4.elf verify reset exit"

# 또는 STM32CubeProgrammer 사용
STM32_Programmer_CLI -c port=SWD -w CM7/build/CM7.elf 0x08000000
STM32_Programmer_CLI -c port=SWD -w CM4/build/CM4.elf 0x08100000
```

### 3. 실행 순서

1. 보드에 전원 공급
2. CM7이 먼저 부팅하여 LED1 점등
3. CM7이 CM4를 활성화
4. CM4가 초기화되어 LED2 점등
5. 두 코어가 동기화 완료 후 LED4 점등
6. LED1과 LED2가 교대로 토글 (동기화된 작업 수행 중)

## 동작 시퀀스

```
시간 →

CM7: [부팅] → [HSEM_0 획득] → [CM4 활성화] → [HSEM_1 대기] → [동기화] → [주기적 작업]
         ↓                                      ↑                ↓
      LED1 ON                                  LED2 ON        LED4 토글

CM4:          [대기] → [부팅] → [HSEM_0 확인] → [HSEM_1 획득] → [동기화] → [주기적 작업]
                                                    ↓                ↓
                                                 LED2 ON        LED2 토글
```

## LED 상태 표시

| LED | 색상 | 상태 | 의미 |
|-----|------|------|------|
| LED1 | 녹색 | ON | CM7 부팅 완료 |
| LED1 | 녹색 | 토글 | CM7 동기화 작업 수행 중 |
| LED2 | 주황색 | ON | CM4 부팅 완료 |
| LED2 | 주황색 | 토글 | CM4 동기화 작업 수행 중 |
| LED3 | 빨간색 | ON | 동기화 오류 발생 |
| LED4 | 파란색 | 토글 | 전체 시스템 정상 동작 |

## 문제 해결

### 1. LED3(빨간색)이 켜지는 경우

**원인**: 세마포어 획득 실패 또는 타임아웃

**해결 방법**:
```c
/* 디버그 정보 추가 */
if (HAL_HSEM_FastTake(HSEM_ID_0) != HAL_OK)
{
    /* 세마포어 상태 확인 */
    uint32_t sem_status = HAL_HSEM_IsSemTaken(HSEM_ID_0);
    printf("HSEM_0 Status: %lu\n", sem_status);

    /* 강제 해제 시도 (주의: 데이터 손상 가능) */
    HAL_HSEM_Release(HSEM_ID_0, 0);
    HAL_HSEM_Release(HSEM_ID_0, 1);
}
```

### 2. CM4가 부팅하지 않는 경우

**원인**: CM4 부팅 주소가 잘못 설정됨

**해결 방법**:
```c
/* CM7에서 CM4 부팅 주소 확인 및 설정 */
#define CM4_BOOT_ADDRESS  0x08100000

void Enable_CM4_Boot(void)
{
    /* CM4 부팅 주소 설정 */
    HAL_SYSCFG_CM4SetBootAddress0(CM4_BOOT_ADDRESS);

    /* CM4 리셋 해제 및 활성화 */
    HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);
}
```

### 3. 공유 메모리 데이터 손상

**원인**: 캐시 일관성 문제

**해결 방법**:
```c
/* 공유 메모리 영역을 캐시 불가능 영역으로 설정 */
void MPU_Config(void)
{
    MPU_Region_InitTypeDef MPU_InitStruct;

    /* MPU 비활성화 */
    HAL_MPU_Disable();

    /* D3 SRAM을 캐시 불가능 영역으로 설정 */
    MPU_InitStruct.Enable = MPU_REGION_ENABLE;
    MPU_InitStruct.Number = MPU_REGION_NUMBER0;
    MPU_InitStruct.BaseAddress = 0x38000000;
    MPU_InitStruct.Size = MPU_REGION_SIZE_64KB;
    MPU_InitStruct.SubRegionDisable = 0x0;
    MPU_InitStruct.TypeExtField = MPU_TEX_LEVEL0;
    MPU_InitStruct.AccessPermission = MPU_REGION_FULL_ACCESS;
    MPU_InitStruct.DisableExec = MPU_INSTRUCTION_ACCESS_DISABLE;
    MPU_InitStruct.IsShareable = MPU_ACCESS_SHAREABLE;
    MPU_InitStruct.IsCacheable = MPU_ACCESS_NOT_CACHEABLE;
    MPU_InitStruct.IsBufferable = MPU_ACCESS_NOT_BUFFERABLE;

    HAL_MPU_ConfigRegion(&MPU_InitStruct);

    /* MPU 활성화 */
    HAL_MPU_Enable(MPU_PRIVILEGED_DEFAULT);
}
```

### 4. 데드락 발생

**원인**: 잘못된 세마포어 획득 순서

**해결 방법**:
```c
/* 타임아웃을 사용한 세마포어 획득 */
HAL_StatusTypeDef Safe_HSEM_Take(uint32_t SemID, uint32_t ProcessID, uint32_t timeout_ms)
{
    uint32_t tickstart = HAL_GetTick();

    while ((HAL_GetTick() - tickstart) < timeout_ms)
    {
        if (HAL_HSEM_Take(SemID, ProcessID) == HAL_OK)
        {
            return HAL_OK;
        }
        HAL_Delay(1);
    }

    return HAL_TIMEOUT;
}
```

## 실용 응용 사례

### 1. 듀얼 코어 부팅 관리자

```c
/* boot_manager.h */

typedef enum
{
    BOOT_STAGE_INIT = 0,
    BOOT_STAGE_CM7_READY,
    BOOT_STAGE_CM4_READY,
    BOOT_STAGE_SYNC_COMPLETE,
    BOOT_STAGE_RUNNING
} BootStage_t;

typedef struct
{
    volatile BootStage_t stage;
    volatile uint32_t cm7_version;
    volatile uint32_t cm4_version;
    volatile uint32_t error_code;
} BootManager_t;

/* boot_manager.c */
HAL_StatusTypeDef BootManager_Init(BootManager_t *manager)
{
    manager->stage = BOOT_STAGE_INIT;
    manager->error_code = 0;

    /* HSEM 초기화 */
    __HAL_RCC_HSEM_CLK_ENABLE();

    return HAL_OK;
}

HAL_StatusTypeDef BootManager_CM7_Ready(BootManager_t *manager)
{
    if (HAL_HSEM_FastTake(HSEM_ID_0) != HAL_OK)
    {
        manager->error_code = 0x01;
        return HAL_ERROR;
    }

    manager->cm7_version = FIRMWARE_VERSION_CM7;
    manager->stage = BOOT_STAGE_CM7_READY;

    return HAL_OK;
}

HAL_StatusTypeDef BootManager_WaitCM4(BootManager_t *manager, uint32_t timeout)
{
    uint32_t tickstart = HAL_GetTick();

    while ((HAL_GetTick() - tickstart) < timeout)
    {
        if (HAL_HSEM_IsSemTaken(HSEM_ID_1))
        {
            manager->stage = BOOT_STAGE_CM4_READY;
            return HAL_OK;
        }
        HAL_Delay(10);
    }

    manager->error_code = 0x02;
    return HAL_TIMEOUT;
}
```

### 2. 동기화된 센서 데이터 수집

```c
/* sensor_sync.c */

typedef struct
{
    volatile uint32_t timestamp;
    volatile float temperature;
    volatile float humidity;
    volatile float pressure;
    volatile uint8_t data_ready;
} SensorData_t;

/* CM7: 온도/습도 센서 읽기 */
void CM7_ReadTempHumidity(void)
{
    if (HAL_HSEM_Take(HSEM_ID_SENSOR, HSEM_PROCESSID_CM7) == HAL_OK)
    {
        sensor_data.temperature = Read_Temperature_Sensor();
        sensor_data.humidity = Read_Humidity_Sensor();
        sensor_data.timestamp = HAL_GetTick();
        sensor_data.data_ready |= 0x01;

        HAL_HSEM_Release(HSEM_ID_SENSOR, HSEM_PROCESSID_CM7);
    }
}

/* CM4: 압력 센서 읽기 */
void CM4_ReadPressure(void)
{
    if (HAL_HSEM_Take(HSEM_ID_SENSOR, HSEM_PROCESSID_CM4) == HAL_OK)
    {
        sensor_data.pressure = Read_Pressure_Sensor();
        sensor_data.data_ready |= 0x02;

        /* 모든 센서 데이터가 준비되면 처리 */
        if (sensor_data.data_ready == 0x03)
        {
            Process_Sensor_Data(&sensor_data);
            sensor_data.data_ready = 0;
        }

        HAL_HSEM_Release(HSEM_ID_SENSOR, HSEM_PROCESSID_CM4);
    }
}
```

## 성능 최적화

### 1. 세마포어 획득 대기 시간 최소화

```c
/* 스핀락 방식 (짧은 대기 시간용) */
static inline HAL_StatusTypeDef HSEM_FastTake_Spinlock(uint32_t SemID, uint32_t ProcessID, uint32_t max_retries)
{
    for (uint32_t i = 0; i < max_retries; i++)
    {
        if (HAL_HSEM_FastTake(SemID) == HAL_OK)
        {
            return HAL_OK;
        }
        __NOP(); /* CPU 사이클 낭비 방지 */
    }
    return HAL_BUSY;
}

/* 이벤트 대기 방식 (긴 대기 시간용) */
HAL_StatusTypeDef HSEM_Take_WithEvent(uint32_t SemID, uint32_t ProcessID)
{
    /* 인터럽트 활성화 */
    HAL_HSEM_ActivateNotification(__HAL_HSEM_SEMID_TO_MASK(SemID));

    /* WFI로 대기 (전력 절약) */
    __WFI();

    /* 세마포어 획득 시도 */
    return HAL_HSEM_Take(SemID, ProcessID);
}
```

### 2. 우선순위 기반 세마포어 관리

```c
typedef enum
{
    HSEM_PRIORITY_LOW = 0,
    HSEM_PRIORITY_NORMAL,
    HSEM_PRIORITY_HIGH,
    HSEM_PRIORITY_CRITICAL
} HSEM_Priority_t;

HAL_StatusTypeDef HSEM_Take_WithPriority(uint32_t SemID, uint32_t ProcessID,
                                           HSEM_Priority_t priority, uint32_t timeout)
{
    uint32_t retry_interval;

    /* 우선순위에 따른 재시도 간격 설정 */
    switch (priority)
    {
        case HSEM_PRIORITY_CRITICAL:
            retry_interval = 0;  /* 스핀락 */
            break;
        case HSEM_PRIORITY_HIGH:
            retry_interval = 1;
            break;
        case HSEM_PRIORITY_NORMAL:
            retry_interval = 10;
            break;
        case HSEM_PRIORITY_LOW:
            retry_interval = 100;
            break;
    }

    uint32_t tickstart = HAL_GetTick();

    while ((HAL_GetTick() - tickstart) < timeout)
    {
        if (HAL_HSEM_Take(SemID, ProcessID) == HAL_OK)
        {
            return HAL_OK;
        }

        if (retry_interval > 0)
        {
            HAL_Delay(retry_interval);
        }
    }

    return HAL_TIMEOUT;
}
```

## 디버깅 도구

### 1. HSEM 상태 모니터

```c
void HSEM_Print_Status(void)
{
    printf("\n=== HSEM Status ===\n");

    for (uint32_t i = 0; i < 32; i++)
    {
        if (HAL_HSEM_IsSemTaken(i))
        {
            /* 세마포어가 획득된 경우 프로세스 ID 확인 */
            uint32_t processid = (HSEM->R[i] & HSEM_R_PROCID) >> HSEM_R_PROCID_Pos;
            uint32_t coreid = (HSEM->R[i] & HSEM_R_COREID) >> HSEM_R_COREID_Pos;

            printf("HSEM[%lu]: TAKEN by ProcessID=%lu, CoreID=%lu\n",
                   i, processid, coreid);
        }
        else
        {
            printf("HSEM[%lu]: FREE\n", i);
        }
    }

    printf("==================\n");
}
```

### 2. 동기화 타이밍 분석

```c
typedef struct
{
    uint32_t take_timestamp;
    uint32_t release_timestamp;
    uint32_t hold_time;
    uint32_t wait_time;
} HSEM_Timing_t;

HSEM_Timing_t timing_stats[32];

HAL_StatusTypeDef HSEM_Take_WithTiming(uint32_t SemID, uint32_t ProcessID)
{
    uint32_t start_time = HAL_GetTick();
    HAL_StatusTypeDef status = HAL_HSEM_Take(SemID, ProcessID);

    if (status == HAL_OK)
    {
        timing_stats[SemID].take_timestamp = HAL_GetTick();
        timing_stats[SemID].wait_time = timing_stats[SemID].take_timestamp - start_time;
    }

    return status;
}

void HSEM_Release_WithTiming(uint32_t SemID, uint32_t ProcessID)
{
    timing_stats[SemID].release_timestamp = HAL_GetTick();
    timing_stats[SemID].hold_time = timing_stats[SemID].release_timestamp -
                                     timing_stats[SemID].take_timestamp;

    HAL_HSEM_Release(SemID, ProcessID);
}

void Print_Timing_Stats(void)
{
    printf("\n=== HSEM Timing Statistics ===\n");

    for (uint32_t i = 0; i < 32; i++)
    {
        if (timing_stats[i].take_timestamp > 0)
        {
            printf("HSEM[%lu]: Wait=%lu ms, Hold=%lu ms\n",
                   i, timing_stats[i].wait_time, timing_stats[i].hold_time);
        }
    }
}
```

## 참고 자료

- **STM32H745/755 Reference Manual (RM0399)**: HSEM 레지스터 상세 설명
- **STM32H745I-DISCO User Manual (UM2488)**: 보드 하드웨어 정보
- **AN5361**: STM32H7 듀얼 코어 마이크로컨트롤러 시작하기
- **AN5557**: STM32H7x5/7x7 듀얼 코어 통신

## 라이선스

이 예제 코드는 STMicroelectronics의 BSD 3-Clause 라이선스에 따라 배포됩니다.
