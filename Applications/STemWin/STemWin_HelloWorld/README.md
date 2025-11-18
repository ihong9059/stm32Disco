# STemWin_HelloWorld - SEGGER emWin 기본 데모

## 개요

이 애플리케이션은 SEGGER emWin (STemWin) 그래픽 라이브러리를 사용하여 STM32H745I-DISCO 보드의 LCD에 텍스트와 간단한 그래픽을 표시하는 기본 예제입니다. GUI 초기화, 텍스트 출력, 기본 도형 그리기를 보여줍니다.

## STemWin이란?

**STemWin**은 ST에서 제공하는 SEGGER emWin 그래픽 라이브러리의 무료 버전입니다.

### 주요 특징
- **하드웨어 가속**: DMA2D (Chrom-ART) 지원
- **다양한 위젯**: 버튼, 슬라이더, 텍스트 등
- **메모리 효율**: 최적화된 메모리 사용
- **멀티 레이어**: LTDC 멀티 레이어 지원

## 시스템 아키텍처

### STemWin 구조
```
┌──────────────────────────────────────────────────────────────┐
│                      STM32H745I-DISCO                        │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                    Application                       │    │
│  │          (HelloWorld Demo)                           │    │
│  └────────────────────┬─────────────────────────────────┘    │
│                       │                                      │
│  ┌────────────────────▼─────────────────────────────────┐    │
│  │                    STemWin                           │    │
│  │                                                      │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │    │
│  │  │  Window     │  │   Drawing   │  │   Memory    │   │    │
│  │  │  Manager    │  │   API       │  │   Devices   │   │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘   │    │
│  │                                                      │    │
│  └────────────────────┬─────────────────────────────────┘    │
│                       │                                      │
│  ┌────────────────────▼─────────────────────────────────┐    │
│  │            LCD Driver (LCDConf.c)                    │    │
│  │                                                      │    │
│  │  LTDC + DMA2D (Chrom-ART Accelerator)                │    │
│  │                                                      │    │
│  └────────────────────┬─────────────────────────────────┘    │
│                       │                                      │
│  ┌────────────────────▼─────────────────────────────────┐    │
│  │              4.3" LCD (480x272)                       │    │
│  │              RGB565 / ARGB8888                        │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## 하드웨어 구성

### LCD 사양
| 항목 | 값 |
|------|-----|
| 해상도 | 480 x 272 픽셀 |
| 색상 | 16bpp (RGB565) 또는 32bpp (ARGB8888) |
| 인터페이스 | RGB (LTDC) |
| 백라이트 | PWM 제어 |
| 터치 | 정전식 (FT5336) |

### 메모리 할당
```
SDRAM (32MB @ 0xD0000000)
├── Frame Buffer 1: 480x272x4 = 522,240 bytes
├── Frame Buffer 2: 480x272x4 = 522,240 bytes
└── GUI Heap: ~2MB
```

## STemWin 설정

### GUIConf.c - GUI 설정
```c
#include "GUI.h"

// GUI 힙 크기
#define GUI_NUMBYTES  (1024 * 1024 * 2)  // 2MB

// GUI 힙 메모리 (SDRAM에 할당)
static __attribute__((section(".sdram")))
U32 aMemory[GUI_NUMBYTES / 4];

void GUI_X_Config(void)
{
  // GUI 메모리 할당
  GUI_ALLOC_AssignMemory(aMemory, GUI_NUMBYTES);

  // 기본 폰트 설정
  GUI_SetDefaultFont(&GUI_Font13_1);
}
```

### LCDConf.c - LCD 드라이버 설정
```c
#include "GUI.h"
#include "GUIDRV_Lin.h"

// 디스플레이 크기
#define XSIZE_PHYS  480
#define YSIZE_PHYS  272

// 색상 모드
#define COLOR_MODE  GUICC_M565   // RGB565
// #define COLOR_MODE  GUICC_M8888I // ARGB8888

// 프레임 버퍼 주소
#define LCD_LAYER0_FRAME_BUFFER  0xD0000000
#define LCD_LAYER1_FRAME_BUFFER  0xD0200000

void LCD_X_Config(void)
{
  // 디스플레이 드라이버 설정
  GUI_DEVICE_CreateAndLink(GUIDRV_LIN_32,
                           COLOR_MODE,
                           GUICC_M565,
                           0, 0);

  // 디스플레이 크기 설정
  LCD_SetSizeEx(0, XSIZE_PHYS, YSIZE_PHYS);
  LCD_SetVSizeEx(0, XSIZE_PHYS, YSIZE_PHYS * 2);

  // 프레임 버퍼 주소 설정
  LCD_SetVRAMAddrEx(0, (void *)LCD_LAYER0_FRAME_BUFFER);
}
```

## 코드 구현

### 1. GUI 초기화
```c
#include "GUI.h"

int main(void)
{
  // 시스템 초기화
  HAL_Init();
  SystemClock_Config();

  // SDRAM 초기화
  BSP_SDRAM_Init(0);

  // LCD 초기화
  BSP_LCD_Init(0, LCD_ORIENTATION_LANDSCAPE);

  // emWin 초기화
  GUI_Init();

  // 백라이트 켜기
  BSP_LCD_DisplayOn(0);

  // 화면 클리어
  GUI_Clear();

  // Hello World 표시
  HelloWorld_Demo();

  while (1)
  {
    // GUI 태스크 처리
    GUI_Exec();
    HAL_Delay(10);
  }
}
```

### 2. Hello World 데모
```c
void HelloWorld_Demo(void)
{
  // 배경색 설정
  GUI_SetBkColor(GUI_WHITE);
  GUI_Clear();

  // 제목 텍스트
  GUI_SetColor(GUI_BLUE);
  GUI_SetFont(&GUI_Font32B_1);
  GUI_DispStringHCenterAt("STM32H745I-DISCO", 240, 20);

  // 부제목
  GUI_SetColor(GUI_DARKGRAY);
  GUI_SetFont(&GUI_Font20_1);
  GUI_DispStringHCenterAt("STemWin Hello World Demo", 240, 60);

  // 환영 메시지
  GUI_SetColor(GUI_BLACK);
  GUI_SetFont(&GUI_Font16_1);
  GUI_DispStringAt("Welcome to emWin Graphics Library!", 50, 120);

  // 하단 정보
  GUI_SetFont(&GUI_Font13_1);
  GUI_DispStringAt("Press any key to continue...", 120, 240);
}
```

### 3. 텍스트 표시 함수들
```c
void Text_Demo(void)
{
  // 배경 클리어
  GUI_SetBkColor(GUI_WHITE);
  GUI_Clear();

  // 왼쪽 정렬
  GUI_SetTextMode(GUI_TM_NORMAL);
  GUI_SetFont(&GUI_Font16_1);
  GUI_SetColor(GUI_BLACK);
  GUI_DispStringAt("Left Aligned Text", 10, 10);

  // 중앙 정렬
  GUI_DispStringHCenterAt("Center Aligned Text", 240, 50);

  // 오른쪽 정렬
  GUI_GotoXY(470, 90);
  GUI_DispStringRightAlign("Right Aligned Text");

  // 여러 줄 텍스트
  char multiline[] = "This is a multi-line\ntext example.\nLine 3 here.";
  GUI_DispStringInRectWrap(multiline,
                           &(GUI_RECT){10, 130, 200, 250},
                           GUI_TA_LEFT | GUI_TA_TOP,
                           GUI_WRAPMODE_WORD);

  // 수직 텍스트
  GUI_SetFont(&GUI_Font13_1);
  GUI_DispStringInRectEx("Vertical",
                         &(GUI_RECT){450, 100, 470, 250},
                         GUI_TA_HCENTER | GUI_TA_VCENTER,
                         strlen("Vertical"),
                         GUI_ROTATE_CCW);

  // 텍스트 배경색
  GUI_SetBkColor(GUI_YELLOW);
  GUI_SetTextMode(GUI_TM_NORMAL);
  GUI_DispStringAt("Text with background", 10, 200);
  GUI_SetTextMode(GUI_TM_TRANS);
  GUI_SetBkColor(GUI_WHITE);
}
```

### 4. 기본 도형 그리기
```c
void Drawing_Demo(void)
{
  // 배경 클리어
  GUI_SetBkColor(GUI_WHITE);
  GUI_Clear();

  GUI_SetFont(&GUI_Font16B_1);
  GUI_DispStringHCenterAt("Basic Drawing Demo", 240, 10);

  // 점
  GUI_SetColor(GUI_BLACK);
  for (int i = 0; i < 10; i++)
  {
    GUI_DrawPixel(50 + i*10, 50);
  }

  // 선
  GUI_SetColor(GUI_RED);
  GUI_DrawLine(20, 70, 150, 70);     // 수평선
  GUI_DrawLine(20, 80, 20, 150);     // 수직선
  GUI_DrawLine(40, 80, 120, 140);    // 대각선

  // 사각형
  GUI_SetColor(GUI_BLUE);
  GUI_DrawRect(160, 50, 260, 120);   // 테두리만
  GUI_SetColor(GUI_LIGHTBLUE);
  GUI_FillRect(280, 50, 380, 120);   // 채움

  // 둥근 사각형
  GUI_SetColor(GUI_GREEN);
  GUI_DrawRoundedRect(160, 140, 260, 200, 10);
  GUI_SetColor(GUI_LIGHTGREEN);
  GUI_FillRoundedRect(280, 140, 380, 200, 10);

  // 원
  GUI_SetColor(GUI_MAGENTA);
  GUI_DrawCircle(430, 85, 30);       // 테두리만
  GUI_SetColor(GUI_CYAN);
  GUI_FillCircle(430, 170, 30);      // 채움

  // 타원
  GUI_SetColor(GUI_ORANGE);
  GUI_DrawEllipse(70, 220, 50, 25);
  GUI_FillEllipse(170, 220, 50, 25);

  // 다각형
  const GUI_POINT aPoints[] = {
    {260, 230},
    {300, 200},
    {340, 230},
    {320, 260},
    {280, 260}
  };
  GUI_SetColor(GUI_DARKRED);
  GUI_FillPolygon(aPoints, 5, 0, 0);

  // 호
  GUI_SetColor(GUI_DARKCYAN);
  GUI_DrawArc(430, 230, 40, 40, 0, 270);
}
```

### 5. 색상 데모
```c
void Color_Demo(void)
{
  GUI_SetBkColor(GUI_WHITE);
  GUI_Clear();

  GUI_SetFont(&GUI_Font16B_1);
  GUI_DispStringHCenterAt("Color Demo", 240, 10);

  // 기본 색상 팔레트
  struct {
    GUI_COLOR color;
    const char *name;
  } colors[] = {
    {GUI_RED, "Red"},
    {GUI_GREEN, "Green"},
    {GUI_BLUE, "Blue"},
    {GUI_CYAN, "Cyan"},
    {GUI_MAGENTA, "Magenta"},
    {GUI_YELLOW, "Yellow"},
    {GUI_BLACK, "Black"},
    {GUI_WHITE, "White"}
  };

  // 색상 박스 그리기
  GUI_SetFont(&GUI_Font13_1);
  for (int i = 0; i < 8; i++)
  {
    int x = (i % 4) * 110 + 30;
    int y = (i / 4) * 80 + 50;

    GUI_SetColor(colors[i].color);
    GUI_FillRect(x, y, x + 80, y + 50);

    GUI_SetColor(GUI_BLACK);
    GUI_DrawRect(x, y, x + 80, y + 50);
    GUI_DispStringAt(colors[i].name, x + 5, y + 55);
  }

  // 그라데이션
  for (int i = 0; i < 256; i++)
  {
    GUI_SetColor(GUI_MAKE_COLOR((i << 16) | (i << 8) | i));  // 그레이스케일
    GUI_DrawVLine(i + 112, 220, 250);
  }

  GUI_SetColor(GUI_BLACK);
  GUI_DrawRect(112, 220, 367, 250);
}
```

### 6. 폰트 데모
```c
void Font_Demo(void)
{
  GUI_SetBkColor(GUI_WHITE);
  GUI_Clear();

  GUI_SetColor(GUI_BLACK);

  // 다양한 폰트 크기
  int y = 10;

  GUI_SetFont(&GUI_Font8_1);
  GUI_DispStringAt("Font8_1 - 8 pixels height", 10, y);
  y += 15;

  GUI_SetFont(&GUI_Font13_1);
  GUI_DispStringAt("Font13_1 - 13 pixels height", 10, y);
  y += 20;

  GUI_SetFont(&GUI_Font16_1);
  GUI_DispStringAt("Font16_1 - 16 pixels height", 10, y);
  y += 25;

  GUI_SetFont(&GUI_Font20_1);
  GUI_DispStringAt("Font20_1 - 20 pixels height", 10, y);
  y += 30;

  GUI_SetFont(&GUI_Font24_1);
  GUI_DispStringAt("Font24_1 - 24 pixels height", 10, y);
  y += 35;

  GUI_SetFont(&GUI_Font32_1);
  GUI_DispStringAt("Font32_1 - 32 pixels", 10, y);
  y += 45;

  // 볼드 폰트
  GUI_SetFont(&GUI_Font16B_1);
  GUI_DispStringAt("Bold Font - Font16B_1", 10, y);
  y += 25;

  // 숫자 폰트
  GUI_SetFont(&GUI_FontD24x32);
  GUI_DispStringAt("1234567890", 10, y);
}
```

## 고급 기능

### 1. 더블 버퍼링
```c
// 더블 버퍼링 활성화
void EnableDoubleBuffering(void)
{
  // 다중 버퍼링 설정
  WM_MULTIBUF_Enable(1);

  // 화면 업데이트
  GUI_Clear();
  // ... 그리기 작업 ...

  // 버퍼 전환
  GUI_MULTIBUF_Begin();
  // ... 그리기 작업 ...
  GUI_MULTIBUF_End();
}
```

### 2. 안티앨리어싱
```c
void AntiAliasing_Demo(void)
{
  GUI_SetBkColor(GUI_WHITE);
  GUI_Clear();

  // 일반 원
  GUI_SetColor(GUI_RED);
  GUI_DrawCircle(100, 100, 50);

  // 안티앨리어싱 원
  GUI_AA_SetFactor(4);  // 4x 안티앨리어싱
  GUI_AA_DrawCircle(250, 100, 50);

  // 안티앨리어싱 선
  GUI_AA_DrawLine(50, 200, 200, 250);
}
```

### 3. 알파 블렌딩
```c
void AlphaBlending_Demo(void)
{
  GUI_SetBkColor(GUI_WHITE);
  GUI_Clear();

  // 배경 그리기
  GUI_SetColor(GUI_BLUE);
  GUI_FillRect(50, 50, 200, 200);

  // 알파 블렌딩 (반투명)
  GUI_EnableAlpha(1);
  GUI_SetAlpha(128);  // 50% 투명도

  GUI_SetColor(GUI_RED);
  GUI_FillRect(100, 100, 250, 250);

  GUI_SetAlpha(255);  // 불투명으로 복원
  GUI_EnableAlpha(0);
}
```

### 4. 비트맵 표시
```c
// 외부 비트맵 (C 배열)
extern const GUI_BITMAP bmLogo;

void Bitmap_Demo(void)
{
  GUI_SetBkColor(GUI_WHITE);
  GUI_Clear();

  // 비트맵 표시
  GUI_DrawBitmap(&bmLogo, 50, 50);

  // 스케일링된 비트맵
  GUI_DrawBitmapMag(&bmLogo, 200, 50, 2, 2);  // 2배 확대

  // 회전된 비트맵
  GUI_DrawBitmapEx(&bmLogo, 350, 100, 0, 0, 1000, 45);  // 45도 회전
}
```

## DMA2D 하드웨어 가속

### DMA2D 설정
```c
// LCDConf.c에서 DMA2D 활성화
#define GUI_USE_DMA2D  1

// DMA2D 색상 변환 가속
int LCD_X_DisplayDriver(unsigned LayerIndex, unsigned Cmd, void *p)
{
  switch (Cmd)
  {
    case LCD_X_SETCOLORREG:
      // DMA2D CLUT 설정
      break;

    case LCD_X_SETALPHA:
      // 레이어 알파 설정
      break;
  }
  return 0;
}
```

### 하드웨어 가속 사각형 채우기
```c
// DMA2D를 통한 빠른 사각형 채우기
void DMA2D_FillRect(U32 LayerIndex, void *pDst, U32 xSize, U32 ySize, U32 OffLine, U32 ColorIndex)
{
  DMA2D->CR = 0x00030000;  // Register to memory mode
  DMA2D->OCOLR = ColorIndex;
  DMA2D->OMAR = (U32)pDst;
  DMA2D->OOR = OffLine;
  DMA2D->OPFCCR = LTDC_PIXEL_FORMAT_RGB565;
  DMA2D->NLR = (U32)(xSize << 16) | ySize;

  DMA2D->CR |= DMA2D_CR_START;
  while (DMA2D->CR & DMA2D_CR_START);
}
```

## 메모리 최적화

### 힙 사용량 모니터링
```c
void PrintMemoryInfo(void)
{
  GUI_ALLOC_INFO AllocInfo;
  GUI_ALLOC_GetMemInfo(&AllocInfo);

  printf("GUI Memory Info:\n");
  printf("  Total: %d bytes\n", AllocInfo.TotalBytes);
  printf("  Free:  %d bytes\n", AllocInfo.FreeBytes);
  printf("  Used:  %d bytes\n", AllocInfo.UsedBytes);
  printf("  Max:   %d bytes\n", AllocInfo.MaxUsedBytes);
}
```

### 메모리 절약 팁
```c
// 작은 폰트 사용
GUI_SetFont(&GUI_Font13_1);  // Font32 대신

// 색상 깊이 줄이기
#define COLOR_MODE  GUICC_M565  // 16bpp (ARGB8888 대신)

// 더블 버퍼링 비활성화 (메모리 절약)
WM_MULTIBUF_Enable(0);
```

## 트러블슈팅

### 화면이 표시되지 않는 경우

1. **LTDC 설정 확인**:
   ```c
   // LTDC 클럭 활성화
   __HAL_RCC_LTDC_CLK_ENABLE();
   __HAL_RCC_DMA2D_CLK_ENABLE();
   ```

2. **프레임 버퍼 주소 확인**:
   - SDRAM 초기화 필요
   - 올바른 메모리 영역 사용

3. **백라이트 확인**:
   ```c
   BSP_LCD_DisplayOn(0);
   ```

### 색상이 잘못 표시되는 경우

1. **색상 모드 확인**: RGB565 vs ARGB8888
2. **바이트 순서 확인**: Little Endian
3. **LCDConf.c 설정 검토**

### 느린 성능

1. **DMA2D 활성화**: 하드웨어 가속
2. **더블 버퍼링 사용**: 깜빡임 방지
3. **MPU/캐시 설정**: SDRAM 캐시 활성화

## 성능 고려사항

### 최적화 팁
- **부분 업데이트**: GUI_Exec1() 사용
- **그리기 영역 제한**: GUI_SetClipRect()
- **메모리 장치 사용**: 오프스크린 렌더링

### 프레임 레이트
```c
// FPS 측정
static uint32_t frame_count = 0;
static uint32_t last_time = 0;

void CalculateFPS(void)
{
  frame_count++;
  uint32_t current_time = HAL_GetTick();

  if (current_time - last_time >= 1000)
  {
    printf("FPS: %lu\n", frame_count);
    frame_count = 0;
    last_time = current_time;
  }
}
```

## 참고 자료

- STemWin 사용자 매뉴얼: UM1718
- emWin 공식 문서: https://www.segger.com/products/user-interface/emwin/
- STM32H7 LTDC 응용 노트: AN4861
- 예제: `STM32H745I-DISCO/Applications/STemWin/STemWin_HelloWorld`
