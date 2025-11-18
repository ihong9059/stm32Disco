# TIM_DMA - DMA를 이용한 타이머 PWM 제어

## 개요

이 예제는 STM32H745의 타이머와 DMA를 결합하여 CPU 개입 없이 PWM 듀티 사이클을 동적으로 변경하는 방법을 보여줍니다. TIM3 채널 1을 사용하여 PWM 신호를 생성하고, DMA가 자동으로 비교 레지스터(CCR)를 업데이트하여 다양한 패턴의 PWM 신호를 생성합니다.

## 하드웨어 요구사항

- **STM32H745I-DISCO 보드**
- **PWM 출력**: PB4 (TIM3_CH1)
- **주파수**: 1kHz PWM
- **LED 또는 오실로스코프**: PWM 신호 확인용

## 주요 기능

### DMA 기반 PWM 제어
- **자동 업데이트**: CPU 개입 없이 듀티 사이클 변경
- **메모리 버퍼**: 미리 정의된 듀티 사이클 패턴
- **순환 모드**: 패턴 반복 재생
- **고정밀 타이밍**: 하드웨어 타이머로 정확한 주기 보장

### 응용 분야
- **LED 밝기 조절**: 부드러운 페이드 인/아웃
- **모터 속도 제어**: 가변 속도 프로파일
- **신호 생성**: 임의 파형 생성
- **전력 제어**: 스위칭 전원 공급 장치

## 동작 원리

### 시스템 구성
```
┌──────────┐
│  Memory  │
│  Buffer  │  [10%, 20%, 30%, ..., 100%]
└────┬─────┘
     │
     │ DMA Transfer
     ▼
┌──────────┐         ┌──────────┐         ┌──────────┐
│   DMA    │────────►│  TIM3    │────────►│   PB4    │
│ Stream 2 │         │   CCR1   │         │ (PWM Out)│
└──────────┘         └──────────┘         └──────────┘
     ▲                    │
     │                    │
     └────────────────────┘
         Update Event
```

### DMA 업데이트 메커니즘
```
타이머 업데이트 이벤트 발생 시마다 DMA가 CCR 업데이트

Timer:    ┌───┐   ┌───┐   ┌───┐   ┌───┐
Update    │   │   │   │   │   │   │   │
Event:    └───┘   └───┘   └───┘   └───┘

DMA:      ─┬───┬───┬───┬───┬───┬───┬───
CCR       │10%│20%│30%│40%│50%│60%│70%│
Update:   └───┴───┴───┴───┴───┴───┴───

PWM:      ▁▂▃▄▅▆▇█▇▆▅▄▃▂▁ (부드러운 변화)
```

## 코드 구조

### 1. 타이머 PWM 초기화

```c
// 전역 변수
TIM_HandleTypeDef htim3;
DMA_HandleTypeDef hdma_tim3_ch1;

// PWM 주파수: 1kHz
// ARR = 200MHz / (PSC+1) / Freq - 1
// ARR = 200MHz / 1 / 1000 - 1 = 199999

#define TIM_PERIOD  19999  // 1kHz @ 200MHz APB1 Timer Clock
#define PWM_STEPS   100    // 듀티 사이클 스텝 수

// 듀티 사이클 버퍼 (0% ~ 100%)
uint16_t pwm_duty_buffer[PWM_STEPS];

void TIM3_PWM_Init(void)
{
  TIM_OC_InitTypeDef sConfigOC = {0};
  TIM_MasterConfigTypeDef sMasterConfig = {0};

  // TIM3 클럭 활성화
  __HAL_RCC_TIM3_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 타이머 기본 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  htim3.Instance = TIM3;
  htim3.Init.Prescaler = 0;                    // 프리스케일러: 200MHz
  htim3.Init.CounterMode = TIM_COUNTERMODE_UP;
  htim3.Init.Period = TIM_PERIOD;              // ARR = 19999 (1kHz)
  htim3.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
  htim3.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_ENABLE;

  if (HAL_TIM_PWM_Init(&htim3) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // PWM 채널 설정 (채널 1)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  sConfigOC.OCMode = TIM_OCMODE_PWM1;          // PWM 모드 1
  sConfigOC.Pulse = 0;                         // 초기 듀티 사이클: 0%
  sConfigOC.OCPolarity = TIM_OCPOLARITY_HIGH;  // 양극성
  sConfigOC.OCFastMode = TIM_OCFAST_DISABLE;

  if (HAL_TIM_PWM_ConfigChannel(&htim3, &sConfigOC, TIM_CHANNEL_1) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // Master 모드 설정 (선택 사항)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  sMasterConfig.MasterOutputTrigger = TIM_TRGO_UPDATE;
  sMasterConfig.MasterSlaveMode = TIM_MASTERSLAVEMODE_DISABLE;

  if (HAL_TIMEx_MasterConfigSynchronization(&htim3, &sMasterConfig) != HAL_OK)
  {
    Error_Handler();
  }
}
```

### 2. DMA 설정

```c
void DMA_Init(void)
{
  // DMA 클럭 활성화
  __HAL_RCC_DMA1_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // DMA 핸들 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  hdma_tim3_ch1.Instance = DMA1_Stream2;
  hdma_tim3_ch1.Init.Request = DMA_REQUEST_TIM3_CH1;  // TIM3 CH1/UP
  hdma_tim3_ch1.Init.Direction = DMA_MEMORY_TO_PERIPH; // 메모리 -> 타이머 CCR
  hdma_tim3_ch1.Init.PeriphInc = DMA_PINC_DISABLE;     // CCR 주소 고정
  hdma_tim3_ch1.Init.MemInc = DMA_MINC_ENABLE;         // 메모리 주소 증가
  hdma_tim3_ch1.Init.PeriphDataAlignment = DMA_PDATAALIGN_HALFWORD;  // 16비트
  hdma_tim3_ch1.Init.MemDataAlignment = DMA_MDATAALIGN_HALFWORD;     // 16비트
  hdma_tim3_ch1.Init.Mode = DMA_CIRCULAR;              // 순환 모드
  hdma_tim3_ch1.Init.Priority = DMA_PRIORITY_HIGH;
  hdma_tim3_ch1.Init.FIFOMode = DMA_FIFOMODE_DISABLE;

  if (HAL_DMA_Init(&hdma_tim3_ch1) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 타이머와 DMA 연결
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // TIM3 Update 이벤트에 DMA 연결
  __HAL_LINKDMA(&htim3, hdma[TIM_DMA_ID_UPDATE], hdma_tim3_ch1);

  // DMA 인터럽트 활성화 (선택 사항)
  HAL_NVIC_SetPriority(DMA1_Stream2_IRQn, 2, 0);
  HAL_NVIC_EnableIRQ(DMA1_Stream2_IRQn);
}
```

### 3. GPIO 설정

```c
void GPIO_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  // GPIOB 클럭 활성화
  __HAL_RCC_GPIOB_CLK_ENABLE();

  // PB4를 TIM3_CH1 Alternate Function으로 설정
  GPIO_InitStruct.Pin = GPIO_PIN_4;
  GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;              // Alternate Function Push-Pull
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
  GPIO_InitStruct.Alternate = GPIO_AF2_TIM3;           // AF2 = TIM3

  HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
}
```

### 4. 듀티 사이클 버퍼 초기화

```c
void PWM_Buffer_Init(void)
{
  // 예제 1: 선형 증가 (0% -> 100%)
  for (uint16_t i = 0; i < PWM_STEPS; i++)
  {
    pwm_duty_buffer[i] = (TIM_PERIOD * i) / PWM_STEPS;
  }

  // 예제 2: 사인파 (부드러운 페이드)
  // for (uint16_t i = 0; i < PWM_STEPS; i++)
  // {
  //   float angle = (2.0f * M_PI * i) / PWM_STEPS;
  //   float sine = (sinf(angle) + 1.0f) / 2.0f;  // 0.0 ~ 1.0
  //   pwm_duty_buffer[i] = (uint16_t)(TIM_PERIOD * sine);
  // }

  // 예제 3: 삼각파 (0% -> 100% -> 0%)
  // for (uint16_t i = 0; i < PWM_STEPS / 2; i++)
  // {
  //   pwm_duty_buffer[i] = (TIM_PERIOD * i * 2) / PWM_STEPS;
  //   pwm_duty_buffer[PWM_STEPS - 1 - i] = pwm_duty_buffer[i];
  // }
}
```

### 5. DMA PWM 시작

```c
void TIM_DMA_Start(void)
{
  // DMA를 사용하여 타이머 CCR 업데이트
  if (HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1) != HAL_OK)
  {
    Error_Handler();
  }

  // DMA 전송 시작
  // 타이머 업데이트 이벤트마다 pwm_duty_buffer에서 CCR1로 전송
  if (HAL_DMA_Start(&hdma_tim3_ch1,
                    (uint32_t)pwm_duty_buffer,      // 소스: 메모리 버퍼
                    (uint32_t)&htim3.Instance->CCR1, // 목적지: CCR1 레지스터
                    PWM_STEPS) != HAL_OK)            // 전송 크기
  {
    Error_Handler();
  }

  // 타이머 DMA 요청 활성화
  __HAL_TIM_ENABLE_DMA(&htim3, TIM_DMA_UPDATE);

  // 타이머 시작
  __HAL_TIM_ENABLE(&htim3);
}
```

### 6. 메인 함수

```c
int main(void)
{
  // HAL 초기화
  HAL_Init();

  // 시스템 클럭 설정 (400MHz CM7, 200MHz APB)
  SystemClock_Config();

  // 주변장치 초기화
  GPIO_Init();
  DMA_Init();
  TIM3_PWM_Init();

  // PWM 듀티 사이클 버퍼 준비
  PWM_Buffer_Init();

  // DMA PWM 시작
  TIM_DMA_Start();

  // 무한 루프
  while (1)
  {
    // PWM은 DMA가 자동으로 제어
    // CPU는 다른 작업 수행 가능

    HAL_Delay(1000);

    // 선택: 런타임에 버퍼 업데이트
    // Update_PWM_Buffer();
  }
}
```

## PWM 파형 패턴

### 1. 선형 램프 (Ramp)
```c
// 0% -> 100% 선형 증가
void Generate_Ramp_Up(void)
{
  for (uint16_t i = 0; i < PWM_STEPS; i++)
  {
    pwm_duty_buffer[i] = (TIM_PERIOD * i) / PWM_STEPS;
  }
}

// 100% -> 0% 선형 감소
void Generate_Ramp_Down(void)
{
  for (uint16_t i = 0; i < PWM_STEPS; i++)
  {
    pwm_duty_buffer[i] = TIM_PERIOD - ((TIM_PERIOD * i) / PWM_STEPS);
  }
}
```

### 2. 사인파 (Sine)
```c
#include <math.h>

void Generate_Sine_Wave(void)
{
  for (uint16_t i = 0; i < PWM_STEPS; i++)
  {
    float angle = (2.0f * M_PI * i) / PWM_STEPS;
    float sine = (sinf(angle) + 1.0f) / 2.0f;  // -1~1 -> 0~1
    pwm_duty_buffer[i] = (uint16_t)(TIM_PERIOD * sine);
  }
}

// 결과: 부드러운 페이드 인/아웃
```

### 3. 삼각파 (Triangle)
```c
void Generate_Triangle_Wave(void)
{
  uint16_t half = PWM_STEPS / 2;

  // 상승 엣지
  for (uint16_t i = 0; i < half; i++)
  {
    pwm_duty_buffer[i] = (TIM_PERIOD * i * 2) / PWM_STEPS;
  }

  // 하강 엣지
  for (uint16_t i = half; i < PWM_STEPS; i++)
  {
    pwm_duty_buffer[i] = TIM_PERIOD - ((TIM_PERIOD * (i - half) * 2) / PWM_STEPS);
  }
}
```

### 4. 계단파 (Step)
```c
void Generate_Step_Wave(void)
{
  uint16_t quarter = PWM_STEPS / 4;

  for (uint16_t i = 0; i < PWM_STEPS; i++)
  {
    if (i < quarter)
      pwm_duty_buffer[i] = TIM_PERIOD / 4;      // 25%
    else if (i < quarter * 2)
      pwm_duty_buffer[i] = TIM_PERIOD / 2;      // 50%
    else if (i < quarter * 3)
      pwm_duty_buffer[i] = TIM_PERIOD * 3 / 4;  // 75%
    else
      pwm_duty_buffer[i] = TIM_PERIOD;          // 100%
  }
}
```

### 5. 펄스 트레인 (Pulse Train)
```c
void Generate_Pulse_Train(void)
{
  uint16_t pulse_width = PWM_STEPS / 10;  // 10% 온타임

  for (uint16_t i = 0; i < PWM_STEPS; i++)
  {
    if ((i % 20) < pulse_width)
      pwm_duty_buffer[i] = TIM_PERIOD;    // 100%
    else
      pwm_duty_buffer[i] = 0;              // 0%
  }
}
```

## 고급 기능

### 1. 런타임 버퍼 업데이트

```c
// DMA를 중지하지 않고 버퍼 업데이트
void Update_PWM_Buffer_Runtime(uint16_t *new_buffer)
{
  // 옵션 1: 전체 버퍼 교체
  memcpy(pwm_duty_buffer, new_buffer, sizeof(pwm_duty_buffer));

  // 캐시 클린 (메모리가 캐시 가능 영역인 경우)
  SCB_CleanDCache_by_Addr((uint32_t*)pwm_duty_buffer, sizeof(pwm_duty_buffer));

  // 옵션 2: 개별 값 업데이트
  // for (uint16_t i = 0; i < PWM_STEPS; i++)
  // {
  //   pwm_duty_buffer[i] = new_buffer[i];
  // }
}
```

### 2. 다중 채널 DMA PWM

```c
TIM_HandleTypeDef htim3;
DMA_HandleTypeDef hdma_tim3_ch1;
DMA_HandleTypeDef hdma_tim3_ch2;

uint16_t pwm_ch1_buffer[PWM_STEPS];
uint16_t pwm_ch2_buffer[PWM_STEPS];

void Multi_Channel_PWM_Init(void)
{
  // 채널 1 DMA 설정
  hdma_tim3_ch1.Instance = DMA1_Stream2;
  hdma_tim3_ch1.Init.Request = DMA_REQUEST_TIM3_CH1;
  // ... (기타 설정)
  HAL_DMA_Init(&hdma_tim3_ch1);

  // 채널 2 DMA 설정
  hdma_tim3_ch2.Instance = DMA1_Stream5;
  hdma_tim3_ch2.Init.Request = DMA_REQUEST_TIM3_CH2;
  // ... (기타 설정)
  HAL_DMA_Init(&hdma_tim3_ch2);

  // 채널별로 다른 파형 생성
  Generate_Sine_Wave_CH1(pwm_ch1_buffer);
  Generate_Triangle_Wave_CH2(pwm_ch2_buffer);

  // 두 채널 모두 시작
  HAL_DMA_Start(&hdma_tim3_ch1, (uint32_t)pwm_ch1_buffer,
                (uint32_t)&htim3.Instance->CCR1, PWM_STEPS);
  HAL_DMA_Start(&hdma_tim3_ch2, (uint32_t)pwm_ch2_buffer,
                (uint32_t)&htim3.Instance->CCR2, PWM_STEPS);

  __HAL_TIM_ENABLE_DMA(&htim3, TIM_DMA_CC1);
  __HAL_TIM_ENABLE_DMA(&htim3, TIM_DMA_CC2);
}
```

### 3. 가변 속도 제어

```c
// PWM 업데이트 속도 변경 (타이머 주파수 조절)
void Set_PWM_Update_Rate(uint32_t frequency_hz)
{
  // ARR 재계산
  uint32_t arr = (200000000 / frequency_hz) - 1;

  // 타이머 중지
  __HAL_TIM_DISABLE(&htim3);

  // ARR 업데이트
  __HAL_TIM_SET_AUTORELOAD(&htim3, arr);

  // 듀티 사이클 스케일링 (비율 유지)
  for (uint16_t i = 0; i < PWM_STEPS; i++)
  {
    pwm_duty_buffer[i] = (pwm_duty_buffer[i] * arr) / TIM_PERIOD;
  }

  // 타이머 재시작
  __HAL_TIM_ENABLE(&htim3);
}
```

### 4. DMA 콜백 활용

```c
// DMA Half/Full Transfer 콜백 등록
void DMA_Callback_Init(void)
{
  // 콜백 등록
  HAL_DMA_RegisterCallback(&hdma_tim3_ch1,
                           HAL_DMA_XFER_HALFCPLT_CB_ID,
                           DMA_HalfComplete_Callback);

  HAL_DMA_RegisterCallback(&hdma_tim3_ch1,
                           HAL_DMA_XFER_CPLT_CB_ID,
                           DMA_Complete_Callback);

  // 인터럽트 활성화
  __HAL_DMA_ENABLE_IT(&hdma_tim3_ch1, DMA_IT_HT | DMA_IT_TC);
}

void DMA_HalfComplete_Callback(DMA_HandleTypeDef *hdma)
{
  // 버퍼 전반부 업데이트 (더블 버퍼링)
  Update_First_Half_Buffer();
}

void DMA_Complete_Callback(DMA_HandleTypeDef *hdma)
{
  // 버퍼 후반부 업데이트
  Update_Second_Half_Buffer();
}
```

## 성능 최적화

### 메모리 배치

```c
// DTCM RAM 사용 (빠른 액세스)
uint16_t pwm_duty_buffer[PWM_STEPS] __attribute__((section(".dtcm")));

// 또는 AXI SRAM (DMA 성능 향상)
// uint16_t pwm_duty_buffer[PWM_STEPS] __attribute__((section(".sram1")));
```

### 타이밍 분석

```
타이머 주파수: 1kHz
PWM 스텝: 100
각 스텝 지속 시간: 1ms
전체 사이클: 100ms

DMA 대역폭:
= 100 전송/초 × 2바이트 = 200 바이트/초 (매우 낮음)
```

## 빌드 및 실행

### 빌드

```bash
# STM32CubeIDE
Build Project

# 명령줄
make clean && make -j8
```

### 테스트 설정

```
┌──────────────┐
│ STM32H745I   │
│              │
│   PB4 (TIM3) │──────┬──────► LED (저항 포함)
│              │      │
└──────────────┘      │
                      │
                      ▼
               ┌─────────────┐
               │Oscilloscope │ (파형 확인)
               └─────────────┘
```

### 파형 확인

```bash
# 오실로스코프 설정
- 주파수: 1kHz (타이머 기본 주파수)
- 듀티 사이클: 0% ~ 100% 변화 관찰
- 업데이트 속도: 100ms마다 전체 사이클 완료
```

## 트러블슈팅

### PWM이 출력되지 않는 경우

```c
// GPIO Alternate Function 확인
uint32_t afr = GPIOB->AFR[0];  // PB4 = AFR[0]
printf("PB4 AFR: 0x%08lX\n", (afr >> 16) & 0xF);  // AF2 (TIM3)

// 타이머 활성화 확인
if (htim3.Instance->CR1 & TIM_CR1_CEN)
{
  printf("TIM3 enabled\n");
}

// PWM 모드 확인
printf("TIM3 CCMR1: 0x%08lX\n", htim3.Instance->CCMR1);
```

### DMA가 CCR을 업데이트하지 않는 경우

```c
// DMA 활성화 확인
if (DMA1_Stream2->CR & DMA_SxCR_EN)
{
  printf("DMA enabled\n");
}

// 타이머 DMA 요청 확인
if (htim3.Instance->DIER & TIM_DIER_UDE)
{
  printf("TIM3 DMA request enabled\n");
}

// 전송 카운터 확인
printf("DMA NDTR: %lu\n", DMA1_Stream2->NDTR);
```

### 듀티 사이클이 예상과 다른 경우

```c
// CCR 값 확인
printf("CCR1: %lu (expected: %u)\n",
       htim3.Instance->CCR1, pwm_duty_buffer[0]);

// ARR 값 확인
printf("ARR: %lu (expected: %u)\n",
       htim3.Instance->ARR, TIM_PERIOD);

// 듀티 사이클 계산
float duty = ((float)htim3.Instance->CCR1 / TIM_PERIOD) * 100.0f;
printf("Duty Cycle: %.2f%%\n", duty);
```

## 참고 자료

- **RM0399**: STM32H745 Reference Manual
  - Chapter 36: General-purpose timers (TIM2 to TIM5)
  - Chapter 15: Direct memory access controller (DMA)
- **AN4776**: General-purpose timer cookbook for STM32 MCUs
- **예제 코드**: `STM32Cube_FW_H7/Projects/STM32H745I-DISCO/Examples/TIM/TIM_DMABurst`

## 관련 예제

- **TIM_PWMOutput**: 기본 PWM 출력
- **DMA_DMAMUX**: DMA 및 DMAMUX 설정
- **DAC_SignalsGeneration**: DAC DMA 파형 생성
