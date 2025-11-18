# IWDG_WindowMode 예제

## 개요

STM32H745I-DISCO의 IWDG(Independent Watchdog)를 윈도우 모드로 사용하여 시스템 안정성을 확보하는 예제입니다.

## 하드웨어

- **보드**: STM32H745I-DISCO
- **LED**: LED1(정상 동작), LED2(리프레시), LED3(리셋 발생)

## 주요 코드

```c
/* IWDG 초기화 */
IWDG_HandleTypeDef hiwdg;

void IWDG_Init(void)
{
    hiwdg.Instance = IWDG1;
    hiwdg.Init.Prescaler = IWDG_PRESCALER_32;
    hiwdg.Init.Window = 0x0FFF;     /* Window: ~1.6초 */
    hiwdg.Init.Reload = 0x0FFF;     /* Timeout: ~2초 */

    if (HAL_IWDG_Init(&hiwdg) != HAL_OK)
    {
        Error_Handler();
    }

    printf("IWDG initialized: Window mode\n");
    printf("  Refresh window: 1.6s ~ 2.0s\n");
}

/* Watchdog 리프레시 */
void IWDG_Refresh(void)
{
    HAL_IWDG_Refresh(&hiwdg);
    BSP_LED_Toggle(LED2);
}

/* 리셋 원인 확인 */
void Check_Reset_Cause(void)
{
    if (__HAL_RCC_GET_FLAG(RCC_FLAG_IWDG1RST))
    {
        printf("System reset by IWDG!\n");
        BSP_LED_On(LED3);
        __HAL_RCC_CLEAR_RESET_FLAGS();
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

    Check_Reset_Cause();

    IWDG_Init();

    while (1)
    {
        /* 정상 작업 */
        BSP_LED_Toggle(LED1);

        /* 1.8초마다 리프레시 (윈도우 내) */
        HAL_Delay(1800);
        IWDG_Refresh();
    }
}
```

## 윈도우 모드 설명

```
타임라인:
0s ────────── 1.6s ────────── 2.0s
    (너무 빨리)  [정상 윈도우]  (타임아웃)
                   리프레시
                   가능 구간
```

- **1.6초 이전**: 리프레시 불가 (리셋 발생)
- **1.6~2.0초**: 리프레시 가능
- **2.0초 이후**: 리프레시 안하면 리셋

## 참고

- Reference Manual: IWDG 챕터
- LSI 클록 사용 (32kHz)
