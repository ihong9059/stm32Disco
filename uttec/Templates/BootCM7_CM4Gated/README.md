# BootCM7_CM4Gated - CM7 부팅, CM4 게이트 템플릿

## 개요

이 템플릿은 **Cortex-M7만 자동으로 부팅**하고, Cortex-M4는 클럭이 게이트(중지)된 상태로 시작합니다. CM7이 필요한 시점에 CM4를 활성화할 수 있습니다.

## 부팅 구성

### User Option Bytes
- **BCM4 = 0**: CM4 부팅 비활성화 (클럭 게이트)
- **BCM7 = 1**: CM7 부팅 활성화

## 동작 방식

### 부팅 시퀀스
```
┌─────────────┐         ┌─────────────┐
│   CM7 Core  │         │   CM4 Core  │
│   (400MHz)  │         │  (Gated)    │
└──────┬──────┘         └──────┬──────┘
       │                       │
       │ Boots immediately     │ Clock OFF
       │                       │ (Not running)
       ▼                       ×
   Running                 Stopped
       │
       │ When ready...
       │
       │ Enable CM4 clock ────►│
       │                       │
       │                       ▼
       │                   Running
       │◄─────HSEM sync───────►│
```

### 시퀀스 다이어그램
```
Time
  │
  │ Reset
  ├─────────────────────────────────
  │
  │    CM7                     CM4
  │     │                       │
  ├─────┼───────────────────────×─────
  │     │ Boot                  Clock OFF
  │     │ (Active)              (Gated)
  ├─────┼───────────────────────×─────
  │     │
  │     │ SystemClock_Config()
  │     │
  │     │ HAL_Init()
  │     │
  │     │ Peripheral_Init()
  │     │
  │     │ Application Init
  │     │
  ├─────┼───────────────────────┼─────
  │     │ Enable CM4 Boot       │
  │     │ RCC_CM4Boot()   ─────►│ Clock ON
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

## 사용 사례

### 적합한 경우

1. **조건부 CM4 사용**
   - CM4가 항상 필요하지 않은 경우
   - 특정 조건에서만 CM4 활성화
   - 예: 특정 작업 모드에서만 CM4 사용

2. **전력 최적화**
   - CM4가 필요 없을 때는 완전히 OFF
   - 배터리 구동 기기에 유리
   - 대기 전력 최소화

3. **순차적 초기화**
   - CM7이 모든 하드웨어 초기화 완료 후 CM4 시작
   - 복잡한 초기화 시퀀스 단순화

4. **동적 작업 할당**
   - 런타임에 CM4 필요 여부 결정
   - 리소스 관리 최적화

### 부적합한 경우
1. CM4가 항상 실행되어야 하는 경우
2. CM4가 독립적으로 부팅해야 하는 경우
3. 실시간 응답이 중요한 경우 (부팅 지연)

## 코드 예제

### CM7: CM4 부팅 제어
```c
int main(void)
{
  // MPU 설정
  MPU_Config();

  // CPU 캐시 활성화
  SCB_EnableICache();
  SCB_EnableDCache();

  // HAL 초기화
  HAL_Init();

  // 시스템 클럭 설정
  SystemClock_Config();

  // LED 초기화
  BSP_LED_Init(LED1);
  BSP_LED_Init(LED2);

  // CM7 애플리케이션 초기화
  Application_Init();

  // 특정 조건에서 CM4 부팅
  if (need_cm4_processing())
  {
    BSP_LED_On(LED1);  // CM4 부팅 신호

    // CM4 클럭 활성화 및 부팅
    HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);

    // HSEM을 통한 동기화
    __HAL_RCC_HSEM_CLK_ENABLE();
    HAL_HSEM_FastTake(HSEM_ID_0);
    HAL_HSEM_Release(HSEM_ID_0, 0);

    BSP_LED_Off(LED1);  // CM4 부팅 완료
  }

  // 메인 루프
  while (1)
  {
    BSP_LED_Toggle(LED2);
    HAL_Delay(500);

    // 런타임에 CM4 활성화도 가능
    if (runtime_condition() && !cm4_running)
    {
      HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);
      cm4_running = 1;
    }
  }
}
```

### CM4: 부팅 대기 및 시작
```c
int main(void)
{
  // CM7이 시스템 초기화 완료까지 대기
  while(__HAL_HSEM_SEMID_GET(HSEM_ID_0) != 0)
  {
    // HSEM 대기
  }

  // HAL 초기화 (CM4)
  HAL_Init();

  // CM4 LED 초기화
  BSP_LED_Init(LED3);

  // CM4 애플리케이션 초기화
  CM4_Application_Init();

  // 메인 루프
  while (1)
  {
    BSP_LED_Toggle(LED3);
    HAL_Delay(1000);

    // CM4 작업 처리
    Process_Background_Tasks();
  }
}
```

## CM4 부팅 제어 함수

### HAL 함수 사용
```c
/**
  * @brief  CM4 부팅 활성화
  * @param  RCC_BOOT_C2: CM4 (Cortex-M4) 부팅
  * @retval None
  */
HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);
```

### 레지스터 직접 제어
```c
// CM4 클럭 활성화 및 리셋 해제
__HAL_RCC_C2_GRP1_FORCE_RESET();
__HAL_RCC_C2_GRP1_RELEASE_RESET();

// CM4 부팅 주소 설정 (옵션)
FLASH->CM4_BOOT_ADD_0 = 0x08020000;  // CM4 시작 주소

// CM4 부트 플래그 설정
__HAL_RCC_C2_GRP1_CLOCK_ENABLE();
```

## 메모리 구성

### Flash 레이아웃
```
0x08000000  ┌─────────────────────┐
            │                     │
            │  CM7 Code           │
            │  (Bank 1)           │
            │                     │
0x08020000  ├─────────────────────┤
            │                     │
            │  CM4 Code           │
            │  (Bank 1 or 2)      │
            │                     │
0x08040000  ├─────────────────────┤
            │                     │
            │  Shared Data        │
            │                     │
0x08100000  └─────────────────────┘
```

### 공유 메모리 영역
```c
// D3 SRAM을 코어 간 통신에 사용
#define SHARED_MEM_BASE   0x38000000
#define SHARED_MEM_SIZE   0x10000  // 64KB

typedef struct {
  uint32_t cm7_to_cm4_flag;
  uint32_t cm4_to_cm7_flag;
  uint8_t  shared_buffer[1024];
  uint32_t cm4_status;
} SharedMemory_t;

SharedMemory_t *shared_mem = (SharedMemory_t*)SHARED_MEM_BASE;
```

## 전력 관리

### CM4 동적 활성화/비활성화
```c
// CM4 활성화 (필요 시)
void Enable_CM4(void)
{
  if (!cm4_running)
  {
    HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);
    cm4_running = 1;

    // CM4 준비 대기
    while (shared_mem->cm4_status != CM4_READY);
  }
}

// CM4 비활성화 (작업 완료 후)
void Disable_CM4(void)
{
  if (cm4_running)
  {
    // CM4에 종료 신호
    shared_mem->cm7_to_cm4_flag = CMD_SHUTDOWN;

    // CM4 종료 대기
    while (shared_mem->cm4_status != CM4_STOPPED);

    // CM4 클럭 비활성화
    __HAL_RCC_C2_GRP1_CLOCK_DISABLE();
    cm4_running = 0;
  }
}
```

### 전력 소비 비교

| 모드 | CM7 | CM4 | 총 전력 |
|------|-----|-----|---------|
| Both Running | 400MHz | 200MHz | ~100% |
| CM4 Gated | 400MHz | OFF | ~70% |
| CM7 Sleep, CM4 Gated | Sleep | OFF | ~5% |

## Option Bytes 설정

### STM32CubeProgrammer 사용
```bash
# 현재 Option Bytes 읽기
STM32_Programmer_CLI -c port=SWD -r32 0x5200201C 1

# BCM7=1, BCM4=0 설정
STM32_Programmer_CLI -c port=SWD -ob BCM7=1 BCM4=0
```

### 프로그래밍 방식
```c
FLASH_OBProgramInitTypeDef OBInit;

// Option Bytes 읽기
HAL_FLASHEx_OBGetConfig(&OBInit);

// BCM4=0, BCM7=1 설정
OBInit.BootConfig = OB_BCM4_DISABLE | OB_BCM7_ENABLE;

// Option Bytes 프로그램
HAL_FLASH_Unlock();
HAL_FLASH_OB_Unlock();
HAL_FLASHEx_OBProgram(&OBInit);
HAL_FLASH_OB_Launch();
HAL_FLASH_OB_Lock();
HAL_FLASH_Lock();
```

## 디버깅

### 멀티코어 디버그 설정

**STM32CubeIDE:**
1. Debug Configuration 열기
2. Debugger 탭
3. "Enable multi-core debug" 체크
4. Core 0 (CM7), Core 1 (CM4) 설정

### CM4 부팅 확인
```c
// CM7에서 CM4 상태 확인
uint32_t cm4_running = READ_BIT(RCC->CR, RCC_CR_D2CKRDY);
if (cm4_running)
{
  printf("CM4 is running\n");
}
else
{
  printf("CM4 is gated\n");
}
```

## 트러블슈팅

### CM4가 부팅되지 않는 경우
1. **Option Bytes 확인**: BCM4=0, BCM7=1
2. **부팅 함수 호출 확인**: `HAL_RCCEx_EnableBootCore()` 호출됨
3. **CM4 바이너리 존재 확인**: Flash에 프로그램됨
4. **리셋 벡터 확인**: CM4 벡터 테이블이 올바른 위치에 있음

### 디버거 연결 문제
- CM4가 게이트된 상태에서는 디버거가 CM4에 연결 불가
- CM7 먼저 디버그하고, CM4 활성화 후 연결

## 사용 예제 시나리오

### 시나리오 1: 이미지 처리
```c
// CM7: 메인 애플리케이션
void process_image(uint8_t *image_data)
{
  // CM4 활성화
  Enable_CM4();

  // 이미지 데이터 공유 메모리로 복사
  memcpy(shared_mem->shared_buffer, image_data, IMAGE_SIZE);

  // CM4에 처리 요청
  shared_mem->cm7_to_cm4_flag = CMD_PROCESS_IMAGE;

  // CM4 처리 완료 대기
  while (shared_mem->cm4_to_cm7_flag != FLAG_DONE);

  // 결과 수신
  memcpy(processed_image, shared_mem->shared_buffer, IMAGE_SIZE);

  // CM4 비활성화 (전력 절약)
  Disable_CM4();
}
```

### 시나리오 2: 백그라운드 모니터링
```c
// CM7: 사용자 인터페이스
// CM4: 센서 모니터링 (필요 시에만)

void start_monitoring(void)
{
  Enable_CM4();
  shared_mem->cm7_to_cm4_flag = CMD_START_MONITORING;
}

void stop_monitoring(void)
{
  shared_mem->cm7_to_cm4_flag = CMD_STOP_MONITORING;
  Disable_CM4();
}
```

## 다른 부팅 모드와의 비교

| 특성 | BootCM7_CM4Gated | BootCM4_CM7 | BootCM4_CM7Gated |
|------|------------------|-------------|------------------|
| CM7 초기 상태 | Running | Running | Gated |
| CM4 초기 상태 | Gated | Running | Running |
| 전력 효율 | ⭐⭐⭐ | ⭐ | ⭐⭐ |
| 유연성 | ⭐⭐⭐ | ⭐ | ⭐⭐ |
| 복잡도 | ⭐⭐ | ⭐ | ⭐⭐⭐ |

## 참고 자료

- AN5361: Getting started with projects based on dual-core STM32H7
- RM0399: STM32H745/755 Reference Manual, Section 6.3
- PM0253: STM32H7 Programming Manual
