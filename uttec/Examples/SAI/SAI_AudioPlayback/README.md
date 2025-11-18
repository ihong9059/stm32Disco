# SAI_AudioPlayback 예제

## 개요

STM32H745I-DISCO의 SAI(Serial Audio Interface)를 사용하여 WM8994 오디오 코덱을 통해 오디오 재생을 구현하는 예제입니다.

## 하드웨어

- **보드**: STM32H745I-DISCO
- **코덱**: WM8994 (I2S/SAI 인터페이스)
- **출력**: 헤드폰 잭, 스피커
- **LED**: LED1(재생 중), LED2(버퍼 전환), LED3(오류)

## 주요 코드

```c
/* SAI 초기화 */
SAI_HandleTypeDef hsai_BlockA1;
DMA_HandleTypeDef hdma_sai1_a;

void SAI_Init(void)
{
    /* SAI1 클록 활성화 */
    __HAL_RCC_SAI1_CLK_ENABLE();

    /* SAI 설정 */
    hsai_BlockA1.Instance = SAI1_Block_A;
    hsai_BlockA1.Init.AudioMode = SAI_MODEMASTER_TX;
    hsai_BlockA1.Init.Synchro = SAI_ASYNCHRONOUS;
    hsai_BlockA1.Init.OutputDrive = SAI_OUTPUTDRIVE_ENABLE;
    hsai_BlockA1.Init.NoDivider = SAI_MASTERDIVIDER_ENABLE;
    hsai_BlockA1.Init.FIFOThreshold = SAI_FIFOTHRESHOLD_1QF;
    hsai_BlockA1.Init.AudioFrequency = SAI_AUDIO_FREQUENCY_48K;
    hsai_BlockA1.Init.SynchroExt = SAI_SYNCEXT_DISABLE;
    hsai_BlockA1.Init.MonoStereoMode = SAI_STEREOMODE;
    hsai_BlockA1.Init.CompandingMode = SAI_NOCOMPANDING;
    hsai_BlockA1.Init.TriState = SAI_OUTPUT_NOTRELEASED;

    /* Frame 설정 */
    hsai_BlockA1.FrameInit.FrameLength = 64;
    hsai_BlockA1.FrameInit.ActiveFrameLength = 32;
    hsai_BlockA1.FrameInit.FSDefinition = SAI_FS_CHANNEL_IDENTIFICATION;
    hsai_BlockA1.FrameInit.FSPolarity = SAI_FS_ACTIVE_LOW;
    hsai_BlockA1.FrameInit.FSOffset = SAI_FS_BEFOREFIRSTBIT;

    /* Slot 설정 */
    hsai_BlockA1.SlotInit.FirstBitOffset = 0;
    hsai_BlockA1.SlotInit.SlotSize = SAI_SLOTSIZE_32B;
    hsai_BlockA1.SlotInit.SlotNumber = 2;
    hsai_BlockA1.SlotInit.SlotActive = SAI_SLOTACTIVE_0 | SAI_SLOTACTIVE_1;

    if (HAL_SAI_Init(&hsai_BlockA1) != HAL_OK)
    {
        Error_Handler();
    }
}

/* WM8994 코덱 초기화 */
void WM8994_Init(void)
{
    /* I2C를 통한 코덱 설정 */
    uint8_t wm8994_addr = 0x34;  /* 7-bit I2C 주소 */

    /* Software Reset */
    WM8994_WriteReg(wm8994_addr, 0x00, 0x0000);
    HAL_Delay(10);

    /* Power Management */
    WM8994_WriteReg(wm8994_addr, 0x01, 0x0003);  /* Enable VMID and VREF */
    WM8994_WriteReg(wm8994_addr, 0x02, 0x6240);  /* Enable Left/Right Headphone */

    /* Clock Configuration */
    WM8994_WriteReg(wm8994_addr, 0x200, 0x0001);  /* AIF1 Enable */
    WM8994_WriteReg(wm8994_addr, 0x208, 0x000A);  /* BCLK = MCLK/4 */
    WM8994_WriteReg(wm8994_addr, 0x210, 0x0083);  /* 48kHz, 16-bit */

    /* Audio Interface */
    WM8994_WriteReg(wm8994_addr, 0x300, 0x4010);  /* I2S mode, 16-bit */

    /* Volume Control */
    WM8994_WriteReg(wm8994_addr, 0x1C, 0x017F);  /* Left Headphone Volume */
    WM8994_WriteReg(wm8994_addr, 0x1D, 0x017F);  /* Right Headphone Volume */

    printf("WM8994 initialized\n");
}

/* DMA 초기화 */
void SAI_DMA_Init(void)
{
    __HAL_RCC_DMA2_CLK_ENABLE();

    hdma_sai1_a.Instance = DMA2_Stream1;
    hdma_sai1_a.Init.Request = DMA_REQUEST_SAI1_A;
    hdma_sai1_a.Init.Direction = DMA_MEMORY_TO_PERIPH;
    hdma_sai1_a.Init.PeriphInc = DMA_PINC_DISABLE;
    hdma_sai1_a.Init.MemInc = DMA_MINC_ENABLE;
    hdma_sai1_a.Init.PeriphDataAlignment = DMA_PDATAALIGN_HALFWORD;
    hdma_sai1_a.Init.MemDataAlignment = DMA_MDATAALIGN_HALFWORD;
    hdma_sai1_a.Init.Mode = DMA_CIRCULAR;
    hdma_sai1_a.Init.Priority = DMA_PRIORITY_HIGH;

    HAL_DMA_Init(&hdma_sai1_a);

    __HAL_LINKDMA(&hsai_BlockA1, hdmatx, hdma_sai1_a);

    /* DMA 인터럽트 */
    HAL_NVIC_SetPriority(DMA2_Stream1_IRQn, 5, 0);
    HAL_NVIC_EnableIRQ(DMA2_Stream1_IRQn);
}

/* 오디오 버퍼 (더블 버퍼링) */
#define AUDIO_BUFFER_SIZE  4096

int16_t audio_buffer[AUDIO_BUFFER_SIZE * 2];  /* 스테레오 */
uint32_t buffer_state = 0;

/* 오디오 재생 시작 */
void Audio_Play(int16_t *data, uint32_t size)
{
    /* DMA를 통한 데이터 전송 */
    HAL_SAI_Transmit_DMA(&hsai_BlockA1, (uint8_t*)data, size);
    BSP_LED_On(LED1);
}

/* DMA Half Transfer 콜백 */
void HAL_SAI_TxHalfCpltCallback(SAI_HandleTypeDef *hsai)
{
    /* 버퍼 전반부 업데이트 */
    buffer_state = 0;
    BSP_LED_Toggle(LED2);

    /* 새 데이터 채우기 */
    Fill_Audio_Buffer(audio_buffer, AUDIO_BUFFER_SIZE);
}

/* DMA Full Transfer 콜백 */
void HAL_SAI_TxCpltCallback(SAI_HandleTypeDef *hsai)
{
    /* 버퍼 후반부 업데이트 */
    buffer_state = 1;
    BSP_LED_Toggle(LED2);

    /* 새 데이터 채우기 */
    Fill_Audio_Buffer(&audio_buffer[AUDIO_BUFFER_SIZE], AUDIO_BUFFER_SIZE);
}

/* 사인파 생성 예제 */
void Fill_Audio_Buffer(int16_t *buffer, uint32_t size)
{
    static float phase = 0.0f;
    float freq = 440.0f;  /* A4 음 */
    float sample_rate = 48000.0f;
    float amplitude = 16000.0f;

    for (uint32_t i = 0; i < size; i += 2)
    {
        float sample = amplitude * sinf(2.0f * M_PI * phase);

        buffer[i] = (int16_t)sample;      /* Left channel */
        buffer[i + 1] = (int16_t)sample;  /* Right channel */

        phase += freq / sample_rate;
        if (phase >= 1.0f)
            phase -= 1.0f;
    }
}

/* 메인 */
int main(void)
{
    HAL_Init();
    SystemClock_Config();

    BSP_LED_Init(LED1);
    BSP_LED_Init(LED2);
    BSP_LED_Init(LED3);

    /* I2C 초기화 (코덱 제어용) */
    I2C_Init();

    /* SAI 초기화 */
    SAI_Init();
    SAI_DMA_Init();

    /* WM8994 코덱 초기화 */
    WM8994_Init();

    printf("Audio playback example\n");

    /* 초기 버퍼 채우기 */
    Fill_Audio_Buffer(audio_buffer, AUDIO_BUFFER_SIZE * 2);

    /* 재생 시작 */
    Audio_Play(audio_buffer, AUDIO_BUFFER_SIZE * 2);

    while (1)
    {
        /* 메인 루프 */
        HAL_Delay(100);
    }
}
```

## WAV 파일 재생

```c
/* WAV 파일 헤더 */
typedef struct
{
    uint32_t chunk_id;
    uint32_t chunk_size;
    uint32_t format;
    uint32_t subchunk1_id;
    uint32_t subchunk1_size;
    uint16_t audio_format;
    uint16_t num_channels;
    uint32_t sample_rate;
    uint32_t byte_rate;
    uint16_t block_align;
    uint16_t bits_per_sample;
    uint32_t subchunk2_id;
    uint32_t subchunk2_size;
} WAV_Header_t;

/* WAV 재생 */
void Play_WAV_File(uint8_t *wav_data, uint32_t wav_size)
{
    WAV_Header_t *header = (WAV_Header_t*)wav_data;

    printf("WAV File Info:\n");
    printf("  Sample Rate: %lu Hz\n", header->sample_rate);
    printf("  Channels: %u\n", header->num_channels);
    printf("  Bits/Sample: %u\n", header->bits_per_sample);

    /* 오디오 데이터 시작 위치 */
    uint8_t *audio_data = wav_data + sizeof(WAV_Header_t);
    uint32_t audio_size = header->subchunk2_size;

    /* 재생 */
    Audio_Play((int16_t*)audio_data, audio_size / 2);
}
```

## 참고

- Reference Manual: SAI 챕터
- WM8994 데이터시트
- I2S/SAI 프로토콜
