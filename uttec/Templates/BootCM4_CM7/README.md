# BootCM4_CM7 - 양쪽 코어 동시 부팅 템플릿

## 개요

이 템플릿은 Cortex-M4와 Cortex-M7이 **동시에 부팅**하는 구성을 보여줍니다. 이것이 STM32H745의 기본 부팅 모드입니다.

## 부팅 구성

### User Option Bytes
- **BCM4 = 1**: CM4 부팅 활성화
- **BCM7 = 1**: CM7 부팅 활성화

두 비트가 모두 1로 설정되어 있어 파워온/리셋 시 두 코어가 모두 자동으로 부팅됩니다.

## 동작 방식

### 1. 파워온/리셋 직후
```
┌─────────────┐         ┌─────────────┐
│   CM7 Core  │         │   CM4 Core  │
│   (400MHz)  │         │   (200MHz)  │
└──────┬──────┘         └──────┬──────┘
       │                       │
       │ Both cores boot       │
       │◄─────────────────────►│
       ▼                       ▼
   Running                 STOP mode
```

### 2. CM7 초기화 과정
```c
// 1. 시스템 클럭 설정
SystemClock_Config();

// 2. CPU 캐시 활성화
SCB_EnableICache();
SCB_EnableDCache();

// 3. 주변장치 초기화
MPU_Config();
HAL_Init();

// 4. HSEM 초기화
__HAL_RCC_HSEM_CLK_ENABLE();
```

### 3. CM4 활성화
```c
// CM7이 HSEM을 통해 CM4 해제
HAL_HSEM_FastTake(HSEM_ID_0);
HAL_HSEM_Release(HSEM_ID_0, 0);

// CM4는 HSEM 대기 후 실행
while(__HAL_HSEM_SEMID_GET(HSEM_ID_0) != 0) {}
```

## 시퀀스 다이어그램

```
Time
  │
  │ Reset
  ├─────────────────────────────────
  │
  │    CM7                     CM4
  │     │                       │
  ├─────┼───────────────────────┼─────
  │     │ Boot                  │ Boot
  │     │ (Active)              │ (STOP mode)
  ├─────┼───────────────────────┼─────
  │     │                       │
  │     │ SystemClock_Config()  │
  │     │                       │
  │     │ HAL_Init()            │
  │     │                       │
  │     │ Peripheral_Init()     │
  │     │                       │
  ├─────┼───────────────────────┼─────
  │     │                       │
  │     │ Release HSEM ────────►│
  │     │                       │ Wake up
  │     │                       │ from STOP
  ├─────┼───────────────────────┼─────
  │     │                       │
  │     │ Application           │ Application
  │     │ Running               │ Running
  │     │                       │
  ▼     ▼                       ▼
```

## 사용 사례

### 적합한 경우
1. **대부분의 듀얼 코어 애플리케이션**
   - 가장 일반적이고 간단한 구성
   - CM7이 시스템 마스터 역할

2. **병렬 처리 애플리케이션**
   - CM7: 메인 제어 및 UI
   - CM4: 백그라운드 데이터 처리

3. **실시간 시스템**
   - CM7: 사용자 인터페이스
   - CM4: 실시간 제어 루프

### 부적합한 경우
1. CM4가 메인 프로세서로 동작해야 하는 경우
2. CM7의 전력 소비를 최소화해야 하는 경우
3. 동적으로 코어를 활성화/비활성화해야 하는 경우

## 메모리 구성

### Flash 메모리
```
0x08000000  ┌─────────────────────┐
            │                     │
            │  CM7 Code & Data    │
            │                     │
0x08020000  ├─────────────────────┤
            │                     │
            │  CM4 Code & Data    │
            │                     │
0x08040000  └─────────────────────┘
```

### RAM 메모리
```
CM7 전용:
0x00000000  ┌─────────────────────┐
            │   ITCM-RAM (64KB)   │
0x00010000  └─────────────────────┘

0x20000000  ┌─────────────────────┐
            │  DTCM-RAM (128KB)   │
0x20020000  └─────────────────────┘

공유 메모리:
0x24000000  ┌─────────────────────┐
            │  AXI-SRAM (512KB)   │
            │  (CM7 cacheable)    │
0x24080000  └─────────────────────┘

0x30000000  ┌─────────────────────┐
            │  D2 SRAM (288KB)    │
0x30048000  └─────────────────────┘

0x38000000  ┌─────────────────────┐
            │  D3 SRAM (64KB)     │
            │  (Inter-core comm)  │
0x38010000  └─────────────────────┘
```

## 코드 예제

### CM7 메인 코드
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

  // 시스템 클럭 설정 (CM7: 400MHz, CM4: 200MHz)
  SystemClock_Config();

  // LED 초기화
  BSP_LED_Init(LED1);

  // HSEM 클럭 활성화
  __HAL_RCC_HSEM_CLK_ENABLE();

  // CM4 부팅 허용
  HAL_HSEM_FastTake(HSEM_ID_0);
  HAL_HSEM_Release(HSEM_ID_0, 0);

  // 메인 루프
  while (1)
  {
    BSP_LED_Toggle(LED1);
    HAL_Delay(500);
  }
}
```

### CM4 메인 코드
```c
int main(void)
{
  // CM7이 시스템 초기화 완료까지 대기
  while(__HAL_HSEM_SEMID_GET(HSEM_ID_0) != 0)
  {
    // STOP 모드에서 대기
  }

  // HAL 초기화 (CM4)
  HAL_Init();

  // LED 초기화
  BSP_LED_Init(LED2);

  // 메인 루프
  while (1)
  {
    BSP_LED_Toggle(LED2);
    HAL_Delay(1000);
  }
}
```

## HSEM을 통한 동기화

### HSEM (Hardware Semaphore) 역할
- 코어 간 부팅 순서 제어
- 공유 리소스 접근 제어
- 코어 간 이벤트 통지

### 사용 예제
```c
// CM7: 리소스 획득
if (HAL_HSEM_FastTake(HSEM_ID_1) == HAL_OK)
{
  // 공유 리소스 사용
  shared_resource_access();

  // 리소스 해제
  HAL_HSEM_Release(HSEM_ID_1, 0);
}

// CM4: 리소스 획득 대기
while (HAL_HSEM_Take(HSEM_ID_1, 0) != HAL_OK)
{
  // 대기
}
// 공유 리소스 사용
shared_resource_access();
HAL_HSEM_Release(HSEM_ID_1, 0);
```

## 빌드 및 플래시

### STM32CubeIDE 사용
1. **프로젝트 임포트**
   - File → Import → Existing Projects
   - BootCM4_CM7/STM32CubeIDE 선택

2. **빌드 순서**
   - CM7 프로젝트 먼저 빌드
   - CM4 프로젝트 빌드

3. **디버그/실행**
   - CM7 프로젝트에서 Run/Debug 시작
   - 두 코어 모두 자동으로 플래시됨

### 커맨드라인 사용
```bash
# CM7 플래시
STM32_Programmer_CLI -c port=SWD -w CM7/Debug/BootCM4_CM7_CM7.elf -v

# CM4 플래시
STM32_Programmer_CLI -c port=SWD -w CM4/Debug/BootCM4_CM7_CM4.elf -v

# 리셋
STM32_Programmer_CLI -c port=SWD -rst
```

## 트러블슈팅

### CM4가 시작하지 않는 경우
1. **HSEM 확인**: CM7이 HSEM을 올바르게 해제하는지 확인
2. **Option Bytes 확인**: BCM4와 BCM7이 모두 1인지 확인
   ```bash
   STM32_Programmer_CLI -c port=SWD -r32 0x5200201C 1
   ```
3. **CM4 바이너리 확인**: Flash에 올바르게 프로그램되었는지 확인

### 코어 간 통신 오류
1. **공유 메모리 영역 확인**: 올바른 주소 사용 여부
2. **캐시 일관성**: CM7 D-Cache 관리 확인
   ```c
   SCB_CleanDCache_by_Addr((uint32_t*)shared_buffer, size);
   ```

### 디버깅 팁
- STM32CubeIDE에서 멀티코어 디버그 세션 사용
- 각 코어에 브레이크포인트 설정 가능
- HSEM 레지스터 모니터링으로 동기화 상태 확인

## 다른 부팅 모드와의 비교

| 특성 | BootCM4_CM7 | BootCM7_CM4Gated | BootCM4_CM7Gated |
|------|-------------|------------------|------------------|
| CM7 자동 부팅 | ✅ | ✅ | ❌ |
| CM4 자동 부팅 | ✅ | ❌ | ✅ |
| 초기화 담당 | CM7 | CM7 | CM4 |
| 전력 소비 | 높음 | 중간 | 중간 |
| 사용 복잡도 | 낮음 | 중간 | 높음 |

## 참고 자료

- AN5361: Getting started with projects based on dual-core STM32H7
- RM0399: STM32H745/755 Reference Manual
- 섹션 6.3: Boot configuration
