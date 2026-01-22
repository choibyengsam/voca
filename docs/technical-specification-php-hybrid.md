# 생사 확인 앱 기술 명세서
## PHP + MySQL 백엔드 & 하이브리드 앱 아키텍처

---

## 1. 기술 스택 개요

### 1.1 선정 기술 스택

```
┌─────────────────────────────────────────────────────────────┐
│                    기술 스택 구성                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [Frontend - 하이브리드 앱]                                  │
│  ├── Framework: Apache Cordova 또는 Capacitor               │
│  ├── UI: HTML5 + CSS3 + JavaScript (Vue.js/React)           │
│  ├── 네이티브 플러그인: SMS, 센서, 백그라운드 서비스           │
│  └── 빌드: Android Studio / Xcode                           │
│                                                             │
│  [Backend - PHP + MySQL]                                    │
│  ├── Language: PHP 8.x                                      │
│  ├── Framework: Laravel 10.x (권장) 또는 Pure PHP            │
│  ├── Database: MySQL 8.x / MariaDB 10.x                     │
│  ├── API: RESTful JSON API                                  │
│  └── 호스팅: 카페24, 가비아, AWS Lightsail 등                │
│                                                             │
│  [핵심 기능]                                                 │
│  ├── 자이로센서/가속도계 움직임 감지 → 자동 생존 확인         │
│  ├── 기기 직접 SMS 발송 (서버 경유 X)                         │
│  └── 백그라운드 서비스 상시 실행                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 기술 선정 이유

| 기술 | 선정 이유 |
|------|----------|
| **PHP + MySQL** | 저비용 호스팅, 개발자 풍부, 빠른 개발, 유지보수 용이 |
| **하이브리드 앱** | 크로스플랫폼, 웹 기술 활용, 네이티브 기능 접근 가능 |
| **Capacitor** | Cordova 대비 현대적, iOS/Android 네이티브 API 접근 우수 |
| **기기 SMS** | 서버 비용 절감, 통신사 심사 불필요, 사용자 요금 부담 |
| **자이로센서** | 수동 체크인 없이 자동 생존 확인, 사용 편의성 극대화 |

---

## 2. 시스템 아키텍처

### 2.1 전체 아키텍처

```
┌─────────────────────────────────────────────────────────────────────┐
│                         시스템 아키텍처                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                     하이브리드 앱 (스마트폰)                    │  │
│  ├───────────────────────────────────────────────────────────────┤  │
│  │                                                               │  │
│  │   ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │  │
│  │   │   WebView   │  │  센서 모듈   │  │  백그라운드 서비스    │  │  │
│  │   │ (Vue.js UI) │  │ (자이로/가속)│  │  (상시 모니터링)     │  │  │
│  │   └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘  │  │
│  │          │                │                    │              │  │
│  │          └────────────────┼────────────────────┘              │  │
│  │                           │                                   │  │
│  │   ┌───────────────────────▼───────────────────────────────┐  │  │
│  │   │                  Cordova/Capacitor Bridge              │  │  │
│  │   └───────────────────────┬───────────────────────────────┘  │  │
│  │                           │                                   │  │
│  │   ┌───────────┬───────────┼───────────┬───────────────────┐  │  │
│  │   ▼           ▼           ▼           ▼                   │  │  │
│  │ ┌─────┐   ┌─────┐   ┌──────────┐   ┌────────────────┐    │  │  │
│  │ │ SMS │   │Push │   │ Sensors  │   │ Background     │    │  │  │
│  │ │ 직접│   │Local│   │가속도/자이│   │ Service        │    │  │  │
│  │ │ 발송│   │Notif│   │ 로센서   │   │ (상시 실행)    │    │  │  │
│  │ └─────┘   └─────┘   └──────────┘   └────────────────┘    │  │  │
│  │                                                           │  │  │
│  └───────────────────────────┬───────────────────────────────┘  │
│                              │ HTTPS (API 호출)                  │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                      PHP 백엔드 서버                        │  │
│  ├───────────────────────────────────────────────────────────┤  │
│  │                                                           │  │
│  │   ┌────────────────────────────────────────────────────┐  │  │
│  │   │                   API Gateway (index.php)          │  │  │
│  │   └───────────────────────┬────────────────────────────┘  │  │
│  │                           │                               │  │
│  │   ┌───────────┬───────────┼───────────┬───────────────┐  │  │
│  │   ▼           ▼           ▼           ▼               │  │  │
│  │ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────┐  │  │  │
│  │ │Auth API │ │Check-in │ │Family   │ │Notification │  │  │  │
│  │ │         │ │  API    │ │  API    │ │   API       │  │  │  │
│  │ └────┬────┘ └────┬────┘ └────┬────┘ └──────┬──────┘  │  │  │
│  │      │           │           │             │          │  │  │
│  │      └───────────┴───────────┴─────────────┘          │  │  │
│  │                           │                           │  │  │
│  │                           ▼                           │  │  │
│  │   ┌────────────────────────────────────────────────┐  │  │
│  │   │                 MySQL Database                 │  │  │
│  │   │                                                │  │  │
│  │   │  users, check_ins, emergency_contacts,         │  │  │
│  │   │  families, notifications, activity_logs        │  │  │
│  │   └────────────────────────────────────────────────┘  │  │
│  │                                                       │  │
│  │   ┌────────────────────────────────────────────────┐  │  │
│  │   │              Cron Job (스케줄러)                │  │  │
│  │   │  - 미체크인 사용자 감지 (매 시간)               │  │  │
│  │   │  - 푸시 알림 트리거                            │  │  │
│  │   │  - 통계 집계 (일일)                            │  │  │
│  │   └────────────────────────────────────────────────┘  │  │
│  │                                                       │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 데이터 흐름

```
[자동 생존 확인 플로우 - 자이로센서 기반]

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  1. 백그라운드 서비스 (상시 실행)                             │
│     │                                                       │
│     ▼                                                       │
│  2. 자이로센서/가속도계 모니터링                              │
│     │                                                       │
│     ├── 움직임 감지됨 ──────────────────────────┐           │
│     │                                          │           │
│     │                                          ▼           │
│     │                              3. 로컬 DB 기록          │
│     │                                 (마지막 활동 시간)     │
│     │                                          │           │
│     │                                          ▼           │
│     │                              4. 서버 동기화           │
│     │                                 (배치, 10분 주기)     │
│     │                                                       │
│     └── 장시간 움직임 없음 (임계값 초과) ────────┐           │
│                                                │           │
│                                                ▼           │
│                              5. 경고 단계 진입              │
│                                 (로컬 푸시 알림)            │
│                                                │           │
│                                                ▼           │
│                              6. 사용자 무응답 시            │
│                                 (설정 시간 경과)            │
│                                                │           │
│                                                ▼           │
│                              7. 기기에서 직접 SMS 발송      │
│                                 (긴급연락처에)              │
│                                                │           │
│                                                ▼           │
│                              8. 서버에 이벤트 기록          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. 핵심 기능 상세 설계

### 3.1 자이로센서/가속도계 움직임 감지

#### 센서 데이터 처리 로직

```javascript
// Capacitor 센서 플러그인 사용 예시
// plugins/motion-detector.js

class MotionDetector {
  constructor() {
    this.lastActivity = Date.now();
    this.threshold = {
      acceleration: 0.5,  // m/s² - 미세한 움직임도 감지
      rotation: 0.1       // rad/s - 회전 감지
    };
    this.checkInterval = 30000;  // 30초마다 체크
    this.alertThreshold = {
      warning: 4 * 60 * 60 * 1000,   // 4시간 미활동 시 경고
      critical: 12 * 60 * 60 * 1000, // 12시간 미활동 시 긴급 알림
      emergency: 24 * 60 * 60 * 1000 // 24시간 미활동 시 SMS 발송
    };
  }

  // 센서 모니터링 시작
  async startMonitoring() {
    // 가속도계 리스너
    const { DeviceMotion } = Capacitor.Plugins;

    DeviceMotion.addListener('accel', (event) => {
      const { x, y, z } = event.acceleration;
      const magnitude = Math.sqrt(x*x + y*y + z*z);

      // 중력 제외한 순수 가속도
      const netAcceleration = Math.abs(magnitude - 9.8);

      if (netAcceleration > this.threshold.acceleration) {
        this.recordActivity('acceleration', netAcceleration);
      }
    });

    // 자이로스코프 리스너
    DeviceMotion.addListener('gyro', (event) => {
      const { alpha, beta, gamma } = event.rotationRate;
      const rotation = Math.sqrt(alpha*alpha + beta*beta + gamma*gamma);

      if (rotation > this.threshold.rotation) {
        this.recordActivity('rotation', rotation);
      }
    });

    // 주기적 상태 체크
    setInterval(() => this.checkActivityStatus(), this.checkInterval);
  }

  // 활동 기록
  recordActivity(type, value) {
    this.lastActivity = Date.now();

    // 로컬 DB 저장
    LocalDB.insert('activity_log', {
      timestamp: this.lastActivity,
      type: type,
      value: value
    });

    // 경고 상태 해제
    if (this.alertStatus !== 'normal') {
      this.alertStatus = 'normal';
      this.cancelAlerts();
    }
  }

  // 활동 상태 체크
  async checkActivityStatus() {
    const inactiveTime = Date.now() - this.lastActivity;

    if (inactiveTime >= this.alertThreshold.emergency) {
      // 24시간 미활동 - SMS 발송
      await this.sendEmergencySMS();
    } else if (inactiveTime >= this.alertThreshold.critical) {
      // 12시간 미활동 - 긴급 알림
      await this.sendCriticalAlert();
    } else if (inactiveTime >= this.alertThreshold.warning) {
      // 4시간 미활동 - 경고 알림
      await this.sendWarningAlert();
    }
  }

  // 긴급 SMS 발송 (기기에서 직접)
  async sendEmergencySMS() {
    const contacts = await LocalDB.get('emergency_contacts');
    const user = await LocalDB.get('user_profile');

    const message = `[긴급] ${user.name}님이 24시간 이상 활동이 감지되지 않습니다. 안부를 확인해 주세요. - 안심톡`;

    for (const contact of contacts) {
      await SMS.send({
        to: contact.phone,
        message: message
      });
    }

    // 서버에 이벤트 기록
    await API.post('/notifications/emergency', {
      user_id: user.id,
      type: 'sms',
      sent_at: Date.now()
    });
  }
}

export default MotionDetector;
```

#### 센서 설정 화면

```javascript
// 사용자 맞춤 설정
const sensorSettings = {
  // 감도 설정 (시니어는 낮은 감도 권장)
  sensitivity: {
    low: { acceleration: 1.0, rotation: 0.5 },      // 큰 움직임만 감지
    medium: { acceleration: 0.5, rotation: 0.2 },   // 기본값
    high: { acceleration: 0.2, rotation: 0.1 }      // 미세 움직임도 감지
  },

  // 알림 시간 설정
  alertTiming: {
    warning: 4,    // 시간 (기본 4시간)
    critical: 12,  // 시간 (기본 12시간)
    emergency: 24  // 시간 (기본 24시간) - SMS 발송
  },

  // 활성 시간대 (이 시간대만 모니터링)
  activeHours: {
    start: 7,   // 오전 7시
    end: 23     // 오후 11시
  },

  // 수면 모드 (밤에는 센서 감도 낮춤)
  sleepMode: {
    enabled: true,
    start: 23,  // 오후 11시
    end: 7      // 오전 7시
  }
};
```

### 3.2 기기 직접 SMS 발송

#### SMS 플러그인 구현

```javascript
// plugins/sms-sender.js
// Capacitor용 네이티브 SMS 플러그인

import { Plugins } from '@capacitor/core';

class SMSSender {
  constructor() {
    this.maxRetries = 3;
    this.retryDelay = 5000; // 5초
  }

  // SMS 발송 (기기의 기본 SMS 앱 사용)
  async send(phoneNumber, message, options = {}) {
    const { SMS } = Plugins;

    try {
      // 방법 1: 사용자 확인 없이 직접 발송 (Android)
      if (options.direct && this.isAndroid()) {
        const result = await SMS.sendDirect({
          phoneNumber: phoneNumber,
          message: message
        });
        return result;
      }

      // 방법 2: SMS 앱 열기 (iOS/Android 공통)
      await SMS.open({
        phoneNumber: phoneNumber,
        message: message
      });

      return { success: true, method: 'app' };

    } catch (error) {
      console.error('SMS 발송 실패:', error);

      // 재시도
      if (options.retry < this.maxRetries) {
        await this.delay(this.retryDelay);
        return this.send(phoneNumber, message, {
          ...options,
          retry: (options.retry || 0) + 1
        });
      }

      throw error;
    }
  }

  // 다중 수신자 발송
  async sendToMultiple(contacts, message) {
    const results = [];

    for (const contact of contacts) {
      try {
        const result = await this.send(contact.phone, message, { direct: true });
        results.push({ contact, success: true, result });
      } catch (error) {
        results.push({ contact, success: false, error });
      }

      // 연속 발송 시 딜레이
      await this.delay(1000);
    }

    return results;
  }

  // 긴급 SMS 템플릿
  getEmergencyMessage(userName, lastActivity, contactName) {
    const lastActivityStr = this.formatTime(lastActivity);

    return `[긴급 안부 확인 요청]

${contactName}님, 안녕하세요.

${userName}님이 ${lastActivityStr} 이후
앱에서 활동이 감지되지 않습니다.

안부를 확인해 주세요.

- 안심톡 앱`;
  }

  // Android 직접 발송 권한 체크
  async checkDirectSMSPermission() {
    const { Permissions } = Plugins;

    const status = await Permissions.query({ name: 'SEND_SMS' });

    if (status.state !== 'granted') {
      const request = await Permissions.request({ name: 'SEND_SMS' });
      return request.state === 'granted';
    }

    return true;
  }

  isAndroid() {
    return Capacitor.getPlatform() === 'android';
  }

  delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  formatTime(timestamp) {
    const date = new Date(timestamp);
    const now = new Date();
    const diff = now - date;

    const hours = Math.floor(diff / (1000 * 60 * 60));

    if (hours < 24) {
      return `${hours}시간 전`;
    } else {
      const days = Math.floor(hours / 24);
      return `${days}일 전`;
    }
  }
}

export default new SMSSender();
```

#### Android 네이티브 플러그인 (Java)

```java
// android/app/src/main/java/com/ansimtalk/plugins/SMSPlugin.java

package com.ansimtalk.plugins;

import android.Manifest;
import android.content.pm.PackageManager;
import android.telephony.SmsManager;
import androidx.core.app.ActivityCompat;
import com.getcapacitor.JSObject;
import com.getcapacitor.Plugin;
import com.getcapacitor.PluginCall;
import com.getcapacitor.PluginMethod;
import com.getcapacitor.annotation.CapacitorPlugin;
import com.getcapacitor.annotation.Permission;

@CapacitorPlugin(
    name = "SMS",
    permissions = {
        @Permission(
            strings = { Manifest.permission.SEND_SMS },
            alias = "SEND_SMS"
        )
    }
)
public class SMSPlugin extends Plugin {

    @PluginMethod
    public void sendDirect(PluginCall call) {
        String phoneNumber = call.getString("phoneNumber");
        String message = call.getString("message");

        if (phoneNumber == null || message == null) {
            call.reject("전화번호와 메시지가 필요합니다.");
            return;
        }

        // 권한 체크
        if (ActivityCompat.checkSelfPermission(
                getContext(),
                Manifest.permission.SEND_SMS
            ) != PackageManager.PERMISSION_GRANTED) {
            call.reject("SMS 권한이 없습니다.");
            return;
        }

        try {
            SmsManager smsManager = SmsManager.getDefault();

            // 긴 메시지 분할 처리
            if (message.length() > 160) {
                ArrayList<String> parts = smsManager.divideMessage(message);
                smsManager.sendMultipartTextMessage(
                    phoneNumber,
                    null,
                    parts,
                    null,
                    null
                );
            } else {
                smsManager.sendTextMessage(
                    phoneNumber,
                    null,
                    message,
                    null,
                    null
                );
            }

            JSObject result = new JSObject();
            result.put("success", true);
            result.put("phoneNumber", phoneNumber);
            call.resolve(result);

        } catch (Exception e) {
            call.reject("SMS 발송 실패: " + e.getMessage());
        }
    }
}
```

#### iOS SMS 플러그인 (Swift)

```swift
// ios/App/App/Plugins/SMSPlugin.swift

import Foundation
import Capacitor
import MessageUI

@objc(SMSPlugin)
public class SMSPlugin: CAPPlugin, MFMessageComposeViewControllerDelegate {

    var savedCall: CAPPluginCall?

    @objc func open(_ call: CAPPluginCall) {
        guard let phoneNumber = call.getString("phoneNumber"),
              let message = call.getString("message") else {
            call.reject("전화번호와 메시지가 필요합니다.")
            return
        }

        if !MFMessageComposeViewController.canSendText() {
            call.reject("이 기기에서는 SMS를 보낼 수 없습니다.")
            return
        }

        DispatchQueue.main.async {
            let composer = MFMessageComposeViewController()
            composer.recipients = [phoneNumber]
            composer.body = message
            composer.messageComposeDelegate = self

            self.savedCall = call

            self.bridge?.viewController?.present(
                composer,
                animated: true,
                completion: nil
            )
        }
    }

    public func messageComposeViewController(
        _ controller: MFMessageComposeViewController,
        didFinishWith result: MessageComposeResult
    ) {
        controller.dismiss(animated: true) {
            switch result {
            case .sent:
                self.savedCall?.resolve(["success": true])
            case .cancelled:
                self.savedCall?.reject("사용자가 취소했습니다.")
            case .failed:
                self.savedCall?.reject("SMS 발송 실패")
            @unknown default:
                self.savedCall?.reject("알 수 없는 오류")
            }
            self.savedCall = nil
        }
    }
}
```

### 3.3 백그라운드 서비스

```javascript
// plugins/background-service.js
// 앱이 종료되어도 계속 실행되는 백그라운드 서비스

import { App } from '@capacitor/app';
import { BackgroundTask } from '@capacitor/background-task';
import MotionDetector from './motion-detector';
import LocalDB from './local-db';

class BackgroundService {
  constructor() {
    this.motionDetector = new MotionDetector();
    this.syncInterval = 10 * 60 * 1000; // 10분
  }

  async initialize() {
    // 앱 상태 변화 리스너
    App.addListener('appStateChange', async ({ isActive }) => {
      if (!isActive) {
        // 앱이 백그라운드로 전환
        await this.startBackgroundTask();
      }
    });

    // 센서 모니터링 시작
    await this.motionDetector.startMonitoring();

    // 서버 동기화 시작
    this.startSync();
  }

  async startBackgroundTask() {
    const taskId = await BackgroundTask.beforeExit(async () => {
      // 백그라운드에서 실행할 작업
      console.log('백그라운드 작업 시작');

      // 센서 데이터 계속 수집
      await this.motionDetector.checkActivityStatus();

      // 서버 동기화
      await this.syncWithServer();

      // 작업 완료 알림
      BackgroundTask.finish({ taskId });
    });
  }

  // 서버 동기화
  async startSync() {
    setInterval(async () => {
      await this.syncWithServer();
    }, this.syncInterval);
  }

  async syncWithServer() {
    try {
      // 로컬에 저장된 체크인 기록 가져오기
      const pendingLogs = await LocalDB.get('pending_sync');

      if (pendingLogs.length === 0) return;

      // 서버로 전송
      const response = await fetch(API_URL + '/sync/checkins', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${await LocalDB.get('token')}`
        },
        body: JSON.stringify({ logs: pendingLogs })
      });

      if (response.ok) {
        // 동기화 완료된 로그 삭제
        await LocalDB.clear('pending_sync');
      }
    } catch (error) {
      console.error('동기화 실패:', error);
      // 오프라인 상태 - 다음 동기화 시 재시도
    }
  }
}

export default new BackgroundService();
```

#### Android 포그라운드 서비스 (Java)

```java
// android/app/src/main/java/com/ansimtalk/services/MotionService.java

package com.ansimtalk.services;

import android.app.Notification;
import android.app.NotificationChannel;
import android.app.NotificationManager;
import android.app.Service;
import android.content.Intent;
import android.hardware.Sensor;
import android.hardware.SensorEvent;
import android.hardware.SensorEventListener;
import android.hardware.SensorManager;
import android.os.Build;
import android.os.IBinder;
import androidx.core.app.NotificationCompat;

public class MotionService extends Service implements SensorEventListener {

    private static final String CHANNEL_ID = "MotionServiceChannel";
    private static final int NOTIFICATION_ID = 1;

    private SensorManager sensorManager;
    private Sensor accelerometer;
    private Sensor gyroscope;

    private long lastActivityTime = System.currentTimeMillis();
    private static final float THRESHOLD = 0.5f;

    @Override
    public void onCreate() {
        super.onCreate();

        // 센서 초기화
        sensorManager = (SensorManager) getSystemService(SENSOR_SERVICE);
        accelerometer = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER);
        gyroscope = sensorManager.getDefaultSensor(Sensor.TYPE_GYROSCOPE);

        // 포그라운드 서비스 시작
        createNotificationChannel();
        startForeground(NOTIFICATION_ID, createNotification());

        // 센서 리스너 등록
        sensorManager.registerListener(this, accelerometer,
            SensorManager.SENSOR_DELAY_NORMAL);
        sensorManager.registerListener(this, gyroscope,
            SensorManager.SENSOR_DELAY_NORMAL);
    }

    @Override
    public void onSensorChanged(SensorEvent event) {
        float x = event.values[0];
        float y = event.values[1];
        float z = event.values[2];

        float magnitude = (float) Math.sqrt(x*x + y*y + z*z);

        if (event.sensor.getType() == Sensor.TYPE_ACCELEROMETER) {
            float netAcceleration = Math.abs(magnitude - 9.8f);
            if (netAcceleration > THRESHOLD) {
                recordActivity();
            }
        } else if (event.sensor.getType() == Sensor.TYPE_GYROSCOPE) {
            if (magnitude > THRESHOLD / 5) {
                recordActivity();
            }
        }
    }

    private void recordActivity() {
        lastActivityTime = System.currentTimeMillis();
        // WebView로 이벤트 전달
        Intent intent = new Intent("ACTIVITY_DETECTED");
        intent.putExtra("timestamp", lastActivityTime);
        sendBroadcast(intent);
    }

    private void createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationChannel channel = new NotificationChannel(
                CHANNEL_ID,
                "안심톡 모니터링",
                NotificationManager.IMPORTANCE_LOW
            );
            channel.setDescription("안전을 위해 백그라운드에서 실행 중입니다.");

            NotificationManager manager = getSystemService(NotificationManager.class);
            manager.createNotificationChannel(channel);
        }
    }

    private Notification createNotification() {
        return new NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("안심톡")
            .setContentText("안전 모니터링 중...")
            .setSmallIcon(R.drawable.ic_notification)
            .setPriority(NotificationCompat.PRIORITY_LOW)
            .setOngoing(true)
            .build();
    }

    @Override
    public void onAccuracyChanged(Sensor sensor, int accuracy) {}

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        sensorManager.unregisterListener(this);
    }
}
```

---

## 4. PHP 백엔드 설계

### 4.1 디렉토리 구조

```
ansimtalk-api/
├── public/
│   └── index.php           # 진입점
├── src/
│   ├── Controllers/
│   │   ├── AuthController.php
│   │   ├── CheckInController.php
│   │   ├── FamilyController.php
│   │   ├── NotificationController.php
│   │   └── SyncController.php
│   ├── Models/
│   │   ├── User.php
│   │   ├── CheckIn.php
│   │   ├── EmergencyContact.php
│   │   ├── Family.php
│   │   └── Notification.php
│   ├── Services/
│   │   ├── AuthService.php
│   │   ├── PushNotificationService.php
│   │   └── KakaoAlimtalkService.php
│   ├── Middleware/
│   │   └── AuthMiddleware.php
│   └── Utils/
│       ├── Database.php
│       ├── JWT.php
│       └── Response.php
├── config/
│   ├── database.php
│   └── app.php
├── cron/
│   ├── check_inactive_users.php
│   └── daily_report.php
├── sql/
│   └── schema.sql
├── .htaccess
└── composer.json
```

### 4.2 데이터베이스 스키마

```sql
-- sql/schema.sql

-- 사용자 테이블
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    device_id VARCHAR(255) UNIQUE NOT NULL COMMENT '기기 고유 식별자',
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    email VARCHAR(255),
    fcm_token VARCHAR(255) COMMENT 'Firebase 푸시 토큰',
    settings JSON COMMENT '사용자 설정 (알림 시간, 센서 감도 등)',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    last_activity_at TIMESTAMP COMMENT '마지막 활동 시간 (센서 기반)',
    INDEX idx_device_id (device_id),
    INDEX idx_last_activity (last_activity_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 긴급 연락처 테이블
CREATE TABLE emergency_contacts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) NOT NULL,
    email VARCHAR(255),
    relationship VARCHAR(50) COMMENT '관계 (자녀, 배우자, 친구 등)',
    priority INT DEFAULT 1 COMMENT '알림 순서',
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user_id (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 가족 그룹 테이블
CREATE TABLE families (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    invite_code VARCHAR(20) UNIQUE NOT NULL,
    created_by INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (created_by) REFERENCES users(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 가족 멤버 테이블
CREATE TABLE family_members (
    id INT AUTO_INCREMENT PRIMARY KEY,
    family_id INT NOT NULL,
    user_id INT NOT NULL,
    role ENUM('admin', 'member', 'monitored') DEFAULT 'member' COMMENT '역할',
    joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (family_id) REFERENCES families(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY unique_family_user (family_id, user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 체크인 기록 테이블
CREATE TABLE check_ins (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    check_in_type ENUM('manual', 'auto_sensor', 'voice', 'widget') DEFAULT 'manual',
    sensor_data JSON COMMENT '센서 데이터 (가속도, 자이로 등)',
    location JSON COMMENT '위치 정보 (선택)',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user_created (user_id, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 활동 로그 테이블 (센서 기반)
CREATE TABLE activity_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    activity_type ENUM('acceleration', 'rotation', 'screen_on', 'app_open') NOT NULL,
    value DECIMAL(10, 4) COMMENT '센서 값',
    recorded_at TIMESTAMP NOT NULL,
    synced_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user_recorded (user_id, recorded_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 알림 기록 테이블
CREATE TABLE notifications (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    contact_id INT COMMENT '수신자 (긴급연락처)',
    type ENUM('push', 'sms', 'kakao', 'email') NOT NULL,
    trigger_reason ENUM('manual_checkin', 'auto_checkin', 'warning', 'critical', 'emergency') NOT NULL,
    message TEXT,
    status ENUM('pending', 'sent', 'delivered', 'failed') DEFAULT 'pending',
    sent_at TIMESTAMP,
    delivered_at TIMESTAMP,
    error_message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (contact_id) REFERENCES emergency_contacts(id),
    INDEX idx_user_type (user_id, type),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 사용자 상태 테이블 (실시간 상태 관리)
CREATE TABLE user_status (
    user_id INT PRIMARY KEY,
    status ENUM('normal', 'warning', 'critical', 'emergency') DEFAULT 'normal',
    last_check_in_at TIMESTAMP,
    last_activity_at TIMESTAMP,
    warning_sent_at TIMESTAMP,
    critical_sent_at TIMESTAMP,
    emergency_sent_at TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 4.3 API 엔드포인트

```php
<?php
// public/index.php

require_once __DIR__ . '/../vendor/autoload.php';
require_once __DIR__ . '/../config/database.php';

use App\Utils\Response;
use App\Middleware\AuthMiddleware;

// CORS 헤더
header('Access-Control-Allow-Origin: *');
header('Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS');
header('Access-Control-Allow-Headers: Content-Type, Authorization');
header('Content-Type: application/json; charset=utf-8');

if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') {
    http_response_code(200);
    exit();
}

// 라우팅
$uri = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
$uri = str_replace('/api', '', $uri);
$method = $_SERVER['REQUEST_METHOD'];

// 라우트 정의
$routes = [
    // 인증
    'POST /auth/register' => ['AuthController', 'register'],
    'POST /auth/login' => ['AuthController', 'login'],
    'POST /auth/refresh' => ['AuthController', 'refresh'],

    // 체크인 (인증 필요)
    'POST /checkin' => ['CheckInController', 'store', true],
    'GET /checkin/history' => ['CheckInController', 'history', true],
    'GET /checkin/stats' => ['CheckInController', 'stats', true],

    // 동기화 (인증 필요)
    'POST /sync/activities' => ['SyncController', 'activities', true],
    'POST /sync/checkins' => ['SyncController', 'checkins', true],

    // 가족 (인증 필요)
    'POST /family/create' => ['FamilyController', 'create', true],
    'POST /family/join' => ['FamilyController', 'join', true],
    'GET /family/members' => ['FamilyController', 'members', true],
    'GET /family/dashboard' => ['FamilyController', 'dashboard', true],

    // 긴급연락처 (인증 필요)
    'GET /contacts' => ['ContactController', 'index', true],
    'POST /contacts' => ['ContactController', 'store', true],
    'PUT /contacts/{id}' => ['ContactController', 'update', true],
    'DELETE /contacts/{id}' => ['ContactController', 'destroy', true],

    // 알림 (인증 필요)
    'POST /notifications/emergency' => ['NotificationController', 'emergency', true],
    'GET /notifications/history' => ['NotificationController', 'history', true],

    // 설정 (인증 필요)
    'GET /settings' => ['SettingsController', 'get', true],
    'PUT /settings' => ['SettingsController', 'update', true],
];

// 라우트 매칭 및 실행
$routeKey = "$method $uri";
$matched = false;

foreach ($routes as $route => $handler) {
    $pattern = preg_replace('/\{[^}]+\}/', '([^/]+)', $route);
    $pattern = str_replace('/', '\/', $pattern);

    if (preg_match("/^$pattern$/", $routeKey, $matches)) {
        $matched = true;
        array_shift($matches); // 전체 매치 제거

        $controllerName = "App\\Controllers\\" . $handler[0];
        $methodName = $handler[1];
        $requiresAuth = $handler[2] ?? false;

        // 인증 체크
        if ($requiresAuth) {
            $user = AuthMiddleware::authenticate();
            if (!$user) {
                Response::json(['error' => '인증이 필요합니다.'], 401);
            }
        }

        // 컨트롤러 실행
        $controller = new $controllerName();
        $controller->$methodName(...$matches);
        break;
    }
}

if (!$matched) {
    Response::json(['error' => '엔드포인트를 찾을 수 없습니다.'], 404);
}
```

### 4.4 핵심 컨트롤러

```php
<?php
// src/Controllers/CheckInController.php

namespace App\Controllers;

use App\Models\CheckIn;
use App\Models\UserStatus;
use App\Services\NotificationService;
use App\Utils\Response;

class CheckInController
{
    private $checkIn;
    private $userStatus;
    private $notificationService;

    public function __construct()
    {
        $this->checkIn = new CheckIn();
        $this->userStatus = new UserStatus();
        $this->notificationService = new NotificationService();
    }

    // 체크인 기록
    public function store()
    {
        $user = $GLOBALS['auth_user'];
        $data = json_decode(file_get_contents('php://input'), true);

        // 체크인 저장
        $checkInId = $this->checkIn->create([
            'user_id' => $user['id'],
            'check_in_type' => $data['type'] ?? 'manual',
            'sensor_data' => isset($data['sensor_data']) ? json_encode($data['sensor_data']) : null,
            'location' => isset($data['location']) ? json_encode($data['location']) : null
        ]);

        // 사용자 상태 업데이트
        $this->userStatus->update($user['id'], [
            'status' => 'normal',
            'last_check_in_at' => date('Y-m-d H:i:s'),
            'last_activity_at' => date('Y-m-d H:i:s')
        ]);

        // 가족에게 알림 (설정된 경우)
        if ($user['settings']['notify_family_on_checkin'] ?? false) {
            $this->notificationService->notifyFamily($user['id'], 'checkin');
        }

        Response::json([
            'success' => true,
            'check_in_id' => $checkInId,
            'message' => '체크인이 완료되었습니다.'
        ]);
    }

    // 체크인 히스토리
    public function history()
    {
        $user = $GLOBALS['auth_user'];
        $page = $_GET['page'] ?? 1;
        $limit = $_GET['limit'] ?? 30;

        $history = $this->checkIn->getHistory($user['id'], $page, $limit);

        Response::json([
            'success' => true,
            'data' => $history
        ]);
    }

    // 통계
    public function stats()
    {
        $user = $GLOBALS['auth_user'];
        $period = $_GET['period'] ?? 'week'; // week, month, year

        $stats = $this->checkIn->getStats($user['id'], $period);

        Response::json([
            'success' => true,
            'data' => $stats
        ]);
    }
}
```

```php
<?php
// src/Controllers/SyncController.php

namespace App\Controllers;

use App\Models\ActivityLog;
use App\Models\CheckIn;
use App\Models\UserStatus;
use App\Utils\Response;

class SyncController
{
    private $activityLog;
    private $checkIn;
    private $userStatus;

    public function __construct()
    {
        $this->activityLog = new ActivityLog();
        $this->checkIn = new CheckIn();
        $this->userStatus = new UserStatus();
    }

    // 활동 로그 동기화 (센서 데이터)
    public function activities()
    {
        $user = $GLOBALS['auth_user'];
        $data = json_decode(file_get_contents('php://input'), true);

        if (empty($data['logs'])) {
            Response::json(['error' => '동기화할 데이터가 없습니다.'], 400);
            return;
        }

        $insertedCount = 0;
        $latestActivity = null;

        foreach ($data['logs'] as $log) {
            $this->activityLog->create([
                'user_id' => $user['id'],
                'activity_type' => $log['type'],
                'value' => $log['value'] ?? null,
                'recorded_at' => date('Y-m-d H:i:s', $log['timestamp'] / 1000)
            ]);
            $insertedCount++;

            // 최신 활동 시간 추적
            if (!$latestActivity || $log['timestamp'] > $latestActivity) {
                $latestActivity = $log['timestamp'];
            }
        }

        // 사용자 상태 업데이트
        if ($latestActivity) {
            $this->userStatus->update($user['id'], [
                'last_activity_at' => date('Y-m-d H:i:s', $latestActivity / 1000)
            ]);
        }

        Response::json([
            'success' => true,
            'synced_count' => $insertedCount,
            'message' => "{$insertedCount}개의 활동이 동기화되었습니다."
        ]);
    }

    // 체크인 배치 동기화
    public function checkins()
    {
        $user = $GLOBALS['auth_user'];
        $data = json_decode(file_get_contents('php://input'), true);

        if (empty($data['checkins'])) {
            Response::json(['error' => '동기화할 체크인이 없습니다.'], 400);
            return;
        }

        $insertedCount = 0;

        foreach ($data['checkins'] as $checkin) {
            $this->checkIn->create([
                'user_id' => $user['id'],
                'check_in_type' => $checkin['type'] ?? 'auto_sensor',
                'sensor_data' => isset($checkin['sensor_data']) ? json_encode($checkin['sensor_data']) : null,
                'created_at' => date('Y-m-d H:i:s', $checkin['timestamp'] / 1000)
            ]);
            $insertedCount++;
        }

        // 사용자 상태를 normal로 리셋
        $this->userStatus->update($user['id'], [
            'status' => 'normal',
            'last_check_in_at' => date('Y-m-d H:i:s')
        ]);

        Response::json([
            'success' => true,
            'synced_count' => $insertedCount
        ]);
    }
}
```

### 4.5 Cron 스케줄러

```php
<?php
// cron/check_inactive_users.php
// 매 시간 실행: 0 * * * * php /path/to/cron/check_inactive_users.php

require_once __DIR__ . '/../vendor/autoload.php';
require_once __DIR__ . '/../config/database.php';

use App\Models\User;
use App\Models\UserStatus;
use App\Services\NotificationService;

$userModel = new User();
$userStatus = new UserStatus();
$notificationService = new NotificationService();

// 알림 임계값 (시간)
$thresholds = [
    'warning' => 4,    // 4시간
    'critical' => 12,  // 12시간
    'emergency' => 24  // 24시간
];

$now = new DateTime();

// 모든 활성 사용자 조회
$users = $userModel->getAllActive();

foreach ($users as $user) {
    $status = $userStatus->getByUserId($user['id']);

    if (!$status) continue;

    $lastActivity = new DateTime($status['last_activity_at'] ?? $status['last_check_in_at']);
    $inactiveHours = ($now->getTimestamp() - $lastActivity->getTimestamp()) / 3600;

    // 사용자 설정 확인
    $settings = json_decode($user['settings'], true) ?? [];
    $userThresholds = $settings['alert_thresholds'] ?? $thresholds;

    // 활성 시간대 확인
    $activeHours = $settings['active_hours'] ?? ['start' => 7, 'end' => 23];
    $currentHour = (int) $now->format('H');

    if ($currentHour < $activeHours['start'] || $currentHour >= $activeHours['end']) {
        // 비활성 시간대 - 알림 스킵
        continue;
    }

    // 상태별 처리
    if ($inactiveHours >= $userThresholds['emergency'] && $status['status'] !== 'emergency') {
        // 긴급 상태 - SMS는 기기에서 발송하므로 서버는 상태만 업데이트
        $userStatus->update($user['id'], [
            'status' => 'emergency',
            'emergency_sent_at' => $now->format('Y-m-d H:i:s')
        ]);

        // 푸시 알림 (앱이 SMS 발송하도록)
        $notificationService->sendPush($user['id'], [
            'title' => '긴급: SMS 발송 필요',
            'body' => '장시간 활동이 없습니다. 긴급연락처에 SMS를 발송합니다.',
            'action' => 'send_emergency_sms'
        ]);

        echo "User {$user['id']}: Emergency status set\n";

    } elseif ($inactiveHours >= $userThresholds['critical'] && $status['status'] !== 'critical') {
        // 심각 상태
        $userStatus->update($user['id'], [
            'status' => 'critical',
            'critical_sent_at' => $now->format('Y-m-d H:i:s')
        ]);

        // 가족에게 푸시 알림
        $notificationService->notifyFamily($user['id'], 'critical');

        echo "User {$user['id']}: Critical status set\n";

    } elseif ($inactiveHours >= $userThresholds['warning'] && $status['status'] === 'normal') {
        // 경고 상태
        $userStatus->update($user['id'], [
            'status' => 'warning',
            'warning_sent_at' => $now->format('Y-m-d H:i:s')
        ]);

        // 사용자에게 리마인더
        $notificationService->sendPush($user['id'], [
            'title' => '안부 확인',
            'body' => '오늘 아직 안부를 전하지 않으셨어요. 가족이 걱정하고 있어요.',
            'action' => 'remind_checkin'
        ]);

        echo "User {$user['id']}: Warning status set\n";
    }
}

echo "Check completed at " . $now->format('Y-m-d H:i:s') . "\n";
```

---

## 5. 장기 로드맵

### 5.1 개발 로드맵 (Phase별)

```
┌─────────────────────────────────────────────────────────────┐
│                    장기 개발 로드맵                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [Phase 1: MVP] 출시                                         │
│  ─────────────────────                                      │
│  핵심 기능                                                   │
│  ├── 수동 체크인 (원터치 버튼)                               │
│  ├── 자이로센서 기반 자동 감지 (베타)                         │
│  ├── 기기 직접 SMS 발송                                     │
│  ├── 긴급연락처 관리 (최대 3명)                              │
│  └── 기본 푸시 알림                                         │
│                                                             │
│  기술                                                       │
│  ├── PHP + MySQL 백엔드                                     │
│  ├── Capacitor 하이브리드 앱 (Android 먼저)                  │
│  └── 기본 UI/UX                                             │
│                                                             │
│                                                             │
│  [Phase 2: 안정화]                                          │
│  ─────────────────────                                      │
│  기능 추가                                                   │
│  ├── 자이로센서 감도 최적화                                  │
│  ├── iOS 버전 출시                                          │
│  ├── 위젯 체크인                                            │
│  ├── 가족 연결 (초대 코드)                                   │
│  ├── 카카오톡 알림톡 연동                                    │
│  └── 체크인 히스토리 / 통계                                  │
│                                                             │
│  기술 개선                                                   │
│  ├── 백그라운드 서비스 안정화                                │
│  ├── 배터리 최적화                                          │
│  └── 오프라인 모드 강화                                     │
│                                                             │
│                                                             │
│  [Phase 3: 성장]                                            │
│  ─────────────────────                                      │
│  기능 확장                                                   │
│  ├── 음성 체크인 ("안심톡, 오늘도 괜찮아")                   │
│  ├── 자녀용 대시보드                                        │
│  ├── 양방향 안부 (자녀 → 부모)                              │
│  ├── 프리미엄 구독 모델                                     │
│  ├── 주간/월간 리포트                                       │
│  └── 다국어 지원 (영어, 중국어)                              │
│                                                             │
│  AI 기능                                                    │
│  ├── 활동 패턴 학습 (정상/비정상 판단)                       │
│  ├── 이상 행동 조기 감지                                    │
│  └── 개인화된 알림 시간 추천                                 │
│                                                             │
│                                                             │
│  [Phase 4: 확장]                                            │
│  ─────────────────────                                      │
│  B2G 솔루션                                                 │
│  ├── 관리자 대시보드 (웹)                                   │
│  ├── 돌봄매니저 앱                                          │
│  ├── 다수 사용자 관리                                       │
│  ├── API 제공                                               │
│  └── 119 연계 시스템                                        │
│                                                             │
│  하드웨어 연동                                               │
│  ├── 스마트워치 연동 (삼성, 애플)                            │
│  ├── IoT 센서 연동 (문 열림, 움직임)                         │
│  └── 스마트 스피커 연동                                     │
│                                                             │
│                                                             │
│  [Phase 5: 플랫폼]                                          │
│  ─────────────────────                                      │
│  생태계 구축                                                 │
│  ├── 오픈 API                                               │
│  ├── 파트너 연동 (요양원, 보험사)                            │
│  ├── 건강관리 서비스 연동                                    │
│  └── 커뮤니티 기능                                          │
│                                                             │
│  글로벌 확장                                                 │
│  ├── 일본 시장 진출                                         │
│  ├── 동남아 시장 진출                                       │
│  └── 현지화 (문화, 규제 적응)                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 기능 우선순위 매트릭스

```
┌─────────────────────────────────────────────────────────────┐
│                 기능 우선순위 매트릭스                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│        높은 영향력                                            │
│              │                                              │
│              │  [Must Have]          [High Value]           │
│              │  ● 자이로센서 감지      ○ AI 패턴 학습         │
│              │  ● 기기 SMS 발송       ○ 스마트워치 연동       │
│              │  ● 백그라운드 서비스    ○ 음성 체크인          │
│              │  ● 긴급연락처 관리      ○ B2G 대시보드         │
│              │                                              │
│  낮은 노력 ──┼────────────────────────────── 높은 노력       │
│              │                                              │
│              │  [Quick Wins]          [Nice to Have]        │
│              │  ● 위젯 체크인          ○ IoT 센서 연동        │
│              │  ● 푸시 알림           ○ 커뮤니티 기능         │
│              │  ● 카카오 알림톡       ○ 글로벌 확장          │
│              │  ● 체크인 히스토리      ○ 보험사 연동          │
│              │                                              │
│        낮은 영향력                                            │
│                                                             │
│  ● = Phase 1-2 (필수)                                       │
│  ○ = Phase 3-5 (확장)                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 센서 기반 기능 고도화 로드맵

```
┌─────────────────────────────────────────────────────────────┐
│              센서 기반 기능 고도화 로드맵                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [Level 1: 기본 감지]                                        │
│  ─────────────────────                                      │
│  현재 구현                                                   │
│  ├── 가속도계: 걷기, 이동 감지                               │
│  ├── 자이로스코프: 기기 회전 감지                            │
│  └── 화면 켜짐: 기기 사용 감지                               │
│                                                             │
│  ↓ 진화                                                     │
│                                                             │
│  [Level 2: 패턴 인식]                                        │
│  ─────────────────────                                      │
│  다음 단계                                                   │
│  ├── 일상 패턴 학습 (평균 활동 시간, 수면 시간)              │
│  ├── 이상 패턴 감지 (평소와 다른 행동)                       │
│  ├── 위치 기반 컨텍스트 (집/외출 구분)                       │
│  └── 시간대별 감도 자동 조절                                 │
│                                                             │
│  ↓ 진화                                                     │
│                                                             │
│  [Level 3: AI 분석]                                          │
│  ─────────────────────                                      │
│  고급 기능                                                   │
│  ├── 걸음걸이 분석 (낙상 위험 예측)                          │
│  ├── 활동량 트렌드 분석 (건강 상태 추정)                     │
│  ├── 이상 징후 조기 경보                                    │
│  └── 개인화된 알림 최적화                                   │
│                                                             │
│  ↓ 진화                                                     │
│                                                             │
│  [Level 4: 멀티모달 융합]                                    │
│  ─────────────────────                                      │
│  통합 분석                                                   │
│  ├── 스마트워치 생체 데이터 연동                             │
│  ├── IoT 센서 데이터 통합                                   │
│  ├── 음성/영상 분석 (선택적)                                │
│  └── 종합 건강 스코어링                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. 보안 및 개인정보

### 6.1 보안 설계

```
┌─────────────────────────────────────────────────────────────┐
│                    보안 아키텍처                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [데이터 암호화]                                             │
│  ├── 전송 중: TLS 1.3 (HTTPS 필수)                          │
│  ├── 저장 시: AES-256 암호화 (민감 정보)                     │
│  └── 로컬 DB: SQLCipher 암호화                              │
│                                                             │
│  [인증/인가]                                                │
│  ├── JWT 토큰 기반 인증                                     │
│  ├── Refresh Token 회전                                    │
│  ├── 기기 바인딩 (device_id)                                │
│  └── API Rate Limiting                                     │
│                                                             │
│  [개인정보 보호]                                             │
│  ├── 최소 수집 원칙 (이름, 연락처만)                         │
│  ├── 위치정보 선택적 (명시적 동의)                           │
│  ├── 센서 데이터 익명화                                     │
│  ├── 데이터 보관 기간 제한 (1년)                             │
│  └── 삭제 요청 시 즉시 파기                                  │
│                                                             │
│  [앱 보안]                                                   │
│  ├── 코드 난독화 (ProGuard/R8)                              │
│  ├── 루팅/탈옥 감지                                         │
│  ├── SSL Pinning                                           │
│  └── 안전한 키 저장 (Keystore/Keychain)                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 개인정보 수집 항목

| 항목 | 필수/선택 | 목적 | 보관 기간 |
|------|----------|------|----------|
| 이름 | 필수 | 서비스 제공, 알림 표시 | 탈퇴 시 삭제 |
| 전화번호 | 선택 | SMS 알림 수신 | 탈퇴 시 삭제 |
| 기기 식별자 | 필수 | 기기 인증, 중복 가입 방지 | 탈퇴 시 삭제 |
| 센서 데이터 | 필수 | 활동 감지, 생존 확인 | 30일 |
| 위치 정보 | 선택 | 긴급 상황 시 위치 전달 | 7일 |
| 푸시 토큰 | 필수 | 푸시 알림 발송 | 탈퇴 시 삭제 |

---

## 7. 배포 및 운영

### 7.1 인프라 구성

```
┌─────────────────────────────────────────────────────────────┐
│                    인프라 구성 (저비용)                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [Option 1: 국내 웹호스팅] - 초기 추천                        │
│  ─────────────────────────                                  │
│  ├── 카페24 / 가비아 / 닷홈                                  │
│  ├── PHP + MySQL 기본 제공                                  │
│  ├── 월 5,000원 ~ 30,000원                                  │
│  └── SSL 인증서 포함                                        │
│                                                             │
│  [Option 2: 클라우드 VPS] - 확장 시                          │
│  ─────────────────────────                                  │
│  ├── AWS Lightsail: 월 $5 ~                                │
│  ├── Vultr / DigitalOcean: 월 $5 ~                         │
│  └── NCP (네이버): 월 10,000원 ~                            │
│                                                             │
│  [권장 초기 구성]                                            │
│  ─────────────────                                          │
│  ├── 웹서버: Apache/Nginx + PHP 8.x                         │
│  ├── DB: MySQL 8.x (동일 서버)                              │
│  ├── 스토리지: 10GB (로그, 백업)                             │
│  ├── 도메인: ansimtalk.com 또는 .kr                         │
│  └── SSL: Let's Encrypt (무료)                              │
│                                                             │
│  [예상 월 비용]                                              │
│  ─────────────                                              │
│  ├── 호스팅: 20,000원                                       │
│  ├── 도메인: 1,500원 (연간 18,000원)                        │
│  ├── 카카오 알림톡: 건당 8원 (1만 건 = 80,000원)            │
│  ├── FCM 푸시: 무료                                         │
│  └── 총: 약 100,000원/월 (초기)                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 앱 스토어 배포

```
[Android - Google Play]
├── 개발자 계정: $25 (1회)
├── 심사 기간: 1-3일
├── 필요 권한:
│   ├── SEND_SMS: SMS 직접 발송
│   ├── BODY_SENSORS: 센서 접근
│   ├── FOREGROUND_SERVICE: 백그라운드 실행
│   ├── RECEIVE_BOOT_COMPLETED: 부팅 시 자동 시작
│   └── INTERNET: 네트워크 통신
└── 주의: SMS 권한은 심사 강화됨 (용도 설명 필요)

[iOS - App Store]
├── 개발자 계정: $99/년
├── 심사 기간: 1-7일
├── 제한 사항:
│   ├── SMS 직접 발송 불가 (iOS 정책)
│   └── 대안: MFMessageComposeViewController (앱 열림)
├── 필요 권한:
│   ├── 백그라운드 처리
│   └── 알림
└── 주의: 백그라운드 실행 제한 있음
```

---

## 8. 결론

### 8.1 핵심 차별화 포인트

| 기존 서비스 | 안심톡 차별화 |
|------------|-------------|
| 수동 체크인만 | **자이로센서 자동 감지** |
| 서버 SMS (비용/심사) | **기기 직접 SMS (무료/심사 불필요)** |
| 항상 앱 실행 필요 | **백그라운드 상시 모니터링** |
| 복잡한 설정 | **원터치 설정, 시니어 친화** |
| 고가 구독 | **무료 기본 + 저렴한 프리미엄** |

### 8.2 기술적 도전 과제

1. **배터리 최적화**: 센서 상시 모니터링으로 인한 배터리 소모
2. **iOS 제한**: 백그라운드 실행 및 SMS 직접 발송 제한
3. **센서 정확도**: 오탐지(False Positive) 최소화
4. **Android 버전 파편화**: 다양한 기기 호환성

### 8.3 권장 다음 단계

1. **Capacitor 프로젝트 생성** 및 기본 UI 구현
2. **센서 플러그인 개발** (Android 먼저)
3. **PHP API 서버 구축** (카페24 등 호스팅)
4. **Android 베타 테스트** (센서 정확도 검증)
5. **iOS 버전 개발** (제한 사항 우회 방안 적용)
6. **정식 출시 및 마케팅**

---

*문서 작성일: 2026-01-22*
*버전: 1.0*
*참조: [한국 시장 진출 전략](./korea-market-entry-strategy.md)*
