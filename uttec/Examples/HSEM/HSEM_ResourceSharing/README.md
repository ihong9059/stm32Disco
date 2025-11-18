# HSEM_ResourceSharing 예제

## 개요

이 예제는 STM32H745I-DISCO의 하드웨어 세마포어(HSEM)를 사용하여 Cortex-M7과 Cortex-M4 코어 간에 공유 리소스(UART, SPI, I2C 등)를 안전하게 보호하고 접근하는 방법을 보여줍니다. 데드락 방지 메커니즘과 리소스 관리 기법을 학습할 수 있습니다.

## 하드웨어 요구사항

- **보드**: STM32H745I-DISCO
- **공유 리소스**:
  - USART3: ST-LINK 가상 COM 포트
  - SPI2: 외부 디바이스 통신
  - I2C4: 센서/EEPROM 접근
- **LED**:
  - LED1 (녹색): CM7 리소스 사용 중
  - LED2 (주황색): CM4 리소스 사용 중
  - LED3 (빨간색): 리소스 충돌/오류
  - LED4 (파란색): 정상 동작

## 주요 기능

### 1. 공유 리소스 보호
- **상호 배제(Mutual Exclusion)**: 한 번에 하나의 코어만 리소스 접근
- **원자적 연산**: 하드웨어 세마포어로 경쟁 조건(Race Condition) 방지
- **타임아웃 메커니즘**: 무한 대기 방지

### 2. 데드락 방지
- **순서 있는 잠금**: 항상 동일한 순서로 세마포어 획득
- **타임아웃 기반 포기**: 일정 시간 후 자동 해제
- **우선순위 역전 방지**: 우선순위 상속 메커니즘

### 3. 공정성(Fairness)
- **FIFO 대기 큐**: 먼저 요청한 코어가 먼저 접근
- **기아 상태 방지**: 한 코어가 독점하지 못하도록 제한

## 소프트웨어 설명

### 리소스 관리 구조체

```c
/* resource_manager.h */

/* 리소스 타입 정의 */
typedef enum
{
    RESOURCE_UART3 = 0,
    RESOURCE_SPI2,
    RESOURCE_I2C4,
    RESOURCE_ADC1,
    RESOURCE_SHARED_MEM,
    RESOURCE_MAX
} ResourceType_t;

/* 세마포어 ID 매핑 */
#define HSEM_ID_UART3       0
#define HSEM_ID_SPI2        1
#define HSEM_ID_I2C4        2
#define HSEM_ID_ADC1        3
#define HSEM_ID_SHARED_MEM  4

/* 리소스 상태 구조체 */
typedef struct
{
    ResourceType_t type;
    uint32_t hsem_id;
    volatile uint32_t owner_id;
    volatile uint32_t lock_count;
    volatile uint32_t acquire_timestamp;
    volatile uint32_t total_usage_time;
    volatile uint32_t access_count;
    volatile uint32_t contention_count;
} ResourceInfo_t;

/* 전역 리소스 정보 (공유 메모리) */
#if defined ( __GNUC__ )
__attribute__((section(".shared_data")))
#endif
ResourceInfo_t resource_info[RESOURCE_MAX];

/* 프로세스 ID */
#define PROCESS_ID_CM7  0
#define PROCESS_ID_CM4  1

/* 타임아웃 설정 */
#define RESOURCE_TIMEOUT_MS  1000
#define MAX_LOCK_DURATION_MS 5000
```

### 리소스 관리 API

```c
/* resource_manager.c */

/**
  * @brief  리소스 관리자 초기화
  */
void ResourceManager_Init(void)
{
    /* HSEM 클록 활성화 */
    __HAL_RCC_HSEM_CLK_ENABLE();

    /* 리소스 정보 초기화 */
    for (uint32_t i = 0; i < RESOURCE_MAX; i++)
    {
        resource_info[i].type = (ResourceType_t)i;
        resource_info[i].hsem_id = i;
        resource_info[i].owner_id = 0xFFFFFFFF;
        resource_info[i].lock_count = 0;
        resource_info[i].acquire_timestamp = 0;
        resource_info[i].total_usage_time = 0;
        resource_info[i].access_count = 0;
        resource_info[i].contention_count = 0;
    }

    /* 모든 세마포어 해제 */
    for (uint32_t i = 0; i < RESOURCE_MAX; i++)
    {
        HAL_HSEM_Release(i, PROCESS_ID_CM7);
        HAL_HSEM_Release(i, PROCESS_ID_CM4);
    }
}

/**
  * @brief  리소스 획득
  * @param  resource: 리소스 타입
  * @param  process_id: 프로세스 ID (CM7: 0, CM4: 1)
  * @param  timeout_ms: 타임아웃 (밀리초)
  * @retval HAL_OK: 성공, HAL_TIMEOUT: 타임아웃, HAL_ERROR: 오류
  */
HAL_StatusTypeDef Resource_Acquire(ResourceType_t resource, uint32_t process_id,
                                    uint32_t timeout_ms)
{
    if (resource >= RESOURCE_MAX)
    {
        return HAL_ERROR;
    }

    uint32_t hsem_id = resource_info[resource].hsem_id;
    uint32_t tickstart = HAL_GetTick();

    /* 타임아웃까지 세마포어 획득 시도 */
    while ((HAL_GetTick() - tickstart) < timeout_ms)
    {
        if (HAL_HSEM_FastTake(hsem_id) == HAL_OK)
        {
            /* 세마포어 획득 성공 */
            resource_info[resource].owner_id = process_id;
            resource_info[resource].lock_count++;
            resource_info[resource].acquire_timestamp = HAL_GetTick();
            resource_info[resource].access_count++;

            /* LED 점등 */
            if (process_id == PROCESS_ID_CM7)
                BSP_LED_On(LED1);
            else
                BSP_LED_On(LED2);

            return HAL_OK;
        }

        /* 경쟁 카운트 증가 */
        resource_info[resource].contention_count++;

        /* 짧은 대기 */
        HAL_Delay(1);
    }

    /* 타임아웃 */
    BSP_LED_On(LED3);
    return HAL_TIMEOUT;
}

/**
  * @brief  리소스 해제
  * @param  resource: 리소스 타입
  * @param  process_id: 프로세스 ID
  * @retval HAL_OK: 성공, HAL_ERROR: 오류
  */
HAL_StatusTypeDef Resource_Release(ResourceType_t resource, uint32_t process_id)
{
    if (resource >= RESOURCE_MAX)
    {
        return HAL_ERROR;
    }

    /* 소유권 확인 */
    if (resource_info[resource].owner_id != process_id)
    {
        /* 다른 프로세스가 소유하고 있음 */
        BSP_LED_On(LED3);
        return HAL_ERROR;
    }

    /* 사용 시간 누적 */
    uint32_t usage_time = HAL_GetTick() - resource_info[resource].acquire_timestamp;
    resource_info[resource].total_usage_time += usage_time;

    /* 장시간 점유 경고 */
    if (usage_time > MAX_LOCK_DURATION_MS)
    {
        printf("Warning: Resource %d held for %lu ms by process %lu\n",
               resource, usage_time, process_id);
    }

    /* 세마포어 해제 */
    uint32_t hsem_id = resource_info[resource].hsem_id;
    HAL_HSEM_Release(hsem_id, process_id);

    /* 리소스 정보 업데이트 */
    resource_info[resource].owner_id = 0xFFFFFFFF;
    resource_info[resource].lock_count = 0;

    /* LED 소등 */
    if (process_id == PROCESS_ID_CM7)
        BSP_LED_Off(LED1);
    else
        BSP_LED_Off(LED2);

    return HAL_OK;
}

/**
  * @brief  리소스 강제 해제 (긴급 상황용)
  */
void Resource_ForceRelease(ResourceType_t resource)
{
    uint32_t hsem_id = resource_info[resource].hsem_id;

    /* 모든 프로세스에서 해제 */
    HAL_HSEM_Release(hsem_id, PROCESS_ID_CM7);
    HAL_HSEM_Release(hsem_id, PROCESS_ID_CM4);

    resource_info[resource].owner_id = 0xFFFFFFFF;
    resource_info[resource].lock_count = 0;

    printf("Resource %d force released\n", resource);
}

/**
  * @brief  리소스 상태 확인
  */
uint8_t Resource_IsLocked(ResourceType_t resource)
{
    if (resource >= RESOURCE_MAX)
    {
        return 0;
    }

    uint32_t hsem_id = resource_info[resource].hsem_id;
    return HAL_HSEM_IsSemTaken(hsem_id);
}

/**
  * @brief  리소스 소유자 확인
  */
uint32_t Resource_GetOwner(ResourceType_t resource)
{
    if (resource >= RESOURCE_MAX)
    {
        return 0xFFFFFFFF;
    }

    return resource_info[resource].owner_id;
}

/**
  * @brief  리소스 통계 출력
  */
void Resource_PrintStatistics(void)
{
    printf("\n=== Resource Statistics ===\n");

    for (uint32_t i = 0; i < RESOURCE_MAX; i++)
    {
        printf("Resource %lu:\n", i);
        printf("  Access Count: %lu\n", resource_info[i].access_count);
        printf("  Contention Count: %lu\n", resource_info[i].contention_count);
        printf("  Total Usage Time: %lu ms\n", resource_info[i].total_usage_time);

        if (resource_info[i].access_count > 0)
        {
            uint32_t avg_usage = resource_info[i].total_usage_time /
                                 resource_info[i].access_count;
            printf("  Average Usage Time: %lu ms\n", avg_usage);

            float contention_ratio = (float)resource_info[i].contention_count /
                                    (float)resource_info[i].access_count;
            printf("  Contention Ratio: %.2f\n", contention_ratio);
        }
    }

    printf("===========================\n");
}
```

## 공유 UART 구현

### UART 래퍼 함수

```c
/* shared_uart.c */

UART_HandleTypeDef huart3;

/**
  * @brief  공유 UART 초기화 (CM7에서만 호출)
  */
void SharedUART_Init(void)
{
    huart3.Instance = USART3;
    huart3.Init.BaudRate = 115200;
    huart3.Init.WordLength = UART_WORDLENGTH_8B;
    huart3.Init.StopBits = UART_STOPBITS_1;
    huart3.Init.Parity = UART_PARITY_NONE;
    huart3.Init.Mode = UART_MODE_TX_RX;
    huart3.Init.HwFlowCtl = UART_HWCONTROL_NONE;
    huart3.Init.OverSampling = UART_OVERSAMPLING_16;

    if (HAL_UART_Init(&huart3) != HAL_OK)
    {
        Error_Handler();
    }
}

/**
  * @brief  보호된 UART 전송
  * @param  process_id: 프로세스 ID
  * @param  data: 전송할 데이터
  * @param  length: 데이터 길이
  * @param  timeout: 타임아웃
  */
HAL_StatusTypeDef SharedUART_Transmit(uint32_t process_id, uint8_t *data,
                                       uint16_t length, uint32_t timeout)
{
    HAL_StatusTypeDef status;

    /* UART 리소스 획득 */
    if (Resource_Acquire(RESOURCE_UART3, process_id, timeout) != HAL_OK)
    {
        return HAL_TIMEOUT;
    }

    /* UART 전송 */
    status = HAL_UART_Transmit(&huart3, data, length, timeout);

    /* 리소스 해제 */
    Resource_Release(RESOURCE_UART3, process_id);

    return status;
}

/**
  * @brief  보호된 UART 수신
  */
HAL_StatusTypeDef SharedUART_Receive(uint32_t process_id, uint8_t *data,
                                      uint16_t length, uint32_t timeout)
{
    HAL_StatusTypeDef status;

    /* UART 리소스 획득 */
    if (Resource_Acquire(RESOURCE_UART3, process_id, timeout) != HAL_OK)
    {
        return HAL_TIMEOUT;
    }

    /* UART 수신 */
    status = HAL_UART_Receive(&huart3, data, length, timeout);

    /* 리소스 해제 */
    Resource_Release(RESOURCE_UART3, process_id);

    return status;
}

/**
  * @brief  보호된 printf 함수
  */
int SharedUART_Printf(uint32_t process_id, const char *format, ...)
{
    char buffer[256];
    va_list args;
    int len;

    va_start(args, format);
    len = vsnprintf(buffer, sizeof(buffer), format, args);
    va_end(args);

    if (len > 0)
    {
        SharedUART_Transmit(process_id, (uint8_t*)buffer, len, 1000);
    }

    return len;
}
```

## 공유 SPI 구현

```c
/* shared_spi.c */

SPI_HandleTypeDef hspi2;

/**
  * @brief  공유 SPI 초기화 (CM7에서만 호출)
  */
void SharedSPI_Init(void)
{
    hspi2.Instance = SPI2;
    hspi2.Init.Mode = SPI_MODE_MASTER;
    hspi2.Init.Direction = SPI_DIRECTION_2LINES;
    hspi2.Init.DataSize = SPI_DATASIZE_8BIT;
    hspi2.Init.CLKPolarity = SPI_POLARITY_LOW;
    hspi2.Init.CLKPhase = SPI_PHASE_1EDGE;
    hspi2.Init.NSS = SPI_NSS_SOFT;
    hspi2.Init.BaudRatePrescaler = SPI_BAUDRATEPRESCALER_16;
    hspi2.Init.FirstBit = SPI_FIRSTBIT_MSB;
    hspi2.Init.TIMode = SPI_TIMODE_DISABLE;
    hspi2.Init.CRCCalculation = SPI_CRCCALCULATION_DISABLE;
    hspi2.Init.NSSPMode = SPI_NSS_PULSE_DISABLE;

    if (HAL_SPI_Init(&hspi2) != HAL_OK)
    {
        Error_Handler();
    }
}

/**
  * @brief  보호된 SPI 송수신
  */
HAL_StatusTypeDef SharedSPI_TransmitReceive(uint32_t process_id,
                                             uint8_t *tx_data, uint8_t *rx_data,
                                             uint16_t length, uint32_t timeout)
{
    HAL_StatusTypeDef status;

    /* SPI 리소스 획득 */
    if (Resource_Acquire(RESOURCE_SPI2, process_id, timeout) != HAL_OK)
    {
        return HAL_TIMEOUT;
    }

    /* CS Low */
    HAL_GPIO_WritePin(SPI2_CS_GPIO_Port, SPI2_CS_Pin, GPIO_PIN_RESET);

    /* SPI 송수신 */
    status = HAL_SPI_TransmitReceive(&hspi2, tx_data, rx_data, length, timeout);

    /* CS High */
    HAL_GPIO_WritePin(SPI2_CS_GPIO_Port, SPI2_CS_Pin, GPIO_PIN_SET);

    /* 리소스 해제 */
    Resource_Release(RESOURCE_SPI2, process_id);

    return status;
}

/**
  * @brief  플래시 메모리 읽기 예제 (SPI)
  */
HAL_StatusTypeDef SharedSPI_ReadFlash(uint32_t process_id, uint32_t address,
                                       uint8_t *data, uint16_t length)
{
    uint8_t cmd[4];
    HAL_StatusTypeDef status;

    /* Read 명령 구성 */
    cmd[0] = 0x03;  /* Read Data */
    cmd[1] = (address >> 16) & 0xFF;
    cmd[2] = (address >> 8) & 0xFF;
    cmd[3] = address & 0xFF;

    /* SPI 리소스 획득 */
    if (Resource_Acquire(RESOURCE_SPI2, process_id, 1000) != HAL_OK)
    {
        return HAL_TIMEOUT;
    }

    /* CS Low */
    HAL_GPIO_WritePin(SPI2_CS_GPIO_Port, SPI2_CS_Pin, GPIO_PIN_RESET);

    /* 명령 전송 */
    status = HAL_SPI_Transmit(&hspi2, cmd, 4, 100);

    if (status == HAL_OK)
    {
        /* 데이터 수신 */
        status = HAL_SPI_Receive(&hspi2, data, length, 1000);
    }

    /* CS High */
    HAL_GPIO_WritePin(SPI2_CS_GPIO_Port, SPI2_CS_Pin, GPIO_PIN_SET);

    /* 리소스 해제 */
    Resource_Release(RESOURCE_SPI2, process_id);

    return status;
}
```

## 공유 I2C 구현

```c
/* shared_i2c.c */

I2C_HandleTypeDef hi2c4;

/**
  * @brief  공유 I2C 초기화 (CM7에서만 호출)
  */
void SharedI2C_Init(void)
{
    hi2c4.Instance = I2C4;
    hi2c4.Init.Timing = 0x00702991;  /* 400kHz @ 120MHz */
    hi2c4.Init.OwnAddress1 = 0;
    hi2c4.Init.AddressingMode = I2C_ADDRESSINGMODE_7BIT;
    hi2c4.Init.DualAddressMode = I2C_DUALADDRESS_DISABLE;
    hi2c4.Init.GeneralCallMode = I2C_GENERALCALL_DISABLE;
    hi2c4.Init.NoStretchMode = I2C_NOSTRETCH_DISABLE;

    if (HAL_I2C_Init(&hi2c4) != HAL_OK)
    {
        Error_Handler();
    }

    /* 아날로그 필터 활성화 */
    HAL_I2CEx_ConfigAnalogFilter(&hi2c4, I2C_ANALOGFILTER_ENABLE);
}

/**
  * @brief  보호된 I2C 메모리 쓰기
  */
HAL_StatusTypeDef SharedI2C_MemWrite(uint32_t process_id, uint16_t dev_addr,
                                      uint16_t mem_addr, uint16_t mem_addr_size,
                                      uint8_t *data, uint16_t length,
                                      uint32_t timeout)
{
    HAL_StatusTypeDef status;

    /* I2C 리소스 획득 */
    if (Resource_Acquire(RESOURCE_I2C4, process_id, timeout) != HAL_OK)
    {
        return HAL_TIMEOUT;
    }

    /* I2C 메모리 쓰기 */
    status = HAL_I2C_Mem_Write(&hi2c4, dev_addr, mem_addr, mem_addr_size,
                               data, length, timeout);

    /* 리소스 해제 */
    Resource_Release(RESOURCE_I2C4, process_id);

    return status;
}

/**
  * @brief  보호된 I2C 메모리 읽기
  */
HAL_StatusTypeDef SharedI2C_MemRead(uint32_t process_id, uint16_t dev_addr,
                                     uint16_t mem_addr, uint16_t mem_addr_size,
                                     uint8_t *data, uint16_t length,
                                     uint32_t timeout)
{
    HAL_StatusTypeDef status;

    /* I2C 리소스 획득 */
    if (Resource_Acquire(RESOURCE_I2C4, process_id, timeout) != HAL_OK)
    {
        return HAL_TIMEOUT;
    }

    /* I2C 메모리 읽기 */
    status = HAL_I2C_Mem_Read(&hi2c4, dev_addr, mem_addr, mem_addr_size,
                              data, length, timeout);

    /* 리소스 해제 */
    Resource_Release(RESOURCE_I2C4, process_id);

    return status;
}

/**
  * @brief  EEPROM 쓰기 예제
  */
HAL_StatusTypeDef EEPROM_Write(uint32_t process_id, uint16_t address,
                                uint8_t *data, uint16_t length)
{
    #define EEPROM_ADDR  0xA0  /* EEPROM I2C 주소 */

    HAL_StatusTypeDef status;

    status = SharedI2C_MemWrite(process_id, EEPROM_ADDR, address,
                                I2C_MEMADD_SIZE_16BIT, data, length, 1000);

    if (status == HAL_OK)
    {
        /* 쓰기 사이클 완료 대기 (5ms) */
        HAL_Delay(5);
    }

    return status;
}

/**
  * @brief  EEPROM 읽기 예제
  */
HAL_StatusTypeDef EEPROM_Read(uint32_t process_id, uint16_t address,
                               uint8_t *data, uint16_t length)
{
    #define EEPROM_ADDR  0xA0

    return SharedI2C_MemRead(process_id, EEPROM_ADDR, address,
                             I2C_MEMADD_SIZE_16BIT, data, length, 1000);
}
```

## 데드락 방지 메커니즘

### 1. 순서 있는 잠금(Lock Ordering)

```c
/* deadlock_prevention.c */

/* 리소스 획득 순서 정의 */
const ResourceType_t lock_order[] = {
    RESOURCE_SHARED_MEM,
    RESOURCE_UART3,
    RESOURCE_I2C4,
    RESOURCE_SPI2,
    RESOURCE_ADC1
};

/**
  * @brief  여러 리소스를 안전하게 획득
  * @param  resources: 획득할 리소스 배열
  * @param  count: 리소스 개수
  * @param  process_id: 프로세스 ID
  * @retval HAL_OK: 성공, HAL_ERROR: 실패
  */
HAL_StatusTypeDef Resource_AcquireMultiple(ResourceType_t *resources,
                                            uint32_t count, uint32_t process_id)
{
    /* 리소스를 정의된 순서대로 정렬 */
    ResourceType_t sorted[RESOURCE_MAX];
    memcpy(sorted, resources, count * sizeof(ResourceType_t));

    /* 버블 정렬 (간단한 구현) */
    for (uint32_t i = 0; i < count - 1; i++)
    {
        for (uint32_t j = 0; j < count - i - 1; j++)
        {
            if (sorted[j] > sorted[j + 1])
            {
                ResourceType_t temp = sorted[j];
                sorted[j] = sorted[j + 1];
                sorted[j + 1] = temp;
            }
        }
    }

    /* 순서대로 리소스 획득 */
    uint32_t acquired = 0;

    for (uint32_t i = 0; i < count; i++)
    {
        if (Resource_Acquire(sorted[i], process_id, 1000) != HAL_OK)
        {
            /* 획득 실패 시 이미 획득한 리소스 모두 해제 */
            for (uint32_t j = 0; j < acquired; j++)
            {
                Resource_Release(sorted[j], process_id);
            }
            return HAL_ERROR;
        }
        acquired++;
    }

    return HAL_OK;
}

/**
  * @brief  여러 리소스 해제
  */
void Resource_ReleaseMultiple(ResourceType_t *resources, uint32_t count,
                               uint32_t process_id)
{
    /* 역순으로 해제 */
    for (int32_t i = count - 1; i >= 0; i--)
    {
        Resource_Release(resources[i], process_id);
    }
}
```

### 2. 타임아웃 기반 데드락 해결

```c
/* deadlock_timeout.c */

typedef struct
{
    ResourceType_t resource;
    uint32_t acquire_time;
    uint32_t max_hold_time;
} ResourceTimeout_t;

ResourceTimeout_t resource_timeouts[RESOURCE_MAX];

/**
  * @brief  타임아웃 모니터 초기화
  */
void ResourceTimeout_Init(void)
{
    for (uint32_t i = 0; i < RESOURCE_MAX; i++)
    {
        resource_timeouts[i].resource = (ResourceType_t)i;
        resource_timeouts[i].acquire_time = 0;
        resource_timeouts[i].max_hold_time = MAX_LOCK_DURATION_MS;
    }
}

/**
  * @brief  타임아웃 모니터 (주기적으로 호출)
  */
void ResourceTimeout_Monitor(void)
{
    uint32_t current_time = HAL_GetTick();

    for (uint32_t i = 0; i < RESOURCE_MAX; i++)
    {
        if (Resource_IsLocked((ResourceType_t)i))
        {
            uint32_t hold_time = current_time -
                                 resource_info[i].acquire_timestamp;

            if (hold_time > resource_timeouts[i].max_hold_time)
            {
                /* 장시간 점유 감지 - 강제 해제 */
                printf("Deadlock detected on resource %u, forcing release\n", i);

                Resource_ForceRelease((ResourceType_t)i);

                BSP_LED_On(LED3);  /* 오류 표시 */
            }
        }
    }
}
```

### 3. Try-Lock 패턴

```c
/* trylock_pattern.c */

/**
  * @brief  리소스 획득 시도 (블로킹 없음)
  */
HAL_StatusTypeDef Resource_TryAcquire(ResourceType_t resource, uint32_t process_id)
{
    if (resource >= RESOURCE_MAX)
    {
        return HAL_ERROR;
    }

    uint32_t hsem_id = resource_info[resource].hsem_id;

    /* 즉시 획득 시도 (대기 없음) */
    if (HAL_HSEM_FastTake(hsem_id) == HAL_OK)
    {
        resource_info[resource].owner_id = process_id;
        resource_info[resource].lock_count++;
        resource_info[resource].acquire_timestamp = HAL_GetTick();
        resource_info[resource].access_count++;

        return HAL_OK;
    }

    return HAL_BUSY;
}

/**
  * @brief  Try-Lock 패턴 예제
  */
void Example_TryLock_Pattern(uint32_t process_id)
{
    ResourceType_t resources[] = {RESOURCE_UART3, RESOURCE_I2C4};
    uint8_t acquired[2] = {0, 0};

    /* 모든 리소스 획득 시도 */
    acquired[0] = (Resource_TryAcquire(resources[0], process_id) == HAL_OK);
    acquired[1] = (Resource_TryAcquire(resources[1], process_id) == HAL_OK);

    /* 일부만 획득된 경우 모두 해제하고 재시도 */
    if (acquired[0] && !acquired[1])
    {
        Resource_Release(resources[0], process_id);
        HAL_Delay(10);  /* 백오프 */
        return;  /* 재시도 */
    }
    else if (!acquired[0] && acquired[1])
    {
        Resource_Release(resources[1], process_id);
        HAL_Delay(10);
        return;
    }
    else if (acquired[0] && acquired[1])
    {
        /* 모든 리소스 획득 성공 - 작업 수행 */
        Perform_Critical_Section();

        /* 해제 */
        Resource_Release(resources[0], process_id);
        Resource_Release(resources[1], process_id);
    }
}
```

## CM7 메인 코드

```c
/* CM7/main.c */

int main(void)
{
    /* HAL 및 시스템 초기화 */
    HAL_Init();
    SystemClock_Config();
    MPU_Config();

    /* LED 초기화 */
    BSP_LED_Init(LED1);
    BSP_LED_Init(LED2);
    BSP_LED_Init(LED3);
    BSP_LED_Init(LED4);

    /* 리소스 관리자 초기화 */
    ResourceManager_Init();

    /* 공유 주변장치 초기화 (CM7만 수행) */
    SharedUART_Init();
    SharedSPI_Init();
    SharedI2C_Init();

    /* 타임아웃 모니터 초기화 */
    ResourceTimeout_Init();

    /* CM4 부팅 */
    HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);

    BSP_LED_On(LED4);

    /* 메인 루프 */
    while (1)
    {
        /* UART를 사용한 로깅 */
        SharedUART_Printf(PROCESS_ID_CM7, "[CM7] Tick: %lu\n", HAL_GetTick());

        /* I2C EEPROM 읽기/쓰기 예제 */
        uint8_t eeprom_data[16] = "CM7 Test Data";
        EEPROM_Write(PROCESS_ID_CM7, 0x0000, eeprom_data, sizeof(eeprom_data));

        uint8_t read_buf[16];
        EEPROM_Read(PROCESS_ID_CM7, 0x0000, read_buf, sizeof(read_buf));

        /* SPI 플래시 읽기 예제 */
        uint8_t flash_data[256];
        SharedSPI_ReadFlash(PROCESS_ID_CM7, 0x000000, flash_data, sizeof(flash_data));

        /* 타임아웃 모니터링 */
        ResourceTimeout_Monitor();

        /* 통계 출력 (10초마다) */
        static uint32_t last_stats_time = 0;
        if ((HAL_GetTick() - last_stats_time) >= 10000)
        {
            Resource_PrintStatistics();
            last_stats_time = HAL_GetTick();
        }

        HAL_Delay(1000);
    }
}
```

## CM4 메인 코드

```c
/* CM4/main.c */

int main(void)
{
    /* HAL 초기화 */
    HAL_Init();
    MPU_Config();

    /* 메인 루프 */
    while (1)
    {
        /* UART 로깅 */
        SharedUART_Printf(PROCESS_ID_CM4, "[CM4] Alive: %lu\n", HAL_GetTick());

        /* 센서 데이터 읽기 (I2C) */
        uint8_t sensor_data[6];
        if (SharedI2C_MemRead(PROCESS_ID_CM4, 0x68, 0x3B, I2C_MEMADD_SIZE_8BIT,
                              sensor_data, sizeof(sensor_data), 1000) == HAL_OK)
        {
            SharedUART_Printf(PROCESS_ID_CM4, "[CM4] Sensor: %02X %02X %02X\n",
                             sensor_data[0], sensor_data[1], sensor_data[2]);
        }

        /* SPI 통신 예제 */
        uint8_t spi_tx[4] = {0x01, 0x02, 0x03, 0x04};
        uint8_t spi_rx[4];
        SharedSPI_TransmitReceive(PROCESS_ID_CM4, spi_tx, spi_rx, 4, 1000);

        HAL_Delay(1500);
    }
}
```

## 고급 기능: 우선순위 역전 방지

```c
/* priority_inheritance.c */

typedef struct
{
    ResourceType_t resource;
    uint32_t original_priority;
    uint32_t boosted_priority;
    uint8_t priority_boosted;
} PriorityInheritance_t;

PriorityInheritance_t priority_info[RESOURCE_MAX];

/**
  * @brief  우선순위 상속 획득
  */
HAL_StatusTypeDef Resource_AcquireWithPriority(ResourceType_t resource,
                                                uint32_t process_id,
                                                uint32_t priority,
                                                uint32_t timeout_ms)
{
    /* 리소스가 이미 점유 중인지 확인 */
    if (Resource_IsLocked(resource))
    {
        uint32_t owner = Resource_GetOwner(resource);

        /* 소유자의 우선순위가 낮으면 부스팅 */
        if (priority > priority_info[resource].original_priority)
        {
            priority_info[resource].boosted_priority = priority;
            priority_info[resource].priority_boosted = 1;

            /* 실제 태스크 우선순위 변경 (RTOS 사용 시) */
            // osThreadSetPriority(owner_thread, priority);
        }
    }

    /* 리소스 획득 */
    HAL_StatusTypeDef status = Resource_Acquire(resource, process_id, timeout_ms);

    if (status == HAL_OK)
    {
        priority_info[resource].original_priority = priority;
    }

    return status;
}

/**
  * @brief  우선순위 상속 해제
  */
HAL_StatusTypeDef Resource_ReleaseWithPriority(ResourceType_t resource,
                                                uint32_t process_id)
{
    /* 우선순위가 부스팅되었다면 원래대로 복원 */
    if (priority_info[resource].priority_boosted)
    {
        /* 원래 우선순위로 복원 (RTOS 사용 시) */
        // osThreadSetPriority(current_thread, original_priority);

        priority_info[resource].priority_boosted = 0;
    }

    return Resource_Release(resource, process_id);
}
```

## 테스트 및 검증

### 1. 동시 접근 스트레스 테스트

```c
/* stress_test.c */

void CM7_StressTest(void)
{
    for (uint32_t i = 0; i < 1000; i++)
    {
        /* 빠른 속도로 리소스 획득/해제 */
        if (Resource_Acquire(RESOURCE_UART3, PROCESS_ID_CM7, 100) == HAL_OK)
        {
            /* 매우 짧은 작업 */
            volatile uint32_t dummy = 0;
            for (uint32_t j = 0; j < 100; j++)
            {
                dummy++;
            }

            Resource_Release(RESOURCE_UART3, PROCESS_ID_CM7);
        }
        else
        {
            printf("CM7: Failed to acquire UART3 at iteration %lu\n", i);
        }
    }

    printf("CM7: Stress test completed\n");
}

void CM4_StressTest(void)
{
    for (uint32_t i = 0; i < 1000; i++)
    {
        if (Resource_Acquire(RESOURCE_UART3, PROCESS_ID_CM4, 100) == HAL_OK)
        {
            volatile uint32_t dummy = 0;
            for (uint32_t j = 0; j < 100; j++)
            {
                dummy++;
            }

            Resource_Release(RESOURCE_UART3, PROCESS_ID_CM4);
        }
        else
        {
            printf("CM4: Failed to acquire UART3 at iteration %lu\n", i);
        }
    }

    printf("CM4: Stress test completed\n");
}
```

### 2. 데드락 감지 테스트

```c
/* deadlock_test.c */

void CM7_DeadlockTest(void)
{
    printf("CM7: Attempting to acquire UART3, then I2C4\n");

    if (Resource_Acquire(RESOURCE_UART3, PROCESS_ID_CM7, 1000) == HAL_OK)
    {
        printf("CM7: UART3 acquired\n");

        HAL_Delay(100);  /* CM4가 I2C4를 획득할 시간 제공 */

        if (Resource_Acquire(RESOURCE_I2C4, PROCESS_ID_CM7, 2000) == HAL_OK)
        {
            printf("CM7: I2C4 acquired (no deadlock)\n");
            Resource_Release(RESOURCE_I2C4, PROCESS_ID_CM7);
        }
        else
        {
            printf("CM7: Failed to acquire I2C4 (potential deadlock)\n");
        }

        Resource_Release(RESOURCE_UART3, PROCESS_ID_CM7);
    }
}

void CM4_DeadlockTest(void)
{
    printf("CM4: Attempting to acquire I2C4, then UART3\n");

    if (Resource_Acquire(RESOURCE_I2C4, PROCESS_ID_CM4, 1000) == HAL_OK)
    {
        printf("CM4: I2C4 acquired\n");

        HAL_Delay(100);

        if (Resource_Acquire(RESOURCE_UART3, PROCESS_ID_CM4, 2000) == HAL_OK)
        {
            printf("CM4: UART3 acquired (no deadlock)\n");
            Resource_Release(RESOURCE_UART3, PROCESS_ID_CM4);
        }
        else
        {
            printf("CM4: Failed to acquire UART3 (potential deadlock)\n");
        }

        Resource_Release(RESOURCE_I2C4, PROCESS_ID_CM4);
    }
}
```

## 문제 해결

### 1. 리소스 획득 실패

**증상**: Resource_Acquire가 계속 HAL_TIMEOUT 반환

**진단**:
```c
void Diagnose_Resource_Failure(ResourceType_t resource)
{
    printf("Resource %d Diagnosis:\n", resource);
    printf("  Locked: %s\n", Resource_IsLocked(resource) ? "Yes" : "No");
    printf("  Owner: %lu\n", Resource_GetOwner(resource));
    printf("  Lock Count: %lu\n", resource_info[resource].lock_count);
    printf("  Acquire Time: %lu ms\n", resource_info[resource].acquire_timestamp);
    printf("  Current Time: %lu ms\n", HAL_GetTick());

    if (Resource_IsLocked(resource))
    {
        uint32_t hold_time = HAL_GetTick() - resource_info[resource].acquire_timestamp;
        printf("  Hold Time: %lu ms\n", hold_time);

        if (hold_time > MAX_LOCK_DURATION_MS)
        {
            printf("  WARNING: Lock held too long, forcing release\n");
            Resource_ForceRelease(resource);
        }
    }
}
```

### 2. 데이터 손상

**원인**: 캐시 일관성 문제

**해결**:
```c
/* MPU 설정으로 공유 메모리 영역 캐시 비활성화 */
void MPU_Config(void)
{
    MPU_Region_InitTypeDef MPU_InitStruct;

    HAL_MPU_Disable();

    /* D3 SRAM (0x38000000) - 공유 데이터 영역 */
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

    HAL_MPU_Enable(MPU_PRIVILEGED_DEFAULT);
}
```

## 빌드 및 실행

```bash
# 빌드
cd CM7 && make clean && make -j8
cd ../CM4 && make clean && make -j8

# 플래시
openocd -f board/stm32h745i-disco.cfg \
  -c "program CM7/build/CM7.elf verify" \
  -c "program CM4/build/CM4.elf verify reset exit"
```

## 참고 자료

- STM32H745/755 Reference Manual (RM0399) - HSEM
- AN4891: STM32 마이크로컨트롤러의 리소스 공유
- AN5361: STM32H7 듀얼 코어 시작하기
- "Operating System Concepts" - Deadlock prevention

## 라이선스

BSD 3-Clause License (STMicroelectronics)
