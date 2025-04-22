---
layout: post
title: "2025.04.22 - STM32 + FreeRTOS UART 통신 구조 및 디버깅"
date: 2025-04-22 18:00:00 +0900
categories: embedded freertos stm32 uart
---

## 📌 오늘 한 일 요약
- UART 인터럽트 수신 → FreeRTOS 큐로 전달 → 파싱 태스크 구현
- UTM Ubuntu에서 STM32 UART 연결 문제 해결

---

## 💻 구현 및 디버깅 정리

### ✅ 1. FreeRTOS 기반 UART 수신 구조

**목표**: 라즈베리파이에서 들어오는 문자열(`L100R120\n`)을 STM32가 UART 인터럽트로 받고, 큐를 통해 파싱 태스크로 전달해 모터 제어.

#### ISR에서 큐에 데이터 전달

```c

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
	if (huart->Instance == UART5) {
		BaseType_t xHigherPriorityTaskWoken = pdFALSE;

		// 큐에 수신 바이트 삽입
		xQueueSendFromISR(uartQueue, &rx_data, &xHigherPriorityTaskWoken);

		// 다시 수신 시작
		HAL_UART_Receive_IT(&huart5, &rx_data, 1);

		// 필요 시 context switch
		portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
	}
}
```

#### UART 파싱 태스크 (FreeRTOS Task)

```c
void StartDefaultTask(void *argument) {
	/* init code for USB_HOST */
	MX_USB_HOST_Init();
	/* USER CODE BEGIN 5 */
	/* Infinite loop */
	char buffer[RX_BUFFER_SIZE];
	int index = 0;

	uint8_t byte;
	HAL_GPIO_TogglePin(GPIOD, LD3_Pin);
	osDelay(500);
	for (;;) {
		if (xQueueReceive(uartQueue, &byte, 1) == pdTRUE) {
			if (byte == '\n') {
				buffer[index] = '\0';
				parseCommand(buffer);  // 예: "L100R120"
				index = 0;
			} else {
				if (index < RX_BUFFER_SIZE - 1) {
					buffer[index++] = byte;
				} else {
					index = 0; // overflow 방지
				}
			}
		}
	}
	/* USER CODE END 5 */
}
```
### serial 파싱하고 모터 pwm 제어
```c
void parseCommand(char *cmd) {
	int left_pwm = 0, right_pwm = 0;

	char *l_ptr = strchr(cmd, 'L');
	char *r_ptr = strchr(cmd, 'R');

	if (l_ptr && r_ptr) {
		left_pwm = atoi(l_ptr + 1);   // 'L' 다음부터 정수로 변환
		right_pwm = atoi(r_ptr + 1);  // 'R' 다음부터 정수로 변환

		// PWM 제한 범위 적용
		if (left_pwm > 255)
			left_pwm = 255;
		if (left_pwm < -255)
			left_pwm = -255;
		if (right_pwm > 255)
			right_pwm = 255;
		if (right_pwm < -255)
			right_pwm = -255;

		if (left_pwm > 0 & right_pwm > 0) {
			setMotorMode(FORWARD);

		} else if (left_pwm < 0 & right_pwm < 0) {
			setMotorMode(BACKWARD);

		} else if (left_pwm > 0 & right_pwm < 0) {
			setMotorMode(ROTATE_RIGHT);

		} else if (left_pwm < 0 & right_pwm > 0) {
			setMotorMode(ROTATE_LEFT);

		} else {
			setMotorMode(STOP);
		}

		setMotorSpeed('L', left_pwm);
		setMotorSpeed('R', right_pwm);
	}
}
```
---

### ✅ 2. UTM Ubuntu ↔ STM32 USB 연결 문제 해결

**현상**: Mac에서는 minicom으로 UART 통신이 잘 되는데, UTM 우분투에서는 `ttyUSB0` 장치가 생기지 않음.

**원인 및 해결**:
- UTM VM에서 USB 3.0으로 passthrough 되어 있었음
- STM32 USB-UART 칩셋(CH341 등)이 USB 2.0을 선호함
- 👉 USB 버전을 2.0으로 바꾸자 바로 `ttyUSB0` 연결 성공!

```bash
$ dmesg | grep tty
[...]
usb 5-3: ch341-uart converter now attached to ttyUSB0
```
---

## 📝 느낀 점

- UART 수신 → 큐 → 파싱 구조가 잘 동작함
- RTOS 사용할 때 인터럽트를 안 쓸 줄 알았는데 인터럽트와 큐를 사용하고 테스크에서 처리하는 방식이 더 효율적인 것 같다

# 다음 목표
- turtlebot3 telop node에서 발행되는 cmd_vel/ 토픽을 시리얼로 전달하는 node 사용해서 telop으로 제어해보기
- 모터 하중 부하로 pwm에 30을 주면 모터가 정지, 이 이상으로 구현하는 로직 추가
- 
# 동작 영상
  https://github.com/user-attachments/assets/980a514c-15f2-49e6-baa3-24253e5672b4
