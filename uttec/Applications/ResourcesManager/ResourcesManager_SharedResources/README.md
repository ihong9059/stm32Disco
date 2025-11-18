# ResourcesManager_SharedResources - 공유 주변장치 관리

## 개요

이 애플리케이션은 STM32H745의 듀얼 코어 환경에서 주변장치를 안전하게 공유하는 방법을 보여줍니다. HSEM (Hardware Semaphore)을 사용하여 SPI, I2C, UART 등의 공유 주변장치에 대한 배타적 접근을 제어합니다.

## 공유 리소스 관리의 필요성

듀얼 코어 시스템에서 주변장치 공유는 다음 이유로 필요합니다:

- **제한된 주변장치**: SPI, I2C 버스는 물리적으로 하나
- **하드웨어 경합 방지**: 동시 접근 시 데이터 손상
- **효율적 활용**: 유휴 리소스 활용 극대화

## 시스템 아키텍처

### 공유 리소스 관리 구조
```
┌──────────────────┐              ┌──────────────────┐
│   Cortex-M7      │              │   Cortex-M4      │
│                  │              │                  │
│  ┌────────────┐  │              │  ┌────────────┐  │
│  │ Application │  │              │  │ Application │  │
│  └──────┬─────┘  │              │  └──────┬─────┘  │
│         │        │              │         │        │
│  ┌──────▼─────┐  │              │  ┌──────▼─────┐  │
│  │ Resource   │  │              │  │ Resource   │  │
│  │ Manager    │  │              │  │ Manager    │  │
│  └──────┬─────┘  │              │  └──────┬─────┘  │
│         │        │              │         │        │
└─────────┼────────┘              └─────────┼────────┘
          │                                 │
          │         ┌─────────┐             │
          └────────►│  HSEM   │◄────────────┘
                    │ (Lock)  │
                    └────┬────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   ┌────▼────┐    ┌─────▼─────┐    ┌─────▼─────┐
   │  SPI1   │    │   I2C1    │    │   UART3   │
   └─────────┘    └───────────┘    └───────────┘
```

## HSEM ID 할당

### 리소스별 HSEM 할당
```c
// 주변장치 HSEM ID
#define HSEM_ID_SPI1        0    // SPI1 버스
#define HSEM_ID_SPI2        1    // SPI2 버스
#define HSEM_ID_I2C1        2    // I2C1 버스
#define HSEM_ID_I2C2        3    // I2C2 버스
#define HSEM_ID_UART1       4    // UART1
#define HSEM_ID_UART3       5    // UART3
#define HSEM_ID_ADC1        6    // ADC1
#define HSEM_ID_DAC1        7    // DAC1
#define HSEM_ID_TIM2        8    // TIM2
#define HSEM_ID_DMA_CH1     9    // DMA1 채널 1

// 시스템 HSEM ID
#define HSEM_ID_RESMGR_TABLE  30  // 리소스 테이블 접근
#define HSEM_ID_PRINTF        31  // 디버그 출력
```

## 리소스 관리자 구현

### 리소스 관리자 구조체
```c
typedef enum {
  RES_UNLOCKED = 0,
  RES_LOCKED_CM7,
  RES_LOCKED_CM4
} ResourceState_TypeDef;

typedef enum {
  RES_OK = 0,
  RES_ERROR,
  RES_BUSY,
  RES_TIMEOUT
} ResourceStatus_TypeDef;

typedef struct {
  uint8_t hsem_id;
  ResourceState_TypeDef state;
  uint32_t lock_count;
  uint32_t timeout_count;
} Resource_TypeDef;

// 리소스 테이블
typedef struct {
  Resource_TypeDef SPI1;
  Resource_TypeDef SPI2;
  Resource_TypeDef I2C1;
  Resource_TypeDef I2C2;
  Resource_TypeDef UART1;
  Resource_TypeDef UART3;
  Resource_TypeDef ADC1;
  Resource_TypeDef DAC1;
} ResourceTable_TypeDef;

// 공유 메모리에 배치
__attribute__((section(".shared_data")))
volatile ResourceTable_TypeDef ResourceTable;
```

### 리소스 매니저 초기화
```c
void ResourceManager_Init(void)
{
  // HSEM 클럭 활성화
  __HAL_RCC_HSEM_CLK_ENABLE();

  // 리소스 테이블 초기화
  ResourceTable.SPI1.hsem_id = HSEM_ID_SPI1;
  ResourceTable.SPI1.state = RES_UNLOCKED;
  ResourceTable.SPI1.lock_count = 0;

  ResourceTable.I2C1.hsem_id = HSEM_ID_I2C1;
  ResourceTable.I2C1.state = RES_UNLOCKED;
  ResourceTable.I2C1.lock_count = 0;

  // ... 다른 리소스 초기화 ...
}
```

### 리소스 잠금 함수
```c
ResourceStatus_TypeDef Resource_Lock(Resource_TypeDef *res, uint32_t timeout)
{
  uint32_t start_tick = HAL_GetTick();

  while (1)
  {
    // HSEM 획득 시도
    if (HAL_HSEM_FastTake(res->hsem_id) == HAL_OK)
    {
      // 성공
      #ifdef CORE_CM7
        res->state = RES_LOCKED_CM7;
      #else
        res->state = RES_LOCKED_CM4;
      #endif

      res->lock_count++;
      return RES_OK;
    }

    // 타임아웃 확인
    if ((HAL_GetTick() - start_tick) > timeout)
    {
      res->timeout_count++;
      return RES_TIMEOUT;
    }

    // 잠시 대기
    __NOP();
  }
}

ResourceStatus_TypeDef Resource_Unlock(Resource_TypeDef *res)
{
  // HSEM 해제
  HAL_HSEM_Release(res->hsem_id, 0);
  res->state = RES_UNLOCKED;

  return RES_OK;
}
```

### 즉시 잠금 (Non-blocking)
```c
ResourceStatus_TypeDef Resource_TryLock(Resource_TypeDef *res)
{
  if (HAL_HSEM_FastTake(res->hsem_id) == HAL_OK)
  {
    #ifdef CORE_CM7
      res->state = RES_LOCKED_CM7;
    #else
      res->state = RES_LOCKED_CM4;
    #endif

    res->lock_count++;
    return RES_OK;
  }

  return RES_BUSY;
}
```

## 코드 구현

### 1. CM7 메인 (Master)
```c
int main(void)
{
  // 시스템 초기화
  HAL_Init();
  SystemClock_Config();

  // LED 초기화
  BSP_LED_Init(LED1);
  BSP_LED_Init(LED2);

  // 리소스 매니저 초기화
  ResourceManager_Init();

  // 주변장치 초기화 (SPI, I2C 등)
  MX_SPI1_Init();
  MX_I2C1_Init();

  // CM4 부팅
  HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);

  // CM4 준비 대기
  HAL_Delay(100);

  while (1)
  {
    // SPI1 사용
    if (Resource_Lock(&ResourceTable.SPI1, 1000) == RES_OK)
    {
      SPI_TransmitData();
      Resource_Unlock(&ResourceTable.SPI1);
      BSP_LED_Toggle(LED1);
    }
    else
    {
      BSP_LED_Toggle(LED2);  // 잠금 실패
    }

    HAL_Delay(100);
  }
}
```

### 2. CM4 메인 (Remote)
```c
int main(void)
{
  // HAL 초기화
  HAL_Init();

  // LED 초기화
  BSP_LED_Init(LED3);
  BSP_LED_Init(LED4);

  while (1)
  {
    // SPI1 사용 (CM7과 공유)
    if (Resource_Lock(&ResourceTable.SPI1, 500) == RES_OK)
    {
      SPI_ReceiveData();
      Resource_Unlock(&ResourceTable.SPI1);
      BSP_LED_Toggle(LED3);
    }
    else
    {
      BSP_LED_Toggle(LED4);  // 잠금 실패
    }

    HAL_Delay(200);
  }
}
```

### 3. SPI 공유 예제
```c
// CM7: SPI를 통한 센서 읽기
void CM7_ReadSensor(uint8_t *data, uint16_t size)
{
  if (Resource_Lock(&ResourceTable.SPI1, 1000) == RES_OK)
  {
    // CS 핀 Low
    HAL_GPIO_WritePin(SPI1_CS_GPIO_Port, SPI1_CS_Pin, GPIO_PIN_RESET);

    // 데이터 읽기
    HAL_SPI_Receive(&hspi1, data, size, 1000);

    // CS 핀 High
    HAL_GPIO_WritePin(SPI1_CS_GPIO_Port, SPI1_CS_Pin, GPIO_PIN_SET);

    Resource_Unlock(&ResourceTable.SPI1);
  }
}

// CM4: SPI를 통한 메모리 쓰기
void CM4_WriteMemory(uint8_t *data, uint16_t size)
{
  if (Resource_Lock(&ResourceTable.SPI1, 1000) == RES_OK)
  {
    // CS 핀 Low
    HAL_GPIO_WritePin(SPI1_MEM_CS_GPIO_Port, SPI1_MEM_CS_Pin, GPIO_PIN_RESET);

    // 데이터 쓰기
    HAL_SPI_Transmit(&hspi1, data, size, 1000);

    // CS 핀 High
    HAL_GPIO_WritePin(SPI1_MEM_CS_GPIO_Port, SPI1_MEM_CS_Pin, GPIO_PIN_SET);

    Resource_Unlock(&ResourceTable.SPI1);
  }
}
```

### 4. I2C 공유 예제
```c
// I2C 장치 읽기 (공유)
HAL_StatusTypeDef Shared_I2C_Read(uint16_t DevAddr, uint16_t MemAddr,
                                   uint8_t *pData, uint16_t Size)
{
  HAL_StatusTypeDef status = HAL_ERROR;

  if (Resource_Lock(&ResourceTable.I2C1, 500) == RES_OK)
  {
    status = HAL_I2C_Mem_Read(&hi2c1, DevAddr, MemAddr,
                               I2C_MEMADD_SIZE_8BIT, pData, Size, 1000);
    Resource_Unlock(&ResourceTable.I2C1);
  }

  return status;
}

// I2C 장치 쓰기 (공유)
HAL_StatusTypeDef Shared_I2C_Write(uint16_t DevAddr, uint16_t MemAddr,
                                    uint8_t *pData, uint16_t Size)
{
  HAL_StatusTypeDef status = HAL_ERROR;

  if (Resource_Lock(&ResourceTable.I2C1, 500) == RES_OK)
  {
    status = HAL_I2C_Mem_Write(&hi2c1, DevAddr, MemAddr,
                                I2C_MEMADD_SIZE_8BIT, pData, Size, 1000);
    Resource_Unlock(&ResourceTable.I2C1);
  }

  return status;
}
```

### 5. UART 공유 예제
```c
// 공유 printf (HSEM으로 보호)
int Shared_Printf(const char *format, ...)
{
  char buffer[256];
  va_list args;
  int len;

  va_start(args, format);
  len = vsnprintf(buffer, sizeof(buffer), format, args);
  va_end(args);

  if (Resource_Lock(&ResourceTable.UART1, 100) == RES_OK)
  {
    HAL_UART_Transmit(&huart1, (uint8_t*)buffer, len, 1000);
    Resource_Unlock(&ResourceTable.UART1);
  }

  return len;
}
```

## 고급 기능

### 1. 우선순위 기반 잠금
```c
typedef enum {
  PRIORITY_LOW = 0,
  PRIORITY_NORMAL,
  PRIORITY_HIGH,
  PRIORITY_CRITICAL
} ResourcePriority_TypeDef;

ResourceStatus_TypeDef Resource_LockWithPriority(Resource_TypeDef *res,
                                                  ResourcePriority_TypeDef priority,
                                                  uint32_t timeout)
{
  uint32_t start_tick = HAL_GetTick();
  uint32_t delay_ms = (4 - priority) * 10;  // 우선순위에 따라 대기 시간 조절

  while (1)
  {
    if (HAL_HSEM_FastTake(res->hsem_id) == HAL_OK)
    {
      return RES_OK;
    }

    if ((HAL_GetTick() - start_tick) > timeout)
    {
      return RES_TIMEOUT;
    }

    HAL_Delay(delay_ms);  // 높은 우선순위는 더 짧게 대기
  }
}
```

### 2. 재귀적 잠금 (Recursive Lock)
```c
typedef struct {
  uint8_t hsem_id;
  volatile uint8_t owner_core;
  volatile uint8_t recursion_count;
} RecursiveResource_TypeDef;

ResourceStatus_TypeDef Resource_RecursiveLock(RecursiveResource_TypeDef *res)
{
  uint8_t current_core;

  #ifdef CORE_CM7
    current_core = 1;
  #else
    current_core = 2;
  #endif

  // 이미 이 코어가 소유한 경우
  if (res->owner_core == current_core)
  {
    res->recursion_count++;
    return RES_OK;
  }

  // 새로 획득 시도
  if (HAL_HSEM_FastTake(res->hsem_id) == HAL_OK)
  {
    res->owner_core = current_core;
    res->recursion_count = 1;
    return RES_OK;
  }

  return RES_BUSY;
}

ResourceStatus_TypeDef Resource_RecursiveUnlock(RecursiveResource_TypeDef *res)
{
  if (res->recursion_count > 0)
  {
    res->recursion_count--;

    if (res->recursion_count == 0)
    {
      res->owner_core = 0;
      HAL_HSEM_Release(res->hsem_id, 0);
    }
    return RES_OK;
  }

  return RES_ERROR;
}
```

### 3. 리소스 사용 통계
```c
typedef struct {
  uint32_t total_locks;
  uint32_t successful_locks;
  uint32_t failed_locks;
  uint32_t total_wait_time;
  uint32_t max_wait_time;
} ResourceStats_TypeDef;

// 통계 포함 잠금
ResourceStatus_TypeDef Resource_LockWithStats(Resource_TypeDef *res,
                                               ResourceStats_TypeDef *stats,
                                               uint32_t timeout)
{
  uint32_t start_tick = HAL_GetTick();

  stats->total_locks++;

  ResourceStatus_TypeDef status = Resource_Lock(res, timeout);

  uint32_t wait_time = HAL_GetTick() - start_tick;
  stats->total_wait_time += wait_time;

  if (wait_time > stats->max_wait_time)
  {
    stats->max_wait_time = wait_time;
  }

  if (status == RES_OK)
  {
    stats->successful_locks++;
  }
  else
  {
    stats->failed_locks++;
  }

  return status;
}

// 통계 출력
void PrintResourceStats(ResourceStats_TypeDef *stats)
{
  Shared_Printf("Resource Statistics:\n");
  Shared_Printf("  Total locks: %lu\n", stats->total_locks);
  Shared_Printf("  Successful: %lu\n", stats->successful_locks);
  Shared_Printf("  Failed: %lu\n", stats->failed_locks);
  Shared_Printf("  Avg wait: %lu ms\n",
                stats->total_wait_time / stats->total_locks);
  Shared_Printf("  Max wait: %lu ms\n", stats->max_wait_time);
}
```

### 4. 다중 리소스 잠금
```c
// 여러 리소스를 원자적으로 잠금 (데드락 방지)
ResourceStatus_TypeDef Resource_LockMultiple(Resource_TypeDef **resources,
                                              uint8_t count,
                                              uint32_t timeout)
{
  uint32_t start_tick = HAL_GetTick();

  // 항상 낮은 HSEM ID부터 획득 (데드락 방지)
  // 정렬 생략 - 호출자가 올바른 순서로 전달한다고 가정

  for (int i = 0; i < count; i++)
  {
    uint32_t remaining = timeout - (HAL_GetTick() - start_tick);

    if (Resource_Lock(resources[i], remaining) != RES_OK)
    {
      // 실패 - 이미 획득한 것들 해제
      for (int j = i - 1; j >= 0; j--)
      {
        Resource_Unlock(resources[j]);
      }
      return RES_TIMEOUT;
    }
  }

  return RES_OK;
}

// 사용 예
void MultiResourceOperation(void)
{
  Resource_TypeDef *resources[] = {
    &ResourceTable.SPI1,
    &ResourceTable.I2C1
  };

  if (Resource_LockMultiple(resources, 2, 1000) == RES_OK)
  {
    // SPI와 I2C 모두 사용 가능
    // ...

    Resource_Unlock(&ResourceTable.I2C1);
    Resource_Unlock(&ResourceTable.SPI1);
  }
}
```

## 응용 예제

### 1. 센서 퓨전 시스템
```c
// CM7: IMU 센서 읽기 (I2C)
void CM7_ReadIMU(void)
{
  if (Resource_Lock(&ResourceTable.I2C1, 100) == RES_OK)
  {
    // 가속도계, 자이로 데이터 읽기
    HAL_I2C_Mem_Read(&hi2c1, IMU_ADDR, ACCEL_REG,
                     I2C_MEMADD_SIZE_8BIT, accel_data, 6, 100);
    HAL_I2C_Mem_Read(&hi2c1, IMU_ADDR, GYRO_REG,
                     I2C_MEMADD_SIZE_8BIT, gyro_data, 6, 100);

    Resource_Unlock(&ResourceTable.I2C1);
  }
}

// CM4: 환경 센서 읽기 (같은 I2C 버스)
void CM4_ReadEnvironment(void)
{
  if (Resource_Lock(&ResourceTable.I2C1, 100) == RES_OK)
  {
    // 온습도 센서 읽기
    HAL_I2C_Mem_Read(&hi2c1, ENV_SENSOR_ADDR, TEMP_REG,
                     I2C_MEMADD_SIZE_8BIT, temp_data, 2, 100);

    Resource_Unlock(&ResourceTable.I2C1);
  }
}
```

### 2. 통신 멀티플렉싱
```c
// CM7: WiFi 모듈 통신 (SPI)
void CM7_WiFi_Send(uint8_t *data, uint16_t len)
{
  if (Resource_Lock(&ResourceTable.SPI2, 500) == RES_OK)
  {
    HAL_GPIO_WritePin(WIFI_CS_GPIO, WIFI_CS_PIN, GPIO_PIN_RESET);
    HAL_SPI_Transmit(&hspi2, data, len, 1000);
    HAL_GPIO_WritePin(WIFI_CS_GPIO, WIFI_CS_PIN, GPIO_PIN_SET);

    Resource_Unlock(&ResourceTable.SPI2);
  }
}

// CM4: SD 카드 접근 (같은 SPI 버스)
void CM4_SD_ReadBlock(uint32_t block, uint8_t *data)
{
  if (Resource_Lock(&ResourceTable.SPI2, 500) == RES_OK)
  {
    HAL_GPIO_WritePin(SD_CS_GPIO, SD_CS_PIN, GPIO_PIN_RESET);
    // SD 카드 읽기...
    HAL_GPIO_WritePin(SD_CS_GPIO, SD_CS_PIN, GPIO_PIN_SET);

    Resource_Unlock(&ResourceTable.SPI2);
  }
}
```

## 트러블슈팅

### 데드락 발생

1. **잠금 순서 통일**: 항상 같은 순서로 리소스 획득
2. **타임아웃 사용**: 무한 대기 방지
3. **순환 대기 제거**: 리소스 할당 그래프 분석

### 성능 저하

1. **잠금 범위 최소화**: 필요한 작업만 보호
2. **적절한 타임아웃**: 너무 짧거나 길지 않게
3. **우선순위 조정**: 중요한 작업에 높은 우선순위

### 잠금 누수

```c
// 안전한 패턴: 반드시 해제 보장
void SafeResourceAccess(void)
{
  if (Resource_Lock(&ResourceTable.SPI1, 1000) != RES_OK)
    return;

  // 여러 작업 수행
  // ...

  // 반드시 해제
  Resource_Unlock(&ResourceTable.SPI1);
}
```

## 성능 고려사항

### HSEM 오버헤드
- **획득 시간**: 약 10-100 CPU 사이클
- **해제 시간**: 약 10 CPU 사이클
- **경합 시**: 대기 시간 추가

### 최적화 팁
- 빈번한 접근이 필요한 리소스는 분리 고려
- 작은 전송은 버퍼링하여 한 번에 처리
- 비동기 통신 활용

## 참고 자료

- AN4838: STM32 hardware semaphores (HSEM)
- STM32H7 참조 매뉴얼: RM0399 (HSEM 섹션)
- 예제: `STM32H745I-DISCO/Applications/ResourcesManager/ResourcesManager_SharedResources`
