
---
layout: post
title: "2025.04.25 - STS3032 서보모터 제어 및 시리얼 통신 디버깅"
date: 2025-04-25 21:00:00 +0900
categories: stm32 servo uart
---

## 📌 오늘 한 일

- STS3032 서보모터의 각도 → 제어값 실측 결과 확인
- `WritePos`, `RegWritePos`, `SyncWritePos` 명령 차이 정리
- URT-1 보드의 DTR 핀 역할 및 통신 시 고려사항 확인
- STS3032 제어를 위한 FreeRTOS 기반 큐 처리 구조 확인
- 큐 구조에서 `itemSize` 설정 오류 디버깅
- C에서 C++로 프로젝트 이전 시 유의 사항 학습
- 아두이노용 STS3032 서보 제어 코드를 STM32에서 사용 가능하도록 포팅

---

## ⚙️ STS3032 제어값 실측 결과

공식 문서에서는 0~4095 범위가 전체 회전 범위(0~360°)라 설명되어 있으나, 실제 테스트 결과:

- `0` → 시작 위치
- `15` → 360도 (1바퀴)
- `30` 이상 → 2바퀴 이상 회전하며, 일정 값(예: 720) 이상 시 **오버플로우** 발생 가능

> **주의:** 720 이상 값을 `SyncWritePos`로 두 모터에 동시에 보낼 경우, 하나만 동작하는 문제가 발생함.

```c
// 예: 1바퀴 회전
STS3032_WritePosition(1, 15, &huart4);
```

---

## 🔁 STS3032 명령어 차이점

| 명령어        | 설명 |
|---------------|------|
| `WritePos`    | 단일 모터에 바로 명령 실행 |
| `RegWritePos` | 단일 모터에 예약 실행 (Action 명령 필요) |
| `SyncWritePos`| 여러 모터에 동시에 명령 실행 (오버플로우 주의)

```c
// SyncWritePos 예제
servo.SyncWritePos(0xFE, posArray, speedArray, accArray, count);
```

---

## 🔌 URT-1 DTR 핀의 역할

- DTR 핀은 FTDI나 USB-UART 브리지에서 MCU의 리셋 제어나 통신 초기화 신호로 사용
- STM32와 URT-1 사용 시, 해당 핀 설정 여부에 따라 시리얼 통신이 정상 작동하지 않을 수 있음
- `DTR` 미설정 시 TX 송신 불가 사례 있음 → 핀 연결 및 설정 확인 필요

---

## 🧵 FreeRTOS + UART 큐 처리 구조

- UART 인터럽트에서 문자열을 수신 후, FreeRTOS 큐에 `CommandMessage`로 전송
- 큐 생성 시 `itemSize`를 `sizeof(CommandMessage)`로 정확히 설정해야 함
- `uint8_t`로 생성 시 데이터 손실 발생 가능

```c
xQueueCreate(10, sizeof(CommandMessage)); // 정확한 큐 생성
```

---

## 🔧 C → C++로 프로젝트 이전 시 주의

- 기존 `main.c` 대신 `main.cpp`로 파일 확장자 변경 필요
- C++에서는 `extern "C"` 선언 필수 (특히 HAL 라이브러리 함수 포함 시)

```cpp
extern "C" {
#include "stm32f4xx_hal.h"
}
```

---

## 🛠 STM32용 서보 제어 코드 포팅 (Arduino → STM32)

아, 완벽히 이해했어! 네가 아두이노용으로 사용하던 **헤더파일과 소스코드**를 STM32에서 사용 가능하도록 변경한 내용을 블로그에 추가하고 싶다는 거지? 그럼 "STM32용 서보 제어 코드 포팅" 항목에 **Arduino 헤더 및 소스코드 포팅**이라는 이름으로 예시 포함해서 정리해줄게.

---

### 🔧 STM32용 서보 제어 코드 포팅 (Arduino 헤더/소스 → STM32)

#### ✅ 원본 Arduino 헤더 (`SCSCL.h`)
```cpp
#ifndef _SCSCL_H
#define _SCSCL_H

#include <Arduino.h>
class SCSCL {
public:
    void WritePos(uint8_t ID, uint16_t Position, uint16_t Speed, uint8_t ACC);
    void SyncWritePos(uint8_t IDs[], uint16_t Positions[], uint8_t Count);
    ...
};
#endif
```

#### ✅ STM32용 헤더 (`SCSCL.hpp`)
```cpp
#ifndef SCSCL_HPP
#define SCSCL_HPP

#include "stm32f4xx_hal.h"

class SCSCL {
private:
    UART_HandleTypeDef* huart;

public:
    SCSCL(UART_HandleTypeDef* huart);
    void WritePos(uint8_t ID, uint16_t Position, uint16_t Speed, uint8_t ACC);
    void SyncWritePos(uint8_t IDs[], uint16_t Positions[], uint8_t Count);
};

#endif
```

#### ✅ 원본 Arduino 구현부 (`SCSCL.cpp`)
```cpp
void SCSCL::WritePos(uint8_t ID, uint16_t Position, uint16_t Speed, uint8_t ACC) {
    // Arduino Serial 사용
    Serial.write(...);
}
```

#### ✅ STM32용 구현부 (`SCSCL.cpp`)
```cpp
#include "SCSCL.hpp"

SCSCL::SCSCL(UART_HandleTypeDef* huart) {
    this->huart = huart;
}
```

> 💡 `Serial.write()` → `HAL_UART_Transmit()`으로 변경  
> 💡 클래스에서 `UART_HandleTypeDef*`을 멤버로 관리  
> 💡 포팅 시 `delay()` 대신 `HAL_Delay()` 또는 RTOS `osDelay()`로 대체 필요
---
## 📼 동작 영상

https://github.com/user-attachments/assets/dc6a11c7-e280-4711-acbe-2541225a0d6c

