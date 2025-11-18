# GPIO_EXTI - 외부 인터럽트를 이용한 듀얼 코어 GPIO 제어

## 개요

이 예제는 STM32H745의 듀얼 코어(Cortex-M7과 Cortex-M4) 환경에서 외부 인터럽트(EXTI)를 사용하여 각 코어가 독립적으로 GPIO를 제어하는 방법을 보여줍니다. Wakeup 버튼을 누르면 양쪽 코어에서 인터럽트가 발생하여 각각 다른 LED를 토글합니다.

## 하드웨어 요구사항

- **STM32H745I-DISCO 보드**
- **Wakeup 버튼**: PC13 (내장)
- **LED1 (주황색)**: PI12 - CM7 코어 제어
- **LED2 (초록색)**: PI13 - CM4 코어 제어

## 주요 기능

### 듀얼 코어 EXTI 처리
- **CM7 코어**: Wakeup 버튼 인터럽트로 LED1 제어
- **CM4 코어**: 동일한 인터럽트로 LED2 제어
- **독립적 인터럽트 핸들러**: 각 코어가 별도의 ISR 실행
- **하드웨어 리소스 공유**: 양쪽 코어가 동일한 EXTI 라인 사용

### EXTI (External Interrupt) 기능
- **라인 13**: PC13 핀에 연결
- **트리거**: 하강 엣지 (버튼 누름)
- **인터럽트 우선순위**: 양쪽 코어에서 개별 설정 가능
- **디바운싱**: 소프트웨어 타이머 기반

## 동작 원리

### 듀얼 코어 인터럽트 처리 흐름
```
                    ┌─────────────┐
                    │  PC13 핀    │
                    │ (Wakeup BTN)│
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │  EXTI Line13│
                    └──────┬──────┘
                           │
                ┌──────────┴──────────┐
                │                     │
                ▼                     ▼
         ┌────────────┐        ┌────────────┐
         │ NVIC (CM7) │        │ NVIC (CM4) │
         └──────┬─────┘        └──────┬─────┘
                │                     │
                ▼                     ▼
         ┌────────────┐        ┌────────────┐
         │EXTI15_10   │        │EXTI15_10   │
         │  IRQn      │        │  IRQn      │
         │  (CM7)     │        │  (CM4)     │
         └──────┬─────┘        └──────┬─────┘
                │                     │
                ▼                     ▼
         ┌────────────┐        ┌────────────┐
         │ Toggle LED1│        │ Toggle LED2│
         │   (PI12)   │        │   (PI13)   │
         └────────────┘        └────────────┘
```

### 인터럽트 메커니즘
1. **버튼 누름**: PC13 핀이 LOW로 전환 (하강 엣지)
2. **EXTI 활성화**: EXTI 라인 13에서 이벤트 감지
3. **듀얼 인터럽트 발생**:
   - CM7의 NVIC에 인터럽트 전달
   - CM4의 NVIC에 동시에 인터럽트 전달
4. **각 코어에서 ISR 실행**:
   - CM7: LED1 토글
   - CM4: LED2 토글
5. **인터럽트 플래그 클리어**: 각 코어에서 독립적으로 처리

## 코드 구조

### 1. GPIO 초기화 (CM7)

```c
// CM7/Core/Src/main.c

void GPIO_Init_CM7(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  // GPIO 클럭 활성화
  __HAL_RCC_GPIOI_CLK_ENABLE();  // LED
  __HAL_RCC_GPIOC_CLK_ENABLE();  // Button

  // LED1 설정 (PI12)
  GPIO_InitStruct.Pin = GPIO_PIN_12;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;  // Push-Pull 출력
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  HAL_GPIO_Init(GPIOI, &GPIO_InitStruct);

  // 초기 상태: LED OFF
  HAL_GPIO_WritePin(GPIOI, GPIO_PIN_12, GPIO_PIN_RESET);
}
```

### 2. EXTI 설정 (CM7)

```c
void EXTI_Config_CM7(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  // Wakeup 버튼 설정 (PC13)
  GPIO_InitStruct.Pin = GPIO_PIN_13;
  GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING;  // 하강 엣지 인터럽트
  GPIO_InitStruct.Pull = GPIO_NOPULL;           // 외부 풀업 존재
  HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);

  // EXTI 인터럽트 우선순위 설정
  HAL_NVIC_SetPriority(EXTI15_10_IRQn, 2, 0);

  // EXTI 인터럽트 활성화
  HAL_NVIC_EnableIRQ(EXTI15_10_IRQn);
}
```

### 3. 인터럽트 핸들러 (CM7)

```c
// CM7/Core/Src/stm32h7xx_it.c

// 디바운싱을 위한 변수
static uint32_t last_interrupt_time = 0;
#define DEBOUNCE_TIME_MS  200  // 200ms 디바운싱

void EXTI15_10_IRQHandler(void)
{
  // EXTI 라인 13 인터럽트 확인
  if(__HAL_GPIO_EXTI_GET_IT(GPIO_PIN_13) != RESET)
  {
    uint32_t current_time = HAL_GetTick();

    // 디바운싱 체크
    if ((current_time - last_interrupt_time) > DEBOUNCE_TIME_MS)
    {
      // LED1 토글
      HAL_GPIO_TogglePin(GPIOI, GPIO_PIN_12);

      // 타임스탬프 업데이트
      last_interrupt_time = current_time;
    }

    // 인터럽트 플래그 클리어
    __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_13);
  }
}
```

### 4. GPIO 초기화 (CM4)

```c
// CM4/Core/Src/main.c

void GPIO_Init_CM4(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  // GPIO 클럭 활성화
  __HAL_RCC_GPIOI_CLK_ENABLE();  // LED
  __HAL_RCC_GPIOC_CLK_ENABLE();  // Button

  // LED2 설정 (PI13)
  GPIO_InitStruct.Pin = GPIO_PIN_13;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  HAL_GPIO_Init(GPIOI, &GPIO_InitStruct);

  // 초기 상태: LED OFF
  HAL_GPIO_WritePin(GPIOI, GPIO_PIN_13, GPIO_PIN_RESET);
}
```

### 5. EXTI 설정 (CM4)

```c
void EXTI_Config_CM4(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  // Wakeup 버튼 설정 (PC13) - CM7과 동일
  GPIO_InitStruct.Pin = GPIO_PIN_13;
  GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);

  // EXTI 인터럽트 우선순위 설정 (CM4 NVIC)
  HAL_NVIC_SetPriority(EXTI15_10_IRQn, 2, 0);

  // EXTI 인터럽트 활성화
  HAL_NVIC_EnableIRQ(EXTI15_10_IRQn);
}
```

### 6. 인터럽트 핸들러 (CM4)

```c
// CM4/Core/Src/stm32h7xx_it.c

static uint32_t last_interrupt_time = 0;
#define DEBOUNCE_TIME_MS  200

void EXTI15_10_IRQHandler(void)
{
  if(__HAL_GPIO_EXTI_GET_IT(GPIO_PIN_13) != RESET)
  {
    uint32_t current_time = HAL_GetTick();

    // 디바운싱 체크
    if ((current_time - last_interrupt_time) > DEBOUNCE_TIME_MS)
    {
      // LED2 토글 (CM4 담당)
      HAL_GPIO_TogglePin(GPIOI, GPIO_PIN_13);

      last_interrupt_time = current_time;
    }

    // 인터럽트 플래그 클리어
    __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_13);
  }
}
```

### 7. 메인 함수 (CM7)

```c
// CM7/Core/Src/main.c

int main(void)
{
  // HAL 라이브러리 초기화
  HAL_Init();

  // 시스템 클럭 설정 (400MHz)
  SystemClock_Config();

  // GPIO 초기화
  GPIO_Init_CM7();

  // EXTI 설정
  EXTI_Config_CM7();

  // CM4 부팅 (HSEM 사용)
  __HAL_RCC_HSEM_CLK_ENABLE();
  HAL_HSEM_FastTake(HSEM_ID_0);
  HAL_HSEM_Release(HSEM_ID_0, 0);

  // CM4 부팅
  HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);

  // 무한 루프
  while (1)
  {
    // 인터럽트 대기 (저전력 모드 가능)
    __WFI();  // Wait For Interrupt
  }
}
```

### 8. 메인 함수 (CM4)

```c
// CM4/Core/Src/main.c

int main(void)
{
  // HAL 라이브러리 초기화
  HAL_Init();

  // GPIO 초기화
  GPIO_Init_CM4();

  // EXTI 설정
  EXTI_Config_CM4();

  // 무한 루프
  while (1)
  {
    // 인터럽트 대기
    __WFI();
  }
}
```

## 메모리 구성

### 듀얼 코어 메모리 맵
```
Flash Memory (2MB)
0x08000000  ┌─────────────────────────┐
            │                         │
            │  CM7 Code & Data        │
            │  (Sector 0-7)           │
            │  1MB                    │
            │                         │
0x08100000  ├─────────────────────────┤
            │                         │
            │  CM4 Code & Data        │
            │  (Sector 8-15)          │
            │  1MB                    │
            │                         │
0x08200000  └─────────────────────────┘

RAM Memory
0x20000000  ┌─────────────────────────┐
            │  CM7 DTCM RAM           │
            │  128KB                  │
0x20020000  ├─────────────────────────┤
            │                         │
            │  AXI SRAM (Shared)      │
            │  512KB                  │
            │                         │
0x24000000  ├─────────────────────────┤
            │  CM4 AHB SRAM           │
            │  288KB                  │
0x30000000  └─────────────────────────┘
```

### EXTI 레지스터 공유
양쪽 코어가 동일한 EXTI 레지스터를 공유하지만, 각각 독립적인 NVIC를 통해 인터럽트를 처리합니다.

## 고급 기능

### 1. 인터럽트 우선순위 관리

```c
// CM7에서 높은 우선순위 설정
HAL_NVIC_SetPriority(EXTI15_10_IRQn, 1, 0);  // Preempt: 1, Sub: 0

// CM4에서 낮은 우선순위 설정
HAL_NVIC_SetPriority(EXTI15_10_IRQn, 3, 0);  // Preempt: 3, Sub: 0
```

### 2. 여러 EXTI 라인 처리

```c
void EXTI_Multi_Config(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  // PC13: 하강 엣지
  GPIO_InitStruct.Pin = GPIO_PIN_13;
  GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);

  // PJ0: 상승 엣지
  GPIO_InitStruct.Pin = GPIO_PIN_0;
  GPIO_InitStruct.Mode = GPIO_MODE_IT_RISING;
  GPIO_InitStruct.Pull = GPIO_PULLDOWN;
  HAL_GPIO_Init(GPIOJ, &GPIO_InitStruct);

  // 인터럽트 활성화
  HAL_NVIC_EnableIRQ(EXTI15_10_IRQn);  // PC13
  HAL_NVIC_EnableIRQ(EXTI0_IRQn);      // PJ0
}

// 개별 핸들러
void EXTI0_IRQHandler(void)
{
  if(__HAL_GPIO_EXTI_GET_IT(GPIO_PIN_0) != RESET)
  {
    // PJ0 처리
    __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_0);
  }
}
```

### 3. 코어 간 동기화 (HSEM 사용)

```c
// LED를 토글하기 전에 세마포어 획득
void SafeLED_Toggle(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin, uint32_t sem_id)
{
  // 세마포어 획득 (타임아웃 1000ms)
  if (HAL_HSEM_FastTake(sem_id) == HAL_OK)
  {
    // 크리티컬 섹션
    HAL_GPIO_TogglePin(GPIOx, GPIO_Pin);
    HAL_Delay(10);  // LED 시각적 효과

    // 세마포어 해제
    HAL_HSEM_Release(sem_id, 0);
  }
}

// CM7 인터럽트에서 사용
void EXTI15_10_IRQHandler(void)
{
  if(__HAL_GPIO_EXTI_GET_IT(GPIO_PIN_13) != RESET)
  {
    SafeLED_Toggle(GPIOI, GPIO_PIN_12, HSEM_ID_1);
    __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_13);
  }
}
```

### 4. 하드웨어 디바운싱 (EXTI 필터)

```c
// EXTI 라인에 디지털 필터 적용 (펌웨어 구현)
#define FILTER_SAMPLES  5
static uint8_t filter_buffer[FILTER_SAMPLES] = {0};
static uint8_t filter_index = 0;

uint8_t EXTI_DebounceFilter(void)
{
  // 현재 핀 상태 읽기
  filter_buffer[filter_index] = HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13);
  filter_index = (filter_index + 1) % FILTER_SAMPLES;

  // 모든 샘플이 동일한지 확인
  uint8_t first = filter_buffer[0];
  for (int i = 1; i < FILTER_SAMPLES; i++)
  {
    if (filter_buffer[i] != first)
      return 0;  // 불안정
  }

  return 1;  // 안정
}
```

## 빌드 및 실행

### 빌드 방법

#### STM32CubeIDE 사용
```bash
# CM7 프로젝트 빌드
1. File -> Import -> Existing Projects into Workspace
2. Browse to: Examples/GPIO/GPIO_EXTI/CM7
3. Build Project (Ctrl+B)

# CM4 프로젝트 빌드
1. File -> Import -> Existing Projects into Workspace
2. Browse to: Examples/GPIO/GPIO_EXTI/CM4
3. Build Project (Ctrl+B)
```

#### 명령줄 빌드
```bash
# CM7 빌드
cd CM7
make clean
make -j8

# CM4 빌드
cd ../CM4
make clean
make -j8
```

### 플래싱

#### OpenOCD 사용
```bash
# CM7 플래시
openocd -f board/stm32h745i-disco.cfg \
        -c "program build/CM7.elf verify reset exit"

# CM4 플래시
openocd -f board/stm32h745i-disco.cfg \
        -c "program build/CM4.elf verify reset exit"
```

#### ST-LINK Utility 사용
1. CM7 바이너리를 0x08000000에 플래시
2. CM4 바이너리를 0x08100000에 플래시

### 실행 및 테스트

1. **보드 리셋**: 리셋 버튼 누름
2. **CM7 부팅**: CM7이 먼저 부팅되고 CM4를 깨움
3. **버튼 테스트**:
   - Wakeup 버튼 누름
   - LED1과 LED2가 동시에 토글됨을 확인
4. **개별 동작 확인**:
   - LED1: 주황색 (CM7 제어)
   - LED2: 초록색 (CM4 제어)

## 디버깅

### 듀얼 코어 디버깅 (STM32CubeIDE)

```bash
# 디버그 설정
1. Run -> Debug Configurations
2. STM32 Cortex-M C/C++ Application 생성

# CM7 디버그
- Name: GPIO_EXTI_CM7
- C/C++ Application: CM7/build/CM7.elf
- Debug probe: ST-LINK

# CM4 디버그
- Name: GPIO_EXTI_CM4
- C/C++ Application: CM4/build/CM4.elf
- Debug probe: ST-LINK
- Core: Cortex-M4

# 동시 디버깅
1. CM7 디버그 시작
2. 새 디버그 세션에서 CM4 시작
3. 두 세션을 동시에 관찰
```

### 중단점 설정

```c
// CM7 인터럽트에 중단점
void EXTI15_10_IRQHandler(void)
{
  if(__HAL_GPIO_EXTI_GET_IT(GPIO_PIN_13) != RESET)
  {
    HAL_GPIO_TogglePin(GPIOI, GPIO_PIN_12);  // <- 중단점
    __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_13);
  }
}
```

## 트러블슈팅

### LED가 토글되지 않는 경우

#### 1. GPIO 클럭 확인
```c
// 클럭이 활성화되었는지 확인
if (!(RCC->AHB4ENR & RCC_AHB4ENR_GPIOIEN))
{
  // GPIOI 클럭이 비활성화됨
  __HAL_RCC_GPIOI_CLK_ENABLE();
}
```

#### 2. 핀 설정 확인
```c
// GPIO 모드 레지스터 확인
uint32_t moder = GPIOI->MODER;
printf("GPIOI->MODER = 0x%08lX\n", moder);
// PI12는 출력 모드(01)여야 함
```

#### 3. 인터럽트 활성화 확인
```c
// NVIC에서 인터럽트가 활성화되었는지 확인
if (NVIC_GetEnableIRQ(EXTI15_10_IRQn))
{
  printf("EXTI15_10 interrupt enabled\n");
}
else
{
  printf("EXTI15_10 interrupt disabled!\n");
  HAL_NVIC_EnableIRQ(EXTI15_10_IRQn);
}
```

### 인터럽트가 발생하지 않는 경우

#### 1. EXTI 설정 확인
```c
// EXTI 레지스터 확인
printf("EXTI->IMR1 = 0x%08lX\n", EXTI->IMR1);     // 마스크
printf("EXTI->RTSR1 = 0x%08lX\n", EXTI->RTSR1);   // 상승 엣지
printf("EXTI->FTSR1 = 0x%08lX\n", EXTI->FTSR1);   // 하강 엣지

// EXTI 라인 13이 활성화되어야 함
if (!(EXTI->IMR1 & (1 << 13)))
{
  printf("EXTI Line 13 masked!\n");
}
```

#### 2. SYSCFG 매핑 확인
```c
// PC13이 EXTI 라인 13에 연결되었는지 확인
// SYSCFG_EXTICR4[13] = 0010 (GPIOC)
uint32_t exticr = SYSCFG->EXTICR[3];
uint32_t exti13_cfg = (exticr >> 4) & 0xF;
if (exti13_cfg != 2)  // 2 = GPIOC
{
  printf("EXTI13 not mapped to PC13!\n");
}
```

### CM4가 부팅되지 않는 경우

#### 1. 부팅 플래그 확인
```c
// CM7에서 CM4 부팅 상태 확인
if (!(RCC->GCR & RCC_GCR_BOOT_C2))
{
  printf("CM4 boot not enabled\n");
  HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);
}
```

#### 2. CM4 리셋 벡터 확인
```c
// CM4 바이너리가 올바른 위치에 있는지 확인
uint32_t cm4_stack = *(uint32_t*)0x08100000;
uint32_t cm4_reset = *(uint32_t*)0x08100004;

printf("CM4 Stack: 0x%08lX\n", cm4_stack);
printf("CM4 Reset: 0x%08lX\n", cm4_reset);

// Stack은 RAM 영역, Reset은 Flash 영역이어야 함
```

### 디바운싱 문제

#### 채터링으로 여러 번 토글되는 경우
```c
// 디바운싱 시간 증가
#define DEBOUNCE_TIME_MS  500  // 200ms -> 500ms

// 또는 하드웨어 RC 필터 추가
// Button --- R(10K) --- MCU Pin
//              |
//              C(100nF)
//              |
//             GND
```

## 성능 측정

### 인터럽트 응답 시간 측정

```c
// GPIO를 사용한 응답 시간 측정
void EXTI15_10_IRQHandler(void)
{
  if(__HAL_GPIO_EXTI_GET_IT(GPIO_PIN_13) != RESET)
  {
    // 측정 시작 신호 (예: PH0를 HIGH)
    HAL_GPIO_WritePin(GPIOH, GPIO_PIN_0, GPIO_PIN_SET);

    // LED 토글
    HAL_GPIO_TogglePin(GPIOI, GPIO_PIN_12);

    // 측정 종료 신호
    HAL_GPIO_WritePin(GPIOH, GPIO_PIN_0, GPIO_PIN_RESET);

    __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_13);
  }
}

// 오실로스코프로 PH0 펄스 폭 측정
// 일반적으로 1-2 μs (400MHz CM7)
```

### 코어 간 동기화 오버헤드

```c
// HSEM 획득/해제 시간 측정
uint32_t start = DWT->CYCCNT;  // 사이클 카운터
HAL_HSEM_FastTake(HSEM_ID_1);
uint32_t take_cycles = DWT->CYCCNT - start;

start = DWT->CYCCNT;
HAL_HSEM_Release(HSEM_ID_1, 0);
uint32_t release_cycles = DWT->CYCCNT - start;

printf("HSEM Take: %lu cycles\n", take_cycles);
printf("HSEM Release: %lu cycles\n", release_cycles);
```

## 확장 아이디어

### 1. 코어 간 메시지 전달

```c
// 공유 메모리 사용
#define SHARED_MSG_ADDR  0x30000000  // AHB SRAM

typedef struct {
  uint8_t from_core;
  uint8_t to_core;
  uint16_t msg_id;
  uint32_t data;
} Message_t;

// CM7에서 CM4로 메시지 전송
void SendMessage_CM7_to_CM4(uint16_t msg_id, uint32_t data)
{
  Message_t *msg = (Message_t*)SHARED_MSG_ADDR;

  HAL_HSEM_FastTake(HSEM_ID_2);
  msg->from_core = 7;
  msg->to_core = 4;
  msg->msg_id = msg_id;
  msg->data = data;
  HAL_HSEM_Release(HSEM_ID_2, 0);

  // CM4에 인터럽트 전송 (HSEM 또는 SW 인터럽트)
}
```

### 2. 멀티 LED 패턴

```c
// LED 패턴 배열
typedef struct {
  uint16_t on_time;
  uint16_t off_time;
  uint8_t repeat;
} LED_Pattern_t;

LED_Pattern_t patterns[] = {
  {100, 100, 5},   // 빠른 깜박임 5회
  {500, 500, 3},   // 느린 깜박임 3회
  {50, 50, 10},    // 매우 빠른 깜박임 10회
};

void PlayLED_Pattern(LED_Pattern_t *pattern)
{
  for (uint8_t i = 0; i < pattern->repeat; i++)
  {
    HAL_GPIO_WritePin(GPIOI, GPIO_PIN_12, GPIO_PIN_SET);
    HAL_Delay(pattern->on_time);
    HAL_GPIO_WritePin(GPIOI, GPIO_PIN_12, GPIO_PIN_RESET);
    HAL_Delay(pattern->off_time);
  }
}
```

## 참고 자료

- **RM0399**: STM32H745 Reference Manual
  - Chapter 20: General-purpose I/Os (GPIO)
  - Chapter 21: External interrupt/event controller (EXTI)
  - Chapter 6: Nested vectored interrupt controller (NVIC)
- **AN5361**: Getting started with projects based on dual-core STM32H7 microcontrollers in STM32CubeIDE
- **AN4838**: Managing memory protection unit in STM32 MCUs
- **예제 코드**: `STM32Cube_FW_H7_V1.xx.x/Projects/STM32H745I-DISCO/Examples/GPIO/GPIO_EXTI`

## 관련 예제

- **HSEM**: 코어 간 동기화 상세
- **OpenAMP**: 고급 듀얼 코어 통신
- **GPIO_IOToggle**: 기본 GPIO 출력
- **EXTI_WakeUp**: 저전력 모드에서 깨우기
