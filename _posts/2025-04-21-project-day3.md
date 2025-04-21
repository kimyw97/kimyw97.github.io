---
layout: post
title: "2025.04.21 - FreeRTOS 기반 로봇 제어 설계 및 디버깅"
date: 2025-04-21 19:30:00 +0900
categories: kongji-patji devlog
---

## 📌 오늘 한 일
- FreeRTOS 기반 태스크 구조 설계
- 긴급 정지 이벤트 그룹 처리 설계
- vNavigationTask 내 LED 점멸 및 이벤트 대기 구현
- 브랜치 이름 변경 후 PR 생성 문제 확인

---

## 💻 구현/코드 정리

### ✅ vNavigationTask 코드

```c
void vNavigationTask(void *argument) {
  for (;;) {
    HAL_GPIO_TogglePin(GPIOD, LD4_Pin);  // LED 점멸 확인
    osDelay(500);

    setMotorSpeed('L', 50);
    setMotorSpeed('R', 50);

    EventBits_t bits = xEventGroupWaitBits(
      emegencyEventGroup,
      EmergencyOccure,
      pdTRUE,     // Clear on exit
      pdFALSE,    // Wait for ANY
      portMAX_DELAY
    );

    if (bits & EmergencyOccure) {
      motorShutdown();
    }

    osDelay(1);
  }
}
```
---

### ✅ 긴급 정지 처리 구조

- EventGroup 사용
- `EmergencyOccure` 비트가 설정되면 `motorShutdown()` 호출
- `pdTRUE` 옵션으로 이벤트를 소모하듯 사용

---

### ✅ Git 브랜치 이름 변경 후 PR 안 생기는 이슈
- `git branch -m`으로 로컬 이름 변경만 된 상태
- GitHub에 push 한 후 `origin/변경된브랜치`로 푸시 필요
- 예시:
  ```bash
  git push origin -u 새로운브랜치이름
  ```

---

## 🧩 이슈

- `xEventGroupWaitBits()` 사용 시 LED가 깜빡이지 않음 → FreeRTOS에서 HAL_Delay 사용시 무한 대기 상태로 빠짐
- `git branch -m` 만으로는 PR 생성 안 됨 → remote에 push 필요

---

## 🔜 내일 할 일

- `xEventGroupWaitBits()` 사용 구조 재검토
- 긴급정지 외 다른 센서/이벤트 추가 설계
- 타 태스크 설계 (ex. 센서, 환경 인식 등)

---

## 🪞 짧은 회고

- 단순히 동작만 확인할 게 아니라, RTOS 내부 동작 원리와 blocking/non-blocking 구조를 명확히 이해할 필요 있음
- 디버깅은 설정이 매우 중요하니 초기 세팅부터 꼼꼼히 잡자

---

## 📼 구현 영상

https://github.com/user-attachments/assets/2c6e25be-98ae-41d9-8a39-d92cef771bed

