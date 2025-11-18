# LTDC 2레이어 디스플레이 - 알파 블렌딩

## 개요

이 예제는 STM32H745I-DISCO 보드의 LTDC(LCD-TFT Display Controller) 주변장치를 사용하여 두 개의 레이어를 동시에 표시하고 알파 블렌딩을 수행하는 방법을 보여줍니다. 레이어 0(배경)과 레이어 1(전경)을 독립적으로 제어하며, 각 레이어의 위치를 동적으로 변경하여 매끄러운 애니메이션 효과를 구현합니다.

## 주요 기능

### 1. 2레이어 구조
- **레이어 0 (배경)**: L8 픽셀 포맷 (256색 팔레트)
- **레이어 1 (전경)**: RGB565 픽셀 포맷 (65,536색)
- **독립적인 위치 제어**: 각 레이어의 X, Y 좌표를 실시간으로 변경
- **알파 블렌딩**: 레이어 간 투명도 조절 및 혼합

### 2. 알파 블렌딩 모드
- **PAxCA (Pixel Alpha x Constant Alpha)**: 픽셀 알파값과 상수 알파값을 곱한 값 사용
- **레이어 0 알파**: 255 (완전 불투명)
- **레이어 1 알파**: 200 (약 78% 불투명)

### 3. CLUT (Color Look-Up Table)
- 레이어 0에 256색 CLUT 적용
- 메모리 효율적인 L8 포맷 사용
- 하드웨어 기반 색상 변환

## 하드웨어 구성

### LCD 사양
```
┌─────────────────────────────────────┐
│  RK043FN48H LCD Panel               │
│  - 해상도: 480 x 272 픽셀           │
│  - 인터페이스: RGB888               │
│  - 픽셀 클럭: 9.63 MHz              │
│  - 수평 동기: Active Low            │
│  - 수직 동기: Active Low            │
│  - 데이터 인에이블: Active Low      │
└─────────────────────────────────────┘
```

### 메모리 맵
```
┌──────────────────────────────────────────┐
│  Flash Memory (이미지 데이터 저장)       │
│  - L8_320x240: L8 포맷 이미지            │
│  - RGB565_320x240: RGB565 포맷 이미지    │
│  - L8_320x240_CLUT: 256색 팔레트        │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│  LTDC 레이어 구조                        │
│                                          │
│  ┌────────────────────────────┐         │
│  │  Layer 1 (전경)             │         │
│  │  - RGB565                  │         │
│  │  - Alpha: 200              │         │
│  │  - 위치: (160, 32)          │         │
│  └────────────────────────────┘         │
│         ↓ 알파 블렌딩                    │
│  ┌────────────────────────────┐         │
│  │  Layer 0 (배경)             │         │
│  │  - L8 + CLUT               │         │
│  │  - Alpha: 255              │         │
│  │  - 위치: (0, 0)             │         │
│  └────────────────────────────┘         │
└──────────────────────────────────────────┘
```

## 클럭 구성

### PLL3 (LCD 클럭 생성)
```c
// PLL3 설정
// 입력: HSE = 25 MHz
// PLL3_VCO = (HSE / PLL3M) * PLL3N = (25 / 5) * 160 = 800 MHz
// LCD_CLK = PLL3_VCO / PLL3R = 800 / 83 = 9.63 MHz

periph_clk_init_struct.PeriphClockSelection = RCC_PERIPHCLK_LTDC;
periph_clk_init_struct.PLL3.PLL3M = 5;
periph_clk_init_struct.PLL3.PLL3N = 160;
periph_clk_init_struct.PLL3.PLL3R = 83;  // LCD 클럭 분주기
periph_clk_init_struct.PLL3.PLL3VCOSEL = RCC_PLL3VCOWIDE;
periph_clk_init_struct.PLL3.PLL3RGE = RCC_PLL3VCIRANGE_2;
HAL_RCCEx_PeriphCLKConfig(&periph_clk_init_struct);
```

### 시스템 클럭
```
System Clock:
  CPU (CM7):    400 MHz
  AHB:          200 MHz
  APB1/2/3/4:   100 MHz
  LTDC Clock:   9.63 MHz
```

## 코드 상세 분석

### 1. LTDC 초기화

#### LCD 타이밍 설정
```c
static void LCD_Config(void)
{
  LTDC_HandleTypeDef LtdcHandle;

  // LTDC 극성 설정
  LtdcHandle.Init.HSPolarity = LTDC_HSPOLARITY_AL;     // 수평 동기: Active Low
  LtdcHandle.Init.VSPolarity = LTDC_VSPOLARITY_AL;     // 수직 동기: Active Low
  LtdcHandle.Init.DEPolarity = LTDC_DEPOLARITY_AL;     // 데이터 인에이블: Active Low
  LtdcHandle.Init.PCPolarity = LTDC_PCPOLARITY_IPC;    // 픽셀 클럭: 입력 클럭 사용

  // RK043FN48H LCD 타이밍 파라미터
  // 수평: HSYNC(41) + HBP(13) + Active(480) + HFP(32) = 566 클럭
  // 수직: VSYNC(10) + VBP(2) + Active(272) + VFP(2) = 286 라인
  LtdcHandle.Init.HorizontalSync = 40;                 // HSYNC - 1
  LtdcHandle.Init.VerticalSync = 9;                    // VSYNC - 1
  LtdcHandle.Init.AccumulatedHBP = 53;                 // HSYNC + HBP - 1
  LtdcHandle.Init.AccumulatedVBP = 11;                 // VSYNC + VBP - 1
  LtdcHandle.Init.AccumulatedActiveW = 533;            // HSYNC + HBP + Width - 1
  LtdcHandle.Init.AccumulatedActiveH = 283;            // VSYNC + VBP + Height - 1
  LtdcHandle.Init.TotalWidth = 565;                    // 전체 수평 - 1
  LtdcHandle.Init.TotalHeigh = 285;                    // 전체 수직 - 1

  // 배경 색상 (레이어가 없는 영역)
  LtdcHandle.Init.Backcolor.Blue = 0;
  LtdcHandle.Init.Backcolor.Green = 0;
  LtdcHandle.Init.Backcolor.Red = 0;

  HAL_LTDC_Init(&LtdcHandle);
}
```

### 2. 레이어 0 설정 (배경 - L8 포맷)

```c
// 레이어 0: L8 포맷 (256색 인덱스)
LTDC_LayerCfgTypeDef pLayerCfg;

// 윈도우 위치 및 크기
pLayerCfg.WindowX0 = 0;           // 시작 X 좌표
pLayerCfg.WindowX1 = 320;         // 끝 X 좌표 (320 픽셀 너비)
pLayerCfg.WindowY0 = 0;           // 시작 Y 좌표
pLayerCfg.WindowY1 = 240;         // 끝 Y 좌표 (240 픽셀 높이)

// 픽셀 포맷: L8 (8비트 인덱스)
// - 각 픽셀은 8비트 값으로 CLUT의 인덱스를 나타냄
// - CLUT에는 256개의 24비트 RGB 색상이 저장됨
// - 메모리 사용량: 320 x 240 x 1 = 76,800 바이트
pLayerCfg.PixelFormat = LTDC_PIXEL_FORMAT_L8;

// 프레임버퍼 주소 (Flash 메모리에 저장된 이미지)
pLayerCfg.FBStartAdress = (uint32_t)&L8_320x240;

// 알파 블렌딩 설정
pLayerCfg.Alpha = 255;            // 레이어 상수 알파 (255 = 완전 불투명)
pLayerCfg.Alpha0 = 0;             // 기본 알파값

// 블렌딩 팩터
// PAxCA: Pixel Alpha x Constant Alpha
// 최종 알파 = (픽셀 알파 / 255) * (상수 알파 / 255)
pLayerCfg.BlendingFactor1 = LTDC_BLENDING_FACTOR1_PAxCA;
pLayerCfg.BlendingFactor2 = LTDC_BLENDING_FACTOR2_PAxCA;

// 이미지 크기
pLayerCfg.ImageWidth = 320;
pLayerCfg.ImageHeight = 240;

// 기본 배경색 (윈도우 밖 영역)
pLayerCfg.Backcolor.Blue = 0;
pLayerCfg.Backcolor.Green = 0;
pLayerCfg.Backcolor.Red = 0;

// 레이어 설정 적용
HAL_LTDC_ConfigLayer(&LtdcHandle, &pLayerCfg, 0);  // 레이어 0

// CLUT 설정 (256색 팔레트)
// L8_320x240_CLUT 배열에는 256개의 ARGB8888 색상이 저장되어 있음
HAL_LTDC_ConfigCLUT(&LtdcHandle, (uint32_t *)L8_320x240_CLUT, 256, 0);

// CLUT 활성화
HAL_LTDC_EnableCLUT(&LtdcHandle, 0);
```

### 3. 레이어 1 설정 (전경 - RGB565 포맷)

```c
// 레이어 1: RGB565 포맷 (직접 색상)
LTDC_LayerCfgTypeDef pLayerCfg1;

// 윈도우 위치 및 크기
pLayerCfg1.WindowX0 = 160;        // 시작 X: 화면 중앙에서 시작
pLayerCfg1.WindowX1 = 480;        // 끝 X: 화면 오른쪽 끝 (320 픽셀 너비)
pLayerCfg1.WindowY0 = 32;         // 시작 Y: 위에서 32 픽셀 아래
pLayerCfg1.WindowY1 = 272;        // 끝 Y: 화면 아래쪽 끝 (240 픽셀 높이)

// 픽셀 포맷: RGB565
// - R: 5비트 (31 레벨)
// - G: 6비트 (63 레벨)
// - B: 5비트 (31 레벨)
// - 메모리 사용량: 320 x 240 x 2 = 153,600 바이트
pLayerCfg1.PixelFormat = LTDC_PIXEL_FORMAT_RGB565;

// 프레임버퍼 주소
pLayerCfg1.FBStartAdress = (uint32_t)&RGB565_320x240;

// 알파 블렌딩 설정
pLayerCfg1.Alpha = 200;           // 레이어 상수 알파 (200 = 약 78% 불투명)
                                  // 이 값이 작을수록 아래 레이어가 더 많이 보임
pLayerCfg1.Alpha0 = 0;

// 블렌딩 팩터 (레이어 0과 동일)
pLayerCfg1.BlendingFactor1 = LTDC_BLENDING_FACTOR1_PAxCA;
pLayerCfg1.BlendingFactor2 = LTDC_BLENDING_FACTOR2_PAxCA;

// 이미지 크기
pLayerCfg1.ImageWidth = 320;
pLayerCfg1.ImageHeight = 240;

// 기본 배경색
pLayerCfg1.Backcolor.Blue = 0;
pLayerCfg1.Backcolor.Green = 0;
pLayerCfg1.Backcolor.Red = 0;

// 레이어 설정 적용
HAL_LTDC_ConfigLayer(&LtdcHandle, &pLayerCfg1, 1);  // 레이어 1
```

### 4. 알파 블렌딩 공식

```
최종 색상 계산:

For Layer 1 (전경):
  Alpha1 = (픽셀 알파 / 255) * (상수 알파 / 255)
  Alpha1 = (255 / 255) * (200 / 255) = 0.784

For Layer 0 (배경):
  Alpha0 = (픽셀 알파 / 255) * (상수 알파 / 255)
  Alpha0 = (CLUT의 알파 / 255) * (255 / 255)

최종 픽셀 색상:
  Output_R = (Layer1_R × Alpha1) + (Layer0_R × Alpha0 × (1 - Alpha1))
  Output_G = (Layer1_G × Alpha1) + (Layer0_G × Alpha0 × (1 - Alpha1))
  Output_B = (Layer1_B × Alpha1) + (Layer0_B × Alpha0 × (1 - Alpha1))

예제:
  Layer 1: RGB(255, 0, 0), Alpha = 200
  Layer 0: RGB(0, 255, 0), Alpha = 255

  Result_R = (255 × 0.784) + (0 × 1.0 × 0.216) = 200
  Result_G = (0 × 0.784) + (255 × 1.0 × 0.216) = 55
  Result_B = (0 × 0.784) + (0 × 1.0 × 0.216) = 0

  최종 색상: RGB(200, 55, 0) - 주황색 계열
```

### 5. 동적 위치 변경 및 애니메이션

```c
int main(void)
{
  uint32_t index = 0;
  uint32_t Xpos1 = 0, Ypos1 = 0;    // 레이어 0 위치
  uint32_t Xpos2 = 160, Ypos2 = 32;  // 레이어 1 위치

  // 하드웨어 초기화
  MPU_Config();
  CPU_CACHE_Enable();
  HAL_Init();
  SystemClock_Config();
  BSP_LED_Init(LED2);

  // LCD 및 레이어 설정
  LCD_Config();

  // CLUT 로드 및 활성화
  HAL_LTDC_ConfigCLUT(&LtdcHandle, (uint32_t *)L8_320x240_CLUT, 256, 0);
  HAL_LTDC_EnableCLUT(&LtdcHandle, 0);

  // 무한 루프 - 애니메이션
  while (1)
  {
    // 전진 애니메이션 (0 → 32 프레임)
    for (index = 0; index < 32; index++)
    {
      // 새 위치 계산
      PicturesPosition(&Xpos1, &Ypos1, &Xpos2, &Ypos2, (index+1));

      // 레이어 0 위치 변경 (리로드 없이)
      // 이 함수는 내부 레지스터만 업데이트하고 즉시 적용하지 않음
      HAL_LTDC_SetWindowPosition_NoReload(&LtdcHandle, Xpos1, Ypos1, 0);

      // 레이어 1 위치 변경 (리로드 없이)
      HAL_LTDC_SetWindowPosition_NoReload(&LtdcHandle, Xpos2, Ypos2, 1);

      // 수직 블랭킹 기간에 레지스터 리로드 요청
      // 이렇게 하면 화면이 깜박이지 않고 부드럽게 업데이트됨
      ReloadFlag = 0;
      HAL_LTDC_Reload(&LtdcHandle, LTDC_RELOAD_VERTICAL_BLANKING);

      // 리로드 완료 대기
      // HAL_LTDC_ReloadEventCallback()에서 ReloadFlag = 1로 설정됨
      while(ReloadFlag == 0)
      {
        // 수직 블랭킹 기간에 업데이트가 완료될 때까지 대기
      }
    }

    HAL_Delay(500);  // 0.5초 대기

    // 후진 애니메이션 (32 → 0 프레임)
    for (index = 0; index < 32; index++)
    {
      // 위치를 반대로 계산 (레이어 순서 바꿈)
      PicturesPosition(&Xpos2, &Ypos2, &Xpos1, &Ypos1, (index+1));

      HAL_LTDC_SetWindowPosition_NoReload(&LtdcHandle, Xpos1, Ypos1, 0);
      HAL_LTDC_SetWindowPosition_NoReload(&LtdcHandle, Xpos2, Ypos2, 1);

      ReloadFlag = 0;
      HAL_LTDC_Reload(&LtdcHandle, LTDC_RELOAD_VERTICAL_BLANKING);

      while(ReloadFlag == 0)
      {
        // 리로드 완료 대기
      }
    }

    HAL_Delay(500);  // 0.5초 대기
  }
}

// 위치 계산 함수
static void PicturesPosition(uint32_t* x1, uint32_t* y1,
                             uint32_t* x2, uint32_t* y2,
                             uint32_t index)
{
  // 레이어 0 위치 (오른쪽 아래로 이동)
  *x1 = index * 5;      // X: 0 → 160 (5픽셀씩 32프레임)
  *y1 = index;          // Y: 0 → 32 (1픽셀씩 32프레임)

  // 레이어 1 위치 (왼쪽 위로 이동)
  *x2 = 160 - index * 5;  // X: 160 → 0 (반대 방향)
  *y2 = 32 - index;       // Y: 32 → 0 (반대 방향)
}

// 리로드 완료 콜백
void HAL_LTDC_ReloadEventCallback(LTDC_HandleTypeDef *hltdc)
{
  // 수직 블랭킹 기간에 레지스터가 업데이트됨
  ReloadFlag = 1;
}
```

### 6. 수직 블랭킹과 티어링 방지

```c
// 수직 블랭킹 (Vertical Blanking)
//
// LCD는 다음과 같이 동작:
// 1. Active Display: 화면에 픽셀 표시 (272 라인)
// 2. Vertical Blanking: 화면 표시 안 함 (VFP + VSYNC + VBP = 14 라인)
//
// 타이밍:
// ┌──────────────────────────────────────┐
// │ Active Display (272 lines)           │ ← 화면에 표시 중
// ├──────────────────────────────────────┤
// │ VFP (2 lines)                        │ ↑
// │ VSYNC (10 lines)                     │ │ Vertical Blanking
// │ VBP (2 lines)                        │ ↓ (14 lines)
// └──────────────────────────────────────┘
//
// 수직 블랭킹 기간에 레지스터 업데이트:
// - 장점: 티어링(화면 찢어짐) 없음
// - 단점: 업데이트 타이밍이 고정됨 (약 60Hz)

// 티어링 예제:
// 잘못된 방법 (즉시 업데이트):
HAL_LTDC_SetWindowPosition(&LtdcHandle, new_x, new_y, 0);
// → 화면 표시 중에 레지스터가 변경되어 화면이 찢어짐

// 올바른 방법 (수직 블랭킹 기간에 업데이트):
HAL_LTDC_SetWindowPosition_NoReload(&LtdcHandle, new_x, new_y, 0);
HAL_LTDC_Reload(&LtdcHandle, LTDC_RELOAD_VERTICAL_BLANKING);
// → 수직 블랭킹 기간에 레지스터가 변경되어 깔끔한 화면
```

## MPU 설정

```c
static void MPU_Config(void)
{
  MPU_Region_InitTypeDef MPU_InitStruct;

  HAL_MPU_Disable();

  // 전체 메모리 영역을 접근 불가로 설정 (기본)
  MPU_InitStruct.Enable = MPU_REGION_ENABLE;
  MPU_InitStruct.BaseAddress = 0x00;
  MPU_InitStruct.Size = MPU_REGION_SIZE_4GB;
  MPU_InitStruct.AccessPermission = MPU_REGION_NO_ACCESS;
  MPU_InitStruct.IsBufferable = MPU_ACCESS_NOT_BUFFERABLE;
  MPU_InitStruct.IsCacheable = MPU_ACCESS_NOT_CACHEABLE;
  MPU_InitStruct.IsShareable = MPU_ACCESS_SHAREABLE;
  MPU_InitStruct.Number = MPU_REGION_NUMBER0;
  MPU_InitStruct.TypeExtField = MPU_TEX_LEVEL0;
  MPU_InitStruct.SubRegionDisable = 0x87;
  MPU_InitStruct.DisableExec = MPU_INSTRUCTION_ACCESS_DISABLE;

  HAL_MPU_ConfigRegion(&MPU_InitStruct);

  // Flash 메모리에서 이미지 데이터를 읽을 수 있도록 설정
  // - 캐시 활성화로 읽기 성능 향상
  // - 쓰기 버퍼 비활성화 (Flash는 읽기 전용)

  HAL_MPU_Enable(MPU_PRIVILEGED_DEFAULT);
}
```

## 성능 분석

### 메모리 사용량
```
레이어 0 (L8):
  - 이미지 데이터: 320 × 240 × 1 = 76,800 바이트
  - CLUT 데이터: 256 × 4 = 1,024 바이트
  - 합계: 77,824 바이트

레이어 1 (RGB565):
  - 이미지 데이터: 320 × 240 × 2 = 153,600 바이트

전체 메모리 사용량: 231,424 바이트 (약 226 KB)
```

### 대역폭 계산
```
LCD 해상도: 480 × 272 픽셀
프레임 레이트: 60 Hz
픽셀 클럭: 9.63 MHz

픽셀당 필요한 클럭:
  Total = (HSync + HBP + Width + HFP) × (VSync + VBP + Height + VFP)
  Total = 566 × 286 = 161,876 클럭

실제 프레임 레이트:
  FPS = 9,630,000 / 161,876 = 59.5 Hz

메모리 대역폭:
  레이어 0: 320 × 240 × 1 × 60 = 4.608 MB/s
  레이어 1: 320 × 240 × 2 × 60 = 9.216 MB/s
  합계: 13.824 MB/s

AXI 버스 대역폭: 200 MHz × 8 바이트 = 1600 MB/s
LTDC 사용률: 13.824 / 1600 = 0.86% (매우 낮음)
```

### 프레임 업데이트 시간
```
리로드 지연:
  - 수직 블랭킹 대기: 최대 16.8ms (1 / 59.5 Hz)
  - 평균 대기: 8.4ms

애니메이션 프레임:
  - 32 프레임 × 16.8ms = 537.6ms
  - HAL_Delay(500) 추가
  - 전체 사이클: 약 1초
```

## 픽셀 포맷 비교

### L8 (8비트 인덱스 + CLUT)
```
장점:
  - 메모리 효율적 (1바이트/픽셀)
  - 팔레트 기반 색상 변경 용이
  - 레트로 게임, 아이콘에 적합

단점:
  - 256색 제한
  - CLUT 로딩 오버헤드
  - 실사 이미지에 부적합

메모리 사용량 (320×240):
  이미지: 76,800 바이트
  CLUT: 1,024 바이트
  합계: 77,824 바이트
```

### RGB565 (직접 색상)
```
장점:
  - 65,536색 표현 가능
  - CLUT 불필요
  - 사진, 그라디언트에 적합

단점:
  - 메모리 사용량 2배 (2바이트/픽셀)
  - 색상 변경이 어려움

메모리 사용량 (320×240):
  이미지: 153,600 바이트
```

### RGB888 (완전한 색상)
```
장점:
  - 16,777,216색 (True Color)
  - 최고 화질

단점:
  - 메모리 사용량 3배 (3바이트/픽셀)
  - 대역폭 증가
  - LTDC는 내부적으로 RGB888 사용

메모리 사용량 (320×240):
  이미지: 230,400 바이트
```

### ARGB8888 (알파 채널 포함)
```
장점:
  - 픽셀별 투명도 제어
  - 부드러운 블렌딩
  - 안티앨리어싱 지원

단점:
  - 메모리 사용량 4배 (4바이트/픽셀)
  - 최대 대역폭

메모리 사용량 (320×240):
  이미지: 307,200 바이트
```

## 고급 사용 예제

### 1. 풀 스크린 프레임버퍼 (SDRAM 사용)

```c
// SDRAM에 프레임버퍼 할당
#define LAYER0_ADDRESS  (0xD0000000)
#define LAYER1_ADDRESS  (0xD0100000)

void LCD_FullScreen_Init(void)
{
  LTDC_LayerCfgTypeDef pLayerCfg;

  // SDRAM 초기화
  BSP_SDRAM_Init(0);

  // 레이어 0: RGB888, 480×272
  pLayerCfg.WindowX0 = 0;
  pLayerCfg.WindowX1 = 480;
  pLayerCfg.WindowY0 = 0;
  pLayerCfg.WindowY1 = 272;
  pLayerCfg.PixelFormat = LTDC_PIXEL_FORMAT_RGB888;
  pLayerCfg.FBStartAdress = LAYER0_ADDRESS;
  pLayerCfg.Alpha = 255;
  pLayerCfg.BlendingFactor1 = LTDC_BLENDING_FACTOR1_CA;
  pLayerCfg.BlendingFactor2 = LTDC_BLENDING_FACTOR2_CA;
  pLayerCfg.ImageWidth = 480;
  pLayerCfg.ImageHeight = 272;

  HAL_LTDC_ConfigLayer(&LtdcHandle, &pLayerCfg, 0);

  // 레이어 1: ARGB8888, 480×272 (투명도 지원)
  pLayerCfg.PixelFormat = LTDC_PIXEL_FORMAT_ARGB8888;
  pLayerCfg.FBStartAdress = LAYER1_ADDRESS;
  pLayerCfg.Alpha = 128;  // 반투명
  pLayerCfg.BlendingFactor1 = LTDC_BLENDING_FACTOR1_PAxCA;
  pLayerCfg.BlendingFactor2 = LTDC_BLENDING_FACTOR2_PAxCA;

  HAL_LTDC_ConfigLayer(&LtdcHandle, &pLayerCfg, 1);
}

// 픽셀 그리기 함수
void LCD_DrawPixel(uint16_t x, uint16_t y, uint32_t color)
{
  uint32_t *framebuffer = (uint32_t *)LAYER0_ADDRESS;
  framebuffer[y * 480 + x] = color;

  // D-Cache 관리 (중요!)
  SCB_CleanDCache_by_Addr((uint32_t *)&framebuffer[y * 480 + x], 4);
}

// 사각형 그리기
void LCD_FillRect(uint16_t x, uint16_t y, uint16_t width, uint16_t height, uint32_t color)
{
  uint32_t *framebuffer = (uint32_t *)LAYER0_ADDRESS;

  for (uint16_t j = 0; j < height; j++)
  {
    for (uint16_t i = 0; i < width; i++)
    {
      framebuffer[(y + j) * 480 + (x + i)] = color;
    }
  }

  // D-Cache 플러시
  uint32_t start_addr = (uint32_t)&framebuffer[y * 480 + x];
  uint32_t cache_size = height * 480 * 4;
  SCB_CleanDCache_by_Addr((uint32_t *)start_addr, cache_size);
}
```

### 2. 더블 버퍼링 (깜박임 방지)

```c
#define BUFFER0_ADDRESS  (0xD0000000)
#define BUFFER1_ADDRESS  (0xD0200000)

uint8_t active_buffer = 0;

void LCD_DoubleBuffer_Init(void)
{
  LTDC_LayerCfgTypeDef pLayerCfg;

  // 초기 버퍼: BUFFER0
  pLayerCfg.FBStartAdress = BUFFER0_ADDRESS;
  pLayerCfg.PixelFormat = LTDC_PIXEL_FORMAT_RGB888;
  pLayerCfg.ImageWidth = 480;
  pLayerCfg.ImageHeight = 272;
  pLayerCfg.WindowX0 = 0;
  pLayerCfg.WindowX1 = 480;
  pLayerCfg.WindowY0 = 0;
  pLayerCfg.WindowY1 = 272;
  pLayerCfg.Alpha = 255;
  pLayerCfg.BlendingFactor1 = LTDC_BLENDING_FACTOR1_CA;
  pLayerCfg.BlendingFactor2 = LTDC_BLENDING_FACTOR2_CA;

  HAL_LTDC_ConfigLayer(&LtdcHandle, &pLayerCfg, 0);
}

// 버퍼 스왑 (수직 블랭킹에서)
void LCD_SwapBuffers(void)
{
  uint32_t new_address;

  // 다음 버퍼 주소 계산
  if (active_buffer == 0)
  {
    new_address = BUFFER1_ADDRESS;
    active_buffer = 1;
  }
  else
  {
    new_address = BUFFER0_ADDRESS;
    active_buffer = 0;
  }

  // 프레임버퍼 주소 변경 (수직 블랭킹에서)
  HAL_LTDC_SetAddress_NoReload(&LtdcHandle, new_address, 0);

  ReloadFlag = 0;
  HAL_LTDC_Reload(&LtdcHandle, LTDC_RELOAD_VERTICAL_BLANKING);

  while (ReloadFlag == 0);
}

// 백 버퍼에 그리기
void LCD_DrawToBackBuffer(void)
{
  uint32_t *back_buffer;

  // 현재 표시되지 않는 버퍼 선택
  if (active_buffer == 0)
    back_buffer = (uint32_t *)BUFFER1_ADDRESS;
  else
    back_buffer = (uint32_t *)BUFFER0_ADDRESS;

  // 백 버퍼에 그리기
  for (int i = 0; i < 480 * 272; i++)
  {
    back_buffer[i] = 0xFF0000;  // 빨간색
  }

  // D-Cache 플러시
  SCB_CleanDCache_by_Addr((uint32_t *)back_buffer, 480 * 272 * 4);

  // 버퍼 스왑 (화면에 표시)
  LCD_SwapBuffers();
}
```

### 3. 레이어 블렌딩 모드 변경

```c
// 다양한 블렌딩 모드 예제

void LCD_SetBlendingMode_Normal(uint8_t layer)
{
  LTDC_LayerCfgTypeDef pLayerCfg;

  // 일반 블렌딩: 픽셀 알파 × 상수 알파
  pLayerCfg.BlendingFactor1 = LTDC_BLENDING_FACTOR1_PAxCA;
  pLayerCfg.BlendingFactor2 = LTDC_BLENDING_FACTOR2_PAxCA;

  HAL_LTDC_ConfigLayer(&LtdcHandle, &pLayerCfg, layer);
}

void LCD_SetBlendingMode_ConstantAlpha(uint8_t layer, uint8_t alpha)
{
  LTDC_LayerCfgTypeDef pLayerCfg;

  // 상수 알파만 사용 (픽셀 알파 무시)
  pLayerCfg.Alpha = alpha;
  pLayerCfg.BlendingFactor1 = LTDC_BLENDING_FACTOR1_CA;
  pLayerCfg.BlendingFactor2 = LTDC_BLENDING_FACTOR2_CA;

  HAL_LTDC_ConfigLayer(&LtdcHandle, &pLayerCfg, layer);
}

// 알파 페이드 인/아웃 애니메이션
void LCD_FadeIn(uint8_t layer, uint32_t duration_ms)
{
  uint32_t steps = 256;
  uint32_t delay = duration_ms / steps;

  for (uint16_t alpha = 0; alpha < 256; alpha++)
  {
    HAL_LTDC_SetAlpha_NoReload(&LtdcHandle, alpha, layer);

    ReloadFlag = 0;
    HAL_LTDC_Reload(&LtdcHandle, LTDC_RELOAD_VERTICAL_BLANKING);
    while (ReloadFlag == 0);

    HAL_Delay(delay);
  }
}

void LCD_FadeOut(uint8_t layer, uint32_t duration_ms)
{
  uint32_t steps = 256;
  uint32_t delay = duration_ms / steps;

  for (int16_t alpha = 255; alpha >= 0; alpha--)
  {
    HAL_LTDC_SetAlpha_NoReload(&LtdcHandle, alpha, layer);

    ReloadFlag = 0;
    HAL_LTDC_Reload(&LtdcHandle, LTDC_RELOAD_VERTICAL_BLANKING);
    while (ReloadFlag == 0);

    HAL_Delay(delay);
  }
}
```

### 4. 윈도우 크기 조절 (줌 효과)

```c
void LCD_ZoomLayer(uint8_t layer, uint16_t center_x, uint16_t center_y,
                   float scale, uint16_t img_width, uint16_t img_height)
{
  uint16_t new_width = (uint16_t)(img_width * scale);
  uint16_t new_height = (uint16_t)(img_height * scale);

  // 중심점을 기준으로 확대/축소
  uint16_t x0 = center_x - (new_width / 2);
  uint16_t y0 = center_y - (new_height / 2);
  uint16_t x1 = x0 + new_width;
  uint16_t y1 = y0 + new_height;

  // 화면 경계 체크
  if (x0 < 0) x0 = 0;
  if (y0 < 0) y0 = 0;
  if (x1 > 480) x1 = 480;
  if (y1 > 272) y1 = 272;

  // 윈도우 위치 및 크기 변경
  HAL_LTDC_SetWindowSize_NoReload(&LtdcHandle, new_width, new_height, layer);
  HAL_LTDC_SetWindowPosition_NoReload(&LtdcHandle, x0, y0, layer);

  ReloadFlag = 0;
  HAL_LTDC_Reload(&LtdcHandle, LTDC_RELOAD_VERTICAL_BLANKING);
  while (ReloadFlag == 0);
}

// 줌 인 애니메이션
void LCD_AnimateZoomIn(uint8_t layer, uint16_t img_width, uint16_t img_height)
{
  float scale;

  for (uint8_t i = 10; i <= 100; i += 5)
  {
    scale = i / 100.0f;  // 0.1 → 1.0
    LCD_ZoomLayer(layer, 240, 136, scale, img_width, img_height);
    HAL_Delay(50);
  }
}
```

### 5. CLUT 동적 변경 (팔레트 애니메이션)

```c
// 색상 팔레트 회전 (사이클링 효과)
void LCD_RotateCLUT(uint8_t layer)
{
  uint32_t temp_clut[256];
  uint32_t current_clut[256];

  // 현재 CLUT 읽기
  HAL_LTDC_ConfigCLUT(&LtdcHandle, current_clut, 256, layer);

  // 팔레트 회전 (한 칸씩 이동)
  temp_clut[0] = current_clut[255];
  for (uint16_t i = 1; i < 256; i++)
  {
    temp_clut[i] = current_clut[i - 1];
  }

  // 새 CLUT 적용
  HAL_LTDC_ConfigCLUT(&LtdcHandle, temp_clut, 256, layer);
}

// 그레이스케일 → 컬러 애니메이션
void LCD_AnimateCLUT_GrayToColor(uint8_t layer, uint32_t *color_clut)
{
  uint32_t gray_clut[256];
  uint32_t animated_clut[256];

  // 그레이스케일 팔레트 생성
  for (uint16_t i = 0; i < 256; i++)
  {
    gray_clut[i] = (i << 16) | (i << 8) | i;  // RGB(i, i, i)
  }

  // 프레임별로 보간
  for (uint8_t step = 0; step <= 100; step += 5)
  {
    float t = step / 100.0f;

    for (uint16_t i = 0; i < 256; i++)
    {
      uint8_t r_gray = (gray_clut[i] >> 16) & 0xFF;
      uint8_t g_gray = (gray_clut[i] >> 8) & 0xFF;
      uint8_t b_gray = gray_clut[i] & 0xFF;

      uint8_t r_color = (color_clut[i] >> 16) & 0xFF;
      uint8_t g_color = (color_clut[i] >> 8) & 0xFF;
      uint8_t b_color = color_clut[i] & 0xFF;

      uint8_t r = r_gray + (r_color - r_gray) * t;
      uint8_t g = g_gray + (g_color - g_gray) * t;
      uint8_t b = b_gray + (b_color - b_gray) * t;

      animated_clut[i] = (r << 16) | (g << 8) | b;
    }

    HAL_LTDC_ConfigCLUT(&LtdcHandle, animated_clut, 256, layer);
    HAL_Delay(50);
  }
}
```

## 트러블슈팅

### 1. 화면에 아무것도 표시되지 않음

**원인 및 해결책:**

```c
// 문제 1: LCD 백라이트가 꺼져 있음
// 해결: BSP 함수로 백라이트 켜기
BSP_LCD_Init(0, LCD_ORIENTATION_LANDSCAPE);
// 또는 GPIO 직접 제어
HAL_GPIO_WritePin(LCD_BL_CTRL_GPIO_Port, LCD_BL_CTRL_Pin, GPIO_PIN_SET);

// 문제 2: LTDC 클럭이 설정되지 않음
// 해결: PLL3 설정 확인
periph_clk_init_struct.PeriphClockSelection = RCC_PERIPHCLK_LTDC;
periph_clk_init_struct.PLL3.PLL3R = 83;  // LCD 클럭 분주비
HAL_RCCEx_PeriphCLKConfig(&periph_clk_init_struct);

// 문제 3: 레이어가 비활성화됨
// 해결: 레이어 활성화
BSP_LCD_SetLayerVisible(0, 0, ENABLE);  // 레이어 0 활성화
BSP_LCD_SetLayerVisible(0, 1, ENABLE);  // 레이어 1 활성화
```

### 2. 화면이 깜박이거나 찢어짐 (티어링)

**원인:** 수직 블랭킹 기간이 아닌 시점에 레지스터 업데이트

```c
// 잘못된 방법:
HAL_LTDC_SetWindowPosition(&LtdcHandle, x, y, layer);  // 즉시 적용 → 티어링 발생

// 올바른 방법:
HAL_LTDC_SetWindowPosition_NoReload(&LtdcHandle, x, y, layer);
ReloadFlag = 0;
HAL_LTDC_Reload(&LtdcHandle, LTDC_RELOAD_VERTICAL_BLANKING);
while (ReloadFlag == 0);  // 수직 블랭킹 대기

// 또는 인터럽트 사용:
void HAL_LTDC_ReloadEventCallback(LTDC_HandleTypeDef *hltdc)
{
  // 수직 블랭킹에서 자동 호출됨
  ReloadFlag = 1;
}
```

### 3. 색상이 이상하게 표시됨

**원인 및 해결책:**

```c
// 문제 1: 픽셀 포맷 불일치
// 예: RGB565 데이터를 ARGB8888로 설정
// 해결: 픽셀 포맷 확인
pLayerCfg.PixelFormat = LTDC_PIXEL_FORMAT_RGB565;  // 데이터 포맷과 일치해야 함

// 문제 2: 바이트 순서 (엔디안) 문제
// RGB565: 리틀 엔디안으로 저장됨
// 예: 빨간색 (0xF800) → 메모리: [0x00, 0xF8]
uint16_t red_color = 0xF800;  // 올바름
uint16_t red_color_wrong = 0x00F8;  // 잘못됨

// 문제 3: CLUT이 로드되지 않음 (L8 포맷)
// 해결: CLUT 로드 확인
HAL_LTDC_ConfigCLUT(&LtdcHandle, (uint32_t *)clut_data, 256, layer);
HAL_LTDC_EnableCLUT(&LtdcHandle, layer);
```

### 4. 이미지가 왜곡됨

**원인:** 이미지 크기와 윈도우 크기 불일치

```c
// 문제: ImageWidth/Height와 Window 크기가 다름
pLayerCfg.ImageWidth = 320;   // 실제 이미지 크기
pLayerCfg.ImageHeight = 240;
pLayerCfg.WindowX0 = 0;
pLayerCfg.WindowX1 = 480;     // 윈도우가 더 큼 → 이미지가 늘어남

// 해결 1: 윈도우 크기를 이미지와 동일하게
pLayerCfg.WindowX1 = pLayerCfg.WindowX0 + pLayerCfg.ImageWidth;
pLayerCfg.WindowY1 = pLayerCfg.WindowY0 + pLayerCfg.ImageHeight;

// 해결 2: 중앙 정렬
uint16_t x_offset = (480 - 320) / 2;  // 80
uint16_t y_offset = (272 - 240) / 2;  // 16
pLayerCfg.WindowX0 = x_offset;
pLayerCfg.WindowX1 = x_offset + 320;
pLayerCfg.WindowY0 = y_offset;
pLayerCfg.WindowY1 = y_offset + 240;
```

### 5. 알파 블렌딩이 작동하지 않음

**원인 및 해결책:**

```c
// 문제 1: 블렌딩 팩터가 잘못 설정됨
// 해결: 올바른 블렌딩 팩터 사용
pLayerCfg.BlendingFactor1 = LTDC_BLENDING_FACTOR1_PAxCA;  // 전경
pLayerCfg.BlendingFactor2 = LTDC_BLENDING_FACTOR2_PAxCA;  // 배경

// 문제 2: 알파값이 255 (완전 불투명)
// 해결: 알파값 조절
pLayerCfg.Alpha = 128;  // 50% 투명

// 문제 3: 레이어 순서가 잘못됨
// LTDC는 레이어 0이 아래, 레이어 1이 위에 표시됨
// 레이어 1의 알파를 조절하여 레이어 0이 보이게 함
```

### 6. D-Cache 관련 문제 (SDRAM 프레임버퍼 사용 시)

**원인:** CPU 캐시와 DMA 간 데이터 불일치

```c
// 문제: 프레임버퍼에 쓴 데이터가 화면에 표시되지 않음
// 원인: D-Cache에만 쓰여지고 SDRAM에는 쓰여지지 않음

// 해결 1: D-Cache 클린 (쓰기 후)
uint32_t *framebuffer = (uint32_t *)0xD0000000;
framebuffer[100] = 0xFF0000;  // 빨간색 픽셀 쓰기
SCB_CleanDCache_by_Addr((uint32_t *)&framebuffer[100], 4);  // 캐시 → SDRAM

// 해결 2: 전체 영역 클린
SCB_CleanDCache_by_Addr((uint32_t *)framebuffer, 480 * 272 * 4);

// 해결 3: MPU로 캐시 비활성화 (SDRAM 영역)
MPU_InitStruct.Enable = MPU_REGION_ENABLE;
MPU_InitStruct.BaseAddress = 0xD0000000;
MPU_InitStruct.Size = MPU_REGION_SIZE_8MB;
MPU_InitStruct.IsCacheable = MPU_ACCESS_NOT_CACHEABLE;  // 캐시 비활성화
// 주의: 성능 저하!

// 해결 4: Write-Through 캐시 사용
// MPU에서 캐시를 Write-Through로 설정하면 자동으로 SDRAM에 쓰여짐
// 단, 읽기 성능은 Write-Back보다 낮음
```

### 7. 메모리 부족 오류

**원인:** Flash 또는 RAM 부족

```c
// 문제: 큰 이미지 데이터가 Flash에 맞지 않음
// 해결 1: 이미지 압축
// - JPEG 사용 (하드웨어 디코더 지원)
// - RLE 압축 (단순한 이미지에 효과적)

// 해결 2: 이미지를 QSPI Flash에 저장
#define IMAGE_ADDRESS  0x90000000  // QSPI 메모리 맵 주소
pLayerCfg.FBStartAdress = IMAGE_ADDRESS;

// 해결 3: 이미지 크기 줄이기
// - 480×272 대신 작은 이미지 사용 (320×240)
// - 픽셀 포맷 변경: RGB888 → RGB565 (50% 절약)

// 해결 4: SDRAM에 이미지 로드
// QSPI Flash → SDRAM 복사 후 사용
uint32_t *qspi_image = (uint32_t *)0x90000000;
uint32_t *sdram_buffer = (uint32_t *)0xD0000000;
memcpy(sdram_buffer, qspi_image, 480 * 272 * 3);
pLayerCfg.FBStartAdress = (uint32_t)sdram_buffer;
```

## 최적화 팁

### 1. 메모리 최적화

```c
// 팁 1: 작은 이미지는 L8 + CLUT 사용
// RGB565: 320×240×2 = 153,600 바이트
// L8: 320×240×1 + 256×4 = 77,824 바이트 (약 50% 절약)

// 팁 2: 정적 이미지는 Flash에 저장
// - const로 선언하여 Flash에 배치
// - 실행 시간에 RAM으로 복사 불필요
const uint16_t image_data[] __attribute__((section(".rodata"))) = { ... };

// 팁 3: DMA2D로 색상 변환
// RGB888 이미지를 런타임에 RGB565로 변환
// - Flash: RGB888 저장
// - SDRAM: RGB565로 변환 후 사용
// - 메모리 절약 + 유연성

// 팁 4: 이미지 공유
// 여러 레이어에서 같은 이미지 사용 가능
pLayerCfg0.FBStartAdress = (uint32_t)&shared_image;
pLayerCfg1.FBStartAdress = (uint32_t)&shared_image;
// 위치만 다르게 설정
```

### 2. 성능 최적화

```c
// 팁 1: 불필요한 리로드 방지
// 여러 설정을 한 번에 변경 후 리로드
HAL_LTDC_SetWindowPosition_NoReload(&LtdcHandle, x, y, 0);
HAL_LTDC_SetAlpha_NoReload(&LtdcHandle, alpha, 0);
HAL_LTDC_Reload(&LtdcHandle, LTDC_RELOAD_VERTICAL_BLANKING);  // 한 번만

// 팁 2: 작은 이미지 사용
// 480×272 대신 320×240 사용 시:
// - 메모리 대역폭: 13.8 MB/s → 9.2 MB/s (33% 감소)
// - 메모리 사용량: 50% 감소

// 팁 3: DMA2D로 복사/변환
// CPU 대신 DMA2D 사용 시 CPU 부하 감소
// 예: memcpy() 대신 DMA2D_MemoryToMemory()

// 팁 4: 픽셀 클럭 최적화
// LCD 스펙의 최소 클럭 사용
// - 소비 전력 감소
// - EMI 감소
```

### 3. 전력 최적화

```c
// 팁 1: 사용하지 않는 레이어 비활성화
BSP_LCD_SetLayerVisible(0, 1, DISABLE);  // 레이어 1 끄기

// 팁 2: 백라이트 밝기 조절
// PWM으로 백라이트 제어
TIM_HandleTypeDef htim;
__HAL_TIM_SET_COMPARE(&htim, TIM_CHANNEL_1, 128);  // 50% 밝기

// 팁 3: 프레임 레이트 감소
// 60Hz → 30Hz
// 전력 소비 약 50% 감소 (애니메이션이 없는 경우)

// 팁 4: 아이들 모드
// 화면 업데이트가 없을 때 LTDC 일시 정지
HAL_LTDC_ProgramLineEvent(&LtdcHandle, 0);  // 라인 인터럽트 비활성화
```

## 참고 자료

### STM32H7 문서
- **RM0399**: STM32H745/755 Reference Manual
  - 17장: LTDC (LCD-TFT Display Controller)
  - 픽셀 포맷, 블렌딩 모드, 타이밍 파라미터
- **DS12110**: STM32H745xI/G Datasheet
  - 전기적 특성, 핀 배치
- **AN4861**: LCD-TFT display controller (LTDC) on STM32 MCUs
  - LTDC 사용법, 최적화 팁

### 보드 문서
- **UM2411**: STM32H745I-DISCO User Manual
  - 보드 회로도, LCD 연결
- **RK043FN48H**: LCD 패널 데이터시트
  - 타이밍 파라미터, 전기적 특성

### HAL 드라이버
- `stm32h7xx_hal_ltdc.c/h`: LTDC HAL 드라이버
- `stm32h7xx_hal_dma2d.c/h`: DMA2D HAL 드라이버 (그래픽 가속)
- `STM32H745I_Discovery_lcd.c/h`: BSP LCD 드라이버

### 예제 코드
- `STM32Cube_FW_H7/Projects/STM32H745I-DISCO/Examples/LTDC/`
  - LTDC_Display_1Layer
  - LTDC_Display_2Layers
  - LTDC_ColorKeying
  - LTDC_PicturesFromSDCard

### 관련 애플리케이션 노트
- **AN4839**: Overview of Chrom-ART graphics accelerator (DMA2D)
- **AN4943**: MPU usage on STM32H7
- **AN5179**: Managing the D-Cache on STM32H7
