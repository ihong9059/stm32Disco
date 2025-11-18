# RNG_MultiRNG 예제

## 개요

이 예제는 STM32H745I-DISCO의 하드웨어 난수 생성기(RNG)를 사용하여 고품질 난수를 생성하는 방법을 보여줍니다. FIPS PUB 140-2 표준을 준수하며, 암호화, 보안 키 생성, 무작위 데이터 생성 등에 활용할 수 있습니다.

## 하드웨어 요구사항

- **보드**: STM32H745I-DISCO
- **RNG**: 하드웨어 난수 생성기 (True Random Number Generator)
- **LED**:
  - LED1 (녹색): 난수 생성 성공
  - LED2 (주황색): 난수 생성 진행 중
  - LED3 (빨간색): RNG 오류 (시드 오류, 클록 오류)
  - LED4 (파란색): 통계 테스트 통과
- **버튼**: USER 버튼 (난수 배열 생성 트리거)

## STM32H7 RNG 특징

### 1. True Random Number Generator
- **하드웨어 기반**: 물리적 노이즈 소스 활용
- **32-bit 난수**: 한 번에 32비트 난수 생성
- **FIPS 140-2 준수**: 미국 연방 정보 처리 표준
- **인터럽트 지원**: 난수 준비 시 인터럽트 발생

### 2. 난수 품질 보장
- **시드 오류 검출**: 엔트로피 부족 감지
- **클록 오류 검출**: 클록 이상 감지
- **자동 재시도**: 오류 발생 시 자동으로 재생성

## 소프트웨어 설명

### RNG 초기화

```c
/* rng_manager.h */

RNG_HandleTypeDef hrng;

/* RNG 상태 */
typedef enum
{
    RNG_STATE_READY = 0,
    RNG_STATE_BUSY,
    RNG_STATE_ERROR_SEED,
    RNG_STATE_ERROR_CLOCK
} RNG_State_t;

/* RNG 통계 */
typedef struct
{
    uint32_t total_generated;
    uint32_t seed_errors;
    uint32_t clock_errors;
    uint32_t average_time_us;
} RNG_Statistics_t;

RNG_Statistics_t rng_stats = {0};

/**
  * @brief  RNG 초기화
  */
HAL_StatusTypeDef RNG_Init(void)
{
    /* RNG 클록 활성화 (PLL1Q) */
    __HAL_RCC_RNG_CLK_ENABLE();

    /* RNG 핸들 설정 */
    hrng.Instance = RNG;
    hrng.Init.ClockErrorDetection = RNG_CED_ENABLE;

    if (HAL_RNG_Init(&hrng) != HAL_OK)
    {
        printf("RNG initialization failed\n");
        return HAL_ERROR;
    }

    printf("RNG initialized successfully\n");
    printf("  Clock Error Detection: Enabled\n");

    return HAL_OK;
}

/**
  * @brief  32-bit 난수 생성
  * @param  random_number: 생성된 난수를 저장할 포인터
  * @retval HAL status
  */
HAL_StatusTypeDef RNG_Generate32(uint32_t *random_number)
{
    HAL_StatusTypeDef status;
    uint32_t start_time = DWT->CYCCNT;

    /* 난수 생성 */
    status = HAL_RNG_GenerateRandomNumber(&hrng, random_number);

    if (status == HAL_OK)
    {
        uint32_t cycles = DWT->CYCCNT - start_time;
        rng_stats.total_generated++;
        rng_stats.average_time_us = cycles / 480;  /* 480MHz → μs */

        BSP_LED_On(LED1);
    }
    else
    {
        /* 오류 처리 */
        uint32_t error = HAL_RNG_GetError(&hrng);

        if (error & HAL_RNG_ERROR_SEED)
        {
            printf("RNG Seed Error detected\n");
            rng_stats.seed_errors++;
            BSP_LED_On(LED3);
        }

        if (error & HAL_RNG_ERROR_CLOCK)
        {
            printf("RNG Clock Error detected\n");
            rng_stats.clock_errors++;
            BSP_LED_On(LED3);
        }
    }

    return status;
}

/**
  * @brief  난수 배열 생성
  * @param  buffer: 난수를 저장할 버퍼
  * @param  length: 배열 크기 (워드 단위)
  */
HAL_StatusTypeDef RNG_GenerateArray(uint32_t *buffer, uint32_t length)
{
    printf("Generating %lu random numbers...\n", length);

    for (uint32_t i = 0; i < length; i++)
    {
        if (RNG_Generate32(&buffer[i]) != HAL_OK)
        {
            printf("Failed to generate random number at index %lu\n", i);
            return HAL_ERROR;
        }

        /* 진행 상황 표시 */
        if (i % 100 == 0)
        {
            BSP_LED_Toggle(LED2);
        }
    }

    printf("Successfully generated %lu random numbers\n", length);
    BSP_LED_Off(LED2);

    return HAL_OK;
}

/**
  * @brief  인터럽트 방식 난수 생성
  */
volatile uint8_t rng_ready_flag = 0;
uint32_t generated_random = 0;

HAL_StatusTypeDef RNG_Generate32_IT(void)
{
    rng_ready_flag = 0;

    /* 인터럽트 방식 난수 생성 */
    if (HAL_RNG_GenerateRandomNumber_IT(&hrng) != HAL_OK)
    {
        return HAL_ERROR;
    }

    /* 난수 생성 완료 대기 */
    while (rng_ready_flag == 0)
    {
        __WFI();  /* 저전력 대기 */
    }

    return HAL_OK;
}

/**
  * @brief  RNG 데이터 준비 콜백
  */
void HAL_RNG_ReadyDataCallback(RNG_HandleTypeDef *hrng, uint32_t random32bit)
{
    generated_random = random32bit;
    rng_ready_flag = 1;
}

/**
  * @brief  RNG 오류 콜백
  */
void HAL_RNG_ErrorCallback(RNG_HandleTypeDef *hrng)
{
    uint32_t error = HAL_RNG_GetError(hrng);

    if (error & HAL_RNG_ERROR_SEED)
    {
        rng_stats.seed_errors++;
    }

    if (error & HAL_RNG_ERROR_CLOCK)
    {
        rng_stats.clock_errors++;
    }

    BSP_LED_On(LED3);
}
```

### 난수 품질 테스트

```c
/* rng_quality.c */

/**
  * @brief  단순 빈도 테스트 (Monobit Test)
  * @param  data: 난수 데이터
  * @param  length: 데이터 길이 (바이트)
  * @retval 1: 통과, 0: 실패
  */
uint8_t RNG_MonobitTest(uint8_t *data, uint32_t length)
{
    uint32_t ones = 0;
    uint32_t zeros = 0;

    /* 비트 카운트 */
    for (uint32_t i = 0; i < length; i++)
    {
        for (uint8_t bit = 0; bit < 8; bit++)
        {
            if (data[i] & (1 << bit))
                ones++;
            else
                zeros++;
        }
    }

    uint32_t total_bits = length * 8;

    printf("Monobit Test:\n");
    printf("  Total bits: %lu\n", total_bits);
    printf("  Ones: %lu (%.2f%%)\n", ones, (ones * 100.0f) / total_bits);
    printf("  Zeros: %lu (%.2f%%)\n", zeros, (zeros * 100.0f) / total_bits);

    /* 0과 1의 비율이 45%~55% 범위인지 확인 */
    float ones_ratio = (ones * 100.0f) / total_bits;

    if (ones_ratio >= 45.0f && ones_ratio <= 55.0f)
    {
        printf("  Result: PASS\n");
        return 1;
    }
    else
    {
        printf("  Result: FAIL\n");
        return 0;
    }
}

/**
  * @brief  런(Run) 테스트
  * @param  data: 난수 데이터
  * @param  length: 데이터 길이
  */
uint8_t RNG_RunTest(uint8_t *data, uint32_t length)
{
    uint32_t runs = 0;
    uint8_t last_bit = 0;
    uint8_t current_bit;

    /* 첫 번째 비트 */
    last_bit = (data[0] & 0x01);

    /* 런 카운트 */
    for (uint32_t i = 0; i < length; i++)
    {
        for (uint8_t bit = 0; bit < 8; bit++)
        {
            current_bit = (data[i] >> bit) & 0x01;

            if (current_bit != last_bit)
            {
                runs++;
                last_bit = current_bit;
            }
        }
    }

    uint32_t total_bits = length * 8;
    float expected_runs = (total_bits - 1) / 2.0f;
    float deviation = fabsf((float)runs - expected_runs);
    float max_deviation = 2.0f * sqrtf(total_bits - 1);

    printf("Run Test:\n");
    printf("  Total runs: %lu\n", runs);
    printf("  Expected: %.2f\n", expected_runs);
    printf("  Deviation: %.2f (max: %.2f)\n", deviation, max_deviation);

    if (deviation <= max_deviation)
    {
        printf("  Result: PASS\n");
        return 1;
    }
    else
    {
        printf("  Result: FAIL\n");
        return 0;
    }
}

/**
  * @brief  포커(Poker) 테스트
  */
uint8_t RNG_PokerTest(uint8_t *data, uint32_t length)
{
    uint32_t counts[16] = {0};  /* 4-bit 패턴 카운트 */

    /* 4-bit 패턴 카운트 */
    for (uint32_t i = 0; i < length; i++)
    {
        uint8_t high_nibble = (data[i] >> 4) & 0x0F;
        uint8_t low_nibble = data[i] & 0x0F;

        counts[high_nibble]++;
        counts[low_nibble]++;
    }

    /* 카이제곱 통계량 계산 */
    float chi_square = 0.0f;
    float expected = (length * 2) / 16.0f;

    for (uint8_t i = 0; i < 16; i++)
    {
        float deviation = (float)counts[i] - expected;
        chi_square += (deviation * deviation) / expected;
    }

    printf("Poker Test:\n");
    printf("  Chi-square: %.2f\n", chi_square);

    /* 자유도 15, 유의수준 5%: 임계값 약 25 */
    if (chi_square < 25.0f)
    {
        printf("  Result: PASS\n");
        return 1;
    }
    else
    {
        printf("  Result: FAIL\n");
        return 0;
    }
}

/**
  * @brief  전체 품질 테스트 실행
  */
uint8_t RNG_QualityTest(uint32_t *random_data, uint32_t num_words)
{
    uint8_t *byte_data = (uint8_t*)random_data;
    uint32_t num_bytes = num_words * 4;

    printf("\n=== RNG Quality Test ===\n");

    uint8_t monobit_pass = RNG_MonobitTest(byte_data, num_bytes);
    uint8_t run_pass = RNG_RunTest(byte_data, num_bytes);
    uint8_t poker_pass = RNG_PokerTest(byte_data, num_bytes);

    printf("\nOverall Result: ");

    if (monobit_pass && run_pass && poker_pass)
    {
        printf("ALL TESTS PASSED\n");
        BSP_LED_On(LED4);
        return 1;
    }
    else
    {
        printf("SOME TESTS FAILED\n");
        BSP_LED_On(LED3);
        return 0;
    }

    printf("========================\n\n");
}
```

### 실전 응용 예제

```c
/* rng_applications.c */

/**
  * @brief  UUID (Universally Unique Identifier) 생성
  */
typedef struct
{
    uint32_t time_low;
    uint16_t time_mid;
    uint16_t time_hi_and_version;
    uint8_t  clock_seq_hi_and_reserved;
    uint8_t  clock_seq_low;
    uint8_t  node[6];
} UUID_t;

void RNG_GenerateUUID(UUID_t *uuid)
{
    uint32_t random[4];

    /* 128-bit 난수 생성 */
    RNG_GenerateArray(random, 4);

    /* UUID 구조체 채우기 */
    uuid->time_low = random[0];
    uuid->time_mid = (uint16_t)(random[1] >> 16);
    uuid->time_hi_and_version = (uint16_t)(random[1] & 0xFFFF);
    uuid->clock_seq_hi_and_reserved = (uint8_t)(random[2] >> 24);
    uuid->clock_seq_low = (uint8_t)(random[2] >> 16);

    uuid->node[0] = (uint8_t)(random[2] >> 8);
    uuid->node[1] = (uint8_t)(random[2]);
    uuid->node[2] = (uint8_t)(random[3] >> 24);
    uuid->node[3] = (uint8_t)(random[3] >> 16);
    uuid->node[4] = (uint8_t)(random[3] >> 8);
    uuid->node[5] = (uint8_t)(random[3]);

    /* UUID 버전 4 (random) 설정 */
    uuid->time_hi_and_version &= 0x0FFF;
    uuid->time_hi_and_version |= 0x4000;  /* Version 4 */

    /* Variant 설정 (RFC 4122) */
    uuid->clock_seq_hi_and_reserved &= 0x3F;
    uuid->clock_seq_hi_and_reserved |= 0x80;

    printf("Generated UUID: %08lX-%04X-%04X-%02X%02X-%02X%02X%02X%02X%02X%02X\n",
           uuid->time_low, uuid->time_mid, uuid->time_hi_and_version,
           uuid->clock_seq_hi_and_reserved, uuid->clock_seq_low,
           uuid->node[0], uuid->node[1], uuid->node[2],
           uuid->node[3], uuid->node[4], uuid->node[5]);
}

/**
  * @brief  AES 암호화 키 생성
  */
void RNG_GenerateAESKey(uint8_t *key, uint32_t key_size)
{
    /* key_size: 128, 192, 256 bits → 16, 24, 32 bytes */
    uint32_t words = key_size / 4;

    RNG_GenerateArray((uint32_t*)key, words);

    printf("Generated AES-%u key:\n", key_size * 8);
    printf("  ");
    for (uint32_t i = 0; i < key_size; i++)
    {
        printf("%02X", key[i]);
        if ((i + 1) % 16 == 0)
            printf("\n  ");
    }
    printf("\n");
}

/**
  * @brief  난수 기반 지연 (타이밍 공격 방지)
  */
void RNG_RandomDelay(uint32_t min_ms, uint32_t max_ms)
{
    uint32_t random;
    RNG_Generate32(&random);

    uint32_t delay_ms = min_ms + (random % (max_ms - min_ms + 1));

    HAL_Delay(delay_ms);
}

/**
  * @brief  무작위 배열 셔플 (Fisher-Yates 알고리즘)
  */
void RNG_ShuffleArray(uint32_t *array, uint32_t length)
{
    for (uint32_t i = length - 1; i > 0; i--)
    {
        uint32_t random;
        RNG_Generate32(&random);

        uint32_t j = random % (i + 1);

        /* Swap */
        uint32_t temp = array[i];
        array[i] = array[j];
        array[j] = temp;
    }
}

/**
  * @brief  무작위 비밀번호 생성
  */
void RNG_GeneratePassword(char *password, uint32_t length)
{
    const char charset[] = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*";
    uint32_t charset_size = strlen(charset);

    for (uint32_t i = 0; i < length; i++)
    {
        uint32_t random;
        RNG_Generate32(&random);

        password[i] = charset[random % charset_size];
    }

    password[length] = '\0';

    printf("Generated password (%lu chars): %s\n", length, password);
}
```

## 메인 코드

```c
/* main.c */

#define RANDOM_ARRAY_SIZE  1000

uint32_t random_array[RANDOM_ARRAY_SIZE];

int main(void)
{
    /* HAL 초기화 */
    HAL_Init();
    SystemClock_Config();

    /* DWT 초기화 */
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
    printf("========================================\n");
    printf("  STM32H7 RNG (Random Number Generator)\n");
    printf("  FIPS PUB 140-2 Compliant\n");
    printf("========================================\n\n");

    /* RNG 초기화 */
    if (RNG_Init() != HAL_OK)
    {
        Error_Handler();
    }

    /* 단일 난수 생성 테스트 */
    printf("\n--- Single Random Number Test ---\n");
    for (uint8_t i = 0; i < 10; i++)
    {
        uint32_t random;
        if (RNG_Generate32(&random) == HAL_OK)
        {
            printf("Random[%u]: 0x%08lX (%lu)\n", i, random, random);
        }
    }

    /* UUID 생성 */
    printf("\n--- UUID Generation ---\n");
    UUID_t uuid;
    RNG_GenerateUUID(&uuid);

    /* AES 키 생성 */
    printf("\n--- AES Key Generation ---\n");
    uint8_t aes_key[32];
    RNG_GenerateAESKey(aes_key, sizeof(aes_key));

    /* 비밀번호 생성 */
    printf("\n--- Password Generation ---\n");
    char password[17];
    RNG_GeneratePassword(password, 16);

    /* 대용량 난수 배열 생성 */
    printf("\n--- Large Array Generation ---\n");
    if (RNG_GenerateArray(random_array, RANDOM_ARRAY_SIZE) == HAL_OK)
    {
        /* 품질 테스트 */
        RNG_QualityTest(random_array, RANDOM_ARRAY_SIZE);
    }

    /* 통계 출력 */
    printf("\n--- RNG Statistics ---\n");
    printf("Total Generated: %lu\n", rng_stats.total_generated);
    printf("Seed Errors: %lu\n", rng_stats.seed_errors);
    printf("Clock Errors: %lu\n", rng_stats.clock_errors);
    printf("Average Time: %lu μs\n", rng_stats.average_time_us);

    /* 메인 루프 */
    while (1)
    {
        BSP_LED_Toggle(LED1);
        HAL_Delay(1000);
    }
}

/**
  * @brief  버튼 인터럽트 콜백
  */
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == BUTTON_USER_PIN)
    {
        printf("\n[Button Pressed] Generating new random array...\n");

        BSP_LED_On(LED2);

        /* 새로운 난수 배열 생성 */
        if (RNG_GenerateArray(random_array, RANDOM_ARRAY_SIZE) == HAL_OK)
        {
            /* 품질 테스트 */
            RNG_QualityTest(random_array, RANDOM_ARRAY_SIZE);
        }

        BSP_LED_Off(LED2);
    }
}
```

## 성능 측정

```c
/* performance.c */

void RNG_PerformanceBenchmark(void)
{
    printf("\n=== RNG Performance Benchmark ===\n");

    /* 단일 난수 생성 속도 */
    uint32_t start_tick = HAL_GetTick();
    uint32_t random;

    for (uint32_t i = 0; i < 10000; i++)
    {
        RNG_Generate32(&random);
    }

    uint32_t duration_ms = HAL_GetTick() - start_tick;

    printf("10,000 random numbers:\n");
    printf("  Time: %lu ms\n", duration_ms);
    printf("  Throughput: %.2f numbers/sec\n", 10000.0f / (duration_ms / 1000.0f));

    /* 대용량 배열 생성 속도 */
    uint32_t large_array[10000];

    start_tick = HAL_GetTick();
    RNG_GenerateArray(large_array, 10000);
    duration_ms = HAL_GetTick() - start_tick;

    printf("\n40,000 bytes generation:\n");
    printf("  Time: %lu ms\n", duration_ms);
    printf("  Throughput: %.2f KB/sec\n", (40.0f * 1000.0f) / duration_ms);

    printf("=================================\n\n");
}
```

## 보안 고려사항

### 1. 시드 오류 처리

```c
HAL_StatusTypeDef RNG_Generate_Secure(uint32_t *random_number)
{
    uint32_t retries = 0;
    const uint32_t MAX_RETRIES = 10;

    while (retries < MAX_RETRIES)
    {
        if (RNG_Generate32(random_number) == HAL_OK)
        {
            return HAL_OK;
        }

        retries++;
        HAL_Delay(1);  /* 짧은 대기 */
    }

    printf("Failed to generate random number after %u retries\n", MAX_RETRIES);
    return HAL_ERROR;
}
```

### 2. 엔트로피 풀

```c
#define ENTROPY_POOL_SIZE  256

typedef struct
{
    uint32_t pool[ENTROPY_POOL_SIZE];
    uint32_t index;
    uint32_t mix_count;
} EntropyPool_t;

EntropyPool_t entropy_pool;

void EntropyPool_Mix(void)
{
    uint32_t random;

    /* 새로운 엔트로피 추가 */
    if (RNG_Generate32(&random) == HAL_OK)
    {
        entropy_pool.pool[entropy_pool.index] ^= random;
        entropy_pool.pool[entropy_pool.index] ^= HAL_GetTick();

        entropy_pool.index = (entropy_pool.index + 1) % ENTROPY_POOL_SIZE;
        entropy_pool.mix_count++;
    }
}

uint32_t EntropyPool_GetRandom(void)
{
    /* 풀에서 난수 추출 (XOR 믹싱) */
    uint32_t result = 0;

    for (uint32_t i = 0; i < 8; i++)
    {
        uint32_t idx = (entropy_pool.index + i) % ENTROPY_POOL_SIZE;
        result ^= entropy_pool.pool[idx];
    }

    /* 풀 업데이트 */
    EntropyPool_Mix();

    return result;
}
```

## 문제 해결

### 1. RNG 초기화 실패

**증상**: HAL_RNG_Init() 실패

**해결**:
```c
void Debug_RNG_Clock(void)
{
    /* RNG 클록 소스 확인 (PLL1Q) */
    RCC_PeriphCLKInitTypeDef PeriphClkInit = {0};

    HAL_RCCEx_GetPeriphCLKConfig(&PeriphClkInit);

    printf("RNG Clock Source: PLL1Q\n");
    printf("PLL1Q Frequency: %lu Hz\n", HAL_RCCEx_GetPeriphCLKFreq(RCC_PERIPHCLK_RNG));

    /* 48MHz 여부 확인 */
    uint32_t rng_clk = HAL_RCCEx_GetPeriphCLKFreq(RCC_PERIPHCLK_RNG);
    if (rng_clk != 48000000)
    {
        printf("WARNING: RNG clock is not 48MHz!\n");
    }
}
```

### 2. 품질 테스트 실패

**원인**: 샘플 크기 부족 또는 클록 문제

**해결**: 더 많은 샘플 수집, 클록 확인

## 빌드 및 실행

```bash
make clean && make -j8
openocd -f board/stm32h745i-disco.cfg -c "program build/main.elf verify reset exit"
```

## 참고 자료

- STM32H745/755 Reference Manual (RM0399) - RNG 챕터
- FIPS PUB 140-2: Security Requirements for Cryptographic Modules
- AN4230: STM32 마이크로컨트롤러 난수 생성

## 라이선스

BSD 3-Clause License (STMicroelectronics)
