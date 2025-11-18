# BootCM7_CM4Gated_RAM - CM7 부팅, CM4 RAM 실행 템플릿

## 개요

이 템플릿은 CM7이 부팅한 후 **CM4 코드를 RAM으로 복사**하고, CM4가 **RAM에서 실행**되도록 하는 고급 구성입니다. CM4 코드를 동적으로 로드하거나 업데이트할 수 있는 유연성을 제공합니다.

## 부팅 구성

### User Option Bytes
- **BCM4 = 0**: CM4 부팅 비활성화 (초기)
- **BCM7 = 1**: CM7 부팅 활성화

## 주요 특징

### RAM 실행의 장점
1. **고속 실행**: RAM은 Flash보다 빠름 (제로 웨이트)
2. **동적 코드 로딩**: 런타임에 CM4 코드 변경 가능
3. **Flash 절약**: CM4 코드를 Flash에 중복 저장하지 않음
4. **펌웨어 업데이트**: CM4만 독립적으로 업데이트 가능

### 사용 사례
1. **빈번한 코드 업데이트**: CM4 펌웨어를 자주 변경하는 경우
2. **다중 CM4 이미지**: 상황에 따라 다른 CM4 코드 실행
3. **성능 최적화**: CM4 코드가 RAM에서 더 빠르게 실행되어야 하는 경우
4. **보안**: CM4 코드를 암호화하여 저장 후 복호화하여 RAM에 로드

## 메모리 구성

### Flash 레이아웃
```
0x08000000  ┌─────────────────────┐
            │                     │
            │  CM7 Code           │
            │  (Active)           │
            │                     │
0x08040000  ├─────────────────────┤
            │                     │
            │  CM4 Code Image     │
            │  (Copy Source)      │
            │                     │
0x08080000  ├─────────────────────┤
            │                     │
            │  Data / Resources   │
            │                     │
0x08100000  └─────────────────────┘
```

### RAM 레이아웃
```
D2 Domain SRAM (CM4 실행 영역):

0x10000000  ┌─────────────────────┐
            │  CM4 Vector Table   │
            │  (복사됨)           │
0x10000200  ├─────────────────────┤
            │                     │
            │  CM4 Code           │
            │  (CM7이 복사)       │
            │                     │
0x10020000  ├─────────────────────┤
            │  CM4 Data (.data)   │
0x10030000  ├─────────────────────┤
            │  CM4 BSS            │
0x10040000  ├─────────────────────┤
            │  CM4 Heap           │
0x10048000  ├─────────────────────┤
            │  CM4 Stack          │
0x10050000  └─────────────────────┘
```

## CM7 코드 (마스터)

### 메인 함수
```c
#define CM4_FLASH_ADDRESS   0x08040000  // Flash에서 CM4 코드 위치
#define CM4_RAM_ADDRESS     0x10000000  // RAM에 복사할 위치
#define CM4_CODE_SIZE       0x20000     // 128KB

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

  printf("CM7: System initialized\n");

  // HSEM 초기화
  __HAL_RCC_HSEM_CLK_ENABLE();

  // CM4 코드를 Flash에서 RAM으로 복사
  printf("CM7: Copying CM4 code to RAM...\n");
  Copy_CM4_Code_To_RAM();

  // CM4 부팅 주소 설정
  printf("CM7: Setting CM4 boot address...\n");
  Set_CM4_Boot_Address(CM4_RAM_ADDRESS);

  // CM4 부팅
  printf("CM7: Booting CM4 from RAM...\n");
  Boot_CM4();

  // HSEM으로 CM4 준비 대기
  HAL_HSEM_FastTake(HSEM_ID_0);
  HAL_HSEM_Release(HSEM_ID_0, 0);

  printf("CM7: CM4 is running from RAM\n");
  BSP_LED_On(LED1);

  // 메인 루프
  while (1)
  {
    BSP_LED_Toggle(LED2);
    HAL_Delay(500);
  }
}
```

### CM4 코드 복사 함수
```c
void Copy_CM4_Code_To_RAM(void)
{
  uint32_t *src = (uint32_t *)CM4_FLASH_ADDRESS;
  uint32_t *dst = (uint32_t *)CM4_RAM_ADDRESS;
  uint32_t size = CM4_CODE_SIZE / 4;  // Word 단위

  // Flash에서 RAM으로 복사
  for (uint32_t i = 0; i < size; i++)
  {
    dst[i] = src[i];
  }

  printf("CM7: Copied %lu bytes from 0x%08lX to 0x%08lX\n",
         CM4_CODE_SIZE,
         (uint32_t)src,
         (uint32_t)dst);

  // 캐시 클린 (D-Cache)
  SCB_CleanDCache_by_Addr((uint32_t *)dst, CM4_CODE_SIZE);
}

// DMA를 사용한 고속 복사
void Copy_CM4_Code_To_RAM_DMA(void)
{
  DMA_HandleTypeDef hdma;

  // DMA 초기화
  __HAL_RCC_DMA1_CLK_ENABLE();

  hdma.Instance = DMA1_Stream0;
  hdma.Init.Request = DMA_REQUEST_MEM2MEM;
  hdma.Init.Direction = DMA_MEMORY_TO_MEMORY;
  hdma.Init.PeriphInc = DMA_PINC_ENABLE;
  hdma.Init.MemInc = DMA_MINC_ENABLE;
  hdma.Init.PeriphDataAlignment = DMA_PDATAALIGN_WORD;
  hdma.Init.MemDataAlignment = DMA_MDATAALIGN_WORD;
  hdma.Init.Mode = DMA_NORMAL;
  hdma.Init.Priority = DMA_PRIORITY_HIGH;
  hdma.Init.FIFOMode = DMA_FIFOMODE_ENABLE;
  hdma.Init.FIFOThreshold = DMA_FIFO_THRESHOLD_FULL;
  hdma.Init.MemBurst = DMA_MBURST_INC4;
  hdma.Init.PeriphBurst = DMA_PBURST_INC4;

  HAL_DMA_Init(&hdma);

  // DMA 전송 시작
  uint32_t start_time = HAL_GetTick();

  HAL_DMA_Start(&hdma,
                CM4_FLASH_ADDRESS,
                CM4_RAM_ADDRESS,
                CM4_CODE_SIZE / 4);

  HAL_DMA_PollForTransfer(&hdma, HAL_DMA_FULL_TRANSFER, HAL_MAX_DELAY);

  uint32_t elapsed = HAL_GetTick() - start_time;

  printf("CM7: DMA copy completed in %lu ms\n", elapsed);

  // 캐시 클린
  SCB_CleanDCache_by_Addr((uint32_t *)CM4_RAM_ADDRESS, CM4_CODE_SIZE);
}
```

### CM4 부팅 주소 설정
```c
void Set_CM4_Boot_Address(uint32_t address)
{
  // CM4 부팅 주소를 RAM으로 설정
  // 주의: 주소는 128KB 정렬되어야 함

  // Option Bytes 방식 (영구적)
  FLASH_OBProgramInitTypeDef OBInit;

  HAL_FLASH_Unlock();
  HAL_FLASH_OB_Unlock();

  HAL_FLASHEx_OBGetConfig(&OBInit);

  // CM4 부팅 주소 설정 (예: 0x10000000)
  OBInit.BootAddr = address >> 17;  // 128KB 단위로 시프트
  OBInit.BootAddrConfig = OB_BOOTADDR_CM4;

  HAL_FLASHEx_OBProgram(&OBInit);

  // 적용하려면 리셋 필요 (여기서는 하지 않음)
  // HAL_FLASH_OB_Launch();

  HAL_FLASH_OB_Lock();
  HAL_FLASH_Lock();

  printf("CM7: CM4 boot address set to 0x%08lX\n", address);
}

// 레지스터를 직접 사용한 방법 (런타임)
void Set_CM4_Boot_Address_Runtime(uint32_t address)
{
  // BCR (Boot Configuration Register) 사용
  // 이 방법은 런타임에 변경 가능하지만 리셋 시 초기화됨

  // SYSCFG 클럭 활성화
  __HAL_RCC_SYSCFG_CLK_ENABLE();

  // CM4 부팅 주소 설정
  MODIFY_REG(SYSCFG->UR2, SYSCFG_UR2_BCM4, address >> 16);

  printf("CM7: CM4 runtime boot address set to 0x%08lX\n", address);
}
```

### CM4 부팅 함수
```c
void Boot_CM4(void)
{
  // CM4 클럭 활성화
  __HAL_RCC_CM4_CLK_ENABLE();

  // CM4 리셋 해제
  HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);

  printf("CM7: CM4 boot initiated\n");
}
```

## CM4 코드 (슬레이브)

### 링커 스크립트 (중요!)

CM4 코드는 RAM에서 실행되도록 링커 스크립트 수정 필요:

```ld
/* STM32H745xI_CM4_RAM.ld */

/* Entry Point */
ENTRY(Reset_Handler)

/* Memory Regions */
MEMORY
{
  RAM_CM4 (xrw)  : ORIGIN = 0x10000000, LENGTH = 128K
  RAM_DATA (rw)  : ORIGIN = 0x10020000, LENGTH = 64K
  RAM_D3 (xrw)   : ORIGIN = 0x38000000, LENGTH = 64K
}

/* Sections */
SECTIONS
{
  /* Vector table */
  .isr_vector :
  {
    . = ALIGN(4);
    KEEP(*(.isr_vector))
    . = ALIGN(4);
  } >RAM_CM4

  /* Code */
  .text :
  {
    . = ALIGN(4);
    *(.text)
    *(.text*)
    *(.rodata)
    *(.rodata*)
    . = ALIGN(4);
  } >RAM_CM4

  /* Data */
  .data :
  {
    . = ALIGN(4);
    *(.data)
    *(.data*)
    . = ALIGN(4);
  } >RAM_DATA

  /* BSS */
  .bss :
  {
    . = ALIGN(4);
    *(.bss)
    *(.bss*)
    *(COMMON)
    . = ALIGN(4);
  } >RAM_DATA

  /* Stack and Heap */
  ._user_heap_stack :
  {
    . = ALIGN(8);
    . = . + 0x4000;  /* 16KB Heap */
    . = . + 0x2000;  /* 8KB Stack */
    . = ALIGN(8);
  } >RAM_DATA
}
```

### CM4 메인 함수
```c
int main(void)
{
  // HSEM 대기
  while(__HAL_HSEM_SEMID_GET(HSEM_ID_0) != 0);

  // HAL 초기화
  HAL_Init();

  // LED 초기화
  BSP_LED_Init(LED3);

  printf("CM4: Running from RAM at 0x%08lX\n", (uint32_t)main);

  // 벡터 테이블 위치 확인
  uint32_t vtor = SCB->VTOR;
  printf("CM4: VTOR = 0x%08lX\n", vtor);

  BSP_LED_On(LED3);

  // 메인 루프
  while (1)
  {
    BSP_LED_Toggle(LED3);
    HAL_Delay(1000);

    // RAM에서 실행되는 코드
    // Flash보다 빠른 실행 속도!
  }
}
```

### 벡터 테이블 재배치
```c
// startup_stm32h745xx_cm4.s 에서
// 또는 main() 초기에

void SystemInit_CM4(void)
{
  // 벡터 테이블을 RAM 주소로 설정
  SCB->VTOR = 0x10000000;

  // FPU 활성화 (필요 시)
  #if (__FPU_PRESENT == 1) && (__FPU_USED == 1)
    SCB->CPACR |= ((3UL << 10*2)|(3UL << 11*2));
  #endif
}
```

## 동적 CM4 코드 업데이트

### Flash에서 새 이미지 로드
```c
void Update_CM4_Firmware(uint32_t new_image_address)
{
  printf("CM7: Stopping CM4...\n");

  // CM4 정지
  Stop_CM4();

  // 새 코드 복사
  printf("CM7: Loading new CM4 image from 0x%08lX...\n", new_image_address);
  Copy_CM4_Code_From_Address(new_image_address, CM4_RAM_ADDRESS, CM4_CODE_SIZE);

  // CM4 재시작
  printf("CM7: Restarting CM4...\n");
  Boot_CM4();

  printf("CM7: CM4 firmware updated successfully\n");
}

void Stop_CM4(void)
{
  // CM4에 정지 신호
  SHARED_MEM->cm7_to_cm4_cmd = CMD_STOP;

  // CM4 정지 대기
  HAL_Delay(100);

  // CM4 클럭 비활성화
  __HAL_RCC_CM4_CLK_DISABLE();
}
```

### 다중 CM4 이미지 선택
```c
// Flash에 여러 CM4 이미지 저장
#define CM4_IMAGE_A   0x08040000  // 기본 기능
#define CM4_IMAGE_B   0x08080000  // 고급 기능
#define CM4_IMAGE_C   0x080C0000  // 테스트 모드

void Load_CM4_Image(uint8_t image_id)
{
  uint32_t source_address;

  switch (image_id)
  {
    case 0:
      source_address = CM4_IMAGE_A;
      printf("CM7: Loading CM4 Image A (Default)\n");
      break;

    case 1:
      source_address = CM4_IMAGE_B;
      printf("CM7: Loading CM4 Image B (Advanced)\n");
      break;

    case 2:
      source_address = CM4_IMAGE_C;
      printf("CM7: Loading CM4 Image C (Test)\n");
      break;

    default:
      printf("CM7: Invalid image ID\n");
      return;
  }

  // CM4 이미지 로드
  Copy_CM4_Code_From_Address(source_address, CM4_RAM_ADDRESS, CM4_CODE_SIZE);
  Boot_CM4();
}
```

## 성능 비교

### 실행 속도
```
코드 위치        | 클럭 사이클 | 비고
----------------|------------|------------------
Flash (0x08)    | 7-8 cycles | Wait states 고려
SRAM (0x10)     | 1-2 cycles | 제로 웨이트
DTCM (0x20)     | 0 cycles   | CM7만 사용 가능
```

### RAM 실행 장점
- **3-4배 빠른 실행**: Wait state 없음
- **캐시 효율**: RAM은 캐시 관리가 쉬움
- **동적 로딩**: 런타임에 코드 교체

## 주의사항

### 1. 전원 차단 시
RAM의 내용은 전원 차단 시 사라집니다:
```c
// 해결책: 부팅 시마다 Flash에서 복사
void Startup_Sequence(void)
{
  // CM7 부팅
  SystemInit();

  // 항상 CM4 코드를 RAM으로 복사
  Copy_CM4_Code_To_RAM();

  // CM4 부팅
  Boot_CM4();
}
```

### 2. 메모리 제약
CM4 코드 크기가 D2 SRAM 크기(128KB)를 초과할 수 없음

### 3. 디버깅
RAM 실행 코드는 디버거 연결 시 추가 설정 필요:
```c
// STM32CubeIDE Debug Configuration
// - Symbol file: CM4 ELF 파일 지정
// - Load address: 0x10000000
```

## 보안 활용

### 암호화된 CM4 코드
```c
// Flash에는 암호화된 코드 저장
// CM7이 복호화하여 RAM에 로드

void Load_Encrypted_CM4_Code(void)
{
  uint8_t encrypted_code[CM4_CODE_SIZE];
  uint8_t decrypted_code[CM4_CODE_SIZE];

  // Flash에서 암호화된 코드 읽기
  memcpy(encrypted_code, (void *)CM4_FLASH_ADDRESS, CM4_CODE_SIZE);

  // AES 복호화
  AES_Decrypt(encrypted_code, decrypted_code, CM4_CODE_SIZE, aes_key);

  // RAM에 복사
  memcpy((void *)CM4_RAM_ADDRESS, decrypted_code, CM4_CODE_SIZE);

  printf("CM7: Decrypted and loaded CM4 code\n");
}
```

## 트러블슈팅

### CM4가 실행되지 않는 경우
1. **벡터 테이블**: VTOR이 올바른 주소인지 확인
2. **링커 스크립트**: RAM 주소로 빌드되었는지 확인
3. **부팅 주소**: `Set_CM4_Boot_Address()` 호출 확인

### 복사 오류
1. **캐시**: D-Cache를 클린했는지 확인
2. **주소 정렬**: 4바이트 정렬 확인
3. **크기**: 복사 크기가 올바른지 확인

## 참고 자료

- AN5361: Dual-core STM32H7 Getting Started
- RM0399: STM32H745 Reference Manual, Section 4 (Memory and bus architecture)
- 예제: `STM32H745I-DISCO/Templates/BootCM7_CM4Gated_RAM`
