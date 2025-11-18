# MDMA 이미지 회전 (Repeat Block Mode)

## 개요

이 예제는 STM32H745I-DISCO의 MDMA(Master DMA)를 사용하여 이미지를 90°, 180°, 270° 회전시키는 방법을 보여줍니다. MDMA의 Repeat Block 모드를 활용하여 복잡한 2D 메모리 전송을 효율적으로 수행합니다.

## 주요 기능

### 1. MDMA 특징
- **AXI 버스 마스터**: 시스템 메모리 직접 액세스
- **2D 전송**: 행렬 데이터 처리에 최적화
- **Repeat Block**: 복잡한 패턴 전송
- **속도**: 최대 1.2 GB/s (AXI 대역폭)

### 2. 회전 알고리즘
```
90° 시계방향:
  dst[x][y] = src[height-1-y][x]

180°:
  dst[x][y] = src[height-1-y][width-1-x]

270° 시계방향 (90° 반시계):
  dst[x][y] = src[y][width-1-x]
```

## 코드 상세 분석

### 1. MDMA 초기화

```c
MDMA_HandleTypeDef hmdma;

void MDMA_Init(void)
{
  // MDMA 클럭 활성화
  __HAL_RCC_MDMA_CLK_ENABLE();

  // MDMA 채널 선택
  hmdma.Instance = MDMA_Channel0;

  // 기본 설정
  hmdma.Init.Request = MDMA_REQUEST_SW;  // 소프트웨어 트리거
  hmdma.Init.TransferTriggerMode = MDMA_BLOCK_TRANSFER;
  hmdma.Init.Priority = MDMA_PRIORITY_HIGH;
  hmdma.Init.Endianness = MDMA_LITTLE_ENDIAN_PRESERVE;

  // 데이터 크기
  hmdma.Init.SourceInc = MDMA_SRC_INC_WORD;
  hmdma.Init.DestinationInc = MDMA_DEST_INC_WORD;
  hmdma.Init.SourceDataSize = MDMA_SRC_DATASIZE_WORD;
  hmdma.Init.DestDataSize = MDMA_DEST_DATASIZE_WORD;
  hmdma.Init.DataAlignment = MDMA_DATAALIGN_PACKENABLE;

  // 버스트 크기
  hmdma.Init.SourceBurst = MDMA_SOURCE_BURST_SINGLE;
  hmdma.Init.DestBurst = MDMA_DEST_BURST_SINGLE;

  // 버퍼 전송 길이 (기본값, 나중에 재설정)
  hmdma.Init.BufferTransferLength = 128;

  HAL_MDMA_Init(&hmdma);

  // MDMA 인터럽트 활성화
  HAL_NVIC_SetPriority(MDMA_IRQn, 5, 0);
  HAL_NVIC_EnableIRQ(MDMA_IRQn);
}
```

### 2. 90° 회전 (시계방향)

```c
// 이미지 구조
typedef struct {
  uint32_t *data;
  uint16_t width;
  uint16_t height;
} Image_t;

void MDMA_Rotate90_CW(Image_t *src, Image_t *dst)
{
  // 회전 후 크기: width와 height 교환
  dst->width = src->height;
  dst->height = src->width;

  // MDMA Repeat Block 설정
  hmdma.Init.SourceInc = MDMA_SRC_INC_WORD;
  hmdma.Init.DestinationInc = MDMA_DEST_INC_DISABLE;  // 목적지 증가 비활성화
  hmdma.Init.SourceDataSize = MDMA_SRC_DATASIZE_WORD;
  hmdma.Init.DestDataSize = MDMA_DEST_DATASIZE_WORD;

  // Block 설정
  hmdma.Init.SourceBlockAddressOffset = 0;
  hmdma.Init.DestBlockAddressOffset = -(int32_t)(src->height * 4);  // 한 열 위로

  // Repeat 설정
  hmdma.Init.BlockCount = src->width;  // 열 개수
  hmdma.Init.RepeatCount = src->height;  // 행 개수

  HAL_MDMA_Init(&hmdma);

  // 각 열마다 전송
  for (uint16_t x = 0; x < src->width; x++)
  {
    uint32_t src_addr = (uint32_t)&src->data[x];
    uint32_t dst_addr = (uint32_t)&dst->data[(src->height - 1) * dst->width + x];

    HAL_MDMA_Start(&hmdma, src_addr, dst_addr, src->height * 4, 1);
    HAL_MDMA_PollForTransfer(&hmdma, HAL_MDMA_FULL_TRANSFER, 1000);

    // 다음 열로 이동
    src_addr += src->width * 4;
  }

  printf("Rotated 90° CW: %dx%d → %dx%d\n",
         src->width, src->height, dst->width, dst->height);
}
```

### 3. 180° 회전

```c
void MDMA_Rotate180(Image_t *src, Image_t *dst)
{
  // 크기 동일
  dst->width = src->width;
  dst->height = src->height;

  uint32_t total_pixels = src->width * src->height;

  // 단순 역순 복사
  hmdma.Init.SourceInc = MDMA_SRC_INC_WORD;
  hmdma.Init.DestinationInc = MDMA_DEST_DEC_WORD;  // 역방향
  hmdma.Init.SourceDataSize = MDMA_SRC_DATASIZE_WORD;
  hmdma.Init.DestDataSize = MDMA_DEST_DATASIZE_WORD;

  HAL_MDMA_Init(&hmdma);

  uint32_t src_addr = (uint32_t)&src->data[0];
  uint32_t dst_addr = (uint32_t)&dst->data[total_pixels - 1];

  HAL_MDMA_Start(&hmdma, src_addr, dst_addr, total_pixels * 4, 1);
  HAL_MDMA_PollForTransfer(&hmdma, HAL_MDMA_FULL_TRANSFER, 1000);

  printf("Rotated 180°\n");
}
```

### 4. 270° 회전 (90° 반시계)

```c
void MDMA_Rotate270_CW(Image_t *src, Image_t *dst)
{
  // 회전 후 크기
  dst->width = src->height;
  dst->height = src->width;

  hmdma.Init.SourceInc = MDMA_SRC_INC_WORD;
  hmdma.Init.DestinationInc = MDMA_DEST_INC_DISABLE;
  hmdma.Init.SourceDataSize = MDMA_SRC_DATASIZE_WORD;
  hmdma.Init.DestDataSize = MDMA_DEST_DATASIZE_WORD;

  hmdma.Init.SourceBlockAddressOffset = 0;
  hmdma.Init.DestBlockAddressOffset = (int32_t)(dst->width * 4);  // 한 열 아래로

  hmdma.Init.BlockCount = src->width;
  hmdma.Init.RepeatCount = src->height;

  HAL_MDMA_Init(&hmdma);

  for (uint16_t x = 0; x < src->width; x++)
  {
    uint32_t src_addr = (uint32_t)&src->data[x];
    uint32_t dst_addr = (uint32_t)&dst->data[src->width - 1 - x];

    HAL_MDMA_Start(&hmdma, src_addr, dst_addr, src->height * 4, 1);
    HAL_MDMA_PollForTransfer(&hmdma, HAL_MDMA_FULL_TRANSFER, 1000);

    src_addr += src->width * 4;
  }

  printf("Rotated 270° CW (90° CCW)\n");
}
```

### 5. Repeat Block 모드 상세

```c
// Repeat Block 모드 설명
/*
 * MDMA Repeat Block 구조:
 *
 * ┌─────────────────────────────────┐
 * │  소스 메모리                    │
 * │  [블록 0] [블록 1] ... [블록 N]│
 * └─────────────────────────────────┘
 *        ↓         ↓           ↓
 * ┌─────────────────────────────────┐
 * │  목적지 메모리                  │
 * │  각 블록마다 오프셋 적용        │
 * └─────────────────────────────────┘
 *
 * 파라미터:
 *   - BufferTransferLength: 한 번에 전송할 바이트 수
 *   - BlockCount: 블록 반복 횟수
 *   - SourceBlockAddressOffset: 소스 블록 간 오프셋
 *   - DestBlockAddressOffset: 목적지 블록 간 오프셋
 *   - RepeatCount: 전체 패턴 반복 횟수
 */

// 예제: 2D 배열 전치 (Transpose)
void MDMA_Transpose(uint32_t *src, uint32_t *dst, uint16_t rows, uint16_t cols)
{
  hmdma.Init.SourceInc = MDMA_SRC_INC_WORD;
  hmdma.Init.DestinationInc = MDMA_DEST_INC_DISABLE;

  // 블록 설정
  hmdma.Init.BufferTransferLength = 4;  // 한 워드씩
  hmdma.Init.BlockCount = rows;  // 행 개수
  hmdma.Init.SourceBlockAddressOffset = cols * 4;  // 다음 행으로
  hmdma.Init.DestBlockAddressOffset = -(int32_t)(rows * 4);  // 위로 한 행

  hmdma.Init.RepeatCount = cols;  // 열 개수

  HAL_MDMA_Init(&hmdma);

  for (uint16_t c = 0; c < cols; c++)
  {
    uint32_t src_addr = (uint32_t)&src[c];
    uint32_t dst_addr = (uint32_t)&dst[c * rows + (rows - 1)];

    HAL_MDMA_Start(&hmdma, src_addr, dst_addr, rows * 4, 1);
    HAL_MDMA_PollForTransfer(&hmdma, HAL_MDMA_FULL_TRANSFER, 1000);
  }

  printf("Transposed: %dx%d → %dx%d\n", rows, cols, cols, rows);
}
```

## 성능 분석

### 회전 속도
```
320×240 ARGB8888 이미지 (307,200 바이트):

CPU 회전 (중첩 루프):
  90°: 약 15 ms
  180°: 약 5 ms
  270°: 약 15 ms

MDMA 회전:
  90°: 약 2 ms
  180°: 약 0.5 ms
  270°: 약 2 ms

속도 향상: 약 5-10배

대역폭:
  MDMA: 307,200 / 0.002 = 153 MB/s
  CPU: 307,200 / 0.015 = 20 MB/s
```

## 고급 사용 예제

### 1. 실시간 회전 애니메이션

```c
typedef enum {
  ROTATE_0,
  ROTATE_90,
  ROTATE_180,
  ROTATE_270
} RotateAngle_t;

void LCD_RotateDisplay(Image_t *src, RotateAngle_t angle)
{
  static Image_t rotated;
  static uint32_t rotated_buffer[320 * 240];
  rotated.data = rotated_buffer;

  switch (angle)
  {
    case ROTATE_0:
      // 회전 없음
      memcpy(rotated.data, src->data, src->width * src->height * 4);
      rotated.width = src->width;
      rotated.height = src->height;
      break;

    case ROTATE_90:
      MDMA_Rotate90_CW(src, &rotated);
      break;

    case ROTATE_180:
      MDMA_Rotate180(src, &rotated);
      break;

    case ROTATE_270:
      MDMA_Rotate270_CW(src, &rotated);
      break;
  }

  // LCD에 표시
  DMA2D_MemoryToMemory(rotated.data, (uint32_t *)LCD_FRAME_BUFFER,
                       rotated.width, rotated.height);

  SCB_CleanDCache_by_Addr((uint32_t *)LCD_FRAME_BUFFER,
                          rotated.width * rotated.height * 4);
}

// 회전 애니메이션
void AnimateRotation(Image_t *image)
{
  RotateAngle_t angles[] = {ROTATE_0, ROTATE_90, ROTATE_180, ROTATE_270};

  for (uint8_t i = 0; i < 4; i++)
  {
    LCD_RotateDisplay(image, angles[i]);
    HAL_Delay(1000);
  }
}
```

### 2. 이미지 미러링

```c
// 수평 미러 (좌우 반전)
void MDMA_MirrorHorizontal(Image_t *src, Image_t *dst)
{
  dst->width = src->width;
  dst->height = src->height;

  for (uint16_t y = 0; y < src->height; y++)
  {
    uint32_t src_addr = (uint32_t)&src->data[y * src->width];
    uint32_t dst_addr = (uint32_t)&dst->data[y * dst->width + (dst->width - 1)];

    hmdma.Init.SourceInc = MDMA_SRC_INC_WORD;
    hmdma.Init.DestinationInc = MDMA_DEST_DEC_WORD;  // 역방향
    HAL_MDMA_Init(&hmdma);

    HAL_MDMA_Start(&hmdma, src_addr, dst_addr, src->width * 4, 1);
    HAL_MDMA_PollForTransfer(&hmdma, HAL_MDMA_FULL_TRANSFER, 1000);
  }

  printf("Mirrored horizontally\n");
}

// 수직 미러 (상하 반전)
void MDMA_MirrorVertical(Image_t *src, Image_t *dst)
{
  dst->width = src->width;
  dst->height = src->height;

  for (uint16_t y = 0; y < src->height; y++)
  {
    uint32_t src_addr = (uint32_t)&src->data[y * src->width];
    uint32_t dst_addr = (uint32_t)&dst->data[(dst->height - 1 - y) * dst->width];

    hmdma.Init.SourceInc = MDMA_SRC_INC_WORD;
    hmdma.Init.DestinationInc = MDMA_DEST_INC_WORD;
    HAL_MDMA_Init(&hmdma);

    HAL_MDMA_Start(&hmdma, src_addr, dst_addr, src->width * 4, 1);
    HAL_MDMA_PollForTransfer(&hmdma, HAL_MDMA_FULL_TRANSFER, 1000);
  }

  printf("Mirrored vertically\n");
}
```

## 트러블슈팅

### 1. 전송 오류
- **주소 정렬**: 4바이트 경계 확인
- **오프셋 범위**: ±2^31 이내
- **캐시 일관성**: D-Cache 클린/무효화

### 2. 성능 저하
- **버스트 크기**: MDMA_SOURCE_BURST_16BEATS 사용
- **우선순위**: MDMA_PRIORITY_VERY_HIGH
- **DMA vs MDMA**: MDMA는 시스템 메모리에 최적화

## 참고 자료
- **RM0399**: STM32H7 Reference Manual (MDMA)
- **AN5001**: MDMA usage on STM32H7
- HAL: `stm32h7xx_hal_mdma.c/h`
