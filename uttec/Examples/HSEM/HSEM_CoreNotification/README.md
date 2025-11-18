# HSEM_CoreNotification 예제

## 개요

이 예제는 STM32H745I-DISCO의 하드웨어 세마포어(HSEM) 인터럽트를 사용하여 Cortex-M7과 Cortex-M4 코어 간의 이벤트 알림 및 메시지 전달 메커니즘을 구현합니다. 폴링 방식이 아닌 인터럽트 기반 통신으로 효율적인 코어 간 통신을 학습할 수 있습니다.

## 하드웨어 요구사항

- **보드**: STM32H745I-DISCO
- **버튼**: USER 버튼 (Blue button)
- **LED**:
  - LED1 (녹색): CM7 메시지 수신 표시
  - LED2 (주황색): CM4 메시지 수신 표시
  - LED3 (빨간색): 통신 오류 표시
  - LED4 (파란색): 시스템 활성 상태

## 주요 기능

### 1. HSEM 인터럽트 메커니즘
- **마스크된 인터럽트**: 각 세마포어에 대해 개별적으로 인터럽트 활성화
- **자동 클리어**: 세마포어 해제 시 인터럽트 플래그 자동 클리어
- **저전력 대기**: WFI 명령으로 이벤트 대기 중 전력 소비 최소화

### 2. 메시지 전달 시스템
- **비동기 통신**: 송신 코어는 차단되지 않고 즉시 반환
- **큐 기반 버퍼링**: 여러 메시지를 순차적으로 처리
- **우선순위 지원**: 긴급 메시지 우선 처리

## 소프트웨어 설명

### 메시지 구조 정의

```c
/* message.h */

/* 메시지 타입 정의 */
typedef enum
{
    MSG_TYPE_DATA = 0,      /* 일반 데이터 */
    MSG_TYPE_COMMAND,       /* 명령 */
    MSG_TYPE_EVENT,         /* 이벤트 알림 */
    MSG_TYPE_ACK,          /* 응답 확인 */
    MSG_TYPE_ERROR         /* 오류 알림 */
} MessageType_t;

/* 메시지 우선순위 */
typedef enum
{
    MSG_PRIORITY_LOW = 0,
    MSG_PRIORITY_NORMAL,
    MSG_PRIORITY_HIGH,
    MSG_PRIORITY_URGENT
} MessagePriority_t;

/* 메시지 구조체 */
typedef struct
{
    MessageType_t type;
    MessagePriority_t priority;
    uint32_t sender_id;
    uint32_t timestamp;
    uint32_t sequence_number;
    uint32_t data_length;
    uint8_t data[256];
} Message_t;

/* 메시지 큐 구조체 */
#define MSG_QUEUE_SIZE  16

typedef struct
{
    Message_t messages[MSG_QUEUE_SIZE];
    volatile uint32_t head;
    volatile uint32_t tail;
    volatile uint32_t count;
    volatile uint32_t overflow_count;
} MessageQueue_t;

/* 공유 메모리에 배치 */
#define SHARED_MEM_BASE  0x38000000

#if defined ( __GNUC__ )
__attribute__((section(".shared_data")))
#endif
MessageQueue_t cm7_to_cm4_queue;

#if defined ( __GNUC__ )
__attribute__((section(".shared_data")))
#endif
MessageQueue_t cm4_to_cm7_queue;

/* 세마포어 ID 정의 */
#define HSEM_ID_CM7_TO_CM4    0  /* CM7 → CM4 알림 */
#define HSEM_ID_CM4_TO_CM7    1  /* CM4 → CM7 알림 */
#define HSEM_ID_QUEUE_LOCK    2  /* 큐 접근 보호 */

/* 프로세스 ID */
#define PROCESS_ID_CM7   0
#define PROCESS_ID_CM4   1
```

### 메시지 큐 관리 함수

```c
/* message_queue.c */

/**
  * @brief  메시지 큐 초기화
  */
void MessageQueue_Init(MessageQueue_t *queue)
{
    queue->head = 0;
    queue->tail = 0;
    queue->count = 0;
    queue->overflow_count = 0;
}

/**
  * @brief  메시지를 큐에 추가
  * @param  queue: 메시지 큐 포인터
  * @param  msg: 추가할 메시지
  * @retval HAL_OK: 성공, HAL_ERROR: 큐가 가득 찼음
  */
HAL_StatusTypeDef MessageQueue_Enqueue(MessageQueue_t *queue, Message_t *msg)
{
    /* 큐 접근 보호 */
    if (HAL_HSEM_Take(HSEM_ID_QUEUE_LOCK, PROCESS_ID_CM7) != HAL_OK)
    {
        return HAL_BUSY;
    }

    /* 큐가 가득 찼는지 확인 */
    if (queue->count >= MSG_QUEUE_SIZE)
    {
        queue->overflow_count++;
        HAL_HSEM_Release(HSEM_ID_QUEUE_LOCK, PROCESS_ID_CM7);
        return HAL_ERROR;
    }

    /* 메시지 복사 */
    memcpy(&queue->messages[queue->tail], msg, sizeof(Message_t));

    /* 큐 포인터 업데이트 */
    queue->tail = (queue->tail + 1) % MSG_QUEUE_SIZE;
    queue->count++;

    /* 큐 잠금 해제 */
    HAL_HSEM_Release(HSEM_ID_QUEUE_LOCK, PROCESS_ID_CM7);

    return HAL_OK;
}

/**
  * @brief  큐에서 메시지 추출
  * @param  queue: 메시지 큐 포인터
  * @param  msg: 추출된 메시지를 저장할 버퍼
  * @retval HAL_OK: 성공, HAL_ERROR: 큐가 비어있음
  */
HAL_StatusTypeDef MessageQueue_Dequeue(MessageQueue_t *queue, Message_t *msg)
{
    /* 큐 접근 보호 */
    if (HAL_HSEM_Take(HSEM_ID_QUEUE_LOCK, PROCESS_ID_CM7) != HAL_OK)
    {
        return HAL_BUSY;
    }

    /* 큐가 비어있는지 확인 */
    if (queue->count == 0)
    {
        HAL_HSEM_Release(HSEM_ID_QUEUE_LOCK, PROCESS_ID_CM7);
        return HAL_ERROR;
    }

    /* 메시지 복사 */
    memcpy(msg, &queue->messages[queue->head], sizeof(Message_t));

    /* 큐 포인터 업데이트 */
    queue->head = (queue->head + 1) % MSG_QUEUE_SIZE;
    queue->count--;

    /* 큐 잠금 해제 */
    HAL_HSEM_Release(HSEM_ID_QUEUE_LOCK, PROCESS_ID_CM7);

    return HAL_OK;
}

/**
  * @brief  큐의 메시지 개수 확인
  */
uint32_t MessageQueue_GetCount(MessageQueue_t *queue)
{
    return queue->count;
}

/**
  * @brief  큐가 비어있는지 확인
  */
uint8_t MessageQueue_IsEmpty(MessageQueue_t *queue)
{
    return (queue->count == 0);
}

/**
  * @brief  큐가 가득 찼는지 확인
  */
uint8_t MessageQueue_IsFull(MessageQueue_t *queue)
{
    return (queue->count >= MSG_QUEUE_SIZE);
}
```

### CM7 코어 구현

```c
/* CM7/main.c */

/* 전역 변수 */
volatile uint32_t message_received = 0;
volatile uint32_t message_count = 0;
Message_t rx_message;

int main(void)
{
    /* HAL 및 시스템 초기화 */
    HAL_Init();
    SystemClock_Config();

    /* MPU 설정 (공유 메모리 캐시 비활성화) */
    MPU_Config();

    /* LED 초기화 */
    BSP_LED_Init(LED1);
    BSP_LED_Init(LED2);
    BSP_LED_Init(LED3);
    BSP_LED_Init(LED4);

    /* 버튼 초기화 */
    BSP_PB_Init(BUTTON_USER, BUTTON_MODE_EXTI);

    /* HSEM 초기화 */
    HSEM_Init();

    /* 메시지 큐 초기화 */
    MessageQueue_Init(&cm7_to_cm4_queue);
    MessageQueue_Init(&cm4_to_cm7_queue);

    /* HSEM 인터럽트 활성화 */
    HSEM_EnableNotification();

    /* CM4 부팅 */
    HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);

    BSP_LED_On(LED4);  /* 시스템 준비 완료 */

    /* 메인 루프 */
    while (1)
    {
        /* 메시지 수신 확인 */
        if (message_received)
        {
            message_received = 0;

            /* 큐에서 메시지 추출 */
            if (MessageQueue_Dequeue(&cm4_to_cm7_queue, &rx_message) == HAL_OK)
            {
                /* 메시지 처리 */
                Process_Received_Message(&rx_message);

                /* LED 토글 */
                BSP_LED_Toggle(LED1);

                /* 응답 전송 (ACK) */
                Send_ACK_Message(rx_message.sequence_number);
            }
        }

        /* 주기적인 시스템 상태 LED 토글 */
        BSP_LED_Toggle(LED4);
        HAL_Delay(1000);
    }
}

/**
  * @brief  HSEM 초기화
  */
void HSEM_Init(void)
{
    /* HSEM 클록 활성화 */
    __HAL_RCC_HSEM_CLK_ENABLE();

    /* HSEM 인터럽트 우선순위 설정 */
    HAL_NVIC_SetPriority(HSEM1_IRQn, 5, 0);
    HAL_NVIC_EnableIRQ(HSEM1_IRQn);
}

/**
  * @brief  HSEM 알림 활성화
  */
void HSEM_EnableNotification(void)
{
    /* CM4 → CM7 알림 활성화 */
    HAL_HSEM_ActivateNotification(__HAL_HSEM_SEMID_TO_MASK(HSEM_ID_CM4_TO_CM7));
}

/**
  * @brief  메시지 전송 함수
  */
HAL_StatusTypeDef Send_Message_To_CM4(MessageType_t type, uint8_t *data,
                                       uint32_t length)
{
    static uint32_t sequence = 0;
    Message_t msg;

    /* 메시지 구성 */
    msg.type = type;
    msg.priority = MSG_PRIORITY_NORMAL;
    msg.sender_id = PROCESS_ID_CM7;
    msg.timestamp = HAL_GetTick();
    msg.sequence_number = sequence++;
    msg.data_length = (length > 256) ? 256 : length;

    if (data != NULL && msg.data_length > 0)
    {
        memcpy(msg.data, data, msg.data_length);
    }

    /* 큐에 메시지 추가 */
    if (MessageQueue_Enqueue(&cm7_to_cm4_queue, &msg) != HAL_OK)
    {
        return HAL_ERROR;
    }

    /* CM4에 알림 전송 */
    HAL_HSEM_FastTake(HSEM_ID_CM7_TO_CM4);
    HAL_HSEM_Release(HSEM_ID_CM7_TO_CM4, PROCESS_ID_CM7);

    return HAL_OK;
}

/**
  * @brief  ACK 메시지 전송
  */
void Send_ACK_Message(uint32_t sequence_num)
{
    Message_t ack_msg;

    ack_msg.type = MSG_TYPE_ACK;
    ack_msg.priority = MSG_PRIORITY_NORMAL;
    ack_msg.sender_id = PROCESS_ID_CM7;
    ack_msg.timestamp = HAL_GetTick();
    ack_msg.sequence_number = sequence_num;
    ack_msg.data_length = 0;

    MessageQueue_Enqueue(&cm7_to_cm4_queue, &ack_msg);

    /* 알림 전송 */
    HAL_HSEM_FastTake(HSEM_ID_CM7_TO_CM4);
    HAL_HSEM_Release(HSEM_ID_CM7_TO_CM4, PROCESS_ID_CM7);
}

/**
  * @brief  수신 메시지 처리
  */
void Process_Received_Message(Message_t *msg)
{
    printf("\n[CM7] Message Received:\n");
    printf("  Type: %d\n", msg->type);
    printf("  Sender: %lu\n", msg->sender_id);
    printf("  Sequence: %lu\n", msg->sequence_number);
    printf("  Timestamp: %lu ms\n", msg->timestamp);
    printf("  Data Length: %lu bytes\n", msg->data_length);

    switch (msg->type)
    {
        case MSG_TYPE_DATA:
            printf("  Data: ");
            for (uint32_t i = 0; i < msg->data_length; i++)
            {
                printf("%02X ", msg->data[i]);
            }
            printf("\n");
            break;

        case MSG_TYPE_COMMAND:
            Execute_Command(msg);
            break;

        case MSG_TYPE_EVENT:
            Handle_Event(msg);
            break;

        case MSG_TYPE_ACK:
            printf("  ACK received for sequence %lu\n", msg->sequence_number);
            break;

        case MSG_TYPE_ERROR:
            printf("  Error code: 0x%02X\n", msg->data[0]);
            BSP_LED_On(LED3);
            break;

        default:
            printf("  Unknown message type\n");
            break;
    }
}

/**
  * @brief  HSEM 인터럽트 핸들러
  */
void HSEM1_IRQHandler(void)
{
    uint32_t status_reg;

    /* 인터럽트 상태 레지스터 읽기 */
    status_reg = HSEM->MISR;

    /* CM4 → CM7 알림 확인 */
    if (status_reg & __HAL_HSEM_SEMID_TO_MASK(HSEM_ID_CM4_TO_CM7))
    {
        /* 인터럽트 플래그 클리어 */
        HAL_HSEM_ReleaseCallback(HSEM_ID_CM4_TO_CM7);

        /* 메시지 수신 플래그 설정 */
        message_received = 1;
        message_count++;
    }
}

/**
  * @brief  HSEM 해제 콜백 (HAL에서 호출)
  */
void HAL_HSEM_FreeCallback(uint32_t SemMask)
{
    /* 알림 재활성화 */
    HAL_HSEM_ActivateNotification(SemMask);
}

/**
  * @brief  버튼 인터럽트 콜백
  */
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == BUTTON_USER_PIN)
    {
        /* 테스트 데이터 전송 */
        uint8_t test_data[] = "Hello from CM7!";
        Send_Message_To_CM4(MSG_TYPE_DATA, test_data, sizeof(test_data));

        BSP_LED_Toggle(LED1);
    }
}
```

### CM4 코어 구현

```c
/* CM4/main.c */

volatile uint32_t message_received = 0;
Message_t rx_message;

int main(void)
{
    /* HAL 초기화 */
    HAL_Init();

    /* MPU 설정 */
    MPU_Config();

    /* HSEM 초기화 */
    HSEM_Init();

    /* HSEM 알림 활성화 */
    HSEM_EnableNotification();

    /* 메인 루프 */
    while (1)
    {
        /* 메시지 대기 (저전력 모드) */
        if (MessageQueue_IsEmpty(&cm7_to_cm4_queue))
        {
            __WFI();  /* 인터럽트 대기 */
        }

        /* 메시지 수신 처리 */
        if (message_received)
        {
            message_received = 0;

            while (!MessageQueue_IsEmpty(&cm7_to_cm4_queue))
            {
                if (MessageQueue_Dequeue(&cm7_to_cm4_queue, &rx_message) == HAL_OK)
                {
                    /* 메시지 처리 */
                    Process_Received_Message(&rx_message);

                    /* LED 토글 */
                    BSP_LED_Toggle(LED2);

                    /* ACK 전송 */
                    Send_ACK_Message(rx_message.sequence_number);
                }
            }
        }
    }
}

/**
  * @brief  HSEM 초기화 (CM4)
  */
void HSEM_Init(void)
{
    __HAL_RCC_HSEM_CLK_ENABLE();

    HAL_NVIC_SetPriority(HSEM2_IRQn, 5, 0);
    HAL_NVIC_EnableIRQ(HSEM2_IRQn);
}

/**
  * @brief  HSEM 알림 활성화 (CM4)
  */
void HSEM_EnableNotification(void)
{
    /* CM7 → CM4 알림 활성화 */
    HAL_HSEM_ActivateNotification(__HAL_HSEM_SEMID_TO_MASK(HSEM_ID_CM7_TO_CM4));
}

/**
  * @brief  CM4에서 CM7로 메시지 전송
  */
HAL_StatusTypeDef Send_Message_To_CM7(MessageType_t type, uint8_t *data,
                                       uint32_t length)
{
    static uint32_t sequence = 0;
    Message_t msg;

    msg.type = type;
    msg.priority = MSG_PRIORITY_NORMAL;
    msg.sender_id = PROCESS_ID_CM4;
    msg.timestamp = HAL_GetTick();
    msg.sequence_number = sequence++;
    msg.data_length = (length > 256) ? 256 : length;

    if (data != NULL && msg.data_length > 0)
    {
        memcpy(msg.data, data, msg.data_length);
    }

    if (MessageQueue_Enqueue(&cm4_to_cm7_queue, &msg) != HAL_OK)
    {
        return HAL_ERROR;
    }

    /* CM7에 알림 */
    HAL_HSEM_FastTake(HSEM_ID_CM4_TO_CM7);
    HAL_HSEM_Release(HSEM_ID_CM4_TO_CM7, PROCESS_ID_CM4);

    return HAL_OK;
}

/**
  * @brief  HSEM2 인터럽트 핸들러 (CM4)
  */
void HSEM2_IRQHandler(void)
{
    uint32_t status_reg = HSEM->MISR;

    if (status_reg & __HAL_HSEM_SEMID_TO_MASK(HSEM_ID_CM7_TO_CM4))
    {
        HAL_HSEM_ReleaseCallback(HSEM_ID_CM7_TO_CM4);
        message_received = 1;
    }
}

/**
  * @brief  수신 메시지 처리 (CM4)
  */
void Process_Received_Message(Message_t *msg)
{
    switch (msg->type)
    {
        case MSG_TYPE_DATA:
            /* 데이터 처리 */
            Process_Data(msg->data, msg->data_length);
            break;

        case MSG_TYPE_COMMAND:
            /* 명령 실행 */
            Execute_Command_CM4(msg);
            break;

        case MSG_TYPE_EVENT:
            /* 이벤트 처리 */
            Handle_Event_CM4(msg);
            break;

        default:
            break;
    }
}
```

## 고급 기능 구현

### 1. 우선순위 기반 메시지 처리

```c
/* priority_queue.c */

typedef struct
{
    Message_t messages[MSG_QUEUE_SIZE];
    uint32_t count;
} PriorityQueue_t;

/**
  * @brief  우선순위 순서로 메시지 삽입
  */
HAL_StatusTypeDef PriorityQueue_Insert(PriorityQueue_t *queue, Message_t *msg)
{
    if (queue->count >= MSG_QUEUE_SIZE)
    {
        return HAL_ERROR;
    }

    /* 삽입 위치 찾기 (우선순위 기반) */
    uint32_t insert_pos = queue->count;

    for (uint32_t i = 0; i < queue->count; i++)
    {
        if (msg->priority > queue->messages[i].priority)
        {
            insert_pos = i;
            break;
        }
    }

    /* 기존 메시지를 뒤로 이동 */
    for (uint32_t i = queue->count; i > insert_pos; i--)
    {
        memcpy(&queue->messages[i], &queue->messages[i-1], sizeof(Message_t));
    }

    /* 새 메시지 삽입 */
    memcpy(&queue->messages[insert_pos], msg, sizeof(Message_t));
    queue->count++;

    return HAL_OK;
}

/**
  * @brief  가장 높은 우선순위 메시지 추출
  */
HAL_StatusTypeDef PriorityQueue_Pop(PriorityQueue_t *queue, Message_t *msg)
{
    if (queue->count == 0)
    {
        return HAL_ERROR;
    }

    /* 첫 번째 메시지 (가장 높은 우선순위) 복사 */
    memcpy(msg, &queue->messages[0], sizeof(Message_t));

    /* 나머지 메시지를 앞으로 이동 */
    for (uint32_t i = 0; i < queue->count - 1; i++)
    {
        memcpy(&queue->messages[i], &queue->messages[i+1], sizeof(Message_t));
    }

    queue->count--;

    return HAL_OK;
}
```

### 2. 브로드캐스트 메시지

```c
/* broadcast.c */

typedef struct
{
    uint32_t subscribers[8];
    uint32_t subscriber_count;
} BroadcastChannel_t;

BroadcastChannel_t channels[4];

/**
  * @brief  채널 구독
  */
void Subscribe_To_Channel(uint32_t channel_id, uint32_t subscriber_id)
{
    if (channel_id >= 4 || channels[channel_id].subscriber_count >= 8)
    {
        return;
    }

    channels[channel_id].subscribers[channels[channel_id].subscriber_count++] = subscriber_id;
}

/**
  * @brief  브로드캐스트 메시지 전송
  */
void Broadcast_Message(uint32_t channel_id, Message_t *msg)
{
    if (channel_id >= 4)
    {
        return;
    }

    for (uint32_t i = 0; i < channels[channel_id].subscriber_count; i++)
    {
        /* 각 구독자에게 메시지 전송 */
        Send_Message_To_Subscriber(channels[channel_id].subscribers[i], msg);
    }
}
```

### 3. 타임아웃 및 재전송 메커니즘

```c
/* reliable_messaging.c */

typedef struct
{
    Message_t message;
    uint32_t send_timestamp;
    uint32_t retry_count;
    uint8_t ack_received;
} PendingMessage_t;

#define MAX_PENDING_MESSAGES  8
#define MESSAGE_TIMEOUT_MS    1000
#define MAX_RETRIES          3

PendingMessage_t pending_messages[MAX_PENDING_MESSAGES];

/**
  * @brief  신뢰성 있는 메시지 전송
  */
HAL_StatusTypeDef Send_Reliable_Message(Message_t *msg)
{
    /* 빈 슬롯 찾기 */
    for (uint32_t i = 0; i < MAX_PENDING_MESSAGES; i++)
    {
        if (pending_messages[i].ack_received)
        {
            /* 메시지 저장 */
            memcpy(&pending_messages[i].message, msg, sizeof(Message_t));
            pending_messages[i].send_timestamp = HAL_GetTick();
            pending_messages[i].retry_count = 0;
            pending_messages[i].ack_received = 0;

            /* 전송 */
            Send_Message_To_CM4(msg->type, msg->data, msg->data_length);

            return HAL_OK;
        }
    }

    return HAL_ERROR;  /* 모든 슬롯이 사용 중 */
}

/**
  * @brief  재전송 타이머 (주기적으로 호출)
  */
void Check_Retransmission(void)
{
    uint32_t current_time = HAL_GetTick();

    for (uint32_t i = 0; i < MAX_PENDING_MESSAGES; i++)
    {
        if (!pending_messages[i].ack_received)
        {
            /* 타임아웃 확인 */
            if ((current_time - pending_messages[i].send_timestamp) > MESSAGE_TIMEOUT_MS)
            {
                if (pending_messages[i].retry_count < MAX_RETRIES)
                {
                    /* 재전송 */
                    Send_Message_To_CM4(pending_messages[i].message.type,
                                       pending_messages[i].message.data,
                                       pending_messages[i].message.data_length);

                    pending_messages[i].send_timestamp = current_time;
                    pending_messages[i].retry_count++;
                }
                else
                {
                    /* 최대 재시도 횟수 초과 */
                    printf("Message %lu failed after %d retries\n",
                           pending_messages[i].message.sequence_number,
                           MAX_RETRIES);

                    pending_messages[i].ack_received = 1;  /* 슬롯 해제 */
                }
            }
        }
    }
}

/**
  * @brief  ACK 수신 처리
  */
void Handle_ACK(uint32_t sequence_number)
{
    for (uint32_t i = 0; i < MAX_PENDING_MESSAGES; i++)
    {
        if (pending_messages[i].message.sequence_number == sequence_number)
        {
            pending_messages[i].ack_received = 1;
            printf("ACK received for message %lu (retries: %lu)\n",
                   sequence_number, pending_messages[i].retry_count);
            break;
        }
    }
}
```

## 성능 측정 및 최적화

### 1. 메시지 전송 지연 시간 측정

```c
/* performance.c */

typedef struct
{
    uint32_t send_time;
    uint32_t receive_time;
    uint32_t processing_time;
    uint32_t total_latency;
} MessageLatency_t;

#define LATENCY_SAMPLES  100
MessageLatency_t latency_history[LATENCY_SAMPLES];
uint32_t latency_index = 0;

/**
  * @brief  지연 시간 측정 시작
  */
void Start_Latency_Measurement(Message_t *msg)
{
    msg->timestamp = DWT->CYCCNT;  /* CPU 사이클 카운터 사용 */
}

/**
  * @brief  지연 시간 측정 종료
  */
void End_Latency_Measurement(Message_t *msg)
{
    uint32_t end_time = DWT->CYCCNT;
    uint32_t latency_cycles = end_time - msg->timestamp;

    /* 사이클을 마이크로초로 변환 (480MHz 기준) */
    uint32_t latency_us = latency_cycles / 480;

    /* 기록 저장 */
    latency_history[latency_index].total_latency = latency_us;
    latency_index = (latency_index + 1) % LATENCY_SAMPLES;

    printf("Message latency: %lu us\n", latency_us);
}

/**
  * @brief  평균 지연 시간 계산
  */
uint32_t Calculate_Average_Latency(void)
{
    uint32_t sum = 0;

    for (uint32_t i = 0; i < LATENCY_SAMPLES; i++)
    {
        sum += latency_history[i].total_latency;
    }

    return sum / LATENCY_SAMPLES;
}

/**
  * @brief  DWT 사이클 카운터 초기화
  */
void DWT_Init(void)
{
    CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
    DWT->CYCCNT = 0;
    DWT->CTRL |= DWT_CTRL_CYCCNTENA_Msk;
}
```

### 2. 처리량(Throughput) 측정

```c
/* throughput.c */

typedef struct
{
    uint32_t messages_sent;
    uint32_t messages_received;
    uint32_t bytes_sent;
    uint32_t bytes_received;
    uint32_t start_time;
    uint32_t duration_ms;
} ThroughputStats_t;

ThroughputStats_t stats;

/**
  * @brief  통계 초기화
  */
void Reset_Throughput_Stats(void)
{
    stats.messages_sent = 0;
    stats.messages_received = 0;
    stats.bytes_sent = 0;
    stats.bytes_received = 0;
    stats.start_time = HAL_GetTick();
}

/**
  * @brief  메시지 전송 통계 업데이트
  */
void Update_Send_Stats(Message_t *msg)
{
    stats.messages_sent++;
    stats.bytes_sent += msg->data_length;
}

/**
  * @brief  메시지 수신 통계 업데이트
  */
void Update_Receive_Stats(Message_t *msg)
{
    stats.messages_received++;
    stats.bytes_received += msg->data_length;
}

/**
  * @brief  처리량 계산 및 출력
  */
void Print_Throughput_Stats(void)
{
    stats.duration_ms = HAL_GetTick() - stats.start_time;

    if (stats.duration_ms > 0)
    {
        float msg_rate = (float)stats.messages_sent * 1000.0f / (float)stats.duration_ms;
        float data_rate = (float)stats.bytes_sent * 1000.0f / (float)stats.duration_ms;

        printf("\n=== Throughput Statistics ===\n");
        printf("Duration: %lu ms\n", stats.duration_ms);
        printf("Messages Sent: %lu (%.2f msg/s)\n", stats.messages_sent, msg_rate);
        printf("Messages Received: %lu\n", stats.messages_received);
        printf("Bytes Sent: %lu (%.2f bytes/s)\n", stats.bytes_sent, data_rate);
        printf("Bytes Received: %lu\n", stats.bytes_received);
        printf("============================\n");
    }
}
```

## 실전 응용 예제

### 1. 센서 데이터 스트리밍

```c
/* sensor_streaming.c - CM4에서 센서 읽고 CM7로 전송 */

#define SENSOR_SAMPLE_RATE_HZ  100  /* 100Hz */
#define SENSOR_DATA_SIZE       12   /* 3축 * 4바이트 */

typedef struct
{
    float accel_x;
    float accel_y;
    float accel_z;
} SensorData_t;

void CM4_Sensor_Task(void)
{
    SensorData_t sensor_data;
    uint32_t last_sample_time = 0;

    while (1)
    {
        uint32_t current_time = HAL_GetTick();

        if ((current_time - last_sample_time) >= (1000 / SENSOR_SAMPLE_RATE_HZ))
        {
            /* 센서 데이터 읽기 */
            Read_Accelerometer(&sensor_data);

            /* CM7로 전송 */
            Send_Message_To_CM7(MSG_TYPE_DATA,
                               (uint8_t*)&sensor_data,
                               sizeof(SensorData_t));

            last_sample_time = current_time;
        }
    }
}

/* CM7에서 데이터 수신 및 처리 */
void CM7_Process_Sensor_Data(Message_t *msg)
{
    SensorData_t *data = (SensorData_t*)msg->data;

    /* FFT 분석, 필터링 등 */
    Perform_Signal_Processing(data);

    /* 결과를 UART로 출력 */
    printf("Accel: X=%.2f, Y=%.2f, Z=%.2f\n",
           data->accel_x, data->accel_y, data->accel_z);
}
```

### 2. 명령/응답 프로토콜

```c
/* command_protocol.c */

typedef enum
{
    CMD_GET_STATUS = 0x01,
    CMD_SET_LED,
    CMD_READ_ADC,
    CMD_START_TIMER,
    CMD_STOP_TIMER
} CommandCode_t;

typedef struct
{
    CommandCode_t code;
    uint32_t param1;
    uint32_t param2;
} Command_t;

typedef struct
{
    uint32_t status_code;
    uint32_t result_length;
    uint8_t result_data[248];
} Response_t;

/**
  * @brief  명령 전송 (CM7 → CM4)
  */
HAL_StatusTypeDef Send_Command(CommandCode_t cmd, uint32_t param1, uint32_t param2)
{
    Command_t command;
    command.code = cmd;
    command.param1 = param1;
    command.param2 = param2;

    return Send_Message_To_CM4(MSG_TYPE_COMMAND,
                               (uint8_t*)&command,
                               sizeof(Command_t));
}

/**
  * @brief  명령 처리 (CM4)
  */
void Execute_Command_CM4(Message_t *msg)
{
    Command_t *cmd = (Command_t*)msg->data;
    Response_t response;

    response.status_code = 0;  /* Success */

    switch (cmd->code)
    {
        case CMD_GET_STATUS:
            response.result_length = 4;
            *(uint32_t*)response.result_data = Get_System_Status();
            break;

        case CMD_SET_LED:
            if (cmd->param1 < 4)
            {
                if (cmd->param2)
                    BSP_LED_On(cmd->param1);
                else
                    BSP_LED_Off(cmd->param1);
            }
            response.result_length = 0;
            break;

        case CMD_READ_ADC:
            response.result_length = 4;
            *(uint32_t*)response.result_data = Read_ADC_Channel(cmd->param1);
            break;

        default:
            response.status_code = 0xFF;  /* Unknown command */
            response.result_length = 0;
            break;
    }

    /* 응답 전송 */
    Send_Message_To_CM7(MSG_TYPE_DATA,
                       (uint8_t*)&response,
                       sizeof(Response_t));
}
```

## 문제 해결

### 1. 메시지 손실

**증상**: 일부 메시지가 수신되지 않음

**진단**:
```c
void Diagnose_Message_Loss(void)
{
    printf("CM7→CM4 Queue: %lu/%d (overflow: %lu)\n",
           cm7_to_cm4_queue.count, MSG_QUEUE_SIZE,
           cm7_to_cm4_queue.overflow_count);

    printf("CM4→CM7 Queue: %lu/%d (overflow: %lu)\n",
           cm4_to_cm7_queue.count, MSG_QUEUE_SIZE,
           cm4_to_cm7_queue.overflow_count);
}
```

**해결 방법**:
- 큐 크기 증가
- 메시지 처리 속도 향상
- 우선순위 기반 큐 사용

### 2. 인터럽트가 발생하지 않음

**원인**: HSEM 인터럽트 설정 오류

**해결**:
```c
void Verify_HSEM_Interrupt_Config(void)
{
    /* 인터럽트 마스크 확인 */
    printf("HSEM IER: 0x%08lX\n", HSEM->IER);
    printf("HSEM MISR: 0x%08lX\n", HSEM->MISR);

    /* NVIC 설정 확인 */
    printf("NVIC ISER[1]: 0x%08lX\n", NVIC->ISER[1]);

    /* 재설정 */
    HAL_NVIC_DisableIRQ(HSEM1_IRQn);
    HAL_NVIC_ClearPendingIRQ(HSEM1_IRQn);
    HAL_NVIC_SetPriority(HSEM1_IRQn, 5, 0);
    HAL_NVIC_EnableIRQ(HSEM1_IRQn);
}
```

### 3. 공유 메모리 데이터 불일치

**원인**: 캐시 일관성 문제

**해결**:
```c
/* 메시지 전송 전 캐시 플러시 */
void Send_Message_With_Cache_Coherency(Message_t *msg)
{
    /* 메시지를 큐에 추가 */
    MessageQueue_Enqueue(&cm7_to_cm4_queue, msg);

    /* D-Cache 클린 (메모리로 쓰기) */
    SCB_CleanDCache_by_Addr((uint32_t*)&cm7_to_cm4_queue, sizeof(MessageQueue_t));

    /* 알림 전송 */
    HAL_HSEM_FastTake(HSEM_ID_CM7_TO_CM4);
    HAL_HSEM_Release(HSEM_ID_CM7_TO_CM4, PROCESS_ID_CM7);
}

/* 메시지 수신 전 캐시 무효화 */
void Receive_Message_With_Cache_Coherency(Message_t *msg)
{
    /* D-Cache 무효화 (메모리에서 다시 읽기) */
    SCB_InvalidateDCache_by_Addr((uint32_t*)&cm4_to_cm7_queue, sizeof(MessageQueue_t));

    /* 큐에서 메시지 추출 */
    MessageQueue_Dequeue(&cm4_to_cm7_queue, msg);
}
```

## 빌드 및 테스트

### 빌드

```bash
# CM7 빌드
cd CM7
make clean && make -j8

# CM4 빌드
cd ../CM4
make clean && make -j8
```

### 플래싱

```bash
# 두 코어 모두 플래시
openocd -f board/stm32h745i-disco.cfg \
  -c "program CM7/build/CM7.elf verify" \
  -c "program CM4/build/CM4.elf verify reset exit"
```

### 테스트 시나리오

1. **기본 메시지 전송 테스트**
   - USER 버튼을 눌러 CM7에서 CM4로 메시지 전송
   - LED1 토글 확인 (CM7 전송)
   - LED2 토글 확인 (CM4 수신)

2. **양방향 통신 테스트**
   - CM4가 자동으로 응답 메시지 전송
   - 양쪽 LED가 교대로 토글

3. **고부하 테스트**
   - 빠른 속도로 연속 메시지 전송
   - 큐 오버플로우 확인
   - 처리량 측정

## 참고 자료

- STM32H745/755 Reference Manual (RM0399) - HSEM 챕터
- AN5361: STM32H7x5/x7 듀얼 코어 시작하기
- AN4838: Managing memory protection unit in STM32
- STM32H745I-DISCO User Manual (UM2488)

## 라이선스

이 예제는 STMicroelectronics의 BSD 3-Clause 라이선스를 따릅니다.
