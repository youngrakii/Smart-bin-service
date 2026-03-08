# Smart Bin Service

IoT 스마트 쓰레기통의 펌웨어(센서 수집 + 움직임 감지 AI 모델) - 백엔드(저장/조회) - 프론트엔드(모니터링)를 한 저장소에 모아둔 프로젝트입니다.
<p align="center">
  <img src="https://github.com/user-attachments/assets/5f5958ad-1772-450f-8a82-d11e7324540d" height="300" />
  <img src="https://github.com/user-attachments/assets/e4b79caf-bd70-4017-bb9f-e15c0624e92a" height="300" />
</p>


## 1. 프로젝트 개요

- `iotEdge`: ESP32/Nano 기반 센서 수집, MQTT 발행, 모션 기반 GPS 게이팅
- `sensor-server`: Spring Boot + PostgreSQL + MQTT 구독 + REST API
- `dashboard-frontend`: React + Vite 대시보드/지도/목록 UI

데이터 흐름:

1. 디바이스가 MQTT 토픽 `Bin/test/data`로 센서 JSON 발행
2. `sensor-server`가 구독 후 DB(`sensor_reading`)에 저장
3. `dashboard-frontend`가 REST API로 최신 상태 조회/시각화
<img width="505" height="665" alt="image" src="https://github.com/user-attachments/assets/6baf92fd-3305-4640-8a6f-0d5f61b6cac4" />


## 2. 디렉터리 구조

```text
.
|- dashboard-frontend/    # React + TypeScript + Vite UI
|- sensor-server/         # Spring Boot 4 + JPA + PostgreSQL + MQTT
|- iotEdge/               # ESP32/Nano 펌웨어 + Edge Impulse 모델
`- README.md
```

## 3. 기술 스택

- Frontend: React 18, TypeScript, Vite
- Backend: Java 17, Spring Boot 4.0.0, Spring Data JPA, Eclipse Paho MQTT
- Database: PostgreSQL
- Edge: ESP32, Arduino Nano 33 BLE Sense, VL53L5CX, HX711, 누수 센서, GPS(TinyGPS)
- AI: Edge Impulse (`iotEdge/motion_detect_iot_bin_inferencing`)

## 4. 빠른 실행 (로컬)

권장 실행 순서:

1. `sensor-server` 실행
2. `dashboard-frontend` 실행
3. 디바이스(`iotEdge`) 전원 인가 및 MQTT 발행 확인

### 4.1 Backend 실행 (`sensor-server`)

사전 요구사항:

- JDK 17
- PostgreSQL 접근 정보

실행:

```bash
cd sensor-server
./gradlew bootRun
```

Windows PowerShell:

```powershell
cd sensor-server
.\gradlew.bat bootRun
```

Docker:

```bash
cd sensor-server
docker build -t sensor-server .
docker run -p 8080:8080 --env-file .env sensor-server
```

기본 포트: `8080`

### 4.2 Frontend 실행 (`dashboard-frontend`)

사전 요구사항:

- Node.js 18+

실행:

```bash
cd dashboard-frontend
npm install
npm run dev -- --host
```

기본 개발 서버: Vite 기본값(`5173`)

## 5. 환경변수

### 5.1 Frontend (`dashboard-frontend/.env`)

```env
VITE_API_BASE_URL=http://localhost:8080
VITE_KAKAO_MAP_KEY=<KAKAO_JAVASCRIPT_KEY>
```

### 5.2 Backend (`sensor-server`)

`application.properties` 기본값을 환경변수로 덮어쓸 수 있습니다.

- `SPRING_DATASOURCE_URL`
- `SPRING_DATASOURCE_USERNAME`
- `SPRING_DATASOURCE_PASSWORD`
- `PORT` (기본 `8080`)
- `APP_MQTT_BROKER_URL` (기본 `tcp://165.246.44.72:2025`)
- `APP_MQTT_CLIENT_ID` (기본 `sensor-server-backend`)
- `APP_MQTT_TOPIC` (기본 `Bin/test/data`)
- `APP_MQTT_DEFAULT_BIN_ID` (기본 `Bin-Master`)

## 6. API 스펙 (`sensor-server`)

Base URL: `http://localhost:8080`

### 6.1 최신 목록 조회

`GET /api/readings?binId=<id>&limit=50`

- `binId` 누락 시 `400 Bad Request`
- 최대 `500`개까지 제한

### 6.2 최신 1건 조회

`GET /api/readings/latest?binId=<id>`

- 데이터 없으면 `204 No Content`

응답 필드:

- `id`
- `binId`
- `distanceMm`
- `weightG`
- `waterAdc`
- `needCollection`
- `lat` (optional)
- `lng` (optional)
- `createdAt`

## 7. MQTT 페이로드

토픽: `Bin/test/data`

예시:

```json
{
  "bin_id": "Bin-Master",
  "distance_mm": 120,
  "weight_g": 180.5,
  "water_adc": 420,
  "need_collection": false,
  "lat": 37.45,
  "lng": 126.65
}
```

참고:

- `lat/lng`는 상황에 따라 생략될 수 있음
- `bin_id`가 비어 있으면 서버가 `APP_MQTT_DEFAULT_BIN_ID`로 대체

## 8. iotEdge 구성 요약

- `iotEdge/Master_bin/Master_bin.ino`
  - ToF/HX711/누수 센서 값을 1초 주기로 MQTT 발행
  - 임계값 조건으로 `need_collection` 계산
- `iotEdge/Remote_bin/Remote_bin.ino`
  - Nano에서 `MOVING` UART 신호 수신 시 GPS 활성화
  - GPS 신호가 유효/신선할 때만 `lat/lng` 포함 발행
- `iotEdge/nano_motion_detect/nano_motion_detect.ino`
  - Edge Impulse 추론 결과 moving 점수(평균)가 임계값(0.7) 이상이면 `MOVING` 전송
- `iotEdge/EachSensor_testCode/`
  - 센서별 단독 테스트 스케치
- `iotEdge/motion_detect_iot_bin_inferencing/`
  - Edge Impulse 라이브러리/모델/예제

## 9. 운영 전 체크리스트

- 펌웨어 내 Wi-Fi SSID/비밀번호, MQTT 주소를 실제 환경 값으로 교체
- DB 계정/비밀번호를 환경변수로 분리
- Frontend `VITE_API_BASE_URL`과 Backend 포트/도메인 일치 여부 확인
- CORS/보안 설정(토큰, 방화벽, 비밀정보 분리) 점검



