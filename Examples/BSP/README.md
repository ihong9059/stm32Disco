# BSP (Board Support Package) - 보드 주변장치 통합 예제

## 개요

BSP 예제는 STM32H745I-DISCO 보드의 모든 주변장치를 테스트할 수 있는 종합적인 데모 프로그램입니다. 터치스크린, LCD, 오디오, MMC 카드, SDRAM, QSPI 플래시 등 보드의 모든 기능을 확인할 수 있습니다.

## 주요 기능

### 1. LCD 및 터치스크린
- **LCD 디스플레이**: 480x272 TFT LCD
- **터치스크린**: 정전식 터치 컨트롤러
- **그래픽 기능**: 텍스트, 도형, 비트맵 표시
- **2가지 터치 모드**: 폴링 방식 / 인터럽트 방식

### 2. 오디오
- **오디오 재생**: WAV 파일 재생 (DMA 순환 모드)
- **오디오 녹음**: 마이크로폰 입력 녹음
- **코덱**: WM8994 (I2C 제어, I2S/SAI 데이터)

### 3. 스토리지
- **eMMC 카드**: 읽기/쓰기 테스트
- **QSPI 플래시**: 외부 플래시 메모리 액세스
- **SDRAM**: IS42S32800G (8MB)

### 4. 사용자 인터페이스
- **LED 제어**: 4개 LED (LED1~LED4)
- **버튼**: Wakeup/Tamper 버튼

## 하드웨어 구성

```
┌─────────────────────────────────────────┐
│      STM32H745I-DISCO Board             │
│                                         │
│  ┌──────────┐         ┌──────────┐     │
│  │  CM7     │         │  CM4     │     │
│  │ (400MHz) │         │ (200MHz) │     │
│  └────┬─────┘         └────┬─────┘     │
│       │                    │            │
│  ┌────┴────────────────────┴─────┐     │
│  │      Peripherals               │     │
│  │  - LTDC → LCD (480x272)        │     │
│  │  - I2C  → Touch (FT5336)       │     │
│  │  - SAI  → Audio (WM8994)       │     │
│  │  - SDMMC → eMMC                │     │
│  │  - QSPI → Flash (MX25LM51245G) │     │
│  │  - FMC  → SDRAM (IS42S32800G)  │     │
│  └────────────────────────────────┘     │
└─────────────────────────────────────────┘
```

## 메뉴 구조

### 메인 메뉴
```
┌───────────────────────────────────┐
│  STM32H745I-DISCO BSP Example     │
├───────────────────────────────────┤
│  1. Touchscreen Demo              │
│  2. LCD Features                  │
│  3. Audio Playback                │
│  4. Audio Record                  │
│  5. MMC Test                      │
│  6. SDRAM Test                    │
│  7. QSPI Test                     │
│  8. LED Control                   │
└───────────────────────────────────┘
```

## 코드 구조

### 프로젝트 구조
```
BSP/
├── CM4/              # Cortex-M4 코드
│   ├── Inc/
│   │   └── main.h
│   └── Src/
│       └── main.c
├── CM7/              # Cortex-M7 코드 (메인 애플리케이션)
│   ├── Inc/
│   │   ├── main.h
│   │   └── stm32h7xx_it.h
│   └── Src/
│       ├── main.c
│       ├── touchscreen.c
│       ├── lcd.c
│       ├── audio.c
│       ├── mmc.c
│       ├── sdram.c
│       └── qspi.c
├── Common/           # 공유 코드
│   ├── Inc/
│   └── Src/
└── Binary/           # 오디오 샘플 파일
    └── audio_sample_tdm.bin
```

## 기능별 상세 설명

### 1. Touchscreen Demo

#### 폴링 모드
```c
void Touchscreen_Polling_Demo(void)
{
  TS_State_t TS_State;

  // 터치스크린 초기화
  BSP_TS_Init(0, TS_ORIENTATION_LANDSCAPE);

  while (1)
  {
    // 터치 상태 읽기
    BSP_TS_GetState(0, &TS_State);

    if (TS_State.TouchDetected)
    {
      // 터치 포인트 개수
      for (uint32_t i = 0; i < TS_State.TouchDetected; i++)
      {
        uint16_t x = TS_State.TouchX[i];
        uint16_t y = TS_State.TouchY[i];

        // 터치 포인트에 원 그리기
        BSP_LCD_FillCircle(0, x, y, 10, LCD_COLOR_RED);

        printf("Touch %lu: X=%u, Y=%u\n", i, x, y);
      }
    }

    HAL_Delay(10);
  }
}
```

#### 인터럽트 모드
```c
volatile uint8_t touch_detected = 0;

void Touchscreen_Interrupt_Demo(void)
{
  // 터치 인터럽트 활성화
  BSP_TS_Init(0, TS_ORIENTATION_LANDSCAPE);
  BSP_TS_EnableIT(0);

  while (1)
  {
    if (touch_detected)
    {
      TS_State_t TS_State;
      BSP_TS_GetState(0, &TS_State);

      // 터치 처리
      Process_Touch(&TS_State);

      touch_detected = 0;
    }

    // 다른 작업 수행 가능
  }
}

// 터치 인터럽트 콜백
void BSP_TS_Callback(uint32_t Instance)
{
  touch_detected = 1;
}
```

### 2. LCD Features

#### 기본 도형 그리기
```c
void LCD_Features_Demo(void)
{
  // LCD 초기화
  BSP_LCD_Init(0, LCD_ORIENTATION_LANDSCAPE);
  BSP_LCD_SetLayerVisible(0, 0, ENABLE);

  // 배경 지우기
  BSP_LCD_Clear(0, LCD_COLOR_WHITE);

  // 선 그리기
  BSP_LCD_SetTextColor(0, LCD_COLOR_BLUE);
  BSP_LCD_DrawHLine(0, 50, 50, 200);
  BSP_LCD_DrawVLine(0, 100, 50, 100);

  // 사각형 그리기
  BSP_LCD_SetTextColor(0, LCD_COLOR_RED);
  BSP_LCD_DrawRect(0, 150, 50, 100, 80);
  BSP_LCD_FillRect(0, 170, 70, 60, 40, LCD_COLOR_RED);

  // 원 그리기
  BSP_LCD_SetTextColor(0, LCD_COLOR_GREEN);
  BSP_LCD_DrawCircle(0, 350, 100, 50);
  BSP_LCD_FillCircle(0, 350, 200, 30, LCD_COLOR_GREEN);

  // 텍스트 출력
  BSP_LCD_SetFont(0, &Font24);
  BSP_LCD_SetTextColor(0, LCD_COLOR_BLACK);
  BSP_LCD_SetBackColor(0, LCD_COLOR_WHITE);
  BSP_LCD_DisplayStringAt(0, 50, 220, (uint8_t *)"STM32H745I-DISCO", CENTER_MODE);
}
```

#### 비트맵 표시
```c
// RGB565 비트맵 표시
void Display_Bitmap(uint16_t x, uint16_t y, uint8_t *bitmap)
{
  BSP_LCD_DrawBitmap(0, x, y, bitmap);
}
```

### 3. Audio Playback

```c
// 오디오 샘플 주소 (Flash에 프리로드)
#define AUDIO_SAMPLE_ADDRESS  0x08080000

void Audio_Playback_Demo(void)
{
  BSP_AUDIO_Init_t AudioInit;

  // 오디오 출력 초기화
  AudioInit.Device = AUDIO_OUT_DEVICE_HEADPHONE;
  AudioInit.ChannelsNbr = 2;  // 스테레오
  AudioInit.SampleRate = AUDIO_FREQUENCY_48K;
  AudioInit.BitsPerSample = AUDIO_RESOLUTION_16B;
  AudioInit.Volume = 70;

  BSP_AUDIO_OUT_Init(0, &AudioInit);

  // 오디오 재생 시작 (DMA 순환 모드)
  BSP_AUDIO_OUT_Play(0, (uint8_t *)AUDIO_SAMPLE_ADDRESS, AUDIO_SAMPLE_SIZE);

  printf("Playing audio...\n");

  while (1)
  {
    // 볼륨 조절 (버튼으로)
    if (volume_up_pressed)
    {
      AudioInit.Volume = MIN(100, AudioInit.Volume + 10);
      BSP_AUDIO_OUT_SetVolume(0, AudioInit.Volume);
    }
    if (volume_down_pressed)
    {
      AudioInit.Volume = MAX(0, AudioInit.Volume - 10);
      BSP_AUDIO_OUT_SetVolume(0, AudioInit.Volume);
    }
  }
}

// 오디오 재생 완료 콜백
void BSP_AUDIO_OUT_TransferComplete_CallBack(uint32_t Instance)
{
  // 순환 모드에서는 자동으로 재시작
  printf("Audio transfer complete\n");
}
```

### 4. Audio Record

```c
#define RECORD_BUFFER_SIZE  (48000 * 2 * 2)  // 1초, 스테레오, 16비트

uint8_t record_buffer[RECORD_BUFFER_SIZE];

void Audio_Record_Demo(void)
{
  BSP_AUDIO_Init_t AudioInit;

  // 오디오 입력 초기화
  AudioInit.Device = AUDIO_IN_DEVICE_DIGITAL_MIC;  // 디지털 마이크
  AudioInit.ChannelsNbr = 2;
  AudioInit.SampleRate = AUDIO_FREQUENCY_48K;
  AudioInit.BitsPerSample = AUDIO_RESOLUTION_16B;
  AudioInit.Volume = 70;

  BSP_AUDIO_IN_Init(0, &AudioInit);

  // 녹음 시작
  printf("Recording...\n");
  BSP_AUDIO_IN_Record(0, (uint8_t *)record_buffer, RECORD_BUFFER_SIZE);

  // 녹음 완료 대기
  while (record_complete == 0);

  printf("Recording complete. Playing back...\n");

  // 녹음한 내용 재생
  BSP_AUDIO_OUT_Init(0, &AudioInit);
  BSP_AUDIO_OUT_Play(0, record_buffer, RECORD_BUFFER_SIZE);

  while (1);
}

void BSP_AUDIO_IN_TransferComplete_CallBack(uint32_t Instance)
{
  record_complete = 1;
}
```

### 5. MMC Test

```c
void MMC_Test(void)
{
  uint8_t write_buffer[512];
  uint8_t read_buffer[512];
  uint32_t block_address = 0;

  // MMC 카드 초기화
  if (BSP_MMC_Init(0) != BSP_ERROR_NONE)
  {
    printf("MMC Init Failed!\n");
    Error_Handler();
  }

  // 쓰기 데이터 준비
  for (int i = 0; i < 512; i++)
  {
    write_buffer[i] = i & 0xFF;
  }

  // 블록 쓰기
  printf("Writing to MMC...\n");
  if (BSP_MMC_WriteBlocks(0, write_buffer, block_address, 1) != BSP_ERROR_NONE)
  {
    printf("MMC Write Failed!\n");
    Error_Handler();
  }

  // 블록 읽기
  printf("Reading from MMC...\n");
  if (BSP_MMC_ReadBlocks(0, read_buffer, block_address, 1) != BSP_ERROR_NONE)
  {
    printf("MMC Read Failed!\n");
    Error_Handler();
  }

  // 검증
  if (memcmp(write_buffer, read_buffer, 512) == 0)
  {
    printf("MMC Test PASSED!\n");
    BSP_LED_On(LED1);
  }
  else
  {
    printf("MMC Test FAILED!\n");
    BSP_LED_On(LED2);
  }
}
```

### 6. SDRAM Test

```c
#define SDRAM_WRITE_READ_ADDR  0xD0000000
#define BUFFER_SIZE            0x1000

void SDRAM_Test(void)
{
  uint32_t *sdram_ptr = (uint32_t *)SDRAM_WRITE_READ_ADDR;

  // SDRAM 초기화
  if (BSP_SDRAM_Init(0) != BSP_ERROR_NONE)
  {
    printf("SDRAM Init Failed!\n");
    Error_Handler();
  }

  // 쓰기 테스트
  printf("Writing to SDRAM...\n");
  for (uint32_t i = 0; i < BUFFER_SIZE; i++)
  {
    sdram_ptr[i] = 0xA5A5A5A5 + i;
  }

  // 읽기 및 검증
  printf("Reading from SDRAM...\n");
  for (uint32_t i = 0; i < BUFFER_SIZE; i++)
  {
    if (sdram_ptr[i] != (0xA5A5A5A5 + i))
    {
      printf("SDRAM Test FAILED at address 0x%08lX\n", (uint32_t)&sdram_ptr[i]);
      BSP_LED_On(LED2);
      return;
    }
  }

  printf("SDRAM Test PASSED!\n");
  BSP_LED_On(LED1);
}

// DMA를 사용한 SDRAM 테스트
void SDRAM_DMA_Test(void)
{
  uint32_t src_buffer[BUFFER_SIZE];
  uint32_t *sdram_dst = (uint32_t *)SDRAM_WRITE_READ_ADDR;

  // 소스 버퍼 초기화
  for (uint32_t i = 0; i < BUFFER_SIZE; i++)
  {
    src_buffer[i] = i;
  }

  // DMA로 SDRAM에 복사
  HAL_DMA_Start(&hdma_memtomem, (uint32_t)src_buffer,
                (uint32_t)sdram_dst, BUFFER_SIZE);
  HAL_DMA_PollForTransfer(&hdma_memtomem, HAL_DMA_FULL_TRANSFER, HAL_MAX_DELAY);

  // 검증
  if (memcmp(src_buffer, sdram_dst, BUFFER_SIZE * 4) == 0)
  {
    printf("SDRAM DMA Test PASSED!\n");
  }
}
```

### 7. QSPI Flash Test

```c
#define QSPI_BASE_ADDRESS  0x90000000

void QSPI_Test(void)
{
  uint8_t write_data[256];
  uint8_t read_data[256];
  uint32_t address = 0;

  // QSPI 초기화
  if (BSP_QSPI_Init(0) != BSP_ERROR_NONE)
  {
    printf("QSPI Init Failed!\n");
    Error_Handler();
  }

  // 메모리 맵 모드 활성화
  if (BSP_QSPI_EnableMemoryMappedMode(0) != BSP_ERROR_NONE)
  {
    printf("QSPI Memory Mapped Mode Failed!\n");
    Error_Handler();
  }

  // 메모리 맵 모드에서 읽기
  uint8_t *qspi_ptr = (uint8_t *)QSPI_BASE_ADDRESS;
  memcpy(read_data, qspi_ptr, 256);

  printf("QSPI Flash first 16 bytes:\n");
  for (int i = 0; i < 16; i++)
  {
    printf("0x%02X ", read_data[i]);
  }
  printf("\n");

  // 쓰기 테스트 (먼저 erase 필요)
  BSP_QSPI_DisableMemoryMappedMode(0);

  // 섹터 지우기
  BSP_QSPI_EraseBlock(0, address, BSP_QSPI_ERASE_4K);

  // 데이터 쓰기
  for (int i = 0; i < 256; i++)
  {
    write_data[i] = i;
  }
  BSP_QSPI_Write(0, write_data, address, 256);

  // 읽기 및 검증
  BSP_QSPI_Read(0, read_data, address, 256);

  if (memcmp(write_data, read_data, 256) == 0)
  {
    printf("QSPI Test PASSED!\n");
    BSP_LED_On(LED1);
  }
  else
  {
    printf("QSPI Test FAILED!\n");
    BSP_LED_On(LED2);
  }
}
```

## 오디오 샘플 프리로드

BSP 예제를 실행하려면 오디오 샘플 파일을 Flash에 미리 로드해야 합니다.

### STM32CubeProgrammer 사용
```bash
# 오디오 샘플을 0x08080000에 프로그램
STM32_Programmer_CLI -c port=SWD -w audio_sample_tdm.bin 0x08080000 -v
```

### 빌드 후 자동 프로그래밍
```makefile
# Makefile에 추가
flash_audio:
	STM32_Programmer_CLI -c port=SWD -w Binary/audio_sample_tdm.bin 0x08080000 -v
```

## 메모리 요구사항

### Flash
```
0x08000000  - CM7 코드
0x08040000  - CM4 코드
0x08080000  - 오디오 샘플 (필수!)
```

### RAM
- SDRAM (0xD0000000): LCD 프레임버퍼
- AXI-SRAM: 오디오 버퍼, MMC 버퍼

## 빌드 및 실행

### STM32CubeIDE
1. 프로젝트 임포트: `BSP/STM32CubeIDE`
2. CM7 프로젝트 빌드
3. CM4 프로젝트 빌드
4. 오디오 샘플 프리로드 (중요!)
5. Debug/Run

### 실행 순서
1. 보드 전원 인가
2. 프로그램 실행
3. LCD에 메뉴 표시
4. 터치스크린으로 메뉴 선택

## LED 표시

- **LED1** (Green): 테스트 성공
- **LED2** (Orange): 테스트 실패/오류
- **LED3** (Red): CM4 동작 중
- **LED4** (Blue): 예약

## 트러블슈팅

### 터치스크린이 반응하지 않는 경우
- I2C 연결 확인
- FT5336 드라이버 초기화 확인
- 터치스크린 보호 필름 제거 확인

### 오디오가 재생되지 않는 경우
- 오디오 샘플이 0x08080000에 프로그램되었는지 확인
- SAI 클럭 설정 확인
- WM8994 코덱 초기화 확인

### LCD에 아무것도 표시되지 않는 경우
- SDRAM 초기화 확인
- LTDC 클럭 및 타이밍 확인
- LCD 전원 확인

## 참고 자료

- BSP 드라이버: `Drivers/BSP/STM32H745I-DISCO`
- UM2411: STM32H745I-DISCO board user manual
- 예제 코드: `STM32H745I-DISCO/Examples/BSP`
