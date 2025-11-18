# ADC_DualModeInterleaved - 듀얼 ADC 인터리브 모드

## 개요

이 예제는 STM32H745의 듀얼 ADC를 인터리브(Interleaved) 모드로 사용하여 최대 4 MSPS(Mega Samples Per Second)의 고속 샘플링을 달성하는 방법을 보여줍니다. 두 개의 ADC(ADC1과 ADC2)를 번갈아 사용하여 단일 ADC 대비 2배의 샘플링 속도를 얻습니다.

## 하드웨어 요구사항

- **STM32H745I-DISCO 보드**
- **입력 신호**: PC0 (ADC12_INP10) - 아날로그 입력
- **신호 범위**: 0V ~ 3.3V
- **신호 소스**: 함수 발생기 또는 가변 전압
- **오실로스코프**: 샘플링 성능 검증용 (선택)

## 주요 기능

### 듀얼 ADC 인터리브 모드
- **샘플링 속도**: 최대 4 MSPS (각 ADC 2 MSPS)
- **해상도**: 12비트 (0-4095)
- **DMA 전송**: 순환 모드로 연속 샘플링
- **데이터 정렬**: 32비트 (ADC1: 하위 16비트, ADC2: 상위 16비트)

### 인터리브 동작 원리
```
Time →
ADC1: ━━┳━━━━┳━━━━┳━━━━┳━━━━┳━━━━
ADC2: ━━━━┳━━━━┳━━━━┳━━━━┳━━━━┳━━

Result: ┳┳┳┳┳┳┳┳┳┳┳┳┳┳┳┳┳┳┳┳┳
        └─ 2배 샘플링 속도
```

### DMA 설정
- **모드**: 순환 (Circular)
- **버퍼 크기**: 2048 샘플
- **Half/Full 콜백**: 더블 버퍼링 처리
- **데이터 폭**: 32비트 (Dual mode)

## 동작 원리

### 시스템 구성
```
         ┌─────────────┐
Input ──►│   PC0       │
Signal   │  (ADC_IN10) │
         └──────┬──────┘
                │
        ┌───────┴───────┐
        │               │
   ┌────▼────┐     ┌────▼────┐
   │  ADC1   │     │  ADC2   │
   │ Master  │◄────┤  Slave  │
   └────┬────┘     └────┬────┘
        │               │
        └───────┬───────┘
                │ (32-bit packed)
                ▼
         ┌─────────────┐
         │     DMA1    │
         │  Stream 0   │
         └──────┬──────┘
                │
                ▼
         ┌─────────────┐
         │   Memory    │
         │   Buffer    │
         │ (2048 x 32) │
         └─────────────┘
```

### 인터리브 타이밍
```
                Conversion Time
                ◄──────────►
Trigger: ─┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐
          └─┘   └─┘   └─┘   └─┘   └─┘

ADC1:     ──┐     ┌───┐     ┌───┐     ┌───
            └─────┘   └─────┘   └─────┘

ADC2:     ────┐     ┌───┐     ┌───┐     ┌─
              └─────┘   └─────┘   └─────┘

Result:   ─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
           └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─
```

## 코드 구조

### 1. ADC 듀얼 모드 초기화

```c
// 전역 변수
ADC_HandleTypeDef hadc1;
ADC_HandleTypeDef hadc2;
DMA_HandleTypeDef hdma_adc1;

// 듀얼 모드 버퍼 (32비트 단위)
#define ADC_BUFFER_SIZE  2048
uint32_t adc_dual_buffer[ADC_BUFFER_SIZE];

void ADC_DualMode_Init(void)
{
  ADC_MultiModeTypeDef multimode = {0};
  ADC_ChannelConfTypeDef sConfig = {0};

  // ADC 클럭 활성화
  __HAL_RCC_ADC12_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // ADC1 (Master) 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  hadc1.Instance = ADC1;

  // 클럭 설정 (Synchronous clock / 4)
  // ADC Clock = 400MHz / 4 = 100MHz
  hadc1.Init.ClockPrescaler = ADC_CLOCK_SYNC_PCLK_DIV4;

  // 해상도: 12비트
  hadc1.Init.Resolution = ADC_RESOLUTION_12B;

  // 스캔 모드: 단일 채널
  hadc1.Init.ScanConvMode = ADC_SCAN_DISABLE;

  // 연속 변환 모드
  hadc1.Init.ContinuousConvMode = ENABLE;

  // DMA 연속 요청
  hadc1.Init.DMAContinuousRequests = ENABLE;

  // 변환 끝 플래그 선택
  hadc1.Init.EOCSelection = ADC_EOC_SINGLE_CONV;

  // 외부 트리거: 소프트웨어 시작
  hadc1.Init.ExternalTrigConv = ADC_SOFTWARE_START;
  hadc1.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_NONE;

  // 데이터 정렬: 오른쪽 정렬
  hadc1.Init.DataAlign = ADC_DATAALIGN_RIGHT;

  // 변환 횟수: 1 (단일 채널)
  hadc1.Init.NbrOfConversion = 1;

  // 불연속 모드: 비활성화
  hadc1.Init.DiscontinuousConvMode = DISABLE;

  // 오버런 모드: 덮어쓰기
  hadc1.Init.Overrun = ADC_OVR_DATA_OVERWRITTEN;

  // Low Power 모드: 비활성화
  hadc1.Init.LowPowerAutoWait = DISABLE;

  // 오버샘플링: 비활성화 (최대 속도를 위해)
  hadc1.Init.OversamplingMode = DISABLE;

  // ADC1 초기화
  if (HAL_ADC_Init(&hadc1) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 듀얼 모드 설정 (Interleaved)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  multimode.Mode = ADC_DUALMODE_INTERL;  // 인터리브 모드

  // ADC 간 지연: 4 ADC 클럭 사이클
  // Delay = 4 x 10ns = 40ns (100MHz ADC clock)
  multimode.DualModeData = ADC_DUALMODEDATAFORMAT_32_10_BITS;
  multimode.TwoSamplingDelay = ADC_TWOSAMPLINGDELAY_4CYCLES;

  if (HAL_ADCEx_MultiModeConfigChannel(&hadc1, &multimode) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // ADC1 채널 설정 (PC0 = ADC12_INP10)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  sConfig.Channel = ADC_CHANNEL_10;
  sConfig.Rank = ADC_REGULAR_RANK_1;

  // 샘플링 시간: 1.5 사이클 (최고속)
  // Total conversion time = 1.5 + 7.5 = 9 cycles = 90ns @ 100MHz
  sConfig.SamplingTime = ADC_SAMPLETIME_1CYCLE_5;

  sConfig.SingleDiff = ADC_SINGLE_ENDED;
  sConfig.OffsetNumber = ADC_OFFSET_NONE;
  sConfig.Offset = 0;

  if (HAL_ADC_ConfigChannel(&hadc1, &sConfig) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // ADC2 (Slave) 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  hadc2.Instance = ADC2;

  // ADC1과 동일한 설정
  hadc2.Init = hadc1.Init;

  if (HAL_ADC_Init(&hadc2) != HAL_OK)
  {
    Error_Handler();
  }

  // ADC2 채널 설정 (동일 채널)
  if (HAL_ADC_ConfigChannel(&hadc2, &sConfig) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // ADC 캘리브레이션
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  HAL_ADCEx_Calibration_Start(&hadc1, ADC_CALIB_OFFSET, ADC_SINGLE_ENDED);
  HAL_ADCEx_Calibration_Start(&hadc2, ADC_CALIB_OFFSET, ADC_SINGLE_ENDED);
}
```

### 2. DMA 설정

```c
void DMA_Init(void)
{
  // DMA 클럭 활성화
  __HAL_RCC_DMA1_CLK_ENABLE();

  // DMA 핸들 설정
  hdma_adc1.Instance = DMA1_Stream0;

  // DMA 설정
  hdma_adc1.Init.Request = DMA_REQUEST_ADC1;        // ADC1 요청
  hdma_adc1.Init.Direction = DMA_PERIPH_TO_MEMORY;  // 주변장치->메모리
  hdma_adc1.Init.PeriphInc = DMA_PINC_DISABLE;      // 주변장치 주소 고정
  hdma_adc1.Init.MemInc = DMA_MINC_ENABLE;          // 메모리 주소 증가
  hdma_adc1.Init.PeriphDataAlignment = DMA_PDATAALIGN_WORD;  // 32비트
  hdma_adc1.Init.MemDataAlignment = DMA_MDATAALIGN_WORD;     // 32비트
  hdma_adc1.Init.Mode = DMA_CIRCULAR;               // 순환 모드
  hdma_adc1.Init.Priority = DMA_PRIORITY_HIGH;
  hdma_adc1.Init.FIFOMode = DMA_FIFOMODE_DISABLE;

  if (HAL_DMA_Init(&hdma_adc1) != HAL_OK)
  {
    Error_Handler();
  }

  // ADC와 DMA 연결
  __HAL_LINKDMA(&hadc1, DMA_Handle, hdma_adc1);

  // DMA 인터럽트 활성화
  HAL_NVIC_SetPriority(DMA1_Stream0_IRQn, 0, 0);
  HAL_NVIC_EnableIRQ(DMA1_Stream0_IRQn);
}
```

### 3. GPIO 설정 (아날로그 입력)

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

### 4. ADC 시작 및 DMA 사용

```c
void ADC_Start_DualMode_DMA(void)
{
  // 듀얼 모드 DMA 시작
  if (HAL_ADCEx_MultiModeStart_DMA(&hadc1,
                                    adc_dual_buffer,
                                    ADC_BUFFER_SIZE) != HAL_OK)
  {
    Error_Handler();
  }
}
```

### 5. DMA 콜백 함수

```c
// 버퍼 절반 완료 콜백
void HAL_ADC_ConvHalfCpltCallback(ADC_HandleTypeDef* hadc)
{
  // 버퍼의 첫 번째 절반 처리
  Process_ADC_Data(adc_dual_buffer, ADC_BUFFER_SIZE / 2);
}

// 버퍼 완료 콜백
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)
{
  // 버퍼의 두 번째 절반 처리
  Process_ADC_Data(&adc_dual_buffer[ADC_BUFFER_SIZE / 2],
                   ADC_BUFFER_SIZE / 2);
}

// 데이터 처리 함수
void Process_ADC_Data(uint32_t *buffer, uint32_t size)
{
  for (uint32_t i = 0; i < size; i++)
  {
    // 32비트 데이터에서 개별 ADC 값 추출
    uint16_t adc1_value = buffer[i] & 0xFFFF;         // 하위 16비트
    uint16_t adc2_value = (buffer[i] >> 16) & 0xFFFF; // 상위 16비트

    // 전압 변환 (12비트, 3.3V 기준)
    float voltage1 = (adc1_value * 3.3f) / 4095.0f;
    float voltage2 = (adc2_value * 3.3f) / 4095.0f;

    // 데이터 처리 (평균, 필터링 등)
    // ...
  }
}
```

### 6. 메인 함수

```c
int main(void)
{
  // HAL 초기화
  HAL_Init();

  // 시스템 클럭 설정 (400MHz)
  SystemClock_Config();

  // 주변장치 초기화
  GPIO_Init();
  DMA_Init();
  ADC_DualMode_Init();

  // ADC 듀얼 모드 시작
  ADC_Start_DualMode_DMA();

  // 무한 루프
  while (1)
  {
    // 메인 루프는 비어 있음
    // 모든 처리는 DMA 콜백에서 수행
    HAL_Delay(100);
  }
}
```

## 샘플링 속도 계산

### ADC 클럭 설정
```c
// 시스템 클럭: 400MHz
// ADC 클럭 프리스케일러: /4
// ADC 클럭 = 400MHz / 4 = 100MHz
```

### 변환 시간
```
샘플링 시간 = 1.5 ADC 클럭 사이클
변환 시간 = 7.5 ADC 클럭 사이클 (12비트)
총 변환 시간 = 1.5 + 7.5 = 9 사이클

단일 ADC 샘플링 속도:
= 100MHz / 9 = 11.11 MSPS

인터리브 모드 (듀얼 ADC):
최대 샘플링 속도 = 11.11 MSPS × 2 = 22.22 MSPS (이론적)

실제 달성 가능:
= 4 MSPS (DMA 및 버스 대역폭 고려)
```

## 데이터 포맷

### 듀얼 모드 32비트 데이터
```
┌─────────────────┬─────────────────┐
│  ADC2 (16비트)  │  ADC1 (16비트)  │
└─────────────────┴─────────────────┘
  31           16  15             0

예시:
  0x0ABC0DEF

  ADC1 값 = 0x0DEF = 3567
  ADC2 값 = 0x0ABC = 2748
```

### 데이터 추출 예제
```c
// 방법 1: 비트 마스킹
uint32_t data = adc_dual_buffer[i];
uint16_t adc1 = data & 0xFFFF;
uint16_t adc2 = (data >> 16) & 0xFFFF;

// 방법 2: 포인터 캐스팅
uint16_t *ptr = (uint16_t*)&adc_dual_buffer[i];
uint16_t adc1 = ptr[0];
uint16_t adc2 = ptr[1];

// 방법 3: 공용체 사용
typedef union {
  uint32_t dual;
  struct {
    uint16_t adc1;
    uint16_t adc2;
  };
} ADC_Dual_t;

ADC_Dual_t *data = (ADC_Dual_t*)adc_dual_buffer;
uint16_t adc1 = data[i].adc1;
uint16_t adc2 = data[i].adc2;
```

## 메모리 요구사항

### 버퍼 크기 계산
```c
// 샘플 수: 2048
// 데이터 폭: 32비트 (4바이트)
// 총 크기 = 2048 × 4 = 8192 바이트 = 8KB

// 메모리 배치 (DTCM RAM 사용 권장)
uint32_t adc_dual_buffer[2048] __attribute__((section(".dtcm")));

// 또는 D1 SRAM (AXI SRAM)
uint32_t adc_dual_buffer[2048] __attribute__((section(".sram1")));
```

### 샘플링 시간
```
샘플링 속도: 4 MSPS
버퍼 크기: 2048 샘플
버퍼 채우기 시간 = 2048 / 4,000,000 = 0.512 ms

Half Complete 콜백: 0.256 ms마다
Full Complete 콜백: 0.512 ms마다
```

## 고급 기능

### 1. 실시간 FFT 분석

```c
#include "arm_math.h"

#define FFT_SIZE  1024

// FFT 인스턴스
arm_rfft_fast_instance_f32 fft_instance;
float32_t fft_input[FFT_SIZE];
float32_t fft_output[FFT_SIZE];

void FFT_Init(void)
{
  arm_rfft_fast_init_f32(&fft_instance, FFT_SIZE);
}

void Process_FFT(uint32_t *adc_data, uint32_t size)
{
  // ADC 데이터를 float로 변환
  for (uint32_t i = 0; i < FFT_SIZE; i++)
  {
    uint16_t adc_value = adc_data[i] & 0xFFFF;  // ADC1만 사용
    fft_input[i] = (float32_t)adc_value;
  }

  // FFT 수행
  arm_rfft_fast_f32(&fft_instance, fft_input, fft_output, 0);

  // 크기 계산
  float32_t magnitudes[FFT_SIZE / 2];
  arm_cmplx_mag_f32(fft_output, magnitudes, FFT_SIZE / 2);

  // 주파수 성분 분석
  // ...
}
```

### 2. 오버샘플링으로 해상도 향상

```c
#define OVERSAMPLE_RATE  4  // 4배 오버샘플링

void ADC_Oversample_Init(void)
{
  // 오버샘플링 설정
  hadc1.Init.OversamplingMode = ENABLE;
  hadc1.Init.Oversampling.Ratio = 4;                    // 4배
  hadc1.Init.Oversampling.RightBitShift = 1;            // 1비트 시프트
  hadc1.Init.Oversampling.TriggeredMode = ADC_TRIGGEREDMODE_SINGLE_TRIGGER;
  hadc1.Init.Oversampling.OversamplingStopReset = ADC_REGOVERSAMPLING_CONTINUED_MODE;

  // 12비트 + 1비트 = 13비트 유효 해상도
}
```

### 3. 트리거 기반 샘플링

```c
void ADC_Timer_Trigger_Init(void)
{
  TIM_HandleTypeDef htim2;

  // TIM2 클럭 활성화
  __HAL_RCC_TIM2_CLK_ENABLE();

  // 타이머 설정 (예: 1 MHz 샘플링)
  htim2.Instance = TIM2;
  htim2.Init.Prescaler = 199;      // 200MHz / 200 = 1MHz
  htim2.Init.CounterMode = TIM_COUNTERMODE_UP;
  htim2.Init.Period = 0;           // 최대 속도
  htim2.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
  HAL_TIM_Base_Init(&htim2);

  // TRGO 이벤트 설정
  TIM_MasterConfigTypeDef sMasterConfig = {0};
  sMasterConfig.MasterOutputTrigger = TIM_TRGO_UPDATE;
  sMasterConfig.MasterSlaveMode = TIM_MASTERSLAVEMODE_DISABLE;
  HAL_TIMEx_MasterConfigSynchronization(&htim2, &sMasterConfig);

  // ADC 외부 트리거 설정
  hadc1.Init.ExternalTrigConv = ADC_EXTERNALTRIG_T2_TRGO;
  hadc1.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_RISING;

  // 타이머 시작
  HAL_TIM_Base_Start(&htim2);
}
```

### 4. 평균 및 필터링

```c
#define FILTER_SIZE  8

typedef struct {
  uint32_t buffer[FILTER_SIZE];
  uint8_t index;
  uint32_t sum;
} MovingAverage_t;

MovingAverage_t filter;

void MovingAverage_Init(MovingAverage_t *f)
{
  memset(f, 0, sizeof(MovingAverage_t));
}

uint16_t MovingAverage_Update(MovingAverage_t *f, uint16_t new_value)
{
  // 이전 값 제거
  f->sum -= f->buffer[f->index];

  // 새 값 추가
  f->buffer[f->index] = new_value;
  f->sum += new_value;

  // 인덱스 업데이트
  f->index = (f->index + 1) % FILTER_SIZE;

  // 평균 반환
  return (uint16_t)(f->sum / FILTER_SIZE);
}
```

## 빌드 및 실행

### 빌드 방법

```bash
# STM32CubeIDE
1. File -> Import -> Existing Projects into Workspace
2. Browse to: Examples/ADC/ADC_DualModeInterleaved
3. Build Project (Ctrl+B)

# 명령줄
cd ADC_DualModeInterleaved
make clean
make -j8
```

### 플래싱

```bash
# OpenOCD
openocd -f board/stm32h745i-disco.cfg \
        -c "program build/ADC_DualModeInterleaved.elf verify reset exit"

# ST-LINK Utility
1. Connect to target
2. Program & Verify (0x08000000)
```

### 테스트 설정

```
┌──────────────┐
│  Function    │
│  Generator   │  Sine wave: 1kHz, 1Vpp, 1.65V offset
└───────┬──────┘
        │
        │  Probe
        ▼
     ┌──────┐
     │  PC0 │  STM32H745I-DISCO
     └──────┘
```

## 성능 측정

### 샘플링 속도 확인

```c
// GPIO 토글로 샘플링 속도 측정
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)
{
  static uint32_t count = 0;

  // 매 1000번째 샘플마다 GPIO 토글
  if (++count >= 1000)
  {
    HAL_GPIO_TogglePin(GPIOI, GPIO_PIN_12);  // LED1
    count = 0;
  }

  Process_ADC_Data(&adc_dual_buffer[ADC_BUFFER_SIZE / 2],
                   ADC_BUFFER_SIZE / 2);
}

// 오실로스코프로 LED1 주파수 측정
// 샘플링 속도 = 측정된 주파수 × 2000
```

### CPU 부하 측정

```c
// DWT 사이클 카운터 사용
void Process_ADC_Data(uint32_t *buffer, uint32_t size)
{
  uint32_t start = DWT->CYCCNT;

  // 데이터 처리
  for (uint32_t i = 0; i < size; i++)
  {
    // ...
  }

  uint32_t cycles = DWT->CYCCNT - start;
  float time_us = (float)cycles / 400.0f;  // 400MHz

  printf("Processing time: %.2f us\n", time_us);
}
```

## 트러블슈팅

### ADC 값이 불안정한 경우

#### 1. 입력 임피던스 확인
```c
// 샘플링 시간 증가
sConfig.SamplingTime = ADC_SAMPLETIME_8CYCLES_5;  // 1.5 -> 8.5

// 또는 외부 버퍼 앰프 사용
```

#### 2. 노이즈 필터링
```c
// 아날로그 입력에 RC 필터 추가
// R = 1kΩ, C = 100nF
// fc = 1 / (2π × R × C) ≈ 1.6kHz
```

#### 3. VDDA/VREF+ 확인
```c
// VDDA 전압 측정 (3.3V ± 5% 이내)
// 디커플링 커패시터 추가 (100nF + 10μF)
```

### DMA 오버런 발생

```c
// DMA 콜백 처리 시간 단축
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)
{
  // 무거운 처리는 메인 루프로 이동
  data_ready_flag = 1;
}

void main_loop(void)
{
  if (data_ready_flag)
  {
    data_ready_flag = 0;
    Process_ADC_Data(...);  // 여기서 처리
  }
}
```

### 듀얼 모드 동기화 문제

```c
// 두 ADC의 클럭이 동기화되었는지 확인
if ((ADC1->CR & ADC_CR_ADSTART) && (ADC2->CR & ADC_CR_ADSTART))
{
  printf("Both ADCs running\n");
}

// 샘플링 지연 조정
multimode.TwoSamplingDelay = ADC_TWOSAMPLINGDELAY_8CYCLES;  // 4 -> 8
```

## 참고 자료

- **RM0399**: STM32H745 Reference Manual, Chapter 25 (ADC)
- **AN4894**: Analog-to-digital converter on STM32H7 MCUs
- **AN2834**: How to get the best ADC accuracy in STM32 MCUs
- **예제 코드**: `STM32Cube_FW_H7_V1.xx.x/Projects/STM32H745I-DISCO/Examples/ADC/ADC_DualModeInterleaved`

## 관련 예제

- **ADC_Regular_injected_groups**: 정규 및 주입 그룹 변환
- **ADC_TripleModeInterleaved**: 3개 ADC 인터리브 (STM32H743)
- **TIM_DMA**: 타이머 트리거 ADC
- **DMA_DMAMUX**: DMA 및 DMAMUX 설정
