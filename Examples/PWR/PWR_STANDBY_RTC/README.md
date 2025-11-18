# PWR_STANDBY_RTC - RTC 알람을 이용한 스탠바이 웨이크업

## 개요

이 예제는 STM32H745의 RTC(Real-Time Clock) 알람을 사용하여 스탠바이 모드에서 자동으로 깨어나는 방법을 보여줍니다. 주기적으로 스탠바이 모드에 진입했다가 RTC 알람에 의해 깨어나는 주기적 동작 패턴을 구현합니다.

## 하드웨어 요구사항

- **STM32H745I-DISCO 보드**
- **LED1**: PI12 (주황색) - 실행 상태 표시
- **LED2**: PI13 (초록색) - RTC 웨이크업 표시
- **RTC 백업 배터리**: VBAT 핀 (선택 사항, CR2032)
- **전류계**: 전력 소비 측정용 (선택 사항)

## 주요 기능

### RTC 알람 웨이크업
- **자동 웨이크업**: 설정된 시간 후 자동으로 스탠바이에서 복귀
- **주기적 동작**: Sleep - Wake - Work - Sleep 사이클
- **저전력 타이머**: 외부 크리스탈 없이도 동작 (LSI 사용)
- **정확한 타이밍**: LSE 사용 시 ±5 ppm 정확도

### 응용 분야
- **센서 데이터 수집**: 주기적으로 깨어나 센서 읽기
- **배터리 구동 장치**: 긴 수명을 위한 저전력 동작
- **스케줄링**: 정해진 시간에 작업 수행
- **무선 통신**: 주기적 데이터 전송

## 동작 원리

### 동작 사이클
```
┌─────────────┐
│   Wakeup    │
│  (RTC Alarm)│
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Initialize │  시스템 초기화
│   System    │  RTC 시간 확인
└──────┬──────┘
       │
       ▼
┌─────────────┐
│    Work     │  LED 깜박임
│  (5 seconds)│  센서 읽기 등
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Set Alarm   │  10초 후 알람 설정
│  (10 sec)   │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  STANDBY    │  전력 소비: 2.4 μA
│    Mode     │  (10초간 대기)
└──────┬──────┘
       │
       └──────────► (반복)
```

### 타이밍 다이어그램
```
Time:  0s    5s      15s   20s      30s   35s
       │     │       │     │        │     │
Run:   ████████      ████████       ████████
       │     │       │     │        │     │
Standby:      ████████      ████████      ████████
       │     │       │     │        │     │
LED1:  ▔▔▔▔▔▔        ▔▔▔▔▔▔         ▔▔▔▔▔▔
       ▁▁▁▁▁▁████████▁▁▁▁▁▁████████▁▁▁▁▁▁████████

Work: 5초, Standby: 10초, 주기: 15초
```

## 코드 구조

### 1. RTC 초기화

```c
RTC_HandleTypeDef hrtc;

void RTC_Init(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_PeriphCLKInitTypeDef PeriphClkInitStruct = {0};

  // PWR 클럭 활성화
  __HAL_RCC_PWR_CLK_ENABLE();

  // 백업 도메인 접근 활성화
  HAL_PWR_EnableBkUpAccess();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // RTC 클럭 소스 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // 옵션 1: LSI (내부 32kHz, 정확도 낮음)
  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_LSI;
  RCC_OscInitStruct.LSIState = RCC_LSI_ON;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_NONE;

  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  PeriphClkInitStruct.PeriphClockSelection = RCC_PERIPHCLK_RTC;
  PeriphClkInitStruct.RTCClockSelection = RCC_RTCCLKSOURCE_LSI;

  // 옵션 2: LSE (외부 32.768kHz, 정확도 높음)
  // RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_LSE;
  // RCC_OscInitStruct.LSEState = RCC_LSE_ON;
  // PeriphClkInitStruct.RTCClockSelection = RCC_RTCCLKSOURCE_LSE;

  if (HAL_RCCEx_PeriphCLKConfig(&PeriphClkInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // RTC 초기화
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  __HAL_RCC_RTC_ENABLE();

  hrtc.Instance = RTC;
  hrtc.Init.HourFormat = RTC_HOURFORMAT_24;

  // LSI 32kHz 기준 프리스케일러
  // RTC_CLK = 32kHz / ((127+1) * (249+1)) = 1Hz
  hrtc.Init.AsynchPrediv = 127;
  hrtc.Init.SynchPrediv = 249;

  hrtc.Init.OutPut = RTC_OUTPUT_DISABLE;
  hrtc.Init.OutPutPolarity = RTC_OUTPUT_POLARITY_HIGH;
  hrtc.Init.OutPutType = RTC_OUTPUT_TYPE_OPENDRAIN;

  if (HAL_RTC_Init(&hrtc) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 초기 시간 설정 (최초 부팅 시)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  if (HAL_RTCEx_BKUPRead(&hrtc, RTC_BKP_DR0) != 0x32F2)
  {
    RTC_TimeTypeDef sTime = {0};
    RTC_DateTypeDef sDate = {0};

    // 초기 시간: 00:00:00
    sTime.Hours = 0;
    sTime.Minutes = 0;
    sTime.Seconds = 0;
    sTime.DayLightSaving = RTC_DAYLIGHTSAVING_NONE;
    sTime.StoreOperation = RTC_STOREOPERATION_RESET;
    HAL_RTC_SetTime(&hrtc, &sTime, RTC_FORMAT_BIN);

    // 초기 날짜: 2024-01-01 (Monday)
    sDate.Year = 24;
    sDate.Month = RTC_MONTH_JANUARY;
    sDate.Date = 1;
    sDate.WeekDay = RTC_WEEKDAY_MONDAY;
    HAL_RTC_SetDate(&hrtc, &sDate, RTC_FORMAT_BIN);

    // 초기화 완료 표시
    HAL_RTCEx_BKUPWrite(&hrtc, RTC_BKP_DR0, 0x32F2);

    printf("RTC initialized\n");
  }
  else
  {
    printf("RTC already initialized\n");
  }
}
```

### 2. RTC 알람 설정

```c
void RTC_Alarm_Set(uint32_t seconds)
{
  RTC_AlarmTypeDef sAlarm = {0};
  RTC_TimeTypeDef sTime = {0};

  // 현재 시간 읽기
  HAL_RTC_GetTime(&hrtc, &sTime, RTC_FORMAT_BIN);

  printf("Current time: %02d:%02d:%02d\n",
         sTime.Hours, sTime.Minutes, sTime.Seconds);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 알람 시간 계산
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  uint32_t total_seconds = sTime.Hours * 3600 +
                           sTime.Minutes * 60 +
                           sTime.Seconds +
                           seconds;

  uint8_t alarm_hours = (total_seconds / 3600) % 24;
  uint8_t alarm_minutes = (total_seconds / 60) % 60;
  uint8_t alarm_seconds = total_seconds % 60;

  printf("Alarm set for %02d:%02d:%02d (%lu seconds)\n",
         alarm_hours, alarm_minutes, alarm_seconds, seconds);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 알람 A 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  sAlarm.AlarmTime.Hours = alarm_hours;
  sAlarm.AlarmTime.Minutes = alarm_minutes;
  sAlarm.AlarmTime.Seconds = alarm_seconds;
  sAlarm.AlarmTime.SubSeconds = 0;
  sAlarm.AlarmTime.DayLightSaving = RTC_DAYLIGHTSAVING_NONE;
  sAlarm.AlarmTime.StoreOperation = RTC_STOREOPERATION_RESET;

  // 시, 분, 초 모두 일치해야 알람 발생
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

### 3. 스탠바이 모드 진입

```c
void Enter_Standby_With_RTC_Alarm(uint32_t wakeup_seconds)
{
  printf("Entering STANDBY mode for %lu seconds...\n", wakeup_seconds);
  HAL_Delay(100);  // 메시지 출력 대기

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 1. RTC 알람 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  RTC_Alarm_Set(wakeup_seconds);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 2. 웨이크업 플래그 클리어
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  __HAL_PWR_CLEAR_FLAG(PWR_FLAG_SB);
  __HAL_RTC_ALARM_CLEAR_FLAG(&hrtc, RTC_FLAG_ALRAF);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 3. LED OFF
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  HAL_GPIO_WritePin(GPIOI, GPIO_PIN_12 | GPIO_PIN_13, GPIO_PIN_RESET);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 4. 스탠바이 모드 진입
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  HAL_PWR_EnterSTANDBYMode();

  // RTC 알람 발생 시 시스템 리셋으로 깨어남
}
```

### 4. 메인 함수

```c
int main(void)
{
  uint32_t wakeup_count = 0;

  // HAL 초기화
  HAL_Init();

  // 시스템 클럭 설정
  SystemClock_Config();

  // GPIO 초기화
  GPIO_Init();

  // RTC 초기화
  RTC_Init();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 웨이크업 원인 확인
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  if (__HAL_PWR_GET_FLAG(PWR_FLAG_SB) != RESET)
  {
    // 스탠바이 모드에서 깨어남
    printf("Woke up from STANDBY mode\n");

    // 플래그 클리어
    __HAL_PWR_CLEAR_FLAG(PWR_FLAG_SB);

    // 알람 플래그 확인
    if (__HAL_RTC_ALARM_GET_FLAG(&hrtc, RTC_FLAG_ALRAF))
    {
      printf("Wakeup source: RTC Alarm A\n");
      __HAL_RTC_ALARM_CLEAR_FLAG(&hrtc, RTC_FLAG_ALRAF);
      __HAL_RTC_ALARM_EXTI_CLEAR_FLAG();

      // 웨이크업 카운트 증가
      wakeup_count = HAL_RTCEx_BKUPRead(&hrtc, RTC_BKP_DR1);
      wakeup_count++;
      HAL_RTCEx_BKUPWrite(&hrtc, RTC_BKP_DR1, wakeup_count);

      printf("Wakeup count: %lu\n", wakeup_count);

      // LED2 깜박임 (웨이크업 표시)
      for (int i = 0; i < 3; i++)
      {
        HAL_GPIO_TogglePin(GPIOI, GPIO_PIN_13);
        HAL_Delay(200);
      }
    }
  }
  else
  {
    // 정상 부팅
    printf("Normal boot (Power-On Reset)\n");
    wakeup_count = 0;
    HAL_RTCEx_BKUPWrite(&hrtc, RTC_BKP_DR1, 0);

    // LED1 깜박임
    for (int i = 0; i < 3; i++)
    {
      HAL_GPIO_TogglePin(GPIOI, GPIO_PIN_12);
      HAL_Delay(200);
    }
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 작업 수행 (5초간)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  printf("Working for 5 seconds...\n");

  for (int i = 0; i < 10; i++)
  {
    HAL_GPIO_TogglePin(GPIOI, GPIO_PIN_12);
    HAL_Delay(500);

    // 센서 읽기, 데이터 전송 등의 작업 수행
    // ...
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 다음 웨이크업을 위한 스탠바이 진입
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // 10초 후 깨어나도록 알람 설정
  Enter_Standby_With_RTC_Alarm(10);

  // 여기는 실행되지 않음
  while (1)
  {
  }
}
```

## RTC 웨이크업 타이머 사용 (대안)

### 웨이크업 타이머 초기화

```c
// RTC 알람 대신 웨이크업 타이머 사용
// 더 간단한 주기적 웨이크업에 적합

void RTC_WakeupTimer_Init(uint32_t seconds)
{
  // 웨이크업 타이머 설정
  // WUTR = seconds - 1
  // 클럭: ck_spre (1Hz)

  if (HAL_RTCEx_SetWakeUpTimer_IT(&hrtc,
                                   seconds - 1,
                                   RTC_WAKEUPCLOCK_CK_SPRE_16BITS) != HAL_OK)
  {
    Error_Handler();
  }

  // EXTI 라인 19 설정 (RTC 웨이크업)
  __HAL_RTC_WAKEUPTIMER_EXTI_ENABLE_IT();
  __HAL_RTC_WAKEUPTIMER_EXTI_ENABLE_RISING_EDGE();

  printf("RTC Wakeup Timer set for %lu seconds\n", seconds);
}

void Enter_Standby_With_WakeupTimer(uint32_t seconds)
{
  // 웨이크업 타이머 설정
  RTC_WakeupTimer_Init(seconds);

  // 플래그 클리어
  __HAL_PWR_CLEAR_FLAG(PWR_FLAG_SB);
  __HAL_RTC_WAKEUPTIMER_CLEAR_FLAG(&hrtc, RTC_FLAG_WUTF);

  // 스탠바이 진입
  HAL_PWR_EnterSTANDBYMode();
}

// 웨이크업 확인
if (__HAL_RTC_WAKEUPTIMER_GET_FLAG(&hrtc, RTC_FLAG_WUTF))
{
  printf("Woken up by RTC Wakeup Timer\n");
  __HAL_RTC_WAKEUPTIMER_CLEAR_FLAG(&hrtc, RTC_FLAG_WUTF);
  __HAL_RTC_WAKEUPTIMER_EXTI_CLEAR_FLAG();
}
```

## 고급 기능

### 1. 정확한 시간 관리

```c
void Display_Current_Time(void)
{
  RTC_TimeTypeDef sTime;
  RTC_DateTypeDef sDate;

  HAL_RTC_GetTime(&hrtc, &sTime, RTC_FORMAT_BIN);
  HAL_RTC_GetDate(&hrtc, &sDate, RTC_FORMAT_BIN);  // 시간 읽기 후 날짜 읽기 필수

  printf("Current: %04d-%02d-%02d %02d:%02d:%02d\n",
         2000 + sDate.Year, sDate.Month, sDate.Date,
         sTime.Hours, sTime.Minutes, sTime.Seconds);
}
```

### 2. 조건부 웨이크업

```c
void Conditional_Wakeup(void)
{
  RTC_TimeTypeDef sTime;
  HAL_RTC_GetTime(&hrtc, &sTime, RTC_FORMAT_BIN);

  // 특정 시간에만 작업 수행
  if (sTime.Hours >= 8 && sTime.Hours < 18)
  {
    // 근무 시간: 5초 작업
    printf("Work hours: performing tasks\n");
    Perform_Tasks();
    Enter_Standby_With_RTC_Alarm(60);  // 1분 후 웨이크업
  }
  else
  {
    // 비근무 시간: 즉시 다시 스탠바이
    printf("Non-work hours: going back to standby\n");
    Enter_Standby_With_RTC_Alarm(3600);  // 1시간 후 웨이크업
  }
}
```

### 3. 다중 알람 사용

```c
void Multi_Alarm_Setup(void)
{
  RTC_AlarmTypeDef sAlarm = {0};

  // 알람 A: 매 시간 정각
  sAlarm.AlarmTime.Hours = 0;
  sAlarm.AlarmTime.Minutes = 0;
  sAlarm.AlarmTime.Seconds = 0;
  sAlarm.AlarmMask = RTC_ALARMMASK_DATEWEEKDAY | RTC_ALARMMASK_HOURS;
  sAlarm.Alarm = RTC_ALARM_A;
  HAL_RTC_SetAlarm_IT(&hrtc, &sAlarm, RTC_FORMAT_BIN);

  // 알람 B: 매일 특정 시간 (예: 오전 9시)
  sAlarm.AlarmTime.Hours = 9;
  sAlarm.AlarmTime.Minutes = 0;
  sAlarm.AlarmTime.Seconds = 0;
  sAlarm.AlarmMask = RTC_ALARMMASK_DATEWEEKDAY;
  sAlarm.Alarm = RTC_ALARM_B;
  HAL_RTC_SetAlarm_IT(&hrtc, &sAlarm, RTC_FORMAT_BIN);
}
```

### 4. 웨이크업 로그 기록

```c
typedef struct {
  uint32_t timestamp;
  uint8_t wakeup_source;
  uint16_t battery_level;
} WakeupLog_t;

#define LOG_SIZE  100
WakeupLog_t wakeup_log[LOG_SIZE] __attribute__((section(".backup_sram")));
uint32_t log_index = 0;

void Log_Wakeup_Event(uint8_t source)
{
  RTC_TimeTypeDef sTime;
  RTC_DateTypeDef sDate;

  HAL_RTC_GetTime(&hrtc, &sTime, RTC_FORMAT_BIN);
  HAL_RTC_GetDate(&hrtc, &sDate, RTC_FORMAT_BIN);

  // 타임스탬프 계산 (초 단위)
  uint32_t timestamp = sDate.Date * 86400 +
                       sTime.Hours * 3600 +
                       sTime.Minutes * 60 +
                       sTime.Seconds;

  // 로그 저장
  wakeup_log[log_index].timestamp = timestamp;
  wakeup_log[log_index].wakeup_source = source;
  wakeup_log[log_index].battery_level = Get_Battery_Level();

  log_index = (log_index + 1) % LOG_SIZE;

  // 백업 레지스터에 인덱스 저장
  HAL_RTCEx_BKUPWrite(&hrtc, RTC_BKP_DR2, log_index);
}
```

## LSE vs LSI 비교

### 클럭 소스 선택

```c
클럭 소스  │ 주파수      │ 정확도       │ 전력 소비  │ 외부 부품
─────────┼────────────┼─────────────┼───────────┼──────────
LSI      │ ~32 kHz    │ ±5%          │ 낮음      │ 불필요
LSE      │ 32.768 kHz │ ±5 ppm       │ 매우 낮음  │ 크리스탈 필요
HSE/128  │ ~125 kHz   │ 높음         │ 높음      │ 크리스탈 필요
```

### LSE 사용 예제

```c
void RTC_Init_With_LSE(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_PeriphCLKInitTypeDef PeriphClkInitStruct = {0};

  // LSE 활성화
  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_LSE;
  RCC_OscInitStruct.LSEState = RCC_LSE_ON;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_NONE;

  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  // RTC 클럭 소스: LSE
  PeriphClkInitStruct.PeriphClockSelection = RCC_PERIPHCLK_RTC;
  PeriphClkInitStruct.RTCClockSelection = RCC_RTCCLKSOURCE_LSE;

  if (HAL_RCCEx_PeriphCLKConfig(&PeriphClkInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  // RTC 프리스케일러 (LSE 32.768kHz 기준)
  hrtc.Init.AsynchPrediv = 127;    // (127+1) = 128
  hrtc.Init.SynchPrediv = 255;     // (255+1) = 256
  // RTC_CLK = 32768 / (128 * 256) = 1Hz
}
```

## 빌드 및 실행

### 테스트 시나리오

```
1. 보드 플래싱 및 리셋
2. 시리얼 터미널 확인: "Normal boot"
3. LED1 5초간 점멸 (작업 수행)
4. 메시지: "Entering STANDBY mode for 10 seconds..."
5. LED OFF, 10초 대기
6. 자동 웨이크업, 메시지: "Woke up from STANDBY mode"
7. LED2 깜박임 (웨이크업 표시)
8. 2-7 반복
```

### 전력 소비 측정

```
작업 단계 (5초):    ~100 mA
스탠바이 (10초):    ~2.4 μA

평균 전력 소비:
= (100mA × 5s + 0.0024mA × 10s) / 15s
= 500mA·s + 0.024mA·s) / 15s
≈ 33.3 mA (평균)

배터리 수명 (1000mAh):
≈ 1000mAh / 33.3mA ≈ 30시간

순수 스탠바이 시 배터리 수명:
≈ 1000mAh / 0.0024mA ≈ 47년!
```

## 트러블슈팅

### RTC가 초기화되지 않는 경우

```c
// 백업 도메인 리셋 후 재시도
__HAL_RCC_BACKUPRESET_FORCE();
HAL_Delay(10);
__HAL_RCC_BACKUPRESET_RELEASE();
HAL_Delay(10);

RTC_Init();
```

### 알람이 발생하지 않는 경우

```c
// 알람 인터럽트 확인
if (!(RTC->CR & RTC_CR_ALRAIE))
{
  printf("Alarm interrupt not enabled!\n");
  __HAL_RTC_ALARM_ENABLE_IT(&hrtc, RTC_IT_ALRA);
}

// EXTI 라인 확인
if (!(EXTI->IMR1 & EXTI_IMR1_IM17))
{
  printf("EXTI line 17 not enabled!\n");
  __HAL_RTC_ALARM_EXTI_ENABLE_IT();
}
```

### 시간이 정확하지 않은 경우 (LSI 사용 시)

```c
// LSI 주파수 교정
uint32_t lsi_freq = Measure_LSI_Frequency();
printf("LSI frequency: %lu Hz\n", lsi_freq);

// 프리스케일러 재계산
uint32_t prediv_a = 127;
uint32_t prediv_s = (lsi_freq / (prediv_a + 1)) - 1;

hrtc.Init.AsynchPrediv = prediv_a;
hrtc.Init.SynchPrediv = prediv_s;
HAL_RTC_Init(&hrtc);
```

## 참고 자료

- **RM0399**: STM32H745 Reference Manual, Chapter 47 (RTC)
- **AN4759**: Using the hardware RTC in STM32 MCUs
- **AN4655**: Virtually increasing number of backup registers
- **예제 코드**: `STM32Cube_FW_H7/Projects/STM32H745I-DISCO/Examples/PWR/PWR_STANDBY_RTC`

## 관련 예제

- **PWR_STANDBY**: 기본 스탠바이 모드
- **RTC_Alarm**: RTC 알람 설정
- **RTC_Calendar**: RTC 캘린더 기능
