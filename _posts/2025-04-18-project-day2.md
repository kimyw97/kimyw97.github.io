---
layout: post
title: "2025.04.18 - kongji-patji 프로젝트 2일차"
date: 2025-04-18 18:00:00 +0900
categories: kongji-patji devlog
---

## 📌 오늘 한 일
- 엔코더 값 읽어서 RPM 구하기
- 거리 기준으로 이동 후 정지 테스트

---

## 💻 구현/코드 정리
### 모터 부하 테스트
모터 성능이 소 로봇을 움직일 수 있을지 불확실해 테스트 해봄
결과는 최하단 영상 참고

### ✅ 모터 드라이버 제어 및 엔코더 값 읽기 코드
```c
int main() {
 // 생략
motorStart();
	motorShutdown();
	setMotorSpeed('L', 0);
	setMotorSpeed('R', 0);
	setMotorMode(FOWARD);
	HAL_Delay(3000);
	startEncoder(&htim3);
	startEncoder(&htim2);
	__HAL_TIM_SET_COUNTER(&htim3, 0);
	__HAL_TIM_SET_COUNTER(&htim2, 0);
	int8_t target_distance_cm = 100;
	int8_t PPR = 11;
	int8_t GEAR_RATIO = 30;
	float WHEEL_CIRCUMFERENCE_CM = 2 * 3.1415 * 3.5;
	setMotorSpeed('L', 60);
	setMotorSpeed('R', 60);
  /* USER CODE END 2 */

  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
	while (1) {
    /* USER CODE END WHILE */
    MX_USB_HOST_Process();

    /* USER CODE BEGIN 3 */

		int32_t left_pulse = readEncoder(&htim2);
		int32_t right_pulse = readEncoder(&htim3) * -1;
		int32_t left_rotation = left_pulse / (float) (PPR * GEAR_RATIO);
		int32_t right_rotation = right_pulse / (float) (PPR * GEAR_RATIO);
		int8_t left_distance = (int8_t) (left_rotation * WHEEL_CIRCUMFERENCE_CM);
		int8_t right_distance = (int8_t) (right_rotation
				* WHEEL_CIRCUMFERENCE_CM);
		int8_t distance_cm = (left_distance + right_distance) / 2.0;
		if (left_pulse > right_pulse) {
			// 왼쪽이 더 빨라 → 속도 줄이기
			setMotorSpeed('L', 60 - 10);
			setMotorSpeed('R', 60);
		} else if (left_pulse < right_pulse) {
			// 오른쪽이 더 빨라 → 속도 줄이기
			setMotorSpeed('L', 60);
			setMotorSpeed('R', 60 - 10);
		} else {
			// 같으면 유지
			setMotorSpeed('L', 60);
			setMotorSpeed('R', 60);
		}

		if (distance_cm >= target_distance_cm) {
			setMotorSpeed('L', 0);
			setMotorSpeed('R', 0);
			break;
		}
	}
}

void startEncoder(TIM_HandleTypeDef *htim) {
	HAL_TIM_Encoder_Start(htim, TIM_CHANNEL_ALL);
}

int16_t readEncoder(TIM_HandleTypeDef *htim) {
	return (int16_t) __HAL_TIM_GET_COUNTER(htim);
}

/* MF = Multiplication Factor
 * method = M or T
 */
int16_t calRPM(char method, int8_t MT, int16_t encoder_count, float time,
		int8_t PPR, int8_t ratio) {
	int16_t result;
	float temp;
	if (method == 'M') {
		temp = (60 * encoder_count) / (time * ratio * PPR * MT);
		result = (int) temp;
	} else {
		// T Method
		temp = 60 / (time * PPR * MT);
		result = (int) temp;
	}
	return result;
}
```

### ✅ JGB37-520 모터 및 엔코더 학습
1. JGB37-520 스펙
- 엔코더가 부착된 dc 모터

- 12v 전원과 엔코더 전원인 5v~3.3v를 입력으로 받음

- 일반적은 12v는 pwm제어를 통해 제공(M1,M2 Pin)하고 엔코더 전원은 모터 핀(GND, VCC)에 연결
11PPR * 2로 22PPR

2. 엔코더 학습
- PPR(Pulse Per Revolution)
회전당 펄스, 한바퀴에 발생하는 펄스의 개수

- 모터 회전수 구하는 방법
기어 모터의 경우 기어비(Ratio)가 필요
기어비가 제공 안되었을때 계산 실은 기어비(ratio) = 모터 본체의 무부하 속도 / 출력측 속도
예시의 경우 33 ratio 
1바퀴 ⇒ 33 * 11 = 343개 펄스 발생

- 엔코더 원리
회전 디스크에 다수의 슬릿을 배치(구멍 뚫어 놓음)하고 발광 소자에서 빛 발생 시켜 회전 디스크 반대쪽에서 수광 소자를 사용해 빛이 투과되면 전기 신호를 발생 시킴

- 엔코더 A, B 채널
일반적인 엔코더는 채널이 두개, 이유는 시계 방향과 반시계를 구분하기 위해서
예) A채널이 0 → 1(rising) 일때 B채널이 0 이거나 B 채널이 rising 일때 A 채널이 0 이면 시계 방향

- 체배
A rising, falling, B rising, falling 중 몇개를 사용해 각도 측정을 할 것 인지에 대한 개ㅕㅁ
4개중 하나만 사용하면 1체배
체배가 높을 수록 각도 계산이 정밀해 짐 ⇒ 거리 계산의 오차가 적다
ex)
1개의 edge를 사용한다 ⇒ 1회전 할 때 11개 펄스 발생 ⇒ 11 edge 발생 ⇒ 1 edge 당 각도 32.73
4개의 edge를 사용한다 ⇒ 1회전 할 때 11개 펄스 발생 ⇒ 44 edge 발생 ⇒ 1edge 당 각도 8.18

- 속도 계산 방식
M 방식 : 1초동안 몇 바퀴 회전 했는가로 RPM 계산 ⇒ RPM = (60 * edge) / (t *PPR * 체배)
T 방식: 신호가 들어오는 시간 간격을 기준으로 RPM 계산 ⇒ RPM = 60 / (PPR * 신호간격 * 체배)


## 🧩 이슈
- 엔코더 값을 읽어 오는 것 까지는 문제가 없었으나 RPM 계산이 말도 안되게 높게 나옴 => 
PPR을 축 기준이 아닌 모터 PPR만 생각해서 크게 나온거 였음 => 모터 기어비를 적용하니 정상적으로 나옴
- 모터를 바꾸면서 방향 제어가 처음과 달라짐 => 추후에 설계가 완료되면 다시 잡는게 좋을 듯
- 엔코더 값을 기준으로 일정 거리 이상 움직이면 멈추도록 구현 => 멈추긴하나 자꾸 좌회전 하는 이슈가 발생
---

## 🔜 내일 할 일
- PWM 마스터하기 => 오늘 못해서 내일에 꼭하기
- 피드백 제어로 방향 직진 하도록 변경

---

## 🪞 짧은 회고
기본 원리에 대해서 확실히 이해하고 가니 작업이 오히려 빨라지는 것을 경험했다. 다시 한번 다짐, 암기하기!

## 📼 구현 영상
1. 부하 테스트

https://github.com/user-attachments/assets/aa14f56c-44e4-4b22-916a-07572bec5619

2. 거리 기준 이동


https://github.com/user-attachments/assets/b6c6ea6e-b466-4cee-9555-4f82b8019773





