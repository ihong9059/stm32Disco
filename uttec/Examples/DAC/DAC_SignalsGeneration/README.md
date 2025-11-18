# DAC_SignalsGeneration - DAC 신호 생성

## 개요

이 예제는 STM32H745의 DAC(Digital-to-Analog Converter)를 사용하여 다양한 아날로그 파형을 생성하는 방법을 보여줍니다. DMA를 활용하여 CPU 개입 없이 삼각파(Triangle)와 계단파(Escalator) 파형을 연속적으로 출력합니다.

## 하드웨어 요구사항

- **STM32H745I-DISCO 보드**
- **DAC 출력**: PA4 (DAC1_OUT1)
- **출력 전압 범위**: 0V ~ 3.3V (VDDA 기준)
- **오실로스코프**: 파형 확인용
- **선택 사항**: 외부 로우패스 필터 (고주파 노이즈 제거)

## 주요 기능

### DAC 특성
- **해상도**: 12비트 (0-4095, 0V-3.3V)
- **변환 속도**: 최대 1 MSPS
- **출력 버퍼**: 내장 버퍼로 출력 임피던스 감소
- **DMA 지원**: 연속 파형 생성에 최적

### 생성 가능한 파형
- **삼각파 (Triangle)**: 선형 상승/하강
- **계단파 (Escalator)**: 계단식 증가
- **사인파**: 부드러운 주기 파형
- **톱니파**: 선형 램프
- **임의 파형**: 사용자 정의 데이터

## 동작 원리

### 시스템 구성
```
┌──────────┐
│  Memory  │  [Triangle Waveform Data]
│  Buffer  │  [0, 100, 200, ..., 4095, ..., 0]
└────┬─────┘
     │
     │ DMA Transfer
     ▼
┌──────────┐         ┌──────────┐         ┌──────────┐
│   DMA    │────────►│   DAC1   │────────►│   PA4    │
│ Stream 5 │         │  Channel │         │ (Analog) │
└──────────┘         │    1     │         └──────────┘
     ▲               └──────────┘
     │                    ▲
     │                    │
┌────┴─────┐         ┌────┴─────┐
│  TIM6    │────────►│ Trigger  │
│ (Timer)  │  TRGO   │          │
└──────────┘         └──────────┘
```

### 파형 생성 흐름
```
1. 메모리 버퍼에 파형 데이터 준비
2. TIM6가 주기적으로 트리거 생성
3. 트리거마다 DMA가 DAC 레지스터에 데이터 전송
4. DAC가 디지털 값을 아날로그 전압으로 변환
5. PA4 핀에서 파형 출력

Time →
TIM6:   ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
Trigger └─┘ └─┘ └─┘ └─┘ └─┘ └─┘

DMA:    ─┬──┬──┬──┬──┬──┬──┬──
Data    │0 │1 │2 │3 │4 │5 │6 │
        └──┴──┴──┴──┴──┴──┴──

DAC:    ▁▂▃▄▅▆▇█ (삼각파)
Output
```

## 코드 구조

### 1. DAC 초기화

```c
// 전역 변수
DAC_HandleTypeDef hdac1;
TIM_HandleTypeDef htim6;
DMA_HandleTypeDef hdma_dac1_ch1;

// 파형 데이터 버퍼
#define WAVEFORM_SAMPLES  256

// 삼각파 버퍼
uint16_t triangle_wave[WAVEFORM_SAMPLES];

// 계단파 버퍼
uint16_t escalator_wave[WAVEFORM_SAMPLES];

void DAC1_Init(void)
{
  DAC_ChannelConfTypeDef sConfig = {0};

  // DAC 클럭 활성화
  __HAL_RCC_DAC12_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // DAC 기본 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  hdac1.Instance = DAC1;

  if (HAL_DAC_Init(&hdac1) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // DAC 채널 1 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  sConfig.DAC_HighFrequency = DAC_HIGH_FREQUENCY_INTERFACE_MODE_AUTOMATIC;
  sConfig.DAC_DMADoubleDataMode = DISABLE;
  sConfig.DAC_SignedFormat = DISABLE;

  // 외부 트리거: TIM6 TRGO
  sConfig.DAC_Trigger = DAC_TRIGGER_T6_TRGO;
  sConfig.DAC_Trigger2 = DAC_TRIGGER_NONE;

  // 출력 버퍼: 활성화 (임피던스 감소)
  sConfig.DAC_OutputBuffer = DAC_OUTPUTBUFFER_ENABLE;

  // 연결 모드: 외부 핀으로 출력
  sConfig.DAC_ConnectOnChipPeripheral = DAC_CHIPCONNECT_EXTERNAL;

  // 샘플 앤 홀드: 비활성화
  sConfig.DAC_SampleAndHold = DAC_SAMPLEANDHOLD_DISABLE;

  if (HAL_DAC_ConfigChannel(&hdac1, &sConfig, DAC_CHANNEL_1) != HAL_OK)
  {
    Error_Handler();
  }
}
```

### 2. 타이머 트리거 설정

```c
void TIM6_Init(void)
{
  TIM_MasterConfigTypeDef sMasterConfig = {0};

  // TIM6 클럭 활성화
  __HAL_RCC_TIM6_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 타이머 기본 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // 목표: 1kHz 파형 생성 (256 샘플)
  // DAC 업데이트 주파수 = 1kHz × 256 = 256kHz
  // APB1 Timer Clock = 200MHz
  // Prescaler = 0 (200MHz)
  // ARR = 200MHz / 256kHz - 1 = 780

  htim6.Instance = TIM6;
  htim6.Init.Prescaler = 0;
  htim6.Init.CounterMode = TIM_COUNTERMODE_UP;
  htim6.Init.Period = 780;  // 256kHz 업데이트
  htim6.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_DISABLE;

  if (HAL_TIM_Base_Init(&htim6) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // Master 모드 설정 (TRGO 출력)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  sMasterConfig.MasterOutputTrigger = TIM_TRGO_UPDATE;
  sMasterConfig.MasterSlaveMode = TIM_MASTERSLAVEMODE_DISABLE;

  if (HAL_TIMEx_MasterConfigSynchronization(&htim6, &sMasterConfig) != HAL_OK)
  {
    Error_Handler();
  }
}
```

### 3. DMA 설정

```c
void DMA_Init(void)
{
  // DMA 클럭 활성화
  __HAL_RCC_DMA1_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // DMA 핸들 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  hdma_dac1_ch1.Instance = DMA1_Stream5;
  hdma_dac1_ch1.Init.Request = DMA_REQUEST_DAC1_CH1;
  hdma_dac1_ch1.Init.Direction = DMA_MEMORY_TO_PERIPH;  // 메모리 -> DAC
  hdma_dac1_ch1.Init.PeriphInc = DMA_PINC_DISABLE;      // DAC 주소 고정
  hdma_dac1_ch1.Init.MemInc = DMA_MINC_ENABLE;          // 메모리 주소 증가
  hdma_dac1_ch1.Init.PeriphDataAlignment = DMA_PDATAALIGN_HALFWORD;  // 16비트
  hdma_dac1_ch1.Init.MemDataAlignment = DMA_MDATAALIGN_HALFWORD;     // 16비트
  hdma_dac1_ch1.Init.Mode = DMA_CIRCULAR;               // 순환 모드
  hdma_dac1_ch1.Init.Priority = DMA_PRIORITY_HIGH;
  hdma_dac1_ch1.Init.FIFOMode = DMA_FIFOMODE_DISABLE;

  if (HAL_DMA_Init(&hdma_dac1_ch1) != HAL_OK)
  {
    Error_Handler();
  }

  // DAC와 DMA 연결
  __HAL_LINKDMA(&hdac1, DMA_Handle1, hdma_dac1_ch1);
}
```

### 4. GPIO 설정

```c
void GPIO_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  // GPIOA 클럭 활성화
  __HAL_RCC_GPIOA_CLK_ENABLE();

  // PA4를 아날로그 모드로 설정 (DAC1_OUT1)
  GPIO_InitStruct.Pin = GPIO_PIN_4;
  GPIO_InitStruct.Mode = GPIO_MODE_ANALOG;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
}
```

### 5. 파형 데이터 생성

```c
void Generate_Waveforms(void)
{
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 삼각파 생성
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  uint16_t half_samples = WAVEFORM_SAMPLES / 2;

  // 상승 엣지 (0 -> 4095)
  for (uint16_t i = 0; i < half_samples; i++)
  {
    triangle_wave[i] = (4095 * i) / (half_samples - 1);
  }

  // 하강 엣지 (4095 -> 0)
  for (uint16_t i = 0; i < half_samples; i++)
  {
    triangle_wave[half_samples + i] = 4095 - ((4095 * i) / (half_samples - 1));
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 계단파 생성 (8단계)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  uint16_t steps = 8;
  uint16_t samples_per_step = WAVEFORM_SAMPLES / steps;

  for (uint16_t step = 0; step < steps; step++)
  {
    uint16_t level = (4095 * step) / (steps - 1);

    for (uint16_t i = 0; i < samples_per_step; i++)
    {
      escalator_wave[step * samples_per_step + i] = level;
    }
  }
}
```

### 6. DAC 시작

```c
void DAC_Start_Waveform(uint16_t *waveform_data)
{
  // DAC 채널 1 시작 (DMA 사용)
  if (HAL_DAC_Start_DMA(&hdac1,
                        DAC_CHANNEL_1,
                        (uint32_t*)waveform_data,
                        WAVEFORM_SAMPLES,
                        DAC_ALIGN_12B_R) != HAL_OK)  // 12비트 오른쪽 정렬
  {
    Error_Handler();
  }

  // 타이머 시작 (DAC 트리거)
  if (HAL_TIM_Base_Start(&htim6) != HAL_OK)
  {
    Error_Handler();
  }
}
```

### 7. 메인 함수

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
  TIM6_Init();
  DAC1_Init();

  // 파형 데이터 생성
  Generate_Waveforms();

  // 삼각파 출력 시작
  DAC_Start_Waveform(triangle_wave);

  printf("Triangle wave generation started on PA4\n");

  uint32_t switch_time = HAL_GetTick();

  // 무한 루프
  while (1)
  {
    // 5초마다 파형 전환
    if (HAL_GetTick() - switch_time >= 5000)
    {
      // DAC 정지
      HAL_DAC_Stop_DMA(&hdac1, DAC_CHANNEL_1);

      // 파형 전환
      static uint8_t waveform_index = 0;
      waveform_index = !waveform_index;

      if (waveform_index == 0)
      {
        DAC_Start_Waveform(triangle_wave);
        printf("Switched to Triangle wave\n");
      }
      else
      {
        DAC_Start_Waveform(escalator_wave);
        printf("Switched to Escalator wave\n");
      }

      switch_time = HAL_GetTick();
    }

    HAL_Delay(100);
  }
}
```

## 다양한 파형 생성

### 1. 사인파 (Sine Wave)

```c
#include <math.h>

void Generate_Sine_Wave(uint16_t *buffer, uint16_t samples)
{
  for (uint16_t i = 0; i < samples; i++)
  {
    float angle = (2.0f * M_PI * i) / samples;
    float sine = sinf(angle);

    // -1~1 -> 0~4095
    buffer[i] = (uint16_t)((sine + 1.0f) * 2047.5f);
  }
}
```

### 2. 톱니파 (Sawtooth Wave)

```c
void Generate_Sawtooth_Wave(uint16_t *buffer, uint16_t samples)
{
  for (uint16_t i = 0; i < samples; i++)
  {
    buffer[i] = (4095 * i) / samples;
  }
}
```

### 3. 구형파 (Square Wave)

```c
void Generate_Square_Wave(uint16_t *buffer, uint16_t samples)
{
  uint16_t half = samples / 2;

  for (uint16_t i = 0; i < samples; i++)
  {
    buffer[i] = (i < half) ? 4095 : 0;
  }
}
```

### 4. 임의 파형 (Arbitrary Waveform)

```c
// 사용자 정의 파형 테이블
const uint16_t custom_waveform[] = {
  0, 512, 1024, 2048, 4095, 2048, 1024, 512,
  0, 256, 512, 1024, 512, 256, 0, 0
};

void Generate_Custom_Wave(uint16_t *buffer, uint16_t samples)
{
  uint16_t table_size = sizeof(custom_waveform) / sizeof(custom_waveform[0]);

  for (uint16_t i = 0; i < samples; i++)
  {
    // 테이블 인터폴레이션
    uint16_t index = (i * table_size) / samples;
    buffer[i] = custom_waveform[index];
  }
}
```

## 주파수 및 진폭 제어

### 주파수 변경

```c
// DAC 업데이트 주파수를 변경하여 파형 주파수 조절
void Set_Waveform_Frequency(float frequency_hz)
{
  // 파형 주파수 = DAC 업데이트 주파수 / 샘플 수
  // DAC 업데이트 주파수 = 파형 주파수 × 샘플 수

  uint32_t update_freq = (uint32_t)(frequency_hz * WAVEFORM_SAMPLES);

  // TIM6 ARR 계산
  // ARR = TIM_CLK / update_freq - 1
  uint32_t arr = (200000000 / update_freq) - 1;

  // 타이머 정지
  HAL_TIM_Base_Stop(&htim6);

  // ARR 업데이트
  __HAL_TIM_SET_AUTORELOAD(&htim6, arr);

  // 타이머 재시작
  HAL_TIM_Base_Start(&htim6);

  printf("Waveform frequency set to %.2f Hz\n", frequency_hz);
}
```

### 진폭 및 오프셋 제어

```c
void Set_Waveform_Amplitude_Offset(uint16_t *buffer,
                                     uint16_t samples,
                                     float amplitude,   // 0.0 ~ 1.0
                                     float offset)      // 0.0 ~ 1.0
{
  for (uint16_t i = 0; i < samples; i++)
  {
    // 정규화 (0.0 ~ 1.0)
    float normalized = (float)buffer[i] / 4095.0f;

    // 진폭 및 오프셋 적용
    normalized = (normalized * amplitude) + offset;

    // 클리핑 (0.0 ~ 1.0)
    if (normalized > 1.0f) normalized = 1.0f;
    if (normalized < 0.0f) normalized = 0.0f;

    // 다시 12비트로 변환
    buffer[i] = (uint16_t)(normalized * 4095.0f);
  }
}

// 사용 예제
Generate_Sine_Wave(sine_buffer, WAVEFORM_SAMPLES);
Set_Waveform_Amplitude_Offset(sine_buffer, WAVEFORM_SAMPLES,
                               0.5f,   // 50% 진폭
                               0.25f); // 25% 오프셋
// 결과: 0.825V ~ 2.475V (VDDA = 3.3V 기준)
```

## 고급 기능

### 1. 듀얼 DAC 출력

```c
DAC_HandleTypeDef hdac1;
uint16_t waveform_ch1[WAVEFORM_SAMPLES];
uint16_t waveform_ch2[WAVEFORM_SAMPLES];

void Dual_DAC_Init(void)
{
  // 채널 1: PA4
  // 채널 2: PA5

  DAC_ChannelConfTypeDef sConfig = {0};

  // 채널 1 설정
  sConfig.DAC_Trigger = DAC_TRIGGER_T6_TRGO;
  sConfig.DAC_OutputBuffer = DAC_OUTPUTBUFFER_ENABLE;
  HAL_DAC_ConfigChannel(&hdac1, &sConfig, DAC_CHANNEL_1);

  // 채널 2 설정
  HAL_DAC_ConfigChannel(&hdac1, &sConfig, DAC_CHANNEL_2);

  // 파형 생성
  Generate_Sine_Wave(waveform_ch1, WAVEFORM_SAMPLES);      // 사인파
  Generate_Triangle_Wave(waveform_ch2, WAVEFORM_SAMPLES);  // 삼각파

  // 양쪽 채널 시작
  HAL_DAC_Start_DMA(&hdac1, DAC_CHANNEL_1,
                    (uint32_t*)waveform_ch1, WAVEFORM_SAMPLES,
                    DAC_ALIGN_12B_R);

  HAL_DAC_Start_DMA(&hdac1, DAC_CHANNEL_2,
                    (uint32_t*)waveform_ch2, WAVEFORM_SAMPLES,
                    DAC_ALIGN_12B_R);
}
```

### 2. 노이즈 생성

```c
void Generate_Noise_Wave(uint16_t *buffer, uint16_t samples)
{
  // 선형 피드백 시프트 레지스터 (LFSR)를 사용한 의사 난수
  uint32_t lfsr = 0xACE1u;
  uint32_t bit;

  for (uint16_t i = 0; i < samples; i++)
  {
    // LFSR 업데이트
    bit = ((lfsr >> 0) ^ (lfsr >> 2) ^ (lfsr >> 3) ^ (lfsr >> 5)) & 1;
    lfsr = (lfsr >> 1) | (bit << 15);

    // 12비트로 스케일링
    buffer[i] = lfsr & 0xFFF;
  }
}

// 또는 DAC 하드웨어 노이즈 생성기 사용
void DAC_Hardware_Noise_Init(void)
{
  DAC_ChannelConfTypeDef sConfig = {0};

  sConfig.DAC_Trigger = DAC_TRIGGER_NONE;
  sConfig.DAC_OutputBuffer = DAC_OUTPUTBUFFER_ENABLE;

  // 노이즈 파형 생성 모드
  sConfig.DAC_ConnectOnChipPeripheral = DAC_CHIPCONNECT_EXTERNAL;
  sConfig.DAC_UserTrimming = DAC_TRIMMING_FACTORY;

  HAL_DAC_ConfigChannel(&hdac1, &sConfig, DAC_CHANNEL_1);

  // 노이즈 생성 시작
  HAL_DACEx_NoiseWaveGenerate(&hdac1, DAC_CHANNEL_1,
                               DAC_LFSRUNMASK_BITS11_0);  // 12비트 마스크
  HAL_DAC_Start(&hdac1, DAC_CHANNEL_1);
}
```

### 3. 삼각파 생성기 (하드웨어)

```c
void DAC_Hardware_Triangle_Init(void)
{
  DAC_ChannelConfTypeDef sConfig = {0};

  sConfig.DAC_Trigger = DAC_TRIGGER_T6_TRGO;
  sConfig.DAC_OutputBuffer = DAC_OUTPUTBUFFER_ENABLE;

  HAL_DAC_ConfigChannel(&hdac1, &sConfig, DAC_CHANNEL_1);

  // 삼각파 생성 시작 (진폭: 12비트)
  HAL_DACEx_TriangleWaveGenerate(&hdac1, DAC_CHANNEL_1,
                                  DAC_TRIANGLEAMPLITUDE_4095);
  HAL_DAC_Start(&hdac1, DAC_CHANNEL_1);
}
```

## 성능 최적화

### 메모리 사용 최적화

```c
// Flash에 파형 데이터 저장 (ROM)
const uint16_t sine_wave_rom[256] __attribute__((section(".rodata"))) = {
  2048, 2098, 2148, /* ... 나머지 데이터 ... */
};

// 런타임에 RAM으로 복사
memcpy(sine_wave_ram, sine_wave_rom, sizeof(sine_wave_rom));
```

### DMA 성능

```
DAC 업데이트 주파수: 256kHz
데이터 폭: 16비트
DMA 대역폭: 256k × 2 = 512 KB/s (매우 낮음)
```

## 빌드 및 실행

### 테스트 설정

```
┌──────────────┐
│ STM32H745I   │
│              │
│   PA4 (DAC1) │──────┬──────► 오실로스코프
│              │      │
└──────────────┘      │
                      │
                      ▼
               ┌─────────────┐
               │  RC Filter  │ (선택 사항)
               │  R=1k       │
               │  C=100nF    │
               └─────────────┘
```

### 예상 출력

```
삼각파:
  주파수: 1kHz
  진폭: 0V ~ 3.3V
  파형: /\/\/\/\

계단파:
  주파수: 1kHz
  진폭: 8단계 (0V ~ 3.3V)
  파형: ┌─┐┌─┐┌─┐
```

## 트러블슈팅

### DAC 출력이 없는 경우

```c
// GPIO 모드 확인
if ((GPIOA->MODER & (3 << 8)) == (3 << 8))  // PA4 = 아날로그
{
  printf("PA4 is in analog mode\n");
}

// DAC 활성화 확인
if (DAC1->CR & DAC_CR_EN1)
{
  printf("DAC1 Channel 1 enabled\n");
}
```

### 파형이 왜곡되는 경우

```c
// 출력 버퍼 확인
// 고임피던스 부하에는 버퍼 활성화 필요

// 샘플링 속도 감소
// 너무 빠른 업데이트는 왜곡 발생 가능
```

### DMA 전송 오류

```c
// DMA 상태 확인
if (HAL_DMA_GetState(&hdma_dac1_ch1) == HAL_DMA_STATE_ERROR)
{
  printf("DMA error detected\n");
  HAL_DMA_Abort(&hdma_dac1_ch1);
  HAL_DMA_Start(&hdma_dac1_ch1, ...);  // 재시작
}
```

## 참고 자료

- **RM0399**: STM32H745 Reference Manual, Chapter 27 (DAC)
- **AN4566**: Extending the DAC performance of STM32 MCUs
- **예제 코드**: `STM32Cube_FW_H7/Projects/STM32H745I-DISCO/Examples/DAC/DAC_SignalsGeneration`

## 관련 예제

- **TIM_DMA**: 타이머 DMA PWM
- **DMA_DMAMUX**: DMA 설정
- **ADC_DualModeInterleaved**: ADC와 DAC 조합
