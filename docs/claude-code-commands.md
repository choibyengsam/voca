# Claude Code 실행 명령어 가이드
## 안심톡 프로젝트 개발용

---

## 🚀 프로젝트 시작 명령어

### 1단계: 프로젝트 초기화 (맨 처음 실행)

```
docs/claude-code-development-plan.md 파일을 읽고 안심톡 프로젝트를 시작해줘.

XAMPP 환경에서 PHP + MySQL 백엔드를 만들 거야.
프론트엔드는 Vue.js 3 + Capacitor 하이브리드 앱이야.

먼저 다음 작업을 해줘:
1. 프로젝트 폴더 구조 생성 (backend, frontend, mobile)
2. backend/sql/schema.sql 파일 생성
3. backend/composer.json 생성
4. backend/.env.example 생성
```

---

## 📁 Phase 1: 환경 설정

### 백엔드 기본 구조 생성

```
docs/claude-code-development-plan.md를 참고해서
backend 폴더의 기본 구조를 만들어줘.

필요한 파일들:
- api/index.php (API 라우터)
- config/database.php (DB 연결)
- config/app.php (앱 설정)
- utils/Database.php (PDO 싱글톤)
- utils/Response.php (JSON 응답 헬퍼)
- .htaccess (URL 리라이트)

XAMPP의 htdocs에서 실행할 수 있도록 해줘.
```

### 프론트엔드 초기화

```
frontend 폴더에 Vue.js 3 + Vite 프로젝트를 설정해줘.

필요한 작업:
1. package.json 생성 (vue, vue-router, pinia, axios)
2. vite.config.js 설정
3. capacitor.config.ts 설정
4. 기본 폴더 구조 (views, components, plugins, services, stores)
5. src/main.js, src/App.vue, src/router.js 기본 틀
```

---

## 🔧 Phase 2: 백엔드 API 개발

### 인증 API 개발

```
docs/claude-code-development-plan.md를 참고해서
인증 API를 개발해줘.

만들 파일들:
- api/controllers/AuthController.php
- api/models/User.php
- api/services/JWTService.php
- api/middleware/AuthMiddleware.php

API 엔드포인트:
- POST /api/auth/register (기기 등록)
- POST /api/auth/login (기기 인증)
- POST /api/auth/refresh (토큰 갱신)
- GET /api/auth/me (내 정보)

JWT 토큰 기반 인증으로 구현해줘.
```

### 체크인 API 개발

```
체크인 API를 개발해줘.

만들 파일들:
- api/controllers/CheckInController.php
- api/models/CheckIn.php
- api/models/UserStatus.php

API 엔드포인트:
- POST /api/checkin (체크인 기록)
- GET /api/checkin/history (히스토리 조회)
- GET /api/checkin/stats (통계)

체크인 타입: manual, auto_sensor, voice, widget
센서 데이터와 위치정보는 JSON으로 저장해줘.
```

### 긴급연락처 API 개발

```
긴급연락처 CRUD API를 개발해줘.

만들 파일:
- api/controllers/ContactController.php
- api/models/EmergencyContact.php

API 엔드포인트:
- GET /api/contacts (목록)
- POST /api/contacts (추가)
- PUT /api/contacts/{id} (수정)
- DELETE /api/contacts/{id} (삭제)

우선순위(priority) 필드로 알림 순서를 관리해줘.
```

### 가족 API 개발

```
가족 그룹 API를 개발해줘.

만들 파일들:
- api/controllers/FamilyController.php
- api/models/Family.php
- api/models/FamilyMember.php

API 엔드포인트:
- POST /api/family/create (가족 생성, 초대코드 발급)
- POST /api/family/join (초대코드로 참가)
- GET /api/family/members (멤버 목록)
- GET /api/family/dashboard (가족 상태 대시보드)

역할: admin, guardian, monitored
```

### 동기화 API 개발

```
앱에서 오프라인으로 저장된 데이터를 서버에 동기화하는 API를 만들어줘.

만들 파일:
- api/controllers/SyncController.php
- api/models/ActivityLog.php

API 엔드포인트:
- POST /api/sync/activities (센서 활동 로그 동기화)
- POST /api/sync/checkins (체크인 배치 동기화)

여러 개의 로그를 한 번에 받아서 처리해줘.
```

---

## 🎨 Phase 3: 프론트엔드 개발

### 온보딩 화면

```
Vue.js로 온보딩 화면을 만들어줘.

파일: frontend/src/views/Onboarding.vue

단계:
1. 환영 메시지
2. 이름 입력
3. 긴급연락처 등록 (전화번호)
4. 권한 요청 안내 (센서, SMS, 알림)
5. 완료

시니어 친화적으로:
- 큰 글씨 (20px 이상)
- 간단한 문구
- 큰 버튼
```

### 메인 체크인 화면

```
메인 체크인 화면을 만들어줘.

파일: frontend/src/views/Home.vue
컴포넌트: frontend/src/components/CheckInButton.vue

디자인:
- 화면 중앙에 큰 원형 버튼 (너비 60% 이상)
- 버튼 텍스트: "오늘도 괜찮아요"
- 색상: 초록색 계열 (안심 느낌)
- 체크인 완료 시 체크마크 애니메이션

표시 정보:
- 인사말 ("좋은 아침이에요, OOO님")
- 마지막 체크인 시간
- 연속 체크인 일수
- 연결된 가족 수

하단에 긴급 SOS 버튼 (5초 누르기)
```

### 설정 화면

```
설정 화면을 만들어줘.

파일: frontend/src/views/Settings.vue

설정 항목:
1. 내 정보 (이름, 연락처)
2. 체크인 시간 설정 (시간 선택)
3. 센서 감도 (낮음/중간/높음)
4. 알림 설정 (푸시, 카카오톡, SMS 토글)
5. 긴급연락처 관리 (목록, 추가/수정/삭제)
6. 가족 관리 (가족 코드, 탈퇴)

각 섹션을 카드 형태로 구분해줘.
```

### 체크인 기록 화면

```
체크인 기록 화면을 만들어줘.

파일: frontend/src/views/History.vue

기능:
1. 캘린더 뷰 (체크인한 날 표시)
2. 최근 체크인 목록 (시간, 타입)
3. 통계 카드
   - 이번 달 체크인 횟수
   - 연속 체크인 일수
   - 평균 체크인 시간
```

### 가족 대시보드

```
가족 대시보드 화면을 만들어줘.

파일: frontend/src/views/Dashboard.vue

기능:
1. 가족 멤버 카드 목록
   - 이름, 별명
   - 현재 상태 (정상/경고/긴급)
   - 마지막 체크인 시간
   - 연속 체크인 일수
2. 각 멤버에 전화하기 버튼
3. 가족 초대 코드 공유 버튼
```

---

## 📱 Phase 4: 네이티브 플러그인

### 센서 감지 플러그인

```
자이로센서/가속도계로 움직임을 감지하는 Capacitor 플러그인을 만들어줘.

파일들:
- frontend/src/plugins/motion-detector.js (JS 브릿지)
- mobile/android/app/.../plugins/MotionPlugin.java
- mobile/android/app/.../services/MotionService.java

기능:
1. 가속도계 + 자이로스코프 모니터링
2. 움직임 임계값 감지 (설정 가능)
3. 마지막 활동 시간 기록
4. 미활동 시간 체크 (4시간 → 경고, 12시간 → 심각, 24시간 → 긴급)
5. 백그라운드에서 실행

Android 포그라운드 서비스로 구현해줘.
```

### SMS 발송 플러그인

```
기기에서 직접 SMS를 발송하는 Capacitor 플러그인을 만들어줘.

파일들:
- frontend/src/plugins/sms-sender.js (JS 브릿지)
- mobile/android/app/.../plugins/SMSPlugin.java

기능:
1. Android: SmsManager로 직접 발송 (SEND_SMS 권한)
2. 다중 수신자 발송
3. 발송 결과 콜백
4. 긴급 메시지 템플릿

권한 요청 처리도 포함해줘.
```

### 백그라운드 서비스

```
앱이 종료되어도 계속 실행되는 백그라운드 서비스를 만들어줘.

파일들:
- frontend/src/plugins/background-service.js
- mobile/android/app/.../services/MotionService.java (포그라운드 서비스)

기능:
1. 센서 상시 모니터링
2. 주기적 서버 동기화 (10분)
3. 미활동 시 로컬 알림
4. 긴급 상황 시 SMS 자동 발송 트리거

Android에서 포그라운드 서비스 알림 표시해줘.
```

---

## 🔗 Phase 5: 통합

### API 클라이언트 설정

```
Axios 기반 API 클라이언트를 설정해줘.

파일: frontend/src/services/api.js

기능:
1. 기본 URL 설정 (환경변수)
2. JWT 토큰 자동 첨부 (인터셉터)
3. 토큰 만료 시 자동 갱신
4. 오프라인 에러 처리
5. 로딩 상태 관리
```

### 상태 관리 (Pinia)

```
Pinia 스토어를 설정해줘.

파일들:
- frontend/src/stores/user.js (사용자 정보, 인증)
- frontend/src/stores/checkin.js (체크인 상태)
- frontend/src/stores/settings.js (설정)
- frontend/src/stores/family.js (가족)

localStorage와 연동해서 앱 재시작 시에도 유지되게 해줘.
```

### 전체 통합 테스트

```
전체 앱 플로우를 테스트할 수 있도록 정리해줘.

테스트 시나리오:
1. 온보딩 → 이름/연락처 입력 → 완료
2. 메인 화면 → 체크인 버튼 클릭 → 서버 기록 확인
3. 센서 감지 → 자동 체크인 → 로그 확인
4. 미활동 4시간 → 경고 알림 확인
5. 미활동 24시간 → SMS 발송 확인
6. 가족 대시보드 → 상태 확인

각 단계별로 확인할 사항을 알려줘.
```

---

## 🛠️ 유용한 추가 명령어

### 버그 수정

```
[에러 메시지 또는 문제 설명]

위 문제를 해결해줘.
관련 파일: [파일 경로]
```

### 코드 리뷰

```
다음 파일들을 리뷰해줘:
- [파일 경로 1]
- [파일 경로 2]

보안 취약점, 성능 문제, 코드 품질을 체크해줘.
```

### 기능 추가

```
[기능 설명]을 추가해줘.

영향 받는 파일:
- [파일 경로]

기존 코드와 일관성 있게 작성해줘.
```

### 빌드 및 배포

```
Android APK 빌드를 위한 준비를 해줘.

1. frontend에서 npm run build
2. npx cap copy android
3. Android Studio에서 빌드할 수 있도록 설정 확인

필요한 권한들이 AndroidManifest.xml에 있는지 확인해줘:
- SEND_SMS
- BODY_SENSORS
- FOREGROUND_SERVICE
- RECEIVE_BOOT_COMPLETED
- INTERNET
```

---

## 📋 빠른 참조 카드

| 작업 | 명령어 시작 |
|------|------------|
| 프로젝트 시작 | "docs/claude-code-development-plan.md 파일을 읽고..." |
| 새 API 개발 | "[API명] API를 개발해줘. 엔드포인트: ..." |
| 새 화면 개발 | "[화면명] 화면을 만들어줘. 파일: ..." |
| 플러그인 개발 | "[기능명] Capacitor 플러그인을 만들어줘..." |
| 버그 수정 | "[에러 내용] 문제를 해결해줘..." |
| 코드 리뷰 | "다음 파일들을 리뷰해줘: ..." |

---

## ⚡ 원라인 명령어 (바로 복사용)

```bash
# 프로젝트 시작
docs/claude-code-development-plan.md 읽고 안심톡 프로젝트 폴더 구조 만들어줘

# DB 스키마
backend/sql/schema.sql 파일 만들어줘. 계획서의 스키마 참고해

# 인증 API
인증 API 만들어줘: register, login, refresh, me 엔드포인트. JWT 사용

# 체크인 API
체크인 API 만들어줘: POST /api/checkin, GET /api/checkin/history

# 메인 화면
Vue.js로 메인 체크인 화면 만들어줘. 큰 원형 버튼, 시니어 친화적으로

# 센서 플러그인
자이로센서 감지하는 Capacitor 플러그인 만들어줘. Android 먼저

# SMS 플러그인
기기에서 직접 SMS 보내는 Android 플러그인 만들어줘
```

---

*이 파일을 참고해서 Claude Code에 명령하세요!*
