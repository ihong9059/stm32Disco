# FreeRTOS_AMP_RTOS_BareMetal - FreeRTOS + Bare-Metal 혼합 구성

## 개요

이 애플리케이션은 STM32H745의 비대칭 멀티프로세싱(AMP) 환경에서 Cortex-M7은 FreeRTOS를 실행하고, Cortex-M4는 Bare-Metal(RTOS 없음)로 실행하는 혼합 구성을 보여줍니다. 두 코어 간의 통신은 공유 메모리와 HSEM을 통해 이루어집니다.

## 혼합 구성의 필요성

### 사용 사례
- **실시간 제어**: CM4는 결정적 타이밍 필요 (모터 제어, ADC 샘플링)
- **복잡한 애플리케이션**: CM7은 GUI, 네트워크 스택 등
- **저전력**: CM4는 최소한의 오버헤드로 저전력 동작
- **레거시 통합**: 기존 Bare-Metal 코드 재사용

### 장단점
| 구성 | 장점 | 단점 |
|------|------|------|
| CM7: RTOS | 멀티태스킹, 유연성 | 오버헤드, 비결정적 |
| CM4: Bare-Metal | 결정적, 저오버헤드 | 단일 루프, 수동 관리 |

## 시스템 아키텍처

### 혼합 AMP 구조
```
┌──────────────────────────────────────────────────────────────┐
│                        Cortex-M7                             │
│                        @ 400 MHz                             │
│  ┌─────────────────────────────────────────────────────┐     │
│  │                    FreeRTOS                         │     │
│  │  ┌─────────────┐  ┌─────────────┐  ┌────────────┐   │     │
│  │  │  GUI Task   │  │  Comm Task  │  │  App Task  │   │     │
│  │  │  (Display)  │  │  (Network)  │  │  (Logic)   │   │     │
│  │  └──────┬──────┘  └──────┬──────┘  └─────┬──────┘   │     │
│  │         │                │               │          │     │
│  │  ┌──────▼────────────────▼───────────────▼──────┐   │     │
│  │  │          Inter-Core Communication            │   │     │
│  │  └──────────────────┬───────────────────────────┘   │     │
│  └─────────────────────┼───────────────────────────────┘     │
└────────────────────────┼─────────────────────────────────────┘
                         │
                         │  Shared Memory (D3 SRAM)
                         │  HSEM Notification
                         │
┌────────────────────────┼─────────────────────────────────────┐
│                        │                                     │
│  ┌─────────────────────▼───────────────────────────────┐     │
│  │                 Bare-Metal                          │     │
│  │                                                     │     │
│  │  while (1)                                          │     │
│  │  {                                                  │     │
│  │    // 인터럽트 처리                                 │     │
│  │    // 센서 읽기                                     │     │
│  │    // 제어 루프                                     │     │
│  │    // CM7 통신                                      │     │
│  │  }                                                  │     │
│  │                                                     │     │
│  └─────────────────────────────────────────────────────┘     │
│                        Cortex-M4                             │
│                        @ 200 MHz                             │
└──────────────────────────────────────────────────────────────┘
```

## 메모리 구성

### 메모리 맵
```
┌─────────────────────────────────────────────────┐
│  Flash Bank 1 (0x08000000)                      │
│  ├── CM7 Code (1MB)                             │
│  └── CM7 Data                                   │
├─────────────────────────────────────────────────┤
│  Flash Bank 2 (0x08100000)                      │
│  ├── CM4 Code (1MB)                             │
│  └── CM4 Data                                   │
├─────────────────────────────────────────────────┤
│  DTCM RAM (0x20000000) - CM7 전용               │
│  └── FreeRTOS Heap (64KB)                       │
├─────────────────────────────────────────────────┤
│  D1 AXI SRAM (0x24000000)                       │
│  └── CM7 Application Data (512KB)               │
├─────────────────────────────────────────────────┤
│  D2 SRAM (0x30000000)                           │
│  └── CM4 Application Data (288KB)               │
├─────────────────────────────────────────────────┤
│  D3 SRAM (0x38000000) - 공유 메모리             │
│  ├── Command Buffer                             │
│  ├── Data Buffer                                │
│  └── Status Flags (64KB)                        │
└─────────────────────────────────────────────────┘
```

### 공유 메모리 구조체
```c
// 공유 데이터 구조 (D3 SRAM에 배치)
#define SHARED_MEM_BASE  0x38000000

typedef struct {
  // 명령/상태
  volatile uint32_t cm7_to_cm4_cmd;
  volatile uint32_t cm4_to_cm7_status;
  volatile uint32_t error_code;

  // 데이터 버퍼
  uint8_t cm7_to_cm4_data[1024];
  uint8_t cm4_to_cm7_data[1024];

  // 센서 데이터 (CM4 -> CM7)
  volatile int16_t adc_values[8];
  volatile uint32_t encoder_count;
  volatile float temperature;

  // 제어 명령 (CM7 -> CM4)
  volatile int32_t motor_speed;
  volatile uint8_t gpio_outputs;
  volatile uint8_t control_mode;

  // 타임스탬프
  volatile uint32_t cm4_timestamp;
  volatile uint32_t cm7_timestamp;
} __attribute__((packed)) SharedData_TypeDef;

// 공유 메모리 인스턴스
#define shared_data  ((SharedData_TypeDef *)SHARED_MEM_BASE)
```

## HSEM 통신 프로토콜

### HSEM ID 할당
```c
#define HSEM_ID_CM7_TO_CM4_CMD    0   // CM7 -> CM4 명령 알림
#define HSEM_ID_CM4_TO_CM7_DATA   1   // CM4 -> CM7 데이터 알림
#define HSEM_ID_SHARED_MEM_LOCK   2   // 공유 메모리 잠금
```

### 알림 메커니즘
```c
// CM7: CM4에 명령 전송
void CM7_SendCommand(uint32_t cmd, void *data, uint32_t size)
{
  // 공유 메모리 잠금
  while (HAL_HSEM_FastTake(HSEM_ID_SHARED_MEM_LOCK) != HAL_OK);

  // 데이터 복사
  if (data && size > 0)
  {
    memcpy((void *)shared_data->cm7_to_cm4_data, data, size);
  }

  // 명령 설정
  shared_data->cm7_to_cm4_cmd = cmd;
  shared_data->cm7_timestamp = HAL_GetTick();

  // 공유 메모리 해제
  HAL_HSEM_Release(HSEM_ID_SHARED_MEM_LOCK, 0);

  // CM4에 알림
  HAL_HSEM_ActivateNotification(__HAL_HSEM_SEMID_TO_MASK(HSEM_ID_CM7_TO_CM4_CMD));
}

// CM4: 명령 수신 (인터럽트에서)
void HSEM2_IRQHandler(void)
{
  HAL_HSEM_IRQHandler();
  command_pending = 1;
}
```

## 코드 구현

### 1. CM7 메인 (FreeRTOS)
```c
#include "FreeRTOS.h"
#include "task.h"
#include "semphr.h"

// 태스크 핸들
TaskHandle_t xGUITaskHandle;
TaskHandle_t xCommTaskHandle;
TaskHandle_t xControlTaskHandle;

// 동기화
SemaphoreHandle_t xSharedMemMutex;
SemaphoreHandle_t xDataReadySem;

int main(void)
{
  // 시스템 초기화
  HAL_Init();
  SystemClock_Config();

  // 주변장치 초기화
  MX_GPIO_Init();
  BSP_LED_Init(LED1);
  BSP_LED_Init(LED2);

  // HSEM 초기화
  __HAL_RCC_HSEM_CLK_ENABLE();
  HAL_NVIC_SetPriority(HSEM1_IRQn, 5, 0);
  HAL_NVIC_EnableIRQ(HSEM1_IRQn);

  // 공유 메모리 초기화
  memset((void *)SHARED_MEM_BASE, 0, sizeof(SharedData_TypeDef));

  // FreeRTOS 동기화 객체 생성
  xSharedMemMutex = xSemaphoreCreateMutex();
  xDataReadySem = xSemaphoreCreateBinary();

  // CM4 부팅
  HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);

  // 태스크 생성
  xTaskCreate(vGUITask, "GUI", 1024, NULL, 2, &xGUITaskHandle);
  xTaskCreate(vCommTask, "Comm", 512, NULL, 3, &xCommTaskHandle);
  xTaskCreate(vControlTask, "Control", 512, NULL, 4, &xControlTaskHandle);

  // 스케줄러 시작
  vTaskStartScheduler();

  while (1);
}
```

### 2. CM7 통신 태스크
```c
void vCommTask(void *pvParameters)
{
  for (;;)
  {
    // CM4 데이터 대기
    if (xSemaphoreTake(xDataReadySem, pdMS_TO_TICKS(100)) == pdTRUE)
    {
      // 공유 메모리 접근
      if (xSemaphoreTake(xSharedMemMutex, pdMS_TO_TICKS(50)) == pdTRUE)
      {
        // CM4 데이터 처리
        ProcessCM4Data();
        xSemaphoreGive(xSharedMemMutex);
      }
    }

    // 주기적으로 명령 전송
    static uint32_t last_cmd_time = 0;
    if ((HAL_GetTick() - last_cmd_time) > 1000)
    {
      SendControlCommand();
      last_cmd_time = HAL_GetTick();
    }
  }
}

void ProcessCM4Data(void)
{
  // ADC 값 읽기
  int16_t adc[8];
  for (int i = 0; i < 8; i++)
  {
    adc[i] = shared_data->adc_values[i];
  }

  // 엔코더 값
  uint32_t encoder = shared_data->encoder_count;

  // 온도
  float temp = shared_data->temperature;

  // 데이터 처리 (필터링, 변환 등)
  printf("ADC[0]=%d, Encoder=%lu, Temp=%.1f\n", adc[0], encoder, temp);

  BSP_LED_Toggle(LED1);
}

void SendControlCommand(void)
{
  if (xSemaphoreTake(xSharedMemMutex, pdMS_TO_TICKS(50)) == pdTRUE)
  {
    // 모터 속도 설정
    shared_data->motor_speed = desired_speed;

    // 제어 모드 설정
    shared_data->control_mode = current_mode;

    // 명령 전송
    shared_data->cm7_to_cm4_cmd = CMD_SET_CONTROL;

    xSemaphoreGive(xSharedMemMutex);

    // CM4에 알림
    HAL_HSEM_ActivateNotification(__HAL_HSEM_SEMID_TO_MASK(HSEM_ID_CM7_TO_CM4_CMD));
  }
}
```

### 3. CM7 HSEM 인터럽트
```c
void HSEM1_IRQHandler(void)
{
  HAL_HSEM_IRQHandler();
}

void HAL_HSEM_FreeCallback(uint32_t SemMask)
{
  if (SemMask & __HAL_HSEM_SEMID_TO_MASK(HSEM_ID_CM4_TO_CM7_DATA))
  {
    // CM4에서 새 데이터
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    xSemaphoreGiveFromISR(xDataReadySem, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
  }
}
```

### 4. CM4 메인 (Bare-Metal)
```c
// 플래그
volatile uint8_t command_pending = 0;
volatile uint8_t timer_tick = 0;

int main(void)
{
  // HSEM으로 CM7 준비 대기
  __HAL_RCC_HSEM_CLK_ENABLE();

  // HAL 초기화
  HAL_Init();

  // 주변장치 초기화
  MX_GPIO_Init();
  MX_ADC1_Init();
  MX_TIM1_Init();  // 모터 PWM
  MX_TIM2_Init();  // 엔코더

  // LED 초기화
  BSP_LED_Init(LED3);
  BSP_LED_Init(LED4);

  // HSEM 인터럽트 설정
  HAL_NVIC_SetPriority(HSEM2_IRQn, 0, 0);
  HAL_NVIC_EnableIRQ(HSEM2_IRQn);

  // 타이머 인터럽트 시작 (1ms)
  HAL_TIM_Base_Start_IT(&htim3);

  // 제어 루프 알림 활성화
  HAL_HSEM_ActivateNotification(__HAL_HSEM_SEMID_TO_MASK(HSEM_ID_CM7_TO_CM4_CMD));

  // 메인 루프 (Bare-Metal)
  while (1)
  {
    // 명령 처리
    if (command_pending)
    {
      command_pending = 0;
      ProcessCommand();
    }

    // 주기적 제어 루프 (1kHz)
    if (timer_tick)
    {
      timer_tick = 0;
      ControlLoop();
    }

    // 저전력 대기
    __WFI();
  }
}
```

### 5. CM4 제어 루프
```c
void ControlLoop(void)
{
  // ADC 읽기 (DMA 완료 확인)
  if (adc_ready)
  {
    for (int i = 0; i < 8; i++)
    {
      shared_data->adc_values[i] = adc_buffer[i];
    }
    adc_ready = 0;
  }

  // 엔코더 읽기
  shared_data->encoder_count = __HAL_TIM_GET_COUNTER(&htim2);

  // 온도 계산
  shared_data->temperature = CalculateTemperature(shared_data->adc_values[7]);

  // 모터 제어
  int32_t target_speed = shared_data->motor_speed;
  int32_t current_speed = GetCurrentSpeed();
  int32_t pwm_output = PID_Calculate(target_speed, current_speed);

  // PWM 출력
  __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, pwm_output);

  // 타임스탬프 업데이트
  shared_data->cm4_timestamp = HAL_GetTick();

  // 주기적으로 CM7에 알림 (100Hz)
  static uint32_t notify_counter = 0;
  if (++notify_counter >= 10)
  {
    notify_counter = 0;
    HAL_HSEM_ActivateNotification(__HAL_HSEM_SEMID_TO_MASK(HSEM_ID_CM4_TO_CM7_DATA));
    BSP_LED_Toggle(LED3);
  }
}
```

### 6. CM4 명령 처리
```c
void ProcessCommand(void)
{
  uint32_t cmd = shared_data->cm7_to_cm4_cmd;

  switch (cmd)
  {
    case CMD_SET_CONTROL:
      // 제어 파라미터 업데이트
      ApplyControlParameters();
      break;

    case CMD_START_MOTOR:
      StartMotor();
      break;

    case CMD_STOP_MOTOR:
      StopMotor();
      break;

    case CMD_CALIBRATE:
      RunCalibration();
      break;

    case CMD_GET_STATUS:
      // 상태 준비
      shared_data->cm4_to_cm7_status = GetSystemStatus();
      HAL_HSEM_ActivateNotification(__HAL_HSEM_SEMID_TO_MASK(HSEM_ID_CM4_TO_CM7_DATA));
      break;

    default:
      shared_data->error_code = ERR_UNKNOWN_CMD;
      break;
  }

  // 명령 처리 완료
  shared_data->cm7_to_cm4_cmd = CMD_NONE;
}
```

### 7. CM4 타이머 인터럽트
```c
void TIM3_IRQHandler(void)
{
  if (__HAL_TIM_GET_FLAG(&htim3, TIM_FLAG_UPDATE))
  {
    __HAL_TIM_CLEAR_FLAG(&htim3, TIM_FLAG_UPDATE);
    timer_tick = 1;
  }
}
```

## 명령 및 상태 정의

### 명령 코드
```c
typedef enum {
  CMD_NONE = 0,
  CMD_SET_CONTROL,
  CMD_START_MOTOR,
  CMD_STOP_MOTOR,
  CMD_CALIBRATE,
  CMD_GET_STATUS,
  CMD_RESET,
  CMD_SET_GPIO
} Command_TypeDef;
```

### 상태 코드
```c
typedef enum {
  STATUS_IDLE = 0,
  STATUS_RUNNING,
  STATUS_ERROR,
  STATUS_CALIBRATING
} Status_TypeDef;
```

### 에러 코드
```c
typedef enum {
  ERR_NONE = 0,
  ERR_UNKNOWN_CMD,
  ERR_INVALID_PARAM,
  ERR_TIMEOUT,
  ERR_HARDWARE
} ErrorCode_TypeDef;
```

## 고급 기능

### 1. 워치독 감시
```c
// CM7: CM4 상태 감시
void vWatchdogTask(void *pvParameters)
{
  uint32_t last_timestamp = 0;

  for (;;)
  {
    uint32_t current_timestamp = shared_data->cm4_timestamp;

    if (current_timestamp == last_timestamp)
    {
      // CM4가 응답하지 않음
      printf("CM4 Watchdog Timeout!\n");
      BSP_LED_On(LED2);

      // 복구 시도
      HAL_HSEM_ActivateNotification(__HAL_HSEM_SEMID_TO_MASK(HSEM_ID_CM7_TO_CM4_CMD));
    }
    else
    {
      last_timestamp = current_timestamp;
      BSP_LED_Off(LED2);
    }

    vTaskDelay(pdMS_TO_TICKS(100));
  }
}
```

### 2. 데이터 버퍼링
```c
// CM4: 링 버퍼를 사용한 센서 데이터 버퍼링
#define BUFFER_SIZE 256

typedef struct {
  int16_t data[BUFFER_SIZE];
  volatile uint32_t head;
  volatile uint32_t tail;
} RingBuffer_TypeDef;

RingBuffer_TypeDef sensor_buffer;

void BufferSensorData(int16_t value)
{
  uint32_t next = (sensor_buffer.head + 1) % BUFFER_SIZE;

  if (next != sensor_buffer.tail)
  {
    sensor_buffer.data[sensor_buffer.head] = value;
    sensor_buffer.head = next;
  }
}

// CM7: 버퍼된 데이터 읽기
void ReadBufferedData(void)
{
  while (sensor_buffer.head != sensor_buffer.tail)
  {
    int16_t value = sensor_buffer.data[sensor_buffer.tail];
    sensor_buffer.tail = (sensor_buffer.tail + 1) % BUFFER_SIZE;

    // 데이터 처리
    ProcessSensorValue(value);
  }
}
```

### 3. 실시간 성능 측정
```c
// CM4: 제어 루프 타이밍 측정
volatile uint32_t loop_min_time = UINT32_MAX;
volatile uint32_t loop_max_time = 0;
volatile uint32_t loop_avg_time = 0;
volatile uint32_t loop_count = 0;

void MeasureLoopTiming(void)
{
  static uint32_t last_time = 0;
  uint32_t current_time = DWT->CYCCNT;  // 사이클 카운터

  if (last_time > 0)
  {
    uint32_t elapsed = current_time - last_time;

    if (elapsed < loop_min_time) loop_min_time = elapsed;
    if (elapsed > loop_max_time) loop_max_time = elapsed;

    loop_avg_time = (loop_avg_time * loop_count + elapsed) / (loop_count + 1);
    loop_count++;
  }

  last_time = current_time;
}
```

## 응용 예제

### 1. 모터 제어 시스템
```c
// CM7: 상위 제어 (경로 계획)
void vMotionPlannerTask(void *pvParameters)
{
  for (;;)
  {
    // 목표 위치 계산
    CalculateTrajectory();

    // CM4에 속도 명령
    shared_data->motor_speed = calculated_speed;
    shared_data->cm7_to_cm4_cmd = CMD_SET_CONTROL;

    HAL_HSEM_ActivateNotification(__HAL_HSEM_SEMID_TO_MASK(HSEM_ID_CM7_TO_CM4_CMD));

    vTaskDelay(pdMS_TO_TICKS(10));  // 100Hz
  }
}

// CM4: 하위 제어 (전류 제어)
void ControlLoop(void)
{
  // 전류 측정
  int16_t current = MeasureMotorCurrent();

  // PID 제어
  int32_t output = PID_Control(shared_data->motor_speed, current);

  // PWM 출력
  SetMotorPWM(output);
}
```

### 2. 데이터 수집 시스템
```c
// CM4: 고속 ADC 샘플링
void ADC_Sampling(void)
{
  // 10kHz 샘플링
  HAL_ADC_Start_DMA(&hadc1, (uint32_t*)adc_buffer, ADC_BUFFER_SIZE);
}

// DMA 완료 인터럽트
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
  // 공유 메모리에 복사
  memcpy((void*)shared_data->cm4_to_cm7_data, adc_buffer, ADC_BUFFER_SIZE * 2);

  // CM7에 알림
  HAL_HSEM_ActivateNotification(__HAL_HSEM_SEMID_TO_MASK(HSEM_ID_CM4_TO_CM7_DATA));
}

// CM7: 데이터 분석
void vAnalysisTask(void *pvParameters)
{
  for (;;)
  {
    if (xSemaphoreTake(xDataReadySem, portMAX_DELAY) == pdTRUE)
    {
      // FFT, 필터링 등 복잡한 처리
      ProcessADCData((int16_t*)shared_data->cm4_to_cm7_data, ADC_BUFFER_SIZE);
    }
  }
}
```

## 트러블슈팅

### CM4가 응답하지 않는 경우

1. **부팅 순서 확인**: CM7이 먼저 초기화 완료
2. **HSEM 알림**: 올바르게 활성화 및 처리
3. **인터럽트 우선순위**: CM4 인터럽트가 차단되지 않음

### 데이터 불일치

1. **캐시 관리**: 공유 메모리는 non-cacheable
   ```c
   MPU_Region_InitTypeDef MPU_InitStruct;
   MPU_InitStruct.IsCacheable = MPU_ACCESS_NOT_CACHEABLE;
   ```

2. **원자성**: 대용량 데이터는 잠금 사용
3. **휘발성**: volatile 키워드 사용

### 타이밍 문제

1. **인터럽트 지연**: CM4 인터럽트 우선순위
2. **제어 루프 오버런**: 실행 시간 측정
3. **통신 지연**: HSEM 오버헤드

## 성능 고려사항

### CM4 결정성
- **인터럽트 레이턴시**: 최소 12 사이클
- **제어 루프 주기**: 안정적인 1kHz 달성 가능
- **최악 실행 시간**: 측정 및 보장 필요

### CM7 처리량
- **태스크 스위칭**: 수 us
- **멀티태스킹 효율**: 복잡한 작업 분배
- **GUI 응답성**: 우선순위 기반 관리

### 통신 대역폭
- **HSEM 알림**: 약 100ns
- **데이터 전송**: 메모리 복사 시간 의존
- **최적화**: DMA 활용

## 참고 자료

- FreeRTOS 공식 문서: https://www.freertos.org/
- AN5617: Asymmetric multiprocessing with STM32H7
- STM32H7 참조 매뉴얼: RM0399
- 예제: `STM32H745I-DISCO/Applications/FreeRTOS/FreeRTOS_AMP_RTOS_BareMetal`
