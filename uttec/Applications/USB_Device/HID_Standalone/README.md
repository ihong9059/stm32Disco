# USB_Device HID_Standalone - USB HID 장치 (마우스 에뮬레이션)

## 개요

이 애플리케이션은 STM32H745I-DISCO 보드를 USB HID (Human Interface Device) 장치로 구현하여 마우스를 에뮬레이션하는 예제입니다. 보드의 조이스틱 입력이 USB 마우스 움직임으로 변환됩니다.

## USB HID란?

**USB HID (Human Interface Device)**는 사람과 컴퓨터 간의 상호작용을 위한 USB 디바이스 클래스입니다.

### 주요 특징
- **드라이버 불필요**: OS 기본 드라이버로 동작
- **저지연**: 인터럽트 전송 사용
- **다양한 장치 지원**: 마우스, 키보드, 조이스틱 등

## 시스템 아키텍처

### USB HID 마우스 구조
```
┌──────────────────────────────────────────────────────────┐
│                    STM32H745I-DISCO                      │
│                                                          │
│  ┌─────────────┐      ┌──────────────┐      ┌─────────┐  │
│  │  Joystick   │──────│  HID Class   │──────│ USB FS  │  │
│  │  (5-way)    │      │  Driver      │      │ (OTG)   │  │
│  │             │      │              │      │         │  │
│  │  UP/DOWN    │      │ Mouse Report │      │ CN13    │  │
│  │  LEFT/RIGHT │      │ Generator    │      │ USB-C   │  │
│  │  CENTER     │      │              │      │         │  │
│  └─────────────┘      └──────────────┘      └────┬────┘  │
│                                                   │      │
└───────────────────────────────────────────────────┼──────┘
                                                    │
                                              USB Cable
                                                    │
                                              ┌─────▼─────┐
                                              │    Host   │
                                              │    PC     │
                                              └───────────┘
```

## 하드웨어 연결

### USB 커넥터
| 커넥터 | 위치 | 용도 |
|--------|------|------|
| CN13 | USB-C | USB Full Speed Device |
| CN14 | Micro-USB | ST-Link (디버깅) |

### 조이스틱 매핑
```
          UP (GPIO PI11)
            ▲
            │
LEFT ◄──── CENTER ────► RIGHT
(PI9)      (PI10)       (PF14)
            │
            ▼
         DOWN (PI8)
```

### LED 상태 표시
| LED | 상태 | 의미 |
|-----|------|------|
| LED1 (녹색) | 점멸 | USB 연결됨 |
| LED2 (주황) | 켜짐 | USB 전송 중 |
| LED3 (빨강) | 켜짐 | 오류 발생 |

## USB 설정

### USB Device Descriptor
```c
USBD_DescriptorsTypeDef HID_Desc = {
  USBD_HID_DeviceDescriptor,
  USBD_HID_LangIDStrDescriptor,
  USBD_HID_ManufacturerStrDescriptor,
  USBD_HID_ProductStrDescriptor,
  USBD_HID_SerialStrDescriptor,
  USBD_HID_ConfigStrDescriptor,
  USBD_HID_InterfaceStrDescriptor
};

// Device Descriptor
__ALIGN_BEGIN uint8_t USBD_HID_DeviceDesc[USB_LEN_DEV_DESC] __ALIGN_END = {
  0x12,                       /* bLength */
  USB_DESC_TYPE_DEVICE,       /* bDescriptorType */
  0x00, 0x02,                 /* bcdUSB = 2.00 */
  0x00,                       /* bDeviceClass */
  0x00,                       /* bDeviceSubClass */
  0x00,                       /* bDeviceProtocol */
  USB_MAX_EP0_SIZE,           /* bMaxPacketSize */
  LOBYTE(USBD_VID),           /* idVendor */
  HIBYTE(USBD_VID),
  LOBYTE(USBD_PID_HID),       /* idProduct */
  HIBYTE(USBD_PID_HID),
  0x00, 0x02,                 /* bcdDevice = 2.00 */
  USBD_IDX_MFC_STR,           /* Index of manufacturer string */
  USBD_IDX_PRODUCT_STR,       /* Index of product string */
  USBD_IDX_SERIAL_STR,        /* Index of serial number string */
  USBD_MAX_NUM_CONFIGURATION  /* bNumConfigurations */
};
```

### HID Report Descriptor (마우스)
```c
__ALIGN_BEGIN static uint8_t HID_MOUSE_ReportDesc[HID_MOUSE_REPORT_DESC_SIZE] __ALIGN_END = {
  0x05, 0x01,        /* Usage Page (Generic Desktop) */
  0x09, 0x02,        /* Usage (Mouse) */
  0xA1, 0x01,        /* Collection (Application) */
  0x09, 0x01,        /*   Usage (Pointer) */
  0xA1, 0x00,        /*   Collection (Physical) */

  /* Buttons (3개) */
  0x05, 0x09,        /*     Usage Page (Buttons) */
  0x19, 0x01,        /*     Usage Minimum (Button 1) */
  0x29, 0x03,        /*     Usage Maximum (Button 3) */
  0x15, 0x00,        /*     Logical Minimum (0) */
  0x25, 0x01,        /*     Logical Maximum (1) */
  0x95, 0x03,        /*     Report Count (3) */
  0x75, 0x01,        /*     Report Size (1) */
  0x81, 0x02,        /*     Input (Data,Var,Abs) */

  /* Padding (5 bits) */
  0x95, 0x01,        /*     Report Count (1) */
  0x75, 0x05,        /*     Report Size (5) */
  0x81, 0x01,        /*     Input (Cnst,Var,Abs) */

  /* X, Y 이동 */
  0x05, 0x01,        /*     Usage Page (Generic Desktop) */
  0x09, 0x30,        /*     Usage (X) */
  0x09, 0x31,        /*     Usage (Y) */
  0x15, 0x81,        /*     Logical Minimum (-127) */
  0x25, 0x7F,        /*     Logical Maximum (127) */
  0x75, 0x08,        /*     Report Size (8) */
  0x95, 0x02,        /*     Report Count (2) */
  0x81, 0x06,        /*     Input (Data,Var,Rel) */

  0xC0,              /*   End Collection */
  0xC0               /* End Collection */
};
```

## 코드 구현

### 1. USB 초기화
```c
#include "usbd_core.h"
#include "usbd_desc.h"
#include "usbd_hid.h"

USBD_HandleTypeDef hUsbDeviceFS;

int main(void)
{
  // 시스템 초기화
  HAL_Init();
  SystemClock_Config();

  // GPIO 초기화 (조이스틱, LED)
  MX_GPIO_Init();

  // USB Device 초기화
  USBD_Init(&hUsbDeviceFS, &HID_Desc, DEVICE_FS);

  // HID 클래스 등록
  USBD_RegisterClass(&hUsbDeviceFS, &USBD_HID);

  // USB 시작
  USBD_Start(&hUsbDeviceFS);

  while (1)
  {
    // 조이스틱 상태 확인 및 마우스 리포트 전송
    HID_Mouse_UpdateState();
  }
}
```

### 2. 조이스틱 입력 처리
```c
typedef struct {
  uint8_t buttons;   // 버튼 상태 (bit 0-2: Button 1-3)
  int8_t x;          // X 이동량 (-127 ~ 127)
  int8_t y;          // Y 이동량 (-127 ~ 127)
} HID_MouseReport_TypeDef;

#define MOUSE_SPEED  5
#define JOYSTICK_CENTER_PRESSED  0x01

void HID_Mouse_UpdateState(void)
{
  static uint32_t last_send_time = 0;
  HID_MouseReport_TypeDef mouse_report = {0, 0, 0};

  // 폴링 간격 (10ms)
  if (HAL_GetTick() - last_send_time < 10)
    return;

  last_send_time = HAL_GetTick();

  // 조이스틱 상태 읽기
  if (HAL_GPIO_ReadPin(JOY_UP_GPIO_Port, JOY_UP_Pin) == GPIO_PIN_SET)
  {
    mouse_report.y = -MOUSE_SPEED;  // 위로 이동
  }
  else if (HAL_GPIO_ReadPin(JOY_DOWN_GPIO_Port, JOY_DOWN_Pin) == GPIO_PIN_SET)
  {
    mouse_report.y = MOUSE_SPEED;   // 아래로 이동
  }

  if (HAL_GPIO_ReadPin(JOY_LEFT_GPIO_Port, JOY_LEFT_Pin) == GPIO_PIN_SET)
  {
    mouse_report.x = -MOUSE_SPEED;  // 왼쪽으로 이동
  }
  else if (HAL_GPIO_ReadPin(JOY_RIGHT_GPIO_Port, JOY_RIGHT_Pin) == GPIO_PIN_SET)
  {
    mouse_report.x = MOUSE_SPEED;   // 오른쪽으로 이동
  }

  // 중앙 버튼 = 마우스 왼쪽 클릭
  if (HAL_GPIO_ReadPin(JOY_CENTER_GPIO_Port, JOY_CENTER_Pin) == GPIO_PIN_SET)
  {
    mouse_report.buttons = JOYSTICK_CENTER_PRESSED;
  }

  // USB를 통해 리포트 전송
  USBD_HID_SendReport(&hUsbDeviceFS, (uint8_t*)&mouse_report, sizeof(mouse_report));
}
```

### 3. USB 이벤트 콜백
```c
// USB 연결 콜백
void USBD_HID_Connected(USBD_HandleTypeDef *pdev)
{
  BSP_LED_On(LED1);
}

// USB 연결 해제 콜백
void USBD_HID_Disconnected(USBD_HandleTypeDef *pdev)
{
  BSP_LED_Off(LED1);
}

// USB 서스펜드 콜백
void USBD_HID_Suspend(USBD_HandleTypeDef *pdev)
{
  // 저전력 모드 진입 준비
  BSP_LED_Off(LED1);
}

// USB 리줌 콜백
void USBD_HID_Resume(USBD_HandleTypeDef *pdev)
{
  BSP_LED_On(LED1);
}
```

## Remote Wakeup 지원

### Remote Wakeup란?

USB 장치가 서스펜드 상태의 호스트를 깨울 수 있는 기능입니다.

### 구성 설정
```c
// usbd_conf.h
#define USBD_SELF_POWERED          0
#define USBD_REMOTE_WAKEUP_ENABLED 1

// Configuration Descriptor에서 Remote Wakeup 활성화
0x80 | (USBD_SELF_POWERED << 6) | (USBD_REMOTE_WAKEUP_ENABLED << 5),
```

### Remote Wakeup 구현
```c
void HID_RemoteWakeup_Trigger(void)
{
  USBD_HID_HandleTypeDef *hhid = (USBD_HID_HandleTypeDef *)hUsbDeviceFS.pClassData;

  // 장치가 서스펜드 상태인지 확인
  if (hUsbDeviceFS.dev_state == USBD_STATE_SUSPENDED)
  {
    // Remote Wakeup 기능이 호스트에서 허용되었는지 확인
    if (hUsbDeviceFS.dev_remote_wakeup)
    {
      // Remote Wakeup 신호 발생
      HAL_PCD_ActivateRemoteWakeup(hUsbDeviceFS.pData);
      HAL_Delay(10);  // 최소 1ms 유지
      HAL_PCD_DeActivateRemoteWakeup(hUsbDeviceFS.pData);
    }
  }
}

// 조이스틱 인터럽트에서 Remote Wakeup 트리거
void EXTI_Joystick_IRQHandler(void)
{
  if (__HAL_GPIO_EXTI_GET_IT(JOY_CENTER_Pin) != RESET)
  {
    __HAL_GPIO_EXTI_CLEAR_IT(JOY_CENTER_Pin);

    // Remote Wakeup 시도
    HID_RemoteWakeup_Trigger();
  }
}
```

## USB Full Speed 설정

### 클럭 설정 (48MHz USB 클럭)
```c
void USB_Clock_Config(void)
{
  RCC_PeriphCLKInitTypeDef PeriphClkInitStruct = {0};

  // USB FS 클럭 소스: PLL3Q = 48MHz
  PeriphClkInitStruct.PeriphClockSelection = RCC_PERIPHCLK_USB;
  PeriphClkInitStruct.UsbClockSelection = RCC_USBCLKSOURCE_PLL3;

  if (HAL_RCCEx_PeriphCLKConfig(&PeriphClkInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  // USB FS 클럭 활성화
  __HAL_RCC_USB2_OTG_FS_CLK_ENABLE();
}
```

### USB OTG 핀 설정
```c
void USB_GPIO_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  // USB_OTG_FS GPIO 설정
  // PA11: USB_OTG_FS_DM
  // PA12: USB_OTG_FS_DP

  __HAL_RCC_GPIOA_CLK_ENABLE();

  GPIO_InitStruct.Pin = GPIO_PIN_11 | GPIO_PIN_12;
  GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;
  GPIO_InitStruct.Alternate = GPIO_AF10_OTG1_FS;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  // USB OTG FS 인터럽트 설정
  HAL_NVIC_SetPriority(OTG_FS_IRQn, 6, 0);
  HAL_NVIC_EnableIRQ(OTG_FS_IRQn);
}
```

## 고급 기능

### 1. 가속도 기반 마우스 속도 조절
```c
#define MOUSE_SPEED_MIN    1
#define MOUSE_SPEED_MAX    20
#define ACCELERATION_TIME  500  // ms

static int8_t calculate_mouse_speed(uint32_t press_duration)
{
  // 조이스틱을 오래 누를수록 속도 증가
  if (press_duration > ACCELERATION_TIME)
  {
    return MOUSE_SPEED_MAX;
  }

  // 선형 가속
  return MOUSE_SPEED_MIN +
         (press_duration * (MOUSE_SPEED_MAX - MOUSE_SPEED_MIN)) / ACCELERATION_TIME;
}
```

### 2. 더블 클릭 지원
```c
#define DOUBLE_CLICK_TIME  300  // ms

void handle_center_button(void)
{
  static uint32_t last_click_time = 0;
  static uint8_t click_count = 0;

  uint32_t current_time = HAL_GetTick();

  if (current_time - last_click_time < DOUBLE_CLICK_TIME)
  {
    click_count++;
    if (click_count == 2)
    {
      // 더블 클릭 처리
      send_double_click();
      click_count = 0;
    }
  }
  else
  {
    click_count = 1;
  }

  last_click_time = current_time;
}
```

### 3. 휠 스크롤 지원
```c
// 휠이 포함된 마우스 리포트
typedef struct {
  uint8_t buttons;
  int8_t x;
  int8_t y;
  int8_t wheel;  // 스크롤 휠 (-127 ~ 127)
} HID_MouseReportExt_TypeDef;

// 추가 Report Descriptor 필요
// 0x09, 0x38,        /*     Usage (Wheel) */
```

## 응용 예제

### 1. 프레젠테이션 리모컨
```c
// 조이스틱을 프레젠테이션 리모컨으로 사용
// LEFT/RIGHT: 슬라이드 이동 (Page Up/Down 키 전송)
// CENTER: 마우스 클릭

// HID 키보드 기능과 결합 필요
```

### 2. 게임 컨트롤러
```c
// 조이스틱을 게임 컨트롤러로 사용
// 아날로그 값이 아닌 디지털이므로 제한적
```

### 3. 접근성 장치
```c
// 신체적 제약이 있는 사용자를 위한 대체 입력 장치
```

## 트러블슈팅

### USB가 인식되지 않는 경우

1. **클럭 설정 확인**: USB 48MHz 클럭 필요
   ```c
   // PLL3 Q 출력이 48MHz인지 확인
   RCC_OscInitStruct.PLL3.PLL3Q = ...;  // 48MHz
   ```

2. **USB 케이블 확인**: 데이터 케이블 사용 (충전 전용 케이블 X)

3. **드라이버 문제**: 장치 관리자에서 HID 드라이버 확인

4. **VID/PID 충돌**: 다른 VID/PID 사용
   ```c
   #define USBD_VID     0x0483  // ST Microelectronics
   #define USBD_PID_HID 0x5710  // HID Device
   ```

### 마우스 움직임이 불규칙한 경우

1. **폴링 간격 확인**: 10ms 권장
   ```c
   // bInterval = 10 (10ms)
   HID_IN_EP,                  /* bEndpointAddress */
   0x03,                       /* bmAttributes */
   HID_EPIN_SIZE, 0x00,        /* wMaxPacketSize */
   HID_FS_BINTERVAL,           /* bInterval: 10ms */
   ```

2. **디바운싱 추가**: 조이스틱 입력에 디바운스 적용

### Remote Wakeup가 동작하지 않는 경우

1. **호스트 설정 확인**: 전원 관리에서 "이 장치를 사용하여 컴퓨터의 대기 모드를 종료할 수 있음" 체크

2. **Configuration Descriptor 확인**: Remote Wakeup 비트 설정

3. **SET_FEATURE 요청 처리**: 호스트가 Remote Wakeup을 활성화했는지 확인

## 성능 고려사항

### 인터럽트 전송 특성
- **보장된 대역폭**: 매 폴링 주기마다 전송 기회
- **최소 지연**: 1ms (Full Speed)
- **최대 패킷 크기**: 64 바이트

### 전력 소비
- **동작 모드**: 약 20-30 mA
- **서스펜드 모드**: < 2.5 mA (USB 규격)

### CPU 부하
- **폴링 방식**: 매 10ms마다 조이스틱 읽기
- **인터럽트 방식 권장**: GPIO EXTI 사용

## 참고 자료

- USB HID 사양: https://www.usb.org/hid
- STM32 USB Device Library: UM1734
- AN4879: USB hardware and PCB guidelines
- 예제: `STM32H745I-DISCO/Applications/USB_Device/HID_Standalone`
