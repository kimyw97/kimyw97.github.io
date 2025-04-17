---
layout: post
title: "2025.04.17 - kongji-patji 프로젝트 시작"
date: 2025-04-17 19:30:00 +0900
categories: kongji-patji devlog
---

## 📌 오늘 한 일
- 프로젝트 개발 환경 세팅
- stm32F4-discovery 보드 핀 매핑
- 모터 드라이버를 사용해 pwm 제어
- 기본 제어 함수 만들기

---

## 💻 구현/코드 정리

### ✅ 모터 드라이버 제어 코드
```c
void setMotorMode(DriveMode mode) {
  switch (mode) {
  case 0:
    HAL_GPIO_WritePin(GPIOD, Left_Motor_IN1_Pin, GPIO_PIN_SET);
    HAL_GPIO_WritePin(GPIOD, Left_Motor_IN2_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(GPIOD, Right_Motor_IN1_Pin, GPIO_PIN_SET);
    HAL_GPIO_WritePin(GPIOD, Right_Motor_IN2_Pin, GPIO_PIN_RESET);
    break;
  case 1:
    HAL_GPIO_WritePin(GPIOD, Left_Motor_IN1_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(GPIOD, Left_Motor_IN2_Pin, GPIO_PIN_SET);
    HAL_GPIO_WritePin(GPIOD, Right_Motor_IN1_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(GPIOD, Right_Motor_IN2_Pin, GPIO_PIN_SET);
    break;
    // 으론쪽으로 회전 => Left Motor 전진 , Right Motor 후진
  case 2:
    HAL_GPIO_WritePin(GPIOD, Left_Motor_IN1_Pin, GPIO_PIN_SET);
    HAL_GPIO_WritePin(GPIOD, Left_Motor_IN2_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(GPIOD, Right_Motor_IN1_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(GPIOD, Right_Motor_IN2_Pin, GPIO_PIN_SET);
    break;
    // 왼쪽으로 회전 => Left Motor 후진 , Right Motor 전진
  case 3:
    HAL_GPIO_WritePin(GPIOD, Left_Motor_IN1_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(GPIOD, Left_Motor_IN2_Pin, GPIO_PIN_SET);
    HAL_GPIO_WritePin(GPIOD, Right_Motor_IN1_Pin, GPIO_PIN_SET);
    HAL_GPIO_WritePin(GPIOD, Right_Motor_IN2_Pin, GPIO_PIN_RESET);
    break;
  case 4:
    HAL_GPIO_WritePin(GPIOD, Left_Motor_IN1_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(GPIOD, Left_Motor_IN2_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(GPIOD, Right_Motor_IN1_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(GPIOD, Right_Motor_IN2_Pin, GPIO_PIN_RESET);
    break;
  }
}

void setMotorSpeed(char motor_position, int speed) {
	if (motor_position == 'L') {
		__HAL_TIM_SET_COMPARE(&htim4, TIM_CHANNEL_2, speed * 10);
	} else if(motor_position == 'R'){
		__HAL_TIM_SET_COMPARE(&htim4, TIM_CHANNEL_3, speed * 10);
	}
}

void motorStart() {
	HAL_TIM_PWM_Start(&htim4, TIM_CHANNEL_2);
	HAL_TIM_PWM_Start(&htim4, TIM_CHANNEL_3);
}

void motorShutdown() {
	__HAL_TIM_SET_COMPARE(&htim4, TIM_CHANNEL_2, 0);
	__HAL_TIM_SET_COMPARE(&htim4, TIM_CHANNEL_3, 0);
}
```

### ✅ 회로 연결
- 9v dc 모터
- TIM2: PA0 (CH1), PA1 (CH2)
  
| 핀 이름     | 모드                                 | Pull 설정                  | 출력 상태 | 라벨                             |
|-------------|--------------------------------------|-----------------------------|------------|----------------------------------------|
| PD8         | Output Push Pull                     | No pull-up and no pull-down | Low        | Left_Motor_IN1                        |
| PD9         | Output Push Pull                     | No pull-up and no pull-down | Low        | Left_Motor_IN2                        |
| PD10        | Output Push Pull                     | No pull-up and no pull-down | Low        | Right_Motor_IN1                       |
| PD11        | Output Push Pull                     | No pull-up and no pull-down | Low        | Right_Motor_IN2                       |
| PA5     | TIM2_CH1    | Alternate Function Push Pull   | No pull-up and no pull-down | Low        | Left_Motor_Encoder_A         |
| PA1     | TIM2_CH2    | Alternate Function Push Pull   | No pull-up and no pull-down | Low        | Left_Motor_Encoder_B         |
| PB7     | TIM4_CH2    | Alternate Function Push Pull   | No pull-up and no pull-down | Low        | Left_Motor_PWM               |
| PB5     | TIM3_CH2    | Alternate Function Push Pull   | No pull-up and no pull-down | Low        | Right_Motor_Encoder_A        |
| PC6     | TIM3_CH1    | Alternate Function Push Pull   | No pull-up and no pull-down | Low        | Right_Motor_Encoder_B        |
| PB8     | TIM4_CH3    | Alternate Function Push Pull   | No pull-up and no pull-down | Low        | Right_Motor_PWM              |

---

## 🧩 이슈
- stm32cubeide가 너무 느려서 vscode를 사용할려고 세팅 => 오히려 빌드할때 왔다 갔다 하면서 더 불편함
- pwm 제어에 대해서 명확히 알 필요를 느낌, 공식화 해서 ((Clock)/(Prescaler))/Period를 외웠는데 막상 모터 제어에 적용 할려고 하니 그럼 속도가 뭐지? 생각이 들었음
---

## 🔜 내일 할 일
- PWM 마스터하기
- 엔코더 값 읽어서 거리 계산

---

## 🪞 짧은 회고
응용에 대해서만 이해하고 넘어가지 말고 정확한 정의와 개념을 암기하고 넘어가자

## 📼 구현 영상

https://github.com/user-attachments/assets/5467f75b-99ea-4690-bd86-aac582577668



