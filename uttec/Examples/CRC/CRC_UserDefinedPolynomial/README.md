# CRC_UserDefinedPolynomial 예제

## 개요

이 예제는 STM32H745I-DISCO의 CRC 계산 유닛을 사용하여 사용자 정의 다항식(8-bit, polynomial 0x9B)으로 데이터 무결성을 검사하는 방법을 보여줍니다. 통신 프로토콜, 파일 시스템, 메모리 검증 등에 필수적인 CRC 계산 기법을 학습할 수 있습니다.

## 하드웨어 요구사항

- **보드**: STM32H745I-DISCO
- **LED**:
  - LED1 (녹색): CRC 계산 성공
  - LED2 (주황색): CRC 검증 진행 중
  - LED3 (빨간색): CRC 불일치 (데이터 손상)
  - LED4 (파란색): 연속 테스트 중
- **버튼**: USER 버튼 (CRC 테스트 트리거)

## CRC(Cyclic Redundancy Check) 개요

### CRC의 원리

CRC는 다항식 나눗셈을 사용하여 데이터의 체크섬을 계산하는 오류 검출 코드입니다.

```
데이터: 1101 0110
다항식: 1001 (x^3 + 1)

나눗셈 수행 → 나머지가 CRC 값
```

### STM32H7 CRC 유닛 특징

- **하드웨어 가속**: 소프트웨어보다 수십 배 빠른 계산
- **다양한 다항식**: 7/8/16/32-bit 다항식 지원
- **사용자 정의 가능**: 초기값, 다항식, 입력/출력 반전 설정
- **DMA 지원**: 대용량 데이터 자동 처리

## 주요 기능

### 1. 8-bit CRC (Polynomial 0x9B)
- **다항식**: x^8 + x^7 + x^4 + x^3 + x + 1 (0x9B)
- **초기값**: 0xFF
- **용도**: SMBus, I2C 센서 통신

### 2. 표준 CRC 알고리즘
- **CRC-8**: 0x07, 0x9B, 0x31 등
- **CRC-16**: CCITT, Modbus, USB
- **CRC-32**: Ethernet, ZIP, PNG

## 소프트웨어 설명

### CRC 구성 구조체

```c
/* crc_calculator.h */

/* CRC 다항식 정의 */
#define CRC_POLYNOMIAL_8BIT_0x9B    0x9B    /* x^8+x^7+x^4+x^3+x+1 */
#define CRC_POLYNOMIAL_8BIT_0x07    0x07    /* CRC-8 */
#define CRC_POLYNOMIAL_16BIT_CCITT  0x1021  /* CRC-16 CCITT */
#define CRC_POLYNOMIAL_32BIT_ETH    0x04C11DB7  /* CRC-32 Ethernet */

/* 초기값 */
#define CRC_INIT_VALUE_0xFF         0xFF
#define CRC_INIT_VALUE_0x00         0x00
#define CRC_INIT_VALUE_0xFFFF       0xFFFF
#define CRC_INIT_VALUE_0xFFFFFFFF   0xFFFFFFFF

/* CRC 설정 구조체 */
typedef struct
{
    uint32_t polynomial;
    uint32_t init_value;
    uint8_t  polynomial_size;  /* 7, 8, 16, 32 */
    uint8_t  input_reverse;    /* 0: No reverse, 1: Byte reverse, 2: Half-word, 3: Word */
    uint8_t  output_reverse;   /* 0: No reverse, 1: Reverse */
} CRC_Config_t;

/* CRC 테스트 결과 */
typedef struct
{
    uint32_t calculated_crc;
    uint32_t expected_crc;
    uint32_t computation_time_us;
    uint8_t  match;
} CRC_TestResult_t;
```

### CRC 초기화 및 계산

```c
/* crc_calculator.c */

CRC_HandleTypeDef hcrc;

/**
  * @brief  CRC 유닛 초기화 (8-bit, 0x9B)
  */
HAL_StatusTypeDef CRC_Init_8bit_0x9B(void)
{
    /* CRC 클록 활성화 */
    __HAL_RCC_CRC_CLK_ENABLE();

    /* CRC 핸들 설정 */
    hcrc.Instance = CRC;

    /* 사용자 정의 다항식 사용 */
    hcrc.Init.DefaultPolynomialUse = DEFAULT_POLYNOMIAL_DISABLE;
    hcrc.Init.GeneratingPolynomial = CRC_POLYNOMIAL_8BIT_0x9B;
    hcrc.Init.CRCLength = CRC_POLYLENGTH_8B;

    /* 사용자 정의 초기값 사용 */
    hcrc.Init.DefaultInitValueUse = DEFAULT_INIT_VALUE_DISABLE;
    hcrc.Init.InitValue = CRC_INIT_VALUE_0xFF;

    /* 입력 데이터 반전 없음 */
    hcrc.Init.InputDataInversionMode = CRC_INPUTDATA_INVERSION_NONE;

    /* 출력 데이터 반전 없음 */
    hcrc.Init.OutputDataInversionMode = CRC_OUTPUTDATA_INVERSION_DISABLE;

    /* 입력 데이터 형식: 바이트 단위 */
    hcrc.InputDataFormat = CRC_INPUTDATA_FORMAT_BYTES;

    if (HAL_CRC_Init(&hcrc) != HAL_OK)
    {
        return HAL_ERROR;
    }

    printf("CRC initialized: 8-bit, Polynomial=0x%02X, Init=0x%02X\n",
           CRC_POLYNOMIAL_8BIT_0x9B, CRC_INIT_VALUE_0xFF);

    return HAL_OK;
}

/**
  * @brief  CRC 계산 (바이트 배열)
  * @param  data: 입력 데이터 버퍼
  * @param  length: 데이터 길이 (바이트)
  * @retval CRC 값
  */
uint32_t CRC_Calculate(uint8_t *data, uint32_t length)
{
    uint32_t crc;

    /* 하드웨어 CRC 계산 */
    crc = HAL_CRC_Calculate(&hcrc, (uint32_t*)data, length);

    return crc;
}

/**
  * @brief  CRC 누적 계산 (스트리밍 데이터용)
  * @param  data: 입력 데이터 버퍼
  * @param  length: 데이터 길이
  * @retval CRC 값
  */
uint32_t CRC_Accumulate(uint8_t *data, uint32_t length)
{
    uint32_t crc;

    /* 이전 CRC 값에 누적 */
    crc = HAL_CRC_Accumulate(&hcrc, (uint32_t*)data, length);

    return crc;
}

/**
  * @brief  CRC 계산 시작 (수동 제어)
  */
void CRC_Reset(void)
{
    /* CRC 레지스터 초기화 */
    __HAL_CRC_DR_RESET(&hcrc);
}

/**
  * @brief  현재 CRC 값 읽기
  */
uint32_t CRC_GetValue(void)
{
    return hcrc.Instance->DR;
}

/**
  * @brief  소프트웨어 CRC-8 계산 (비교용)
  * @param  data: 입력 데이터
  * @param  length: 데이터 길이
  * @param  polynomial: CRC 다항식
  * @param  init_value: 초기값
  * @retval CRC-8 값
  */
uint8_t CRC8_Software(uint8_t *data, uint32_t length,
                       uint8_t polynomial, uint8_t init_value)
{
    uint8_t crc = init_value;

    for (uint32_t i = 0; i < length; i++)
    {
        crc ^= data[i];

        for (uint8_t bit = 0; bit < 8; bit++)
        {
            if (crc & 0x80)
            {
                crc = (crc << 1) ^ polynomial;
            }
            else
            {
                crc = crc << 1;
            }
        }
    }

    return crc;
}

/**
  * @brief  하드웨어 vs 소프트웨어 CRC 비교
  */
void CRC_CompareHardwareSoftware(uint8_t *data, uint32_t length)
{
    uint32_t start_tick, end_tick;
    uint8_t hw_crc, sw_crc;

    /* 하드웨어 CRC 계산 */
    start_tick = DWT->CYCCNT;
    hw_crc = (uint8_t)CRC_Calculate(data, length);
    end_tick = DWT->CYCCNT;
    uint32_t hw_cycles = end_tick - start_tick;

    /* 소프트웨어 CRC 계산 */
    start_tick = DWT->CYCCNT;
    sw_crc = CRC8_Software(data, length, CRC_POLYNOMIAL_8BIT_0x9B,
                            CRC_INIT_VALUE_0xFF);
    end_tick = DWT->CYCCNT;
    uint32_t sw_cycles = end_tick - start_tick;

    /* 결과 출력 */
    printf("\n=== CRC Calculation Comparison ===\n");
    printf("Data Length: %lu bytes\n", length);
    printf("Hardware CRC: 0x%02X (%lu cycles)\n", hw_crc, hw_cycles);
    printf("Software CRC: 0x%02X (%lu cycles)\n", sw_crc, sw_cycles);
    printf("Speedup: %.2fx\n", (float)sw_cycles / (float)hw_cycles);
    printf("Match: %s\n", (hw_crc == sw_crc) ? "YES" : "NO");
    printf("==================================\n\n");
}
```

### 다양한 CRC 알고리즘 구현

```c
/* crc_algorithms.c */

/**
  * @brief  CRC-8 (0x07) 초기화
  */
HAL_StatusTypeDef CRC_Init_8bit_0x07(void)
{
    hcrc.Init.DefaultPolynomialUse = DEFAULT_POLYNOMIAL_DISABLE;
    hcrc.Init.GeneratingPolynomial = 0x07;
    hcrc.Init.CRCLength = CRC_POLYLENGTH_8B;
    hcrc.Init.DefaultInitValueUse = DEFAULT_INIT_VALUE_DISABLE;
    hcrc.Init.InitValue = 0x00;
    hcrc.Init.InputDataInversionMode = CRC_INPUTDATA_INVERSION_NONE;
    hcrc.Init.OutputDataInversionMode = CRC_OUTPUTDATA_INVERSION_DISABLE;
    hcrc.InputDataFormat = CRC_INPUTDATA_FORMAT_BYTES;

    return HAL_CRC_Init(&hcrc);
}

/**
  * @brief  CRC-16 CCITT 초기화
  */
HAL_StatusTypeDef CRC_Init_16bit_CCITT(void)
{
    hcrc.Init.DefaultPolynomialUse = DEFAULT_POLYNOMIAL_DISABLE;
    hcrc.Init.GeneratingPolynomial = 0x1021;  /* x^16+x^12+x^5+1 */
    hcrc.Init.CRCLength = CRC_POLYLENGTH_16B;
    hcrc.Init.DefaultInitValueUse = DEFAULT_INIT_VALUE_DISABLE;
    hcrc.Init.InitValue = 0xFFFF;
    hcrc.Init.InputDataInversionMode = CRC_INPUTDATA_INVERSION_NONE;
    hcrc.Init.OutputDataInversionMode = CRC_OUTPUTDATA_INVERSION_DISABLE;
    hcrc.InputDataFormat = CRC_INPUTDATA_FORMAT_BYTES;

    return HAL_CRC_Init(&hcrc);
}

/**
  * @brief  CRC-16 Modbus 초기화
  */
HAL_StatusTypeDef CRC_Init_16bit_Modbus(void)
{
    hcrc.Init.DefaultPolynomialUse = DEFAULT_POLYNOMIAL_DISABLE;
    hcrc.Init.GeneratingPolynomial = 0x8005;  /* x^16+x^15+x^2+1 */
    hcrc.Init.CRCLength = CRC_POLYLENGTH_16B;
    hcrc.Init.DefaultInitValueUse = DEFAULT_INIT_VALUE_DISABLE;
    hcrc.Init.InitValue = 0xFFFF;
    hcrc.Init.InputDataInversionMode = CRC_INPUTDATA_INVERSION_BYTE;
    hcrc.Init.OutputDataInversionMode = CRC_OUTPUTDATA_INVERSION_ENABLE;
    hcrc.InputDataFormat = CRC_INPUTDATA_FORMAT_BYTES;

    return HAL_CRC_Init(&hcrc);
}

/**
  * @brief  CRC-32 Ethernet 초기화
  */
HAL_StatusTypeDef CRC_Init_32bit_Ethernet(void)
{
    hcrc.Init.DefaultPolynomialUse = DEFAULT_POLYNOMIAL_ENABLE;  /* 0x04C11DB7 */
    hcrc.Init.CRCLength = CRC_POLYLENGTH_32B;
    hcrc.Init.DefaultInitValueUse = DEFAULT_INIT_VALUE_ENABLE;   /* 0xFFFFFFFF */
    hcrc.Init.InputDataInversionMode = CRC_INPUTDATA_INVERSION_BYTE;
    hcrc.Init.OutputDataInversionMode = CRC_OUTPUTDATA_INVERSION_ENABLE;
    hcrc.InputDataFormat = CRC_INPUTDATA_FORMAT_BYTES;

    return HAL_CRC_Init(&hcrc);
}
```

### 실전 응용: 프로토콜 구현

```c
/* protocol.c */

/* 패킷 구조체 */
typedef struct __attribute__((packed))
{
    uint8_t  header;        /* 0x55 */
    uint8_t  command;
    uint16_t length;
    uint8_t  data[256];
    uint8_t  crc8;          /* CRC-8 (0x9B) */
} Packet_t;

/**
  * @brief  패킷 생성 (CRC 자동 계산)
  */
HAL_StatusTypeDef Packet_Create(Packet_t *packet, uint8_t cmd,
                                 uint8_t *data, uint16_t length)
{
    if (length > 256)
    {
        return HAL_ERROR;
    }

    /* 헤더 설정 */
    packet->header = 0x55;
    packet->command = cmd;
    packet->length = length;

    /* 데이터 복사 */
    memcpy(packet->data, data, length);

    /* CRC 계산 (헤더부터 데이터까지) */
    uint32_t crc_length = 4 + length;  /* header(1) + cmd(1) + length(2) + data */
    packet->crc8 = (uint8_t)CRC_Calculate((uint8_t*)packet, crc_length);

    printf("Packet created: cmd=0x%02X, len=%u, CRC=0x%02X\n",
           cmd, length, packet->crc8);

    return HAL_OK;
}

/**
  * @brief  패킷 검증
  */
HAL_StatusTypeDef Packet_Verify(Packet_t *packet)
{
    /* 헤더 확인 */
    if (packet->header != 0x55)
    {
        printf("Invalid header: 0x%02X\n", packet->header);
        return HAL_ERROR;
    }

    /* CRC 계산 */
    uint32_t crc_length = 4 + packet->length;
    uint8_t calculated_crc = (uint8_t)CRC_Calculate((uint8_t*)packet, crc_length);

    /* CRC 비교 */
    if (calculated_crc != packet->crc8)
    {
        printf("CRC mismatch! Expected: 0x%02X, Got: 0x%02X\n",
               packet->crc8, calculated_crc);
        BSP_LED_On(LED3);
        return HAL_ERROR;
    }

    printf("Packet verified: CRC OK (0x%02X)\n", calculated_crc);
    BSP_LED_On(LED1);

    return HAL_OK;
}

/**
  * @brief  UART로 패킷 전송
  */
HAL_StatusTypeDef Packet_Transmit(UART_HandleTypeDef *huart, Packet_t *packet)
{
    uint32_t total_length = 4 + packet->length + 1;  /* header+cmd+len+data+crc */

    return HAL_UART_Transmit(huart, (uint8_t*)packet, total_length, 1000);
}

/**
  * @brief  UART에서 패킷 수신 및 검증
  */
HAL_StatusTypeDef Packet_Receive(UART_HandleTypeDef *huart, Packet_t *packet,
                                  uint32_t timeout)
{
    /* 헤더 + 명령 + 길이 수신 */
    if (HAL_UART_Receive(huart, (uint8_t*)packet, 4, timeout) != HAL_OK)
    {
        return HAL_TIMEOUT;
    }

    /* 데이터 + CRC 수신 */
    uint32_t remaining = packet->length + 1;
    if (HAL_UART_Receive(huart, packet->data, remaining, timeout) != HAL_OK)
    {
        return HAL_TIMEOUT;
    }

    /* 패킷 검증 */
    return Packet_Verify(packet);
}
```

### 메모리 무결성 검사

```c
/* memory_check.c */

/* 메모리 영역 CRC 테이블 */
typedef struct
{
    uint32_t start_address;
    uint32_t length;
    uint8_t  expected_crc;
} MemoryRegion_t;

#define NUM_MEMORY_REGIONS  4

MemoryRegion_t memory_regions[NUM_MEMORY_REGIONS] = {
    {0x08000000, 0x10000, 0x00},  /* Flash Bank 1, 64KB */
    {0x08100000, 0x10000, 0x00},  /* Flash Bank 2, 64KB */
    {0x24000000, 0x1000,  0x00},  /* SRAM1, 4KB */
    {0x30000000, 0x1000,  0x00},  /* SRAM2, 4KB */
};

/**
  * @brief  메모리 영역 CRC 초기화 (최초 한 번)
  */
void Memory_CRC_Initialize(void)
{
    printf("Initializing memory region CRCs...\n");

    for (uint32_t i = 0; i < NUM_MEMORY_REGIONS; i++)
    {
        uint8_t *addr = (uint8_t*)memory_regions[i].start_address;
        uint32_t len = memory_regions[i].length;

        uint8_t crc = (uint8_t)CRC_Calculate(addr, len);
        memory_regions[i].expected_crc = crc;

        printf("  Region %lu (0x%08lX, %lu bytes): CRC=0x%02X\n",
               i, memory_regions[i].start_address,
               memory_regions[i].length, crc);
    }

    printf("Initialization complete\n");
}

/**
  * @brief  메모리 무결성 검사
  * @retval HAL_OK: 모든 영역 정상, HAL_ERROR: 손상 감지
  */
HAL_StatusTypeDef Memory_IntegrityCheck(void)
{
    HAL_StatusTypeDef result = HAL_OK;

    printf("\nPerforming memory integrity check...\n");

    for (uint32_t i = 0; i < NUM_MEMORY_REGIONS; i++)
    {
        uint8_t *addr = (uint8_t*)memory_regions[i].start_address;
        uint32_t len = memory_regions[i].length;

        uint8_t current_crc = (uint8_t)CRC_Calculate(addr, len);

        if (current_crc != memory_regions[i].expected_crc)
        {
            printf("  [FAIL] Region %lu: Expected 0x%02X, Got 0x%02X\n",
                   i, memory_regions[i].expected_crc, current_crc);
            result = HAL_ERROR;
            BSP_LED_On(LED3);
        }
        else
        {
            printf("  [OK] Region %lu: CRC=0x%02X\n", i, current_crc);
        }
    }

    if (result == HAL_OK)
    {
        printf("Memory integrity check PASSED\n");
        BSP_LED_On(LED1);
    }
    else
    {
        printf("Memory integrity check FAILED\n");
    }

    return result;
}
```

### DMA를 이용한 고속 CRC 계산

```c
/* crc_dma.c */

DMA_HandleTypeDef hdma_crc;
volatile uint8_t crc_dma_complete = 0;

/**
  * @brief  DMA 초기화 (CRC용)
  */
void CRC_DMA_Init(void)
{
    /* DMA 클록 활성화 */
    __HAL_RCC_DMA1_CLK_ENABLE();

    /* DMA 스트림 설정 */
    hdma_crc.Instance = DMA1_Stream0;
    hdma_crc.Init.Request = DMA_REQUEST_MEM2MEM;
    hdma_crc.Init.Direction = DMA_MEMORY_TO_MEMORY;
    hdma_crc.Init.PeriphInc = DMA_PINC_ENABLE;
    hdma_crc.Init.MemInc = DMA_MINC_ENABLE;
    hdma_crc.Init.PeriphDataAlignment = DMA_PDATAALIGN_WORD;
    hdma_crc.Init.MemDataAlignment = DMA_MDATAALIGN_WORD;
    hdma_crc.Init.Mode = DMA_NORMAL;
    hdma_crc.Init.Priority = DMA_PRIORITY_HIGH;
    hdma_crc.Init.FIFOMode = DMA_FIFOMODE_ENABLE;
    hdma_crc.Init.FIFOThreshold = DMA_FIFO_THRESHOLD_FULL;

    if (HAL_DMA_Init(&hdma_crc) != HAL_OK)
    {
        Error_Handler();
    }

    /* DMA 인터럽트 설정 */
    HAL_NVIC_SetPriority(DMA1_Stream0_IRQn, 5, 0);
    HAL_NVIC_EnableIRQ(DMA1_Stream0_IRQn);
}

/**
  * @brief  DMA를 이용한 CRC 계산
  */
HAL_StatusTypeDef CRC_Calculate_DMA(uint8_t *data, uint32_t length)
{
    crc_dma_complete = 0;

    /* CRC 리셋 */
    __HAL_CRC_DR_RESET(&hcrc);

    /* DMA 전송 시작 */
    if (HAL_DMA_Start_IT(&hdma_crc, (uint32_t)data,
                         (uint32_t)&(hcrc.Instance->DR),
                         length / 4) != HAL_OK)
    {
        return HAL_ERROR;
    }

    /* DMA 완료 대기 */
    while (crc_dma_complete == 0)
    {
        __WFI();
    }

    return HAL_OK;
}

/**
  * @brief  DMA 전송 완료 콜백
  */
void HAL_DMA_XferCpltCallback(DMA_HandleTypeDef *hdma)
{
    if (hdma->Instance == DMA1_Stream0)
    {
        crc_dma_complete = 1;
    }
}

/**
  * @brief  DMA 인터럽트 핸들러
  */
void DMA1_Stream0_IRQHandler(void)
{
    HAL_DMA_IRQHandler(&hdma_crc);
}
```

## 메인 코드

```c
/* main.c */

int main(void)
{
    /* HAL 초기화 */
    HAL_Init();

    /* 시스템 클록 설정 */
    SystemClock_Config();

    /* DWT 사이클 카운터 활성화 (성능 측정용) */
    DWT_Init();

    /* LED 초기화 */
    BSP_LED_Init(LED1);
    BSP_LED_Init(LED2);
    BSP_LED_Init(LED3);
    BSP_LED_Init(LED4);

    /* 버튼 초기화 */
    BSP_PB_Init(BUTTON_USER, BUTTON_MODE_EXTI);

    /* UART 초기화 */
    UART_Init();

    printf("\n\n");
    printf("=======================================\n");
    printf("  STM32H7 CRC Calculation Example\n");
    printf("  8-bit CRC, Polynomial 0x9B\n");
    printf("=======================================\n\n");

    /* CRC 초기화 (8-bit, 0x9B) */
    if (CRC_Init_8bit_0x9B() != HAL_OK)
    {
        Error_Handler();
    }

    /* 테스트 데이터 */
    uint8_t test_data[] = "Hello STM32H745I-DISCO Board!";
    uint32_t data_length = strlen((char*)test_data);

    /* CRC 계산 */
    uint8_t crc_value = (uint8_t)CRC_Calculate(test_data, data_length);

    printf("Test Data: \"%s\"\n", test_data);
    printf("Data Length: %lu bytes\n", data_length);
    printf("CRC-8 (0x9B): 0x%02X\n\n", crc_value);

    /* 하드웨어 vs 소프트웨어 성능 비교 */
    CRC_CompareHardwareSoftware(test_data, data_length);

    /* 프로토콜 테스트 */
    Protocol_Test();

    /* 메모리 무결성 검사 초기화 */
    Memory_CRC_Initialize();

    /* 메인 루프 */
    while (1)
    {
        /* 주기적인 메모리 무결성 검사 (10초마다) */
        static uint32_t last_check_time = 0;
        if ((HAL_GetTick() - last_check_time) >= 10000)
        {
            Memory_IntegrityCheck();
            last_check_time = HAL_GetTick();
        }

        /* LED 토글 */
        BSP_LED_Toggle(LED4);
        HAL_Delay(1000);
    }
}

/**
  * @brief  프로토콜 테스트
  */
void Protocol_Test(void)
{
    Packet_t tx_packet, rx_packet;
    uint8_t test_payload[] = {0x11, 0x22, 0x33, 0x44, 0x55};

    printf("\n=== Protocol Test ===\n");

    /* 패킷 생성 */
    Packet_Create(&tx_packet, 0xA5, test_payload, sizeof(test_payload));

    /* 패킷을 수신 버퍼로 복사 (실제로는 UART 통신) */
    memcpy(&rx_packet, &tx_packet, sizeof(Packet_t));

    /* 패킷 검증 */
    if (Packet_Verify(&rx_packet) == HAL_OK)
    {
        printf("Packet verification SUCCESS\n");

        /* 데이터 처리 */
        printf("Command: 0x%02X\n", rx_packet.command);
        printf("Payload: ");
        for (uint16_t i = 0; i < rx_packet.length; i++)
        {
            printf("%02X ", rx_packet.data[i]);
        }
        printf("\n");
    }

    /* 데이터 손상 시뮬레이션 */
    printf("\n--- Simulating data corruption ---\n");
    rx_packet.data[0] ^= 0xFF;  /* 첫 바이트 반전 */

    if (Packet_Verify(&rx_packet) != HAL_OK)
    {
        printf("Corruption detected successfully!\n");
    }

    printf("======================\n\n");
}

/**
  * @brief  버튼 인터럽트 콜백
  */
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == BUTTON_USER_PIN)
    {
        printf("\n[Button Pressed] Running CRC test...\n");

        /* 대용량 데이터 CRC 테스트 */
        uint8_t large_data[4096];

        /* 랜덤 데이터 생성 */
        for (uint32_t i = 0; i < sizeof(large_data); i++)
        {
            large_data[i] = (uint8_t)(HAL_GetTick() + i);
        }

        /* 성능 측정 */
        uint32_t start_time = HAL_GetTick();
        uint8_t crc = (uint8_t)CRC_Calculate(large_data, sizeof(large_data));
        uint32_t duration = HAL_GetTick() - start_time;

        printf("CRC of %lu bytes: 0x%02X (took %lu ms)\n",
               sizeof(large_data), crc, duration);

        BSP_LED_Toggle(LED2);
    }
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

## 실전 응용 예제

### 1. 파일 시스템 CRC

```c
/* filesystem_crc.c */

typedef struct
{
    char     filename[32];
    uint32_t file_size;
    uint8_t  crc8;
} FileEntry_t;

/**
  * @brief  파일 CRC 계산 및 저장
  */
HAL_StatusTypeDef File_CalculateCRC(const char *filename, uint8_t *file_data,
                                      uint32_t file_size)
{
    FileEntry_t entry;

    /* 파일 정보 설정 */
    strncpy(entry.filename, filename, sizeof(entry.filename) - 1);
    entry.file_size = file_size;

    /* CRC 계산 */
    entry.crc8 = (uint8_t)CRC_Calculate(file_data, file_size);

    printf("File: %s, Size: %lu bytes, CRC: 0x%02X\n",
           entry.filename, entry.file_size, entry.crc8);

    /* 파일 엔트리를 비휘발성 메모리에 저장 */
    // Save_FileEntry(&entry);

    return HAL_OK;
}
```

### 2. 센서 데이터 검증

```c
/* sensor_crc.c */

typedef struct __attribute__((packed))
{
    uint16_t temperature;  /* 0.1°C 단위 */
    uint16_t humidity;     /* 0.1% 단위 */
    uint16_t pressure;     /* Pa */
    uint32_t timestamp;
    uint8_t  crc8;
} SensorData_t;

/**
  * @brief  센서 데이터 읽기 (CRC 검증)
  */
HAL_StatusTypeDef Sensor_ReadData(SensorData_t *data)
{
    /* I2C 또는 SPI로 센서 데이터 읽기 */
    // I2C_Read(...);

    /* CRC 검증 */
    uint8_t calculated_crc = (uint8_t)CRC_Calculate((uint8_t*)data,
                                                     sizeof(SensorData_t) - 1);

    if (calculated_crc != data->crc8)
    {
        printf("Sensor data CRC error!\n");
        return HAL_ERROR;
    }

    printf("Temp: %.1f°C, Humidity: %.1f%%, Pressure: %lu Pa\n",
           data->temperature / 10.0f,
           data->humidity / 10.0f,
           data->pressure);

    return HAL_OK;
}
```

## 문제 해결

### 1. CRC 값 불일치

**증상**: 계산된 CRC가 예상과 다름

**해결**:
```c
void Debug_CRC_Mismatch(void)
{
    /* CRC 설정 확인 */
    printf("CRC Config:\n");
    printf("  Polynomial: 0x%08lX\n", hcrc.Init.GeneratingPolynomial);
    printf("  Init Value: 0x%08lX\n", hcrc.Init.InitValue);
    printf("  Input Reverse: %u\n", hcrc.Init.InputDataInversionMode);
    printf("  Output Reverse: %u\n", hcrc.Init.OutputDataInversionMode);

    /* 소프트웨어 CRC와 비교 */
    uint8_t test_data[] = {0x01, 0x02, 0x03};
    uint8_t hw_crc = CRC_Calculate(test_data, 3);
    uint8_t sw_crc = CRC8_Software(test_data, 3, 0x9B, 0xFF);

    printf("HW CRC: 0x%02X, SW CRC: 0x%02X\n", hw_crc, sw_crc);
}
```

### 2. 성능 문제

**해결**: DMA 사용 또는 테이블 기반 소프트웨어 CRC

```c
/* CRC 테이블 (256 바이트) */
static const uint8_t crc8_table[256] = {
    /* 미리 계산된 CRC 값 */
};

uint8_t CRC8_Table(uint8_t *data, uint32_t length)
{
    uint8_t crc = 0xFF;

    for (uint32_t i = 0; i < length; i++)
    {
        crc = crc8_table[crc ^ data[i]];
    }

    return crc;
}
```

## 빌드 및 실행

```bash
make clean && make -j8
openocd -f board/stm32h745i-disco.cfg -c "program build/main.elf verify reset exit"
```

## 참고 자료

- STM32H745/755 Reference Manual (RM0399) - CRC 챕터
- AN4187: Using the CRC peripheral in STM32 family
- "A Painless Guide to CRC Error Detection Algorithms"

## 라이선스

BSD 3-Clause License (STMicroelectronics)
