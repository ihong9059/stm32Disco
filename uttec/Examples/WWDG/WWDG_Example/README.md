# WWDG_Example 예제

## 개요

STM32H745I-DISCO의 WWDG(Window Watchdog)를 사용하여 조기 경고 인터럽트와 함께 시스템을 모니터링하는 예제입니다.

## 하드웨어

- **보드**: STM32H745I-DISCO
- **LED**: LED1(정상), LED2(경고), LED3(리셋)

## 주요 코드

```c
/* WWDG 초기화 */
WWDG_HandleTypeDef hwwdg;

void WWDG_Init(void)
{
    __HAL_RCC_WWDG1_CLK_ENABLE();

    hwwdg.Instance = WWDG1;
    hwwdg.Init.Prescaler = WWDG_PRESCALER_8;
    hwwdg.Init.Window = 0x50;        /* Window value */
    hwwdg.Init.Counter = 0x7F;       /* Counter initial value */
    hwwdg.Init.EWIMode = WWDG_EWI_ENABLE;  /* Early Wakeup Interrupt */

    if (HAL_WWDG_Init(&hwwdg) != HAL_OK)
    {
        Error_Handler();
    }

    /* WWDG 인터럽트 활성화 */
    HAL_NVIC_SetPriority(WWDG1_IRQn, 3, 0);
    HAL_NVIC_EnableIRQ(WWDG1_IRQn);

    printf("WWDG initialized\n");
}

/* 조기 경고 인터럽트 콜백 */
void HAL_WWDG_EarlyWakeupCallback(WWDG_HandleTypeDef *hwwdg)
{
    printf("WWDG Early Wakeup!\n");
    BSP_LED_Toggle(LED2);

    /* 카운터 리프레시 */
    HAL_WWDG_Refresh(hwwdg);
}

/* 리셋 원인 확인 */
void Check_Reset_Cause(void)
{
    if (__HAL_RCC_GET_FLAG(RCC_FLAG_WWDG1RST))
    {
        printf("System reset by WWDG!\n");
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

    WWDG_Init();

    while (1)
    {
        BSP_LED_Toggle(LED1);
        HAL_Delay(10);

        /* WWDG는 EWI 콜백에서 자동 리프레시됨 */
    }
}

/* WWDG 인터럽트 핸들러 */
void WWDG1_IRQHandler(void)
{
    HAL_WWDG_IRQHandler(&hwwdg);
}
```

## WWDG vs IWDG

| 특징 | WWDG | IWDG |
|------|------|------|
| 클록 | APB1 | LSI (32kHz) |
| 윈도우 | 가변 윈도우 | 고정 윈도우 |
| 조기 인터럽트 | 지원 | 미지원 |
| 타임아웃 | 짧음 (ms) | 김 (초) |

## 참고

- Reference Manual: WWDG 챕터
- 조기 경고 인터럽트 활용
