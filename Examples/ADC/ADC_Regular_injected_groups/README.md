# ADC_Regular_injected_groups - 정규 및 주입 그룹 변환

## 개요

이 예제는 STM32H745의 ADC에서 정규(Regular) 그룹과 주입(Injected) 그룹을 동시에 사용하는 방법을 보여줍니다. 정규 그룹은 타이머 트리거로 1kHz 주기적 샘플링을 수행하고, 주입 그룹은 소프트웨어 트리거로 내부 기준 전압(VREFINT)을 측정합니다.

## 하드웨어 요구사항

- **STM32H745I-DISCO 보드**
- **정규 그룹 입력**: PC0 (ADC12_INP10) - 외부 아날로그 신호
- **주입 그룹 입력**: VREFINT (내부 기준 전압, 약 1.21V)
- **타이머**: TIM2 - 정규 그룹 트리거 (1kHz)

## 주요 기능

### 정규(Regular) 그룹
- **트리거 소스**: TIM2 TRGO (1kHz)
- **채널**: ADC12_INP10 (PC0)
- **DMA 전송**: 순환 모드
- **용도**: 연속적인 센서 데이터 수집

### 주입(Injected) 그룹
- **트리거 소스**: 소프트웨어
- **채널**: VREFINT (내부 기준 전압)
- **인터럽트**: 변환 완료 시 ISR 호출
- **용도**: 보정 또는 긴급 측정

### 정규 vs 주입 그룹 비교
```
특성              │ 정규 그룹        │ 주입 그룹
─────────────────┼─────────────────┼──────────────────
최대 채널 수      │ 16개            │ 4개
우선순위         │ 낮음            │ 높음 (선점 가능)
트리거 소스      │ HW/SW           │ HW/SW
DMA 지원         │ Yes             │ No (인터럽트 사용)
데이터 레지스터   │ ADC_DR (1개)    │ ADC_JDRx (4개)
변환 중 중단     │ 불가능          │ 정규 그룹 중단 가능
```

## 동작 원리

### 시스템 구성
```
         ┌──────────┐
         │   TIM2   │
         │  (1kHz)  │
         └────┬─────┘
              │ TRGO
              ▼
         ┌──────────┐        ┌─────────┐
   PC0 ─►│   ADC    │───────►│  DMA    │
         │ Regular  │        │         │
         │  Group   │        └────┬────┘
         └──────────┘             │
              ▲                   ▼
              │             ┌──────────┐
         ┌────┴─────┐       │  Memory  │
         │ Software │       │  Buffer  │
         │ Trigger  │       └──────────┘
         └────┬─────┘
              │
              ▼
         ┌──────────┐        ┌──────────┐
VREFINT─►│   ADC    │───────►│   ISR    │
         │ Injected │        │          │
         │  Group   │        └──────────┘
         └──────────┘
```

### 우선순위 메커니즘
```
Time →

Regular:  ━━━R━━━R━━━R━━━R━━━R━━━R━━━
                 ▲       ▲
                 │       │
Injected: ━━━━━━━I━━━━━━━I━━━━━━━━━━━

R = Regular conversion (1ms)
I = Injected conversion (즉시 선점)
```

## 코드 구조

### 1. ADC 및 그룹 초기화

```c
// 전역 변수
ADC_HandleTypeDef hadc1;
TIM_HandleTypeDef htim2;

#define REGULAR_BUFFER_SIZE  128
uint16_t regular_buffer[REGULAR_BUFFER_SIZE];
uint16_t vrefint_value = 0;

void ADC_Regular_Injected_Init(void)
{
  ADC_ChannelConfTypeDef sConfig = {0};
  ADC_InjectionConfTypeDef sConfigInjected = {0};

  // ADC 클럭 활성화
  __HAL_RCC_ADC12_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // ADC 기본 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  hadc1.Instance = ADC1;
  hadc1.Init.ClockPrescaler = ADC_CLOCK_SYNC_PCLK_DIV4;
  hadc1.Init.Resolution = ADC_RESOLUTION_12B;
  hadc1.Init.ScanConvMode = ADC_SCAN_ENABLE;  // 스캔 모드 활성화
  hadc1.Init.EOCSelection = ADC_EOC_SINGLE_CONV;
  hadc1.Init.LowPowerAutoWait = DISABLE;
  hadc1.Init.ContinuousConvMode = DISABLE;  // 타이머 트리거 사용
  hadc1.Init.NbrOfConversion = 1;           // 정규 그룹: 1채널
  hadc1.Init.DiscontinuousConvMode = DISABLE;
  hadc1.Init.ExternalTrigConv = ADC_EXTERNALTRIG_T2_TRGO;
  hadc1.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_RISING;
  hadc1.Init.ConversionDataManagement = ADC_CONVERSIONDATA_DMA_CIRCULAR;
  hadc1.Init.Overrun = ADC_OVR_DATA_OVERWRITTEN;
  hadc1.Init.OversamplingMode = DISABLE;

  if (HAL_ADC_Init(&hadc1) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 정규 그룹 채널 설정 (PC0 = INP10)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  sConfig.Channel = ADC_CHANNEL_10;
  sConfig.Rank = ADC_REGULAR_RANK_1;
  sConfig.SamplingTime = ADC_SAMPLETIME_64CYCLES_5;
  sConfig.SingleDiff = ADC_SINGLE_ENDED;
  sConfig.OffsetNumber = ADC_OFFSET_NONE;
  sConfig.Offset = 0;

  if (HAL_ADC_ConfigChannel(&hadc1, &sConfig) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 주입 그룹 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  sConfigInjected.InjectedChannel = ADC_CHANNEL_VREFINT;
  sConfigInjected.InjectedRank = ADC_INJECTED_RANK_1;
  sConfigInjected.InjectedSamplingTime = ADC_SAMPLETIME_64CYCLES_5;
  sConfigInjected.InjectedSingleDiff = ADC_SINGLE_ENDED;
  sConfigInjected.InjectedOffsetNumber = ADC_OFFSET_NONE;
  sConfigInjected.InjectedOffset = 0;
  sConfigInjected.InjectedNbrOfConversion = 1;  // 1채널

  // 소프트웨어 트리거
  sConfigInjected.ExternalTrigInjecConv = ADC_INJECTED_SOFTWARE_START;
  sConfigInjected.ExternalTrigInjecConvEdge = ADC_EXTERNALTRIGINJECCONV_EDGE_NONE;

  // 자동 주입 비활성화 (수동 트리거)
  sConfigInjected.AutoInjectedConv = DISABLE;

  // 주입 변환 중단 정규 그룹 설정
  sConfigInjected.InjectedDiscontinuousConvMode = DISABLE;

  // 큐 모드 비활성화
  sConfigInjected.QueueInjectedContext = DISABLE;

  if (HAL_ADCEx_InjectedConfigChannel(&hadc1, &sConfigInjected) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // ADC 내부 채널 활성화 (VREFINT)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  HAL_ADCEx_EnableVREFINT();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // ADC 캘리브레이션
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  HAL_ADCEx_Calibration_Start(&hadc1, ADC_CALIB_OFFSET, ADC_SINGLE_ENDED);
}
```

### 2. 타이머 트리거 설정 (1kHz)

```c
void TIM2_Init(void)
{
  TIM_MasterConfigTypeDef sMasterConfig = {0};

  // TIM2 클럭 활성화
  __HAL_RCC_TIM2_CLK_ENABLE();

  // 타이머 기본 설정
  // APB1 Timer Clock = 200MHz
  // 목표 주파수 = 1kHz
  // Prescaler = 200 - 1 = 199  (200MHz / 200 = 1MHz)
  // Period = 1000 - 1 = 999    (1MHz / 1000 = 1kHz)

  htim2.Instance = TIM2;
  htim2.Init.Prescaler = 199;
  htim2.Init.CounterMode = TIM_COUNTERMODE_UP;
  htim2.Init.Period = 999;
  htim2.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
  htim2.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_DISABLE;

  if (HAL_TIM_Base_Init(&htim2) != HAL_OK)
  {
    Error_Handler();
  }

  // Master 모드 설정 (TRGO)
  sMasterConfig.MasterOutputTrigger = TIM_TRGO_UPDATE;
  sMasterConfig.MasterSlaveMode = TIM_MASTERSLAVEMODE_DISABLE;

  if (HAL_TIMEx_MasterConfigSynchronization(&htim2, &sMasterConfig) != HAL_OK)
  {
    Error_Handler();
  }
}
```

### 3. DMA 설정 (정규 그룹용)

```c
DMA_HandleTypeDef hdma_adc1;

void DMA_Init(void)
{
  // DMA 클럭 활성화
  __HAL_RCC_DMA1_CLK_ENABLE();

  // DMA 설정
  hdma_adc1.Instance = DMA1_Stream1;
  hdma_adc1.Init.Request = DMA_REQUEST_ADC1;
  hdma_adc1.Init.Direction = DMA_PERIPH_TO_MEMORY;
  hdma_adc1.Init.PeriphInc = DMA_PINC_DISABLE;
  hdma_adc1.Init.MemInc = DMA_MINC_ENABLE;
  hdma_adc1.Init.PeriphDataAlignment = DMA_PDATAALIGN_HALFWORD;  // 16비트
  hdma_adc1.Init.MemDataAlignment = DMA_MDATAALIGN_HALFWORD;     // 16비트
  hdma_adc1.Init.Mode = DMA_CIRCULAR;
  hdma_adc1.Init.Priority = DMA_PRIORITY_MEDIUM;
  hdma_adc1.Init.FIFOMode = DMA_FIFOMODE_DISABLE;

  if (HAL_DMA_Init(&hdma_adc1) != HAL_OK)
  {
    Error_Handler();
  }

  // ADC와 DMA 연결
  __HAL_LINKDMA(&hadc1, DMA_Handle, hdma_adc1);

  // DMA 인터럽트 활성화
  HAL_NVIC_SetPriority(DMA1_Stream1_IRQn, 1, 0);
  HAL_NVIC_EnableIRQ(DMA1_Stream1_IRQn);
}
```

### 4. GPIO 설정

```c
void GPIO_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  // GPIOC 클럭 활성화
  __HAL_RCC_GPIOC_CLK_ENABLE();

  // PC0를 아날로그 모드로 설정
  GPIO_InitStruct.Pin = GPIO_PIN_0;
  GPIO_InitStruct.Mode = GPIO_MODE_ANALOG;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);
}
```

### 5. ADC 시작

```c
void ADC_Start(void)
{
  // 정규 그룹 시작 (DMA 사용)
  if (HAL_ADC_Start_DMA(&hadc1,
                        (uint32_t*)regular_buffer,
                        REGULAR_BUFFER_SIZE) != HAL_OK)
  {
    Error_Handler();
  }

  // 타이머 시작 (정규 그룹 트리거)
  if (HAL_TIM_Base_Start(&htim2) != HAL_OK)
  {
    Error_Handler();
  }

  // 주입 그룹 인터럽트 활성화
  HAL_NVIC_SetPriority(ADC_IRQn, 0, 0);  // 높은 우선순위
  HAL_NVIC_EnableIRQ(ADC_IRQn);
}
```

### 6. 주입 그룹 트리거 (소프트웨어)

```c
void Trigger_Injected_Conversion(void)
{
  // 주입 그룹 변환 시작 (인터럽트 사용)
  if (HAL_ADCEx_InjectedStart_IT(&hadc1) != HAL_OK)
  {
    Error_Handler();
  }
}
```

### 7. 인터럽트 핸들러

```c
// 주입 그룹 변환 완료 콜백
void HAL_ADCEx_InjectedConvCpltCallback(ADC_HandleTypeDef* hadc)
{
  // VREFINT 값 읽기
  vrefint_value = HAL_ADCEx_InjectedGetValue(hadc, ADC_INJECTED_RANK_1);

  // VREFINT를 사용한 VDDA 계산
  // VREFINT_CAL = 1.21V (25°C에서 교정값)
  // VDDA = 3.3V * VREFINT_CAL / vrefint_value
  uint16_t vrefint_cal = *((uint16_t*)VREFINT_CAL_ADDR);
  float vdda = (3.3f * vrefint_cal) / vrefint_value;

  printf("VREFINT: %d, VDDA: %.3f V\n", vrefint_value, vdda);
}

// 정규 그룹 DMA 콜백
void HAL_ADC_ConvHalfCpltCallback(ADC_HandleTypeDef* hadc)
{
  // 버퍼 전반부 처리
  Process_Regular_Data(regular_buffer, REGULAR_BUFFER_SIZE / 2);
}

void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)
{
  // 버퍼 후반부 처리
  Process_Regular_Data(&regular_buffer[REGULAR_BUFFER_SIZE / 2],
                       REGULAR_BUFFER_SIZE / 2);
}

// 데이터 처리 함수
void Process_Regular_Data(uint16_t *buffer, uint32_t size)
{
  for (uint32_t i = 0; i < size; i++)
  {
    // ADC 값을 전압으로 변환
    float voltage = (buffer[i] * 3.3f) / 4095.0f;

    // 추가 처리...
  }
}
```

### 8. ADC 인터럽트 핸들러

```c
// stm32h7xx_it.c

void ADC_IRQHandler(void)
{
  HAL_ADC_IRQHandler(&hadc1);
}

void DMA1_Stream1_IRQHandler(void)
{
  HAL_DMA_IRQHandler(&hdma_adc1);
}
```

### 9. 메인 함수

```c
int main(void)
{
  // HAL 초기화
  HAL_Init();

  // 시스템 클럭 설정
  SystemClock_Config();

  // 주변장치 초기화
  GPIO_Init();
  DMA_Init();
  TIM2_Init();
  ADC_Regular_Injected_Init();

  // ADC 시작
  ADC_Start();

  uint32_t last_inject_time = 0;

  // 무한 루프
  while (1)
  {
    // 1초마다 주입 그룹 트리거
    if (HAL_GetTick() - last_inject_time >= 1000)
    {
      Trigger_Injected_Conversion();
      last_inject_time = HAL_GetTick();
    }

    // 기타 작업
    HAL_Delay(10);
  }
}
```

## VREFINT를 이용한 VDDA 보정

### VREFINT 원리
```
VREFINT = 내부 기준 전압 (약 1.21V, 온도 의존성 있음)

VDDA (실제) = VDDA (명목) × VREFINT_CAL / VREFINT_DATA

여기서:
- VDDA (명목) = 3.3V
- VREFINT_CAL = 공장 교정값 (0x1FF1E860 주소)
- VREFINT_DATA = 현재 측정값
```

### VDDA 계산 예제

```c
#define VREFINT_CAL_ADDR  0x1FF1E860  // VREFINT 교정값 주소
#define VDDA_NOMINAL      3.3f        // 명목 전압

float Calculate_VDDA(uint16_t vrefint_data)
{
  uint16_t vrefint_cal = *((uint16_t*)VREFINT_CAL_ADDR);

  // VDDA 계산
  float vdda = (VDDA_NOMINAL * vrefint_cal) / vrefint_data;

  return vdda;
}

// 보정된 전압 계산
float Get_Calibrated_Voltage(uint16_t adc_value)
{
  float vdda = Calculate_VDDA(vrefint_value);
  float voltage = (adc_value * vdda) / 4095.0f;
  return voltage;
}
```

## 고급 기능

### 1. 여러 주입 채널 사용

```c
void ADC_MultiInjected_Init(void)
{
  ADC_InjectionConfTypeDef sConfigInjected = {0};

  // 주입 그룹: 4채널
  sConfigInjected.InjectedNbrOfConversion = 4;
  sConfigInjected.ExternalTrigInjecConv = ADC_INJECTED_SOFTWARE_START;
  sConfigInjected.AutoInjectedConv = DISABLE;

  // 채널 1: VREFINT
  sConfigInjected.InjectedChannel = ADC_CHANNEL_VREFINT;
  sConfigInjected.InjectedRank = ADC_INJECTED_RANK_1;
  sConfigInjected.InjectedSamplingTime = ADC_SAMPLETIME_64CYCLES_5;
  HAL_ADCEx_InjectedConfigChannel(&hadc1, &sConfigInjected);

  // 채널 2: TEMPSENSOR
  sConfigInjected.InjectedChannel = ADC_CHANNEL_TEMPSENSOR;
  sConfigInjected.InjectedRank = ADC_INJECTED_RANK_2;
  HAL_ADCEx_InjectedConfigChannel(&hadc1, &sConfigInjected);

  // 채널 3: VBAT
  sConfigInjected.InjectedChannel = ADC_CHANNEL_VBAT;
  sConfigInjected.InjectedRank = ADC_INJECTED_RANK_3;
  HAL_ADCEx_InjectedConfigChannel(&hadc1, &sConfigInjected);

  // 채널 4: 외부 채널
  sConfigInjected.InjectedChannel = ADC_CHANNEL_5;
  sConfigInjected.InjectedRank = ADC_INJECTED_RANK_4;
  HAL_ADCEx_InjectedConfigChannel(&hadc1, &sConfigInjected);

  // 내부 채널 활성화
  HAL_ADCEx_EnableVREFINT();
  HAL_ADCEx_EnableTemperatureSensor();
  HAL_ADCEx_EnableVBAT();
}

// 모든 주입 채널 읽기
void Read_All_Injected(void)
{
  uint16_t vrefint = HAL_ADCEx_InjectedGetValue(&hadc1, ADC_INJECTED_RANK_1);
  uint16_t temp = HAL_ADCEx_InjectedGetValue(&hadc1, ADC_INJECTED_RANK_2);
  uint16_t vbat = HAL_ADCEx_InjectedGetValue(&hadc1, ADC_INJECTED_RANK_3);
  uint16_t ext = HAL_ADCEx_InjectedGetValue(&hadc1, ADC_INJECTED_RANK_4);

  // 온도 계산
  float temperature = Calculate_Temperature(temp);

  // VBAT 계산
  float vbat_voltage = Calculate_VBAT(vbat);
}
```

### 2. 온도 센서 읽기

```c
#define TEMP30_CAL_ADDR  0x1FF1E820  // 30°C 교정값
#define TEMP110_CAL_ADDR 0x1FF1E840  // 110°C 교정값

float Calculate_Temperature(uint16_t temp_data)
{
  uint16_t temp30 = *((uint16_t*)TEMP30_CAL_ADDR);
  uint16_t temp110 = *((uint16_t*)TEMP110_CAL_ADDR);

  // 선형 보간
  float temperature = 30.0f + ((temp_data - temp30) * 80.0f) / (temp110 - temp30);

  return temperature;
}
```

### 3. 자동 주입 모드

```c
void ADC_AutoInjected_Init(void)
{
  ADC_InjectionConfTypeDef sConfigInjected = {0};

  // 자동 주입 활성화
  // 정규 그룹 변환 후 자동으로 주입 그룹 변환
  sConfigInjected.AutoInjectedConv = ENABLE;
  sConfigInjected.InjectedChannel = ADC_CHANNEL_VREFINT;
  sConfigInjected.InjectedRank = ADC_INJECTED_RANK_1;
  sConfigInjected.InjectedNbrOfConversion = 1;
  sConfigInjected.InjectedSamplingTime = ADC_SAMPLETIME_64CYCLES_5;
  sConfigInjected.ExternalTrigInjecConv = ADC_INJECTED_SOFTWARE_START;

  HAL_ADCEx_InjectedConfigChannel(&hadc1, &sConfigInjected);
}
```

### 4. 주입 그룹 외부 트리거

```c
void ADC_InjectedExtTrigger_Init(void)
{
  ADC_InjectionConfTypeDef sConfigInjected = {0};

  // TIM1 TRGO를 주입 그룹 트리거로 사용
  sConfigInjected.ExternalTrigInjecConv = ADC_EXTERNALTRIGINJEC_T1_TRGO;
  sConfigInjected.ExternalTrigInjecConvEdge = ADC_EXTERNALTRIGINJECCONV_EDGE_RISING;
  sConfigInjected.AutoInjectedConv = DISABLE;
  sConfigInjected.InjectedChannel = ADC_CHANNEL_VREFINT;
  sConfigInjected.InjectedRank = ADC_INJECTED_RANK_1;
  sConfigInjected.InjectedNbrOfConversion = 1;
  sConfigInjected.InjectedSamplingTime = ADC_SAMPLETIME_64CYCLES_5;

  HAL_ADCEx_InjectedConfigChannel(&hadc1, &sConfigInjected);
}
```

## 빌드 및 실행

### 빌드

```bash
# STM32CubeIDE
1. File -> Import -> Existing Projects
2. Build Project

# 명령줄
make clean && make -j8
```

### 테스트 시나리오

```c
// 테스트 1: 정규 그룹만 동작
// - TIM2가 1kHz로 ADC 트리거
// - DMA가 regular_buffer에 저장
// - 버퍼가 가득 차면 콜백 호출

// 테스트 2: 주입 그룹 트리거
// - 메인 루프에서 1초마다 Trigger_Injected_Conversion() 호출
// - VREFINT 측정 후 인터럽트 발생
// - 정규 그룹 변환은 일시 중단됨

// 테스트 3: 동시 동작
// - 정규 그룹: 연속 샘플링 (1kHz)
// - 주입 그룹: 온디맨드 측정 (필요시)
```

## 트러블슈팅

### 주입 그룹이 정규 그룹을 중단하지 못하는 경우

```c
// 주입 그룹 우선순위 확인
HAL_NVIC_SetPriority(ADC_IRQn, 0, 0);  // 최고 우선순위

// 정규 그룹이 연속 모드인지 확인
hadc1.Init.ContinuousConvMode = DISABLE;  // 타이머 트리거 사용
```

### VREFINT 값이 불안정한 경우

```c
// 샘플링 시간 증가
sConfigInjected.InjectedSamplingTime = ADC_SAMPLETIME_810CYCLES_5;

// VREFINT 안정화 대기
HAL_ADCEx_EnableVREFINT();
HAL_Delay(10);  // 10ms 대기
```

### DMA 오버런

```c
// 콜백 처리 시간 단축
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)
{
  // 플래그만 설정
  data_ready = 1;
}

// 메인 루프에서 처리
if (data_ready)
{
  data_ready = 0;
  Process_Regular_Data(...);
}
```

## 참고 자료

- **RM0399**: STM32H745 Reference Manual, Chapter 25 (ADC)
- **AN4894**: Analog-to-digital converter on STM32H7 MCUs
- **예제 코드**: `STM32Cube_FW_H7/Projects/STM32H745I-DISCO/Examples/ADC/ADC_RegularConversion_Polling`

## 관련 예제

- **ADC_DualModeInterleaved**: 듀얼 ADC 인터리브 모드
- **TIM_DMA**: 타이머 트리거 설정
- **DMA_DMAMUX**: DMA 설정
