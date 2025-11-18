# FDCAN_Image_transmission 예제

## 개요

STM32H745I-DISCO의 FDCAN을 사용하여 이미지 데이터를 CAN FD 프로토콜로 전송하는 예제입니다. 대용량 데이터 전송, 분할/재조립, 오류 복구 기능을 구현합니다.

## 하드웨어

- **보드**: STM32H745I-DISCO × 2
- **트랜시버**: TJA1043 (CAN FD)
- **연결**: CANH, CANL, 120Ω 종단
- **LED**:
  - LED1: 전송 진행
  - LED2: 수신 진행
  - LED3: 오류
  - LED4: 완료

## 프로토콜 설계

### 패킷 구조

```c
/* 이미지 전송 프로토콜 */
typedef enum
{
    PKT_TYPE_START = 0x01,    /* 전송 시작 */
    PKT_TYPE_DATA,            /* 데이터 청크 */
    PKT_TYPE_END,             /* 전송 완료 */
    PKT_TYPE_ACK,             /* 응답 확인 */
    PKT_TYPE_NACK,            /* 재전송 요청 */
    PKT_TYPE_ABORT            /* 전송 중단 */
} PacketType_t;

/* START 패킷 (8바이트) */
typedef struct __attribute__((packed))
{
    uint8_t  type;            /* PKT_TYPE_START */
    uint8_t  image_id;
    uint16_t width;
    uint16_t height;
    uint16_t total_chunks;
} StartPacket_t;

/* DATA 패킷 (CAN FD: 64바이트) */
typedef struct __attribute__((packed))
{
    uint8_t  type;            /* PKT_TYPE_DATA */
    uint16_t chunk_id;
    uint8_t  data_length;     /* 1~60 */
    uint8_t  data[60];
} DataPacket_t;

/* ACK/NACK 패킷 (4바이트) */
typedef struct __attribute__((packed))
{
    uint8_t  type;
    uint16_t chunk_id;
    uint8_t  reserved;
} AckPacket_t;
```

## 송신 측 코드

```c
/* 전역 변수 */
#define MAX_IMAGE_SIZE  (320 * 240 * 2)  /* RGB565: 153,600 bytes */
#define CHUNK_SIZE      60
#define MAX_CHUNKS      ((MAX_IMAGE_SIZE + CHUNK_SIZE - 1) / CHUNK_SIZE)

uint8_t image_buffer[MAX_IMAGE_SIZE];
uint32_t image_size = 0;
uint16_t total_chunks = 0;

typedef struct
{
    uint16_t chunk_id;
    uint8_t  ack_received;
    uint8_t  retry_count;
} ChunkStatus_t;

ChunkStatus_t chunk_status[MAX_CHUNKS];

/**
  * @brief  이미지 전송 초기화
  */
HAL_StatusTypeDef Image_TransmitInit(uint8_t *image_data, uint32_t size,
                                      uint16_t width, uint16_t height)
{
    if (size > MAX_IMAGE_SIZE)
    {
        printf("Image too large: %lu bytes\n", size);
        return HAL_ERROR;
    }

    /* 이미지 복사 */
    memcpy(image_buffer, image_data, size);
    image_size = size;
    total_chunks = (size + CHUNK_SIZE - 1) / CHUNK_SIZE;

    /* 청크 상태 초기화 */
    for (uint16_t i = 0; i < total_chunks; i++)
    {
        chunk_status[i].chunk_id = i;
        chunk_status[i].ack_received = 0;
        chunk_status[i].retry_count = 0;
    }

    printf("Image prepared: %lu bytes, %u chunks\n", size, total_chunks);

    /* START 패킷 전송 */
    StartPacket_t start_pkt;
    start_pkt.type = PKT_TYPE_START;
    start_pkt.image_id = 1;
    start_pkt.width = width;
    start_pkt.height = height;
    start_pkt.total_chunks = total_chunks;

    FDCAN_Transmit(0x200, (uint8_t*)&start_pkt, sizeof(StartPacket_t));

    printf("START packet sent\n");

    return HAL_OK;
}

/**
  * @brief  데이터 청크 전송
  */
HAL_StatusTypeDef Image_TransmitChunk(uint16_t chunk_id)
{
    if (chunk_id >= total_chunks)
    {
        return HAL_ERROR;
    }

    DataPacket_t data_pkt;
    data_pkt.type = PKT_TYPE_DATA;
    data_pkt.chunk_id = chunk_id;

    /* 청크 데이터 복사 */
    uint32_t offset = chunk_id * CHUNK_SIZE;
    uint32_t remaining = image_size - offset;
    data_pkt.data_length = (remaining < CHUNK_SIZE) ? remaining : CHUNK_SIZE;

    memcpy(data_pkt.data, &image_buffer[offset], data_pkt.data_length);

    /* CAN FD 전송 (64바이트) */
    FDCAN_FD_Transmit(0x201, (uint8_t*)&data_pkt, 4 + data_pkt.data_length);

    printf("Chunk %u/%u sent (%u bytes)\n",
           chunk_id + 1, total_chunks, data_pkt.data_length);

    BSP_LED_Toggle(LED1);

    return HAL_OK;
}

/**
  * @brief  전체 이미지 전송 (순차)
  */
HAL_StatusTypeDef Image_TransmitAll(void)
{
    for (uint16_t i = 0; i < total_chunks; i++)
    {
        /* 청크 전송 */
        if (Image_TransmitChunk(i) != HAL_OK)
        {
            return HAL_ERROR;
        }

        /* ACK 대기 */
        uint32_t timeout = HAL_GetTick() + 100;
        while (!chunk_status[i].ack_received && HAL_GetTick() < timeout)
        {
            HAL_Delay(1);
        }

        /* 타임아웃 처리 */
        if (!chunk_status[i].ack_received)
        {
            printf("Chunk %u: ACK timeout, retrying...\n", i);

            if (chunk_status[i].retry_count++ < 3)
            {
                i--;  /* 재전송 */
            }
            else
            {
                printf("Chunk %u: Max retries exceeded\n", i);
                BSP_LED_On(LED3);
                return HAL_ERROR;
            }
        }
    }

    /* END 패킷 전송 */
    uint8_t end_pkt[1] = {PKT_TYPE_END};
    FDCAN_Transmit(0x202, end_pkt, 1);

    printf("Image transmission completed\n");
    BSP_LED_On(LED4);

    return HAL_OK;
}

/**
  * @brief  ACK 수신 처리
  */
void Handle_ACK_Packet(AckPacket_t *ack)
{
    if (ack->chunk_id < total_chunks)
    {
        chunk_status[ack->chunk_id].ack_received = 1;
    }
}
```

## 수신 측 코드

```c
/* 수신 버퍼 */
uint8_t rx_image_buffer[MAX_IMAGE_SIZE];
uint32_t rx_image_size = 0;
uint16_t rx_total_chunks = 0;
uint16_t rx_received_chunks = 0;

typedef struct
{
    uint16_t chunk_id;
    uint8_t  received;
} RxChunkStatus_t;

RxChunkStatus_t rx_chunk_status[MAX_CHUNKS];

/**
  * @brief  START 패킷 처리
  */
void Handle_START_Packet(StartPacket_t *start)
{
    printf("Receiving image: %ux%u, %u chunks\n",
           start->width, start->height, start->total_chunks);

    rx_total_chunks = start->total_chunks;
    rx_received_chunks = 0;
    rx_image_size = 0;

    /* 수신 상태 초기화 */
    for (uint16_t i = 0; i < rx_total_chunks; i++)
    {
        rx_chunk_status[i].chunk_id = i;
        rx_chunk_status[i].received = 0;
    }

    /* ACK 전송 */
    AckPacket_t ack;
    ack.type = PKT_TYPE_ACK;
    ack.chunk_id = 0xFFFF;  /* START ACK */
    ack.reserved = 0;

    FDCAN_Transmit(0x210, (uint8_t*)&ack, sizeof(AckPacket_t));
}

/**
  * @brief  DATA 패킷 처리
  */
void Handle_DATA_Packet(DataPacket_t *data)
{
    uint16_t chunk_id = data->chunk_id;

    if (chunk_id >= rx_total_chunks)
    {
        printf("Invalid chunk ID: %u\n", chunk_id);
        return;
    }

    /* 중복 체크 */
    if (rx_chunk_status[chunk_id].received)
    {
        printf("Duplicate chunk %u ignored\n", chunk_id);
        return;
    }

    /* 데이터 저장 */
    uint32_t offset = chunk_id * CHUNK_SIZE;
    memcpy(&rx_image_buffer[offset], data->data, data->data_length);

    rx_chunk_status[chunk_id].received = 1;
    rx_received_chunks++;
    rx_image_size += data->data_length;

    printf("Chunk %u/%u received\n", rx_received_chunks, rx_total_chunks);

    BSP_LED_Toggle(LED2);

    /* ACK 전송 */
    AckPacket_t ack;
    ack.type = PKT_TYPE_ACK;
    ack.chunk_id = chunk_id;
    ack.reserved = 0;

    FDCAN_Transmit(0x210, (uint8_t*)&ack, sizeof(AckPacket_t));

    /* 진행률 표시 */
    uint32_t progress = (rx_received_chunks * 100) / rx_total_chunks;
    printf("Progress: %lu%%\n", progress);
}

/**
  * @brief  END 패킷 처리
  */
void Handle_END_Packet(void)
{
    printf("Image reception completed: %lu bytes\n", rx_image_size);

    /* 모든 청크 수신 확인 */
    uint16_t missing_chunks = 0;

    for (uint16_t i = 0; i < rx_total_chunks; i++)
    {
        if (!rx_chunk_status[i].received)
        {
            printf("Missing chunk: %u\n", i);
            missing_chunks++;
        }
    }

    if (missing_chunks > 0)
    {
        printf("ERROR: %u chunks missing\n", missing_chunks);
        BSP_LED_On(LED3);

        /* NACK 전송 (재전송 요청) */
        for (uint16_t i = 0; i < rx_total_chunks; i++)
        {
            if (!rx_chunk_status[i].received)
            {
                AckPacket_t nack;
                nack.type = PKT_TYPE_NACK;
                nack.chunk_id = i;
                nack.reserved = 0;

                FDCAN_Transmit(0x210, (uint8_t*)&nack, sizeof(AckPacket_t));
            }
        }
    }
    else
    {
        printf("All chunks received successfully\n");
        BSP_LED_On(LED4);

        /* 이미지 저장 또는 표시 */
        Save_Image(rx_image_buffer, rx_image_size);
    }
}

/**
  * @brief  수신 메시지 처리
  */
void Process_RX_Message(uint32_t id, uint8_t *data, uint32_t length)
{
    switch (id)
    {
        case 0x200:  /* START */
            Handle_START_Packet((StartPacket_t*)data);
            break;

        case 0x201:  /* DATA */
            Handle_DATA_Packet((DataPacket_t*)data);
            break;

        case 0x202:  /* END */
            Handle_END_Packet();
            break;

        case 0x210:  /* ACK/NACK (송신 측) */
            if (data[0] == PKT_TYPE_ACK)
                Handle_ACK_Packet((AckPacket_t*)data);
            else if (data[0] == PKT_TYPE_NACK)
                Handle_NACK_Packet((AckPacket_t*)data);
            break;

        default:
            break;
    }
}
```

## 이미지 저장

```c
/* FatFS를 사용한 SD 카드 저장 */
void Save_Image(uint8_t *image_data, uint32_t size)
{
    FIL file;
    UINT bytes_written;

    /* 파일 생성 */
    if (f_open(&file, "image.raw", FA_CREATE_ALWAYS | FA_WRITE) == FR_OK)
    {
        /* 데이터 쓰기 */
        f_write(&file, image_data, size, &bytes_written);

        /* 파일 닫기 */
        f_close(&file);

        printf("Image saved: %lu bytes\n", bytes_written);
    }
    else
    {
        printf("Failed to save image\n");
    }
}
```

## 성능 최적화

```c
/* 멀티스레드 전송 (RTOS) */
void Image_TransmitTask(void *argument)
{
    osMessageQueueId_t tx_queue = osMessageQueueNew(16, sizeof(uint16_t), NULL);

    /* Producer: 청크 ID 생성 */
    for (uint16_t i = 0; i < total_chunks; i++)
    {
        osMessageQueuePut(tx_queue, &i, 0, osWaitForever);
    }

    /* Consumer: 청크 전송 */
    while (1)
    {
        uint16_t chunk_id;
        if (osMessageQueueGet(tx_queue, &chunk_id, NULL, 1000) == osOK)
        {
            Image_TransmitChunk(chunk_id);
        }
    }
}

/* 압축 전송 (간단한 RLE) */
uint32_t Compress_Image(uint8_t *src, uint8_t *dst, uint32_t size)
{
    uint32_t dst_idx = 0;

    for (uint32_t i = 0; i < size; )
    {
        uint8_t value = src[i];
        uint8_t count = 1;

        /* 연속된 동일 값 카운트 */
        while (i + count < size && src[i + count] == value && count < 255)
        {
            count++;
        }

        /* RLE 인코딩 */
        if (count > 3)
        {
            dst[dst_idx++] = 0xFF;  /* RLE 마커 */
            dst[dst_idx++] = count;
            dst[dst_idx++] = value;
            i += count;
        }
        else
        {
            dst[dst_idx++] = value;
            i++;
        }
    }

    return dst_idx;
}
```

## 메인 코드

```c
/* 송신 측 메인 */
int main(void)
{
    HAL_Init();
    SystemClock_Config();

    BSP_LED_Init(LED1);
    BSP_LED_Init(LED2);
    BSP_LED_Init(LED3);
    BSP_LED_Init(LED4);

    FDCAN_Init();

    printf("Image Transmitter Ready\n");

    /* 테스트 이미지 (320x240 RGB565) */
    uint8_t test_image[320 * 240 * 2];
    Generate_Test_Image(test_image, 320, 240);

    /* 이미지 전송 초기화 */
    Image_TransmitInit(test_image, sizeof(test_image), 320, 240);

    /* 전송 시작 */
    Image_TransmitAll();

    while (1)
    {
        HAL_Delay(1000);
    }
}

/* 수신 측 메인 */
int main(void)
{
    HAL_Init();
    SystemClock_Config();

    BSP_LED_Init(LED1);
    BSP_LED_Init(LED2);
    BSP_LED_Init(LED3);
    BSP_LED_Init(LED4);

    FDCAN_Init();

    /* SD 카드 초기화 */
    SD_Init();
    f_mount(&SDFatFS, "", 0);

    printf("Image Receiver Ready\n");

    while (1)
    {
        /* RX 인터럽트에서 처리 */
        HAL_Delay(100);
    }
}
```

## 성능 분석

```
이미지 크기: 320×240 RGB565 = 153,600 bytes
청크 크기: 60 bytes
총 청크: 2,560 chunks

CAN FD (2 Mbps 데이터):
- 청크당 전송 시간: ~0.3ms
- 총 전송 시간: ~0.8초 (이상적)
- 실제 (ACK 포함): ~1.5초

클래식 CAN (500 kbps):
- 청크당 전송 시간: ~1.2ms
- 총 전송 시간: ~3초
- 실제: ~5초
```

## 참고

- Reference Manual: FDCAN 챕터
- CAN FD 최대 64바이트
- 오류 검출 및 재전송 필수
