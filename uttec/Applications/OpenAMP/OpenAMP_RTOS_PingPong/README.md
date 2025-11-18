# OpenAMP_RTOS_PingPong - FreeRTOS 기반 OpenAMP 통신

## 개요

이 애플리케이션은 FreeRTOS 환경에서 OpenAMP 프레임워크를 사용하여 Cortex-M7과 Cortex-M4 간에 RPMsg 메시지를 주고받는 예제입니다. 태스크 기반으로 메시지 송수신을 관리합니다.

## OpenAMP + FreeRTOS

### OpenAMP의 RTOS 통합 이점
- **비동기 처리**: 태스크 기반 메시지 처리
- **우선순위 관리**: 메시지 처리 우선순위 설정
- **타이밍 제어**: 정확한 주기적 통신
- **리소스 보호**: FreeRTOS 동기화 기능 활용

## 시스템 아키텍처

### RTOS 기반 OpenAMP 구조
```
┌──────────────────────────────────────────────────────────────┐
│                        Cortex-M7                             │
│  ┌─────────────────────────────────────────────────────┐     │
│  │                    FreeRTOS                         │     │
│  │  ┌─────────────┐  ┌─────────────┐  ┌────────────┐   │     │
│  │  │  Sender     │  │  Receiver   │  │  Monitor   │   │     │
│  │  │  Task       │  │  Task       │  │  Task      │   │     │
│  │  └──────┬──────┘  └──────┬──────┘  └─────┬──────┘   │     │
│  │         │                │               │          │     │
│  │  ┌──────▼────────────────▼───────────────▼──────┐   │     │
│  │  │              OpenAMP Layer                   │   │     │
│  │  │  (RPMsg Endpoint, VirtIO)                    │   │     │
│  │  └──────────────────┬───────────────────────────┘   │     │
│  └─────────────────────┼───────────────────────────────┘     │
└────────────────────────┼─────────────────────────────────────┘
                         │
                         │  Shared Memory (D3 SRAM)
                         │  + HSEM Interrupt
                         │
┌────────────────────────┼─────────────────────────────────────┐
│                        │                                     │
│  ┌─────────────────────┼───────────────────────────────┐     │
│  │                    FreeRTOS                         │     │
│  │  ┌──────────────────▼───────────────────────────┐   │     │
│  │  │              OpenAMP Layer                   │   │     │
│  │  └──────┬────────────────────────────────┬──────┘   │     │
│  │         │                                │          │     │
│  │  ┌──────▼──────┐              ┌──────────▼───────┐  │     │
│  │  │  Sender     │              │  Receiver        │  │     │
│  │  │  Task       │              │  Task            │  │     │
│  │  └─────────────┘              └──────────────────┘  │     │
│  └─────────────────────────────────────────────────────┘     │
│                        Cortex-M4                             │
└──────────────────────────────────────────────────────────────┘
```

## 메모리 구성

### 공유 메모리 레이아웃
```c
// D3 SRAM (64KB)
#define RPMSG_VIRTIO_SHMEM_BASE  0x38000000
#define RPMSG_VIRTIO_SHMEM_SIZE  0x10000    // 64KB

// 리소스 테이블
#define RESOURCE_TABLE_ADDR      0x38000000

// VirtIO 버퍼
#define VRING_TX_ADDR            0x38000400
#define VRING_RX_ADDR            0x38008400
#define VRING_SIZE               16
#define VRING_ALIGNMENT          32
```

### FreeRTOS 힙 구성
```c
// CM7 FreeRTOS 설정
#define configTOTAL_HEAP_SIZE    (64 * 1024)  // 64KB DTCM

// CM4 FreeRTOS 설정
#define configTOTAL_HEAP_SIZE    (32 * 1024)  // 32KB D2 SRAM
```

## 코드 구현

### 1. CM7 메인 (Master)
```c
#include "FreeRTOS.h"
#include "task.h"
#include "semphr.h"
#include "openamp/open_amp.h"

// 태스크 핸들
TaskHandle_t xSenderTaskHandle;
TaskHandle_t xReceiverTaskHandle;
TaskHandle_t xMonitorTaskHandle;

// 동기화 객체
SemaphoreHandle_t xRPMsgRxSemaphore;
SemaphoreHandle_t xRPMsgMutex;

// 메시지 통계
volatile uint32_t tx_count = 0;
volatile uint32_t rx_count = 0;

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

  // 동기화 객체 생성
  xRPMsgRxSemaphore = xSemaphoreCreateBinary();
  xRPMsgMutex = xSemaphoreCreateMutex();

  // OpenAMP 초기화 (Master)
  MX_OPENAMP_Init(RPMSG_MASTER, NULL);

  // CM4 부팅
  HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);

  // RPMsg 채널 생성
  OPENAMP_create_endpoint(&rp_endpoint, "rpmsg-client-rtos",
                          RPMSG_ADDR_ANY, rpmsg_recv_callback, NULL);

  // FreeRTOS 태스크 생성
  xTaskCreate(vSenderTask, "Sender", 512, NULL, 3, &xSenderTaskHandle);
  xTaskCreate(vReceiverTask, "Receiver", 512, NULL, 4, &xReceiverTaskHandle);
  xTaskCreate(vMonitorTask, "Monitor", 256, NULL, 2, &xMonitorTaskHandle);

  // 스케줄러 시작
  vTaskStartScheduler();

  // 여기까지 오면 안됨
  while (1);
}
```

### 2. CM7 송신 태스크
```c
void vSenderTask(void *pvParameters)
{
  uint32_t message_counter = 0;
  RPMsg_TypeDef msg;

  // 채널이 준비될 때까지 대기
  while (!OPENAMP_endpoint_is_ready(&rp_endpoint))
  {
    vTaskDelay(pdMS_TO_TICKS(10));
  }

  for (;;)
  {
    // 메시지 준비
    msg.cmd = CMD_PING;
    msg.counter = message_counter++;
    msg.timestamp = HAL_GetTick();

    // RPMsg 전송 (뮤텍스 보호)
    if (xSemaphoreTake(xRPMsgMutex, pdMS_TO_TICKS(100)) == pdTRUE)
    {
      if (OPENAMP_send(&rp_endpoint, &msg, sizeof(msg)) >= 0)
      {
        tx_count++;
        BSP_LED_Toggle(LED1);
      }
      xSemaphoreGive(xRPMsgMutex);
    }

    // 2초마다 전송
    vTaskDelay(pdMS_TO_TICKS(2000));
  }
}
```

### 3. CM7 수신 태스크
```c
void vReceiverTask(void *pvParameters)
{
  for (;;)
  {
    // 수신 세마포어 대기
    if (xSemaphoreTake(xRPMsgRxSemaphore, portMAX_DELAY) == pdTRUE)
    {
      // OpenAMP 메시지 처리
      if (xSemaphoreTake(xRPMsgMutex, pdMS_TO_TICKS(100)) == pdTRUE)
      {
        OPENAMP_check_for_message();
        xSemaphoreGive(xRPMsgMutex);
      }
    }
  }
}
```

### 4. CM7 수신 콜백
```c
int rpmsg_recv_callback(struct rpmsg_endpoint *ept, void *data,
                        size_t len, uint32_t src, void *priv)
{
  RPMsg_TypeDef *msg = (RPMsg_TypeDef *)data;

  // 수신 처리
  if (msg->cmd == CMD_PONG)
  {
    rx_count++;

    // 왕복 시간 계산
    uint32_t rtt = HAL_GetTick() - msg->timestamp;
    printf("Pong received: counter=%lu, RTT=%lu ms\n", msg->counter, rtt);

    BSP_LED_Toggle(LED2);
  }

  return 0;
}
```

### 5. CM7 모니터 태스크
```c
void vMonitorTask(void *pvParameters)
{
  for (;;)
  {
    // 통계 출력
    printf("\n--- RPMsg Statistics ---\n");
    printf("TX Count: %lu\n", tx_count);
    printf("RX Count: %lu\n", rx_count);
    printf("Success Rate: %.1f%%\n", (float)rx_count * 100 / tx_count);

    // 태스크 상태
    printf("\n--- Task Status ---\n");
    printf("Free Heap: %u bytes\n", xPortGetFreeHeapSize());

    // 5초마다 출력
    vTaskDelay(pdMS_TO_TICKS(5000));
  }
}
```

### 6. HSEM 인터럽트 핸들러 (CM7)
```c
void HSEM1_IRQHandler(void)
{
  HAL_HSEM_IRQHandler();

  // FreeRTOS 세마포어 해제
  BaseType_t xHigherPriorityTaskWoken = pdFALSE;
  xSemaphoreGiveFromISR(xRPMsgRxSemaphore, &xHigherPriorityTaskWoken);
  portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

### 7. CM4 메인 (Remote)
```c
#include "FreeRTOS.h"
#include "task.h"
#include "semphr.h"
#include "openamp/open_amp.h"

TaskHandle_t xProcessTaskHandle;
SemaphoreHandle_t xMsgRxSemaphore;

int main(void)
{
  // HSEM으로 CM7 준비 대기
  __HAL_RCC_HSEM_CLK_ENABLE();

  // HAL 초기화
  HAL_Init();

  // LED 초기화
  BSP_LED_Init(LED3);
  BSP_LED_Init(LED4);

  // 세마포어 생성
  xMsgRxSemaphore = xSemaphoreCreateBinary();

  // OpenAMP 초기화 (Remote)
  MX_OPENAMP_Init(RPMSG_REMOTE, NULL);

  // RPMsg 엔드포인트 생성
  OPENAMP_create_endpoint(&rp_endpoint, "rpmsg-client-rtos",
                          RPMSG_ADDR_ANY, cm4_rpmsg_callback, NULL);

  // FreeRTOS 태스크 생성
  xTaskCreate(vProcessTask, "Process", 512, NULL, 3, &xProcessTaskHandle);

  // 스케줄러 시작
  vTaskStartScheduler();

  while (1);
}
```

### 8. CM4 처리 태스크
```c
void vProcessTask(void *pvParameters)
{
  for (;;)
  {
    // 메시지 대기
    if (xSemaphoreTake(xMsgRxSemaphore, portMAX_DELAY) == pdTRUE)
    {
      // OpenAMP 메시지 처리
      OPENAMP_check_for_message();
    }
  }
}

int cm4_rpmsg_callback(struct rpmsg_endpoint *ept, void *data,
                       size_t len, uint32_t src, void *priv)
{
  RPMsg_TypeDef *rx_msg = (RPMsg_TypeDef *)data;

  if (rx_msg->cmd == CMD_PING)
  {
    // Pong 응답 준비
    RPMsg_TypeDef tx_msg;
    tx_msg.cmd = CMD_PONG;
    tx_msg.counter = rx_msg->counter;
    tx_msg.timestamp = rx_msg->timestamp;  // 원래 타임스탬프 유지

    // 응답 전송
    OPENAMP_send(ept, &tx_msg, sizeof(tx_msg));

    BSP_LED_Toggle(LED3);
  }

  return 0;
}

// HSEM 인터럽트 핸들러 (CM4)
void HSEM2_IRQHandler(void)
{
  HAL_HSEM_IRQHandler();

  BaseType_t xHigherPriorityTaskWoken = pdFALSE;
  xSemaphoreGiveFromISR(xMsgRxSemaphore, &xHigherPriorityTaskWoken);
  portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

## 메시지 정의

### RPMsg 구조체
```c
typedef enum {
  CMD_PING = 0x01,
  CMD_PONG = 0x02,
  CMD_DATA = 0x03,
  CMD_CONFIG = 0x04,
  CMD_STATUS = 0x05
} RPMsgCommand_TypeDef;

typedef struct {
  uint8_t cmd;
  uint8_t reserved;
  uint16_t length;
  uint32_t counter;
  uint32_t timestamp;
  uint8_t data[48];  // 가변 데이터
} __attribute__((packed)) RPMsg_TypeDef;

#define RPMSG_MAX_SIZE  sizeof(RPMsg_TypeDef)
```

## 고급 기능

### 1. 메시지 큐 기반 통신
```c
QueueHandle_t xTxQueue;
QueueHandle_t xRxQueue;

void vQueueSenderTask(void *pvParameters)
{
  RPMsg_TypeDef msg;

  for (;;)
  {
    // 큐에서 메시지 대기
    if (xQueueReceive(xTxQueue, &msg, portMAX_DELAY) == pdTRUE)
    {
      // RPMsg 전송
      if (xSemaphoreTake(xRPMsgMutex, portMAX_DELAY) == pdTRUE)
      {
        OPENAMP_send(&rp_endpoint, &msg, sizeof(msg));
        xSemaphoreGive(xRPMsgMutex);
      }
    }
  }
}

// 애플리케이션에서 메시지 전송
void SendMessage(RPMsg_TypeDef *msg)
{
  xQueueSend(xTxQueue, msg, pdMS_TO_TICKS(100));
}
```

### 2. 다중 채널 지원
```c
// 여러 RPMsg 채널 정의
struct rpmsg_endpoint ep_control;
struct rpmsg_endpoint ep_data;
struct rpmsg_endpoint ep_debug;

void CreateMultipleChannels(void)
{
  OPENAMP_create_endpoint(&ep_control, "rpmsg-control",
                          RPMSG_ADDR_ANY, control_callback, NULL);
  OPENAMP_create_endpoint(&ep_data, "rpmsg-data",
                          RPMSG_ADDR_ANY, data_callback, NULL);
  OPENAMP_create_endpoint(&ep_debug, "rpmsg-debug",
                          RPMSG_ADDR_ANY, debug_callback, NULL);
}
```

### 3. 흐름 제어
```c
// 버퍼 사용량 기반 흐름 제어
#define BUFFER_HIGH_WATERMARK  80
#define BUFFER_LOW_WATERMARK   20

volatile uint8_t flow_control_enabled = 0;

void FlowControl_Check(void)
{
  uint32_t buffer_usage = GetVirtIOBufferUsage();

  if (buffer_usage > BUFFER_HIGH_WATERMARK)
  {
    flow_control_enabled = 1;
    // CM4에게 정지 요청
    SendFlowControlMessage(FLOW_STOP);
  }
  else if (buffer_usage < BUFFER_LOW_WATERMARK && flow_control_enabled)
  {
    flow_control_enabled = 0;
    // CM4에게 재개 요청
    SendFlowControlMessage(FLOW_RESUME);
  }
}
```

### 4. 에러 처리
```c
typedef enum {
  RPMSG_ERR_NONE = 0,
  RPMSG_ERR_NO_MEM,
  RPMSG_ERR_TIMEOUT,
  RPMSG_ERR_INVALID_MSG,
  RPMSG_ERR_CHANNEL_CLOSED
} RPMsgError_TypeDef;

RPMsgError_TypeDef SendWithRetry(RPMsg_TypeDef *msg, uint8_t max_retries)
{
  for (int i = 0; i < max_retries; i++)
  {
    int result = OPENAMP_send(&rp_endpoint, msg, sizeof(*msg));

    if (result >= 0)
    {
      return RPMSG_ERR_NONE;
    }
    else if (result == -ENOMEM)
    {
      // 버퍼 부족 - 잠시 대기 후 재시도
      vTaskDelay(pdMS_TO_TICKS(10));
    }
    else
    {
      return RPMSG_ERR_CHANNEL_CLOSED;
    }
  }

  return RPMSG_ERR_TIMEOUT;
}
```

## FreeRTOS 설정

### FreeRTOSConfig.h (CM7)
```c
#define configUSE_PREEMPTION              1
#define configCPU_CLOCK_HZ                400000000
#define configTICK_RATE_HZ                1000
#define configMAX_PRIORITIES              7
#define configMINIMAL_STACK_SIZE          128
#define configTOTAL_HEAP_SIZE             (64 * 1024)
#define configUSE_MUTEXES                 1
#define configUSE_COUNTING_SEMAPHORES     1
#define configUSE_QUEUE_SETS              1
#define configUSE_TASK_NOTIFICATIONS      1

// 인터럽트 우선순위
#define configLIBRARY_LOWEST_INTERRUPT_PRIORITY         15
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY    5
```

### OpenAMP 설정
```c
// openamp_conf.h
#define MAILBOX_HSEM_ID_TX    0
#define MAILBOX_HSEM_ID_RX    1

#define VRING_TX_ADDRESS      0x38000400
#define VRING_RX_ADDRESS      0x38008400
#define VRING_BUFF_ADDRESS    0x38010400
#define VRING_ALIGNMENT       32
#define VRING_NUM_BUFFS       16
```

## 응용 예제

### 1. 센서 데이터 스트리밍
```c
// CM4: 센서 데이터 수집 및 전송
void vSensorTask(void *pvParameters)
{
  SensorData_TypeDef sensor_data;
  RPMsg_TypeDef msg;

  for (;;)
  {
    // 센서 읽기
    ReadAccelerometer(&sensor_data.accel);
    ReadGyroscope(&sensor_data.gyro);

    // 메시지 구성
    msg.cmd = CMD_DATA;
    msg.timestamp = HAL_GetTick();
    memcpy(msg.data, &sensor_data, sizeof(sensor_data));

    // CM7으로 전송
    OPENAMP_send(&rp_endpoint, &msg, sizeof(msg));

    // 100Hz 샘플링
    vTaskDelay(pdMS_TO_TICKS(10));
  }
}

// CM7: 센서 데이터 처리
int sensor_data_callback(struct rpmsg_endpoint *ept, void *data,
                         size_t len, uint32_t src, void *priv)
{
  RPMsg_TypeDef *msg = (RPMsg_TypeDef *)data;
  SensorData_TypeDef *sensor = (SensorData_TypeDef *)msg->data;

  // 데이터 처리 (필터링, 융합 등)
  ProcessSensorData(sensor);

  return 0;
}
```

### 2. 원격 함수 호출 (RPC)
```c
// RPC 요청 구조체
typedef struct {
  uint8_t function_id;
  uint8_t param_count;
  uint32_t params[4];
  uint32_t return_value;
} RPCRequest_TypeDef;

// CM7: RPC 요청
uint32_t RPC_Call(uint8_t func_id, uint32_t *params, uint8_t count)
{
  RPCRequest_TypeDef req;
  req.function_id = func_id;
  req.param_count = count;
  memcpy(req.params, params, count * sizeof(uint32_t));

  // 요청 전송
  OPENAMP_send(&rpc_endpoint, &req, sizeof(req));

  // 응답 대기
  xSemaphoreTake(xRPCResponseSem, pdMS_TO_TICKS(1000));

  return rpc_response.return_value;
}

// CM4: RPC 처리
int rpc_handler(struct rpmsg_endpoint *ept, void *data,
                size_t len, uint32_t src, void *priv)
{
  RPCRequest_TypeDef *req = (RPCRequest_TypeDef *)data;

  // 함수 실행
  switch (req->function_id)
  {
    case RPC_READ_ADC:
      req->return_value = HAL_ADC_GetValue(&hadc1);
      break;

    case RPC_SET_PWM:
      __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, req->params[0]);
      req->return_value = 0;
      break;
  }

  // 응답 전송
  OPENAMP_send(ept, req, sizeof(*req));

  return 0;
}
```

## 트러블슈팅

### 메시지가 수신되지 않는 경우

1. **채널 상태 확인**: OPENAMP_endpoint_is_ready()
2. **HSEM 인터럽트**: 올바르게 설정 및 처리
3. **버퍼 오버플로우**: VirtIO 버퍼 크기 확인

### 태스크가 블로킹되는 경우

1. **데드락**: 뮤텍스 순서 확인
2. **세마포어 누수**: Take/Give 쌍 확인
3. **우선순위 역전**: 우선순위 상속 사용

### 성능 문제

1. **태스크 우선순위**: 수신 태스크 우선순위 높게
2. **스택 크기**: 충분한 스택 할당
3. **폴링 빈도**: OPENAMP_check_for_message() 호출 빈도

## 성능 고려사항

### 메시지 처리량
- **최대**: 수만 msg/sec
- **실제**: 태스크 우선순위 및 처리 시간에 따라 다름
- **최적화**: 배치 처리, DMA 활용

### CPU 사용률
- **폴링 모드**: 높은 CPU 사용
- **인터럽트 모드**: 낮은 CPU 사용, 추천

### 메모리 사용
- **VirtIO 버퍼**: 약 16KB
- **FreeRTOS 태스크**: 스택 크기에 따라
- **힙**: OpenAMP 및 FreeRTOS 힙

## 참고 자료

- OpenAMP GitHub: https://github.com/OpenAMP/open-amp
- FreeRTOS 공식 문서: https://www.freertos.org/
- AN5617: Asymmetric multiprocessing with OpenAMP framework
- 예제: `STM32H745I-DISCO/Applications/OpenAMP/OpenAMP_RTOS_PingPong`
