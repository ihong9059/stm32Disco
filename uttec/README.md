# STM32H745I-DISCO 예제 및 애플리케이션 설명서

이 문서는 STM32H745I-DISCO 개발보드를 위한 모든 예제 프로젝트와 애플리케이션에 대한 한글 설명을 제공합니다.

## 목차

1. [보드 개요](#보드-개요)
2. [Templates (템플릿)](#templates)
3. [Examples (예제)](#examples)
4. [Applications (애플리케이션)](#applications)
5. [Demonstrations (데모)](#demonstrations)

---

## 보드 개요

**STM32H745I-DISCO**는 듀얼 코어 마이크로컨트롤러 개발보드입니다.

### 주요 사양

- **CPU**:
  - Cortex-M7 (400MHz) - 메인 프로세서
  - Cortex-M4 (200MHz) - 보조 프로세서

- **메모리**:
  - Flash: 2MB (듀얼 뱅크)
  - ITCM-RAM: 64KB (M7 제로 웨이트 스테이트)
  - DTCM-RAM: 128KB (M7 고속 데이터 액세스)
  - D1 AXI-SRAM: 512KB (@0x24000000)
  - D2 SRAM: 288KB
  - D3 SRAM: 64KB (@0x38000000, BDMA 액세스 가능)

- **클럭 설정**:
  - D1 Domain (AXI/AHB3): 200MHz
  - D2 Domain (AHB1/AHB2): 200MHz
  - D3 Domain (AHB4): 200MHz
  - APB 주변장치: 100MHz

- **주요 페리페럴**:
  - LCD-TFT 디스플레이 (480x272)
  - 터치스크린
  - eMMC 카드
  - QSPI 플래시
  - SDRAM
  - USB OTG FS
  - Audio (SAI + WM8994)
  - CAN FD
  - 등등

### 듀얼 코어 부팅 방식

STM32H745는 다양한 부팅 구성을 지원합니다:

1. **Both cores boot together**: CM7과 CM4가 동시에 부팅 (기본값)
2. **CM7 boots, CM4 gated**: CM7만 부팅, CM4는 나중에 활성화
3. **CM4 boots, CM7 gated**: CM4만 부팅, CM7은 나중에 활성화

부팅 설정은 User Option Bytes (BCM7, BCM4)로 제어됩니다.

---

## Templates

템플릿 프로젝트는 새로운 프로젝트를 시작할 때 사용하는 기본 골격입니다.

### 1. [Templates_LL](./Templates_LL/README.md)
Low-Level (LL) API를 사용한 듀얼 코어 템플릿

### 2. Boot Configuration Templates
- [BootCM4_CM7](./Templates/BootCM4_CM7/README.md) - 양쪽 코어 동시 부팅
- [BootCM4_CM7Gated](./Templates/BootCM4_CM7Gated/README.md) - CM4 부팅, CM7 게이트
- [BootCM7_CM4Gated](./Templates/BootCM7_CM4Gated/README.md) - CM7 부팅, CM4 게이트
- [BootCM7_CM4Gated_RAM](./Templates/BootCM7_CM4Gated_RAM/README.md) - CM7 부팅, CM4 RAM 실행

---

## Examples

### 주변장치별 예제

#### 통신 인터페이스
- [UART - Wake from STOP using FIFO](./Examples/UART/UART_WakeUpFromStopUsingFIFO/README.md)
- [FDCAN - Classic Frame Networking](./Examples/FDCAN/FDCAN_Classic_Frame_Networking/README.md)
- [FDCAN - Image Transmission](./Examples/FDCAN/FDCAN_Image_transmission/README.md)

#### 디스플레이 및 그래픽
- [LTDC - 1 Layer Display](./Examples/LTDC/LTDC_Display_1Layer/README.md)
- [LTDC - 2 Layers Display](./Examples/LTDC/LTDC_Display_2Layers/README.md)
- [DMA2D - Memory to Memory with Blending](./Examples/DMA2D/DMA2D_MemToMemWithBlending/README.md)
- [JPEG - Decoding from Flash with DMA](./Examples/JPEG/JPEG_DecodingFromFLASH_DMA/README.md)

#### 메모리
- [FMC - SDRAM Data Memory](./Examples/FMC/FMC_SDRAM_DataMemory/README.md)
- [QSPI - Memory Mapped Dual Mode](./Examples/QSPI/QSPI_MemoryMappedDual/README.md)
- [MMC - Read/Write with DMA](./Examples/MMC/MMC_ReadWrite_DMA/README.md)
- [MMC - Read/Write with IT](./Examples/MMC/MMC_ReadWrite_IT/README.md)
- [FLASH - Core Configuration](./Examples/FLASH/FLASH_CoreConfiguration/README.md)

#### DMA 및 데이터 전송
- [DMA - DMAMUX Request Generator](./Examples/DMA/DMAMUX_RequestGen/README.md)
- [MDMA - Repeat Block Rotation](./Examples/MDMA/MDMA_RepeatBlock_Rotation/README.md)
- [MDMA - Repeat Block ZoomOut](./Examples/MDMA/MDMA_RepeatBlock_ZoomOut/README.md)

#### 아날로그
- [ADC - Dual Mode Interleaved](./Examples/ADC/ADC_DualModeInterleaved/README.md)
- [ADC - Regular and Injected Groups](./Examples/ADC/ADC_Regular_injected_groups/README.md)
- [DAC - Signals Generation](./Examples/DAC/DAC_SignalsGeneration/README.md)

#### 타이머
- [TIM - DMA](./Examples/TIM/TIM_DMA/README.md)
- [LPTIM - Pulse Counter](./Examples/LPTIM/LPTIM_PulseCounter/README.md)
- [HRTIM - Multiple PWM](./Examples/HRTIM/HRTIM_MultiplePWM/README.md)
- [RTC - Alarm](./Examples/RTC/RTC_Alarm/README.md)

#### 전력 관리
- [PWR - STANDBY](./Examples/PWR/PWR_STANDBY/README.md)
- [PWR - STANDBY with RTC](./Examples/PWR/PWR_STANDBY_RTC/README.md)
- [PWR - STOP with RTC](./Examples/PWR/PWR_STOP_RTC/README.md)
- [PWR - Hold Mechanism](./Examples/PWR/PWR_Hold_Mechanism/README.md)
- [PWR - Domain3 System Control](./Examples/PWR/PWR_Domain3SystemControl/README.md)
- [PWR - D1 ON D2 OFF](./Examples/PWR/PWR_D1ON_D2OFF/README.md)
- [PWR - D2 ON D1 OFF](./Examples/PWR/PWR_D2ON_D1OFF/README.md)

#### 보안 및 유틸리티
- [CRC - User Defined Polynomial](./Examples/CRC/CRC_UserDefinedPolynomial/README.md)
- [RNG - Multi RNG](./Examples/RNG/RNG_MultiRNG/README.md)
- [WWDG - Example](./Examples/WWDG/WWDG_Example/README.md)
- [IWDG - Window Mode](./Examples/IWDG/IWDG_WindowMode/README.md)

#### 멀티코어 관리
- [HSEM - Core Sync](./Examples/HSEM/HSEM_CoreSync/README.md)
- [HSEM - Core Notification](./Examples/HSEM/HSEM_CoreNotification/README.md)
- [HSEM - Resource Sharing](./Examples/HSEM/HSEM_ResourceSharing/README.md)

#### 기타
- [GPIO - EXTI](./Examples/GPIO/GPIO_EXTI/README.md)
- [RCC - Clock Config](./Examples/RCC/RCC_ClockConfig/README.md)
- [SAI - Audio Playback](./Examples/SAI/SAI_AudioPlayback/README.md)
- [BSP - Board Support Package](./Examples/BSP/README.md)

---

## Applications

### RTOS 애플리케이션
- [FreeRTOS - AMP Dual RTOS](./Applications/FreeRTOS/FreeRTOS_AMP_Dual_RTOS/README.md)
- [FreeRTOS - AMP RTOS BareMetal](./Applications/FreeRTOS/FreeRTOS_AMP_RTOS_BareMetal/README.md)

### 멀티코어 통신
- [OpenAMP - PingPong](./Applications/OpenAMP/OpenAMP_PingPong/README.md)
- [OpenAMP - RTOS PingPong](./Applications/OpenAMP/OpenAMP_RTOS_PingPong/README.md)

### 리소스 관리
- [ResourcesManager - Shared Resources](./Applications/ResourcesManager/ResourcesManager_SharedResources/README.md)
- [ResourcesManager - Usage with Notification](./Applications/ResourcesManager/ResourcesManager_UsageWithNotification/README.md)

### 파일 시스템
- [FatFs - Shared Device](./Applications/FatFs/FatFs_Shared_Device/README.md)

### GUI
- [STemWin - HelloWorld](./Applications/STemWin/STemWin_HelloWorld/README.md)
- [STemWin - Sample Demo](./Applications/STemWin/STemWin_SampleDemo/README.md)

### USB
- [USB Device - HID Standalone](./Applications/USB_Device/HID_Standalone/README.md)
- [USB Device - DFU Standalone](./Applications/USB_Device/DFU_Standalone/README.md)
- [USB Host - HID Standalone](./Applications/USB_Host/HID_Standalone/README.md)
- [USB Host - MSC Standalone](./Applications/USB_Host/MSC_Standalone/README.md)

---

## Demonstrations

- [메인 데모 애플리케이션](./Demonstrations/README.md) - 오실로스코프/신호 발생기, CoreMark 벤치마크 등

---

## 개발 환경

### 지원 툴체인

모든 예제는 다음 개발환경을 지원합니다:

1. **STM32CubeIDE** (GCC 기반, 무료)
2. **IAR EWARM** (상용)
3. **Keil MDK-ARM** (상용)

### 빌드 및 플래시

STM32CubeProgrammer를 사용하여 펌웨어를 보드에 업로드할 수 있습니다.

---

## 라이선스

모든 예제는 ST Microelectronics의 라이선스 조건에 따릅니다.

---

**문의사항이나 기술지원이 필요하신 경우 ST Microelectronics 공식 웹사이트를 참조하세요.**
