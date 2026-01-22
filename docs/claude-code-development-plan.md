# 안심톡 (SafeGuard) 개발 계획서
## Claude Code 실행용 - XAMPP + PHP + MySQL

---

## 🎯 프로젝트 개요

### 목표
독거노인 및 1인 가구를 위한 생사 확인 앱 개발
- **자동 생존 확인**: 자이로센서/가속도계로 움직임 감지
- **기기 직접 SMS**: 서버 경유 없이 스마트폰에서 직접 발송
- **카카오톡 연동**: 알림톡으로 가족에게 안부 전달

### 기술 스택
```
Backend:  PHP 8.x + MySQL 8.x (XAMPP)
Frontend: HTML5 + CSS3 + JavaScript (Vue.js 3)
Mobile:   Capacitor 5.x (하이브리드 앱)
API:      RESTful JSON API
```

---

## 📁 프로젝트 구조

```
ansimtalk/
├── 📁 backend/                    # PHP 백엔드 (XAMPP htdocs에 배치)
│   ├── 📁 api/
│   │   ├── index.php             # API 라우터
│   │   ├── 📁 controllers/
│   │   │   ├── AuthController.php
│   │   │   ├── CheckInController.php
│   │   │   ├── FamilyController.php
│   │   │   ├── ContactController.php
│   │   │   ├── NotificationController.php
│   │   │   └── SyncController.php
│   │   ├── 📁 models/
│   │   │   ├── User.php
│   │   │   ├── CheckIn.php
│   │   │   ├── EmergencyContact.php
│   │   │   ├── Family.php
│   │   │   └── Notification.php
│   │   ├── 📁 services/
│   │   │   ├── JWTService.php
│   │   │   ├── KakaoService.php
│   │   │   └── FCMService.php
│   │   ├── 📁 middleware/
│   │   │   └── AuthMiddleware.php
│   │   └── 📁 utils/
│   │       ├── Database.php
│   │       ├── Response.php
│   │       └── Validator.php
│   ├── 📁 config/
│   │   ├── database.php
│   │   ├── app.php
│   │   └── kakao.php
│   ├── 📁 cron/
│   │   ├── check_inactive.php
│   │   └── daily_report.php
│   ├── 📁 sql/
│   │   └── schema.sql
│   ├── .htaccess
│   └── composer.json
│
├── 📁 frontend/                   # Vue.js 프론트엔드
│   ├── 📁 src/
│   │   ├── 📁 views/
│   │   │   ├── Home.vue          # 메인 체크인 화면
│   │   │   ├── Onboarding.vue    # 온보딩
│   │   │   ├── Settings.vue      # 설정
│   │   │   ├── History.vue       # 체크인 기록
│   │   │   ├── Family.vue        # 가족 관리
│   │   │   └── Dashboard.vue     # 가족 대시보드
│   │   ├── 📁 components/
│   │   │   ├── CheckInButton.vue # 큰 체크인 버튼
│   │   │   ├── StatusCard.vue    # 상태 카드
│   │   │   ├── ContactList.vue   # 연락처 목록
│   │   │   └── AlertModal.vue    # 알림 모달
│   │   ├── 📁 plugins/
│   │   │   ├── motion-detector.js    # 센서 감지
│   │   │   ├── sms-sender.js         # SMS 발송
│   │   │   ├── background-service.js # 백그라운드
│   │   │   └── local-db.js           # 로컬 DB
│   │   ├── 📁 services/
│   │   │   ├── api.js            # API 클라이언트
│   │   │   ├── auth.js           # 인증
│   │   │   └── notification.js   # 푸시 알림
│   │   ├── 📁 stores/
│   │   │   ├── user.js           # 사용자 상태
│   │   │   ├── checkin.js        # 체크인 상태
│   │   │   └── settings.js       # 설정
│   │   ├── App.vue
│   │   ├── main.js
│   │   └── router.js
│   ├── 📁 public/
│   │   └── index.html
│   ├── package.json
│   ├── vite.config.js
│   └── capacitor.config.ts
│
├── 📁 mobile/                     # Capacitor 네이티브 코드
│   ├── 📁 android/
│   │   └── 📁 app/src/main/java/com/ansimtalk/
│   │       ├── 📁 plugins/
│   │       │   ├── SMSPlugin.java
│   │       │   └── MotionPlugin.java
│   │       └── 📁 services/
│   │           └── MotionService.java
│   └── 📁 ios/
│       └── 📁 App/Plugins/
│           ├── SMSPlugin.swift
│           └── MotionPlugin.swift
│
├── 📁 docs/
│   └── (기존 기획 문서들)
│
└── README.md
```

---

## 🗄️ 데이터베이스 스키마

### MySQL 테이블 설계

```sql
-- ============================================
-- 안심톡 데이터베이스 스키마
-- XAMPP MySQL 8.x 용
-- ============================================

CREATE DATABASE IF NOT EXISTS ansimtalk
CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

USE ansimtalk;

-- 1. 사용자 테이블
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    device_id VARCHAR(255) UNIQUE NOT NULL COMMENT '기기 고유 ID',
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    email VARCHAR(255),
    fcm_token VARCHAR(500) COMMENT 'Firebase 푸시 토큰',
    profile_image VARCHAR(500),

    -- 설정
    settings JSON DEFAULT '{}' COMMENT '사용자 설정',
    /*
    settings 구조:
    {
        "check_in_time": "09:00",
        "sensor_sensitivity": "medium",
        "alert_thresholds": {
            "warning": 4,
            "critical": 12,
            "emergency": 24
        },
        "active_hours": {
            "start": 7,
            "end": 23
        },
        "notifications": {
            "push": true,
            "kakao": true,
            "sms": true
        }
    }
    */

    -- 상태
    status ENUM('active', 'inactive', 'suspended') DEFAULT 'active',
    last_activity_at TIMESTAMP NULL COMMENT '마지막 활동 (센서)',
    last_check_in_at TIMESTAMP NULL COMMENT '마지막 체크인',

    -- 타임스탬프
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_device_id (device_id),
    INDEX idx_last_activity (last_activity_at),
    INDEX idx_status (status)
) ENGINE=InnoDB;

-- 2. 긴급 연락처 테이블
CREATE TABLE emergency_contacts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) NOT NULL,
    email VARCHAR(255),
    relationship VARCHAR(50) COMMENT '관계 (자녀, 배우자, 친구 등)',
    priority INT DEFAULT 1 COMMENT '알림 순서 (1=최우선)',

    -- 알림 설정
    notify_on_checkin BOOLEAN DEFAULT FALSE COMMENT '체크인 시 알림',
    notify_on_warning BOOLEAN DEFAULT TRUE COMMENT '경고 시 알림',
    notify_on_emergency BOOLEAN DEFAULT TRUE COMMENT '긴급 시 알림',

    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user_id (user_id),
    INDEX idx_priority (user_id, priority)
) ENGINE=InnoDB;

-- 3. 가족 그룹 테이블
CREATE TABLE families (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL COMMENT '가족 그룹명',
    invite_code VARCHAR(10) UNIQUE NOT NULL COMMENT '초대 코드',
    created_by INT NOT NULL,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (created_by) REFERENCES users(id),
    INDEX idx_invite_code (invite_code)
) ENGINE=InnoDB;

-- 4. 가족 멤버 테이블
CREATE TABLE family_members (
    id INT AUTO_INCREMENT PRIMARY KEY,
    family_id INT NOT NULL,
    user_id INT NOT NULL,
    role ENUM('admin', 'guardian', 'monitored') DEFAULT 'monitored' COMMENT '역할',
    nickname VARCHAR(50) COMMENT '가족 내 별명',

    joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (family_id) REFERENCES families(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY uk_family_user (family_id, user_id),
    INDEX idx_user_id (user_id)
) ENGINE=InnoDB;

-- 5. 체크인 기록 테이블
CREATE TABLE check_ins (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,

    check_in_type ENUM('manual', 'auto_sensor', 'voice', 'widget') DEFAULT 'manual',
    sensor_data JSON COMMENT '센서 데이터',
    /*
    sensor_data 구조:
    {
        "acceleration": { "x": 0.1, "y": 0.2, "z": 9.8 },
        "rotation": { "alpha": 0, "beta": 0, "gamma": 0 },
        "magnitude": 0.5
    }
    */

    location JSON COMMENT '위치 정보 (선택)',
    /*
    location 구조:
    {
        "latitude": 37.5665,
        "longitude": 126.9780,
        "accuracy": 10
    }
    */

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user_created (user_id, created_at DESC)
) ENGINE=InnoDB;

-- 6. 활동 로그 테이블 (센서 데이터)
CREATE TABLE activity_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,

    activity_type ENUM('acceleration', 'rotation', 'screen_on', 'app_open', 'walking', 'stationary') NOT NULL,
    value DECIMAL(10, 4) COMMENT '센서 값 (magnitude)',

    recorded_at TIMESTAMP NOT NULL COMMENT '기기에서 기록된 시간',
    synced_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '서버 동기화 시간',

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user_recorded (user_id, recorded_at DESC),
    INDEX idx_recorded_at (recorded_at)
) ENGINE=InnoDB;

-- 7. 사용자 상태 테이블 (실시간 상태 관리)
CREATE TABLE user_status (
    user_id INT PRIMARY KEY,

    current_status ENUM('normal', 'warning', 'critical', 'emergency') DEFAULT 'normal',

    last_check_in_at TIMESTAMP NULL,
    last_activity_at TIMESTAMP NULL,

    warning_sent_at TIMESTAMP NULL COMMENT '경고 알림 발송 시간',
    critical_sent_at TIMESTAMP NULL COMMENT '심각 알림 발송 시간',
    emergency_sent_at TIMESTAMP NULL COMMENT '긴급 알림 발송 시간',

    consecutive_checkin_days INT DEFAULT 0 COMMENT '연속 체크인 일수',

    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- 8. 알림 기록 테이블
CREATE TABLE notifications (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL COMMENT '알림 대상 사용자',
    contact_id INT COMMENT '수신자 (긴급연락처)',

    type ENUM('push', 'sms', 'kakao', 'email') NOT NULL,
    trigger_reason ENUM('manual_checkin', 'auto_checkin', 'reminder', 'warning', 'critical', 'emergency') NOT NULL,

    title VARCHAR(200),
    message TEXT,

    status ENUM('pending', 'sent', 'delivered', 'failed') DEFAULT 'pending',
    error_message TEXT COMMENT '실패 시 에러 메시지',

    sent_at TIMESTAMP NULL,
    delivered_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (contact_id) REFERENCES emergency_contacts(id) ON DELETE SET NULL,
    INDEX idx_user_type (user_id, type),
    INDEX idx_status (status),
    INDEX idx_created (created_at DESC)
) ENGINE=InnoDB;

-- 9. 인증 토큰 테이블
CREATE TABLE auth_tokens (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,

    access_token VARCHAR(500) NOT NULL,
    refresh_token VARCHAR(500) NOT NULL,

    device_info JSON COMMENT '기기 정보',

    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_access_token (access_token(100)),
    INDEX idx_refresh_token (refresh_token(100)),
    INDEX idx_user_id (user_id)
) ENGINE=InnoDB;

-- 10. 앱 설정 테이블 (전역)
CREATE TABLE app_settings (
    setting_key VARCHAR(100) PRIMARY KEY,
    setting_value TEXT,
    description VARCHAR(500),
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- 초기 설정 데이터
INSERT INTO app_settings (setting_key, setting_value, description) VALUES
('default_warning_hours', '4', '기본 경고 시간 (시간)'),
('default_critical_hours', '12', '기본 심각 시간 (시간)'),
('default_emergency_hours', '24', '기본 긴급 시간 (시간)'),
('default_checkin_time', '09:00', '기본 체크인 시간'),
('app_version', '1.0.0', '현재 앱 버전'),
('maintenance_mode', 'false', '점검 모드');

-- ============================================
-- 뷰 생성
-- ============================================

-- 사용자 상태 요약 뷰
CREATE OR REPLACE VIEW v_user_status_summary AS
SELECT
    u.id,
    u.name,
    u.phone,
    us.current_status,
    us.last_check_in_at,
    us.last_activity_at,
    us.consecutive_checkin_days,
    TIMESTAMPDIFF(HOUR, COALESCE(us.last_activity_at, us.last_check_in_at, u.created_at), NOW()) as inactive_hours,
    (SELECT COUNT(*) FROM emergency_contacts ec WHERE ec.user_id = u.id AND ec.is_active = TRUE) as contact_count
FROM users u
LEFT JOIN user_status us ON u.id = us.user_id
WHERE u.status = 'active';

-- 가족 대시보드 뷰
CREATE OR REPLACE VIEW v_family_dashboard AS
SELECT
    f.id as family_id,
    f.name as family_name,
    fm.user_id,
    u.name as user_name,
    fm.role,
    fm.nickname,
    us.current_status,
    us.last_check_in_at,
    us.consecutive_checkin_days
FROM families f
JOIN family_members fm ON f.id = fm.family_id
JOIN users u ON fm.user_id = u.id
LEFT JOIN user_status us ON u.id = us.user_id;
```

---

## 🔧 개발 태스크 목록

### Phase 1: 환경 설정 및 기본 구조

```markdown
## Task 1.1: XAMPP 환경 설정
- [ ] XAMPP 설치 확인 (Apache, MySQL, PHP 8.x)
- [ ] MySQL에 ansimtalk 데이터베이스 생성
- [ ] schema.sql 실행하여 테이블 생성
- [ ] PHP 확장 모듈 확인 (pdo_mysql, json, curl, openssl)
- [ ] Apache 가상호스트 설정 (api.ansimtalk.local)

## Task 1.2: 백엔드 프로젝트 초기화
- [ ] composer.json 생성 (firebase/php-jwt, vlucas/phpdotenv)
- [ ] composer install 실행
- [ ] 디렉토리 구조 생성
- [ ] .htaccess 설정 (URL 리라이트)
- [ ] config 파일 생성 (database.php, app.php)

## Task 1.3: 프론트엔드 프로젝트 초기화
- [ ] Vite + Vue 3 프로젝트 생성
- [ ] Capacitor 설치 및 초기화
- [ ] 필요한 패키지 설치 (pinia, vue-router, axios)
- [ ] 디렉토리 구조 생성
```

### Phase 2: 백엔드 API 개발

```markdown
## Task 2.1: 핵심 유틸리티 개발
- [ ] Database.php - PDO 싱글톤 클래스
- [ ] Response.php - JSON 응답 헬퍼
- [ ] Validator.php - 입력값 검증
- [ ] JWTService.php - JWT 토큰 생성/검증

## Task 2.2: 인증 API
- [ ] POST /api/auth/register - 회원가입 (기기 등록)
- [ ] POST /api/auth/login - 로그인 (기기 인증)
- [ ] POST /api/auth/refresh - 토큰 갱신
- [ ] GET /api/auth/me - 내 정보 조회
- [ ] AuthMiddleware.php - 인증 미들웨어

## Task 2.3: 체크인 API
- [ ] POST /api/checkin - 체크인 기록
- [ ] GET /api/checkin/history - 체크인 히스토리
- [ ] GET /api/checkin/stats - 통계 조회
- [ ] POST /api/checkin/batch - 배치 체크인 (동기화)

## Task 2.4: 긴급연락처 API
- [ ] GET /api/contacts - 연락처 목록
- [ ] POST /api/contacts - 연락처 추가
- [ ] PUT /api/contacts/{id} - 연락처 수정
- [ ] DELETE /api/contacts/{id} - 연락처 삭제

## Task 2.5: 가족 API
- [ ] POST /api/family/create - 가족 그룹 생성
- [ ] POST /api/family/join - 가족 참가 (초대코드)
- [ ] GET /api/family/members - 가족 멤버 조회
- [ ] GET /api/family/dashboard - 가족 대시보드
- [ ] DELETE /api/family/leave - 가족 탈퇴

## Task 2.6: 동기화 API
- [ ] POST /api/sync/activities - 활동 로그 동기화
- [ ] POST /api/sync/checkins - 체크인 배치 동기화
- [ ] GET /api/sync/status - 동기화 상태 조회

## Task 2.7: 알림 API
- [ ] POST /api/notifications/test - 테스트 알림 발송
- [ ] GET /api/notifications/history - 알림 기록
- [ ] KakaoService.php - 카카오 알림톡 연동
- [ ] FCMService.php - Firebase 푸시 연동
```

### Phase 3: 프론트엔드 개발

```markdown
## Task 3.1: 기본 UI 컴포넌트
- [ ] App.vue - 앱 레이아웃
- [ ] router.js - 라우터 설정
- [ ] CheckInButton.vue - 큰 체크인 버튼
- [ ] StatusCard.vue - 상태 카드
- [ ] BottomNav.vue - 하단 네비게이션

## Task 3.2: 온보딩 화면
- [ ] Onboarding.vue - 온보딩 메인
- [ ] 이름 입력 단계
- [ ] 긴급연락처 등록 단계
- [ ] 권한 요청 단계 (센서, SMS, 알림)
- [ ] 완료 화면

## Task 3.3: 메인 화면
- [ ] Home.vue - 메인 체크인 화면
- [ ] 체크인 버튼 (큰 원형)
- [ ] 마지막 체크인 시간 표시
- [ ] 연속 체크인 일수 표시
- [ ] 현재 상태 표시

## Task 3.4: 설정 화면
- [ ] Settings.vue - 설정 메인
- [ ] 체크인 시간 설정
- [ ] 센서 감도 설정
- [ ] 알림 설정
- [ ] 긴급연락처 관리

## Task 3.5: 기록/통계 화면
- [ ] History.vue - 체크인 기록
- [ ] 캘린더 뷰
- [ ] 통계 차트 (연속 일수, 체크인 패턴)

## Task 3.6: 가족 화면
- [ ] Family.vue - 가족 관리
- [ ] 가족 생성/참가
- [ ] Dashboard.vue - 가족 대시보드
- [ ] 가족 멤버 상태 카드
```

### Phase 4: Capacitor 네이티브 기능

```markdown
## Task 4.1: 센서 플러그인 개발
- [ ] motion-detector.js - JS 브릿지
- [ ] MotionPlugin.java - Android 네이티브
- [ ] MotionService.java - Android 백그라운드 서비스
- [ ] MotionPlugin.swift - iOS 네이티브
- [ ] 센서 데이터 처리 로직

## Task 4.2: SMS 플러그인 개발
- [ ] sms-sender.js - JS 브릿지
- [ ] SMSPlugin.java - Android 네이티브 (직접 발송)
- [ ] SMSPlugin.swift - iOS 네이티브 (앱 열기)

## Task 4.3: 백그라운드 서비스
- [ ] background-service.js - JS 브릿지
- [ ] Android 포그라운드 서비스 설정
- [ ] iOS 백그라운드 모드 설정
- [ ] 배터리 최적화 처리

## Task 4.4: 로컬 데이터베이스
- [ ] local-db.js - SQLite 래퍼
- [ ] 오프라인 체크인 저장
- [ ] 동기화 큐 관리
```

### Phase 5: 통합 및 테스트

```markdown
## Task 5.1: API 통합
- [ ] api.js - Axios 클라이언트 설정
- [ ] 인증 인터셉터 (토큰 자동 갱신)
- [ ] 오프라인 처리

## Task 5.2: 상태 관리
- [ ] user.js - 사용자 스토어 (Pinia)
- [ ] checkin.js - 체크인 스토어
- [ ] settings.js - 설정 스토어
- [ ] 로컬 스토리지 연동

## Task 5.3: 알림 시스템 통합
- [ ] 푸시 알림 수신 처리
- [ ] 로컬 알림 (리마인더)
- [ ] 알림 클릭 시 앱 이동

## Task 5.4: 테스트
- [ ] API 단위 테스트
- [ ] 센서 감지 테스트
- [ ] SMS 발송 테스트
- [ ] 전체 플로우 테스트
```

---

## 🤖 추천 서브 에이전트 구성

### Agent 1: Backend Developer Agent
```yaml
name: "backend-dev"
description: "PHP 백엔드 API 개발 전문"
skills:
  - PHP 8.x 개발
  - MySQL 쿼리 최적화
  - RESTful API 설계
  - JWT 인증 구현
  - 보안 취약점 검토
tasks:
  - API 컨트롤러 개발
  - 데이터베이스 모델 구현
  - 미들웨어 작성
  - API 문서화
```

### Agent 2: Frontend Developer Agent
```yaml
name: "frontend-dev"
description: "Vue.js 프론트엔드 개발 전문"
skills:
  - Vue.js 3 Composition API
  - Pinia 상태관리
  - CSS/SCSS 스타일링
  - 반응형 디자인
  - 접근성 (a11y)
tasks:
  - Vue 컴포넌트 개발
  - 라우터 설정
  - 상태 관리 구현
  - UI/UX 최적화
```

### Agent 3: Mobile Developer Agent
```yaml
name: "mobile-dev"
description: "Capacitor 네이티브 플러그인 개발"
skills:
  - Capacitor 플러그인 개발
  - Android Java/Kotlin
  - iOS Swift
  - 센서 API
  - 백그라운드 서비스
tasks:
  - 네이티브 플러그인 작성
  - 센서 데이터 처리
  - SMS 발송 구현
  - 배터리 최적화
```

### Agent 4: Code Reviewer Agent
```yaml
name: "code-reviewer"
description: "코드 품질 및 보안 검토"
skills:
  - 코드 리뷰
  - 보안 취약점 분석
  - 성능 최적화 제안
  - 베스트 프랙티스 적용
tasks:
  - PR 코드 리뷰
  - 보안 검사
  - 성능 분석
  - 리팩토링 제안
```

### Agent 5: DevOps Agent
```yaml
name: "devops"
description: "배포 및 운영 자동화"
skills:
  - XAMPP 설정
  - 빌드 자동화
  - 앱 스토어 배포
  - 모니터링 설정
tasks:
  - 환경 설정
  - 빌드 스크립트 작성
  - 배포 자동화
  - 로그 모니터링
```

---

## 🛠️ 추천 스킬/도구

### 개발 도구
```yaml
IDE:
  - VS Code + PHP Intelephense
  - VS Code + Volar (Vue)

Testing:
  - Postman (API 테스트)
  - PHPUnit (PHP 단위 테스트)
  - Vitest (Vue 테스트)

Database:
  - phpMyAdmin (XAMPP 포함)
  - MySQL Workbench

Mobile:
  - Android Studio
  - Xcode
  - Capacitor CLI
```

### PHP 패키지
```json
{
  "require": {
    "php": ">=8.0",
    "firebase/php-jwt": "^6.0",
    "vlucas/phpdotenv": "^5.5",
    "guzzlehttp/guzzle": "^7.0",
    "monolog/monolog": "^3.0"
  }
}
```

### NPM 패키지
```json
{
  "dependencies": {
    "vue": "^3.4",
    "vue-router": "^4.2",
    "pinia": "^2.1",
    "axios": "^1.6",
    "@capacitor/core": "^5.6",
    "@capacitor/cli": "^5.6",
    "@capacitor/android": "^5.6",
    "@capacitor/ios": "^5.6",
    "@capacitor/push-notifications": "^5.1",
    "@capacitor/local-notifications": "^5.0",
    "@capacitor/preferences": "^5.0",
    "@capacitor/device": "^5.0",
    "@capacitor/motion": "^5.0"
  },
  "devDependencies": {
    "vite": "^5.0",
    "@vitejs/plugin-vue": "^5.0",
    "sass": "^1.69"
  }
}
```

---

## 📋 실행 명령어 시퀀스

### 1. 프로젝트 초기화
```bash
# 1. 프로젝트 디렉토리 생성
mkdir -p ansimtalk/{backend,frontend,mobile,docs}
cd ansimtalk

# 2. 백엔드 초기화
cd backend
composer init --name="ansimtalk/api" --type="project"
composer require firebase/php-jwt vlucas/phpdotenv guzzlehttp/guzzle monolog/monolog

# 3. 프론트엔드 초기화
cd ../frontend
npm create vite@latest . -- --template vue
npm install
npm install vue-router@4 pinia axios sass

# 4. Capacitor 초기화
npm install @capacitor/core @capacitor/cli
npx cap init "안심톡" "com.ansimtalk.app"
npm install @capacitor/android @capacitor/ios
npx cap add android
npx cap add ios

# 5. Capacitor 플러그인 설치
npm install @capacitor/push-notifications @capacitor/local-notifications
npm install @capacitor/preferences @capacitor/device @capacitor/motion
```

### 2. XAMPP 설정
```bash
# Apache 가상호스트 설정 (httpd-vhosts.conf)
<VirtualHost *:80>
    ServerName api.ansimtalk.local
    DocumentRoot "C:/xampp/htdocs/ansimtalk/backend"
    <Directory "C:/xampp/htdocs/ansimtalk/backend">
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>

# hosts 파일에 추가
127.0.0.1 api.ansimtalk.local
```

### 3. 데이터베이스 설정
```bash
# MySQL 접속 후
mysql -u root -p

# 데이터베이스 생성 및 스키마 실행
CREATE DATABASE ansimtalk CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE ansimtalk;
SOURCE C:/xampp/htdocs/ansimtalk/backend/sql/schema.sql;
```

### 4. 개발 서버 실행
```bash
# 프론트엔드 개발 서버
cd frontend
npm run dev

# Android 빌드 및 실행
npm run build
npx cap copy android
npx cap open android
```

---

## 🎯 마일스톤

```
┌─────────────────────────────────────────────────────────────┐
│                    개발 마일스톤                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [M1] 환경 설정 완료                                         │
│  ─────────────────                                          │
│  ✓ XAMPP 설정                                               │
│  ✓ 데이터베이스 생성                                         │
│  ✓ 프로젝트 구조 완성                                        │
│                                                             │
│  [M2] 백엔드 API 완료                                        │
│  ─────────────────                                          │
│  ✓ 인증 API                                                 │
│  ✓ 체크인 API                                               │
│  ✓ 가족/연락처 API                                          │
│  ✓ 동기화 API                                               │
│                                                             │
│  [M3] 프론트엔드 UI 완료                                     │
│  ─────────────────                                          │
│  ✓ 온보딩 화면                                              │
│  ✓ 메인 체크인 화면                                          │
│  ✓ 설정/기록 화면                                            │
│  ✓ 가족 대시보드                                             │
│                                                             │
│  [M4] 네이티브 기능 완료                                     │
│  ─────────────────                                          │
│  ✓ 센서 감지 플러그인                                        │
│  ✓ SMS 발송 플러그인                                         │
│  ✓ 백그라운드 서비스                                         │
│                                                             │
│  [M5] 통합 테스트 및 출시                                    │
│  ─────────────────                                          │
│  ✓ 전체 플로우 테스트                                        │
│  ✓ 버그 수정                                                │
│  ✓ 앱 스토어 등록                                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 📝 Claude Code 실행 프롬프트 예시

### 프로젝트 시작
```
@claude-code

안심톡 프로젝트를 시작합니다.
XAMPP + PHP + MySQL 환경에서 개발합니다.

1. 먼저 backend/sql/schema.sql 파일을 생성해주세요.
2. 위 스키마를 MySQL에서 실행할 수 있도록 해주세요.
3. backend 폴더 구조를 생성하고 기본 파일들을 만들어주세요.

참고 문서: docs/claude-code-development-plan.md
```

### API 개발 요청
```
@claude-code

backend-dev 에이전트로 작업합니다.

Task 2.2: 인증 API를 개발해주세요.
- POST /api/auth/register
- POST /api/auth/login
- JWT 토큰 발급 및 검증

참고: docs/claude-code-development-plan.md의 API 명세를 따라주세요.
```

### 프론트엔드 개발 요청
```
@claude-code

frontend-dev 에이전트로 작업합니다.

Task 3.3: 메인 체크인 화면을 개발해주세요.
- 큰 원형 체크인 버튼 (시니어 친화적)
- 마지막 체크인 시간 표시
- 상태 카드 컴포넌트

디자인 참고:
- 버튼 크기: 화면의 50% 이상
- 색상: 그린 계열 (안심)
- 폰트: 큰 글씨 (24px 이상)
```

### 네이티브 플러그인 요청
```
@claude-code

mobile-dev 에이전트로 작업합니다.

Task 4.1: 센서 플러그인을 개발해주세요.
- 가속도계와 자이로스코프 데이터 수집
- 움직임 임계값 감지
- 백그라운드에서 실행

Android 먼저 개발하고, iOS는 그 다음에 진행합니다.
```

---

## 🔐 환경 변수 템플릿

### backend/.env
```env
# 데이터베이스
DB_HOST=localhost
DB_PORT=3306
DB_DATABASE=ansimtalk
DB_USERNAME=root
DB_PASSWORD=

# JWT
JWT_SECRET=your-secret-key-here-change-in-production
JWT_EXPIRY=3600
JWT_REFRESH_EXPIRY=604800

# 카카오
KAKAO_REST_API_KEY=
KAKAO_ALIMTALK_SENDER_KEY=
KAKAO_ALIMTALK_TEMPLATE_CODE=

# Firebase
FCM_SERVER_KEY=
FCM_PROJECT_ID=

# 앱 설정
APP_ENV=development
APP_DEBUG=true
APP_URL=http://api.ansimtalk.local
```

### frontend/.env
```env
VITE_API_URL=http://api.ansimtalk.local/api
VITE_APP_NAME=안심톡
VITE_FCM_VAPID_KEY=
```

---

## ✅ 체크리스트

### 개발 전 확인사항
- [ ] XAMPP 설치 및 실행 확인
- [ ] PHP 8.x 버전 확인 (`php -v`)
- [ ] MySQL 실행 확인
- [ ] Node.js 18+ 설치 확인 (`node -v`)
- [ ] Android Studio 설치 (Android 개발 시)
- [ ] Xcode 설치 (iOS 개발 시, Mac만)

### 개발 완료 확인사항
- [ ] 모든 API 엔드포인트 테스트 완료
- [ ] 센서 감지 정상 동작 확인
- [ ] SMS 발송 테스트 완료
- [ ] 오프라인 모드 테스트
- [ ] 배터리 소모 테스트
- [ ] 다양한 기기에서 테스트

---

*문서 버전: 1.0*
*최종 업데이트: 2026-01-22*
*다음 단계: 환경 설정 후 Task 1.1부터 순차 진행*
