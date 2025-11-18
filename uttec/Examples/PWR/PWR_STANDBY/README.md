# PWR_STANDBY - 스탠바이 모드 진입 및 복귀

## 개요

이 예제는 STM32H745의 스탠바이(Standby) 모드 진입과 웨이크업 메커니즘을 보여줍니다. Wakeup 버튼을 사용하여 스탠바이 모드에서 깨어나고, LED로 현재 상태를 표시합니다. 스탠바이 모드는 STM32H7의 가장 낮은 전력 소비 모드 중 하나입니다.

## 하드웨어 요구사항

- **STM32H745I-DISCO 보드**
- **Wakeup 버튼**: PC13 (내장)
- **LED1**: PI12 (주황색)
- **LED2**: PI13 (초록색)
- **전류계**: 전력 소비 측정용 (선택 사항)

## 주요 기능

### 스탠바이 모드 특성
- **전력 소비**: 약 2.4 μA (전형값)
- **웨이크업 소스**: WKUP 핀, RTC 알람, IWDG 리셋
- **RAM 보존**: 모든 SRAM 내용 손실 (백업 SRAM 제외)
- **복귀 방식**: 시스템 리셋 (프로그램 처음부터 실행)

### 전력 모드 비교
```
모드          │ 전류 소비      │ 웨이크업 시간  │ RAM 보존
─────────────┼───────────────┼───────────────┼─────────
Run          │ ~100 mA       │ -              │ Yes
Sleep        │ ~40 mA        │ ~μs            │ Yes
Stop         │ ~100 μA       │ ~μs            │ Yes
Standby      │ ~2.4 μA       │ ~ms            │ No
Shutdown     │ ~130 nA       │ ~ms            │ No
```

## 동작 원리

### 상태 머신
```
        Power-On Reset
              │
              ▼
        ┌──────────┐
        │   RUN    │◄─────────┐
        │  Mode    │          │
        └────┬─────┘          │
             │                │
        Press Button          │
             │            WKUP Pin
             ▼                │
        ┌──────────┐          │
        │ STANDBY  │──────────┘
        │  Mode    │
        └──────────┘
         (~2.4 μA)
```

### 웨이크업 시퀀스
```
1. 스탠바이 진입
   - 모든 클럭 정지
   - 레귤레이터 OFF (LDO)
   - SRAM 내용 손실

2. 웨이크업 이벤트 (WKUP 핀 활성화)
   - 전원 리셋 시퀀스 시작
   - 시스템 초기화
   - main() 처음부터 실행

3. 웨이크업 플래그 확인
   - PWR_FLAG_SB 체크
   - 스탠바이에서 깨어났는지 확인
```

## 코드 구조

### 1. 시스템 초기화 및 웨이크업 확인

```c
// 전역 변수
uint32_t standby_count = 0;  // 백업 레지스터에 저장

int main(void)
{
  // HAL 초기화
  HAL_Init();

  // 시스템 클럭 설정 (400MHz CM7)
  SystemClock_Config();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 웨이크업 플래그 확인
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  if (__HAL_PWR_GET_FLAG(PWR_FLAG_SB) != RESET)
  {
    // 스탠바이 모드에서 깨어남
    printf("Woke up from STANDBY mode\n");

    // 스탠바이 플래그 클리어
    __HAL_PWR_CLEAR_FLAG(PWR_FLAG_SB);

    // 백업 레지스터에서 카운트 읽기
    standby_count = HAL_RTCEx_BKUPRead(&hrtc, RTC_BKP_DR0);
    standby_count++;

    printf("Standby count: %lu\n", standby_count);

    // LED2 깜박임 (웨이크업 표시)
    for (int i = 0; i < 5; i++)
    {
      HAL_GPIO_TogglePin(GPIOI, GPIO_PIN_13);
      HAL_Delay(200);
    }
  }
  else
  {
    // 정상 부팅 (파워온 리셋 또는 외부 리셋)
    printf("Normal boot\n");
    standby_count = 0;

    // LED1 깜박임 (부팅 표시)
    for (int i = 0; i < 3; i++)
    {
      HAL_GPIO_TogglePin(GPIOI, GPIO_PIN_12);
      HAL_Delay(200);
    }
  }

  // 백업 레지스터에 카운트 저장
  HAL_RTCEx_BKUPWrite(&hrtc, RTC_BKP_DR0, standby_count);

  // GPIO 및 주변장치 초기화
  GPIO_Init();

  // 메인 루프
  while (1)
  {
    printf("Press USER button to enter STANDBY mode\n");

    // LED1 점멸 (실행 중 표시)
    HAL_GPIO_TogglePin(GPIOI, GPIO_PIN_12);
    HAL_Delay(500);

    // 버튼 입력 대기
    if (HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_RESET)
    {
      // 버튼 누름 감지
      HAL_Delay(50);  // 디바운싱

      if (HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_RESET)
      {
        // 스탠바이 모드 진입
        Enter_Standby_Mode();
      }
    }
  }
}
```

### 2. 스탠바이 모드 진입

```c
void Enter_Standby_Mode(void)
{
  printf("Entering STANDBY mode...\n");
  HAL_Delay(100);  // 메시지 출력 대기

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 1. 웨이크업 핀 설정
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // PC13을 WKUP2로 활성화 (상승 엣지)
  HAL_PWR_EnableWakeUpPin(PWR_WAKEUP_PIN2_HIGH);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 2. 모든 웨이크업 플래그 클리어
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  __HAL_PWR_CLEAR_FLAG(PWR_FLAG_SB);
  __HAL_PWR_CLEAR_FLAG(PWR_WAKEUP_FLAG2);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 3. LED OFF (전력 절약)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  HAL_GPIO_WritePin(GPIOI, GPIO_PIN_12, GPIO_PIN_RESET);
  HAL_GPIO_WritePin(GPIOI, GPIO_PIN_13, GPIO_PIN_RESET);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 4. 스탠바이 모드 진입
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  HAL_PWR_EnterSTANDBYMode();

  // 이 코드는 실행되지 않음 (시스템이 스탠바이 모드로 진입)
  // 웨이크업 시 시스템 리셋 발생
}
```

### 3. GPIO 초기화

```c
void GPIO_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  // GPIO 클럭 활성화
  __HAL_RCC_GPIOI_CLK_ENABLE();
  __HAL_RCC_GPIOC_CLK_ENABLE();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // LED 설정 (PI12, PI13)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  GPIO_InitStruct.Pin = GPIO_PIN_12 | GPIO_PIN_13;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  HAL_GPIO_Init(GPIOI, &GPIO_InitStruct);

  // 초기 상태: OFF
  HAL_GPIO_WritePin(GPIOI, GPIO_PIN_12 | GPIO_PIN_13, GPIO_PIN_RESET);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // Wakeup 버튼 설정 (PC13)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  GPIO_InitStruct.Pin = GPIO_PIN_13;
  GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
  GPIO_InitStruct.Pull = GPIO_NOPULL;  // 외부 풀업 존재
  HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);
}
```

### 4. 백업 도메인 초기화 (RTC)

```c
RTC_HandleTypeDef hrtc;

void RTC_Init(void)
{
  // PWR 클럭 활성화
  __HAL_RCC_PWR_CLK_ENABLE();

  // 백업 도메인 접근 활성화
  HAL_PWR_EnableBkUpAccess();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // RTC 클럭 소스 설정 (LSI 또는 LSE)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_PeriphCLKInitTypeDef PeriphClkInitStruct = {0};

  // LSI 활성화 (내부 저속 오실레이터, 32kHz)
  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_LSI;
  RCC_OscInitStruct.LSIState = RCC_LSI_ON;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_NONE;

  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  // RTC 클럭 소스: LSI
  PeriphClkInitStruct.PeriphClockSelection = RCC_PERIPHCLK_RTC;
  PeriphClkInitStruct.RTCClockSelection = RCC_RTCCLKSOURCE_LSI;

  if (HAL_RCCEx_PeriphCLKConfig(&PeriphClkInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // RTC 초기화
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  hrtc.Instance = RTC;
  hrtc.Init.HourFormat = RTC_HOURFORMAT_24;
  hrtc.Init.AsynchPrediv = 127;    // LSI 32kHz 기준
  hrtc.Init.SynchPrediv = 255;
  hrtc.Init.OutPut = RTC_OUTPUT_DISABLE;
  hrtc.Init.OutPutPolarity = RTC_OUTPUT_POLARITY_HIGH;
  hrtc.Init.OutPutType = RTC_OUTPUT_TYPE_OPENDRAIN;

  if (HAL_RTC_Init(&hrtc) != HAL_OK)
  {
    Error_Handler();
  }
}
```

## 웨이크업 소스

### 1. WKUP 핀

STM32H745는 6개의 WKUP 핀을 지원합니다:

```c
// WKUP 핀 목록
PWR_WAKEUP_PIN1      // PA0
PWR_WAKEUP_PIN2      // PA2 (또는 PC13)
PWR_WAKEUP_PIN3      // PC1
PWR_WAKEUP_PIN4      // PC13
PWR_WAKEUP_PIN5      // PI8
PWR_WAKEUP_PIN6      // PI11

// 활성화 예제
HAL_PWR_EnableWakeUpPin(PWR_WAKEUP_PIN2_HIGH);   // 상승 엣지
HAL_PWR_EnableWakeUpPin(PWR_WAKEUP_PIN2_LOW);    // 하강 엣지
```

### 2. RTC 알람 (PWR_STANDBY_RTC 예제 참조)

```c
// 5초 후 웨이크업 알람 설정
RTC_AlarmTypeDef sAlarm = {0};

sAlarm.Alarm = RTC_ALARM_A;
sAlarm.AlarmTime.Hours = 0;
sAlarm.AlarmTime.Minutes = 0;
sAlarm.AlarmTime.Seconds = 5;

HAL_RTC_SetAlarm_IT(&hrtc, &sAlarm, RTC_FORMAT_BIN);
```

## 전력 소비 측정

### 측정 설정

```
1. STM32H745I-DISCO 보드의 IDD 점퍼 제거
2. 전류계를 IDD 점퍼 위치에 직렬 연결
3. 스탠바이 모드 진입 후 전류 측정

예상 전류:
- Run 모드: ~100 mA
- Standby 모드: ~2.4 μA (전형값)
- Shutdown 모드: ~130 nA
```

### 전력 절약 팁

```c
void Optimize_Power_Consumption(void)
{
  // 1. 미사용 GPIO를 아날로그 모드로 설정
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  GPIO_InitStruct.Mode = GPIO_MODE_ANALOG;
  GPIO_InitStruct.Pull = GPIO_NOPULL;

  __HAL_RCC_GPIOA_CLK_ENABLE();
  GPIO_InitStruct.Pin = GPIO_PIN_All;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  // 2. 미사용 주변장치 클럭 비활성화
  __HAL_RCC_USART1_CLK_DISABLE();
  __HAL_RCC_SPI1_CLK_DISABLE();
  // ...

  // 3. D1, D2, D3 도메인 저전력 모드 설정
  HAL_PWREx_EnterSTANDBYMode(PWR_D1_DOMAIN);
  HAL_PWREx_EnterSTANDBYMode(PWR_D2_DOMAIN);
  HAL_PWREx_EnterSTANDBYMode(PWR_D3_DOMAIN);
}
```

## 백업 SRAM 사용

### 데이터 보존

```c
// 백업 SRAM (4KB, 0x38800000)
// 스탠바이 모드에서도 내용 유지 (VBAT 전원 공급)

#define BACKUP_SRAM_BASE  0x38800000

typedef struct {
  uint32_t magic;
  uint32_t boot_count;
  uint32_t total_standby_time;
  uint8_t  data[256];
} BackupData_t;

void Save_To_BackupSRAM(void)
{
  BackupData_t *backup = (BackupData_t*)BACKUP_SRAM_BASE;

  // 백업 SRAM 클럭 활성화
  __HAL_RCC_BKPRAM_CLK_ENABLE();

  // 백업 레귤레이터 활성화
  HAL_PWREx_EnableBkUpReg();

  // 데이터 저장
  backup->magic = 0xDEADBEEF;
  backup->boot_count++;
  backup->total_standby_time += standby_duration;

  printf("Backup SRAM: boot_count = %lu\n", backup->boot_count);
}

void Read_From_BackupSRAM(void)
{
  BackupData_t *backup = (BackupData_t*)BACKUP_SRAM_BASE;

  if (backup->magic == 0xDEADBEEF)
  {
    printf("Valid backup data found\n");
    printf("Boot count: %lu\n", backup->boot_count);
  }
  else
  {
    printf("No valid backup data (first boot)\n");
    Save_To_BackupSRAM();  // 초기화
  }
}
```

## 고급 기능

### 1. 조건부 스탠바이 진입

```c
void Conditional_Standby(void)
{
  // 조건 확인 (예: 배터리 잔량, 시간 등)
  uint32_t battery_level = Get_Battery_Level();

  if (battery_level < 20)
  {
    printf("Low battery, entering standby\n");
    Enter_Standby_Mode();
  }
  else
  {
    printf("Battery OK, staying in run mode\n");
  }
}
```

### 2. 스탠바이 타이머

```c
// RTC 웨이크업 타이머 사용
void Standby_With_Timer(uint32_t seconds)
{
  // RTC 웨이크업 타이머 설정
  HAL_RTCEx_SetWakeUpTimer_IT(&hrtc, seconds, RTC_WAKEUPCLOCK_CK_SPRE_16BITS);

  // 스탠바이 진입
  Enter_Standby_Mode();

  // seconds 후 자동 웨이크업
}
```

### 3. 웨이크업 소스 확인

```c
void Check_Wakeup_Source(void)
{
  // 웨이크업 핀 확인
  if (__HAL_PWR_GET_FLAG(PWR_WAKEUP_FLAG1))
  {
    printf("Woken up by WKUP1\n");
    __HAL_PWR_CLEAR_FLAG(PWR_WAKEUP_FLAG1);
  }

  if (__HAL_PWR_GET_FLAG(PWR_WAKEUP_FLAG2))
  {
    printf("Woken up by WKUP2\n");
    __HAL_PWR_CLEAR_FLAG(PWR_WAKEUP_FLAG2);
  }

  // RTC 알람 확인
  if (__HAL_RTC_ALARM_GET_FLAG(&hrtc, RTC_FLAG_ALRAF))
  {
    printf("Woken up by RTC Alarm\n");
    __HAL_RTC_ALARM_CLEAR_FLAG(&hrtc, RTC_FLAG_ALRAF);
  }
}
```

## 빌드 및 실행

### 테스트 시나리오

```
1. 보드 연결 및 플래싱
2. 시리얼 터미널 연결 (115200 baud)
3. 리셋 후 메시지 확인: "Normal boot"
4. LED1 점멸 확인 (실행 중)
5. USER 버튼 누름
6. 메시지 확인: "Entering STANDBY mode..."
7. LED OFF 확인
8. 전류 측정: ~2.4 μA
9. USER 버튼 다시 누름 (웨이크업)
10. 메시지 확인: "Woke up from STANDBY mode"
11. LED2 깜박임 확인
```

## 트러블슈팅

### 스탠바이 모드로 진입하지 않는 경우

```c
// 웨이크업 핀이 활성화되어 있는지 확인
uint32_t csr1 = PWR->WKUPCR;
printf("PWR_WKUPCR: 0x%08lX\n", csr1);

// 웨이크업 핀 비활성화 후 재시도
HAL_PWR_DisableWakeUpPin(PWR_WAKEUP_PIN2);
HAL_Delay(10);
HAL_PWR_EnableWakeUpPin(PWR_WAKEUP_PIN2_HIGH);
```

### 웨이크업이 즉시 발생하는 경우

```c
// 웨이크업 플래그를 클리어했는지 확인
__HAL_PWR_CLEAR_FLAG(PWR_FLAG_SB);
__HAL_PWR_CLEAR_FLAG(PWR_WAKEUP_FLAG_ALL);

// 웨이크업 핀 상태 확인
if (HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_SET)
{
  printf("WKUP pin already active!\n");
  // 버튼을 놓을 때까지 대기
  while (HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_SET);
  HAL_Delay(100);
}
```

### 백업 레지스터 데이터 손실

```c
// VBAT 전원 확인
// 백업 도메인 리셋 확인
if (__HAL_RCC_GET_FLAG(RCC_FLAG_BORRST))
{
  printf("Backup domain was reset\n");
  // 백업 데이터 재초기화
}
```

## 참고 자료

- **RM0399**: STM32H745 Reference Manual, Chapter 7 (PWR)
- **AN4879**: Introduction to power management on STM32H7 MCUs
- **DS12110**: STM32H745 Datasheet (전력 소비 특성)
- **예제 코드**: `STM32Cube_FW_H7/Projects/STM32H745I-DISCO/Examples/PWR/PWR_STANDBY`

## 관련 예제

- **PWR_STANDBY_RTC**: RTC 알람으로 웨이크업
- **PWR_STOP_RTC**: Stop 모드 (SRAM 보존)
- **PWR_Domain3SystemControl**: 도메인별 전력 제어
