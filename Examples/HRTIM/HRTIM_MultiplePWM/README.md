# HRTIM_MultiplePWM 예제

## 개요

STM32H745I-DISCO의 HRTIM(High-Resolution Timer)을 사용하여 고해상도 PWM 신호를 생성하는 예제입니다. 184ps 해상도로 정밀한 PWM 제어가 가능합니다.

## 하드웨어

- **보드**: STM32H745I-DISCO
- **출력 핀**: HRTIM_CHA1, HRTIM_CHA2 (고해상도 PWM)
- **해상도**: 184ps (480MHz 기준)

## 주요 코드

```c
/* HRTIM 초기화 */
HRTIM_HandleTypeDef hhrtim1;

void HRTIM_Init(void)
{
    __HAL_RCC_HRTIM1_CLK_ENABLE();

    hhrtim1.Instance = HRTIM1;
    hhrtim1.Init.HRTIMInterruptResquests = HRTIM_IT_NONE;
    hhrtim1.Init.SyncOptions = HRTIM_SYNCOPTION_NONE;

    if (HAL_HRTIM_Init(&hhrtim1) != HAL_OK)
    {
        Error_Handler();
    }

    /* Timer A 설정 */
    HRTIM_TimeBaseCfgTypeDef pTimeBaseCfg = {0};
    pTimeBaseCfg.Period = 0xFFDF;  /* ~217ns @ 480MHz */
    pTimeBaseCfg.RepetitionCounter = 0;
    pTimeBaseCfg.PrescalerRatio = HRTIM_PRESCALERRATIO_DIV1;
    pTimeBaseCfg.Mode = HRTIM_MODE_CONTINUOUS;

    HAL_HRTIM_TimeBaseConfig(&hhrtim1, HRTIM_TIMERINDEX_TIMER_A, &pTimeBaseCfg);

    /* Output 1 설정 */
    HRTIM_OutputCfgTypeDef pOutputCfg = {0};
    pOutputCfg.Polarity = HRTIM_OUTPUTPOLARITY_HIGH;
    pOutputCfg.SetSource = HRTIM_OUTPUTSET_TIMPER;
    pOutputCfg.ResetSource = HRTIM_OUTPUTRESET_TIMCMP1;
    pOutputCfg.IdleMode = HRTIM_OUTPUTIDLEMODE_NONE;
    pOutputCfg.IdleLevel = HRTIM_OUTPUTIDLELEVEL_INACTIVE;
    pOutputCfg.FaultLevel = HRTIM_OUTPUTFAULTLEVEL_INACTIVE;
    pOutputCfg.ChopperModeEnable = HRTIM_OUTPUTCHOPPERMODE_DISABLED;
    pOutputCfg.BurstModeEntryDelayed = HRTIM_OUTPUTBURSTMODEENTRY_REGULAR;

    HAL_HRTIM_WaveformOutputConfig(&hhrtim1, HRTIM_TIMERINDEX_TIMER_A,
                                   HRTIM_OUTPUT_TA1, &pOutputCfg);

    /* Compare 1 설정 (Duty Cycle) */
    HRTIM_CompareCfgTypeDef pCompareCfg = {0};
    pCompareCfg.CompareValue = 0x7FEF;  /* 50% duty */
    pCompareCfg.AutoDelayedMode = HRTIM_AUTODELAYEDMODE_REGULAR;
    pCompareCfg.AutoDelayedTimeout = 0;

    HAL_HRTIM_WaveformCompareConfig(&hhrtim1, HRTIM_TIMERINDEX_TIMER_A,
                                    HRTIM_COMPAREUNIT_1, &pCompareCfg);

    /* 출력 시작 */
    HAL_HRTIM_WaveformOutputStart(&hhrtim1, HRTIM_OUTPUT_TA1);
    HAL_HRTIM_WaveformCountStart(&hhrtim1, HRTIM_TIMERID_TIMER_A);

    printf("HRTIM PWM started: 184ps resolution\n");
}

/* PWM Duty Cycle 변경 */
void HRTIM_SetDutyCycle(uint8_t duty_percent)
{
    uint32_t compare_value = (0xFFDF * duty_percent) / 100;

    __HAL_HRTIM_SETCOMPARE(&hhrtim1, HRTIM_TIMERINDEX_TIMER_A,
                           HRTIM_COMPAREUNIT_1, compare_value);
}

/* 데드타임 설정 */
void HRTIM_DeadTime_Config(uint16_t rising_ns, uint16_t falling_ns)
{
    HRTIM_DeadTimeCfgTypeDef pDeadTimeCfg = {0};

    /* 184ps * 5.4 ≈ 1ns */
    uint16_t rising_value = (rising_ns * 54) / 10;
    uint16_t falling_value = (falling_ns * 54) / 10;

    pDeadTimeCfg.Prescaler = HRTIM_TIMDEADTIME_PRESCALERRATIO_DIV1;
    pDeadTimeCfg.RisingValue = rising_value;
    pDeadTimeCfg.RisingSign = HRTIM_TIMDEADTIME_RISINGSIGN_POSITIVE;
    pDeadTimeCfg.RisingLock = HRTIM_TIMDEADTIME_RISINGLOCK_WRITE;
    pDeadTimeCfg.RisingSignLock = HRTIM_TIMDEADTIME_RISINGSIGNLOCK_WRITE;

    pDeadTimeCfg.FallingValue = falling_value;
    pDeadTimeCfg.FallingSign = HRTIM_TIMDEADTIME_FALLINGSIGN_POSITIVE;
    pDeadTimeCfg.FallingLock = HRTIM_TIMDEADTIME_FALLINGLOCK_WRITE;
    pDeadTimeCfg.FallingSignLock = HRTIM_TIMDEADTIME_FALLINGSIGNLOCK_WRITE;

    HAL_HRTIM_DeadTimeConfig(&hhrtim1, HRTIM_TIMERINDEX_TIMER_A, &pDeadTimeCfg);

    printf("Dead time: Rising=%uns, Falling=%uns\n", rising_ns, falling_ns);
}

/* 메인 */
int main(void)
{
    HAL_Init();
    SystemClock_Config();

    HRTIM_Init();

    /* 데드타임 100ns 설정 */
    HRTIM_DeadTime_Config(100, 100);

    while (1)
    {
        /* Duty cycle sweep: 10% → 90% */
        for (uint8_t duty = 10; duty <= 90; duty += 10)
        {
            HRTIM_SetDutyCycle(duty);
            printf("Duty: %u%%\n", duty);
            HAL_Delay(1000);
        }
    }
}
```

## 응용: 3상 모터 제어

```c
/* 3상 PWM 생성 */
void HRTIM_3Phase_Init(void)
{
    /* Timer A: Phase U */
    /* Timer B: Phase V */
    /* Timer C: Phase W */

    /* 120도 위상차 설정 */
    uint32_t period = 0xFFDF;
    uint32_t phase_shift = period / 3;

    HAL_HRTIM_WaveformSetPhase(&hhrtim1, HRTIM_TIMERINDEX_TIMER_A, 0);
    HAL_HRTIM_WaveformSetPhase(&hhrtim1, HRTIM_TIMERINDEX_TIMER_B, phase_shift);
    HAL_HRTIM_WaveformSetPhase(&hhrtim1, HRTIM_TIMERINDEX_TIMER_C, phase_shift * 2);
}
```

## 참고

- Reference Manual: HRTIM 챕터
- AN4539: HRTIM 사용하기
