# STM32H745I-DISCO 예제 프로젝트 핀 설정 주의사항

## 개요

이 문서는 STM32H745I-DISCO 예제 프로젝트들의 GPIO 핀 설정에 대한 중요한 주의사항을 설명합니다. 특히 UART 핀 설정에서 혼동될 수 있는 부분을 명확히 합니다.

## 핵심 이슈: USART3 핀 설정

### 문서와 실제 코드의 차이

일부 문서에서 USART3의 핀이 `PD8/PD9`로 기재되어 있을 수 있으나, **실제 예제 프로젝트 코드는 `PB10/PB11`을 사용합니다**.

### 실제 프로젝트 코드 확인

**파일 위치**: `STM32H745I-DISCO/Examples/UART/UART_WakeUpFromStopUsingFIFO/CM7/Inc/main.h`

```c
// Lines 45-50
#define USARTx_TX_PIN                    GPIO_PIN_10
#define USARTx_TX_GPIO_PORT              GPIOB
#define USARTx_TX_AF                     GPIO_AF7_USART3
#define USARTx_RX_PIN                    GPIO_PIN_11
#define USARTx_RX_GPIO_PORT              GPIOB
#define USARTx_RX_AF                     GPIO_AF7_USART3
```

**실제 사용 핀**:
- **TX**: PB10 (GPIO_AF7_USART3)
- **RX**: PB11 (GPIO_AF7_USART3)

## 왜 이런 차이가 발생하는가?

### 1. STM32H745의 Alternate Function 시스템

STM32H745 MCU에서 USART3는 여러 GPIO 핀에 매핑될 수 있습니다:

| 핀 옵션 | TX | RX | Alternate Function |
|---------|----|----|-------------------|
| **옵션 1** | PB10 | PB11 | AF7 |
| 옵션 2 | PC10 | PC11 | AF7 |
| 옵션 3 | PD8 | PD9 | AF7 |

### 2. 보드별 하드웨어 연결

**STM32H745I-DISCO 보드**에서는 ST-LINK의 VCP(Virtual COM Port)가 특정 핀에 연결되어 있습니다:

```
ST-LINK VCP ←→ STM32H745
    TX     ←→  PB10 (USART3_TX)
    RX     ←→  PB11 (USART3_RX)
```

이 하드웨어 연결은 보드 회로도에 의해 **고정**되어 있으며, 소프트웨어로 변경할 수 없습니다.

### 3. PD8/PD9를 사용할 수 없는 이유

STM32H745I-DISCO 보드에서:

- **PD8**: FMC (Flexible Memory Controller)의 D13 데이터 라인에 연결됨
- **PD9**: FMC의 D14 데이터 라인에 연결됨

이 핀들은 외부 SDRAM과의 통신에 사용되므로, UART용으로 사용하면 **메모리 충돌**이 발생합니다.

## 핀 충돌 분석

### STM32H745I-DISCO 보드의 주요 핀 할당

```
┌─────────────────────────────────────────────────────┐
│ GPIO Port D (PD) - 주로 FMC에 할당됨                │
├──────┬──────────────────────────────────────────────┤
│ PD0  │ FMC_D2                                       │
│ PD1  │ FMC_D3                                       │
│ PD8  │ FMC_D13  ← USART3_TX로 사용 불가             │
│ PD9  │ FMC_D14  ← USART3_RX로 사용 불가             │
│ PD10 │ FMC_D15                                      │
│ PD14 │ FMC_D0                                       │
│ PD15 │ FMC_D1                                       │
└──────┴──────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ GPIO Port B (PB) - UART/I2C 등에 사용 가능          │
├──────┬──────────────────────────────────────────────┤
│ PB10 │ USART3_TX → ST-LINK VCP                      │
│ PB11 │ USART3_RX → ST-LINK VCP                      │
└──────┴──────────────────────────────────────────────┘
```

## 다른 STM32 보드와의 비교

### Nucleo 보드 (예: NUCLEO-H745ZI-Q)

Nucleo 보드에서는 **PD8/PD9**가 ST-LINK VCP에 연결될 수 있습니다:

```c
// Nucleo-H745ZI-Q의 경우
#define USARTx_TX_PIN     GPIO_PIN_8
#define USARTx_TX_PORT    GPIOD
#define USARTx_RX_PIN     GPIO_PIN_9
#define USARTx_RX_PORT    GPIOD
```

이는 Nucleo 보드에는 외부 SDRAM이 없어서 PD8/PD9 핀이 자유롭기 때문입니다.

### Discovery 보드 (STM32H745I-DISCO)

Discovery 보드는 외부 SDRAM이 있어서 **PB10/PB11**을 사용합니다:

```c
// STM32H745I-DISCO의 경우
#define USARTx_TX_PIN     GPIO_PIN_10
#define USARTx_TX_PORT    GPIOB
#define USARTx_RX_PIN     GPIO_PIN_11
#define USARTx_RX_PORT    GPIOB
```

## 예제별 핀 설정 확인 방법

### 1. main.h 파일 확인

각 예제의 `CM7/Inc/main.h` 파일에서 핀 정의를 확인합니다:

```bash
# 예제
grep -n "USARTx_TX_PIN\|USARTx_RX_PIN" CM7/Inc/main.h
```

### 2. stm32h7xx_hal_msp.c 확인

GPIO 초기화 코드에서 실제 사용되는 핀을 확인합니다:

```bash
# 예제
grep -n "GPIO_PIN_\|GPIOB\|GPIOD" CM7/Src/stm32h7xx_hal_msp.c
```

## 영향받는 예제 프로젝트 목록

다음 UART 관련 예제들은 모두 **PB10/PB11**을 사용합니다:

1. **UART_WakeUpFromStopUsingFIFO** - FIFO 웨이크업
2. **UART_HyperTerminal_IT** - 인터럽트 방식
3. **UART_HyperTerminal_DMA** - DMA 방식
4. **UART_ReceptionToIdle_CircularDMA** - 순환 DMA
5. **UART_TwoBoards_ComIT** - 보드 간 통신 (IT)
6. **UART_TwoBoards_ComDMA** - 보드 간 통신 (DMA)
7. **UART_TwoBoards_ComPolling** - 보드 간 통신 (Polling)

## 실제 테스트 시 주의사항

### 1. 터미널 프로그램 설정

ST-LINK VCP를 통해 통신할 때:
- **COM 포트**: ST-LINK Virtual COM Port 선택
- **Baud Rate**: 프로젝트별로 다름 (9600, 115200 등)
- **Data Bits**: 8
- **Stop Bits**: 1
- **Parity**: 프로젝트별로 다름 (None, Odd 등)

### 2. 하드웨어 연결

외부 UART 장치를 연결하려면:
- ST-LINK VCP를 사용하지 않고
- CN12 커넥터의 PB10/PB11 핀에 직접 연결

### 3. 코드 수정 시

다른 UART 핀을 사용하려면 main.h의 정의를 수정해야 하지만, **반드시 보드 회로도를 확인**하여 핀 충돌이 없는지 확인해야 합니다.

## CubeMX와의 차이

### CubeMX 기본 설정

STM32CubeMX에서 STM32H745I-DISCO 보드를 선택하고 USART3를 활성화하면:
- 기본적으로 **PD8/PD9**가 선택될 수 있음
- 이는 CubeMX가 보드 특성을 완전히 반영하지 않기 때문

### 올바른 설정

CubeMX에서 STM32H745I-DISCO 보드용 프로젝트를 생성할 때:
1. USART3 활성화
2. Pinout 뷰에서 PD8/PD9를 **비활성화**
3. PB10/PB11을 USART3_TX/RX로 **수동 설정**

```
CubeMX Pinout Configuration:
┌─────────┬───────────────┬────────────┐
│ Pin     │ Signal        │ AF         │
├─────────┼───────────────┼────────────┤
│ PB10    │ USART3_TX     │ AF7        │
│ PB11    │ USART3_RX     │ AF7        │
│ PD8     │ FMC_D13       │ AF12       │
│ PD9     │ FMC_D14       │ AF12       │
└─────────┴───────────────┴────────────┘
```

## 결론

1. **항상 실제 프로젝트 코드를 확인**하세요. 문서나 일반적인 정보가 특정 보드의 실제 구성과 다를 수 있습니다.

2. **STM32H745I-DISCO 보드에서 USART3는 PB10/PB11을 사용**합니다. PD8/PD9는 FMC(외부 메모리)에 할당되어 있어 사용할 수 없습니다.

3. **보드 회로도 참조**: 핀 설정에 의문이 있을 때는 항상 보드의 회로도(Schematic)를 확인하세요.
   - STM32H745I-DISCO 회로도는 ST 웹사이트에서 다운로드 가능

4. **코드 이식 시 주의**: 다른 보드(예: Nucleo)의 예제를 Discovery 보드로 이식할 때, 핀 설정을 반드시 수정해야 합니다.

## 참고 자료

- **UM2488**: Discovery kit with STM32H745XI MCU - User Manual
- **MB1381**: STM32H745I-DISCO Schematic
- **DS12923**: STM32H745xI/G Datasheet (Alternate Function 표)
- **RM0399**: STM32H745/755 Reference Manual

## 수정된 문서 목록

다음 문서들의 핀 설정이 수정되었습니다:

### 1. UART_WakeUpFromStopUsingFIFO/README.md
- **수정 내용**:
  - TX/RX 핀: PD8/PD9 → PB10/PB11
  - GPIO 포트: GPIOD → GPIOB
  - GPIO 클럭: __HAL_RCC_GPIOD_CLK_ENABLE() → __HAL_RCC_GPIOB_CLK_ENABLE()
  - Pull 설정: GPIO_NOPULL → GPIO_PULLUP
  - Speed 설정: GPIO_SPEED_FREQ_LOW → GPIO_SPEED_FREQ_VERY_HIGH
  - Baud Rate: 115200 → 9600
  - Parity: UART_PARITY_NONE → UART_PARITY_ODD

### 2. PWR_D2ON_D1OFF/README.md
- **수정 내용**:
  - TX/RX 핀: PD8/PD9 → PB10/PB11
  - GPIO 포트: GPIOD → GPIOB
  - GPIO 클럭: __HAL_RCC_GPIOD_CLK_ENABLE() → __HAL_RCC_GPIOB_CLK_ENABLE()
  - Pull 설정: GPIO_NOPULL → GPIO_PULLUP
  - Speed 설정: GPIO_SPEED_FREQ_LOW → GPIO_SPEED_FREQ_VERY_HIGH

## 문서 이력

| 날짜 | 버전 | 설명 |
|------|------|------|
| 2025-11-19 | 1.0 | 초기 작성 - USART3 핀 설정 주의사항 |
| 2025-11-19 | 1.1 | 오류 수정 - UART_WakeUpFromStopUsingFIFO, PWR_D2ON_D1OFF 문서 핀 설정 수정 |
