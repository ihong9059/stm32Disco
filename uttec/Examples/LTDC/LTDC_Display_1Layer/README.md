# LTDC_Display_1Layer - 단일 레이어 LCD 디스플레이

## 개요

이 예제는 STM32H745의 LTDC (LCD-TFT Display Controller)를 사용하여 단일 레이어로 LCD 화면에 이미지를 표시하는 방법을 보여줍니다.

## 하드웨어 요구사항

- **STM32H745I-DISCO 보드**
- **내장 LCD**: 480x272 해상도, RGB565 포맷

## 주요 기능

### LTDC (LCD-TFT Display Controller)
- **단일 레이어 출력**
- **RGB565 픽셀 포맷** (16비트 컬러)
- **프레임버퍼**: SDRAM 사용
- **하드웨어 가속 디스플레이**

### 디스플레이 사양
- 해상도: 480 x 272 픽셀
- 컬러 포맷: RGB565 (5비트 R, 6비트 G, 5비트 B)
- 프레임버퍼 크기: 480 × 272 × 2 = 261,120 bytes

## 동작 원리

### 시스템 구성
```
┌──────────┐         ┌──────────┐         ┌──────────┐
│  CM7     │────────►│  LTDC    │────────►│   LCD    │
│  Core    │         │Controller│         │  Panel   │
└──────────┘         └──────────┘         └──────────┘
      │                    │
      │                    │
      ▼                    ▼
┌──────────┐         ┌──────────┐
│  SDRAM   │◄────────│ DMA      │
│(FrameBuf)│         │          │
└──────────┘         └──────────┘
```

### 데이터 흐름
1. **프레임버퍼 준비**: CPU가 SDRAM에 이미지 데이터 작성
2. **LTDC DMA**: LTDC가 DMA를 통해 프레임버퍼 읽기
3. **픽셀 출력**: LTDC가 LCD 패널로 픽셀 데이터 전송
4. **자동 갱신**: LTDC가 수평/수직 동기화 신호와 함께 자동으로 화면 갱신

## 코드 구조

### 1. LTDC 초기화
```c
void LTDC_Init(void)
{
  LTDC_HandleTypeDef hltdc;

  // LTDC 클럭 활성화
  __HAL_RCC_LTDC_CLK_ENABLE();

  // 타이밍 파라미터 설정 (480x272)
  hltdc.Init.HorizontalSync = 40;      // HSYNC 폭
  hltdc.Init.VerticalSync = 9;         // VSYNC 폭
  hltdc.Init.AccumulatedHBP = 53;      // 수평 백 포치
  hltdc.Init.AccumulatedVBP = 11;      // 수직 백 포치
  hltdc.Init.AccumulatedActiveW = 533; // 활성 폭 (480 + 53)
  hltdc.Init.AccumulatedActiveH = 283; // 활성 높이 (272 + 11)
  hltdc.Init.TotalWidth = 565;         // 총 폭
  hltdc.Init.TotalHeigh = 285;         // 총 높이

  // 배경색 설정 (검정)
  hltdc.Init.Backcolor.Blue = 0;
  hltdc.Init.Backcolor.Green = 0;
  hltdc.Init.Backcolor.Red = 0;

  // LTDC 초기화
  HAL_LTDC_Init(&hltdc);
}
```

### 2. 레이어 설정
```c
void LTDC_Layer_Init(void)
{
  LTDC_LayerCfgTypeDef pLayerCfg;

  // 레이어 윈도우 설정
  pLayerCfg.WindowX0 = 0;         // X 시작
  pLayerCfg.WindowX1 = 480;       // X 끝
  pLayerCfg.WindowY0 = 0;         // Y 시작
  pLayerCfg.WindowY1 = 272;       // Y 끝

  // 픽셀 포맷
  pLayerCfg.PixelFormat = LTDC_PIXEL_FORMAT_RGB565;

  // 알파 블렌딩 (불투명)
  pLayerCfg.Alpha = 255;
  pLayerCfg.Alpha0 = 0;

  // 블렌딩 팩터
  pLayerCfg.BlendingFactor1 = LTDC_BLENDING_FACTOR1_CA;
  pLayerCfg.BlendingFactor2 = LTDC_BLENDING_FACTOR2_CA;

  // 프레임버퍼 주소 (SDRAM)
  pLayerCfg.FBStartAdress = LCD_FRAME_BUFFER;

  // 라인 및 이미지 길이
  pLayerCfg.ImageWidth = 480;
  pLayerCfg.ImageHeight = 272;

  // 백그라운드 컬러
  pLayerCfg.Backcolor.Blue = 0;
  pLayerCfg.Backcolor.Green = 0;
  pLayerCfg.Backcolor.Red = 0;

  // 레이어 0 설정
  HAL_LTDC_ConfigLayer(&hltdc, &pLayerCfg, 0);
}
```

### 3. 픽셀 그리기
```c
// 프레임버퍼 주소
#define LCD_FRAME_BUFFER  0xD0000000  // SDRAM 주소

// RGB565로 픽셀 그리기
void DrawPixel(uint16_t x, uint16_t y, uint16_t color)
{
  uint16_t *fb = (uint16_t*)LCD_FRAME_BUFFER;
  fb[y * 480 + x] = color;
}

// RGB888을 RGB565로 변환
uint16_t RGB888_to_RGB565(uint8_t r, uint8_t g, uint8_t b)
{
  return ((r & 0xF8) << 8) | ((g & 0xFC) << 3) | (b >> 3);
}

// 사각형 그리기
void FillRect(uint16_t x, uint16_t y, uint16_t w, uint16_t h, uint16_t color)
{
  for (uint16_t i = 0; i < h; i++)
  {
    for (uint16_t j = 0; j < w; j++)
    {
      DrawPixel(x + j, y + i, color);
    }
  }
}
```

### 4. 이미지 표시
```c
// RGB565 이미지 표시
void Display_Image(const uint16_t *image_data)
{
  uint16_t *fb = (uint16_t*)LCD_FRAME_BUFFER;

  // 전체 화면 복사
  memcpy(fb, image_data, 480 * 272 * 2);

  // 캐시 클린 (SDRAM은 캐시 가능 영역)
  SCB_CleanDCache_by_Addr((uint32_t*)fb, 480 * 272 * 2);
}
```

## 메모리 맵

### SDRAM 레이아웃
```
0xD0000000  ┌─────────────────────────┐
            │                         │
            │  Layer 0 Frame Buffer   │
            │  (480 x 272 x 2 bytes)  │
            │  = 261,120 bytes        │
            │                         │
0xD003FC00  ├─────────────────────────┤
            │                         │
            │  Free SDRAM             │
            │                         │
0xD0800000  └─────────────────────────┘
            (8MB SDRAM)
```

## RGB565 컬러 포맷

### 비트 구성
```
15 14 13 12 11 | 10 9 8 7 6 5 | 4 3 2 1 0
  R4 R3 R2 R1 R0 | G5 G4 G3 G2 G1 G0 | B4 B3 B2 B1 B0
     Red (5비트)  |    Green (6비트)   |   Blue (5비트)
```

### 주요 색상 값
```c
#define COLOR_BLACK      0x0000  // 검정
#define COLOR_WHITE      0xFFFF  // 흰색
#define COLOR_RED        0xF800  // 빨강
#define COLOR_GREEN      0x07E0  // 초록
#define COLOR_BLUE       0x001F  // 파랑
#define COLOR_YELLOW     0xFFE0  // 노랑
#define COLOR_CYAN       0x07FF  // 청록
#define COLOR_MAGENTA    0xF81F  // 자홍
```

## 예제 애플리케이션

### 메인 코드
```c
int main(void)
{
  // 시스템 초기화
  HAL_Init();
  SystemClock_Config();

  // SDRAM 초기화
  BSP_SDRAM_Init();

  // LTDC 초기화
  LTDC_Init();
  LTDC_Layer_Init();

  // 화면 지우기 (검정)
  FillRect(0, 0, 480, 272, COLOR_BLACK);

  // 빨간 사각형 그리기
  FillRect(50, 50, 100, 100, COLOR_RED);

  // 초록 사각형 그리기
  FillRect(200, 50, 100, 100, COLOR_GREEN);

  // 파란 사각형 그리기
  FillRect(350, 50, 100, 100, COLOR_BLUE);

  // 무한 루프
  while (1)
  {
    // 애니메이션 또는 업데이트
  }
}
```

## 성능 최적화

### DMA 사용
```c
// DMA2D를 사용한 고속 사각형 채우기
void FillRect_DMA2D(uint16_t x, uint16_t y, uint16_t w, uint16_t h, uint16_t color)
{
  DMA2D_HandleTypeDef hdma2d;

  hdma2d.Init.Mode = DMA2D_R2M;  // Register to Memory
  hdma2d.Init.ColorMode = DMA2D_OUTPUT_RGB565;
  hdma2d.Init.OutputOffset = 480 - w;

  HAL_DMA2D_Init(&hdma2d);

  uint32_t dest_addr = LCD_FRAME_BUFFER + 2 * (y * 480 + x);

  HAL_DMA2D_Start(&hdma2d, color, dest_addr, w, h);
  HAL_DMA2D_PollForTransfer(&hdma2d, 100);
}
```

### 더블 버퍼링
```c
#define BUFFER_0  0xD0000000
#define BUFFER_1  0xD0040000

uint8_t active_buffer = 0;

void SwapBuffers(void)
{
  if (active_buffer == 0)
  {
    HAL_LTDC_SetAddress(&hltdc, BUFFER_1, 0);
    active_buffer = 1;
  }
  else
  {
    HAL_LTDC_SetAddress(&hltdc, BUFFER_0, 0);
    active_buffer = 0;
  }
}
```

## 주의사항

### 캐시 일관성
SDRAM은 캐시 가능 영역이므로, 프레임버퍼 업데이트 후 캐시 클린 필요:
```c
// 변경된 영역만 클린
SCB_CleanDCache_by_Addr((uint32_t*)address, size);

// 또는 전체 D-Cache 클린
SCB_CleanDCache();
```

### LTDC 클럭 설정
LTDC는 전용 PLL을 사용:
```c
// PLL3 설정 (LTDC 클럭)
RCC_PeriphCLKInitTypeDef PeriphClkInitStruct;
PeriphClkInitStruct.PeriphClockSelection = RCC_PERIPHCLK_LTDC;
PeriphClkInitStruct.PLL3.PLL3M = 5;
PeriphClkInitStruct.PLL3.PLL3N = 160;
PeriphClkInitStruct.PLL3.PLL3P = 2;
PeriphClkInitStruct.PLL3.PLL3Q = 2;
PeriphClkInitStruct.PLL3.PLL3R = 48;  // 27MHz for LTDC
HAL_RCCEx_PeriphCLKConfig(&PeriphClkInitStruct);
```

## 트러블슈팅

### 화면에 아무것도 표시되지 않는 경우
1. **LTDC 클럭 확인**: PLL3 설정 확인
2. **타이밍 파라미터**: LCD 사양과 일치하는지 확인
3. **프레임버퍼 주소**: SDRAM이 올바르게 초기화되었는지 확인
4. **GPIO 설정**: LTDC 핀이 올바르게 설정되었는지 확인

### 화면이 깜빡이는 경우
1. **LTDC 동기화 신호 확인**
2. **프레임버퍼 업데이트 시점**: VSYNC 인터럽트 사용 권장

### 색상이 이상한 경우
1. **픽셀 포맷 확인**: RGB565 설정 확인
2. **RGB 순서**: 일부 디스플레이는 BGR 순서 사용

## 확장 기능

### 2레이어로 업그레이드
LTDC_Display_2Layers 예제 참조

### 터치스크린 통합
BSP 예제의 터치스크린 코드 참조

### 그래픽 라이브러리 사용
- STemWin (SEGGER emWin)
- TouchGFX

## 참고 자료

- RM0399: STM32H745 Reference Manual, Chapter 42 (LTDC)
- AN4861: LCD-TFT display controller on STM32 MCUs
- 예제 코드: `STM32H745I-DISCO/Examples/LTDC/LTDC_Display_1Layer`
