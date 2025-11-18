# Templates_LL - Low-Level API 템플릿

## 개요

이 템플릿은 STM32H7 Low-Level (LL) API를 사용하여 듀얼 코어 애플리케이션을 개발하기 위한 기본 프로젝트입니다.

## 주요 특징

### 듀얼 코어 구성
- **Cortex-M7 (CM7)**: 400MHz에서 동작하는 메인 프로세서
- **Cortex-M4 (CM4)**: 200MHz에서 동작하는 보조 프로세서

### 시스템 초기화
- CM7이 시스템 클럭과 주변장치 초기화 담당
- CM4는 초기에 STOP 모드로 대기
- HSEM (Hardware Semaphore)을 통해 CM4 활성화

### LL API 사용
- HAL 대신 Low-Level 드라이버 사용
- 코드 사이즈 최적화 가능
- 더 빠른 실행 속도
- 하드웨어에 대한 직접적인 제어

## 프로젝트 구조

```
Templates_LL/
├── CM4/              # Cortex-M4 코드
│   ├── Inc/         # 헤더 파일
│   └── Src/         # 소스 파일
├── CM7/              # Cortex-M7 코드
│   ├── Inc/         # 헤더 파일
│   └── Src/         # 소스 파일
├── Common/           # 공유 코드
│   ├── Inc/         # 공유 헤더
│   └── Src/         # 공유 소스
├── EWARM/           # IAR 프로젝트 파일
├── MDK-ARM/         # Keil 프로젝트 파일
└── STM32CubeIDE/    # STM32CubeIDE 프로젝트 파일
    ├── CM4/         # CM4 빌드 설정
    └── CM7/         # CM7 빌드 설정
```

## 클럭 설정

### 시스템 클럭
- **CM7**: 400 MHz
- **CM4**: 200 MHz

### 도메인 클럭
- **D1 Domain (AXI/AHB3)**: 200 MHz
- **D2 Domain (AHB1/AHB2)**: 200 MHz
- **D3 Domain (AHB4)**: 200 MHz
- **APB Peripherals**: 100 MHz

## 부팅 시퀀스

1. **파워온/리셋**
   - 두 코어 모두 부팅 준비 (BCM4=1, BCM7=1)

2. **CM7 초기화**
   - 시스템 클럭 설정
   - L1 캐시 활성화 (I-Cache, D-Cache)
   - 주변장치 초기화
   - HSEM 초기화

3. **CM4 활성화**
   - CM7이 HSEM을 통해 CM4 해제
   - CM4가 STOP 모드에서 깨어남
   - CM4 애플리케이션 시작

## 메모리 맵

### CM7 메모리
- **ITCM-RAM**: 0x00000000 (64KB) - 제로 웨이트 명령어 실행
- **DTCM-RAM**: 0x20000000 (128KB) - 고속 데이터 액세스
- **AXI-SRAM**: 0x24000000 (512KB) - 캐시 가능

### CM4 메모리
- **D2 SRAM**: CM4 주 메모리 영역
- **Shared Memory**: CM7과 공유 가능한 메모리 영역

### 공유 메모리
- **D3 SRAM**: 0x38000000 (64KB) - BDMA 액세스 가능, 코어 간 통신용

## LL API 주요 모듈

### 클럭 및 전원
- `stm32h7xx_ll_rcc.h` - RCC (Reset and Clock Control)
- `stm32h7xx_ll_pwr.h` - 전원 관리

### GPIO
- `stm32h7xx_ll_gpio.h` - GPIO 제어

### 통신
- `stm32h7xx_ll_usart.h` - UART/USART
- `stm32h7xx_ll_spi.h` - SPI
- `stm32h7xx_ll_i2c.h` - I2C

### 타이머
- `stm32h7xx_ll_tim.h` - 범용 타이머
- `stm32h7xx_ll_lptim.h` - 저전력 타이머

### DMA
- `stm32h7xx_ll_dma.h` - DMA
- `stm32h7xx_ll_bdma.h` - BDMA

## 사용 방법

### 1. 프로젝트 임포트
STM32CubeIDE에서:
1. File → Import → Existing Projects into Workspace
2. Templates_LL 폴더 선택
3. CM4, CM7 프로젝트 모두 임포트

### 2. 빌드
- CM7 프로젝트 먼저 빌드
- CM4 프로젝트 빌드

### 3. 플래시
```bash
# CM7 바이너리 플래시
STM32_Programmer_CLI -c port=SWD -w CM7/Debug/Templates_LL_CM7.elf -v -rst

# CM4 바이너리 플래시
STM32_Programmer_CLI -c port=SWD -w CM4/Debug/Templates_LL_CM4.elf -v -rst
```

## 커스터마이징

### CM7 메인 함수 수정
`CM7/Src/main.c` 파일을 수정하여 CM7의 동작 정의

### CM4 메인 함수 수정
`CM4/Src/main.c` 파일을 수정하여 CM4의 동작 정의

### 공유 리소스 추가
`Common/` 폴더에 양쪽 코어가 사용할 코드 추가

## LED 예제 코드

### CM7 LED 제어 (LL API)
```c
// LED 초기화
LL_AHB4_GRP1_EnableClock(LL_AHB4_GRP1_PERIPH_GPIOI);

LL_GPIO_InitTypeDef GPIO_InitStruct = {0};
GPIO_InitStruct.Pin = LL_GPIO_PIN_12;  // LED1
GPIO_InitStruct.Mode = LL_GPIO_MODE_OUTPUT;
GPIO_InitStruct.Speed = LL_GPIO_SPEED_FREQ_LOW;
GPIO_InitStruct.OutputType = LL_GPIO_OUTPUT_PUSHPULL;
GPIO_InitStruct.Pull = LL_GPIO_PULL_NO;
LL_GPIO_Init(GPIOI, &GPIO_InitStruct);

// LED 토글
while(1)
{
  LL_GPIO_TogglePin(GPIOI, LL_GPIO_PIN_12);
  LL_mDelay(500);
}
```

## 주의사항

### 캐시 일관성
- CM7은 D-Cache를 사용하므로 공유 메모리 사용 시 캐시 관리 필요
- 공유 데이터는 D3 SRAM 사용 권장 (캐시 불가 영역)

### HSEM 사용
- 공유 리소스 접근 시 HSEM으로 동기화 필수
- 데드락 방지를 위한 타임아웃 설정 권장

### 인터럽트 우선순위
- CM7, CM4 각각 독립적인 NVIC 보유
- 인터럽트 우선순위 충돌 주의

## 다음 단계

이 템플릿을 기반으로:
1. 필요한 주변장치 초기화 추가
2. 코어 간 통신 메커니즘 구현 (메시지 큐, 공유 버퍼 등)
3. 애플리케이션 로직 구현

## 참고 자료

- STM32H7xx Reference Manual (RM0399)
- STM32H745/755 Datasheet
- UM2609: STM32CubeH7 Low-Layer API description
- AN5361: Getting started with projects based on dual-core STM32H7 microcontrollers
