# LPTIM_PulseCounter 예제

## 개요

STM32H745I-DISCO의 LPTIM(Low-Power Timer)을 사용하여 저전력 모드에서 외부 펄스를 카운트하는 예제입니다.

## 하드웨어

- **보드**: STM32H745I-DISCO
- **입력**: LPTIM1_IN1 (외부 펄스)
- **LED**: LED1(카운트 업데이트), LED2(저전력 모드)

## 주요 코드

```c
/* LPTIM 초기화 */
LPTIM_HandleTypeDef hlptim1;

void LPTIM_Init(void)
{
    __HAL_RCC_LPTIM1_CLK_ENABLE();

    hlptim1.Instance = LPTIM1;
    hlptim1.Init.Clock.Source = LPTIM_CLOCKSOURCE_APBCLOCK_LPOSC;
    hlptim1.Init.Clock.Prescaler = LPTIM_PRESCALER_DIV1;
    hlptim1.Init.Trigger.Source = LPTIM_TRIGSOURCE_SOFTWARE;
    hlptim1.Init.OutputPolarity = LPTIM_OUTPUTPOLARITY_HIGH;
    hlptim1.Init.UpdateMode = LPTIM_UPDATE_IMMEDIATE;
    hlptim1.Init.CounterSource = LPTIM_COUNTERSOURCE_EXTERNAL;
    hlptim1.Init.Input1Source = LPTIM_INPUT1SOURCE_GPIO;
    hlptim1.Init.Input2Source = LPTIM_INPUT2SOURCE_GPIO;

    if (HAL_LPTIM_Init(&hlptim1) != HAL_OK)
    {
        Error_Handler();
    }

    /* 카운터 시작 */
    HAL_LPTIM_Counter_Start_IT(&hlptim1, 0xFFFF);

    printf("LPTIM initialized: External pulse counter\n");
}

/* 펄스 카운트 읽기 */
uint32_t LPTIM_GetCount(void)
{
    return HAL_LPTIM_ReadCounter(&hlptim1);
}

/* 인터럽트 콜백 */
void HAL_LPTIM_AutoReloadMatchCallback(LPTIM_HandleTypeDef *hlptim)
{
    printf("Counter overflow!\n");
    BSP_LED_Toggle(LED1);
}

/* 메인 */
int main(void)
{
    HAL_Init();
    SystemClock_Config();

    BSP_LED_Init(LED1);
    BSP_LED_Init(LED2);

    LPTIM_Init();

    while (1)
    {
        uint32_t count = LPTIM_GetCount();
        printf("Pulse count: %lu\n", count);

        /* 저전력 모드 진입 */
        BSP_LED_On(LED2);
        HAL_PWR_EnterSLEEPMode(PWR_MAINREGULATOR_ON, PWR_SLEEPENTRY_WFI);
        BSP_LED_Off(LED2);

        HAL_Delay(1000);
    }
}
```

## 참고

- Reference Manual: LPTIM 챕터
- 저전력 클록 소스 필요
