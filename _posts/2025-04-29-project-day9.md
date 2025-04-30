---
layout: post
title: "2025.04.29 - 라즈베리파이, ROS2, 스테레오 비전 실습"
date: 2025-04-29 21:00:00 +0900
categories: robotics devlog
---

## ✅ 오늘 한 일

- [x] 라즈베리파이에서 스테레오 카메라 테스트 환경 구축
- [x] OpenCV 기반 스테레오 캘리브레이션 코드 테스트
- [x] YOLOv8 + Webcam 실시간 테스트 코드 작성
- [x] 쓰레기 객체 탐지를 위한 모델 리서치 및 TACO dataset 학습 모델 조사

## 🔍 주요 이슈 및 해결

### 1. OpenCV 스테레오 보정 실패 오류
- **문제**: `cv2.calibrateCamera()` 함수에서 `nimages > 0` 오류 발생
- **원인**: 체스보드 이미지를 제대로 인식하지 못해 objpoints, imgpointsL가 비어 있었음
- **해결**: 이미지 수집 방식 수정 예정, 밝기 및 각도 개선 필요

### 2. YOLOv8 실시간 테스트 코드 작성
```python
from ultralytics import YOLO
import cv2

model = YOLO("best.pt")
cap = cv2.VideoCapture(0)

while True:
    ret, frame = cap.read()
    if not ret:
        break

    results = model.predict(frame)
    annotated = results[0].plot()
    cv2.imshow("YOLO", annotated)

    if cv2.waitKey(1) & 0xFF == ord("q"):
        break

cap.release()
cv2.destroyAllWindows()

3. TACO dataset 관련 YOLO 모델 조사
	•	GitHub에서 TACO 기반으로 학습된 YOLOv8 모델 검색
	•	추후 직접 학습 또는 사전 학습 모델 fine-tuning 예정


📝 내일 할 일
	•	ROS2에서 stereo_node 실행 및 거리 측정 확인
	•	YOLO + 거리 정보 결합 로직 설계
	•	스테레오 보정 이미지 추가 수집 및 보정 재시도
	•	쓰레기 분류 모델 학습 데이터 정리
⸻
