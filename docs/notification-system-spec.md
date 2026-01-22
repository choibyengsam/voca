# 알림 시스템 기획서
## Android 기기 SMS + iOS 푸시 알림 (사업자 없이 시작)

---

## 📋 개요

### 알림 발송 전략
```
┌─────────────────────────────────────────────────────────────┐
│                    알림 발송 전략                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [사용자 기기]              [긴급연락처 기기]                │
│                                                             │
│  ┌─────────────┐           ┌─────────────┐                 │
│  │  Android    │ ────SMS──→│  Android    │ 직접 수신       │
│  │  사용자     │           │  연락처     │                 │
│  └─────────────┘           └─────────────┘                 │
│                                                             │
│  ┌─────────────┐           ┌─────────────┐                 │
│  │  Android    │ ──푸시───→│  iOS        │ 앱 설치 필요    │
│  │  사용자     │   (FCM)   │  연락처     │                 │
│  └─────────────┘           └─────────────┘                 │
│                                                             │
│  ┌─────────────┐           ┌─────────────┐                 │
│  │  iOS        │ ──푸시───→│ Android/iOS │ 앱 설치 필요    │
│  │  사용자     │   (FCM)   │  연락처     │                 │
│  └─────────────┘           └─────────────┘                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 플랫폼별 알림 방식

| 사용자 기기 | 연락처 기기 | 알림 방식 | 앱 설치 필요 |
|------------|------------|----------|-------------|
| Android | Android | 기기 직접 SMS | ❌ 연락처 불필요 |
| Android | iOS | 푸시 알림 (FCM) | ✅ 연락처 필요 |
| iOS | Android | 푸시 알림 (FCM) | ✅ 연락처 필요 |
| iOS | iOS | 푸시 알림 (FCM) | ✅ 연락처 필요 |

---

## 🔗 친구/가족 연결 시스템

### 연결 방식 2가지

#### 방식 1: 초대 링크 (푸시 알림용) - iOS 연락처
```
┌─────────────────────────────────────────────────────────────┐
│                  초대 링크 플로우                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [사용자 - 어머니]                                          │
│       │                                                     │
│       ▼                                                     │
│  1. 앱에서 "가족 초대" 버튼 클릭                             │
│       │                                                     │
│       ▼                                                     │
│  2. 초대 링크 생성                                          │
│     https://areyouok.dongwd.kr/invite/ABC123                │
│       │                                                     │
│       ▼                                                     │
│  3. 카카오톡/문자로 링크 공유                                │
│       │                                                     │
│       ▼                                                     │
│  [연락처 - 아들]                                            │
│       │                                                     │
│       ▼                                                     │
│  4. 링크 클릭 → 앱스토어 이동 → 앱 설치                     │
│       │                                                     │
│       ▼                                                     │
│  5. 앱 실행 → 자동으로 가족 연결                            │
│       │                                                     │
│       ▼                                                     │
│  6. 푸시 알림 수신 허용                                     │
│       │                                                     │
│       ▼                                                     │
│  ✅ 연결 완료! 이제 긴급 알림 수신 가능                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 방식 2: 전화번호 직접 입력 (SMS용) - Android 연락처
```
┌─────────────────────────────────────────────────────────────┐
│                  전화번호 직접 입력 플로우                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [사용자 - 어머니]                                          │
│       │                                                     │
│       ▼                                                     │
│  1. 앱에서 "긴급연락처 추가" 클릭                            │
│       │                                                     │
│       ▼                                                     │
│  2. 이름, 전화번호 입력                                     │
│     - 이름: 민수                                            │
│     - 전화번호: 010-1234-5678                               │
│     - 관계: 아들                                            │
│       │                                                     │
│       ▼                                                     │
│  3. 저장 완료                                               │
│       │                                                     │
│       ▼                                                     │
│  ✅ 긴급 시 직접 SMS 발송 (앱 설치 불필요)                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 📊 데이터베이스 스키마 (알림 관련)

### 테이블 추가/수정

```sql
-- 긴급연락처 테이블 (수정)
CREATE TABLE emergency_contacts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,

    -- 연락처 정보
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) NOT NULL,
    relationship VARCHAR(50),
    priority INT DEFAULT 1,

    -- 연결 상태 (NEW)
    linked_user_id INT NULL COMMENT '앱 설치한 연락처의 user_id',
    is_app_installed BOOLEAN DEFAULT FALSE COMMENT '연락처가 앱 설치했는지',
    fcm_token VARCHAR(500) NULL COMMENT '연락처의 FCM 토큰 (푸시용)',

    -- 알림 방식 (NEW)
    notification_method ENUM('sms', 'push', 'both') DEFAULT 'sms',
    /*
      sms: 기기 직접 SMS (Android → Android)
      push: 푸시 알림 (앱 설치한 연락처)
      both: 둘 다 시도
    */

    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (linked_user_id) REFERENCES users(id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;


-- 초대 링크 테이블 (NEW)
CREATE TABLE invite_links (
    id INT AUTO_INCREMENT PRIMARY KEY,

    user_id INT NOT NULL COMMENT '초대한 사용자',
    invite_code VARCHAR(20) UNIQUE NOT NULL COMMENT '초대 코드',

    -- 초대 정보
    contact_name VARCHAR(100) COMMENT '초대할 연락처 이름',
    contact_phone VARCHAR(20) COMMENT '초대할 연락처 전화번호',
    relationship VARCHAR(50),

    -- 상태
    status ENUM('pending', 'accepted', 'expired') DEFAULT 'pending',
    accepted_by INT NULL COMMENT '수락한 사용자',
    accepted_at TIMESTAMP NULL,

    -- 만료
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (accepted_by) REFERENCES users(id) ON DELETE SET NULL,
    INDEX idx_invite_code (invite_code),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;


-- 사용자 테이블에 역할 추가
ALTER TABLE users ADD COLUMN user_role ENUM('user', 'guardian') DEFAULT 'user';
/*
  user: 일반 사용자 (체크인하는 사람, 예: 어르신)
  guardian: 보호자 (알림 받는 사람, 예: 자녀)
*/

ALTER TABLE users ADD COLUMN monitoring_user_id INT NULL COMMENT '내가 모니터링하는 사용자';
ALTER TABLE users ADD CONSTRAINT fk_monitoring_user
    FOREIGN KEY (monitoring_user_id) REFERENCES users(id) ON DELETE SET NULL;
```

---

## 🔔 알림 발송 로직

### Android 기기 직접 SMS 발송

```javascript
// frontend/src/plugins/sms-sender.js

class SMSSender {

  // SMS 발송 가능 여부 확인
  async canSendSMS() {
    const { Device } = Capacitor.Plugins;
    const info = await Device.getInfo();
    return info.platform === 'android';
  }

  // 긴급 SMS 발송
  async sendEmergencySMS(contacts, userName, lastCheckIn) {
    if (!await this.canSendSMS()) {
      console.log('SMS 발송 불가 (iOS) - 푸시 알림으로 대체');
      return { success: false, reason: 'ios_device' };
    }

    const message = this.createEmergencyMessage(userName, lastCheckIn);
    const results = [];

    for (const contact of contacts) {
      // SMS 방식인 연락처만 발송
      if (contact.notification_method === 'sms' || contact.notification_method === 'both') {
        try {
          await this.sendDirect(contact.phone, message);
          results.push({ contact: contact.name, method: 'sms', success: true });
        } catch (error) {
          results.push({ contact: contact.name, method: 'sms', success: false, error });
        }
      }
    }

    return { success: true, results };
  }

  // SMS 직접 발송 (Android Native)
  async sendDirect(phoneNumber, message) {
    const { SMS } = Capacitor.Plugins;
    return await SMS.send({
      phoneNumber: phoneNumber,
      message: message
    });
  }

  // 긴급 메시지 템플릿
  createEmergencyMessage(userName, lastCheckIn) {
    const hours = this.getHoursSince(lastCheckIn);
    return `[안심톡 긴급알림]

${userName}님이 ${hours}시간 동안
체크인하지 않았습니다.

안부를 확인해 주세요.

마지막 체크인: ${this.formatDate(lastCheckIn)}`;
  }

  getHoursSince(date) {
    return Math.floor((Date.now() - new Date(date).getTime()) / (1000 * 60 * 60));
  }

  formatDate(date) {
    return new Date(date).toLocaleString('ko-KR');
  }
}

export default new SMSSender();
```

### 푸시 알림 발송 (FCM)

```php
<?php
// api/services/FCMService.php

class FCMService {

    private $serverKey;
    private $fcmUrl = 'https://fcm.googleapis.com/fcm/send';

    public function __construct() {
        $this->serverKey = getenv('FCM_SERVER_KEY');
    }

    // 긴급 푸시 알림 발송
    public function sendEmergencyNotification($contacts, $userName, $lastCheckIn) {
        $results = [];

        foreach ($contacts as $contact) {
            // 푸시 방식이고 FCM 토큰이 있는 연락처만
            if (($contact['notification_method'] === 'push' || $contact['notification_method'] === 'both')
                && !empty($contact['fcm_token'])) {

                $result = $this->send(
                    $contact['fcm_token'],
                    '🚨 긴급: 안부 확인 필요',
                    "{$userName}님이 장시간 체크인하지 않았습니다. 안부를 확인해 주세요.",
                    [
                        'type' => 'emergency',
                        'user_name' => $userName,
                        'last_checkin' => $lastCheckIn,
                        'action' => 'check_status'
                    ]
                );

                $results[] = [
                    'contact' => $contact['name'],
                    'method' => 'push',
                    'success' => $result['success']
                ];
            }
        }

        return $results;
    }

    // FCM 발송
    public function send($token, $title, $body, $data = []) {
        $payload = [
            'to' => $token,
            'notification' => [
                'title' => $title,
                'body' => $body,
                'sound' => 'default',
                'badge' => 1
            ],
            'data' => $data,
            'priority' => 'high'
        ];

        $headers = [
            'Authorization: key=' . $this->serverKey,
            'Content-Type: application/json'
        ];

        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $this->fcmUrl);
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        return [
            'success' => $httpCode === 200,
            'response' => json_decode($response, true)
        ];
    }
}
```

### 통합 알림 서비스

```php
<?php
// api/services/NotificationService.php

class NotificationService {

    private $fcm;
    private $db;

    public function __construct() {
        $this->fcm = new FCMService();
        $this->db = Database::getInstance();
    }

    // 긴급 알림 발송 (통합)
    public function sendEmergencyAlert($userId) {
        // 사용자 정보 조회
        $user = $this->getUser($userId);
        $contacts = $this->getActiveContacts($userId);
        $lastCheckIn = $user['last_check_in_at'];

        $results = [
            'sms' => [],
            'push' => []
        ];

        // 1. 푸시 알림 발송 (서버에서)
        $pushResults = $this->fcm->sendEmergencyNotification(
            $contacts,
            $user['name'],
            $lastCheckIn
        );
        $results['push'] = $pushResults;

        // 2. SMS는 앱에서 직접 발송하도록 응답
        $smsContacts = array_filter($contacts, function($c) {
            return $c['notification_method'] === 'sms' || $c['notification_method'] === 'both';
        });

        // 알림 기록 저장
        $this->logNotifications($userId, $contacts, $results);

        return [
            'push_sent' => count($pushResults),
            'sms_contacts' => $smsContacts, // 앱에서 SMS 발송하도록
            'results' => $results
        ];
    }

    // 알림 기록 저장
    private function logNotifications($userId, $contacts, $results) {
        foreach ($results['push'] as $result) {
            $this->db->insert('notifications', [
                'user_id' => $userId,
                'type' => 'push',
                'trigger_reason' => 'emergency',
                'status' => $result['success'] ? 'sent' : 'failed',
                'sent_at' => date('Y-m-d H:i:s')
            ]);
        }
    }
}
```

---

## 📱 API 엔드포인트

### 초대 링크 API

```php
<?php
// api/controllers/InviteController.php

class InviteController {

    // POST /api/invite/create - 초대 링크 생성
    public function create() {
        $user = $GLOBALS['auth_user'];
        $data = json_decode(file_get_contents('php://input'), true);

        // 초대 코드 생성 (6자리 영숫자)
        $inviteCode = $this->generateInviteCode();

        // 만료 시간 (7일)
        $expiresAt = date('Y-m-d H:i:s', strtotime('+7 days'));

        $inviteId = $this->db->insert('invite_links', [
            'user_id' => $user['id'],
            'invite_code' => $inviteCode,
            'contact_name' => $data['name'] ?? null,
            'contact_phone' => $data['phone'] ?? null,
            'relationship' => $data['relationship'] ?? null,
            'expires_at' => $expiresAt
        ]);

        $inviteUrl = "https://areyouok.dongwd.kr/invite/{$inviteCode}";

        Response::json([
            'success' => true,
            'invite_code' => $inviteCode,
            'invite_url' => $inviteUrl,
            'expires_at' => $expiresAt,
            'share_message' => $this->getShareMessage($user['name'], $inviteUrl)
        ]);
    }

    // GET /api/invite/{code} - 초대 정보 조회
    public function get($code) {
        $invite = $this->db->findOne('invite_links', ['invite_code' => $code]);

        if (!$invite) {
            Response::json(['error' => '유효하지 않은 초대 코드입니다.'], 404);
            return;
        }

        if ($invite['status'] !== 'pending') {
            Response::json(['error' => '이미 사용된 초대 코드입니다.'], 400);
            return;
        }

        if (strtotime($invite['expires_at']) < time()) {
            Response::json(['error' => '만료된 초대 코드입니다.'], 400);
            return;
        }

        // 초대한 사용자 정보
        $inviter = $this->db->findOne('users', ['id' => $invite['user_id']]);

        Response::json([
            'success' => true,
            'invite' => [
                'inviter_name' => $inviter['name'],
                'relationship' => $invite['relationship'],
                'expires_at' => $invite['expires_at']
            ]
        ]);
    }

    // POST /api/invite/{code}/accept - 초대 수락
    public function accept($code) {
        $user = $GLOBALS['auth_user']; // 초대 받은 사람 (앱 설치 후 로그인)

        $invite = $this->db->findOne('invite_links', ['invite_code' => $code]);

        if (!$invite || $invite['status'] !== 'pending') {
            Response::json(['error' => '유효하지 않은 초대입니다.'], 400);
            return;
        }

        // 트랜잭션 시작
        $this->db->beginTransaction();

        try {
            // 1. 초대 상태 업데이트
            $this->db->update('invite_links',
                ['status' => 'accepted', 'accepted_by' => $user['id'], 'accepted_at' => date('Y-m-d H:i:s')],
                ['id' => $invite['id']]
            );

            // 2. 사용자 역할 업데이트 (보호자로)
            $this->db->update('users',
                ['user_role' => 'guardian', 'monitoring_user_id' => $invite['user_id']],
                ['id' => $user['id']]
            );

            // 3. 긴급연락처에 추가 (푸시 알림용)
            $this->db->insert('emergency_contacts', [
                'user_id' => $invite['user_id'],
                'name' => $user['name'],
                'phone' => $user['phone'] ?? '',
                'relationship' => $invite['relationship'],
                'linked_user_id' => $user['id'],
                'is_app_installed' => true,
                'fcm_token' => $user['fcm_token'],
                'notification_method' => 'push'
            ]);

            $this->db->commit();

            // 초대한 사람 정보
            $inviter = $this->db->findOne('users', ['id' => $invite['user_id']]);

            Response::json([
                'success' => true,
                'message' => '가족 연결이 완료되었습니다.',
                'monitoring' => [
                    'user_id' => $inviter['id'],
                    'name' => $inviter['name']
                ]
            ]);

        } catch (Exception $e) {
            $this->db->rollback();
            Response::json(['error' => '연결 중 오류가 발생했습니다.'], 500);
        }
    }

    // 공유 메시지 생성
    private function getShareMessage($userName, $inviteUrl) {
        return "[안심톡] {$userName}님이 가족으로 초대했어요.

안심톡 앱을 설치하면 {$userName}님의 안부를 확인할 수 있어요.

👉 {$inviteUrl}

긴급 상황 시 알림을 받을 수 있습니다.";
    }

    // 초대 코드 생성
    private function generateInviteCode() {
        return strtoupper(substr(md5(uniqid(mt_rand(), true)), 0, 6));
    }
}
```

---

## 📲 앱 화면 플로우

### 긴급연락처 추가 화면

```
┌─────────────────────────────────────┐
│  긴급연락처 추가            ✕      │
├─────────────────────────────────────┤
│                                     │
│  연락처 추가 방법을 선택하세요       │
│                                     │
│  ┌─────────────────────────────┐   │
│  │  📱 전화번호로 추가          │   │
│  │                             │   │
│  │  상대방이 앱을 설치하지      │   │
│  │  않아도 SMS로 알림 가능      │   │
│  │  (Android만 가능)           │   │
│  └─────────────────────────────┘   │
│                                     │
│  ┌─────────────────────────────┐   │
│  │  🔗 초대 링크로 추가         │   │
│  │                             │   │
│  │  상대방이 앱을 설치하면      │   │
│  │  푸시 알림으로 즉시 알림     │   │
│  │  (Android/iOS 모두 가능)    │   │
│  └─────────────────────────────┘   │
│                                     │
└─────────────────────────────────────┘
```

### 전화번호 직접 입력

```
┌─────────────────────────────────────┐
│  전화번호로 추가            ← 뒤로  │
├─────────────────────────────────────┤
│                                     │
│  이름                               │
│  ┌─────────────────────────────┐   │
│  │ 민수                        │   │
│  └─────────────────────────────┘   │
│                                     │
│  전화번호                           │
│  ┌─────────────────────────────┐   │
│  │ 010-1234-5678               │   │
│  └─────────────────────────────┘   │
│                                     │
│  관계                               │
│  ┌─────────────────────────────┐   │
│  │ 아들           ▼            │   │
│  └─────────────────────────────┘   │
│                                     │
│  ⚠️ 긴급 상황 시 이 번호로 SMS가    │
│     발송됩니다. (Android 전용)      │
│                                     │
│  ┌─────────────────────────────┐   │
│  │         저장하기             │   │
│  └─────────────────────────────┘   │
│                                     │
└─────────────────────────────────────┘
```

### 초대 링크 생성

```
┌─────────────────────────────────────┐
│  가족 초대하기              ← 뒤로  │
├─────────────────────────────────────┤
│                                     │
│  초대할 가족 정보                    │
│                                     │
│  이름                               │
│  ┌─────────────────────────────┐   │
│  │ 민수                        │   │
│  └─────────────────────────────┘   │
│                                     │
│  관계                               │
│  ┌─────────────────────────────┐   │
│  │ 아들           ▼            │   │
│  └─────────────────────────────┘   │
│                                     │
│  ┌─────────────────────────────┐   │
│  │       초대 링크 만들기       │   │
│  └─────────────────────────────┘   │
│                                     │
└─────────────────────────────────────┘

          ↓ 버튼 클릭 후

┌─────────────────────────────────────┐
│  초대 링크 생성 완료!               │
├─────────────────────────────────────┤
│                                     │
│  ┌─────────────────────────────┐   │
│  │  https://areyouok.dongwd.   │   │
│  │  kr/invite/ABC123           │   │
│  │                    📋 복사  │   │
│  └─────────────────────────────┘   │
│                                     │
│  이 링크를 민수님에게 공유하세요     │
│  7일 후 만료됩니다                  │
│                                     │
│  ┌─────────────────────────────┐   │
│  │  💬 카카오톡으로 공유        │   │
│  └─────────────────────────────┘   │
│                                     │
│  ┌─────────────────────────────┐   │
│  │  📱 문자로 공유              │   │
│  └─────────────────────────────┘   │
│                                     │
└─────────────────────────────────────┘
```

### 보호자 앱 화면 (초대 수락 후)

```
┌─────────────────────────────────────┐
│  안심톡 - 보호자              ⚙️   │
├─────────────────────────────────────┤
│                                     │
│  👵 어머니 (김순자)                 │
│                                     │
│  ┌─────────────────────────────┐   │
│  │                             │   │
│  │    ✅ 오늘 체크인 완료       │   │
│  │                             │   │
│  │    09:15 체크인             │   │
│  │    연속 7일째               │   │
│  │                             │   │
│  └─────────────────────────────┘   │
│                                     │
│  📊 이번 주                         │
│  ┌───┬───┬───┬───┬───┬───┬───┐   │
│  │ 월│ 화│ 수│ 목│ 금│ 토│ 일│   │
│  │ ✓│ ✓│ ✓│ ✓│ ✓│ ✓│ ✓│   │
│  └───┴───┴───┴───┴───┴───┴───┘   │
│                                     │
│  ┌─────────────────────────────┐   │
│  │      📞 어머니께 전화        │   │
│  └─────────────────────────────┘   │
│                                     │
└─────────────────────────────────────┘
```

---

## 🔄 알림 발송 시나리오

### 시나리오 1: Android 사용자 → Android 연락처
```
1. 어머니(Android)가 24시간 미체크인
2. 앱 백그라운드 서비스가 감지
3. 앱에서 직접 SMS 발송
4. 아들(Android) 문자 수신
   발신: 어머니 번호 (010-xxxx-xxxx)
   내용: [안심톡 긴급알림] 어머니님이 24시간...
```

### 시나리오 2: Android 사용자 → iOS 연락처 (앱 설치)
```
1. 어머니(Android)가 24시간 미체크인
2. 서버 Cron이 감지 → 푸시 알림 발송
3. 아들(iOS) 푸시 알림 수신
   "🚨 긴급: 어머니님이 24시간 동안..."
4. 알림 클릭 → 앱 열림 → 상태 확인
```

### 시나리오 3: iOS 사용자 → 모든 연락처
```
1. 어머니(iOS)가 24시간 미체크인
2. 서버 Cron이 감지
3. 앱 설치한 연락처 → 푸시 알림
4. 앱 미설치 연락처 → 알림 불가 (사전 안내 필요)
```

---

## ⚙️ 환경 변수

```env
# .env

# Firebase Cloud Messaging
FCM_SERVER_KEY=your_fcm_server_key_here
FCM_PROJECT_ID=your_project_id

# 앱 URL
APP_URL=https://areyouok.dongwd.kr
INVITE_URL=https://areyouok.dongwd.kr/invite
```

---

## 📋 Claude Code 실행 명령어

### 1. 초대 시스템 구현

```
초대 링크 시스템을 만들어줘.

테이블: invite_links
- user_id, invite_code, contact_name, relationship
- status (pending/accepted/expired), expires_at

API:
- POST /api/invite/create - 초대 링크 생성
- GET /api/invite/{code} - 초대 정보 조회
- POST /api/invite/{code}/accept - 초대 수락

초대 수락 시:
1. emergency_contacts에 추가 (notification_method='push')
2. 수락한 사용자의 user_role을 'guardian'으로 변경
3. monitoring_user_id 설정

docs/notification-system-spec.md 참고해줘.
```

### 2. SMS 발송 플러그인

```
Android에서 직접 SMS 발송하는 Capacitor 플러그인 만들어줘.

파일:
- frontend/src/plugins/sms-sender.js
- mobile/android/.../plugins/SMSPlugin.java

기능:
- Android SmsManager로 직접 발송
- 여러 연락처에 순차 발송
- 발송 결과 반환

docs/notification-system-spec.md 참고해줘.
```

### 3. 푸시 알림 서비스

```
FCM 푸시 알림 서비스를 만들어줘.

파일: api/services/FCMService.php

기능:
- FCM 서버로 푸시 발송
- 긴급 알림 템플릿
- 발송 결과 로깅

docs/notification-system-spec.md 참고해줘.
```

---

*문서 버전: 1.0*
*최종 업데이트: 2026-01-22*
