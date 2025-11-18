# FDCAN_Classic_Frame_Networking 예제

## 개요

STM32H745I-DISCO의 FDCAN(CAN with Flexible Data-Rate)을 사용하여 두 보드 간 클래식 CAN 프레임으로 통신하는 예제입니다.

## 하드웨어

- **보드**: STM32H745I-DISCO × 2
- **트랜시버**: TJA1042 또는 TJA1043 (CAN FD 지원)
- **연결**:
  - CANH ↔ CANH
  - CANL ↔ CANL
  - GND ↔ GND
- **종단 저항**: 120Ω (양쪽 끝)

## 주요 코드

```c
/* FDCAN 초기화 */
FDCAN_HandleTypeDef hfdcan1;

void FDCAN_Init(void)
{
    __HAL_RCC_FDCAN_CLK_ENABLE();

    hfdcan1.Instance = FDCAN1;
    hfdcan1.Init.FrameFormat = FDCAN_FRAME_CLASSIC;
    hfdcan1.Init.Mode = FDCAN_MODE_NORMAL;
    hfdcan1.Init.AutoRetransmission = ENABLE;
    hfdcan1.Init.TransmitPause = DISABLE;
    hfdcan1.Init.ProtocolException = DISABLE;

    /* Nominal Bit Timing (500 kbps) */
    hfdcan1.Init.NominalPrescaler = 10;
    hfdcan1.Init.NominalSyncJumpWidth = 1;
    hfdcan1.Init.NominalTimeSeg1 = 13;
    hfdcan1.Init.NominalTimeSeg2 = 2;

    /* Message RAM 설정 */
    hfdcan1.Init.MessageRAMOffset = 0;
    hfdcan1.Init.StdFiltersNbr = 1;
    hfdcan1.Init.ExtFiltersNbr = 0;
    hfdcan1.Init.RxFifo0ElmtsNbr = 8;
    hfdcan1.Init.RxFifo0ElmtSize = FDCAN_DATA_BYTES_8;
    hfdcan1.Init.RxBuffersNbr = 0;
    hfdcan1.Init.TxEventsNbr = 0;
    hfdcan1.Init.TxBuffersNbr = 0;
    hfdcan1.Init.TxFifoQueueElmtsNbr = 8;
    hfdcan1.Init.TxFifoQueueMode = FDCAN_TX_FIFO_OPERATION;
    hfdcan1.Init.TxElmtSize = FDCAN_DATA_BYTES_8;

    if (HAL_FDCAN_Init(&hfdcan1) != HAL_OK)
    {
        Error_Handler();
    }

    /* RX 필터 설정 */
    FDCAN_FilterTypeDef sFilterConfig;
    sFilterConfig.IdType = FDCAN_STANDARD_ID;
    sFilterConfig.FilterIndex = 0;
    sFilterConfig.FilterType = FDCAN_FILTER_MASK;
    sFilterConfig.FilterConfig = FDCAN_FILTER_TO_RXFIFO0;
    sFilterConfig.FilterID1 = 0x123;
    sFilterConfig.FilterID2 = 0x7FF;  /* Accept all */

    if (HAL_FDCAN_ConfigFilter(&hfdcan1, &sFilterConfig) != HAL_OK)
    {
        Error_Handler();
    }

    /* RX FIFO 인터럽트 활성화 */
    if (HAL_FDCAN_ActivateNotification(&hfdcan1, FDCAN_IT_RX_FIFO0_NEW_MESSAGE, 0) != HAL_OK)
    {
        Error_Handler();
    }

    /* FDCAN 시작 */
    if (HAL_FDCAN_Start(&hfdcan1) != HAL_OK)
    {
        Error_Handler();
    }

    printf("FDCAN initialized: 500 kbps, Classic CAN\n");
}

/* CAN 메시지 전송 */
HAL_StatusTypeDef FDCAN_Transmit(uint32_t id, uint8_t *data, uint32_t length)
{
    FDCAN_TxHeaderTypeDef TxHeader;

    TxHeader.Identifier = id;
    TxHeader.IdType = FDCAN_STANDARD_ID;
    TxHeader.TxFrameType = FDCAN_DATA_FRAME;
    TxHeader.DataLength = (length << 16);  /* DLC */
    TxHeader.ErrorStateIndicator = FDCAN_ESI_ACTIVE;
    TxHeader.BitRateSwitch = FDCAN_BRS_OFF;
    TxHeader.FDFormat = FDCAN_CLASSIC_CAN;
    TxHeader.TxEventFifoControl = FDCAN_NO_TX_EVENTS;
    TxHeader.MessageMarker = 0;

    if (HAL_FDCAN_AddMessageToTxFifoQ(&hfdcan1, &TxHeader, data) != HAL_OK)
    {
        return HAL_ERROR;
    }

    printf("TX: ID=0x%03lX, Len=%lu, Data=", id, length);
    for (uint32_t i = 0; i < length; i++)
    {
        printf("%02X ", data[i]);
    }
    printf("\n");

    return HAL_OK;
}

/* RX 인터럽트 콜백 */
void HAL_FDCAN_RxFifo0Callback(FDCAN_HandleTypeDef *hfdcan, uint32_t RxFifo0ITs)
{
    if ((RxFifo0ITs & FDCAN_IT_RX_FIFO0_NEW_MESSAGE) != 0)
    {
        FDCAN_RxHeaderTypeDef RxHeader;
        uint8_t RxData[8];

        if (HAL_FDCAN_GetRxMessage(hfdcan, FDCAN_RX_FIFO0, &RxHeader, RxData) == HAL_OK)
        {
            uint32_t dlc = (RxHeader.DataLength >> 16);

            printf("RX: ID=0x%03lX, Len=%lu, Data=", RxHeader.Identifier, dlc);
            for (uint32_t i = 0; i < dlc; i++)
            {
                printf("%02X ", RxData[i]);
            }
            printf("\n");

            /* 수신 데이터 처리 */
            Process_CAN_Message(RxHeader.Identifier, RxData, dlc);

            BSP_LED_Toggle(LED1);
        }
    }
}

/* 메시지 처리 */
void Process_CAN_Message(uint32_t id, uint8_t *data, uint32_t length)
{
    switch (id)
    {
        case 0x123:  /* LED 제어 */
            if (data[0] == 1)
                BSP_LED_On(LED2);
            else
                BSP_LED_Off(LED2);
            break;

        case 0x124:  /* 센서 데이터 */
            uint16_t temperature = (data[0] << 8) | data[1];
            printf("Temperature: %.1f°C\n", temperature / 10.0f);
            break;

        default:
            break;
    }
}

/* 메인 (보드 1: 송신) */
int main(void)
{
    HAL_Init();
    SystemClock_Config();

    BSP_LED_Init(LED1);
    BSP_LED_Init(LED2);
    BSP_PB_Init(BUTTON_USER, BUTTON_MODE_EXTI);

    FDCAN_Init();

    printf("FDCAN Board 1 (Transmitter)\n");

    while (1)
    {
        /* 1초마다 메시지 전송 */
        uint8_t tx_data[8] = {0x11, 0x22, 0x33, 0x44, 0x55, 0x66, 0x77, 0x88};
        FDCAN_Transmit(0x123, tx_data, 8);

        BSP_LED_Toggle(LED2);
        HAL_Delay(1000);
    }
}

/* 버튼 인터럽트 */
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == BUTTON_USER_PIN)
    {
        /* LED 제어 명령 전송 */
        uint8_t led_cmd[1] = {1};  /* LED ON */
        FDCAN_Transmit(0x123, led_cmd, 1);
    }
}
```

## CAN FD 모드 (확장)

```c
/* FDCAN FD 모드 초기화 */
void FDCAN_FD_Init(void)
{
    /* ... (기본 설정 동일) ... */

    hfdcan1.Init.FrameFormat = FDCAN_FRAME_FD_BRS;

    /* Data Bit Timing (2 Mbps) */
    hfdcan1.Init.DataPrescaler = 2;
    hfdcan1.Init.DataSyncJumpWidth = 1;
    hfdcan1.Init.DataTimeSeg1 = 13;
    hfdcan1.Init.DataTimeSeg2 = 2;

    /* ... */
}

/* CAN FD 프레임 전송 (최대 64바이트) */
HAL_StatusTypeDef FDCAN_FD_Transmit(uint32_t id, uint8_t *data, uint32_t length)
{
    /* length: 0, 1, 2, ..., 64 */
    TxHeader.FDFormat = FDCAN_FD_CAN;
    TxHeader.BitRateSwitch = FDCAN_BRS_ON;
    TxHeader.DataLength = FDCAN_DLC_BYTES_64;  /* 64바이트 */

    /* ... */
}
```

## 오류 처리

```c
/* CAN 버스 오류 확인 */
void FDCAN_CheckErrors(void)
{
    uint32_t errors = HAL_FDCAN_GetError(&hfdcan1);

    if (errors & HAL_FDCAN_ERROR_BIT)
        printf("Bit error\n");
    if (errors & HAL_FDCAN_ERROR_STUFF)
        printf("Stuff error\n");
    if (errors & HAL_FDCAN_ERROR_CRC)
        printf("CRC error\n");
    if (errors & HAL_FDCAN_ERROR_FORM)
        printf("Form error\n");
    if (errors & HAL_FDCAN_ERROR_ACK)
        printf("ACK error\n");
}

/* 버스 오프 복구 */
void FDCAN_RecoverBusOff(void)
{
    if (HAL_FDCAN_Stop(&hfdcan1) == HAL_OK)
    {
        HAL_Delay(100);
        HAL_FDCAN_Start(&hfdcan1);
        printf("Bus-off recovery\n");
    }
}
```

## 통신 프로토콜 예제

```c
/* CAN 기반 프로토콜 */
typedef enum
{
    CMD_GET_STATUS = 0x01,
    CMD_SET_LED,
    CMD_READ_SENSOR,
    CMD_RESET
} CANCommand_t;

typedef struct
{
    uint8_t command;
    uint8_t data[7];
} CANProtocol_t;

void Send_Command(CANCommand_t cmd, uint8_t *params, uint8_t length)
{
    CANProtocol_t packet;
    packet.command = cmd;
    memcpy(packet.data, params, length);

    FDCAN_Transmit(0x100, (uint8_t*)&packet, 1 + length);
}
```

## 참고

- Reference Manual: FDCAN 챕터
- ISO 11898-1: CAN 프로토콜
- 120Ω 종단 저항 필수
