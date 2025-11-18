# FreeRTOS_AMP_Dual_RTOS - 듀얼 코어 FreeRTOS (AMP 구성)

## 개요

이 애플리케이션은 STM32H745의 Cortex-M7과 Cortex-M4에서 **각각 독립적인 FreeRTOS 인스턴스**를 실행하는 AMP (Asymmetric Multi-Processing) 구성을 보여줍니다. 두 코어는 Message Buffer를 통해 데이터를 주고받습니다.

## AMP (Asymmetric Multi-Processing)

### AMP vs SMP
- **AMP**: 각 코어가 독립적인 OS 실행 (이 예제)
- **SMP**: 단일 OS가 여러 코어 관리 (STM32H7는 SMP 미지원)

### 특징
```
┌──────────────────────┐    ┌──────────────────────┐
│   Cortex-M7          │    │   Cortex-M4          │
│   @ 400 MHz          │    │   @ 200 MHz          │
├──────────────────────┤    ├──────────────────────┤
│  FreeRTOS Instance 1 │    │  FreeRTOS Instance 2 │
│                      │    │                      │
│  - Scheduler         │    │  - Scheduler         │
│  - Tasks             │    │  - Tasks             │
│  - Queues            │    │  - Queues            │
│  - Semaphores        │    │  - Semaphores        │
│                      │    │                      │
└──────────┬───────────┘    └───────────┬──────────┘
           │                            │
           │   Message Buffer (SRAM)    │
           │   Stream Buffer (SRAM)     │
           └────────────┬───────────────┘
                        │
                  Shared Memory
                  (0x38000000)
```

## 시스템 아키텍처

### 코어별 역할

#### Cortex-M7 (Master)
- FreeRTOS 커널 실행
- 사용자 인터페이스 (LED1)
- CM4로 메시지 전송
- 2초마다 LED1 토글

#### Cortex-M4 (Remote)
- FreeRTOS 커널 실행
- 백그라운드 처리
- CM7에서 메시지 수신
- 수신 시 LED3 토글

## 공유 메모리

### 메모리 맵
```c
// D3 SRAM (0x38000000)을 코어 간 통신에 사용
#define SHARED_MEM_BASE    0x38000000
#define SHARED_MEM_SIZE    0x10000  // 64KB

// Message Buffer 위치
#define MESSAGE_BUFFER_CM7_TO_CM4  (SHARED_MEM_BASE + 0x0000)
#define MESSAGE_BUFFER_CM4_TO_CM7  (SHARED_MEM_BASE + 0x1000)

// Stream Buffer 위치 (대용량 데이터용)
#define STREAM_BUFFER_CM7_TO_CM4   (SHARED_MEM_BASE + 0x2000)
#define STREAM_BUFFER_CM4_TO_CM7   (SHARED_MEM_BASE + 0x4000)
```

### MPU 설정 (중요!)

공유 메모리는 non-cacheable로 설정해야 합니다:

```c
void MPU_Config_Shared_Memory(void)
{
  MPU_Region_InitTypeDef MPU_InitStruct;

  HAL_MPU_Disable();

  // D3 SRAM을 Shared, Non-cacheable로 설정
  MPU_InitStruct.Enable = MPU_REGION_ENABLE;
  MPU_InitStruct.Number = MPU_REGION_NUMBER0;
  MPU_InitStruct.BaseAddress = 0x38000000;
  MPU_InitStruct.Size = MPU_REGION_SIZE_64KB;
  MPU_InitStruct.SubRegionDisable = 0x00;
  MPU_InitStruct.TypeExtField = MPU_TEX_LEVEL0;
  MPU_InitStruct.AccessPermission = MPU_REGION_FULL_ACCESS;
  MPU_InitStruct.DisableExec = MPU_INSTRUCTION_ACCESS_ENABLE;
  MPU_InitStruct.IsShareable = MPU_ACCESS_SHAREABLE;
  MPU_InitStruct.IsCacheable = MPU_ACCESS_NOT_CACHEABLE;
  MPU_InitStruct.IsBufferable = MPU_ACCESS_NOT_BUFFERABLE;

  HAL_MPU_ConfigRegion(&MPU_InitStruct);

  HAL_MPU_Enable(MPU_PRIVILEGED_DEFAULT);
}
```

## FreeRTOS 설정

### FreeRTOSConfig.h (각 코어별)

```c
// CM7 설정
#define configCPU_CLOCK_HZ                    400000000
#define configTICK_RATE_HZ                    1000
#define configMAX_PRIORITIES                  7
#define configMINIMAL_STACK_SIZE              128
#define configTOTAL_HEAP_SIZE                 (64 * 1024)  // 64KB

// CM4 설정
#define configCPU_CLOCK_HZ                    200000000
#define configTICK_RATE_HZ                    1000
#define configMAX_PRIORITIES                  7
#define configMINIMAL_STACK_SIZE              128
#define configTOTAL_HEAP_SIZE                 (32 * 1024)  // 32KB

// Message Buffer 지원
#define configUSE_MESSAGE_BUFFERS             1
#define configSUPPORT_DYNAMIC_ALLOCATION      1
```

## CM7 코드 구현

### 메인 함수
```c
#include "FreeRTOS.h"
#include "task.h"
#include "message_buffer.h"

// Message Buffer 핸들 (정적 할당)
MessageBufferHandle_t xMessageBufferToM4 = NULL;
MessageBufferHandle_t xMessageBufferFromM4 = NULL;

// 공유 메모리에 Message Buffer 스토리지 할당
__attribute__((section(".shared_memory")))
uint8_t ucStorageBufferToM4[MESSAGE_BUFFER_SIZE];

__attribute__((section(".shared_memory")))
StaticMessageBuffer_t xMessageBufferStructToM4;

int main(void)
{
  // 시스템 초기화
  HAL_Init();
  SystemClock_Config();

  // MPU 설정 (공유 메모리)
  MPU_Config_Shared_Memory();

  // LED 초기화
  BSP_LED_Init(LED1);
  BSP_LED_Init(LED2);

  // HSEM 초기화
  __HAL_RCC_HSEM_CLK_ENABLE();

  // Message Buffer 생성 (정적 할당)
  xMessageBufferToM4 = xMessageBufferCreateStatic(
    MESSAGE_BUFFER_SIZE,
    ucStorageBufferToM4,
    &xMessageBufferStructToM4
  );

  // CM4 부팅
  HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);

  // HSEM으로 CM4에 준비 신호
  HAL_HSEM_FastTake(HSEM_ID_0);
  HAL_HSEM_Release(HSEM_ID_0, 0);

  // FreeRTOS 태스크 생성
  xTaskCreate(TaskSender, "Sender", 256, NULL, 3, NULL);
  xTaskCreate(TaskReceiver, "Receiver", 256, NULL, 3, NULL);
  xTaskCreate(TaskLED, "LED", 128, NULL, 2, NULL);

  // 스케줄러 시작
  vTaskStartScheduler();

  // 여기까지 오면 안됨
  while (1);
}
```

### 송신 태스크
```c
void TaskSender(void *argument)
{
  uint32_t counter = 0;
  char message[64];

  for (;;)
  {
    // 메시지 준비
    snprintf(message, sizeof(message), "CM7 Message #%lu", counter++);

    // CM4로 메시지 전송
    size_t sent = xMessageBufferSend(
      xMessageBufferToM4,
      message,
      strlen(message),
      portMAX_DELAY
    );

    if (sent > 0)
    {
      printf("CM7: Sent message: %s\n", message);
    }

    // 2초 대기
    vTaskDelay(pdMS_TO_TICKS(2000));
  }
}
```

### 수신 태스크
```c
void TaskReceiver(void *argument)
{
  char received_message[64];
  size_t received_length;

  for (;;)
  {
    // CM4로부터 메시지 대기
    received_length = xMessageBufferReceive(
      xMessageBufferFromM4,
      received_message,
      sizeof(received_message),
      portMAX_DELAY
    );

    if (received_length > 0)
    {
      received_message[received_length] = '\0';
      printf("CM7: Received from CM4: %s\n", received_message);

      // 수신 확인 LED 토글
      BSP_LED_Toggle(LED2);
    }
  }
}
```

### LED 제어 태스크
```c
void TaskLED(void *argument)
{
  for (;;)
  {
    BSP_LED_Toggle(LED1);
    vTaskDelay(pdMS_TO_TICKS(2000));
  }
}
```

## CM4 코드 구현

### 메인 함수
```c
#include "FreeRTOS.h"
#include "task.h"
#include "message_buffer.h"

// Message Buffer 핸들 (CM7과 동일한 메모리 사용)
MessageBufferHandle_t xMessageBufferFromM7 = NULL;
MessageBufferHandle_t xMessageBufferToM7 = NULL;

int main(void)
{
  // HSEM 대기
  while(__HAL_HSEM_SEMID_GET(HSEM_ID_0) != 0);

  // HAL 초기화
  HAL_Init();

  // MPU 설정
  MPU_Config_Shared_Memory();

  // LED 초기화
  BSP_LED_Init(LED3);
  BSP_LED_Init(LED4);

  // Message Buffer 연결 (CM7이 생성한 것 사용)
  // 주의: CM7과 같은 메모리 주소 사용
  xMessageBufferFromM7 = xMessageBufferCreateStatic(
    MESSAGE_BUFFER_SIZE,
    (uint8_t*)MESSAGE_BUFFER_CM7_TO_CM4,  // CM7이 할당한 메모리
    (StaticMessageBuffer_t*)(MESSAGE_BUFFER_CM7_TO_CM4 + MESSAGE_BUFFER_SIZE)
  );

  // FreeRTOS 태스크 생성
  xTaskCreate(TaskReceiver, "Receiver", 256, NULL, 3, NULL);
  xTaskCreate(TaskSender, "Sender", 256, NULL, 3, NULL);
  xTaskCreate(TaskProcess, "Process", 256, NULL, 2, NULL);

  // 스케줄러 시작
  vTaskStartScheduler();

  while (1);
}
```

### CM4 수신 태스크
```c
void TaskReceiver(void *argument)
{
  char received_message[64];
  size_t received_length;

  for (;;)
  {
    // CM7로부터 메시지 수신
    received_length = xMessageBufferReceive(
      xMessageBufferFromM7,
      received_message,
      sizeof(received_message),
      portMAX_DELAY
    );

    if (received_length > 0)
    {
      received_message[received_length] = '\0';
      printf("CM4: Received from CM7: %s\n", received_message);

      // 수신 시 LED 토글
      BSP_LED_Toggle(LED3);

      // 응답 메시지 준비
      char reply[64];
      snprintf(reply, sizeof(reply), "CM4 ACK: %s", received_message);

      // CM7로 응답 전송
      xMessageBufferSend(xMessageBufferToM7, reply, strlen(reply), 0);
    }
  }
}
```

### CM4 처리 태스크
```c
void TaskProcess(void *argument)
{
  uint32_t process_count = 0;

  for (;;)
  {
    // 백그라운드 작업 시뮬레이션
    process_count++;

    if (process_count % 1000 == 0)
    {
      printf("CM4: Processed %lu items\n", process_count);
    }

    // 다른 태스크에 양보
    vTaskDelay(pdMS_TO_TICKS(1));
  }
}
```

## Stream Buffer (대용량 데이터)

Message Buffer는 작은 메시지에 적합합니다. 대용량 데이터는 Stream Buffer 사용:

### Stream Buffer 생성
```c
// CM7
StreamBufferHandle_t xStreamBufferToM4;

xStreamBufferToM4 = xStreamBufferCreateStatic(
  STREAM_BUFFER_SIZE,
  1,  // Trigger level (1바이트 이상이면 수신 가능)
  ucStreamBufferStorage,
  &xStreamBufferStruct
);
```

### 대용량 데이터 전송
```c
// CM7: 송신
uint8_t data_buffer[1024];
// ... 데이터 준비 ...
size_t sent = xStreamBufferSend(
  xStreamBufferToM4,
  data_buffer,
  sizeof(data_buffer),
  portMAX_DELAY
);

// CM4: 수신
uint8_t recv_buffer[1024];
size_t received = xStreamBufferReceive(
  xStreamBufferFromM7,
  recv_buffer,
  sizeof(recv_buffer),
  portMAX_DELAY
);
```

## 동기화

### HSEM을 사용한 알림
```c
// CM7: CM4에 이벤트 알림
void NotifyCM4(void)
{
  // HSEM 인터럽트 트리거
  HAL_HSEM_ActivateNotification(__HAL_HSEM_SEMID_TO_MASK(HSEM_CH_MESSAGE));
}

// CM4: HSEM 인터럽트 핸들러
void HSEM1_IRQHandler(void)
{
  HAL_HSEM_IRQHandler();

  // FreeRTOS 태스크 알림
  BaseType_t xHigherPriorityTaskWoken = pdFALSE;
  vTaskNotifyGiveFromISR(xReceiverTaskHandle, &xHigherPriorityTaskWoken);
  portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

## 메모리 할당

### 힙 메모리

각 코어는 독립적인 FreeRTOS 힙 사용:

```c
// CM7: DTCM에 힙 할당 (고속 액세스)
__attribute__((section(".dtcm_heap")))
uint8_t ucHeap[configTOTAL_HEAP_SIZE];

// CM4: D2 SRAM에 힙 할당
__attribute__((section(".d2_heap")))
uint8_t ucHeap[configTOTAL_HEAP_SIZE];
```

### 링커 스크립트

#### CM7 (STM32CubeIDE)
```ld
.dtcm_heap :
{
  . = ALIGN(8);
  PROVIDE(ucHeap = .);
  . = . + 65536;  /* 64KB */
  . = ALIGN(8);
} >DTCMRAM
```

#### CM4
```ld
.d2_heap :
{
  . = ALIGN(8);
  PROVIDE(ucHeap = .);
  . = . + 32768;  /* 32KB */
  . = ALIGN(8);
} >RAM_D2
```

## 성능 모니터링

### Task 통계
```c
void PrintTaskStats(void)
{
  TaskStatus_t *pxTaskStatusArray;
  volatile UBaseType_t uxArraySize, x;
  uint32_t ulTotalRunTime, ulStatsAsPercentage;

  // 태스크 개수 확인
  uxArraySize = uxTaskGetNumberOfTasks();

  // 메모리 할당
  pxTaskStatusArray = pvPortMalloc(uxArraySize * sizeof(TaskStatus_t));

  if (pxTaskStatusArray != NULL)
  {
    // 태스크 정보 수집
    uxArraySize = uxTaskGetSystemState(pxTaskStatusArray,
                                       uxArraySize,
                                       &ulTotalRunTime);

    // 통계 출력
    printf("Task\t\tState\tPrio\tStack\tCPU%%\n");
    for (x = 0; x < uxArraySize; x++)
    {
      ulStatsAsPercentage = (pxTaskStatusArray[x].ulRunTimeCounter * 100) /
                            ulTotalRunTime;

      printf("%s\t\t%d\t%lu\t%u\t%lu%%\n",
             pxTaskStatusArray[x].pcTaskName,
             pxTaskStatusArray[x].eCurrentState,
             pxTaskStatusArray[x].uxCurrentPriority,
             pxTaskStatusArray[x].usStackHighWaterMark,
             ulStatsAsPercentage);
    }

    vPortFree(pxTaskStatusArray);
  }
}
```

### 힙 사용량
```c
void PrintHeapInfo(void)
{
  size_t free_heap = xPortGetFreeHeapSize();
  size_t min_free_heap = xPortGetMinimumEverFreeHeapSize();

  printf("Heap Free: %lu bytes\n", free_heap);
  printf("Heap Min Free: %lu bytes\n", min_free_heap);
}
```

## 트러블슈팅

### 메시지가 전달되지 않는 경우
1. **공유 메모리 MPU 설정**: Non-cacheable 확인
2. **Message Buffer 주소**: 양쪽 코어가 같은 주소 사용
3. **HSEM 동기화**: CM4 부팅 대기 확인

### 스택 오버플로우
```c
// FreeRTOSConfig.h에 추가
#define configCHECK_FOR_STACK_OVERFLOW  2

// 콜백 구현
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName)
{
  printf("Stack overflow in task: %s\n", pcTaskName);
  Error_Handler();
}
```

### 힙 할당 실패
```c
void vApplicationMallocFailedHook(void)
{
  printf("Malloc failed!\n");
  PrintHeapInfo();
  Error_Handler();
}
```

## 확장 예제

### 1. 우선순위 기반 작업 분배
```c
// CM7: 고우선순위 작업
// CM4: 저우선순위 백그라운드 작업
```

### 2. 센서 데이터 처리
```c
// CM7: UI + 센서 읽기
// CM4: 신호 처리 + 필터링
```

### 3. 통신 프로토콜 처리
```c
// CM7: 네트워크 스택
// CM4: 프로토콜 인코딩/디코딩
```

## 참고 자료

- FreeRTOS 공식 문서: https://www.freertos.org/
- AN5617: STM32H7 dual-core OpenAMP framework
- Mastering the FreeRTOS Real Time Kernel (무료 PDF)
- 예제: `STM32H745I-DISCO/Applications/FreeRTOS/FreeRTOS_AMP_Dual_RTOS`
