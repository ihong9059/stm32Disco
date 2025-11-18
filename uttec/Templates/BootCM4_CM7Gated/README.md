# BootCM4_CM7Gated - CM4 부팅, CM7 게이트 템플릿

## 개요

이 템플릿은 **Cortex-M4만 자동으로 부팅**하고, Cortex-M7은 클럭이 게이트(중지)된 상태로 시작합니다. CM4가 시스템을 초기화하고 필요한 시점에 CM7을 활성화합니다.

## 부팅 구성

### User Option Bytes
- **BCM4 = 1**: CM4 부팅 활성화
- **BCM7 = 0**: CM7 부팅 비활성화 (클럭 게이트)

## 사용 사례

### 적합한 경우
1. **CM4가 메인 컨트롤러**인 경우
2. **저전력 애플리케이션**: CM7이 항상 필요하지 않은 경우
3. **동적 부하 분산**: 필요 시에만 CM7 활성화
4. **CM4 우선 초기화**: 특정 하드웨어를 CM4가 먼저 설정해야 하는 경우

### 부적합한 경우
1. CM7의 고성능이 항상 필요한 경우
2. CM7이 시스템 마스터여야 하는 경우

## 동작 시퀀스

```
Time
  │
  │ Reset
  ├─────────────────────────────────
  │
  │    CM4                     CM7
  │     │                       │
  ├─────┼───────────────────────×─────
  │     │ Boot                  Clock OFF
  │     │ (Active)              (Gated)
  ├─────┼───────────────────────×─────
  │     │
  │     │ HAL_Init()
  │     │
  │     │ SystemClock_Config()
  │     │
  │     │ Peripheral_Init()
  │     │
  │     │ Application_Init()
  │     │
  ├─────┼───────────────────────┼─────
  │     │ Enable CM7 Boot       │
  │     │ RCC_CM7Boot()   ─────►│ Clock ON
  │     │                       │ Boot
  │     │                       │ HAL_Init()
  ├─────┼───────────────────────┼─────
  │     │                       │
  │     │ HSEM Sync       ◄────►│
  │     │                       │
  ├─────┼───────────────────────┼─────
  │     │                       │
  │     │ Application           │ Application
  │     │ Running               │ Running
  │     │                       │
  ▼     ▼                       ▼
```

## CM4 코드 (마스터)

### 메인 함수
```c
int main(void)
{
  // HAL 초기화 (CM4가 먼저)
  HAL_Init();

  // 시스템 클럭 설정
  // CM4: 200MHz, CM7: 400MHz (부팅 후)
  SystemClock_Config();

  // LED 초기화
  BSP_LED_Init(LED1);
  BSP_LED_Init(LED2);

  // CM4 주변장치 초기화
  MX_GPIO_Init();
  MX_I2C_Init();
  MX_SPI_Init();

  // HSEM 초기화
  __HAL_RCC_HSEM_CLK_ENABLE();

  printf("CM4: System initialized\n");
  BSP_LED_On(LED1);

  // CM4 애플리케이션 로직
  // ...

  // 특정 조건에서 CM7 활성화
  if (need_high_performance_processing())
  {
    printf("CM4: Enabling CM7...\n");

    // CM7 부팅
    Enable_CM7();

    // CM7 준비 대기
    while(__HAL_HSEM_SEMID_GET(HSEM_ID_0) != 0);

    printf("CM4: CM7 is ready\n");
    BSP_LED_On(LED2);
  }

  // 메인 루프
  while (1)
  {
    BSP_LED_Toggle(LED1);
    HAL_Delay(500);

    // CM4 작업 수행
    Process_CM4_Tasks();
  }
}
```

### CM7 부팅 함수
```c
void Enable_CM7(void)
{
  // CM7 클럭 활성화
  __HAL_RCC_CM7_CLK_ENABLE();

  // CM7 리셋 해제
  __HAL_RCC_CM7_FORCE_RESET();
  HAL_Delay(1);
  __HAL_RCC_CM7_RELEASE_RESET();

  // 또는 HAL 함수 사용
  HAL_RCCEx_EnableBootCore(RCC_BOOT_C1);  // C1 = CM7

  printf("CM4: CM7 boot initiated\n");
}
```

### 시스템 클럭 설정 (CM4)
```c
void SystemClock_Config(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  // 전압 스케일링
  __HAL_PWR_VOLTAGESCALING_CONFIG(PWR_REGULATOR_VOLTAGE_SCALE1);

  while(!__HAL_PWR_GET_FLAG(PWR_FLAG_VOSRDY)) {}

  // HSE 설정
  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
  RCC_OscInitStruct.HSEState = RCC_HSE_ON;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
  RCC_OscInitStruct.PLL.PLLM = 5;
  RCC_OscInitStruct.PLL.PLLN = 160;
  RCC_OscInitStruct.PLL.PLLP = 2;
  RCC_OscInitStruct.PLL.PLLQ = 4;
  RCC_OscInitStruct.PLL.PLLR = 2;
  RCC_OscInitStruct.PLL.PLLRGE = RCC_PLL1VCIRANGE_2;
  RCC_OscInitStruct.PLL.PLLVCOSEL = RCC_PLL1VCOWIDE;
  RCC_OscInitStruct.PLL.PLLFRACN = 0;
  HAL_RCC_OscConfig(&RCC_OscInitStruct);

  // 클럭 설정
  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK
                              |RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2
                              |RCC_CLOCKTYPE_D3PCLK1|RCC_CLOCKTYPE_D1PCLK1;
  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
  RCC_ClkInitStruct.SYSCLKDivider = RCC_SYSCLK_DIV1;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_HCLK_DIV2;  // 200MHz
  RCC_ClkInitStruct.APB3CLKDivider = RCC_APB3_DIV2;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_APB1_DIV2;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_APB2_DIV2;
  RCC_ClkInitStruct.APB4CLKDivider = RCC_APB4_DIV2;

  HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_2);
}
```

## CM7 코드 (슬레이브)

### 메인 함수
```c
int main(void)
{
  // HAL 초기화
  HAL_Init();

  // LED 초기화
  BSP_LED_Init(LED3);
  BSP_LED_Init(LED4);

  // CM7 전용 주변장치 초기화
  MX_DMA_Init();
  MX_LTDC_Init();
  MX_DMA2D_Init();

  // HSEM으로 CM4에 준비 완료 신호
  __HAL_RCC_HSEM_CLK_ENABLE();
  HAL_HSEM_FastTake(HSEM_ID_0);
  HAL_HSEM_Release(HSEM_ID_0, 0);

  printf("CM7: Ready\n");
  BSP_LED_On(LED3);

  // CM7 메인 루프
  while (1)
  {
    BSP_LED_Toggle(LED3);
    HAL_Delay(1000);

    // 고성능 처리 작업
    Process_CM7_Tasks();
  }
}
```

## 코어 간 통신

### 공유 메모리
```c
// D3 SRAM을 코어 간 통신에 사용
#define SHARED_MEM_BASE   0x38000000

typedef struct {
  uint32_t cm4_status;
  uint32_t cm7_status;
  uint32_t cm4_to_cm7_cmd;
  uint32_t cm7_to_cm4_cmd;
  uint8_t  shared_data[4096];
} SharedMemory_t;

#define SHARED_MEM  ((SharedMemory_t*)SHARED_MEM_BASE)

// CM4: CM7에 명령 전송
void CM4_SendCommand_ToCM7(uint32_t cmd)
{
  SHARED_MEM->cm4_to_cm7_cmd = cmd;

  // HSEM으로 CM7에 알림
  HAL_HSEM_ActivateNotification(__HAL_HSEM_SEMID_TO_MASK(HSEM_CH_CMD));
}

// CM7: CM4 명령 수신
void CM7_ReceiveCommand_FromCM4(void)
{
  uint32_t cmd = SHARED_MEM->cm4_to_cm7_cmd;

  switch(cmd)
  {
    case CMD_START_PROCESSING:
      Start_HighPerformance_Task();
      break;

    case CMD_STOP_PROCESSING:
      Stop_HighPerformance_Task();
      break;

    default:
      break;
  }

  SHARED_MEM->cm4_to_cm7_cmd = 0;  // 명령 클리어
}
```

## 실제 사용 예제

### 예제 1: 카메라 시스템
```c
// CM4: 카메라 제어, I2C, 전력 관리
// CM7: 이미지 처리 (필요 시에만 활성화)

void Camera_System_Init(void)
{
  // CM4: 카메라 초기화
  Camera_Init();
  Camera_Start_Preview();

  // 프리뷰는 CM4가 처리 (저전력)
  // 사진 촬영 시 CM7 활성화
}

void Camera_TakePicture(void)
{
  // CM4: CM7 활성화
  if (!cm7_running)
  {
    Enable_CM7();
    cm7_running = 1;
  }

  // CM4: 이미지 데이터를 공유 메모리로
  Capture_Image_To_Shared_Memory();

  // CM7에 처리 요청
  CM4_SendCommand_ToCM7(CMD_PROCESS_IMAGE);

  // CM7: 이미지 처리 (JPEG 인코딩, 필터 등)
  // ...

  // 처리 완료 후 CM7 비활성화 (전력 절약)
  Disable_CM7();
  cm7_running = 0;
}
```

### 예제 2: 센서 모니터링
```c
// CM4: 상시 센서 모니터링 (저전력)
// CM7: 복잡한 분석 (이상 감지 시에만)

void Sensor_Monitoring_Task(void)
{
  while (1)
  {
    // CM4: 센서 데이터 읽기
    float temperature = Read_Temperature();
    float pressure = Read_Pressure();

    // 정상 범위 체크
    if (temperature > TEMP_THRESHOLD || pressure > PRESSURE_THRESHOLD)
    {
      // 이상 감지 - CM7 활성화하여 상세 분석
      if (!cm7_running)
      {
        Enable_CM7();
        cm7_running = 1;
      }

      // CM7에 분석 요청
      CM4_SendCommand_ToCM7(CMD_ANALYZE_ANOMALY);
    }
    else
    {
      // 정상 - CM7 비활성화
      if (cm7_running)
      {
        Disable_CM7();
        cm7_running = 0;
      }
    }

    HAL_Delay(100);
  }
}
```

## CM7 비활성화

```c
void Disable_CM7(void)
{
  // CM7에 종료 신호
  CM4_SendCommand_ToCM7(CMD_SHUTDOWN);

  // CM7 종료 대기
  HAL_Delay(10);

  // CM7 클럭 게이트
  __HAL_RCC_CM7_CLK_DISABLE();

  printf("CM4: CM7 disabled\n");
}

// CM7: 종료 핸들러
void CM7_Shutdown_Handler(void)
{
  // 주변장치 비활성화
  HAL_LTDC_DeInit(&hltdc);
  HAL_DMA2D_DeInit(&hdma2d);

  // HSEM 해제
  HAL_HSEM_ReleaseAll(0);

  // 종료 준비 완료
  SHARED_MEM->cm7_status = CM7_STATUS_SHUTDOWN;

  // WFI로 대기 (클럭 게이트 될 때까지)
  while (1)
  {
    __WFI();
  }
}
```

## Option Bytes 설정

### STM32CubeProgrammer 사용
```bash
# 현재 설정 읽기
STM32_Programmer_CLI -c port=SWD -r32 0x5200201C 1

# BCM4=1, BCM7=0 설정
STM32_Programmer_CLI -c port=SWD -ob BCM4=1 BCM7=0

# 시스템 리셋
STM32_Programmer_CLI -c port=SWD -rst
```

### 프로그래밍 방식
```c
void Set_Boot_CM4_Only(void)
{
  FLASH_OBProgramInitTypeDef OBInit;

  HAL_FLASH_Unlock();
  HAL_FLASH_OB_Unlock();

  // 현재 설정 읽기
  HAL_FLASHEx_OBGetConfig(&OBInit);

  // BCM4=1, BCM7=0
  OBInit.BootConfig = OB_BCM4_ENABLE | OB_BCM7_DISABLE;

  // 프로그램
  HAL_FLASHEx_OBProgram(&OBInit);

  // 적용 (시스템 리셋)
  HAL_FLASH_OB_Launch();

  HAL_FLASH_OB_Lock();
  HAL_FLASH_Lock();
}
```

## 전력 소비

### CM4만 실행
- **소비 전력**: ~30-40mA @ 200MHz
- **사용 사례**: 센서 모니터링, 저전력 제어

### CM4 + CM7 실행
- **소비 전력**: ~100-120mA @ (CM4: 200MHz, CM7: 400MHz)
- **사용 사례**: 고성능 처리 필요 시

### 전력 절약 전략
```c
// CM7을 동적으로 ON/OFF
uint32_t total_power_on_time = 0;
uint32_t cm7_active_time = 0;

void Optimize_Power_Consumption(void)
{
  float cm7_usage = (float)cm7_active_time / total_power_on_time;

  printf("CM7 usage: %.1f%%\n", cm7_usage * 100);

  // CM7 사용률이 낮으면 비활성화
  if (cm7_usage < 0.1f && cm7_running)
  {
    Disable_CM7();
  }
}
```

## 트러블슈팅

### CM7이 부팅되지 않는 경우
1. **Option Bytes 확인**: BCM4=1, BCM7=0
2. **Enable 함수 호출**: `HAL_RCCEx_EnableBootCore(RCC_BOOT_C1)` 호출 확인
3. **CM7 바이너리**: Flash에 올바르게 프로그램되었는지 확인

### CM4가 CM7을 제어할 수 없는 경우
1. **RCC 권한**: CM4가 CM7 클럭 제어 권한 있는지 확인
2. **HSEM 동기화**: 타이밍 이슈 확인
3. **공유 메모리**: MPU 설정 확인

## 다른 부팅 모드와 비교

| 특성 | BootCM4_CM7Gated | BootCM7_CM4Gated | BootCM4_CM7 |
|------|------------------|------------------|-------------|
| CM4 초기 상태 | Running | Gated | Running |
| CM7 초기 상태 | Gated | Running | Running |
| 초기화 담당 | CM4 | CM7 | CM7 |
| 전력 효율 | ⭐⭐⭐ | ⭐⭐⭐ | ⭐ |
| 제어 복잡도 | ⭐⭐⭐ | ⭐⭐ | ⭐ |
| 사용 빈도 | 낮음 | 높음 | 높음 |

## 참고 자료

- AN5361: Getting started with projects based on dual-core STM32H7
- RM0399: STM32H745/755 Reference Manual
- 예제: `STM32H745I-DISCO/Templates/BootCM4_CM7Gated`
