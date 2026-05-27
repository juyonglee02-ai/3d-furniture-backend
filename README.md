# 3d-furniture-backend
3D 모델링 ai 을 이용한 가구 인테리어(학교 프로젝트)

# 🪐 3D Space - Backend Server

> **YOLOv8 기반 객체 인식 및 rembg를 활용한 3D 가구 배치 웹 서비스의 백엔드 서버 시스템입니다.**
> 대용량 이미지 처리와 3D 모델링 연산의 부하를 줄이기 위해 **웹 서버와 AI/렌더링 워커 서버를 분리하고 비동기 큐를 도입하여 안정성을 극대화**했습니다.

---

## 🛠 1. Tech Stack (기술 스택)

### Backend & Framework
- **Java 17 / Spring Boot 3.x** (또는 Python / FastAPI): 메인 웹 API 서버 구축 및 비즈니스 로직 처리
- **Python 3.10**: YOLOv8 및 3D 모델링(Mesh 생성) 연산 엔진 구성

### Database & Caching
- **MySQL 8.0**: 회원, 웹 데이터, 메타데이터 관리 (영속성 확보)
- **Redis**: 사용자 세션 관리 및 무거운 3D 객체 데이터 캐싱을 통한 조회 성능 개선

### DevOps & Infrastructure
- **AWS EC2 / S3**: 인프라 호스팅 및 3D 모델 파일(.obj, .gltf) 저장소 활용
- **Docker / Docker Compose**: 개발 환경 동기화 및 컨테이너 기반 배포 자동화
- **RabbitMQ (또는 Redis Queue)**: 웹 서버와 AI/3D 연산 서버 간의 비동기 작업 메시징 처리

---

## 🏗 2. System Architecture (시스템 아키텍처)

*백엔드 중심의 데이터 흐름과 서버 구조를 시각화한 다이어그램을 첨부하세요.*
*(팁: Draw.io나 미리캔버스 등으로 구조도를 그려 이미지로 넣으면 효과가 배가됩니다.)*

- **서버 분리 전략**: HTTP 요청을 처리하는 웹 서버와, CPU/GPU 소모가 큰 3D/AI 연산 서버를 분리하여 대량의 요청에도 웹 서비스가 다운되지 않도록 설계했습니다.
- **비동기 작업 큐**: 사용자가 3D 변환을 요청하면 즉시 응답을 반환(202 Accepted)하고, 백그라운드(Queue)에서 연산을 처리한 뒤 완료 시 알림을 보내는 방식으로 UX와 서버 안정성을 모두 잡았습니다.

---

## 💾 3. Database ERD (데이터베이스 구조)

*회원, 프로젝트(공간), 3D 에셋(가구), 로그 테이블 간의 관계를 보여주는 ERD 이미지를 첨부하세요.*

- **성능 최적화 포인트**: 3D 에셋의 메타데이터(크기, 좌표, 파일 경로 등)를 효율적으로 조회하기 위해 `Asset` 테이블에 인덱스(Index)를 적용하여 검색 속도를 기존 대비 O% 개선했습니다.

---

## ✨ 4. Key Features & Backend Implementation (주요 구현 기능)

### 1) 대용량 3D 에셋 업로드 및 AWS S3 연동
- 파일 업로드 시 버퍼 메모리 오버플로우를 방지하기 위해 **MultipartFile** 스트림 처리 적용
- S3 업로드 완료 후 다운로드 보안을 위해 **Pre-signed URL**을 발급하여 보안성 강화

### 2) 무거운 연산 처리를 위한 비동기(Async) 아키텍처 구축
- 3D 모델링 파일 생성 알고리즘이 웹 요청 쓰레드를 차단(Blocking)하지 않도록 **Spring `@Async` 및 메시지 큐** 도입
- 연산 서버의 상태를 체크하는 헬스 체크(Health Check) API 구현

### 3) RESTful API 설계 및 문서화
- HTTP Method와 상태 코드를 준수하여 직관적인 API 설계
- **Swagger (OpenAPI 3.0)**를 도입하여 프론트엔드 개발자와의 협업 효율성 증대

---

## 🚀 5. Troubleshooting & Optimization (트러블슈팅 - 핵심 ⭐️)

### 🔴 문제 1: 3D 모델 변환 요청 시 웹 서버 타임아웃(Timeout) 및 쓰레드 풀 고갈 현상
- **원인**: 하나의 이미지에서 3D 오브젝트를 추출하는 데 약 10~15초가 소요됨에 따라, 사용자의 HTTP 요청이 동기(Synchronous) 방식으로 대기하면서 Spring Boot의 Tomcat 쓰레드 풀이 모두 고갈되어 다른 사용자들의 접속이 불가능해짐.
- **해결**: 요청 구조를 **비동기 이벤트 기반(Event-driven)**으로 전면 개편. 사용자의 요청이 들어오면 메세지 큐(RabbitMQ)에 작업을 적재하고 즉시 작업 ID를 반환. AI 워커 서버가 큐에서 작업을 가져가 처리한 후 완료되면 웹 서버로 콜백(Callback)을 주는 구조로 변경.
- **결과**: 서버 타임아웃 에러 100% 해결, 동시 요청 처리 처리량(Throughput) 대폭 향상.

### 🔴 문제 2: Linux 서버 환경에서 3D 라이브러리 및 파일 권한 에러 (Permission Denied)
- **원인**: 로컬(Windows/Mac) 환경에서는 정상 작동했으나, AWS Ubuntu 서버로 이관 후 3D 파일 저장을 위한 SFTP 경로 설정 및 특정 C-based 3D 그래픽 라이브러리 의존성 결핍으로 에러 발생.
- **해결**: 환경에 구애받지 않도록 OS 의존성 라이브러리를 포함한 **Docker 이미지 구축**. `Dockerfile` 내부에서 파일 쓰기 권한(`chmod 755`)을 명시적으로 부여하고 가상 컨테이너 환경에서 실행하도록 통일.
- **결과**: 배포 환경 구축 시간 단축 및 환경 차이로 인한 런타임 에러 완전 제거.

---

## 📑 6. API Documentation (API 명세 일부)

| Method | URL | Description | Request Body / Param | Response Status |
| :--- | :--- | :--- | :--- | :--- |
| **POST** | `/api/v1/models/convert` | 이미지 3D 모델링 변환 요청 | `{ "image_url": "..." }` | `202 Accepted` |
| **GET** | `/api/v1/models/status/{id}` | 변환 작업 상태 조회 | PathVariable: `id` | `200 OK` |
| **GET** | `/api/v1/assets/{id}` | 생성된 3D 에셋 데이터 조회 | PathVariable: `id` | `200 OK` |
