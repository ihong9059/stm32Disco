# USB_Host HID_Standalone - USB HID 호스트 (마우스/키보드 지원)

## 개요

이 애플리케이션은 STM32H745I-DISCO 보드를 USB HID 호스트로 구현하여 마우스, 키보드 등의 USB HID 장치를 연결하고 사용하는 예제입니다. HID 장치를 열거하고, 리포트를 파싱하여 입력 데이터를 처리합니다.

## USB HID 호스트란?

**USB HID 호스트**는 마우스, 키보드 등의 HID 장치를 연결받아 입력 데이터를 처리하는 호스트 시스템입니다.

### 지원 장치
- **마우스**: X/Y 이동, 버튼, 휠
- **키보드**: 키 입력, 수정자 키
- **게임패드**: 버튼, 축
- **기타 HID 장치**: 터치패드, 펜 타블렛 등

## 시스템 아키텍처

### USB HID 호스트 구조
```
┌──────────────────────────────────────────────────────────────┐
│                      STM32H745I-DISCO                        │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                    Application                       │    │
│  │    (마우스/키보드 데이터 처리)                       │    │
│  └────────────────────┬─────────────────────────────────┘    │
│                       │                                      │
│  ┌────────────────────▼─────────────────────────────────┐    │
│  │                  HID Class Driver                    │    │
│  │                                                      │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │    │
│  │  │   Mouse     │  │  Keyboard   │  │   Generic   │   │    │
│  │  │   Handler   │  │   Handler   │  │   Handler   │   │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘   │    │
│  │                                                      │    │
│  └────────────────────┬─────────────────────────────────┘    │
│                       │                                      │
│  ┌────────────────────▼─────────────────────────────────┐    │
│  │                 USB Host Core                        │    │
│  │    (장치 열거, 리포트 디스크립터 파싱)               │    │
│  └────────────────────┬─────────────────────────────────┘    │
│                       │                                      │
│  ┌────────────────────▼─────────────────────────────────┐    │
│  │              USB OTG FS Controller                   │    │
│  │                   (CN13)                             │    │
│  └────────────────────┬─────────────────────────────────┘    │
│                       │                                      │
└───────────────────────┼──────────────────────────────────────┘
                        │
                   USB Cable
                        │
               ┌────────▼────────┐
               │  HID Device     │
               │  (Mouse/Kbd)    │
               └─────────────────┘
```

## 하드웨어 연결

### USB 커넥터
| 커넥터 | 위치 | 용도 |
|--------|------|------|
| CN13 | USB-C OTG | USB Host (HID 장치 연결) |
| CN14 | Micro-USB | ST-Link (디버깅) |

### 전원 설정
- **VBUS 공급**: 5V (USB-C OTG)
- **최대 전류**: 500mA

### LED 상태
| LED | 상태 | 의미 |
|-----|------|------|
| LED1 (녹색) | 점멸 | HID 장치 연결됨 |
| LED2 (주황) | 켜짐 | 마우스 동작 감지 |
| LED3 (빨강) | 켜짐 | 키보드 입력 감지 |

## HID 장치 열거

### 열거 과정
```
1. 장치 연결 감지 (VBUS)
          │
          ▼
2. USB 리셋 및 속도 검출
          │
          ▼
3. 장치 주소 할당
          │
          ▼
4. Device Descriptor 읽기
          │
          ▼
5. Configuration Descriptor 읽기
          │
          ▼
6. Interface Descriptor 확인 (Class = 0x03)
          │
          ▼
7. HID Descriptor 읽기
          │
          ▼
8. Report Descriptor 읽기
          │
          ▼
9. HID Class 핸들러 호출
          │
          ▼
10. 정기적으로 인터럽트 IN 폴링
```

## 코드 구현

### 1. USB Host 초기화
```c
#include "usbh_core.h"
#include "usbh_hid.h"
#include "usbh_hid_mouse.h"
#include "usbh_hid_keybd.h"

USBH_HandleTypeDef hUsbHostFS;

int main(void)
{
  // 시스템 초기화
  HAL_Init();
  SystemClock_Config();

  // LED 초기화
  BSP_LED_Init(LED1);
  BSP_LED_Init(LED2);
  BSP_LED_Init(LED3);

  // USB Host 초기화
  USBH_Init(&hUsbHostFS, USBH_UserProcess, HOST_FS);

  // HID 클래스 등록
  USBH_RegisterClass(&hUsbHostFS, USBH_HID_CLASS);

  // USB Host 시작
  USBH_Start(&hUsbHostFS);

  while (1)
  {
    // USB Host 상태 머신 처리
    USBH_Process(&hUsbHostFS);

    // HID 데이터 처리
    HID_Process();
  }
}
```

### 2. USB Host 콜백 처리
```c
void USBH_UserProcess(USBH_HandleTypeDef *phost, uint8_t id)
{
  switch (id)
  {
    case HOST_USER_SELECT_CONFIGURATION:
      break;

    case HOST_USER_DISCONNECTION:
      BSP_LED_Off(LED1);
      printf("Device Disconnected\n");
      break;

    case HOST_USER_CLASS_ACTIVE:
      BSP_LED_On(LED1);

      // HID 장치 타입 확인
      HID_TypeTypeDef hid_type = USBH_HID_GetDeviceType(&hUsbHostFS);

      switch (hid_type)
      {
        case HID_MOUSE:
          printf("Mouse Connected\n");
          break;

        case HID_KEYBOARD:
          printf("Keyboard Connected\n");
          break;

        default:
          printf("Unknown HID Device\n");
          break;
      }
      break;

    case HOST_USER_CONNECTION:
      printf("Device Connected\n");
      break;

    default:
      break;
  }
}
```

### 3. 마우스 데이터 처리
```c
typedef struct {
  uint8_t buttons;
  int8_t x;
  int8_t y;
  int8_t wheel;
} HID_MOUSE_Info_TypeDef;

void HID_Mouse_Process(void)
{
  HID_MOUSE_Info_TypeDef mouse_info;

  // 마우스 데이터 읽기
  if (USBH_HID_GetMouseInfo(&hUsbHostFS, &mouse_info) == USBH_OK)
  {
    // 이동 처리
    if (mouse_info.x != 0 || mouse_info.y != 0)
    {
      printf("Mouse Move: X=%d, Y=%d\n", mouse_info.x, mouse_info.y);
      BSP_LED_Toggle(LED2);

      // 커서 위치 업데이트
      cursor_x += mouse_info.x;
      cursor_y += mouse_info.y;

      // 경계 검사
      if (cursor_x < 0) cursor_x = 0;
      if (cursor_x > SCREEN_WIDTH) cursor_x = SCREEN_WIDTH;
      if (cursor_y < 0) cursor_y = 0;
      if (cursor_y > SCREEN_HEIGHT) cursor_y = SCREEN_HEIGHT;
    }

    // 버튼 처리
    if (mouse_info.buttons & 0x01)  // 왼쪽 버튼
    {
      printf("Left Button Pressed\n");
    }
    if (mouse_info.buttons & 0x02)  // 오른쪽 버튼
    {
      printf("Right Button Pressed\n");
    }
    if (mouse_info.buttons & 0x04)  // 중앙 버튼
    {
      printf("Middle Button Pressed\n");
    }

    // 휠 처리
    if (mouse_info.wheel != 0)
    {
      printf("Wheel: %d\n", mouse_info.wheel);
    }
  }
}
```

### 4. 키보드 데이터 처리
```c
typedef struct {
  uint8_t modifiers;   // Shift, Ctrl, Alt, GUI
  uint8_t reserved;
  uint8_t keys[6];     // 동시 입력 키 (최대 6개)
} HID_KEYBD_Info_TypeDef;

void HID_Keyboard_Process(void)
{
  HID_KEYBD_Info_TypeDef keybd_info;

  // 키보드 데이터 읽기
  if (USBH_HID_GetKeybdInfo(&hUsbHostFS, &keybd_info) == USBH_OK)
  {
    // 수정자 키 처리
    if (keybd_info.modifiers & 0x01)  // Left Ctrl
      printf("L-Ctrl ");
    if (keybd_info.modifiers & 0x02)  // Left Shift
      printf("L-Shift ");
    if (keybd_info.modifiers & 0x04)  // Left Alt
      printf("L-Alt ");
    if (keybd_info.modifiers & 0x08)  // Left GUI (Windows)
      printf("L-GUI ");

    // 일반 키 처리
    for (int i = 0; i < 6; i++)
    {
      if (keybd_info.keys[i] != 0)
      {
        // Usage ID를 ASCII로 변환
        char ch = HID_KeyCode_to_ASCII(keybd_info.keys[i], keybd_info.modifiers);

        if (ch != 0)
        {
          printf("Key: '%c' (0x%02X)\n", ch, keybd_info.keys[i]);
          BSP_LED_Toggle(LED3);
        }
        else
        {
          printf("Key Code: 0x%02X\n", keybd_info.keys[i]);
        }
      }
    }
  }
}

// HID Usage ID를 ASCII로 변환
char HID_KeyCode_to_ASCII(uint8_t keycode, uint8_t modifiers)
{
  // 기본 변환 테이블 (US 키보드 레이아웃)
  static const char ascii_table[] = {
    0, 0, 0, 0,                    // 0x00-0x03
    'a', 'b', 'c', 'd', 'e', 'f',  // 0x04-0x09
    'g', 'h', 'i', 'j', 'k', 'l',  // 0x0A-0x0F
    'm', 'n', 'o', 'p', 'q', 'r',  // 0x10-0x15
    's', 't', 'u', 'v', 'w', 'x',  // 0x16-0x1B
    'y', 'z',                      // 0x1C-0x1D
    '1', '2', '3', '4', '5', '6',  // 0x1E-0x23
    '7', '8', '9', '0',            // 0x24-0x27
    '\r', 0x1B, '\b', '\t', ' ',   // Enter, Esc, BS, Tab, Space
    '-', '=', '[', ']', '\\',      // 0x2D-0x31
    // ... 추가 키
  };

  if (keycode < sizeof(ascii_table))
  {
    char ch = ascii_table[keycode];

    // Shift 키 처리
    if (modifiers & 0x22)  // Shift 눌림
    {
      if (ch >= 'a' && ch <= 'z')
        ch = ch - 'a' + 'A';  // 대문자 변환
    }

    return ch;
  }

  return 0;
}
```

### 5. HID 프로세스 메인
```c
void HID_Process(void)
{
  if (USBH_HID_GetDeviceType(&hUsbHostFS) == HID_MOUSE)
  {
    HID_Mouse_Process();
  }
  else if (USBH_HID_GetDeviceType(&hUsbHostFS) == HID_KEYBOARD)
  {
    HID_Keyboard_Process();
  }
}
```

## HID Report Descriptor 파싱

### 파서 구조
```c
// HID 리포트 파서 상태
typedef struct {
  uint8_t ReportID;
  uint8_t ReportType;
  uint32_t UsagePage;
  uint32_t Usage;
  int32_t LogicalMin;
  int32_t LogicalMax;
  uint32_t ReportSize;
  uint32_t ReportCount;
} HID_Report_ItemTypedef;

// 리포트 디스크립터 파싱
USBH_StatusTypeDef USBH_HID_ParseReportDescriptor(USBH_HandleTypeDef *phost)
{
  HID_HandleTypeDef *HID_Handle = phost->pActiveClass->pData;
  uint8_t *report_desc = HID_Handle->Report_Desc;
  uint16_t desc_length = HID_Handle->HID_Desc.wDescriptorLength;

  uint16_t index = 0;
  HID_Report_ItemTypedef item = {0};

  while (index < desc_length)
  {
    // 아이템 태그 읽기
    uint8_t tag = report_desc[index] & 0xFC;
    uint8_t size = report_desc[index] & 0x03;

    uint32_t data = 0;
    for (int i = 0; i < size; i++)
    {
      data |= report_desc[index + 1 + i] << (8 * i);
    }

    switch (tag)
    {
      case HID_ITEM_USAGE_PAGE:
        item.UsagePage = data;
        break;

      case HID_ITEM_USAGE:
        item.Usage = data;
        break;

      case HID_ITEM_LOGICAL_MIN:
        item.LogicalMin = (int32_t)data;
        break;

      case HID_ITEM_LOGICAL_MAX:
        item.LogicalMax = (int32_t)data;
        break;

      case HID_ITEM_REPORT_SIZE:
        item.ReportSize = data;
        break;

      case HID_ITEM_REPORT_COUNT:
        item.ReportCount = data;
        break;

      case HID_ITEM_INPUT:
        // Input 아이템 처리
        process_input_item(&item);
        break;

      // ... 기타 아이템 처리
    }

    index += 1 + size;  // 다음 아이템으로
  }

  return USBH_OK;
}
```

## USB Host 설정

### usbh_conf.h 설정
```c
// 최대 지원 장치 수
#define USBH_MAX_NUM_ENDPOINTS         2
#define USBH_MAX_NUM_INTERFACES        2
#define USBH_MAX_NUM_CONFIGURATION     1
#define USBH_MAX_NUM_SUPPORTED_CLASS   1

// HID 설정
#define USBH_HID_MOUSE_REPORT_SIZE     4
#define USBH_HID_KEYBOARD_REPORT_SIZE  8

// 메모리 풀 크기
#define USBH_MAX_DATA_BUFFER           512
```

### 인터럽트 우선순위
```c
void USB_Host_NVIC_Config(void)
{
  // USB OTG FS 인터럽트
  HAL_NVIC_SetPriority(OTG_FS_IRQn, 6, 0);
  HAL_NVIC_EnableIRQ(OTG_FS_IRQn);
}
```

## 고급 기능

### 1. 복합 장치 지원
```c
// 마우스 + 키보드 복합 장치
void HID_Composite_Process(void)
{
  if (USBH_HID_HasInterface(&hUsbHostFS, HID_MOUSE))
  {
    HID_Mouse_Process();
  }

  if (USBH_HID_HasInterface(&hUsbHostFS, HID_KEYBOARD))
  {
    HID_Keyboard_Process();
  }
}
```

### 2. Generic HID 처리
```c
// 알 수 없는 HID 장치의 Raw 데이터 처리
void HID_Generic_Process(void)
{
  uint8_t raw_report[64];
  uint16_t report_length;

  if (USBH_HID_GetRawReport(&hUsbHostFS, raw_report, &report_length) == USBH_OK)
  {
    printf("Raw HID Report (%d bytes): ", report_length);
    for (int i = 0; i < report_length; i++)
    {
      printf("%02X ", raw_report[i]);
    }
    printf("\n");
  }
}
```

### 3. 마우스 커서 GUI 표시
```c
// LCD에 마우스 커서 표시 (STemWin 사용 시)
int32_t cursor_x = SCREEN_WIDTH / 2;
int32_t cursor_y = SCREEN_HEIGHT / 2;

void Draw_Mouse_Cursor(void)
{
  // 이전 커서 지우기
  GUI_SetColor(GUI_WHITE);
  GUI_DrawPixel(prev_cursor_x, prev_cursor_y);

  // 새 커서 그리기
  GUI_SetColor(GUI_BLACK);
  GUI_DrawPixel(cursor_x, cursor_y);

  // 클릭 상태 표시
  if (mouse_info.buttons & 0x01)
  {
    GUI_FillCircle(cursor_x, cursor_y, 3);
  }
}
```

## 응용 예제

### 1. USB 키보드 터미널
```c
// 키보드 입력을 UART로 전송
void Keyboard_Terminal(void)
{
  char ch = HID_KeyCode_to_ASCII(keybd_info.keys[0], keybd_info.modifiers);

  if (ch != 0)
  {
    HAL_UART_Transmit(&huart1, (uint8_t*)&ch, 1, 100);
  }
}
```

### 2. 마우스 기반 그리기 앱
```c
// 마우스로 LCD에 그리기
void Drawing_App(void)
{
  if (mouse_info.buttons & 0x01)  // 왼쪽 버튼 눌림
  {
    // 현재 위치에 점 그리기
    GUI_DrawPixel(cursor_x, cursor_y);
  }
}
```

### 3. 게임 컨트롤러
```c
// 키보드를 게임 컨트롤러로 사용
// WASD: 이동, Space: 점프, Enter: 액션
void Game_Controller(void)
{
  for (int i = 0; i < 6; i++)
  {
    switch (keybd_info.keys[i])
    {
      case 0x1A:  // W
        player_move_up();
        break;
      case 0x04:  // A
        player_move_left();
        break;
      case 0x16:  // S
        player_move_down();
        break;
      case 0x07:  // D
        player_move_right();
        break;
      case 0x2C:  // Space
        player_jump();
        break;
    }
  }
}
```

## 트러블슈팅

### HID 장치가 인식되지 않는 경우

1. **전원 확인**: VBUS 5V 공급 확인
   - USB-C OTG 커넥터 사용
   - 외부 전원 필요 시 USB 허브 사용

2. **클럭 설정 확인**: 48MHz USB 클럭
   ```c
   // USB FS 클럭 소스 설정
   RCC_PeriphCLKInitTypeDef PeriphClkInitStruct;
   PeriphClkInitStruct.PeriphClockSelection = RCC_PERIPHCLK_USB;
   PeriphClkInitStruct.UsbClockSelection = RCC_USBCLKSOURCE_PLL3;
   ```

3. **인터럽트 확인**: OTG_FS_IRQHandler 등록

### 데이터가 수신되지 않는 경우

1. **폴링 간격 확인**: USBH_Process() 호출 주기
2. **리포트 디스크립터 파싱**: 장치에 맞는 리포트 형식 확인
3. **버퍼 크기**: 리포트 크기에 맞는 버퍼 할당

### 특정 키/버튼이 동작하지 않는 경우

1. **키코드 매핑 확인**: HID Usage Table 참조
2. **수정자 키 처리**: Shift, Ctrl 등 확인
3. **N-Key Rollover**: 동시 입력 키 제한 확인

## 성능 고려사항

### 폴링 간격
- **마우스**: 8ms (125Hz) ~ 1ms (1000Hz)
- **키보드**: 8ms ~ 10ms
- **낮은 지연**: 높은 폴링 레이트 = 높은 CPU 부하

### 메모리 사용
- **HID 클래스 핸들**: 약 1KB
- **리포트 디스크립터**: 최대 512B
- **리포트 버퍼**: 장치에 따라 다름

### CPU 부하 최적화
```c
// 폴링 방식 대신 콜백 사용
void USBH_HID_EventCallback(USBH_HandleTypeDef *phost)
{
  // 새 리포트 도착 시에만 호출됨
  HID_Process();
}
```

## 참고 자료

- USB HID 사양: https://www.usb.org/hid
- HID Usage Tables: https://www.usb.org/document-library/hid-usage-tables-122
- STM32 USB Host Library: UM1720
- 예제: `STM32H745I-DISCO/Applications/USB_Host/HID_Standalone`
