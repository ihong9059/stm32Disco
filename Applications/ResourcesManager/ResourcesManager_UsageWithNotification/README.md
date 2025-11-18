# ResourcesManager_UsageWithNotification - 리소스 알림 기반 관리

## 개요

이 애플리케이션은 HSEM (Hardware Semaphore) 인터럽트를 사용하여 리소스 사용 가능 여부를 알림받는 이벤트 기반 리소스 관리 방법을 보여줍니다. 폴링 방식 대신 인터럽트를 통해 효율적으로 리소스 해제를 감지합니다.

## 이벤트 기반 리소스 관리

### 폴링 vs 알림

| 방식 | 장점 | 단점 |
|------|------|------|
| **폴링** | 단순한 구현 | CPU 낭비, 지연 발생 |
| **알림** | 즉각적 반응, 저전력 | 복잡한 구현 |

### 알림 기반의 이점
- **저전력**: WFI (Wait For Interrupt) 사용 가능
- **즉각적 응답**: 리소스 해제 즉시 획득
- **효율성**: CPU를 다른 작업에 활용

## 시스템 아키텍처

### 알림 기반 리소스 관리
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
│  │            │  │              │  │            │  │
│  │ HSEM IRQ ◄─┼──┼──────────────┼──┼─► HSEM IRQ │  │
│  │ Handler    │  │              │  │   Handler  │  │
│  └──────┬─────┘  │              │  └──────┬─────┘  │
│         │        │              │         │        │
└─────────┼────────┘              └─────────┼────────┘
          │                                 │
          │         ┌─────────┐             │
          └────────►│  HSEM   │◄────────────┘
                    │         │
                    │ IRQ Gen │
                    └────┬────┘
                         │
                    ┌────▼────┐
                    │ Shared  │
                    │Resources│
                    └─────────┘
```

## HSEM 알림 구성

### HSEM 인터럽트 개요
- **HSEM1**: CM7용 인터럽트
- **HSEM2**: CM4용 인터럽트
- **알림 마스크**: 특정 세마포어 해제 시 인터럽트

### HSEM 초기화
```c
void HSEM_Init_WithNotification(void)
{
  // HSEM 클럭 활성화
  __HAL_RCC_HSEM_CLK_ENABLE();

  // HSEM 인터럽트 활성화
  #ifdef CORE_CM7
    HAL_NVIC_SetPriority(HSEM1_IRQn, 5, 0);
    HAL_NVIC_EnableIRQ(HSEM1_IRQn);
  #else
    HAL_NVIC_SetPriority(HSEM2_IRQn, 5, 0);
    HAL_NVIC_EnableIRQ(HSEM2_IRQn);
  #endif
}
```

## 코드 구현

### 1. 알림 콜백 등록
```c
// 리소스별 콜백 함수 포인터
typedef void (*ResourceCallback_t)(void);

typedef struct {
  uint8_t hsem_id;
  ResourceCallback_t callback;
  volatile uint8_t notification_pending;
} NotifiedResource_TypeDef;

NotifiedResource_TypeDef resources[MAX_RESOURCES];

// 콜백 등록
void Resource_RegisterCallback(uint8_t hsem_id, ResourceCallback_t callback)
{
  for (int i = 0; i < MAX_RESOURCES; i++)
  {
    if (resources[i].hsem_id == hsem_id)
    {
      resources[i].callback = callback;
      break;
    }
  }
}
```

### 2. 알림 활성화
```c
void Resource_EnableNotification(uint8_t hsem_id)
{
  // 해당 HSEM의 알림 활성화
  HAL_HSEM_ActivateNotification(__HAL_HSEM_SEMID_TO_MASK(hsem_id));
}

void Resource_DisableNotification(uint8_t hsem_id)
{
  // 해당 HSEM의 알림 비활성화
  HAL_HSEM_DeactivateNotification(__HAL_HSEM_SEMID_TO_MASK(hsem_id));
}
```

### 3. HSEM 인터럽트 핸들러
```c
// CM7용 HSEM 인터럽트 핸들러
void HSEM1_IRQHandler(void)
{
  HAL_HSEM_IRQHandler();
}

// CM4용 HSEM 인터럽트 핸들러
void HSEM2_IRQHandler(void)
{
  HAL_HSEM_IRQHandler();
}

// HAL 콜백
void HAL_HSEM_FreeCallback(uint32_t SemMask)
{
  // 어떤 세마포어가 해제되었는지 확인
  for (int i = 0; i < 32; i++)
  {
    if (SemMask & (1 << i))
    {
      // 해당 리소스의 콜백 호출
      for (int j = 0; j < MAX_RESOURCES; j++)
      {
        if (resources[j].hsem_id == i && resources[j].callback != NULL)
        {
          resources[j].notification_pending = 1;
          resources[j].callback();
          break;
        }
      }
    }
  }
}
```

### 4. 알림 기반 잠금
```c
// 비동기 잠금 요청
ResourceStatus_TypeDef Resource_LockAsync(NotifiedResource_TypeDef *res)
{
  // 즉시 획득 시도
  if (HAL_HSEM_FastTake(res->hsem_id) == HAL_OK)
  {
    return RES_OK;
  }

  // 획득 실패 - 알림 활성화
  Resource_EnableNotification(res->hsem_id);

  return RES_PENDING;
}

// 콜백에서 재시도
void SPI1_ResourceFreeCallback(void)
{
  // 리소스가 해제됨 - 다시 획득 시도
  if (HAL_HSEM_FastTake(HSEM_ID_SPI1) == HAL_OK)
  {
    // 획득 성공
    Resource_DisableNotification(HSEM_ID_SPI1);

    // 대기 중이던 작업 재개
    spi1_pending_operation = 0;
    SPI1_ProcessPendingData();
  }
  else
  {
    // 다른 코어가 먼저 획득 - 계속 대기
    Resource_EnableNotification(HSEM_ID_SPI1);
  }
}
```

### 5. CM7 메인 (이벤트 기반)
```c
volatile uint8_t spi1_available = 0;
volatile uint8_t i2c1_available = 0;

int main(void)
{
  // 시스템 초기화
  HAL_Init();
  SystemClock_Config();

  // LED 초기화
  BSP_LED_Init(LED1);
  BSP_LED_Init(LED2);

  // HSEM 알림 초기화
  HSEM_Init_WithNotification();

  // 리소스 콜백 등록
  Resource_RegisterCallback(HSEM_ID_SPI1, SPI1_AvailableCallback);
  Resource_RegisterCallback(HSEM_ID_I2C1, I2C1_AvailableCallback);

  // CM4 부팅
  HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);

  while (1)
  {
    // 이벤트 기반 처리
    if (spi1_available)
    {
      spi1_available = 0;
      ProcessSPI1Data();
    }

    if (i2c1_available)
    {
      i2c1_available = 0;
      ProcessI2C1Data();
    }

    // 대기 상태 진입 (저전력)
    __WFI();
  }
}

void SPI1_AvailableCallback(void)
{
  spi1_available = 1;
}

void I2C1_AvailableCallback(void)
{
  i2c1_available = 1;
}
```

### 6. CM4 메인 (이벤트 기반)
```c
int main(void)
{
  // HAL 초기화
  HAL_Init();

  // LED 초기화
  BSP_LED_Init(LED3);
  BSP_LED_Init(LED4);

  // HSEM 알림 초기화
  HSEM_Init_WithNotification();

  // 콜백 등록
  Resource_RegisterCallback(HSEM_ID_SPI1, CM4_SPI1_Callback);

  while (1)
  {
    // 백그라운드 작업
    CM4_BackgroundTask();

    // 저전력 대기
    __WFI();
  }
}

void CM4_SPI1_Callback(void)
{
  BSP_LED_Toggle(LED3);

  // SPI1이 사용 가능해짐
  if (HAL_HSEM_FastTake(HSEM_ID_SPI1) == HAL_OK)
  {
    // 데이터 처리
    CM4_SPI1_Operation();
    HAL_HSEM_Release(HSEM_ID_SPI1, 0);
  }
}
```

## 고급 기능

### 1. 대기열 기반 요청 관리
```c
#define MAX_PENDING_REQUESTS 8

typedef struct {
  uint8_t hsem_id;
  void (*callback)(void *);
  void *arg;
} PendingRequest_TypeDef;

// 대기열
PendingRequest_TypeDef pending_queue[MAX_PENDING_REQUESTS];
uint8_t queue_head = 0;
uint8_t queue_tail = 0;

// 요청 추가
void Resource_QueueRequest(uint8_t hsem_id, void (*callback)(void *), void *arg)
{
  // 먼저 즉시 획득 시도
  if (HAL_HSEM_FastTake(hsem_id) == HAL_OK)
  {
    callback(arg);
    HAL_HSEM_Release(hsem_id, 0);
    return;
  }

  // 대기열에 추가
  pending_queue[queue_tail].hsem_id = hsem_id;
  pending_queue[queue_tail].callback = callback;
  pending_queue[queue_tail].arg = arg;
  queue_tail = (queue_tail + 1) % MAX_PENDING_REQUESTS;

  // 알림 활성화
  Resource_EnableNotification(hsem_id);
}

// 대기열 처리 (인터럽트에서 호출)
void ProcessPendingQueue(uint8_t hsem_id)
{
  while (queue_head != queue_tail)
  {
    if (pending_queue[queue_head].hsem_id == hsem_id)
    {
      if (HAL_HSEM_FastTake(hsem_id) == HAL_OK)
      {
        pending_queue[queue_head].callback(pending_queue[queue_head].arg);
        HAL_HSEM_Release(hsem_id, 0);
        queue_head = (queue_head + 1) % MAX_PENDING_REQUESTS;
      }
      else
      {
        // 다시 대기
        break;
      }
    }
    else
    {
      queue_head = (queue_head + 1) % MAX_PENDING_REQUESTS;
    }
  }
}
```

### 2. 타임아웃 기반 알림
```c
// 타임아웃과 알림 조합
ResourceStatus_TypeDef Resource_LockWithTimeout(uint8_t hsem_id, uint32_t timeout_ms)
{
  // 즉시 획득 시도
  if (HAL_HSEM_FastTake(hsem_id) == HAL_OK)
  {
    return RES_OK;
  }

  // 알림 활성화
  Resource_EnableNotification(hsem_id);

  // 타이머 시작
  uint32_t start_tick = HAL_GetTick();

  while ((HAL_GetTick() - start_tick) < timeout_ms)
  {
    // 알림 대기
    __WFI();

    // 재시도
    if (HAL_HSEM_FastTake(hsem_id) == HAL_OK)
    {
      Resource_DisableNotification(hsem_id);
      return RES_OK;
    }
  }

  // 타임아웃
  Resource_DisableNotification(hsem_id);
  return RES_TIMEOUT;
}
```

### 3. 우선순위 기반 알림
```c
typedef struct {
  uint8_t hsem_id;
  uint8_t priority;
  ResourceCallback_t callback;
} PriorityRequest_TypeDef;

#define MAX_PRIORITY_REQUESTS 16
PriorityRequest_TypeDef priority_queue[MAX_PRIORITY_REQUESTS];
uint8_t priority_queue_count = 0;

// 우선순위에 따라 대기열 삽입
void Resource_QueueWithPriority(uint8_t hsem_id, uint8_t priority,
                                 ResourceCallback_t callback)
{
  // 정렬된 위치 찾기
  int insert_pos = priority_queue_count;
  for (int i = 0; i < priority_queue_count; i++)
  {
    if (priority > priority_queue[i].priority)
    {
      insert_pos = i;
      break;
    }
  }

  // 뒤로 밀기
  for (int i = priority_queue_count; i > insert_pos; i--)
  {
    priority_queue[i] = priority_queue[i-1];
  }

  // 삽입
  priority_queue[insert_pos].hsem_id = hsem_id;
  priority_queue[insert_pos].priority = priority;
  priority_queue[insert_pos].callback = callback;
  priority_queue_count++;

  Resource_EnableNotification(hsem_id);
}

// 우선순위 순으로 처리
void ProcessPriorityQueue(uint8_t hsem_id)
{
  for (int i = 0; i < priority_queue_count; i++)
  {
    if (priority_queue[i].hsem_id == hsem_id)
    {
      if (HAL_HSEM_FastTake(hsem_id) == HAL_OK)
      {
        priority_queue[i].callback();
        HAL_HSEM_Release(hsem_id, 0);

        // 제거
        for (int j = i; j < priority_queue_count - 1; j++)
        {
          priority_queue[j] = priority_queue[j+1];
        }
        priority_queue_count--;
        break;
      }
    }
  }
}
```

### 4. FreeRTOS 통합
```c
#include "FreeRTOS.h"
#include "task.h"
#include "semphr.h"

// FreeRTOS 세마포어
SemaphoreHandle_t xSPI1_Semaphore;

// 초기화
void Resource_Init_RTOS(void)
{
  xSPI1_Semaphore = xSemaphoreCreateBinary();

  // HSEM 알림 활성화
  HSEM_Init_WithNotification();
  Resource_EnableNotification(HSEM_ID_SPI1);
}

// HSEM 콜백에서 FreeRTOS 세마포어 해제
void SPI1_HSEM_Callback(void)
{
  BaseType_t xHigherPriorityTaskWoken = pdFALSE;
  xSemaphoreGiveFromISR(xSPI1_Semaphore, &xHigherPriorityTaskWoken);
  portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

// 태스크에서 대기
void SPI1_Task(void *argument)
{
  for (;;)
  {
    // HSEM 획득 시도
    if (HAL_HSEM_FastTake(HSEM_ID_SPI1) != HAL_OK)
    {
      // 획득 실패 - FreeRTOS 세마포어 대기
      xSemaphoreTake(xSPI1_Semaphore, portMAX_DELAY);

      // 재시도
      while (HAL_HSEM_FastTake(HSEM_ID_SPI1) != HAL_OK)
      {
        xSemaphoreTake(xSPI1_Semaphore, portMAX_DELAY);
      }
    }

    // SPI1 사용
    SPI1_Operation();

    HAL_HSEM_Release(HSEM_ID_SPI1, 0);

    vTaskDelay(pdMS_TO_TICKS(100));
  }
}
```

## 응용 예제

### 1. 센서 데이터 수집 시스템
```c
// CM7: 고속 센서 (ADC)
void CM7_ADC_Task(void)
{
  // ADC 리소스 획득
  if (Resource_LockAsync(&adc_resource) == RES_PENDING)
  {
    // 대기 - 콜백에서 처리
    return;
  }

  // ADC 변환 시작
  HAL_ADC_Start_DMA(&hadc1, adc_buffer, ADC_BUFFER_SIZE);
}

void ADC_ResourceCallback(void)
{
  // ADC 사용 가능
  HAL_ADC_Start_DMA(&hadc1, adc_buffer, ADC_BUFFER_SIZE);
}

// CM4: 저속 센서 (I2C)
void CM4_I2C_Task(void)
{
  if (Resource_LockAsync(&i2c_resource) == RES_PENDING)
  {
    return;
  }

  // I2C 센서 읽기
  HAL_I2C_Mem_Read(&hi2c1, SENSOR_ADDR, REG_ADDR,
                   I2C_MEMADD_SIZE_8BIT, data, len, 100);
}
```

### 2. 통신 프로토콜 스택
```c
// CM7: 네트워크 스택
void NetworkStack_Task(void)
{
  for (;;)
  {
    // UART 패킷 수신 대기
    xSemaphoreTake(xUART_RxSemaphore, portMAX_DELAY);

    // UART 리소스 획득 (알림 기반)
    if (HAL_HSEM_FastTake(HSEM_ID_UART1) == HAL_OK)
    {
      ProcessNetworkPacket();
      HAL_HSEM_Release(HSEM_ID_UART1, 0);
    }
    else
    {
      // 대기
      Resource_EnableNotification(HSEM_ID_UART1);
    }
  }
}

// CM4: 디버그 출력
void Debug_Output(const char *msg)
{
  // UART 리소스 획득
  Resource_QueueRequest(HSEM_ID_UART1, DebugOutputCallback, (void*)msg);
}
```

## 트러블슈팅

### 알림이 오지 않는 경우

1. **인터럽트 활성화 확인**:
   ```c
   HAL_NVIC_EnableIRQ(HSEM1_IRQn);  // CM7
   HAL_NVIC_EnableIRQ(HSEM2_IRQn);  // CM4
   ```

2. **알림 마스크 확인**:
   ```c
   HAL_HSEM_ActivateNotification(__HAL_HSEM_SEMID_TO_MASK(hsem_id));
   ```

3. **인터럽트 핸들러 구현 확인**:
   ```c
   void HSEM1_IRQHandler(void)
   {
     HAL_HSEM_IRQHandler();  // 반드시 호출
   }
   ```

### 콜백이 호출되지 않는 경우

1. **HAL_HSEM_FreeCallback 구현 확인**
2. **콜백 등록 확인**
3. **세마포어가 실제로 해제되었는지 확인**

### 인터럽트 폭주

1. **알림 비활성화**: 처리 완료 후 비활성화
2. **인터럽트 우선순위**: 적절히 설정
3. **플래그 클리어**: HAL_HSEM_IRQHandler()가 처리

## 성능 고려사항

### 인터럽트 지연
- **HSEM 인터럽트**: 약 10-100 CPU 사이클
- **콜백 오버헤드**: 함수 호출 비용
- **컨텍스트 스위칭**: RTOS 사용 시 추가

### 최적화
- 불필요한 알림 비활성화
- 콜백에서 최소한의 작업
- 대기열 사용으로 인터럽트 부하 분산

### 전력 효율
- WFI/WFE 사용으로 대기 중 저전력
- 폴링 대비 CPU 사용률 감소

## 참고 자료

- AN4838: STM32 hardware semaphores (HSEM)
- STM32H7 참조 매뉴얼: RM0399 (HSEM 인터럽트 섹션)
- 예제: `STM32H745I-DISCO/Applications/ResourcesManager/ResourcesManager_UsageWithNotification`
