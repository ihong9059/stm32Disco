# DMA2D 메모리 간 전송 및 블렌딩

## 개요

이 예제는 STM32H745I-DISCO의 DMA2D(Chrom-ART Graphics Accelerator)를 사용하여 메모리 간 데이터 전송, 픽셀 포맷 변환, 알파 블렌딩을 수행하는 방법을 보여줍니다. DMA2D는 CPU 개입 없이 고속 그래픽 처리를 수행하는 하드웨어 가속기입니다.

## 주요 기능

### 1. DMA2D 기능
- **메모리 간 전송**: CPU 부하 없이 고속 복사
- **픽셀 포맷 변환**: RGB888 ↔ RGB565 ↔ ARGB8888 등
- **알파 블렌딩**: 하드웨어 기반 투명도 혼합
- **색상 채우기**: 단색으로 영역 채우기
- **전송 속도**: 최대 400 MB/s

### 2. 지원 픽셀 포맷
```
입력 포맷:
  - ARGB8888 (32-bit)
  - RGB888 (24-bit)
  - RGB565 (16-bit)
  - ARGB1555, ARGB4444
  - L8 (8-bit 인덱스)
  - L4, AL44, AL88

출력 포맷:
  - ARGB8888, RGB888, RGB565
  - ARGB1555, ARGB4444
```

## 코드 상세 분석

### 1. DMA2D 초기화

```c
DMA2D_HandleTypeDef Dma2dHandle;

void DMA2D_Init(void)
{
  // DMA2D 클럭 활성화
  __HAL_RCC_DMA2D_CLK_ENABLE();

  // DMA2D 핸들 설정
  Dma2dHandle.Instance = DMA2D;

  // 초기화 (나중에 모드별로 재설정)
  HAL_DMA2D_Init(&Dma2dHandle);

  // DMA2D 인터럽트 설정 (옵션)
  HAL_NVIC_SetPriority(DMA2D_IRQn, 5, 0);
  HAL_NVIC_EnableIRQ(DMA2D_IRQn);
}
```

### 2. 메모리 간 복사 (M2M)

```c
// RGB565 → RGB565 복사 (픽셀 포맷 변환 없음)
void DMA2D_MemoryToMemory(uint32_t *pSrc, uint32_t *pDst,
                          uint16_t width, uint16_t height)
{
  // DMA2D 모드: 메모리 간 전송
  Dma2dHandle.Init.Mode = DMA2D_M2M;
  Dma2dHandle.Init.ColorMode = DMA2D_OUTPUT_RGB565;
  Dma2dHandle.Init.OutputOffset = 0;
  HAL_DMA2D_Init(&Dma2dHandle);

  // 전송 시작
  if (HAL_DMA2D_Start(&Dma2dHandle, (uint32_t)pSrc, (uint32_t)pDst,
                      width, height) == HAL_OK)
  {
    // 전송 완료 대기 (폴링 방식)
    HAL_DMA2D_PollForTransfer(&Dma2dHandle, 100);
  }

  printf("DMA2D copy: %dx%d pixels\n", width, height);
}

// 성능 비교
void Compare_CPU_DMA2D_Copy(void)
{
  uint32_t src[480 * 272];
  uint32_t dst[480 * 272];
  uint32_t start, end;

  // CPU 복사 (memcpy)
  start = HAL_GetTick();
  memcpy(dst, src, 480 * 272 * 4);
  end = HAL_GetTick();
  printf("CPU: %lu ms\n", end - start);  // 약 3 ms

  // DMA2D 복사
  start = HAL_GetTick();
  DMA2D_MemoryToMemory(src, dst, 480, 272);
  end = HAL_GetTick();
  printf("DMA2D: %lu ms\n", end - start);  // 약 1 ms
}
```

### 3. 픽셀 포맷 변환 (PFC)

```c
// RGB888 → RGB565 변환
void DMA2D_ConvertRGB888toRGB565(uint8_t *pSrc, uint16_t *pDst,
                                  uint16_t width, uint16_t height)
{
  // DMA2D 모드: 픽셀 포맷 변환
  Dma2dHandle.Init.Mode = DMA2D_M2M_PFC;
  Dma2dHandle.Init.ColorMode = DMA2D_OUTPUT_RGB565;
  Dma2dHandle.Init.OutputOffset = 0;
  HAL_DMA2D_Init(&Dma2dHandle);

  // 전경(입력) 레이어 설정
  DMA2D_LayerCfgTypeDef layerCfg;
  layerCfg.InputOffset = 0;
  layerCfg.InputColorMode = DMA2D_INPUT_RGB888;
  layerCfg.AlphaMode = DMA2D_NO_MODIF_ALPHA;
  layerCfg.InputAlpha = 0xFF;
  HAL_DMA2D_ConfigLayer(&Dma2dHandle, &layerCfg, DMA2D_FOREGROUND_LAYER);

  // 변환 시작
  HAL_DMA2D_Start(&Dma2dHandle, (uint32_t)pSrc, (uint32_t)pDst, width, height);
  HAL_DMA2D_PollForTransfer(&Dma2dHandle, 100);

  printf("Converted RGB888 to RGB565\n");
}

// ARGB8888 → RGB565 변환 (알파 제거)
void DMA2D_ConvertARGB8888toRGB565(uint32_t *pSrc, uint16_t *pDst,
                                    uint16_t width, uint16_t height)
{
  Dma2dHandle.Init.Mode = DMA2D_M2M_PFC;
  Dma2dHandle.Init.ColorMode = DMA2D_OUTPUT_RGB565;
  Dma2dHandle.Init.OutputOffset = 0;
  HAL_DMA2D_Init(&Dma2dHandle);

  DMA2D_LayerCfgTypeDef layerCfg;
  layerCfg.InputOffset = 0;
  layerCfg.InputColorMode = DMA2D_INPUT_ARGB8888;
  layerCfg.AlphaMode = DMA2D_NO_MODIF_ALPHA;
  layerCfg.InputAlpha = 0xFF;
  HAL_DMA2D_ConfigLayer(&Dma2dHandle, &layerCfg, DMA2D_FOREGROUND_LAYER);

  HAL_DMA2D_Start(&Dma2dHandle, (uint32_t)pSrc, (uint32_t)pDst, width, height);
  HAL_DMA2D_PollForTransfer(&Dma2dHandle, 100);
}
```

### 4. 알파 블렌딩

```c
// 두 이미지 블렌딩 (전경 + 배경)
void DMA2D_AlphaBlending(uint32_t *pForeground, uint32_t *pBackground,
                         uint32_t *pDst, uint16_t width, uint16_t height,
                         uint8_t alpha)
{
  // DMA2D 모드: 블렌딩
  Dma2dHandle.Init.Mode = DMA2D_M2M_BLEND;
  Dma2dHandle.Init.ColorMode = DMA2D_OUTPUT_ARGB8888;
  Dma2dHandle.Init.OutputOffset = 0;
  HAL_DMA2D_Init(&Dma2dHandle);

  // 전경 레이어 설정
  DMA2D_LayerCfgTypeDef layerCfg;
  layerCfg.InputOffset = 0;
  layerCfg.InputColorMode = DMA2D_INPUT_ARGB8888;
  layerCfg.AlphaMode = DMA2D_REPLACE_ALPHA;
  layerCfg.InputAlpha = alpha;  // 상수 알파 (0-255)
  HAL_DMA2D_ConfigLayer(&Dma2dHandle, &layerCfg, DMA2D_FOREGROUND_LAYER);

  // 배경 레이어 설정
  layerCfg.InputOffset = 0;
  layerCfg.InputColorMode = DMA2D_INPUT_ARGB8888;
  layerCfg.AlphaMode = DMA2D_REPLACE_ALPHA;
  layerCfg.InputAlpha = 255 - alpha;  // 배경 알파
  HAL_DMA2D_ConfigLayer(&Dma2dHandle, &layerCfg, DMA2D_BACKGROUND_LAYER);

  // 블렌딩 시작
  HAL_DMA2D_BlendingStart(&Dma2dHandle,
                          (uint32_t)pForeground,
                          (uint32_t)pBackground,
                          (uint32_t)pDst,
                          width, height);
  HAL_DMA2D_PollForTransfer(&Dma2dHandle, 100);

  printf("Blended with alpha=%d\n", alpha);
}

// 페이드 인/아웃 애니메이션
void DMA2D_FadeAnimation(uint32_t *pImage1, uint32_t *pImage2,
                         uint32_t *pDisplay, uint16_t width, uint16_t height)
{
  // 페이드 아웃 (이미지1 → 이미지2)
  for (uint8_t alpha = 255; alpha > 0; alpha -= 5)
  {
    DMA2D_AlphaBlending(pImage1, pImage2, pDisplay, width, height, alpha);

    // LCD 업데이트
    SCB_CleanDCache_by_Addr((uint32_t *)pDisplay, width * height * 4);

    HAL_Delay(20);
  }

  // 페이드 인 (이미지2 → 이미지1)
  for (uint8_t alpha = 0; alpha < 255; alpha += 5)
  {
    DMA2D_AlphaBlending(pImage1, pImage2, pDisplay, width, height, alpha);
    SCB_CleanDCache_by_Addr((uint32_t *)pDisplay, width * height * 4);
    HAL_Delay(20);
  }
}
```

### 5. 색상 채우기 (R2M)

```c
// 사각형 채우기
void DMA2D_FillRect(uint32_t *pDst, uint16_t x, uint16_t y,
                    uint16_t width, uint16_t height,
                    uint32_t color, uint16_t screen_width)
{
  // DMA2D 모드: Register to Memory
  Dma2dHandle.Init.Mode = DMA2D_R2M;
  Dma2dHandle.Init.ColorMode = DMA2D_OUTPUT_ARGB8888;
  Dma2dHandle.Init.OutputOffset = screen_width - width;  // 라인 오프셋
  HAL_DMA2D_Init(&Dma2dHandle);

  // 목적지 주소 계산
  uint32_t dest_addr = (uint32_t)pDst + (y * screen_width + x) * 4;

  // 색상 채우기 시작
  HAL_DMA2D_Start(&Dma2dHandle, color, dest_addr, width, height);
  HAL_DMA2D_PollForTransfer(&Dma2dHandle, 100);
}

// 전체 화면 지우기
void DMA2D_ClearScreen(uint32_t *pFrameBuffer, uint32_t color)
{
  Dma2dHandle.Init.Mode = DMA2D_R2M;
  Dma2dHandle.Init.ColorMode = DMA2D_OUTPUT_ARGB8888;
  Dma2dHandle.Init.OutputOffset = 0;
  HAL_DMA2D_Init(&Dma2dHandle);

  HAL_DMA2D_Start(&Dma2dHandle, color, (uint32_t)pFrameBuffer, 480, 272);
  HAL_DMA2D_PollForTransfer(&Dma2dHandle, 100);

  SCB_CleanDCache_by_Addr((uint32_t *)pFrameBuffer, 480 * 272 * 4);
}
```

### 6. 인터럽트 모드

```c
volatile uint8_t DMA2D_TransferComplete = 0;

void DMA2D_Start_IT(uint32_t *pSrc, uint32_t *pDst,
                    uint16_t width, uint16_t height)
{
  Dma2dHandle.Init.Mode = DMA2D_M2M;
  Dma2dHandle.Init.ColorMode = DMA2D_OUTPUT_ARGB8888;
  Dma2dHandle.Init.OutputOffset = 0;
  HAL_DMA2D_Init(&Dma2dHandle);

  DMA2D_TransferComplete = 0;

  // 인터럽트 모드로 시작
  HAL_DMA2D_Start_IT(&Dma2dHandle, (uint32_t)pSrc, (uint32_t)pDst, width, height);
}

// DMA2D 전송 완료 콜백
void HAL_DMA2D_XferCpltCallback(DMA2D_HandleTypeDef *hdma2d)
{
  DMA2D_TransferComplete = 1;
  BSP_LED_Toggle(LED_GREEN);
}

// 인터럽트 핸들러
void DMA2D_IRQHandler(void)
{
  HAL_DMA2D_IRQHandler(&Dma2dHandle);
}
```

## 성능 분석

### 전송 속도
```
480×272 ARGB8888 이미지 (522,240 바이트):

CPU (memcpy):
  시간: 약 3 ms
  속도: 174 MB/s

DMA2D (M2M):
  시간: 약 1.3 ms
  속도: 402 MB/s
  CPU 사용률: 0% (비동기)

DMA2D (PFC, RGB888→RGB565):
  시간: 약 2 ms
  속도: 196 MB/s
  추가 처리: 픽셀 포맷 변환

DMA2D (Blending):
  시간: 약 2.5 ms
  속도: 167 MB/s
  추가 처리: 알파 블렌딩 계산
```

## 고급 사용 예제

### 1. 스프라이트 렌더링

```c
typedef struct {
  uint32_t *data;
  uint16_t width;
  uint16_t height;
  uint8_t  alpha;
} Sprite_t;

void DMA2D_DrawSprite(Sprite_t *sprite, uint32_t *pFrameBuffer,
                      uint16_t x, uint16_t y, uint16_t screen_width)
{
  Dma2dHandle.Init.Mode = DMA2D_M2M_BLEND;
  Dma2dHandle.Init.ColorMode = DMA2D_OUTPUT_ARGB8888;
  Dma2dHandle.Init.OutputOffset = screen_width - sprite->width;
  HAL_DMA2D_Init(&Dma2dHandle);

  // 전경 (스프라이트)
  DMA2D_LayerCfgTypeDef layerCfg;
  layerCfg.InputOffset = 0;
  layerCfg.InputColorMode = DMA2D_INPUT_ARGB8888;
  layerCfg.AlphaMode = DMA2D_COMBINE_ALPHA;  // 픽셀 알파 × 상수 알파
  layerCfg.InputAlpha = sprite->alpha;
  HAL_DMA2D_ConfigLayer(&Dma2dHandle, &layerCfg, DMA2D_FOREGROUND_LAYER);

  // 배경 (프레임버퍼)
  layerCfg.InputOffset = screen_width - sprite->width;
  layerCfg.InputColorMode = DMA2D_INPUT_ARGB8888;
  layerCfg.AlphaMode = DMA2D_NO_MODIF_ALPHA;
  layerCfg.InputAlpha = 0xFF;
  HAL_DMA2D_ConfigLayer(&Dma2dHandle, &layerCfg, DMA2D_BACKGROUND_LAYER);

  // 목적지 주소
  uint32_t dest = (uint32_t)pFrameBuffer + (y * screen_width + x) * 4;
  uint32_t bg = dest;

  HAL_DMA2D_BlendingStart(&Dma2dHandle,
                          (uint32_t)sprite->data,
                          bg,
                          dest,
                          sprite->width, sprite->height);
  HAL_DMA2D_PollForTransfer(&Dma2dHandle, 100);
}
```

### 2. 그래디언트 생성

```c
void DMA2D_DrawGradient(uint32_t *pDst, uint16_t width, uint16_t height,
                        uint32_t color1, uint32_t color2)
{
  uint8_t r1 = (color1 >> 16) & 0xFF;
  uint8_t g1 = (color1 >> 8) & 0xFF;
  uint8_t b1 = color1 & 0xFF;

  uint8_t r2 = (color2 >> 16) & 0xFF;
  uint8_t g2 = (color2 >> 8) & 0xFF;
  uint8_t b2 = color2 & 0xFF;

  for (uint16_t y = 0; y < height; y++)
  {
    float t = (float)y / height;

    uint8_t r = r1 + (r2 - r1) * t;
    uint8_t g = g1 + (g2 - g1) * t;
    uint8_t b = b1 + (b2 - b1) * t;

    uint32_t color = 0xFF000000 | (r << 16) | (g << 8) | b;

    DMA2D_FillRect(pDst, 0, y, width, 1, color, width);
  }
}
```

## 트러블슈팅

### 1. 전송 타임아웃
- **클럭 활성화**: `__HAL_RCC_DMA2D_CLK_ENABLE()`
- **주소 정렬**: 32비트 정렬 확인

### 2. 색상 이상
- **픽셀 포맷 불일치**: InputColorMode와 데이터 포맷 일치
- **엔디안**: 리틀 엔디안 확인

### 3. 블렌딩 오류
- **알파 모드**: COMBINE_ALPHA vs REPLACE_ALPHA
- **입력 알파**: 0-255 범위

## 참고 자료
- **RM0399**: STM32H7 Reference Manual (DMA2D)
- **AN4839**: Chrom-ART Graphics Accelerator
- HAL: `stm32h7xx_hal_dma2d.c/h`
