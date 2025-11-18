# PWR_STOP_RTC - RTC 웨이크업을 이용한 STOP 모드

## 개요

이 예제는 STM32H745의 STOP 모드를 사용하여 초저전력 동작을 구현하는 방법을 보여줍니다. STOP 모드는 STANDBY 모드보다 빠른 웨이크업 시간을 제공하며, SRAM 내용을 유지하므로 컨텍스트를 보존할 수 있습니다. RTC 알람을 사용하여 주기적으로 웨이크업하여 센서 데이터를 수집하거나 통신을 수행하는 배터리 구동 장치에 이상적입니다.

## 하드웨어 요구사항

- **STM32H745I-DISCO 보드**
- **LED1**: PI12 (주황색) - 실행 상태 표시
- **LED2**: PI13 (초록색) - STOP 모드 웨이크업 표시
- **전류계**: 전력 소비 측정용 (IDD 점퍼)
- **RTC 백업 배터리**: VBAT 핀 (선택 사항)
- **시리얼 터미널**: 115200 baud (디버깅용)

## 주요 기능

### STOP 모드의 특징

**STANDBY vs STOP 모드 비교**
```
특성                │ STANDBY 모드    │ STOP 모드
──────────────────┼────────────────┼──────────────
전력 소비           │ 2.4 μA         │ ~100 μA
웨이크업 시간       │ ~5 ms          │ ~5-10 μs
SRAM 보존          │ 없음 (리셋)     │ 완전 보존
레지스터 보존       │ 없음           │ 완전 보존
IO 상태            │ 하이-Z         │ 유지
웨이크업 후         │ 리셋 (재시작)   │ 중단된 곳에서 계속
사용 사례          │ 장기간 대기     │ 빠른 응답 필요
```

### 응용 분야

1. **주기적 센서 모니터링**
   - 온도, 습도, 압력 센서 읽기
   - 매 5분마다 데이터 수집
   - SRAM에 데이터 버퍼링

2. **이벤트 기반 웨이크업**
   - GPIO 인터럽트로 웨이크업
   - 빠른 응답 시간 필요
   - 컨텍스트 유지 필요

3. **배터리 구동 IoT 장치**
   - LoRaWAN 노드
   - BLE 비콘
   - 스마트 센서

## STM32H745 전력 도메인 구조

### 도메인 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│                    STM32H745 Dual Core                   │
├─────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────┐     │
│  │           D1 Domain (CPU Domain)                │     │
│  │  ┌──────────────────────────────────────┐      │     │
│  │  │  Cortex-M7 @ 480 MHz                 │      │     │
│  │  │  - L1 Cache (16KB I + 16KB D)        │      │     │
│  │  │  - FPU, MPU                          │      │     │
│  │  └──────────────────────────────────────┘      │     │
│  │  - AXI SRAM (512KB)                             │     │
│  │  - DTCM (128KB), ITCM (64KB)                    │     │
│  │  - DMA1, DMA2, MDMA                             │     │
│  │  - Ethernet, USB_OTG_HS                         │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
│  ┌────────────────────────────────────────────────┐     │
│  │           D2 Domain (Peripheral Domain)         │     │
│  │  ┌──────────────────────────────────────┐      │     │
│  │  │  Cortex-M4 @ 240 MHz                 │      │     │
│  │  │  - FPU, MPU                          │      │     │
│  │  └──────────────────────────────────────┘      │     │
│  │  - AHB SRAM1, SRAM2, SRAM3 (288KB)             │     │
│  │  - ADC, DAC, DFSDM                              │     │
│  │  - TIM1-8, 12-17                                │     │
│  │  - SPI1-6, I2C1-4, USART1-6                    │     │
│  │  - BDMA                                         │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
│  ┌────────────────────────────────────────────────┐     │
│  │      D3 Domain (Autonomous Domain)              │     │
│  │  - SRD SRAM (64KB)                              │     │
│  │  - RTC, IWDG                                    │     │
│  │  - LPTIM1-5, LPUART1                            │     │
│  │  - I2C4 (autonomous mode)                       │     │
│  │  - SPI6 (autonomous mode)                       │     │
│  │  - BDMA                                         │     │
│  │  - COMParator, OpAmps                           │     │
│  └────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────┘
```

### STOP 모드의 도메인 동작

```
STOP 모드           │ D1 Domain    │ D2 Domain    │ D3 Domain
──────────────────┼─────────────┼─────────────┼─────────────
CPU 클럭           │ 정지         │ 정지         │ 정지
전원               │ ON (저전압)  │ ON (저전압)  │ ON
SRAM              │ 보존         │ 보존         │ 보존
주변장치           │ 정지         │ 정지         │ RTC, LPTIM 동작
웨이크업 소스      │ EXTI, RTC    │ EXTI, RTC    │ 항상 활성
```

## 동작 원리

### STOP 모드 진입/복귀 시퀀스

```
┌─────────────────┐
│   RUN 모드       │  전력: ~100 mA
│   (정상 동작)    │  - CPU 실행
│                 │  - 모든 주변장치 활성
└────────┬────────┘
         │
         │ Enter_STOP_Mode()
         ▼
┌─────────────────┐
│ STOP 모드 진입   │
│ 1. 주변장치 설정 │
│ 2. 웨이크업 설정 │  - RTC 알람 설정
│ 3. 전압 조정기   │  - Low-power regulator
│ 4. 플래그 클리어 │
└────────┬────────┘
         │
         │ __WFI() 또는 __WFE()
         ▼
┌─────────────────┐
│   STOP 모드      │  전력: ~100 μA
│                 │  - CPU 정지
│  대기 중...      │  - 대부분 주변장치 정지
│                 │  - SRAM 보존
│                 │  - RTC 동작
└────────┬────────┘
         │
         │ RTC Alarm 또는 EXTI
         ▼
┌─────────────────┐
│  하드웨어 웨이크업│  웨이크업 시간: 5-10 μs
│                 │  - 전압 조정기 활성화
│                 │  - 클럭 재시작
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   RUN 모드       │  전력: ~100 mA
│  (복귀 후 계속)  │  - SRAM 내용 보존
│                 │  - 레지스터 보존
│                 │  - 중단된 곳에서 계속
└─────────────────┘
```

### 타이밍 다이어그램

```
시간:    0s     2s     12s    14s    24s    26s
        │      │      │      │      │      │
실행:    ██████        ██████        ██████
        │      │      │      │      │      │
STOP:          ████████      ████████      ████████
        │      │      │      │      │      │
LED1:   ▔▔▔▔▔▔        ▔▔▔▔▔▔        ▔▔▔▔▔▔
        ▁▁▁▁▁▁████████▁▁▁▁▁▁████████▁▁▁▁▁▁████████

실행: 2초, STOP: 10초, 주기: 12초
평균 전력: (100mA × 2s + 0.1mA × 10s) / 12s ≈ 16.7 mA
```

## 코드 구현

### 1. 시스템 초기화

```c
#include "stm32h7xx_hal.h"

RTC_HandleTypeDef hrtc;
uint32_t wakeup_counter = 0;

int main(void)
{
  // HAL 초기화
  HAL_Init();

  // 시스템 클럭 설정 (480 MHz)
  SystemClock_Config();

  // GPIO 초기화
  GPIO_Init();

  // UART 초기화 (디버깅용)
  UART_Init();

  // RTC 초기화
  RTC_Init();

  printf("\n=================================\n");
  printf("STM32H745 STOP Mode with RTC Demo\n");
  printf("=================================\n\n");

  // 리셋 원인 확인
  Check_Reset_Source();

  while (1)
  {
    // 작업 수행 (2초)
    Perform_Work();

    // RTC 알람 설정 (10초 후)
    Set_RTC_Alarm(10);

    // STOP 모드 진입
    Enter_STOP_Mode();

    // 웨이크업 후 여기서 계속 실행됨
    wakeup_counter++;
    printf("\n--- Woke up from STOP mode (count: %lu) ---\n", wakeup_counter);

    // LED2 깜박임 (웨이크업 표시)
    HAL_GPIO_WritePin(GPIOI, GPIO_PIN_13, GPIO_PIN_SET);
    HAL_Delay(200);
    HAL_GPIO_WritePin(GPIOI, GPIO_PIN_13, GPIO_PIN_RESET);
  }
}
```

### 2. STOP 모드 진입 함수

```c
void Enter_STOP_Mode(void)
{
  printf("Entering STOP mode...\n");
  HAL_Delay(100);  // UART 전송 완료 대기

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 1. 주변장치 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // LED OFF
  HAL_GPIO_WritePin(GPIOI, GPIO_PIN_12 | GPIO_PIN_13, GPIO_PIN_RESET);

  // 사용하지 않는 주변장치 클럭 비활성화
  __HAL_RCC_GPIOA_CLK_DISABLE();
  __HAL_RCC_GPIOB_CLK_DISABLE();
  __HAL_RCC_GPIOC_CLK_DISABLE();
  // ... (필요에 따라)

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 2. 웨이크업 플래그 클리어
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  __HAL_PWR_CLEAR_FLAG(PWR_FLAG_STOP);
  __HAL_RTC_ALARM_CLEAR_FLAG(&hrtc, RTC_FLAG_ALRAF);
  __HAL_RTC_ALARM_EXTI_CLEAR_FLAG();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 3. STOP 모드 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // PWR 클럭 활성화
  __HAL_RCC_PWR_CLK_ENABLE();

  // D1 Domain: DStop mode
  HAL_PWREx_EnterSTOPMode(PWR_MAINREGULATOR_ON, PWR_STOPENTRY_WFI, PWR_D1_DOMAIN);

  // 또는 더 낮은 전력을 위해 Low-power regulator 사용
  // HAL_PWREx_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON, PWR_STOPENTRY_WFI, PWR_D1_DOMAIN);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 여기서 CPU 정지 - RTC 알람 대기
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // ... 웨이크업 발생 ...

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 4. 웨이크업 후 시스템 클럭 재설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // STOP 모드에서 깨어난 후 HSI로 동작 중
  // 원래 시스템 클럭으로 복귀
  SystemClock_Config();

  // 주변장치 클럭 재활성화
  __HAL_RCC_GPIOA_CLK_ENABLE();
  __HAL_RCC_GPIOB_CLK_ENABLE();
  __HAL_RCC_GPIOC_CLK_ENABLE();
  __HAL_RCC_GPIOI_CLK_ENABLE();

  printf("Woke up from STOP mode!\n");
}
```

### 3. 멀티 도메인 STOP 모드

STM32H745는 3개 도메인을 독립적으로 제어할 수 있습니다:

```c
void Enter_Multi_Domain_STOP_Mode(void)
{
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // D1 도메인: DStop 모드
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // Cortex-M7과 D1 주변장치 정지
  HAL_PWREx_EnterSTOPMode(PWR_MAINREGULATOR_ON,
                          PWR_STOPENTRY_WFI,
                          PWR_D1_DOMAIN);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // D2 도메인: DStop 모드
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // Cortex-M4와 D2 주변장치 정지
  HAL_PWREx_EnterSTOPMode(PWR_MAINREGULATOR_ON,
                          PWR_STOPENTRY_WFI,
                          PWR_D2_DOMAIN);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // D3 도메인: DStandby 모드
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // D3를 DStandby로 설정 (가장 낮은 전력)
  // RTC와 LPTIM만 동작
  HAL_PWREx_EnterSTANDBYMode(PWR_D3_DOMAIN);
}

// 전력 소비 비교:
// - 모든 도메인 RUN:           ~100 mA
// - D1 DStop, D2/D3 DStop:     ~100 μA
// - D1/D2 DStop, D3 DStandby:  ~50 μA
// - 모든 도메인 Standby:        ~2.4 μA
```

### 4. RTC 알람 설정

```c
void Set_RTC_Alarm(uint32_t seconds)
{
  RTC_AlarmTypeDef sAlarm = {0};
  RTC_TimeTypeDef sTime = {0};

  // 현재 시간 읽기
  HAL_RTC_GetTime(&hrtc, &sTime, RTC_FORMAT_BIN);

  printf("Current time: %02d:%02d:%02d\n",
         sTime.Hours, sTime.Minutes, sTime.Seconds);

  // 알람 시간 계산
  uint32_t total_seconds = sTime.Hours * 3600 +
                           sTime.Minutes * 60 +
                           sTime.Seconds +
                           seconds;

  uint8_t alarm_hours = (total_seconds / 3600) % 24;
  uint8_t alarm_minutes = (total_seconds / 60) % 60;
  uint8_t alarm_seconds = total_seconds % 60;

  printf("Alarm set: %02d:%02d:%02d (+%lu sec)\n",
         alarm_hours, alarm_minutes, alarm_seconds, seconds);

  // 알람 A 설정
  sAlarm.AlarmTime.Hours = alarm_hours;
  sAlarm.AlarmTime.Minutes = alarm_minutes;
  sAlarm.AlarmTime.Seconds = alarm_seconds;
  sAlarm.AlarmTime.SubSeconds = 0;
  sAlarm.AlarmTime.DayLightSaving = RTC_DAYLIGHTSAVING_NONE;
  sAlarm.AlarmTime.StoreOperation = RTC_STOREOPERATION_RESET;

  sAlarm.AlarmMask = RTC_ALARMMASK_DATEWEEKDAY;
  sAlarm.AlarmSubSecondMask = RTC_ALARMSUBSECONDMASK_ALL;
  sAlarm.AlarmDateWeekDaySel = RTC_ALARMDATEWEEKDAYSEL_DATE;
  sAlarm.AlarmDateWeekDay = 1;
  sAlarm.Alarm = RTC_ALARM_A;

  if (HAL_RTC_SetAlarm_IT(&hrtc, &sAlarm, RTC_FORMAT_BIN) != HAL_OK)
  {
    Error_Handler();
  }

  // EXTI 라인 17 설정 (RTC 알람)
  __HAL_RTC_ALARM_EXTI_ENABLE_IT();
  __HAL_RTC_ALARM_EXTI_ENABLE_RISING_EDGE();
}
```

### 5. RTC 웨이크업 타이머 사용 (대안)

더 간단한 주기적 웨이크업을 위해 RTC 웨이크업 타이머를 사용할 수 있습니다:

```c
void Set_RTC_WakeupTimer(uint32_t seconds)
{
  // 웨이크업 타이머 비활성화
  HAL_RTCEx_DeactivateWakeUpTimer(&hrtc);

  // 웨이크업 타이머 설정
  // Clock: ck_spre (1Hz)
  // WakeUp = seconds
  if (HAL_RTCEx_SetWakeUpTimer_IT(&hrtc,
                                   seconds,
                                   RTC_WAKEUPCLOCK_CK_SPRE_16BITS) != HAL_OK)
  {
    Error_Handler();
  }

  // EXTI 라인 19 설정 (RTC 웨이크업)
  __HAL_RTC_WAKEUPTIMER_EXTI_ENABLE_IT();
  __HAL_RTC_WAKEUPTIMER_EXTI_ENABLE_RISING_EDGE();

  printf("RTC Wakeup Timer: %lu seconds\n", seconds);
}

// 웨이크업 타이머 사용 예제
void STOP_Mode_With_WakeupTimer(void)
{
  // 10초마다 웨이크업
  Set_RTC_WakeupTimer(10);

  // STOP 모드 진입
  HAL_PWREx_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON,
                          PWR_STOPENTRY_WFI,
                          PWR_D1_DOMAIN);

  // 웨이크업 후
  SystemClock_Config();

  if (__HAL_RTC_WAKEUPTIMER_GET_FLAG(&hrtc, RTC_FLAG_WUTF))
  {
    printf("Woken by RTC Wakeup Timer\n");
    __HAL_RTC_WAKEUPTIMER_CLEAR_FLAG(&hrtc, RTC_FLAG_WUTF);
    __HAL_RTC_WAKEUPTIMER_EXTI_CLEAR_FLAG();
  }
}
```

## 고급 기능

### 1. EXTI 웨이크업 소스 추가

GPIO를 사용하여 외부 이벤트로 STOP 모드에서 웨이크업:

```c
void Configure_EXTI_Wakeup(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  // USER 버튼 (PC13) EXTI 설정
  __HAL_RCC_GPIOC_CLK_ENABLE();

  GPIO_InitStruct.Pin = GPIO_PIN_13;
  GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING;  // 하강 엣지
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);

  // EXTI 인터럽트 우선순위 설정
  HAL_NVIC_SetPriority(EXTI15_10_IRQn, 0, 0);
  HAL_NVIC_EnableIRQ(EXTI15_10_IRQn);

  printf("EXTI wakeup configured on PC13 (User Button)\n");
}

// EXTI 인터럽트 핸들러
void EXTI15_10_IRQHandler(void)
{
  HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_13);
}

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
  if (GPIO_Pin == GPIO_PIN_13)
  {
    printf("Woken by User Button!\n");
  }
}
```

### 2. 저전력 UART (LPUART) 웨이크업

```c
UART_HandleTypeDef hlpuart1;

void LPUART_WakeUp_Config(void)
{
  // LPUART1 초기화 (D3 도메인)
  hlpuart1.Instance = LPUART1;
  hlpuart1.Init.BaudRate = 9600;
  hlpuart1.Init.WordLength = UART_WORDLENGTH_8B;
  hlpuart1.Init.StopBits = UART_STOPBITS_1;
  hlpuart1.Init.Parity = UART_PARITY_NONE;
  hlpuart1.Init.Mode = UART_MODE_TX_RX;
  hlpuart1.Init.HwFlowCtl = UART_HWCONTROL_NONE;

  if (HAL_UART_Init(&hlpuart1) != HAL_OK)
  {
    Error_Handler();
  }

  // 웨이크업 조건: 특정 문자 수신 (예: 'W')
  HAL_UARTEx_StopModeWakeUpSourceConfig(&hlpuart1,
                                         UART_WAKEUP_ON_READDATA_NONEMPTY);

  // LPUART 웨이크업 활성화
  HAL_UARTEx_EnableStopMode(&hlpuart1);

  printf("LPUART wakeup configured\n");
}
```

### 3. 다중 웨이크업 소스 관리

```c
typedef enum {
  WAKEUP_NONE = 0,
  WAKEUP_RTC_ALARM,
  WAKEUP_RTC_WAKEUP,
  WAKEUP_EXTI_BUTTON,
  WAKEUP_LPUART
} WakeupSource_t;

WakeupSource_t Check_Wakeup_Source(void)
{
  // RTC 알람
  if (__HAL_RTC_ALARM_GET_FLAG(&hrtc, RTC_FLAG_ALRAF))
  {
    __HAL_RTC_ALARM_CLEAR_FLAG(&hrtc, RTC_FLAG_ALRAF);
    __HAL_RTC_ALARM_EXTI_CLEAR_FLAG();
    printf("Wakeup: RTC Alarm\n");
    return WAKEUP_RTC_ALARM;
  }

  // RTC 웨이크업 타이머
  if (__HAL_RTC_WAKEUPTIMER_GET_FLAG(&hrtc, RTC_FLAG_WUTF))
  {
    __HAL_RTC_WAKEUPTIMER_CLEAR_FLAG(&hrtc, RTC_FLAG_WUTF);
    __HAL_RTC_WAKEUPTIMER_EXTI_CLEAR_FLAG();
    printf("Wakeup: RTC Wakeup Timer\n");
    return WAKEUP_RTC_WAKEUP;
  }

  // EXTI (버튼)
  if (__HAL_GPIO_EXTI_GET_IT(GPIO_PIN_13))
  {
    __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_13);
    printf("Wakeup: User Button (EXTI)\n");
    return WAKEUP_EXTI_BUTTON;
  }

  printf("Wakeup: Unknown source\n");
  return WAKEUP_NONE;
}

// 메인 루프에서 사용
void main(void)
{
  // ... 초기화 ...

  while (1)
  {
    // STOP 모드 진입
    Enter_STOP_Mode();

    // 웨이크업 소스 확인
    WakeupSource_t source = Check_Wakeup_Source();

    // 소스에 따라 다른 동작 수행
    switch (source)
    {
      case WAKEUP_RTC_ALARM:
        // 정기 작업 (센서 읽기 등)
        Perform_Periodic_Task();
        Set_RTC_Alarm(60);  // 1분 후 다시 웨이크업
        break;

      case WAKEUP_EXTI_BUTTON:
        // 사용자 인터랙션
        Handle_User_Input();
        Set_RTC_Alarm(10);  // 10초 후 다시 STOP
        break;

      default:
        Set_RTC_Alarm(60);
        break;
    }
  }
}
```

### 4. SRAM 데이터 보존

STOP 모드에서는 SRAM 내용이 보존되므로 데이터를 유지할 수 있습니다:

```c
// SRAM에 센서 데이터 버퍼 유지
#define BUFFER_SIZE  100
typedef struct {
  uint32_t timestamp;
  int16_t temperature;
  uint16_t humidity;
  uint16_t pressure;
} SensorData_t;

SensorData_t sensor_buffer[BUFFER_SIZE];
uint32_t buffer_index = 0;

void Collect_Sensor_Data(void)
{
  // 센서 읽기
  int16_t temp = Read_Temperature();
  uint16_t hum = Read_Humidity();
  uint16_t press = Read_Pressure();

  // RTC 타임스탬프
  RTC_TimeTypeDef sTime;
  RTC_DateTypeDef sDate;
  HAL_RTC_GetTime(&hrtc, &sTime, RTC_FORMAT_BIN);
  HAL_RTC_GetDate(&hrtc, &sDate, RTC_FORMAT_BIN);

  uint32_t timestamp = sDate.Date * 86400 +
                       sTime.Hours * 3600 +
                       sTime.Minutes * 60 +
                       sTime.Seconds;

  // 버퍼에 저장
  sensor_buffer[buffer_index].timestamp = timestamp;
  sensor_buffer[buffer_index].temperature = temp;
  sensor_buffer[buffer_index].humidity = hum;
  sensor_buffer[buffer_index].pressure = press;

  buffer_index = (buffer_index + 1) % BUFFER_SIZE;

  printf("Data collected: T=%d.%dC H=%d%% P=%dhPa\n",
         temp/10, temp%10, hum/10, press);

  // 버퍼가 가득 차면 전송
  if (buffer_index == 0)
  {
    printf("Buffer full - transmitting data...\n");
    Transmit_Sensor_Data();
  }
}
```

### 5. 배터리 전압 모니터링

```c
ADC_HandleTypeDef hadc3;

void Battery_Monitor_Init(void)
{
  ADC_ChannelConfTypeDef sConfig = {0};

  // ADC3 초기화
  hadc3.Instance = ADC3;
  hadc3.Init.ClockPrescaler = ADC_CLOCK_ASYNC_DIV4;
  hadc3.Init.Resolution = ADC_RESOLUTION_12B;
  hadc3.Init.ScanConvMode = ADC_SCAN_DISABLE;
  hadc3.Init.EOCSelection = ADC_EOC_SINGLE_CONV;
  hadc3.Init.LowPowerAutoWait = DISABLE;
  hadc3.Init.ContinuousConvMode = DISABLE;
  hadc3.Init.NbrOfConversion = 1;
  hadc3.Init.DiscontinuousConvMode = DISABLE;
  hadc3.Init.ExternalTrigConv = ADC_SOFTWARE_START;
  hadc3.Init.ConversionDataManagement = ADC_CONVERSIONDATA_DR;
  hadc3.Init.Overrun = ADC_OVR_DATA_OVERWRITTEN;

  if (HAL_ADC_Init(&hadc3) != HAL_OK)
  {
    Error_Handler();
  }

  // 내부 VBAT 채널 설정
  sConfig.Channel = ADC_CHANNEL_VBAT;
  sConfig.Rank = ADC_REGULAR_RANK_1;
  sConfig.SamplingTime = ADC_SAMPLETIME_810CYCLES_5;
  sConfig.SingleDiff = ADC_SINGLE_ENDED;
  sConfig.OffsetNumber = ADC_OFFSET_NONE;
  sConfig.Offset = 0;

  if (HAL_ADC_ConfigChannel(&hadc3, &sConfig) != HAL_OK)
  {
    Error_Handler();
  }

  // ADC 캘리브레이션
  HAL_ADCEx_Calibration_Start(&hadc3, ADC_CALIB_OFFSET, ADC_SINGLE_ENDED);
}

uint16_t Read_Battery_Voltage(void)
{
  uint32_t adc_value;

  // ADC 변환 시작
  HAL_ADC_Start(&hadc3);

  // 변환 완료 대기
  if (HAL_ADC_PollForConversion(&hadc3, 100) == HAL_OK)
  {
    adc_value = HAL_ADC_GetValue(&hadc3);
  }
  else
  {
    adc_value = 0;
  }

  HAL_ADC_Stop(&hadc3);

  // VBAT = ADC_value * 3.3V / 4095 * 4 (VBAT divider)
  // 결과를 mV 단위로 반환
  uint16_t vbat_mv = (adc_value * 3300 * 4) / 4095;

  printf("Battery: %d mV\n", vbat_mv);

  return vbat_mv;
}

// 저전압 감지
void Check_Battery_Level(void)
{
  uint16_t vbat = Read_Battery_Voltage();

  if (vbat < 2700)  // 2.7V 이하
  {
    printf("WARNING: Low battery! Entering ultra-low power mode.\n");

    // 긴급 데이터 저장
    Save_Critical_Data();

    // 웨이크업 간격 증가 (배터리 절약)
    Set_RTC_Alarm(3600);  // 1시간마다

    // 불필요한 기능 비활성화
    Disable_Non_Critical_Functions();
  }
  else if (vbat < 3000)  // 3.0V 이하
  {
    printf("Battery low. Reducing activity.\n");

    // 웨이크업 간격 증가
    Set_RTC_Alarm(300);  // 5분마다
  }
}
```

## 전력 소비 측정 및 최적화

### 전력 측정 방법

1. **IDD 점퍼 사용**
   - JP2 제거
   - 전류계 연결 (mA ~ μA 범위)
   - 실시간 전력 측정

2. **측정 결과 예시**

```
동작 모드              │ 전력 소비    │ 비고
─────────────────────┼─────────────┼──────────────────
RUN (480MHz, 모든 기능) │ ~250 mA      │ 최대 성능
RUN (64MHz, 최소 기능)  │ ~30 mA       │ 저속 동작
STOP (Main Reg)       │ ~100 μA      │ 빠른 웨이크업
STOP (LP Reg)         │ ~50 μA       │ 저전력, 느린 웨이크업
STANDBY               │ ~2.4 μA      │ 최저 전력
```

### 전력 최적화 체크리스트

```c
void Optimize_Power_Consumption(void)
{
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 1. 클럭 최적화
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // 사용하지 않는 주변장치 클럭 비활성화
  __HAL_RCC_GPIOA_CLK_DISABLE();
  __HAL_RCC_GPIOB_CLK_DISABLE();
  __HAL_RCC_GPIOC_CLK_DISABLE();
  __HAL_RCC_GPIOD_CLK_DISABLE();
  __HAL_RCC_GPIOE_CLK_DISABLE();
  __HAL_RCC_GPIOF_CLK_DISABLE();
  __HAL_RCC_GPIOG_CLK_DISABLE();
  __HAL_RCC_GPIOH_CLK_DISABLE();
  // GPIOI는 LED용으로 유지

  __HAL_RCC_DMA1_CLK_DISABLE();
  __HAL_RCC_DMA2_CLK_DISABLE();
  __HAL_RCC_MDMA_CLK_DISABLE();

  __HAL_RCC_SPI1_CLK_DISABLE();
  __HAL_RCC_SPI2_CLK_DISABLE();
  __HAL_RCC_SPI3_CLK_DISABLE();

  __HAL_RCC_I2C1_CLK_DISABLE();
  __HAL_RCC_I2C2_CLK_DISABLE();
  __HAL_RCC_I2C3_CLK_DISABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 2. GPIO 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // 모든 GPIO를 analog 모드로 설정 (최저 전력)
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  __HAL_RCC_GPIOA_CLK_ENABLE();
  GPIO_InitStruct.Pin = GPIO_PIN_All;
  GPIO_InitStruct.Mode = GPIO_MODE_ANALOG;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
  __HAL_RCC_GPIOA_CLK_DISABLE();

  // GPIOB-H도 동일하게 설정
  // ...

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 3. Flash 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // Flash power-down 활성화 (STOP 모드 시)
  __HAL_FLASH_SLEEP_POWERDOWN_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 4. 전압 조정기 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // Low-power regulator 사용 (STOP 모드 진입 시)
  // Main regulator: ~100 μA
  // LP regulator:   ~50 μA

  printf("Power optimization completed\n");
}
```

## 배터리 수명 계산

### 시나리오 1: 주기적 센서 읽기

```
동작 패턴:
- 센서 읽기: 2초 (100 mA)
- STOP 모드: 58초 (0.05 mA)
- 주기: 60초 (1분마다)

평균 전력:
= (100mA × 2s + 0.05mA × 58s) / 60s
= (200 + 2.9) / 60
= 202.9 / 60
≈ 3.38 mA

배터리 수명 (2000 mAh):
= 2000 mAh / 3.38 mA
≈ 592 시간
≈ 24.7 일
```

### 시나리오 2: LoRaWAN 노드

```
동작 패턴:
- 센서 읽기: 1초 (30 mA)
- LoRa 전송: 3초 (120 mA)
- STOP 모드: 596초 (0.05 mA)
- 주기: 600초 (10분마다)

평균 전력:
= (30mA × 1s + 120mA × 3s + 0.05mA × 596s) / 600s
= (30 + 360 + 29.8) / 600
= 419.8 / 600
≈ 0.70 mA

배터리 수명 (2000 mAh):
= 2000 mAh / 0.70 mA
≈ 2857 시간
≈ 119 일
≈ 4개월
```

### 시나리오 3: 긴급 이벤트 감지

```
동작 패턴:
- 이벤트 대기: STOP 모드 (0.05 mA)
- 이벤트 발생 시: 5초 동작 (100 mA)
- 이벤트 빈도: 하루 10회

일일 에너지 소비:
= 0.05mA × 24h + (100mA × 5s × 10) / 3600s/h
= 1.2 mAh + 1.39 mAh
= 2.59 mAh/day

배터리 수명 (2000 mAh):
= 2000 mAh / 2.59 mAh/day
≈ 772 일
≈ 2.1 년
```

## 실전 응용 예제

### 예제 1: 환경 모니터링 스테이션

```c
#define SENSOR_INTERVAL  300  // 5분마다 센서 읽기

typedef struct {
  float temperature;
  float humidity;
  uint16_t battery_mv;
  uint32_t timestamp;
} EnvironmentData_t;

EnvironmentData_t env_buffer[24];  // 2시간 분량 (5분 × 24)
uint8_t env_index = 0;

void Environmental_Monitoring_Task(void)
{
  // RTC 초기화
  RTC_Init();

  // 센서 초기화
  DHT22_Init();  // 온습도 센서
  Battery_Monitor_Init();

  printf("Environmental Monitoring Station Started\n");

  while (1)
  {
    // 센서 데이터 수집
    env_buffer[env_index].temperature = DHT22_Read_Temperature();
    env_buffer[env_index].humidity = DHT22_Read_Humidity();
    env_buffer[env_index].battery_mv = Read_Battery_Voltage();

    // 타임스탬프 기록
    RTC_TimeTypeDef sTime;
    RTC_DateTypeDef sDate;
    HAL_RTC_GetTime(&hrtc, &sTime, RTC_FORMAT_BIN);
    HAL_RTC_GetDate(&hrtc, &sDate, RTC_FORMAT_BIN);
    env_buffer[env_index].timestamp = sDate.Date * 86400 +
                                       sTime.Hours * 3600 +
                                       sTime.Minutes * 60 +
                                       sTime.Seconds;

    printf("[%02d:%02d:%02d] T=%.1fC H=%.1f%% Bat=%dmV\n",
           sTime.Hours, sTime.Minutes, sTime.Seconds,
           env_buffer[env_index].temperature,
           env_buffer[env_index].humidity,
           env_buffer[env_index].battery_mv);

    env_index++;

    // 버퍼 가득 차면 전송
    if (env_index >= 24)
    {
      printf("Transmitting 2 hours of data...\n");
      LoRa_Transmit_Data((uint8_t*)env_buffer, sizeof(env_buffer));
      env_index = 0;
    }

    // 배터리 체크
    Check_Battery_Level();

    // 다음 웨이크업 설정
    Set_RTC_Alarm(SENSOR_INTERVAL);

    // STOP 모드 진입
    Enter_STOP_Mode();
  }
}
```

### 예제 2: 스마트 도어 센서

```c
#define DOOR_CHECK_INTERVAL  60  // 1분마다 상태 확인

typedef enum {
  DOOR_CLOSED = 0,
  DOOR_OPEN = 1
} DoorState_t;

DoorState_t last_door_state = DOOR_CLOSED;
uint32_t door_open_count = 0;

void Smart_Door_Sensor_Task(void)
{
  // GPIO 초기화 (도어 센서: Hall sensor on PA0)
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  __HAL_RCC_GPIOA_CLK_ENABLE();

  GPIO_InitStruct.Pin = GPIO_PIN_0;
  GPIO_InitStruct.Mode = GPIO_MODE_IT_RISING_FALLING;
  GPIO_InitStruct.Pull = GPIO_PULLUP;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  // EXTI 인터럽트 설정
  HAL_NVIC_SetPriority(EXTI0_IRQn, 0, 0);
  HAL_NVIC_EnableIRQ(EXTI0_IRQn);

  // RTC 초기화
  RTC_Init();

  printf("Smart Door Sensor Started\n");

  while (1)
  {
    // 도어 상태 확인
    DoorState_t current_state = HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0);

    if (current_state != last_door_state)
    {
      // 상태 변경 감지
      if (current_state == DOOR_OPEN)
      {
        printf("DOOR OPENED!\n");
        door_open_count++;

        // 즉시 알림 전송
        BLE_Send_Notification("Door Opened");

        // LED 켜기
        HAL_GPIO_WritePin(GPIOI, GPIO_PIN_12, GPIO_PIN_SET);
      }
      else
      {
        printf("DOOR CLOSED\n");

        // LED 끄기
        HAL_GPIO_WritePin(GPIOI, GPIO_PIN_12, GPIO_PIN_RESET);
      }

      last_door_state = current_state;
    }

    // 정기 상태 보고 (1시간마다)
    static uint32_t report_counter = 0;
    report_counter++;

    if (report_counter >= 60)  // 60분
    {
      printf("Hourly report: Door opened %lu times\n", door_open_count);
      BLE_Send_Status_Report(door_open_count);
      door_open_count = 0;
      report_counter = 0;
    }

    // 배터리 체크 (매일 1회)
    if (report_counter % 1440 == 0)  // 24시간
    {
      Check_Battery_Level();
    }

    // STOP 모드 진입 (1분 후 또는 EXTI로 즉시 웨이크업)
    Set_RTC_Alarm(DOOR_CHECK_INTERVAL);
    Enter_STOP_Mode();
  }
}

// EXTI 인터럽트 핸들러
void EXTI0_IRQHandler(void)
{
  HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_0);
}

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
  if (GPIO_Pin == GPIO_PIN_0)
  {
    // STOP 모드에서 즉시 깨어남
    // 메인 루프에서 상태 확인 및 처리
  }
}
```

### 예제 3: BLE 비콘

```c
#define BEACON_INTERVAL  1000  // 1초마다 비콘 전송

typedef struct {
  uint8_t uuid[16];
  uint16_t major;
  uint16_t minor;
  int8_t tx_power;
} BeaconData_t;

BeaconData_t beacon_data = {
  .uuid = {0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08,
           0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0x0E, 0x0F, 0x10},
  .major = 100,
  .minor = 1,
  .tx_power = -59  // RSSI at 1m
};

void BLE_Beacon_Task(void)
{
  // BLE 모듈 초기화
  BLE_Init();

  // RTC 웨이크업 타이머 사용 (더 간단)
  RTC_Init();

  printf("BLE Beacon Started\n");
  printf("UUID: ");
  for (int i = 0; i < 16; i++)
  {
    printf("%02X", beacon_data.uuid[i]);
  }
  printf("\nMajor: %d, Minor: %d\n", beacon_data.major, beacon_data.minor);

  while (1)
  {
    // BLE 비콘 광고 패킷 전송
    BLE_Transmit_Beacon(&beacon_data);

    printf("Beacon transmitted\n");

    // LED 짧게 깜박임
    HAL_GPIO_WritePin(GPIOI, GPIO_PIN_13, GPIO_PIN_SET);
    HAL_Delay(10);
    HAL_GPIO_WritePin(GPIOI, GPIO_PIN_13, GPIO_PIN_RESET);

    // BLE 모듈 저전력 모드
    BLE_Sleep();

    // STOP 모드 진입 (1초 후 웨이크업)
    Set_RTC_WakeupTimer(1);
    Enter_STOP_Mode();

    // BLE 모듈 웨이크업
    BLE_Wakeup();
  }
}

// 전력 소비:
// - BLE 전송: 10ms × 15mA = 0.15mA·s
// - STOP 모드: 990ms × 0.05mA = 0.0495mA·s
// - 평균: (0.15 + 0.0495) / 1s = 0.1995mA
// - 배터리 수명 (220mAh): 220/0.1995 ≈ 1103시간 ≈ 46일
```

## 트러블슈팅

### 문제 1: STOP 모드에서 웨이크업되지 않음

```c
void Debug_STOP_Mode_Issue(void)
{
  // 1. RTC 인터럽트 확인
  if (!(RTC->CR & RTC_CR_ALRAIE))
  {
    printf("ERROR: RTC Alarm interrupt not enabled!\n");
    __HAL_RTC_ALARM_ENABLE_IT(&hrtc, RTC_IT_ALRA);
  }

  // 2. EXTI 라인 확인
  if (!(EXTI->IMR1 & EXTI_IMR1_IM17))
  {
    printf("ERROR: EXTI line 17 (RTC Alarm) not enabled!\n");
    __HAL_RTC_ALARM_EXTI_ENABLE_IT();
  }

  // 3. NVIC 확인
  if (!(NVIC->ISER[RTC_Alarm_IRQn >> 5] & (1 << (RTC_Alarm_IRQn & 0x1F))))
  {
    printf("ERROR: RTC Alarm NVIC not enabled!\n");
    HAL_NVIC_EnableIRQ(RTC_Alarm_IRQn);
  }

  // 4. 알람 설정 확인
  RTC_AlarmTypeDef sAlarm;
  HAL_RTC_GetAlarm(&hrtc, &sAlarm, RTC_ALARM_A, RTC_FORMAT_BIN);
  printf("Alarm A: %02d:%02d:%02d\n",
         sAlarm.AlarmTime.Hours,
         sAlarm.AlarmTime.Minutes,
         sAlarm.AlarmTime.Seconds);
}
```

### 문제 2: 웨이크업 후 시스템이 불안정함

```c
void Debug_Clock_After_Wakeup(void)
{
  // STOP 모드 후 클럭 확인
  uint32_t sysclk = HAL_RCC_GetSysClockFreq();
  uint32_t hclk = HAL_RCC_GetHCLKFreq();

  printf("SysClk: %lu Hz\n", sysclk);
  printf("HCLK: %lu Hz\n", hclk);

  if (sysclk != 480000000)  // 기대값: 480MHz
  {
    printf("WARNING: System clock not restored!\n");
    printf("Re-configuring system clock...\n");
    SystemClock_Config();

    // 재확인
    sysclk = HAL_RCC_GetSysClockFreq();
    printf("SysClk after restore: %lu Hz\n", sysclk);
  }
}
```

### 문제 3: 전력 소비가 예상보다 높음

```c
void Debug_Power_Consumption(void)
{
  printf("\n=== Power Consumption Debug ===\n");

  // 1. 활성화된 클럭 확인
  printf("Active clocks:\n");
  if (RCC->AHB1ENR) printf("  AHB1ENR: 0x%08lX\n", RCC->AHB1ENR);
  if (RCC->AHB2ENR) printf("  AHB2ENR: 0x%08lX\n", RCC->AHB2ENR);
  if (RCC->AHB3ENR) printf("  AHB3ENR: 0x%08lX\n", RCC->AHB3ENR);
  if (RCC->AHB4ENR) printf("  AHB4ENR: 0x%08lX\n", RCC->AHB4ENR);
  if (RCC->APB1LENR) printf("  APB1LENR: 0x%08lX\n", RCC->APB1LENR);
  if (RCC->APB1HENR) printf("  APB1HENR: 0x%08lX\n", RCC->APB1HENR);
  if (RCC->APB2ENR) printf("  APB2ENR: 0x%08lX\n", RCC->APB2ENR);
  if (RCC->APB3ENR) printf("  APB3ENR: 0x%08lX\n", RCC->APB3ENR);
  if (RCC->APB4ENR) printf("  APB4ENR: 0x%08lX\n", RCC->APB4ENR);

  // 2. GPIO 모드 확인 (아날로그 모드가 최저 전력)
  printf("\nGPIO modes:\n");
  printf("  GPIOA_MODER: 0x%08lX\n", GPIOA->MODER);
  printf("  GPIOB_MODER: 0x%08lX\n", GPIOB->MODER);
  // 0xFFFFFFFF = 모든 핀 아날로그 모드 (최적)

  // 3. 전압 조정기 상태
  printf("\nRegulator:\n");
  if (PWR->CR1 & PWR_CR1_LPDS)
  {
    printf("  Low-power regulator enabled (good)\n");
  }
  else
  {
    printf("  Main regulator enabled (higher power)\n");
  }

  // 4. Flash power-down 확인
  if (FLASH->ACR & FLASH_ACR_SLEEP_PD)
  {
    printf("  Flash power-down in STOP: enabled (good)\n");
  }
  else
  {
    printf("  Flash power-down in STOP: disabled\n");
    printf("  Enabling...\n");
    __HAL_FLASH_SLEEP_POWERDOWN_ENABLE();
  }

  printf("================================\n\n");
}
```

### 문제 4: RTC 시간이 부정확함

```c
// LSI 주파수 측정 및 보정
uint32_t Measure_LSI_Frequency(void)
{
  // TIM5를 사용하여 LSI 주파수 측정
  // LSI는 명목상 32kHz이지만 ±5% 오차 가능

  TIM_HandleTypeDef htim5;
  uint32_t lsi_freq;

  // TIM5 클럭 활성화
  __HAL_RCC_TIM5_CLK_ENABLE();

  // TIM5 초기화 (LSI 측정용)
  htim5.Instance = TIM5;
  htim5.Init.Prescaler = 0;
  htim5.Init.CounterMode = TIM_COUNTERMODE_UP;
  htim5.Init.Period = 0xFFFFFFFF;
  htim5.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
  HAL_TIM_Base_Init(&htim5);

  // 입력 캡처로 LSI 측정
  // ... (구현 세부사항)

  lsi_freq = 32000;  // 측정된 값

  printf("LSI frequency measured: %lu Hz\n", lsi_freq);

  return lsi_freq;
}

void Calibrate_RTC_Prescaler(void)
{
  uint32_t lsi_freq = Measure_LSI_Frequency();

  // 프리스케일러 재계산
  uint32_t prediv_a = 127;
  uint32_t prediv_s = (lsi_freq / (prediv_a + 1)) - 1;

  printf("Calibrating RTC prescaler:\n");
  printf("  AsynchPrediv: %lu\n", prediv_a);
  printf("  SynchPrediv: %lu\n", prediv_s);

  hrtc.Init.AsynchPrediv = prediv_a;
  hrtc.Init.SynchPrediv = prediv_s;

  if (HAL_RTC_Init(&hrtc) != HAL_OK)
  {
    Error_Handler();
  }

  printf("RTC calibration completed\n");
}
```

## 참고 자료

- **RM0399**: STM32H745 Reference Manual
  - Chapter 6: Power Controller (PWR)
  - Chapter 47: Real-Time Clock (RTC)
  - Chapter 8: Low-power modes

- **AN4749**: Managing low-power modes in STM32H7 series

- **AN4759**: Using the hardware RTC in STM32 MCUs

- **STM32CubeMX**: 자동 코드 생성 및 전력 계산 도구

## 관련 예제

- **PWR_STANDBY_RTC**: STANDBY 모드 (더 낮은 전력, SRAM 미보존)
- **PWR_Domain3SystemControl**: D3 도메인 자율 동작
- **PWR_D1ON_D2OFF**: 도메인별 전력 제어
- **RTC_Alarm**: RTC 알람 기본 사용법
