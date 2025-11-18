# OpenAMP_PingPong - OpenAMP 프레임워크를 이용한 코어 간 통신

## 개요

이 애플리케이션은 OpenAMP 프레임워크를 사용하여 Cortex-M7과 Cortex-M4 간에 양방향 메시지 통신을 구현하는 예제입니다. "PingPong" 방식으로 카운터 값을 증가시키며 메시지를 주고받습니다.

## OpenAMP란?

**OpenAMP (Open Asymmetric Multi-Processing)**는 비대칭 멀티프로세싱 시스템에서 코어 간 통신을 위한 오픈소스 프레임워크입니다.

### 주요 구성 요소
- **RPMsg (Remote Processor Messaging)**: 메시지 전달 프로토콜
- **VirtIO (Virtual I/O)**: 가상 I/O 디바이스 추상화
- **Resource Table**: 공유 리소스 정의

## 시스템 아키텍처

### 코어 역할
```
┌──────────────────┐              ┌──────────────────┐
│   Cortex-M7      │              │   Cortex-M4      │
│   (Master)       │◄────────────►│   (Remote)       │
│                  │   RPMsg      │                  │
│  - OpenAMP Host  │   Channel    │  - OpenAMP       │
│  - Message Init  │              │    Remote        │
│  - Counter++     │              │  - Counter++     │
└──────────────────┘              └──────────────────┘
         │                                │
         │         Shared Memory          │
         │       (D3-SRAM)                │
         ▼                                ▼
    ┌────────────────────────────────────────┐
    │  VirtIO Buffers (0x38000400)           │
    │  - TX Ring Buffer                      │
    │  - RX Ring Buffer                      │
    │  - Shared Data                         │
    └────────────────────────────────────────┘
         │                                │
         │         HSEM (Mailbox)         │
         └────────────────────────────────┘
```

## 메모리 맵

### D3 SRAM 레이아웃
```
0x38000000  ┌─────────────────────────┐
            │  Resource Table         │
            │  (CM4 리소스 정보)      │
0x38000400  ├─────────────────────────┤
            │                         │
            │  VirtIO Buffers         │
            │  (128 KB)               │
            │                         │
            │  - VirtIO Ring Buffers  │
            │  - Message Buffers      │
            │                         │
0x38020400  ├─────────────────────────┤
            │  Free D3 SRAM           │
0x38010000  └─────────────────────────┘
```

### Resource Table 구조
```c
#define RPMSG_MEM_BASE     0x38000400
#define RPMSG_MEM_SIZE     0x20000    // 128KB

struct remote_resource_table {
  unsigned int version;
  unsigned int num;
  unsigned int reserved[2];
  unsigned int offset[1];
  struct fw_rsc_vdev vdev;
  struct fw_rsc_vdev_vring vring0;
  struct fw_rsc_vdev_vring vring1;
};

__attribute__((section(".resource_table")))
struct remote_resource_table resource_table = {
  .version = 1,
  .num = 1,
  // VirtIO 설정...
};
```

## 통신 프로토콜

### 1. 초기화 시퀀스
```
CM7 (Master)                    CM4 (Remote)
    │                               │
    │ 1. HSEM 초기화                │
    │                               │
    │ 2. Shared Memory 초기화       │
    │                               │
    │ 3. OpenAMP Platform Init      │
    │                               │
    │ 4. CM4 Boot        ──────────►│
    │                               │ 5. Resource Table 읽기
    │                               │
    │                               │ 6. VirtIO 초기화
    │                               │
    │◄──── 7. RPMsg Ready ──────────│
    │                               │
    │ 8. Create RPMsg Channel       │
    │                               │
    │◄──── 9. Channel Created ──────│
    │                               │
    ▼                               ▼
  Ready                          Ready
```

### 2. 메시지 교환 (PingPong)
```
CM7                             CM4
 │                               │
 │ counter = 0                   │
 │                               │
 │ Send: counter++ (1) ─────────►│
 │                               │ Receive: 1
 │                               │ counter++
 │                               │
 │◄───────── Send: counter++ (2) │
 │ Receive: 2                    │
 │ counter++                     │
 │                               │
 │ Send: counter++ (3) ─────────►│
 │                               │ ...
 │                               │
 │ (100회 반복)                  │
 │                               │
 │ LED1 Toggle                   │
 ▼                               ▼
```

## 코드 구현

### CM7 (Master) 코드

#### 1. OpenAMP 초기화
```c
#include "openamp/open_amp.h"
#include "metal/device.h"

struct rpmsg_endpoint rp_endpoint;
struct rpmsg_device *rpmsg_dev;
static uint32_t message_count = 0;

int main(void)
{
  // 시스템 초기화
  HAL_Init();
  SystemClock_Config();

  // HSEM 초기화
  __HAL_RCC_HSEM_CLK_ENABLE();

  // OpenAMP 플랫폼 초기화
  MX_OPENAMP_Init(RPMSG_MASTER, NULL);

  // RPMsg 채널 생성
  OPENAMP_create_endpoint(&rp_endpoint, "rpmsg-client-sample",
                          RPMSG_ADDR_ANY, rpmsg_recv_callback, NULL);

  // CM4 부팅
  HAL_RCCEx_EnableBootCore(RCC_BOOT_C2);

  // 메시지 전송 시작
  while (message_count < 100)
  {
    // OpenAMP 서비스 처리
    OPENAMP_check_for_message();

    // 메시지 전송
    if (message_ready)
    {
      message_count++;
      OPENAMP_send(&rp_endpoint, &message_count, sizeof(message_count));
      message_ready = 0;
    }
  }

  // 100회 완료 - LED 토글
  BSP_LED_Toggle(LED1);

  while (1)
  {
    OPENAMP_check_for_message();
  }
}
```

#### 2. 메시지 수신 콜백
```c
int rpmsg_recv_callback(struct rpmsg_endpoint *ept, void *data,
                        size_t len, uint32_t src, void *priv)
{
  uint32_t *received_count = (uint32_t *)data;

  // 수신한 카운터 값 처리
  printf("CM7 received: %lu\n", *received_count);

  // 다음 메시지 준비
  message_ready = 1;

  return 0;
}
```

### CM4 (Remote) 코드

#### 1. 리소스 테이블 정의
```c
// resource_table.c
#define VRING_COUNT         2
#define VRING_SIZE          16
#define VRING_ALIGN         32

__attribute__((section(".resource_table")))
struct remote_resource_table resource_table = {
  .version = 1,
  .num = 1,
  .reserved = {0, 0},
  .offset = {offsetof(struct remote_resource_table, vdev)},

  // VirtIO device 설정
  .vdev = {
    .type = RSC_VDEV,
    .id = VIRTIO_ID_RPMSG,
    .notifyid = 0,
    .dfeatures = RPMSG_IPU_C0_FEATURES,
    .gfeatures = 0,
    .config_len = 0,
    .status = 0,
    .num_of_vrings = VRING_COUNT,
  },

  // VirtIO Ring 0 (TX)
  .vring0 = {
    .da = VRING_TX_ADDRESS,
    .align = VRING_ALIGN,
    .num = VRING_SIZE,
    .notifyid = 0,
  },

  // VirtIO Ring 1 (RX)
  .vring1 = {
    .da = VRING_RX_ADDRESS,
    .align = VRING_ALIGN,
    .num = VRING_SIZE,
    .notifyid = 1,
  },
};
```

#### 2. CM4 메인 코드
```c
int main(void)
{
  // HSEM 대기
  while(__HAL_HSEM_SEMID_GET(HSEM_ID_0) != 0);

  // HAL 초기화
  HAL_Init();

  // OpenAMP 초기화 (Remote)
  MX_OPENAMP_Init(RPMSG_REMOTE, NULL);

  // Endpoint 생성
  OPENAMP_create_endpoint(&rp_endpoint, "rpmsg-client-sample",
                          RPMSG_ADDR_ANY, rpmsg_recv_callback, NULL);

  // 메시지 대기 및 처리
  while (1)
  {
    OPENAMP_check_for_message();
  }
}
```

#### 3. CM4 수신 콜백
```c
int rpmsg_recv_callback(struct rpmsg_endpoint *ept, void *data,
                        size_t len, uint32_t src, void *priv)
{
  uint32_t *count = (uint32_t *)data;

  // 카운터 증가
  (*count)++;

  printf("CM4 received: %lu, sending back: %lu\n", (*count)-1, *count);

  // CM7로 다시 전송
  OPENAMP_send(ept, count, sizeof(*count));

  return 0;
}
```

## HSEM 인터럽트 처리

### HSEM을 통한 알림
```c
// CM7: 메시지 전송 후 CM4 알림
void OPENAMP_notify_callback(uint32_t id)
{
  // HSEM을 통해 CM4에 알림
  HAL_HSEM_ActivateNotification(__HAL_HSEM_SEMID_TO_MASK(HSEM_CH_OPENAMP));
}

// CM4: HSEM 인터럽트 핸들러
void HSEM1_IRQHandler(void)
{
  HAL_HSEM_IRQHandler();

  // 메시지 처리 플래그 설정
  message_received = 1;
}
```

## 링커 스크립트 설정

### CM4 링커 스크립트
```ld
/* Resource Table은 특정 주소에 위치 */
.resource_table 0x38000000 :
{
  . = ALIGN(4);
  *(.resource_table)
  . = ALIGN(4);
} >RAM_D3
```

## 성능 및 타이밍

### 메시지 전송 속도
- **왕복 시간**: 약 10-50 μs
- **처리량**: 최대 수만 msg/sec
- **오버헤드**: VirtIO + RPMsg 프로토콜

### 레이턴시 최적화
```c
// 폴링 모드 (낮은 레이턴시)
while (1)
{
  OPENAMP_check_for_message();  // 지속적으로 체크
}

// 인터럽트 모드 (낮은 CPU 사용)
while (1)
{
  if (message_received)
  {
    OPENAMP_check_for_message();
    message_received = 0;
  }
  __WFI();  // 인터럽트 대기
}
```

## 디버깅

### 메시지 추적
```c
// 디버그 출력 활성화
#define DEBUG_OPENAMP  1

#if DEBUG_OPENAMP
  #define OPENAMP_DEBUG(fmt, ...) printf("[OpenAMP] " fmt, ##__VA_ARGS__)
#else
  #define OPENAMP_DEBUG(fmt, ...)
#endif

// 사용
OPENAMP_DEBUG("Sending message: %lu\n", count);
```

### 공유 메모리 검사
```c
// Resource Table 확인
struct remote_resource_table *table = (struct remote_resource_table *)0x38000000;
printf("Resource Table Version: %u\n", table->version);
printf("VDev Type: %u\n", table->vdev.type);

// VirtIO Buffer 확인
uint8_t *vring = (uint8_t *)VRING_TX_ADDRESS;
// 데이터 검사...
```

## 트러블슈팅

### 메시지가 수신되지 않는 경우
1. **Resource Table 위치 확인**: 0x38000000
2. **HSEM 동작 확인**: 알림 메커니즘 작동 여부
3. **캐시 관리**: D3 SRAM은 non-cacheable이어야 함
   ```c
   // MPU 설정으로 D3 SRAM을 non-cacheable로 설정
   MPU_Region_InitTypeDef MPU_InitStruct;
   MPU_InitStruct.Enable = MPU_REGION_ENABLE;
   MPU_InitStruct.BaseAddress = 0x38000000;
   MPU_InitStruct.Size = MPU_REGION_SIZE_64KB;
   MPU_InitStruct.AccessPermission = MPU_REGION_FULL_ACCESS;
   MPU_InitStruct.IsBufferable = MPU_ACCESS_NOT_BUFFERABLE;
   MPU_InitStruct.IsCacheable = MPU_ACCESS_NOT_CACHEABLE;
   MPU_InitStruct.IsShareable = MPU_ACCESS_SHAREABLE;
   ```

### LED가 토글되지 않는 경우
- 메시지 카운트 확인 (100회 도달 여부)
- 양쪽 코어의 메시지 처리 로직 확인

## 확장 기능

### 1. 복수 채널
```c
// 두 개의 독립적인 RPMsg 채널 생성
struct rpmsg_endpoint ep_control;
struct rpmsg_endpoint ep_data;

OPENAMP_create_endpoint(&ep_control, "control-channel", ...);
OPENAMP_create_endpoint(&ep_data, "data-channel", ...);
```

### 2. 대용량 데이터 전송
```c
#define MAX_RPMSG_BUFF_SIZE  (512 - 16)  // VirtIO 헤더 제외

void send_large_data(uint8_t *data, uint32_t size)
{
  uint32_t offset = 0;
  while (offset < size)
  {
    uint32_t chunk_size = MIN(MAX_RPMSG_BUFF_SIZE, size - offset);
    OPENAMP_send(&rp_endpoint, data + offset, chunk_size);
    offset += chunk_size;
  }
}
```

### 3. RTOS 통합
OpenAMP_RTOS_PingPong 예제 참조 (FreeRTOS 사용)

## 참고 자료

- OpenAMP GitHub: https://github.com/OpenAMP/open-amp
- AN5617: Asymmetric multiprocessing with OpenAMP framework
- STM32H7 OpenAMP 미들웨어 사용자 매뉴얼
- 예제: `STM32H745I-DISCO/Applications/OpenAMP/OpenAMP_PingPong`
