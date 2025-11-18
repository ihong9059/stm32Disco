# USB_Device DFU_Standalone - USB 장치 펌웨어 업데이트

## 개요

이 애플리케이션은 USB DFU (Device Firmware Upgrade) 클래스를 구현하여 USB를 통해 펌웨어를 다운로드하고 업데이트할 수 있는 예제입니다. STM32CubeProgrammer 또는 dfu-util을 사용하여 펌웨어를 업데이트할 수 있습니다.

## DFU란?

**DFU (Device Firmware Upgrade)**는 USB를 통해 장치의 펌웨어를 업데이트하기 위한 USB 디바이스 클래스 사양입니다.

### 주요 특징
- **표준화된 프로토콜**: USB-IF DFU 1.1 사양
- **드라이버 지원**: Windows, Linux, macOS에서 지원
- **양방향 지원**: 다운로드(쓰기) 및 업로드(읽기)

## 시스템 아키텍처

### DFU 동작 흐름
```
┌─────────────────────────────────────────────────────────────┐
│                     STM32H745I-DISCO                        │
│                                                             │
│  ┌─────────────────┐                                        │
│  │  Application    │                                        │
│  │  (User Code)    │                                        │
│  └────────┬────────┘                                        │
│           │ DFU Mode Entry                                  │
│           ▼                                                 │
│  ┌─────────────────┐      ┌──────────────┐      ┌────────┐  │
│  │  DFU Bootloader │◄────►│  USB DFU     │◄────►│ USB FS │  │
│  │                 │      │  Class       │      │ (CN13) │  │
│  │  - Download     │      │              │      │        │  │
│  │  - Upload       │      │  - Control   │      │        │  │
│  │  - Manifest     │      │    Transfer  │      │        │  │
│  │  - Detach       │      │              │      │        │  │
│  └────────┬────────┘      └──────────────┘      └───┬────┘  │
│           │                                         │       │
│           ▼                                         │       │
│  ┌─────────────────┐                                │       │
│  │  Internal Flash │                                │       │
│  │  (Programming)  │                                │       │
│  └─────────────────┘                                │       │
└─────────────────────────────────────────────────────┼───────┘
                                                      │
                                               USB Connection
                                                      │
                                              ┌───────▼───────┐
                                              │  Host PC      │
                                              │  (Programmer) │
                                              └───────────────┘
```

## DFU 상태 머신

### DFU 상태 다이어그램
```
                         ┌─────────┐
                   ┌────►│ appIDLE │◄───────────┐
                   │     └────┬────┘            │
                   │          │ DFU_DETACH      │ USB Reset
                   │          ▼                 │
                   │     ┌─────────┐            │
                   │     │appDETACH│────────────┤
                   │     └────┬────┘            │
                   │          │ Timeout         │
                   │          ▼                 │
┌─────────┐       │     ┌─────────┐            │
│ dfuERROR│◄──────┼─────│ dfuIDLE │◄───────────┘
└────┬────┘       │     └────┬────┘
     │            │          │ DFU_DNLOAD
     │ CLRSTATUS  │          ▼
     └────────────┤     ┌──────────┐
                  │     │dfuDNLOAD-│
                  │     │  SYNC    │
                  │     └────┬─────┘
                  │          │
                  │          ▼
                  │     ┌──────────┐
                  │     │dfuDNBUSY │
                  │     └────┬─────┘
                  │          │
                  │          ▼
                  │     ┌──────────┐
                  └─────│dfuDNLOAD-│
                        │  IDLE    │
                        └──────────┘
```

## 메모리 맵

### Flash 메모리 레이아웃
```
0x08000000  ┌─────────────────────────┐
            │  DFU Bootloader         │
            │  (Sector 0-1)           │
            │  128KB                  │
0x08020000  ├─────────────────────────┤
            │                         │
            │  Application            │
            │  (Sector 2-7)           │
            │  768KB                  │
            │                         │
0x080E0000  ├─────────────────────────┤
            │  User Data              │
            │  (Sector 7)             │
            │  128KB                  │
0x08100000  └─────────────────────────┘
```

### DFU 메모리 영역 정의
```c
// usbd_dfu_flash.c
#define FLASH_DESC_STR      "@Internal Flash   /0x08020000/01*128Ka,03*128Kg"

// 문자열 형식 설명:
// @Internal Flash    : 장치 이름
// /0x08020000        : 시작 주소
// /01*128Ka          : 1개의 128KB 섹터 (읽기 전용)
// ,03*128Kg          : 3개의 128KB 섹터 (읽기/쓰기)
```

## DFU 모드 진입

### 1. 하드웨어 트리거 (USER 버튼)
```c
// DFU 부트로더에서 체크
#define DFU_BUTTON_PORT  GPIOC
#define DFU_BUTTON_PIN   GPIO_PIN_13

void check_dfu_mode(void)
{
  // USER 버튼 GPIO 초기화
  __HAL_RCC_GPIOC_CLK_ENABLE();

  GPIO_InitTypeDef GPIO_InitStruct = {0};
  GPIO_InitStruct.Pin = DFU_BUTTON_PIN;
  GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  HAL_GPIO_Init(DFU_BUTTON_PORT, &GPIO_InitStruct);

  // 버튼이 눌린 상태면 DFU 모드 유지
  if (HAL_GPIO_ReadPin(DFU_BUTTON_PORT, DFU_BUTTON_PIN) == GPIO_PIN_SET)
  {
    // DFU 모드 시작
    DFU_Start();
  }
  else
  {
    // 애플리케이션으로 점프
    Jump_To_Application();
  }
}
```

### 2. 소프트웨어 트리거 (백업 레지스터)
```c
// 애플리케이션에서 DFU 요청
#define DFU_REQUEST_FLAG  0x12345678
#define DFU_BACKUP_REG    RTC->BKP0R

void Request_DFU_Mode(void)
{
  // RTC 백업 레지스터에 플래그 설정
  HAL_PWR_EnableBkUpAccess();
  DFU_BACKUP_REG = DFU_REQUEST_FLAG;

  // 시스템 리셋
  NVIC_SystemReset();
}

// DFU 부트로더에서 체크
void check_dfu_request(void)
{
  if (DFU_BACKUP_REG == DFU_REQUEST_FLAG)
  {
    // 플래그 클리어
    HAL_PWR_EnableBkUpAccess();
    DFU_BACKUP_REG = 0;

    // DFU 모드 시작
    DFU_Start();
  }
}
```

### 3. USB DFU_DETACH 요청
```c
// 호스트에서 DFU_DETACH 요청 시
static uint8_t USBD_DFU_EP0_RxReady(USBD_HandleTypeDef *pdev)
{
  // DFU 상태가 appIDLE이면
  // DFU_DETACH 후 USB 리셋을 기다림
  // USB 리셋 시 dfuIDLE 상태로 전환
}
```

## 코드 구현

### 1. USB DFU 초기화
```c
#include "usbd_core.h"
#include "usbd_desc.h"
#include "usbd_dfu.h"
#include "usbd_dfu_flash.h"

USBD_HandleTypeDef hUsbDeviceFS;

int main(void)
{
  // 시스템 초기화
  HAL_Init();
  SystemClock_Config();

  // DFU 모드 진입 조건 체크
  check_dfu_mode();

  // LED 초기화
  BSP_LED_Init(LED1);
  BSP_LED_Init(LED2);

  // USB Device 초기화
  USBD_Init(&hUsbDeviceFS, &DFU_Desc, DEVICE_FS);

  // DFU 클래스 등록
  USBD_RegisterClass(&hUsbDeviceFS, &USBD_DFU);

  // Flash 미디어 인터페이스 추가
  USBD_DFU_RegisterMedia(&hUsbDeviceFS, &USBD_DFU_Flash_fops);

  // USB 시작
  USBD_Start(&hUsbDeviceFS);

  // LED 토글로 DFU 모드 표시
  while (1)
  {
    BSP_LED_Toggle(LED1);
    HAL_Delay(500);
  }
}
```

### 2. Flash 미디어 인터페이스
```c
// usbd_dfu_flash.c

USBD_DFU_MediaTypeDef USBD_DFU_Flash_fops = {
  (uint8_t *)FLASH_DESC_STR,
  Flash_If_Init,
  Flash_If_DeInit,
  Flash_If_Erase,
  Flash_If_Write,
  Flash_If_Read,
  Flash_If_GetStatus,
};

// 초기화
uint16_t Flash_If_Init(void)
{
  // Flash 잠금 해제
  HAL_FLASH_Unlock();
  return USBD_OK;
}

// 종료
uint16_t Flash_If_DeInit(void)
{
  // Flash 잠금
  HAL_FLASH_Lock();
  return USBD_OK;
}

// 섹터 삭제
uint16_t Flash_If_Erase(uint32_t Add)
{
  FLASH_EraseInitTypeDef EraseInitStruct;
  uint32_t SectorError = 0;

  // 섹터 번호 계산
  uint32_t sector = GetSector(Add);

  // 삭제 설정
  EraseInitStruct.TypeErase = FLASH_TYPEERASE_SECTORS;
  EraseInitStruct.Banks = FLASH_BANK_1;
  EraseInitStruct.Sector = sector;
  EraseInitStruct.NbSectors = 1;
  EraseInitStruct.VoltageRange = FLASH_VOLTAGE_RANGE_3;

  // 삭제 실행
  if (HAL_FLASHEx_Erase(&EraseInitStruct, &SectorError) != HAL_OK)
  {
    return USBD_FAIL;
  }

  return USBD_OK;
}
```

### 3. Flash 쓰기
```c
// Flash 쓰기 (32바이트 단위)
uint16_t Flash_If_Write(uint8_t *src, uint8_t *dest, uint32_t Len)
{
  uint32_t i = 0;

  // 주소 정렬 확인
  if ((uint32_t)dest % 32 != 0)
  {
    return USBD_FAIL;
  }

  // 32바이트 (256비트) 단위로 프로그래밍
  for (i = 0; i < Len; i += 32)
  {
    if (HAL_FLASH_Program(FLASH_TYPEPROGRAM_FLASHWORD,
                          (uint32_t)(dest + i),
                          (uint32_t)(src + i)) != HAL_OK)
    {
      return USBD_FAIL;
    }
  }

  return USBD_OK;
}
```

### 4. Flash 읽기
```c
// Flash 읽기 (업로드용)
uint8_t *Flash_If_Read(uint8_t *src, uint8_t *dest, uint32_t Len)
{
  // 직접 메모리 복사
  memcpy(dest, src, Len);
  return dest;
}
```

### 5. 상태 확인
```c
uint16_t Flash_If_GetStatus(uint32_t Add, uint8_t Cmd, uint8_t *buffer)
{
  switch (Cmd)
  {
    case DFU_MEDIA_PROGRAM:
      // 프로그래밍 시간
      buffer[1] = 0;    // Poll timeout (ms) - LSB
      buffer[2] = 0;
      buffer[3] = 0;    // Poll timeout (ms) - MSB
      break;

    case DFU_MEDIA_ERASE:
      // 섹터 삭제 시간 (약 1초)
      buffer[1] = (uint8_t)(1000 & 0xFF);
      buffer[2] = (uint8_t)((1000 >> 8) & 0xFF);
      buffer[3] = 0;
      break;

    default:
      break;
  }

  return USBD_OK;
}
```

## DFU Descriptor

### DFU Functional Descriptor
```c
__ALIGN_BEGIN static uint8_t USBD_DFU_CfgDesc[USB_DFU_CONFIG_DESC_SIZ] __ALIGN_END = {
  // Configuration Descriptor
  0x09,                           /* bLength */
  USB_DESC_TYPE_CONFIGURATION,    /* bDescriptorType */
  USB_DFU_CONFIG_DESC_SIZ, 0x00,  /* wTotalLength */
  0x01,                           /* bNumInterfaces */
  0x01,                           /* bConfigurationValue */
  0x00,                           /* iConfiguration */
  0x80,                           /* bmAttributes */
  0xFA,                           /* bMaxPower = 500 mA */

  // Interface Descriptor
  0x09,                           /* bLength */
  USB_DESC_TYPE_INTERFACE,        /* bDescriptorType */
  0x00,                           /* bInterfaceNumber */
  0x00,                           /* bAlternateSetting */
  0x00,                           /* bNumEndpoints */
  0xFE,                           /* bInterfaceClass = Application Specific */
  0x01,                           /* bInterfaceSubClass = DFU */
  0x02,                           /* bInterfaceProtocol = DFU mode */
  0x00,                           /* iInterface */

  // DFU Functional Descriptor
  0x09,                           /* bLength */
  DFU_DESCRIPTOR_TYPE,            /* bDescriptorType = DFU Functional */
  0x0B,                           /* bmAttributes:
                                     bit 0 = 1: Download supported
                                     bit 1 = 1: Upload supported
                                     bit 2 = 0: No detach capable
                                     bit 3 = 1: Manifestation tolerant */
  0xFF, 0x00,                     /* wDetachTimeout = 255 ms */
  LOBYTE(USBD_DFU_XFER_SIZE),     /* wTransferSize */
  HIBYTE(USBD_DFU_XFER_SIZE),
  0x1A, 0x01                      /* bcdDFUVersion = 1.1a */
};
```

## DFU 도구 사용

### 1. STM32CubeProgrammer (권장)
```bash
# 연결된 DFU 장치 확인
STM32_Programmer_CLI -l usb

# 펌웨어 다운로드
STM32_Programmer_CLI -c port=usb1 -w firmware.bin 0x08020000

# 펌웨어 업로드 (읽기)
STM32_Programmer_CLI -c port=usb1 -r 0x08020000 0x10000 firmware_read.bin

# 전체 칩 삭제
STM32_Programmer_CLI -c port=usb1 -e all
```

### 2. dfu-util (크로스 플랫폼)
```bash
# 설치
# Ubuntu: sudo apt install dfu-util
# macOS: brew install dfu-util

# 연결된 DFU 장치 확인
dfu-util -l

# 펌웨어 다운로드
dfu-util -a 0 -s 0x08020000:leave -D firmware.bin

# 펌웨어 업로드
dfu-util -a 0 -s 0x08020000 -U firmware_read.bin
```

### 3. DfuSe Demo (Windows)
- ST에서 제공하는 GUI 도구
- .dfu 파일 형식 지원

## 애플리케이션 점프

### 부트로더에서 애플리케이션으로 점프
```c
#define APPLICATION_ADDRESS  0x08020000

typedef void (*pFunction)(void);

void Jump_To_Application(void)
{
  uint32_t JumpAddress;
  pFunction Jump_To_Application_Function;

  // 애플리케이션이 유효한지 확인
  // (스택 포인터가 RAM 범위 내에 있는지)
  if (((*(__IO uint32_t*)APPLICATION_ADDRESS) & 0x2FFE0000) == 0x20000000)
  {
    // 모든 인터럽트 비활성화
    __disable_irq();

    // SysTick 비활성화
    SysTick->CTRL = 0;
    SysTick->LOAD = 0;
    SysTick->VAL = 0;

    // 모든 인터럽트 클리어
    for (uint8_t i = 0; i < 8; i++)
    {
      NVIC->ICER[i] = 0xFFFFFFFF;
      NVIC->ICPR[i] = 0xFFFFFFFF;
    }

    // 스택 포인터 설정
    __set_MSP(*(__IO uint32_t*)APPLICATION_ADDRESS);

    // 점프 주소 (리셋 핸들러)
    JumpAddress = *(__IO uint32_t*)(APPLICATION_ADDRESS + 4);
    Jump_To_Application_Function = (pFunction)JumpAddress;

    // 애플리케이션으로 점프
    Jump_To_Application_Function();
  }
  else
  {
    // 유효한 애플리케이션이 없음
    Error_Handler();
  }
}
```

### 애플리케이션 벡터 테이블 재배치
```c
// 애플리케이션의 시스템 초기화에서
void SystemInit(void)
{
  // ...기존 초기화...

  // 벡터 테이블 재배치
  SCB->VTOR = APPLICATION_ADDRESS;
}
```

## 보안 고려사항

### 1. 쓰기 보호
```c
// 특정 섹터 쓰기 보호
void Set_Write_Protection(void)
{
  FLASH_OBProgramInitTypeDef OBInit;

  HAL_FLASH_Unlock();
  HAL_FLASH_OB_Unlock();

  HAL_FLASHEx_OBGetConfig(&OBInit);

  // 부트로더 섹터 (0, 1) 보호
  OBInit.OptionType = OPTIONBYTE_WRP;
  OBInit.WRPState = OB_WRPSTATE_ENABLE;
  OBInit.Banks = FLASH_BANK_1;
  OBInit.WRPSector = OB_WRP_SECTOR_0 | OB_WRP_SECTOR_1;

  HAL_FLASHEx_OBProgram(&OBInit);
  HAL_FLASH_OB_Launch();

  HAL_FLASH_OB_Lock();
  HAL_FLASH_Lock();
}
```

### 2. 읽기 보호 (RDP)
```c
// Level 1: JTAG/SWD 디버그 비활성화
void Set_RDP_Level1(void)
{
  FLASH_OBProgramInitTypeDef OBInit;

  HAL_FLASH_Unlock();
  HAL_FLASH_OB_Unlock();

  OBInit.OptionType = OPTIONBYTE_RDP;
  OBInit.RDPLevel = OB_RDP_LEVEL_1;

  HAL_FLASHEx_OBProgram(&OBInit);
  HAL_FLASH_OB_Launch();

  HAL_FLASH_OB_Lock();
  HAL_FLASH_Lock();
}
```

### 3. 체크섬/CRC 검증
```c
// 애플리케이션 무결성 검증
uint32_t Calculate_CRC(uint32_t address, uint32_t size)
{
  __HAL_RCC_CRC_CLK_ENABLE();

  CRC_HandleTypeDef hcrc;
  hcrc.Instance = CRC;
  hcrc.Init.DefaultPolynomialUse = DEFAULT_POLYNOMIAL_ENABLE;
  hcrc.Init.DefaultInitValueUse = DEFAULT_INIT_VALUE_ENABLE;
  hcrc.Init.InputDataInversionMode = CRC_INPUTDATA_INVERSION_NONE;
  hcrc.Init.OutputDataInversionMode = CRC_OUTPUTDATA_INVERSION_DISABLE;
  hcrc.InputDataFormat = CRC_INPUTDATA_FORMAT_WORDS;

  HAL_CRC_Init(&hcrc);

  return HAL_CRC_Calculate(&hcrc, (uint32_t*)address, size / 4);
}
```

## 트러블슈팅

### DFU 장치가 인식되지 않는 경우

1. **USB 클럭 확인**: 48MHz USB 클럭 필요

2. **드라이버 설치** (Windows):
   - Zadig을 사용하여 WinUSB 드라이버 설치
   - 또는 STM32 DFU 드라이버 설치

3. **VID/PID 확인**:
   ```c
   #define USBD_VID           0x0483
   #define USBD_PID           0xDF11  // ST DFU PID
   ```

### 다운로드 실패

1. **주소 확인**: 올바른 Flash 주소 사용
2. **섹터 정렬**: 섹터 경계에서 시작
3. **크기 확인**: Transfer 크기 제한 (보통 1024 바이트)

### 애플리케이션이 실행되지 않는 경우

1. **벡터 테이블**: VTOR 설정 확인
2. **스택 포인터**: 유효한 RAM 주소인지 확인
3. **인터럽트**: 모든 인터럽트 클리어 확인

## 성능 고려사항

### 전송 속도
- **Full Speed USB**: 최대 12 Mbps
- **실제 전송률**: 약 50-100 KB/s
- **병목**: Flash 삭제/프로그래밍 시간

### 최적화
```c
// Transfer 크기 증가 (최대 1024)
#define USBD_DFU_XFER_SIZE  1024

// 대용량 전송 시 진행률 표시
// LED 또는 UART 출력
```

## 참고 자료

- USB DFU 사양 1.1: https://www.usb.org/sites/default/files/DFU_1.1.pdf
- AN3156: USB DFU protocol
- STM32CubeProgrammer 사용자 매뉴얼: UM2237
- 예제: `STM32H745I-DISCO/Applications/USB_Device/DFU_Standalone`
