# RTC_Alarm 예제

## 개요

STM32H745I-DISCO의 RTC(Real-Time Clock) 알람 기능을 사용하여 정확한 시간 관리와 이벤트 스케줄링을 구현하는 예제입니다.

## 하드웨어

- **보드**: STM32H745I-DISCO
- **LED**: LED1(알람 발생), LED2(RTC 동작), LED3(오류)
- **버튼**: USER 버튼 (알람 설정)

## 주요 코드

```c
/* RTC 초기화 */
RTC_HandleTypeDef hrtc;

void RTC_Init(void)
{
    hrtc.Instance = RTC;
    hrtc.Init.HourFormat = RTC_HOURFORMAT_24;
    hrtc.Init.AsynchPrediv = 127;
    hrtc.Init.SynchPrediv = 255;
    hrtc.Init.OutPut = RTC_OUTPUT_DISABLE;

    if (HAL_RTC_Init(&hrtc) != HAL_OK)
    {
        Error_Handler();
    }

    /* 시간 설정: 12:30:00 */
    RTC_TimeTypeDef sTime = {0};
    sTime.Hours = 12;
    sTime.Minutes = 30;
    sTime.Seconds = 0;
    HAL_RTC_SetTime(&hrtc, &sTime, RTC_FORMAT_BIN);

    /* 날짜 설정: 2025-11-18 */
    RTC_DateTypeDef sDate = {0};
    sDate.Year = 25;
    sDate.Month = RTC_MONTH_NOVEMBER;
    sDate.Date = 18;
    sDate.WeekDay = RTC_WEEKDAY_TUESDAY;
    HAL_RTC_SetDate(&hrtc, &sDate, RTC_FORMAT_BIN);
}

/* 알람 설정 */
void RTC_SetAlarm(uint8_t hours, uint8_t minutes, uint8_t seconds)
{
    RTC_AlarmTypeDef sAlarm = {0};

    sAlarm.AlarmTime.Hours = hours;
    sAlarm.AlarmTime.Minutes = minutes;
    sAlarm.AlarmTime.Seconds = seconds;
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

    printf("Alarm set: %02u:%02u:%02u\n", hours, minutes, seconds);
}

/* 알람 인터럽트 콜백 */
void HAL_RTC_AlarmAEventCallback(RTC_HandleTypeDef *hrtc)
{
    printf("Alarm triggered!\n");
    BSP_LED_On(LED1);

    /* 다음 알람 설정 (1분 후) */
    RTC_TimeTypeDef sTime;
    HAL_RTC_GetTime(hrtc, &sTime, RTC_FORMAT_BIN);

    uint8_t next_minutes = (sTime.Minutes + 1) % 60;
    RTC_SetAlarm(sTime.Hours, next_minutes, 0);
}

/* 메인 */
int main(void)
{
    HAL_Init();
    SystemClock_Config();

    BSP_LED_Init(LED1);
    BSP_LED_Init(LED2);
    BSP_PB_Init(BUTTON_USER, BUTTON_MODE_EXTI);

    RTC_Init();

    /* 첫 알람: 12:31:00 */
    RTC_SetAlarm(12, 31, 0);

    while (1)
    {
        /* 현재 시간 출력 */
        RTC_TimeTypeDef sTime;
        RTC_DateTypeDef sDate;

        HAL_RTC_GetTime(&hrtc, &sTime, RTC_FORMAT_BIN);
        HAL_RTC_GetDate(&hrtc, &sDate, RTC_FORMAT_BIN);

        printf("%02d:%02d:%02d  %04d-%02d-%02d\n",
               sTime.Hours, sTime.Minutes, sTime.Seconds,
               2000 + sDate.Year, sDate.Month, sDate.Date);

        BSP_LED_Toggle(LED2);
        HAL_Delay(1000);
    }
}
```

## 응용: 주기적 웨이크업

```c
/* 웨이크업 타이머 설정 (매 10초) */
void RTC_WakeUp_Init(void)
{
    /* 웨이크업 타이머 비활성화 */
    HAL_RTCEx_DeactivateWakeUpTimer(&hrtc);

    /* 웨이크업 타이머 설정: 10초 주기 */
    HAL_RTCEx_SetWakeUpTimer_IT(&hrtc, 10 - 1, RTC_WAKEUPCLOCK_CK_SPRE_16BITS);
}

void HAL_RTCEx_WakeUpTimerEventCallback(RTC_HandleTypeDef *hrtc)
{
    printf("Wake up!\n");
    BSP_LED_Toggle(LED1);
}
```

## 참고

- Reference Manual: RTC 챕터
- 백업 도메인 전원 필요
- LSE 크리스탈 사용 권장
