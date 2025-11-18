# PWR_D1ON_D2OFF - D1 도메인 ON, D2 도메인 OFF

## 개요

이 예제는 STM32H745 듀얼 코어 MCU에서 D1 도메인(Cortex-M7)만 활성화하고 D2 도메인(Cortex-M4)을 STANDBY 모드로 유지하는 방법을 보여줍니다. 단일 코어만 필요한 애플리케이션에서 전력 소비를 최적화할 수 있으며, M7의 고성능이 필요하지만 M4는 사용하지 않는 경우에 유용합니다.

## 하드웨어 요구사항

- **STM32H745I-DISCO 보드**
- **LED1**: PI12 (주황색) - D1/M7 활성 상태
- **LED2**: PI13 (초록색) - 작업 표시
- **USER 버튼**: PC13 - 인터럽트 트리거
- **전류계**: 전력 소비 측정용
- **UART**: 디버깅 메시지 출력

## STM32H745 듀얼 코어 아키텍처

### 도메인 구조

```
┌─────────────────────────────────────────────────────────┐
│              STM32H745 Dual Core MCU                     │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────────────────────────────────────┐  ★ ON   │
│  │         D1 Domain (CPU Domain)             │         │
│  ├────────────────────────────────────────────┤         │
│  │  • Cortex-M7 @ 480 MHz                     │         │
│  │    - 32-bit RISC processor                 │         │
│  │    - Single/Double precision FPU           │         │
│  │    - L1 Cache: 16KB I-Cache + 16KB D-Cache │         │
│  │    - MPU with 16 regions                   │         │
│  │    - ETM for debug                         │         │
│  │                                            │         │
│  │  • Memory                                  │         │
│  │    - AXI SRAM: 512 KB                      │         │
│  │    - DTCM RAM: 128 KB (TCM)                │         │
│  │    - ITCM RAM: 64 KB (TCM)                 │         │
│  │                                            │         │
│  │  • Peripherals                             │         │
│  │    - DMA1, DMA2 (16 streams each)          │         │
│  │    - MDMA (Master DMA)                     │         │
│  │    - Ethernet MAC                          │         │
│  │    - USB OTG HS                            │         │
│  │    - DCMI (Camera interface)               │         │
│  │    - TIM1, TIM8 (Advanced timers)          │         │
│  │    - JPEG codec                            │         │
│  └────────────────────────────────────────────┘         │
│                                                          │
│  ┌────────────────────────────────────────────┐  ★ OFF  │
│  │         D2 Domain (Peripheral Domain)      │         │
│  ├────────────────────────────────────────────┤         │
│  │  • Cortex-M4 @ 240 MHz (STANDBY)           │         │
│  │  • AHB SRAM: 288 KB (Retention OFF)        │         │
│  │  • Peripherals (All OFF)                   │         │
│  │    - ADC1, ADC2, DAC1                      │         │
│  │    - TIM2-7, TIM12-17                      │         │
│  │    - SPI1-5, I2C1-3                        │         │
│  │    - USART1-6, UART7-8                     │         │
│  │    - FDCAN, SDMMC                          │         │
│  └────────────────────────────────────────────┘         │
│                                                          │
│  ┌────────────────────────────────────────────┐         │
│  │      D3 Domain (Autonomous Domain)         │         │
│  │  • RTC, IWDG                               │         │
│  │  • LPTIM, LPUART, I2C4, SPI6               │         │
│  │  • SRD SRAM: 64 KB                         │         │
│  └────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────┘
```

### 전력 모드 비교

```
구성                  │ D1 (M7)  │ D2 (M4)  │ 전력 소비  │ 성능
────────────────────┼─────────┼─────────┼───────────┼──────────
양쪽 코어 RUN        │ RUN      │ RUN      │ ~250 mA    │ 최대
D1 ON, D2 OFF ★     │ RUN      │ STANDBY  │ ~170 mA    │ M7 고성능
D1 OFF, D2 ON       │ STANDBY  │ RUN      │ ~100 mA    │ M4 중성능
양쪽 코어 STOP       │ STOP     │ STOP     │ ~100 μA    │ 저전력 대기
양쪽 코어 STANDBY    │ STANDBY  │ STANDBY  │ ~2.4 μA    │ 최저 전력
```

## 주요 기능

### D1 도메인만 사용하는 이유

**사용 사례**

1. **단일 코어 애플리케이션**
   - M4가 필요 없는 경우
   - 전력 소비 최적화
   - M7의 고성능만 사용

2. **고성능 연산**
   - DSP 처리
   - 부동소수점 연산
   - 이미지 처리
   - 암호화

3. **캐시 활용**
   - I-Cache, D-Cache로 성능 향상
   - 메모리 대역폭 최적화

4. **고속 주변장치**
   - Ethernet
   - USB HS
   - DCMI
   - JPEG

### 전력 절감 효과

```
전체 시스템 대비 절감:
- 양쪽 코어 RUN: 250 mA
- D1만 RUN:       170 mA
- 절감:           80 mA (32% 감소)

배터리 수명 연장:
- 2000 mAh 배터리 기준
- 양쪽 코어: 8시간
- D1만:      11.8시간 (47% 연장)
```

## 동작 원리

### 부팅 시퀀스

```
┌─────────────────┐
│  Power-On Reset │
│   (System Boot) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Boot CM7 Core  │  M7 먼저 부팅 (항상)
│  (Cortex-M7)    │  Flash: 0x08000000
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ SystemInit()    │
│ - Clock config  │
│ - Flash setup   │
│ - Cache enable  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ main() on M7    │
│                 │
└────────┬────────┘
         │
         ├───► CM4 Boot? ────► NO
         │     (Hold)
         │
         ▼
┌─────────────────┐
│ D2 Domain       │
│ Standby Setup   │
│ - Power down M4 │
│ - D2 STANDBY    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   M7 Running    │  ★ D1 도메인만 활성
│   M4 Stopped    │    D2 도메인 STANDBY
│                 │
│ D1: RUN         │    전력: ~170 mA
│ D2: STANDBY     │
└─────────────────┘
```

### 메모리 접근

```
┌─────────────────────────────────────────────┐
│           Memory Map (D1 ON, D2 OFF)        │
├─────────────────────────────────────────────┤
│                                             │
│  M7 접근 가능:                               │
│  ┌───────────────────────────────────┐     │
│  │ Flash:         0x0800_0000 (2MB)  │ ✓   │
│  │ DTCM:          0x2000_0000 (128KB)│ ✓   │
│  │ ITCM:          0x0000_0000 (64KB) │ ✓   │
│  │ AXI SRAM:      0x2400_0000 (512KB)│ ✓   │
│  │ SRAM1:         0x3000_0000 (128KB)│ ✓*  │
│  │ SRAM2:         0x3002_0000 (128KB)│ ✓*  │
│  │ SRAM3:         0x3004_0000 (32KB) │ ✓*  │
│  │ SRAM4:         0x3800_0000 (64KB) │ ✓   │
│  └───────────────────────────────────┘     │
│                                             │
│  * D2 SRAM은 접근 가능하나 저전력 유지     │
│    필요 시 클럭 활성화해야 함                │
│                                             │
│  M4 접근 불가:                               │
│  ┌───────────────────────────────────┐     │
│  │ M4 Flash:      0x0810_0000        │ ✗   │
│  │ M4 SRAM:       D2 SRAM            │ ✗   │
│  └───────────────────────────────────┘     │
└─────────────────────────────────────────────┘
```

## 코드 구현

### 1. 시스템 초기화 (M7만 사용)

```c
#include "stm32h7xx_hal.h"

int main(void)
{
  // HAL 초기화
  HAL_Init();

  // 시스템 클럭 설정 (M7: 480MHz)
  SystemClock_Config();

  // CPU 캐시 활성화
  SCB_EnableICache();
  SCB_EnableDCache();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // D2 도메인 비활성화 (M4 사용 안 함)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  printf("\n========================================\n");
  printf("STM32H745 D1 Only Mode (M7 Active)\n");
  printf("========================================\n\n");

  // M4 부팅 방지 (Hold mechanism)
  Disable_CM4_Boot();

  // D2 도메인 STANDBY 설정
  Configure_D2_Standby();

  // GPIO 초기화 (D1 도메인 전용)
  GPIO_Init();

  // UART 초기화 (D1 도메인 전용)
  UART_Init();

  printf("System Configuration:\n");
  printf("  • M7 Core: RUNNING @ 480 MHz\n");
  printf("  • M4 Core: STOPPED (STANDBY)\n");
  printf("  • D1 Domain: ACTIVE\n");
  printf("  • D2 Domain: STANDBY\n");
  printf("  • Power: ~170 mA\n\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // M7 메인 루프
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  uint32_t counter = 0;

  while (1)
  {
    // M7 고성능 작업 수행
    Perform_M7_Tasks();

    // LED 토글
    HAL_GPIO_TogglePin(GPIOI, GPIO_PIN_12);

    HAL_Delay(1000);

    counter++;
    printf("[M7] Running... count=%lu\n", counter);
  }
}
```

### 2. M4 부팅 방지

```c
void Disable_CM4_Boot(void)
{
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // CM4 부팅 방지 방법
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // 옵션 1: RCC 레지스터로 제어
  // CM4를 리셋 상태로 유지
  __HAL_RCC_CM4_FORCE_RESET();

  printf("CM4 boot disabled (held in reset)\n");

  // 옵션 2: Option Bytes 사용 (영구 설정)
  // STM32CubeProgrammer로 설정:
  // - BCM4: Boot CM4 disable (set to 0)
  // 이 방법은 프로그래밍 시 한 번만 설정하면 됨
}
```

### 3. D2 도메인 STANDBY 설정

```c
void Configure_D2_Standby(void)
{
  printf("Configuring D2 domain for STANDBY...\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 1. D2 주변장치 클럭 비활성화
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // D2 도메인 주변장치 클럭 모두 OFF
  RCC->AHB2ENR = 0;   // DCMI, CRYP, HASH, RNG, SDMMC, USB OTG FS
  RCC->AHB1ENR = 0;   // DMA1, DMA2, ADC12, Ethernet, USB OTG HS
  RCC->APB1LENR = 0;  // TIM2-7, SPI2-3, USART2-3, I2C1-3, etc.
  RCC->APB1HENR = 0;  // FDCAN, MDIOS, OPAMP
  RCC->APB2ENR = 0;   // TIM1, TIM8, USART1, SPI1, SPI4-5

  printf("  • D2 peripheral clocks disabled\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 2. D2 SRAM 저전력 모드
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // SRAM1, SRAM2, SRAM3 retention OFF (더 낮은 전력)
  // 주의: 데이터가 손실됨
  RCC->AHB2SMENR &= ~(RCC_AHB2SMENR_SRAM1SMEN |
                      RCC_AHB2SMENR_SRAM2SMEN |
                      RCC_AHB2SMENR_SRAM3SMEN);

  printf("  • D2 SRAM retention disabled\n");

  // 또는 retention ON (데이터 보존, 약간 높은 전력)
  // RCC->AHB2SMENR |= (RCC_AHB2SMENR_SRAM1SMEN |
  //                    RCC_AHB2SMENR_SRAM2SMEN |
  //                    RCC_AHB2SMENR_SRAM3SMEN);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 3. D2 도메인 전력 모드 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // PWR 클럭 활성화
  __HAL_RCC_PWR_CLK_ENABLE();

  // D2 도메인을 STANDBY 모드로 설정
  // (실제 진입은 WFI/WFE 명령 필요)
  HAL_PWREx_EnterSTANDBYMode(PWR_D2_DOMAIN);

  printf("D2 domain configured for STANDBY\n\n");
}
```

### 4. D1 전용 주변장치 초기화

```c
void Init_D1_Peripherals(void)
{
  printf("Initializing D1 peripherals...\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // DMA2 (D1 domain)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  __HAL_RCC_DMA2_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // MDMA (Master DMA, D1 domain)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  __HAL_RCC_MDMA_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // Ethernet (D1 domain)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // 필요한 경우
  // __HAL_RCC_ETH1MAC_CLK_ENABLE();
  // __HAL_RCC_ETH1TX_CLK_ENABLE();
  // __HAL_RCC_ETH1RX_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // USB OTG HS (D1 domain)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // 필요한 경우
  // __HAL_RCC_USB_OTG_HS_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // DCMI (Camera interface, D1 domain)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // 필요한 경우
  // __HAL_RCC_DCMI_CLK_ENABLE();

  printf("D1 peripherals initialized\n\n");
}
```

### 5. 메모리 접근 최적화

```c
// DTCM (Data Tightly Coupled Memory)
// - M7 전용 초고속 메모리
// - 캐시 없이도 0-wait 접근
// - 스택, 힙, 중요 변수에 사용

__attribute__((section(".dtcm"))) uint8_t dtcm_buffer[4096];
__attribute__((section(".dtcm"))) volatile uint32_t fast_counter;

// ITCM (Instruction Tightly Coupled Memory)
// - M7 전용 명령어 메모리
// - 0-wait 접근
// - 시간 중요 함수 배치

__attribute__((section(".itcm"))) void Critical_Function(void)
{
  // 초고속 실행이 필요한 코드
  fast_counter++;
}

// AXI SRAM (512KB, D1 domain)
// - 일반 데이터 저장
// - DMA 접근 가능

__attribute__((section(".axisram"))) uint32_t large_buffer[128*1024/4];

void Memory_Test(void)
{
  printf("Testing M7 memory access...\n");

  // DTCM 쓰기 테스트
  uint32_t start = HAL_GetTick();
  for (uint32_t i = 0; i < 4096; i++)
  {
    dtcm_buffer[i] = i & 0xFF;
  }
  uint32_t dtcm_time = HAL_GetTick() - start;

  // AXI SRAM 쓰기 테스트
  start = HAL_GetTick();
  for (uint32_t i = 0; i < 4096; i++)
  {
    large_buffer[i] = i;
  }
  uint32_t axi_time = HAL_GetTick() - start;

  printf("  • DTCM write: %lu ms\n", dtcm_time);
  printf("  • AXI SRAM write: %lu ms\n", axi_time);
}
```

### 6. 캐시 활용

```c
void Cache_Demo(void)
{
  printf("Cache demonstration...\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // I-Cache (Instruction Cache)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  SCB_EnableICache();
  printf("  • I-Cache enabled (16KB)\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // D-Cache (Data Cache)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  SCB_EnableDCache();
  printf("  • D-Cache enabled (16KB)\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // DMA 사용 시 주의사항
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // DMA 전송 전: 캐시 정리
  SCB_CleanDCache_by_Addr((uint32_t*)large_buffer, sizeof(large_buffer));

  // DMA 전송...

  // DMA 전송 후: 캐시 무효화
  SCB_InvalidateDCache_by_Addr((uint32_t*)large_buffer, sizeof(large_buffer));

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // MPU (Memory Protection Unit) 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // DMA 버퍼를 non-cacheable로 설정 (선택사항)
  MPU_Region_InitTypeDef MPU_InitStruct = {0};

  HAL_MPU_Disable();

  // Region 0: DMA 버퍼 (non-cacheable)
  MPU_InitStruct.Enable = MPU_REGION_ENABLE;
  MPU_InitStruct.Number = MPU_REGION_NUMBER0;
  MPU_InitStruct.BaseAddress = (uint32_t)&large_buffer;
  MPU_InitStruct.Size = MPU_REGION_SIZE_512KB;
  MPU_InitStruct.SubRegionDisable = 0x00;
  MPU_InitStruct.TypeExtField = MPU_TEX_LEVEL0;
  MPU_InitStruct.AccessPermission = MPU_REGION_FULL_ACCESS;
  MPU_InitStruct.DisableExec = MPU_INSTRUCTION_ACCESS_DISABLE;
  MPU_InitStruct.IsShareable = MPU_ACCESS_NOT_SHAREABLE;
  MPU_InitStruct.IsCacheable = MPU_ACCESS_NOT_CACHEABLE;
  MPU_InitStruct.IsBufferable = MPU_ACCESS_BUFFERABLE;

  HAL_MPU_ConfigRegion(&MPU_InitStruct);

  HAL_MPU_Enable(MPU_PRIVILEGED_DEFAULT);

  printf("  • MPU configured for DMA buffers\n");
}
```

## 고급 기능

### 1. DSP 가속 (M7 전용)

```c
#include "arm_math.h"

// CMSIS-DSP 라이브러리 활용
// M7의 FPU와 SIMD 명령어 사용

#define FFT_SIZE  1024

float32_t fft_input[FFT_SIZE];
float32_t fft_output[FFT_SIZE * 2];

arm_rfft_fast_instance_f32 fft_instance;

void DSP_Processing_Demo(void)
{
  printf("DSP processing with M7 FPU...\n");

  // FFT 초기화
  arm_rfft_fast_init_f32(&fft_instance, FFT_SIZE);

  // 입력 신호 생성 (1kHz sine wave)
  for (uint32_t i = 0; i < FFT_SIZE; i++)
  {
    float t = (float)i / FFT_SIZE;
    fft_input[i] = arm_sin_f32(2.0f * PI * 1000.0f * t);
  }

  // FFT 수행
  uint32_t start = HAL_GetTick();
  arm_rfft_fast_f32(&fft_instance, fft_input, fft_output, 0);
  uint32_t fft_time = HAL_GetTick() - start;

  printf("  • FFT-%d completed in %lu ms\n", FFT_SIZE, fft_time);

  // 주파수 스펙트럼 분석
  float32_t max_value;
  uint32_t max_index;
  arm_max_f32(fft_output, FFT_SIZE, &max_value, &max_index);

  printf("  • Peak at bin %lu (%.2f Hz)\n",
         max_index, (float)max_index * 48000.0f / FFT_SIZE);
}
```

### 2. JPEG 하드웨어 가속 (M7 전용)

```c
#ifdef HAL_JPEG_MODULE_ENABLED

JPEG_HandleTypeDef hjpeg;

void JPEG_Encode_Demo(void)
{
  printf("JPEG hardware encoding...\n");

  // JPEG 클럭 활성화
  __HAL_RCC_JPEG_CLK_ENABLE();

  // JPEG 초기화
  hjpeg.Instance = JPEG;
  if (HAL_JPEG_Init(&hjpeg) != HAL_OK)
  {
    Error_Handler();
  }

  // 이미지 데이터 (예: 320x240 RGB565)
  uint16_t *image_data = (uint16_t*)0x24000000;  // AXI SRAM
  uint32_t image_size = 320 * 240 * 2;

  // JPEG 압축 버퍼
  uint8_t *jpeg_buffer = (uint8_t*)0x24100000;
  uint32_t jpeg_size = 0;

  // JPEG 인코딩 시작
  uint32_t start = HAL_GetTick();

  JPEG_ConfTypeDef conf = {0};
  conf.ImageWidth = 320;
  conf.ImageHeight = 240;
  conf.ColorSpace = JPEG_YCBCR_COLORSPACE;
  conf.ChromaSubsampling = JPEG_420_SUBSAMPLING;
  conf.ImageQuality = 75;

  HAL_JPEG_ConfigEncoding(&hjpeg, &conf);

  // DMA로 인코딩 (비동기)
  HAL_JPEG_Encode_DMA(&hjpeg,
                      (uint8_t*)image_data,
                      image_size,
                      jpeg_buffer,
                      &jpeg_size);

  // 완료 대기
  while (HAL_JPEG_GetState(&hjpeg) == HAL_JPEG_STATE_BUSY_ENCODING)
  {
    HAL_Delay(1);
  }

  uint32_t encode_time = HAL_GetTick() - start;

  printf("  • JPEG encoded in %lu ms\n", encode_time);
  printf("  • Original: %lu bytes\n", image_size);
  printf("  • Compressed: %lu bytes (%.1f%%)\n",
         jpeg_size, (float)jpeg_size * 100 / image_size);
}

#endif
```

### 3. Ethernet (M7 전용)

```c
#ifdef HAL_ETH_MODULE_ENABLED

ETH_HandleTypeDef heth;

void Ethernet_Demo(void)
{
  printf("Ethernet initialization (M7 only)...\n");

  // Ethernet 클럭 활성화
  __HAL_RCC_ETH1MAC_CLK_ENABLE();
  __HAL_RCC_ETH1TX_CLK_ENABLE();
  __HAL_RCC_ETH1RX_CLK_ENABLE();

  // GPIO 설정 (RMII 인터페이스)
  // PA1: ETH_REF_CLK
  // PA2: ETH_MDIO
  // PA7: ETH_CRS_DV
  // PC1: ETH_MDC
  // PC4: ETH_RXD0
  // PC5: ETH_RXD1
  // PG11: ETH_TX_EN
  // PG13: ETH_TXD0
  // PG12: ETH_TXD1

  // ... GPIO 초기화 코드 ...

  // Ethernet 초기화
  heth.Instance = ETH;
  heth.Init.MACAddr[0] = 0x00;
  heth.Init.MACAddr[1] = 0x80;
  heth.Init.MACAddr[2] = 0xE1;
  heth.Init.MACAddr[3] = 0x00;
  heth.Init.MACAddr[4] = 0x00;
  heth.Init.MACAddr[5] = 0x00;
  heth.Init.MediaInterface = HAL_ETH_RMII_MODE;
  heth.Init.RxDesc = DMARxDscrTab;
  heth.Init.TxDesc = DMATxDscrTab;
  heth.Init.RxBuffLen = 1524;

  if (HAL_ETH_Init(&heth) != HAL_OK)
  {
    Error_Handler();
  }

  printf("  • Ethernet MAC initialized\n");
  printf("  • MAC Address: %02X:%02X:%02X:%02X:%02X:%02X\n",
         heth.Init.MACAddr[0], heth.Init.MACAddr[1],
         heth.Init.MACAddr[2], heth.Init.MACAddr[3],
         heth.Init.MACAddr[4], heth.Init.MACAddr[5]);

  // PHY 초기화
  uint32_t phyreg = 0;
  HAL_ETH_ReadPHYRegister(&heth, PHY_BCR, &phyreg);
  printf("  • PHY detected: 0x%08lX\n", phyreg);
}

#endif
```

### 4. 고속 데이터 처리

```c
// MDMA (Master DMA)를 사용한 고속 메모리 복사
// M7 전용 DMA, 최대 1.2 GB/s

MDMA_HandleTypeDef hmdma;

void MDMA_Fast_Copy_Demo(void)
{
  printf("MDMA fast memory copy...\n");

  // MDMA 클럭 활성화
  __HAL_RCC_MDMA_CLK_ENABLE();

  // 소스 및 목적지 버퍼
  uint32_t *src = (uint32_t*)0x24000000;   // AXI SRAM
  uint32_t *dst = (uint32_t*)0x30000000;   // SRAM1

  uint32_t size = 1024 * 1024;  // 1MB

  // 소스 버퍼 초기화
  for (uint32_t i = 0; i < size / 4; i++)
  {
    src[i] = i;
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 방법 1: memcpy (소프트웨어)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  uint32_t start = HAL_GetTick();
  memcpy(dst, src, size);
  uint32_t memcpy_time = HAL_GetTick() - start;

  printf("  • memcpy: %lu ms (%.2f MB/s)\n",
         memcpy_time, (float)size / 1024 / 1024 / memcpy_time * 1000);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 방법 2: MDMA (하드웨어)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  hmdma.Instance = MDMA_Channel0;
  hmdma.Init.Request = MDMA_REQUEST_SW;
  hmdma.Init.TransferTriggerMode = MDMA_BLOCK_TRANSFER;
  hmdma.Init.Priority = MDMA_PRIORITY_HIGH;
  hmdma.Init.Endianness = MDMA_LITTLE_ENDIANNESS_PRESERVE;
  hmdma.Init.SourceInc = MDMA_SRC_INC_WORD;
  hmdma.Init.DestinationInc = MDMA_DEST_INC_WORD;
  hmdma.Init.SourceDataSize = MDMA_SRC_DATASIZE_WORD;
  hmdma.Init.DestDataSize = MDMA_DEST_DATASIZE_WORD;
  hmdma.Init.DataAlignment = MDMA_DATAALIGN_PACKENABLE;
  hmdma.Init.BufferTransferLength = 128;
  hmdma.Init.SourceBurst = MDMA_SOURCE_BURST_SINGLE;
  hmdma.Init.DestBurst = MDMA_DEST_BURST_SINGLE;
  hmdma.Init.SourceBlockAddressOffset = 0;
  hmdma.Init.DestBlockAddressOffset = 0;

  if (HAL_MDMA_Init(&hmdma) != HAL_OK)
  {
    Error_Handler();
  }

  start = HAL_GetTick();

  HAL_MDMA_Start(&hmdma, (uint32_t)src, (uint32_t)dst, size, 1);

  // 완료 대기
  HAL_MDMA_PollForTransfer(&hmdma, HAL_MDMA_FULL_TRANSFER, 1000);

  uint32_t mdma_time = HAL_GetTick() - start;

  printf("  • MDMA: %lu ms (%.2f MB/s)\n",
         mdma_time, (float)size / 1024 / 1024 / mdma_time * 1000);

  // 검증
  if (memcmp(src, dst, size) == 0)
  {
    printf("  • Verification: OK\n");
  }
  else
  {
    printf("  • Verification: FAILED\n");
  }
}
```

## 전력 소비 최적화

### 클럭 최적화

```c
void Optimize_Clocks_For_D1_Only(void)
{
  printf("Optimizing clocks for D1-only operation...\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // D2 도메인 클럭 완전 비활성화
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // AHB1 (D2 domain)
  RCC->AHB1ENR = 0;
  printf("  • AHB1 clocks disabled\n");

  // AHB2 (D2 domain)
  RCC->AHB2ENR = 0;
  printf("  • AHB2 clocks disabled\n");

  // APB1L/H (D2 domain)
  RCC->APB1LENR = 0;
  RCC->APB1HENR = 0;
  printf("  • APB1 clocks disabled\n");

  // APB2 (D2 domain)
  RCC->APB2ENR = 0;
  printf("  • APB2 clocks disabled\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // D1 도메인 클럭 최적화
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // 사용하지 않는 D1 주변장치도 비활성화
  // 예: DCMI, USB, Ethernet 등

  printf("Clock optimization completed\n");
  printf("Estimated power reduction: ~80 mA\n\n");
}
```

### 전력 측정

```c
void Measure_Power_Consumption(void)
{
  printf("=== Power Consumption Measurement ===\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 시나리오 1: 양쪽 코어 RUN
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  printf("\n1. Both cores RUN:\n");
  printf("   • M7 @ 480 MHz:      ~150 mA\n");
  printf("   • M4 @ 240 MHz:      ~80 mA\n");
  printf("   • Peripherals:       ~20 mA\n");
  printf("   • Total:             ~250 mA\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 시나리오 2: D1만 RUN (현재 설정)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  printf("\n2. D1 only (M7 RUN, M4 OFF):\n");
  printf("   • M7 @ 480 MHz:      ~150 mA\n");
  printf("   • M4 STANDBY:        ~1 μA\n");
  printf("   • D1 Peripherals:    ~20 mA\n");
  printf("   • Total:             ~170 mA\n");
  printf("   • Savings:           80 mA (32%%)\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 배터리 수명 계산
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  printf("\n3. Battery life (2000 mAh):\n");
  printf("   • Both cores:        8.0 hours\n");
  printf("   • D1 only:           11.8 hours\n");
  printf("   • Improvement:       +47%%\n");

  printf("\n=====================================\n\n");
}
```

## 실전 응용 예제

### 예제 1: 고성능 데이터 로거

```c
// M7의 고성능을 활용한 고속 데이터 로깅
// ADC + DMA + SD 카드

#define SAMPLE_RATE      100000  // 100 kHz
#define BUFFER_SIZE      10000

uint16_t adc_buffer[BUFFER_SIZE] __attribute__((section(".axisram")));

void High_Speed_Data_Logger(void)
{
  printf("High-speed data logger (M7 only)\n");

  // ADC3 초기화 (D1 domain)
  ADC_HandleTypeDef hadc3;
  __HAL_RCC_ADC3_CLK_ENABLE();

  hadc3.Instance = ADC3;
  hadc3.Init.ClockPrescaler = ADC_CLOCK_ASYNC_DIV2;
  hadc3.Init.Resolution = ADC_RESOLUTION_12B;
  hadc3.Init.ScanConvMode = ADC_SCAN_DISABLE;
  hadc3.Init.ContinuousConvMode = ENABLE;
  hadc3.Init.NbrOfConversion = 1;
  hadc3.Init.ConversionDataManagement = ADC_CONVERSIONDATA_DMA_CIRCULAR;

  HAL_ADC_Init(&hadc3);

  // DMA2 초기화 (D1 domain)
  DMA_HandleTypeDef hdma_adc;
  __HAL_RCC_DMA2_CLK_ENABLE();

  hdma_adc.Instance = DMA2_Stream0;
  hdma_adc.Init.Request = DMA_REQUEST_ADC3;
  hdma_adc.Init.Direction = DMA_PERIPH_TO_MEMORY;
  hdma_adc.Init.PeriphInc = DMA_PINC_DISABLE;
  hdma_adc.Init.MemInc = DMA_MINC_ENABLE;
  hdma_adc.Init.PeriphDataAlignment = DMA_PDATAALIGN_HALFWORD;
  hdma_adc.Init.MemDataAlignment = DMA_MDATAALIGN_HALFWORD;
  hdma_adc.Init.Mode = DMA_CIRCULAR;
  hdma_adc.Init.Priority = DMA_PRIORITY_HIGH;

  HAL_DMA_Init(&hdma_adc);
  __HAL_LINKDMA(&hadc3, DMA_Handle, hdma_adc);

  // ADC + DMA 시작
  HAL_ADC_Start_DMA(&hadc3, (uint32_t*)adc_buffer, BUFFER_SIZE);

  printf("Sampling at %d Hz...\n", SAMPLE_RATE);

  while (1)
  {
    // Half transfer callback에서 처리
    // 또는 버퍼가 가득 차면 SD 카드에 저장
    if (/* buffer full */)
    {
      // SD 카드에 저장
      Save_To_SD_Card(adc_buffer, BUFFER_SIZE);
    }

    HAL_Delay(100);
  }
}
```

### 예제 2: 이미지 처리

```c
// DCMI + DMA + JPEG 인코더
// 카메라 이미지 캡처 및 압축

void Image_Processing_Pipeline(void)
{
  printf("Image processing pipeline (M7 only)\n");

  // DCMI 초기화 (D1 domain)
  DCMI_HandleTypeDef hdcmi;
  __HAL_RCC_DCMI_CLK_ENABLE();

  hdcmi.Instance = DCMI;
  hdcmi.Init.SynchroMode = DCMI_SYNCHRO_HARDWARE;
  hdcmi.Init.PCKPolarity = DCMI_PCKPOLARITY_RISING;
  hdcmi.Init.VSPolarity = DCMI_VSPOLARITY_HIGH;
  hdcmi.Init.HSPolarity = DCMI_HSPOLARITY_LOW;
  hdcmi.Init.CaptureRate = DCMI_CR_ALL_FRAME;
  hdcmi.Init.ExtendedDataMode = DCMI_EXTEND_DATA_8B;
  hdcmi.Init.JPEGMode = DCMI_JPEG_DISABLE;

  HAL_DCMI_Init(&hdcmi);

  // 이미지 버퍼 (AXI SRAM)
  uint8_t *image_buffer = (uint8_t*)0x24000000;
  uint32_t image_size = 640 * 480 * 2;  // RGB565

  printf("Capturing image from camera...\n");

  // DCMI + DMA로 이미지 캡처
  HAL_DCMI_Start_DMA(&hdcmi,
                     DCMI_MODE_SNAPSHOT,
                     (uint32_t)image_buffer,
                     image_size / 4);

  // 캡처 완료 대기
  while (HAL_DCMI_GetState(&hdcmi) == HAL_DCMI_STATE_BUSY);

  printf("Image captured (%lu bytes)\n", image_size);

  // JPEG 압축 (M7 하드웨어 가속)
  printf("Compressing with JPEG...\n");

  uint8_t *jpeg_buffer = (uint8_t*)0x24100000;
  uint32_t jpeg_size = 0;

  JPEG_Encode(image_buffer, image_size, jpeg_buffer, &jpeg_size);

  printf("JPEG compressed (%lu bytes, %.1f%% compression)\n",
         jpeg_size, (float)jpeg_size * 100 / image_size);

  // 네트워크 전송 또는 저장
  Transmit_Over_Network(jpeg_buffer, jpeg_size);
}
```

## 트러블슈팅

### 문제 1: M4가 자동으로 부팅됨

```c
void Fix_CM4_Auto_Boot(void)
{
  printf("Fixing CM4 auto-boot issue...\n");

  // Option Bytes 확인
  FLASH_OBProgramInitTypeDef OBInit;
  HAL_FLASHEx_OBGetConfig(&OBInit);

  printf("Current BCM4: 0x%08lX\n", OBInit.USERConfig & FLASH_OPTSR_BCM4);

  if (OBInit.USERConfig & FLASH_OPTSR_BCM4)
  {
    printf("CM4 boot is enabled in option bytes\n");
    printf("Disabling CM4 boot...\n");

    // Option Bytes 잠금 해제
    HAL_FLASH_Unlock();
    HAL_FLASH_OB_Unlock();

    // BCM4 비트 클리어
    OBInit.OptionType = OPTIONBYTE_USER;
    OBInit.USERType = OB_USER_BCM4;
    OBInit.USERConfig &= ~FLASH_OPTSR_BCM4;

    HAL_FLASHEx_OBProgram(&OBInit);

    // Option Bytes 적용
    HAL_FLASH_OB_Launch();

    // 잠금
    HAL_FLASH_OB_Lock();
    HAL_FLASH_Lock();

    printf("CM4 boot disabled. Please reset the board.\n");
  }
  else
  {
    printf("CM4 boot already disabled\n");
  }
}
```

### 문제 2: D2 SRAM 접근 오류

```c
void Fix_D2_SRAM_Access(void)
{
  // D2 SRAM을 M7에서 접근하려면 클럭 필요

  printf("Enabling D2 SRAM clocks for M7 access...\n");

  // SRAM1, SRAM2, SRAM3 클럭 활성화
  __HAL_RCC_D2SRAM1_CLK_ENABLE();
  __HAL_RCC_D2SRAM2_CLK_ENABLE();
  __HAL_RCC_D2SRAM3_CLK_ENABLE();

  printf("D2 SRAM accessible from M7\n");

  // 테스트
  volatile uint32_t *sram1 = (uint32_t*)0x30000000;
  *sram1 = 0x12345678;

  if (*sram1 == 0x12345678)
  {
    printf("D2 SRAM1 test: OK\n");
  }
  else
  {
    printf("D2 SRAM1 test: FAILED\n");
  }
}
```

### 문제 3: 캐시 일관성 문제

```c
void Fix_Cache_Coherency(void)
{
  printf("Fixing cache coherency issues...\n");

  // DMA 버퍼는 항상 캐시 정리/무효화 필요

  uint32_t *dma_buffer = (uint32_t*)large_buffer;
  uint32_t size = sizeof(large_buffer);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // DMA 전송 전
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // 캐시된 데이터를 메모리에 쓰기
  SCB_CleanDCache_by_Addr((uint32_t*)dma_buffer, size);

  // DMA 전송 시작...

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // DMA 전송 후
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // 캐시 무효화 (메모리에서 새로 읽기)
  SCB_InvalidateDCache_by_Addr((uint32_t*)dma_buffer, size);

  printf("Cache coherency maintained\n");
}
```

## 참고 자료

- **RM0399**: STM32H745 Reference Manual
  - Chapter 2: Memory and bus architecture
  - Chapter 3: Embedded Flash memory
  - Chapter 4: Cortex-M7 with FPU
  - Chapter 6: Power controller

- **PM0253**: STM32H7 programming manual (Cortex-M7)

- **AN5361**: Getting started with projects based on dual-core STM32H7 microcontrollers

- **AN4839**: Managing memory protection unit in STM32 MCUs

## 관련 예제

- **PWR_D2ON_D1OFF**: D2만 활성화 (M4 전용)
- **PWR_Domain3SystemControl**: D3 자율 동작
- **PWR_Hold_Mechanism**: 듀얼 코어 부팅 제어
- **DMA_MDMA**: MDMA 고급 사용법
