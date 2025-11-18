# PWR_Hold_Mechanism - 듀얼 코어 부팅 제어

## 개요

이 예제는 STM32H745 듀얼 코어 MCU에서 Hold Boot 메커니즘을 사용하여 Cortex-M7이 Cortex-M4의 부팅을 제어하는 방법을 보여줍니다. 이를 통해 코어 간 동기화, 리소스 초기화 순서 제어, 동적 코어 활성화 등을 구현할 수 있습니다.

## 하드웨어 요구사항

- **STM32H745I-DISCO 보드**
- **LED1**: PI12 (주황색) - M7 상태
- **LED2**: PI13 (초록색) - M4 상태
- **USER 버튼**: PC13 - M4 부팅 트리거
- **UART**: 115200 baud (디버깅)
- **전류계**: 전력 측정용 (선택 사항)

## STM32H745 듀얼 코어 부팅 구조

### 기본 부팅 순서

```
┌─────────────────────────────────────────────────────────┐
│            STM32H745 Boot Sequence                       │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─────────────┐                                        │
│  │  Power-On   │                                        │
│  │   Reset     │                                        │
│  └──────┬──────┘                                        │
│         │                                                │
│         ▼                                                │
│  ┌─────────────┐                                        │
│  │  System     │  • Clock: HSI 64 MHz                   │
│  │  Startup    │  • Flash: default latency              │
│  └──────┬──────┘                                        │
│         │                                                │
│         ▼                                                │
│  ┌─────────────┐                                        │
│  │   CM7       │  ★ M7 항상 먼저 부팅                    │
│  │   Boot      │  Flash: 0x08000000                     │
│  │             │                                        │
│  │ BCM4 bit?   │                                        │
│  │             │                                        │
│  └──────┬──────┘                                        │
│         │                                                │
│    ┌────┴────┐                                          │
│    │         │                                          │
│    ▼         ▼                                          │
│  BCM4=1    BCM4=0                                       │
│  ┌─────┐   ┌─────┐                                      │
│  │CM4  │   │CM4  │                                      │
│  │Boot │   │Held │                                      │
│  │     │   │     │                                      │
│  │0x081│   │Wait │  ★ Hold Boot Mechanism               │
│  │00000│   │     │                                      │
│  └─────┘   └─────┘                                      │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Option Byte 설정

```
┌─────────────────────────────────────────────────────────┐
│                  Option Bytes                            │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  FLASH_OPTSR (0x5200201C):                               │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Bit 7: BCM4 - Boot CM4                          │    │
│  │        0: CM4 boot held (M7이 제어)              │    │
│  │        1: CM4 auto boot (자동 부팅)              │    │
│  │                                                  │    │
│  │ Bit 6: BCM7 - Boot CM7                          │    │
│  │        0: CM7 boot held                         │    │
│  │        1: CM7 auto boot (기본값)                 │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  Boot Address (SYSCFG):                                  │
│  ┌─────────────────────────────────────────────────┐    │
│  │ UR2: CM4 boot address (default: 0x08100000)     │    │
│  │ UR3: CM7 boot address (default: 0x08000000)     │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Hold Boot 메커니즘

```
┌─────────────────────────────────────────────────────────┐
│            Hold Boot Mechanism                           │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  CM7 (Master)                    CM4 (Slave)            │
│  ┌─────────────┐                 ┌─────────────┐        │
│  │  System     │                 │   Held in   │        │
│  │  Startup    │                 │    Reset    │        │
│  └──────┬──────┘                 └──────┬──────┘        │
│         │                               │               │
│         ▼                               │               │
│  ┌─────────────┐                        │               │
│  │  Initialize │                        │               │
│  │   • Clock   │                        │               │
│  │   • Memory  │                        │               │
│  │   • HSEM    │                        │               │
│  └──────┬──────┘                        │               │
│         │                               │               │
│         ▼                               │               │
│  ┌─────────────┐                        │               │
│  │   Setup     │                        │               │
│  │  Shared     │                        │               │
│  │  Resources  │                        │               │
│  └──────┬──────┘                        │               │
│         │                               │               │
│         │  BOOT_CM4()                   │               │
│         ├───────────────────────────────►               │
│         │                               │               │
│         │                        ┌──────┴──────┐        │
│         │                        │    Boot     │        │
│         │                        │    CM4      │        │
│         │                        └──────┬──────┘        │
│         │                               │               │
│         │    HSEM (Sync)                │               │
│         ◄───────────────────────────────┤               │
│         │                               │               │
│         ▼                               ▼               │
│  ┌─────────────┐                 ┌─────────────┐        │
│  │   CM7       │                 │   CM4       │        │
│  │   Main      │                 │   Main      │        │
│  │   Loop      │                 │   Loop      │        │
│  └─────────────┘                 └─────────────┘        │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

## 주요 기능

### Hold Boot의 장점

1. **리소스 초기화 순서 제어**
   - M7이 먼저 공유 리소스 설정
   - 레이스 컨디션 방지
   - 안전한 초기화 보장

2. **동적 코어 활성화**
   - 필요할 때만 M4 부팅
   - 전력 절감
   - 유연한 시스템 설계

3. **코어 간 동기화**
   - HSEM으로 상호 배제
   - 메시지 전달
   - 데이터 공유

4. **디버깅 용이**
   - 단일 코어로 시작
   - 점진적 통합
   - 문제 격리

### 사용 시나리오

```
시나리오                │ 설명                        │ M7        │ M4
─────────────────────┼────────────────────────────┼──────────┼──────
부팅 시 동시 시작      │ BCM4=1, M4 자동 부팅        │ Auto      │ Auto
부팅 시 M4 대기        │ BCM4=0, M7이 제어           │ Auto      │ Held
동적 M4 활성화         │ M4 필요할 때만 부팅         │ Running   │ Held→Run
M4 재시작             │ 오류 복구                    │ Running   │ Reset→Run
에너지 하베스팅        │ 전력 가용 시 M4 부팅        │ Running   │ Held→Run
```

## 코드 구현

### 1. M7 부트 코드 (마스터)

```c
// M7 Flash: 0x08000000
// M7이 M4의 부팅을 제어

#include "stm32h7xx_hal.h"

// 공유 메모리 영역 (SRAM3)
#define SHARED_MEMORY_BASE  0x30040000
#define SHARED_MEMORY_SIZE  32768

// 하드웨어 세마포어
#define HSEM_CM7_READY    0
#define HSEM_CM4_READY    1
#define HSEM_SHARED_DATA  2

typedef struct {
  uint32_t magic;           // 0xDEADBEEF
  uint32_t cm7_status;
  uint32_t cm4_status;
  uint32_t command;
  uint8_t data[256];
} SharedData_t;

volatile SharedData_t *shared_data = (SharedData_t*)SHARED_MEMORY_BASE;

int main(void)
{
  // HAL 초기화
  HAL_Init();

  // 시스템 클럭 설정 (480 MHz)
  SystemClock_Config();

  // GPIO 초기화
  GPIO_Init();

  // UART 초기화
  UART_Init();

  printf("\n========================================\n");
  printf("STM32H745 Hold Boot Mechanism Demo\n");
  printf("========================================\n\n");

  printf("CM7 started.\n\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 1. HSEM (Hardware Semaphore) 초기화
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  __HAL_RCC_HSEM_CLK_ENABLE();

  printf("Hardware Semaphore initialized.\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 2. 공유 메모리 초기화
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // 공유 메모리 세마포어 획득
  HAL_HSEM_FastTake(HSEM_SHARED_DATA);

  // 공유 메모리 클리어
  memset((void*)shared_data, 0, sizeof(SharedData_t));

  // 초기화 완료 표시
  shared_data->magic = 0xDEADBEEF;
  shared_data->cm7_status = 1;  // CM7 ready

  // 세마포어 해제
  HAL_HSEM_Release(HSEM_SHARED_DATA, 0);

  printf("Shared memory initialized at 0x%08lX\n", (uint32_t)shared_data);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 3. CM4 부팅 대기
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  printf("\nPress USER button to boot CM4...\n");
  printf("Or wait 5 seconds for auto-boot.\n\n");

  uint32_t start = HAL_GetTick();
  uint8_t boot_cm4 = 0;

  while (!boot_cm4)
  {
    // USER 버튼 확인
    if (HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_RESET)
    {
      printf("USER button pressed.\n");
      boot_cm4 = 1;
    }

    // 타임아웃 (5초)
    if (HAL_GetTick() - start > 5000)
    {
      printf("Auto-boot timeout reached.\n");
      boot_cm4 = 1;
    }

    // LED1 토글 (M7 대기 중)
    HAL_GPIO_TogglePin(GPIOI, GPIO_PIN_12);
    HAL_Delay(200);
  }

  // LED1 ON (M7 준비 완료)
  HAL_GPIO_WritePin(GPIOI, GPIO_PIN_12, GPIO_PIN_SET);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 4. CM4 부팅
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  printf("Booting CM4...\n");

  Boot_CM4();

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 5. CM4 부팅 완료 대기
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  printf("Waiting for CM4 ready signal...\n");

  // HSEM으로 CM4 준비 대기
  while (HAL_HSEM_IsSemTaken(HSEM_CM4_READY))
  {
    HAL_Delay(10);
  }

  // 또는 공유 메모리 확인
  while (shared_data->cm4_status == 0)
  {
    HAL_Delay(10);
  }

  printf("CM4 is now running!\n\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 6. M7 메인 루프
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  uint32_t counter = 0;

  while (1)
  {
    // M7 작업 수행
    counter++;

    // 공유 데이터 업데이트
    HAL_HSEM_FastTake(HSEM_SHARED_DATA);
    shared_data->command = counter;
    HAL_HSEM_Release(HSEM_SHARED_DATA, 0);

    // 상태 출력
    printf("[CM7] Counter=%lu, CM4 Status=%lu\n",
           counter, shared_data->cm4_status);

    // LED1 토글
    HAL_GPIO_TogglePin(GPIOI, GPIO_PIN_12);

    HAL_Delay(1000);
  }
}
```

### 2. CM4 부팅 함수

```c
void Boot_CM4(void)
{
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 방법 1: RCC 레지스터 사용 (권장)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // CM4 클럭 활성화
  __HAL_RCC_CM4_CLK_ENABLE();

  // CM4 리셋 해제
  __HAL_RCC_CM4_RELEASE_RESET();

  // CM4 부팅 활성화
  HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);

  printf("CM4 boot enabled via RCC.\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 방법 2: SYSCFG 직접 제어 (대안)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // // CM4 부팅 주소 설정 (0x08100000)
  // __HAL_SYSCFG_CM4_BOOT_ADDR(0x08100000 >> 16);

  // // CM4 부팅 활성화
  // SYSCFG->UR1 |= SYSCFG_UR1_BCM4;
}
```

### 3. CM4 재시작 함수

```c
void Restart_CM4(void)
{
  printf("Restarting CM4...\n");

  // CM4 리셋
  __HAL_RCC_CM4_FORCE_RESET();
  HAL_Delay(10);
  __HAL_RCC_CM4_RELEASE_RESET();

  // 공유 데이터 리셋
  HAL_HSEM_FastTake(HSEM_SHARED_DATA);
  shared_data->cm4_status = 0;
  HAL_HSEM_Release(HSEM_SHARED_DATA, 0);

  // CM4 재부팅
  HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);

  printf("CM4 restart initiated.\n");
}
```

### 4. M4 코드 (슬레이브)

```c
// M4 Flash: 0x08100000 (Sector 8)
// M4는 M7이 부팅할 때까지 대기

#include "stm32h7xx_hal.h"

// 공유 메모리 (M7과 동일한 주소)
#define SHARED_MEMORY_BASE  0x30040000

typedef struct {
  uint32_t magic;
  uint32_t cm7_status;
  uint32_t cm4_status;
  uint32_t command;
  uint8_t data[256];
} SharedData_t;

volatile SharedData_t *shared_data = (SharedData_t*)SHARED_MEMORY_BASE;

int main(void)
{
  // HAL 초기화
  HAL_Init();

  // M4 클럭 설정 (240 MHz)
  SystemClock_Config_M4();

  // GPIO 초기화
  GPIO_Init_M4();

  // UART 초기화 (USART3)
  UART_Init_M4();

  printf("\n[CM4] Started.\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 1. HSEM 초기화
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  __HAL_RCC_HSEM_CLK_ENABLE();

  printf("[CM4] HSEM initialized.\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 2. 공유 메모리 확인
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  printf("[CM4] Checking shared memory...\n");

  // M7이 초기화했는지 확인
  if (shared_data->magic != 0xDEADBEEF)
  {
    printf("[CM4] ERROR: Shared memory not initialized!\n");
    printf("[CM4] Magic: 0x%08lX (expected 0xDEADBEEF)\n",
           shared_data->magic);
    Error_Handler();
  }

  printf("[CM4] Shared memory OK.\n");

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 3. M7에게 준비 완료 신호
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  // 공유 데이터 업데이트
  HAL_HSEM_FastTake(HSEM_SHARED_DATA);
  shared_data->cm4_status = 1;  // CM4 ready
  HAL_HSEM_Release(HSEM_SHARED_DATA, 0);

  // HSEM으로 신호
  HAL_HSEM_Release(HSEM_CM4_READY, 0);

  printf("[CM4] Ready signal sent to CM7.\n\n");

  // LED2 ON
  HAL_GPIO_WritePin(GPIOI, GPIO_PIN_13, GPIO_PIN_SET);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // 4. M4 메인 루프
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  uint32_t last_command = 0;

  while (1)
  {
    // M7로부터 명령 확인
    if (shared_data->command != last_command)
    {
      last_command = shared_data->command;

      printf("[CM4] Received command: %lu\n", last_command);

      // 명령 처리
      Process_Command(last_command);

      // 상태 업데이트
      HAL_HSEM_FastTake(HSEM_SHARED_DATA);
      shared_data->cm4_status = last_command;  // 에코
      HAL_HSEM_Release(HSEM_SHARED_DATA, 0);
    }

    // LED2 토글
    HAL_GPIO_TogglePin(GPIOI, GPIO_PIN_13);

    HAL_Delay(500);
  }
}

void Process_Command(uint32_t command)
{
  // 명령 처리 예제
  switch (command % 4)
  {
    case 0:
      // ADC 읽기
      break;
    case 1:
      // 센서 데이터 수집
      break;
    case 2:
      // 모터 제어
      break;
    case 3:
      // 통신 수행
      break;
  }
}
```

## 고급 기능

### 1. Hardware Semaphore (HSEM)

```c
// HSEM을 사용한 코어 간 동기화

#define HSEM_ID_CM7_MUTEX    0
#define HSEM_ID_CM4_MUTEX    1
#define HSEM_ID_SHARED_MEM   2
#define HSEM_ID_UART         3
#define HSEM_ID_SPI          4
#define HSEM_ID_I2C          5

// CM7: 세마포어 획득 (Blocking)
void CM7_Lock_Resource(uint32_t semaphore_id)
{
  // 세마포어 획득 시도
  while (HAL_HSEM_FastTake(semaphore_id) != HAL_OK)
  {
    // 획득 실패 - 대기
    __NOP();
  }
}

// CM7: 세마포어 해제
void CM7_Unlock_Resource(uint32_t semaphore_id)
{
  HAL_HSEM_Release(semaphore_id, 0);
}

// CM4: 세마포어 획득 (Blocking)
void CM4_Lock_Resource(uint32_t semaphore_id)
{
  while (HAL_HSEM_FastTake(semaphore_id) != HAL_OK)
  {
    __NOP();
  }
}

// CM4: 세마포어 해제
void CM4_Unlock_Resource(uint32_t semaphore_id)
{
  HAL_HSEM_Release(semaphore_id, 0);
}

// 세마포어 인터럽트 사용
void HSEM_Configure_Notification(void)
{
  // CM7: CM4가 세마포어 해제할 때 인터럽트
  HAL_HSEM_ActivateNotification(HSEM_ID_CM4_MUTEX);

  // HSEM 인터럽트 활성화
  HAL_NVIC_SetPriority(HSEM1_IRQn, 5, 0);
  HAL_NVIC_EnableIRQ(HSEM1_IRQn);
}

void HSEM1_IRQHandler(void)
{
  HAL_HSEM_IRQHandler();
}

void HAL_HSEM_FreeCallback(uint32_t SemMask)
{
  // CM4가 세마포어를 해제함
  if (SemMask & (1 << HSEM_ID_CM4_MUTEX))
  {
    printf("CM4 released semaphore %d\n", HSEM_ID_CM4_MUTEX);
  }
}
```

### 2. 메시지 큐 구현

```c
// 공유 메모리에 메시지 큐 구현

#define QUEUE_SIZE  16
#define MSG_SIZE    64

typedef struct {
  uint8_t data[MSG_SIZE];
  uint8_t length;
} Message_t;

typedef struct {
  Message_t buffer[QUEUE_SIZE];
  volatile uint32_t head;
  volatile uint32_t tail;
  volatile uint32_t count;
} MessageQueue_t;

// 공유 메모리에 배치
__attribute__((section(".shared_ram")))
MessageQueue_t cm7_to_cm4_queue;

__attribute__((section(".shared_ram")))
MessageQueue_t cm4_to_cm7_queue;

// CM7 → CM4 메시지 전송
HAL_StatusTypeDef Send_Message_To_CM4(uint8_t *data, uint8_t length)
{
  MessageQueue_t *queue = &cm7_to_cm4_queue;

  // 세마포어 획득
  CM7_Lock_Resource(HSEM_ID_SHARED_MEM);

  // 큐 가득 참 확인
  if (queue->count >= QUEUE_SIZE)
  {
    CM7_Unlock_Resource(HSEM_ID_SHARED_MEM);
    return HAL_BUSY;
  }

  // 메시지 추가
  memcpy(queue->buffer[queue->tail].data, data, length);
  queue->buffer[queue->tail].length = length;

  queue->tail = (queue->tail + 1) % QUEUE_SIZE;
  queue->count++;

  // 세마포어 해제
  CM7_Unlock_Resource(HSEM_ID_SHARED_MEM);

  // CM4에게 알림
  HAL_HSEM_Release(HSEM_ID_CM4_MUTEX, 0);

  return HAL_OK;
}

// CM4: 메시지 수신
HAL_StatusTypeDef Receive_Message_From_CM7(uint8_t *data, uint8_t *length)
{
  MessageQueue_t *queue = &cm7_to_cm4_queue;

  // 세마포어 획득
  CM4_Lock_Resource(HSEM_ID_SHARED_MEM);

  // 큐 비어 있음 확인
  if (queue->count == 0)
  {
    CM4_Unlock_Resource(HSEM_ID_SHARED_MEM);
    return HAL_ERROR;
  }

  // 메시지 읽기
  *length = queue->buffer[queue->head].length;
  memcpy(data, queue->buffer[queue->head].data, *length);

  queue->head = (queue->head + 1) % QUEUE_SIZE;
  queue->count--;

  // 세마포어 해제
  CM4_Unlock_Resource(HSEM_ID_SHARED_MEM);

  return HAL_OK;
}
```

### 3. 동적 코어 활성화/비활성화

```c
// 필요에 따라 CM4 활성화/비활성화

typedef enum {
  CORE_STATE_OFF = 0,
  CORE_STATE_BOOTING,
  CORE_STATE_RUNNING,
  CORE_STATE_STOPPING
} CoreState_t;

CoreState_t cm4_state = CORE_STATE_OFF;

// CM4 활성화
HAL_StatusTypeDef Activate_CM4(void)
{
  if (cm4_state != CORE_STATE_OFF)
  {
    printf("CM4 already running\n");
    return HAL_ERROR;
  }

  printf("Activating CM4...\n");

  cm4_state = CORE_STATE_BOOTING;

  // CM4 부팅
  Boot_CM4();

  // 부팅 완료 대기
  uint32_t timeout = HAL_GetTick() + 5000;

  while (shared_data->cm4_status == 0)
  {
    if (HAL_GetTick() > timeout)
    {
      printf("CM4 boot timeout!\n");
      cm4_state = CORE_STATE_OFF;
      return HAL_TIMEOUT;
    }
    HAL_Delay(10);
  }

  cm4_state = CORE_STATE_RUNNING;
  printf("CM4 activated successfully.\n");

  return HAL_OK;
}

// CM4 비활성화
HAL_StatusTypeDef Deactivate_CM4(void)
{
  if (cm4_state != CORE_STATE_RUNNING)
  {
    printf("CM4 not running\n");
    return HAL_ERROR;
  }

  printf("Deactivating CM4...\n");

  cm4_state = CORE_STATE_STOPPING;

  // CM4에게 종료 명령
  HAL_HSEM_FastTake(HSEM_SHARED_DATA);
  shared_data->command = 0xFFFFFFFF;  // Shutdown command
  HAL_HSEM_Release(HSEM_SHARED_DATA, 0);

  // CM4 종료 대기
  HAL_Delay(100);

  // CM4 리셋
  __HAL_RCC_CM4_FORCE_RESET();

  // 상태 리셋
  HAL_HSEM_FastTake(HSEM_SHARED_DATA);
  shared_data->cm4_status = 0;
  HAL_HSEM_Release(HSEM_SHARED_DATA, 0);

  cm4_state = CORE_STATE_OFF;
  printf("CM4 deactivated.\n");

  return HAL_OK;
}

// 작업 부하에 따른 동적 제어
void Dynamic_Core_Control(void)
{
  static uint32_t workload_counter = 0;

  // 작업 부하 측정 (예시)
  uint32_t workload = Get_System_Workload();

  if (workload > 80 && cm4_state == CORE_STATE_OFF)
  {
    // 높은 부하 - CM4 활성화
    printf("High workload detected (%lu%%)\n", workload);
    Activate_CM4();
  }
  else if (workload < 20 && cm4_state == CORE_STATE_RUNNING)
  {
    // 낮은 부하 - CM4 비활성화
    workload_counter++;

    if (workload_counter > 100)  // 10초간 낮은 부하
    {
      printf("Low workload sustained (%lu%%)\n", workload);
      Deactivate_CM4();
      workload_counter = 0;
    }
  }
  else
  {
    workload_counter = 0;
  }
}
```

### 4. Option Bytes 프로그래밍

```c
// Option Bytes를 통한 부팅 설정

void Configure_Boot_Options(uint8_t bcm4_enable)
{
  FLASH_OBProgramInitTypeDef OBInit;

  printf("Configuring boot options...\n");

  // 현재 설정 읽기
  HAL_FLASHEx_OBGetConfig(&OBInit);

  printf("Current BCM4: %s\n",
         (OBInit.USERConfig & FLASH_OPTSR_BCM4) ? "Enabled" : "Disabled");

  // 변경 필요 확인
  uint8_t current_bcm4 = (OBInit.USERConfig & FLASH_OPTSR_BCM4) ? 1 : 0;

  if (current_bcm4 == bcm4_enable)
  {
    printf("Boot options already configured.\n");
    return;
  }

  // Flash 잠금 해제
  HAL_FLASH_Unlock();
  HAL_FLASH_OB_Unlock();

  // BCM4 설정
  OBInit.OptionType = OPTIONBYTE_USER;
  OBInit.USERType = OB_USER_BCM4;

  if (bcm4_enable)
  {
    OBInit.USERConfig |= FLASH_OPTSR_BCM4;   // CM4 auto boot
    printf("Enabling CM4 auto boot...\n");
  }
  else
  {
    OBInit.USERConfig &= ~FLASH_OPTSR_BCM4;  // CM4 held
    printf("Disabling CM4 auto boot...\n");
  }

  // Option Bytes 프로그래밍
  if (HAL_FLASHEx_OBProgram(&OBInit) != HAL_OK)
  {
    printf("ERROR: Option Bytes programming failed!\n");
    HAL_FLASH_OB_Lock();
    HAL_FLASH_Lock();
    return;
  }

  // 변경사항 적용 (시스템 리셋 발생)
  printf("Launching Option Bytes (system will reset)...\n");
  HAL_FLASH_OB_Launch();

  // 여기는 실행되지 않음
  HAL_FLASH_OB_Lock();
  HAL_FLASH_Lock();
}
```

## 전력 소비 분석

### 동적 코어 관리의 전력 절감

```
┌─────────────────────────────────────────────────────────┐
│            Dynamic Core Power Management                 │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  시나리오: 가변 작업 부하                                │
│                                                          │
│  시간   0s     5s     10s    15s    20s    25s          │
│        │      │      │      │      │      │            │
│  부하:  30%    80%    90%    50%    20%    10%          │
│                                                          │
│  CM7:   ████████████████████████████████████  (항상 ON) │
│  CM4:          ████████████████                (동적)   │
│                │      │      │                          │
│                │      │      │                          │
│  전력:   170mA  250mA  250mA  250mA  170mA  170mA       │
│                                                          │
│  평균 전력: (170×5 + 250×15 + 170×5) / 25 = 218 mA     │
│                                                          │
│  비교 (양쪽 코어 항상 ON): 250 mA                        │
│  절감: 32 mA (12.8%)                                    │
│                                                          │
│  배터리 수명 (2000 mAh):                                 │
│    • 항상 양쪽 ON:    8.0 시간                          │
│    • 동적 관리:       9.2 시간 (+15%)                   │
└─────────────────────────────────────────────────────────┘
```

### 배터리 수명 계산

```c
void Calculate_Battery_Life(void)
{
  printf("\n=== Battery Life Calculation ===\n\n");

  // 시나리오 1: 항상 양쪽 코어 ON
  float power1 = 250.0f;  // mA
  float life1 = 2000.0f / power1;  // hours

  printf("1. Both cores always ON:\n");
  printf("   Power: %.0f mA\n", power1);
  printf("   Life:  %.1f hours\n\n", life1);

  // 시나리오 2: 동적 코어 관리
  // M7 항상 ON: 100%
  // M4 ON: 30% 시간
  float duty_cycle = 0.30f;
  float power2 = 170.0f + (80.0f * duty_cycle);  // mA
  float life2 = 2000.0f / power2;  // hours

  printf("2. Dynamic core management (30%% duty):\n");
  printf("   Power: %.0f mA\n", power2);
  printf("   Life:  %.1f hours\n", life2);
  printf("   Improvement: +%.0f%%\n\n", (life2/life1 - 1) * 100);

  // 시나리오 3: M4 10% 사용
  duty_cycle = 0.10f;
  float power3 = 170.0f + (80.0f * duty_cycle);
  float life3 = 2000.0f / power3;

  printf("3. Low M4 usage (10%% duty):\n");
  printf("   Power: %.0f mA\n", power3);
  printf("   Life:  %.1f hours\n", life3);
  printf("   Improvement: +%.0f%%\n\n", (life3/life1 - 1) * 100);

  printf("================================\n\n");
}
```

## 실전 응용 예제

### 예제: 에너지 하베스팅 시스템

```c
// 태양광 에너지 가용성에 따라 CM4 동적 활성화

#define SOLAR_THRESHOLD_HIGH  3000  // mV - CM4 활성화
#define SOLAR_THRESHOLD_LOW   2500  // mV - CM4 비활성화

void Energy_Harvesting_System(void)
{
  printf("Energy Harvesting System\n");

  // ADC3로 태양광 패널 전압 모니터링
  ADC_Init();

  while (1)
  {
    // 태양광 전압 측정
    uint32_t solar_mv = Read_Solar_Voltage();

    printf("Solar voltage: %lu mV\n", solar_mv);

    if (solar_mv > SOLAR_THRESHOLD_HIGH)
    {
      // 충분한 전력 - CM4 활성화
      if (cm4_state == CORE_STATE_OFF)
      {
        printf("Sufficient solar power - activating CM4\n");
        Activate_CM4();

        // CM4에게 집중 작업 할당
        Send_Message_To_CM4("START_INTENSIVE_TASK", 20);
      }
    }
    else if (solar_mv < SOLAR_THRESHOLD_LOW)
    {
      // 전력 부족 - CM4 비활성화
      if (cm4_state == CORE_STATE_RUNNING)
      {
        printf("Low solar power - deactivating CM4\n");

        // CM4에게 종료 요청
        Send_Message_To_CM4("SHUTDOWN", 8);

        HAL_Delay(100);
        Deactivate_CM4();
      }
    }

    // M7만으로 필수 작업 수행
    Perform_Essential_Tasks();

    HAL_Delay(1000);
  }
}
```

## 트러블슈팅

### 문제 1: CM4가 부팅되지 않음

```c
void Debug_CM4_Boot_Issue(void)
{
  printf("=== CM4 Boot Debug ===\n");

  // 1. Option Bytes 확인
  FLASH_OBProgramInitTypeDef OBInit;
  HAL_FLASHEx_OBGetConfig(&OBInit);

  printf("BCM4 bit: %s\n",
         (OBInit.USERConfig & FLASH_OPTSR_BCM4) ? "1 (Auto)" : "0 (Held)");

  // 2. 부팅 주소 확인
  uint32_t boot_addr = (SYSCFG->UR2 & SYSCFG_UR2_BOOT_ADD0) << 16;
  printf("CM4 boot address: 0x%08lX\n", boot_addr);

  // 3. M4 Vector Table 확인
  uint32_t *m4_vt = (uint32_t*)boot_addr;
  printf("CM4 SP: 0x%08lX\n", m4_vt[0]);
  printf("CM4 Reset: 0x%08lX\n", m4_vt[1]);

  if (m4_vt[0] == 0xFFFFFFFF || m4_vt[1] == 0xFFFFFFFF)
  {
    printf("ERROR: CM4 Flash appears empty!\n");
    printf("Flash CM4 firmware to 0x08100000\n");
  }

  // 4. RCC 상태 확인
  printf("RCC_GCR: 0x%08lX\n", RCC->GCR);

  if (RCC->GCR & RCC_GCR_CM4_RSTN)
  {
    printf("CM4 is held in reset\n");
  }

  printf("=======================\n\n");
}
```

### 문제 2: 코어 간 통신 실패

```c
void Debug_InterCore_Communication(void)
{
  printf("=== Inter-Core Communication Debug ===\n");

  // 1. HSEM 상태 확인
  printf("HSEM status:\n");
  for (int i = 0; i < 6; i++)
  {
    printf("  HSEM[%d]: %s (owner=%lu)\n",
           i,
           HAL_HSEM_IsSemTaken(i) ? "TAKEN" : "FREE",
           (HSEM->RLR[i] & HSEM_RLR_PROCID) >> HSEM_RLR_PROCID_Pos);
  }

  // 2. 공유 메모리 확인
  printf("\nShared memory:\n");
  printf("  Magic: 0x%08lX (expected 0xDEADBEEF)\n", shared_data->magic);
  printf("  CM7 status: %lu\n", shared_data->cm7_status);
  printf("  CM4 status: %lu\n", shared_data->cm4_status);
  printf("  Command: %lu\n", shared_data->command);

  // 3. 데드락 감지
  static uint32_t last_status = 0;
  static uint32_t deadlock_counter = 0;

  if (shared_data->cm4_status == last_status)
  {
    deadlock_counter++;
    if (deadlock_counter > 100)
    {
      printf("WARNING: Possible deadlock detected!\n");
    }
  }
  else
  {
    last_status = shared_data->cm4_status;
    deadlock_counter = 0;
  }

  printf("=====================================\n\n");
}
```

### 문제 3: 세마포어 데드락

```c
// 데드락 방지를 위한 타임아웃 세마포어

HAL_StatusTypeDef HSEM_Take_Timeout(uint32_t sem_id, uint32_t timeout_ms)
{
  uint32_t start = HAL_GetTick();

  while (HAL_HSEM_FastTake(sem_id) != HAL_OK)
  {
    if (HAL_GetTick() - start > timeout_ms)
    {
      printf("ERROR: HSEM[%lu] timeout!\n", sem_id);
      return HAL_TIMEOUT;
    }
    __NOP();
  }

  return HAL_OK;
}

// 사용 예
void Safe_Resource_Access(void)
{
  if (HSEM_Take_Timeout(HSEM_ID_SHARED_MEM, 100) != HAL_OK)
  {
    // 타임아웃 - 오류 처리
    printf("Failed to acquire semaphore\n");
    return;
  }

  // 리소스 접근
  // ...

  HAL_HSEM_Release(HSEM_ID_SHARED_MEM, 0);
}
```

## 참고 자료

- **RM0399**: STM32H745 Reference Manual
  - Chapter 5: Boot configuration
  - Chapter 10: Hardware semaphore (HSEM)
  - Chapter 7: RCC for dual-core

- **AN5361**: Getting started with dual-core STM32H7

- **AN5557**: Inter-processor communication in STM32H7 series

- **STM32H7 Dual-Core Workshop**: ST 공식 교육 자료

## 관련 예제

- **HSEM_DualProcess**: 하드웨어 세마포어 사용
- **PWR_D1ON_D2OFF**: D1만 활성화
- **PWR_D2ON_D1OFF**: D2만 활성화
- **DualCore_Messaging**: 코어 간 메시지 전달
