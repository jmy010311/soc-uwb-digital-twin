# DW3000 ↔ SOC Digital Twin 통신 규격

이 문서는 이후 ESP32-C3 + DW3000 펌웨어가 따라야 할 **서버 통신 인터페이스**입니다.

> R5 실제 구성에서는 `TAG-01`만 Wi-Fi/UDP를 사용합니다. Anchor 4개는
> UWB 전용 응답기이며, 서버는 Tag의 `ranges`에서 응답한 Anchor를 자동으로
> 온라인 등록합니다. 아래 Anchor UDP packet은 이전 펌웨어 호환용입니다.

DW3000에서 실제 거리 측정을 어떤 라이브러리/DS-TWR 코드로 구현하든,
최종적으로 이 JSON 형식만 UDP로 전송하면 웹 디지털 트윈과 연결됩니다.

---

## 1. 네트워크

### ESP → PC 서버
- 방식: UDP
- 목적지: 디지털 트윈 서버 PC의 IPv4
- 기본 포트: `4210`

### PC 서버 → ESP
- 방식: UDP
- 기본 포트: `4211`

환경변수로 변경 가능:

```powershell
$env:SENSOR_UDP_PORT="4210"
$env:SENSOR_COMMAND_PORT="4211"
```

---

## 2. Node ID 규칙

권장:

```text
TAG-01
TAG-02
TAG-03
...

ANCHOR-01
ANCHOR-02
ANCHOR-03
...
```

각 물리 모듈은 항상 같은 `node_id`를 사용해야 합니다.

---

## 3. Tag 데이터 packet

Tag는 DW3000 거리 결과와 ADXL345 데이터를 하나의 packet으로 보냅니다.

```json
{
  "type": "tag_data",
  "node_id": "TAG-01",
  "role": "tag",

  "seq": 1234,

  "tilt_deg": 1.25,
  "vibration": false,
  "dynamic_g": 0.041,

  "uwb_active": true,

  "ranges": [
    {
      "anchor_id": "ANCHOR-01",
      "ok": true,
      "m": 2.431
    },
    {
      "anchor_id": "ANCHOR-02",
      "m": 3.205
    },
    {
      "anchor_id": "ANCHOR-03",
      "m": 4.012
    },
    {
      "anchor_id": "ANCHOR-04",
      "m": 3.558
    }
  ],

  "wifi_rssi": -48,
  "cmd_port": 4211,
  "fw": "dw3000-tag-v1"
}
```

### 필수 필드

- `type`: 반드시 `"tag_data"`
- `node_id`
- `role`: `"tag"`
- `ranges`: 측정된 Anchor 거리 목록

R5는 매 cycle마다 Anchor 4개 항목을 모두 보냅니다. 현재 cycle에서 실패한
Anchor는 `"ok":false, "m":null`로 보내고, `range_count`에는 성공한 개수만
기록합니다. 따라서 화면의 `N / 4`는 실제 최신 UWB 응답 수입니다.


## 4. Anchor 상태 packet

Anchor는 상태 확인을 위해 주기적으로 아래 packet을 보냅니다.

```json
{
  "type": "anchor_status",
  "node_id": "ANCHOR-01",
  "role": "anchor",

  "short_addr": "0xA001",
  "wifi_rssi": -51,
  "cmd_port": 4211,
  "fw": "dw3000-anchor-v1"
}
```

또는 초기 접속 시:

```json
{
  "type": "hello",
  "node_id": "ANCHOR-01",
  "role": "anchor",
  "cmd_port": 4211,
  "fw": "dw3000-anchor-v1"
}
```

---

## 5. Tag 초기 hello

Tag도 Wi-Fi 연결 직후 아래 형식으로 서버에 존재를 알릴 수 있습니다.

```json
{
  "type": "hello",
  "node_id": "TAG-01",
  "role": "tag",
  "cmd_port": 4211,
  "wifi_rssi": -49,
  "fw": "dw3000-tag-v1",
  "imu_ok": true
}
```

---

## 6. 서버가 ESP에 보내는 명령

### 6.1 Tilt Zero

웹에서 `센서값 0`을 누르면 Tag에 전송됩니다.

```json
{
  "cmd": "zero_imu"
}
```

향후 Tag 펌웨어는 이 명령을 받으면 현재 ADXL345 자세를 0° 기준으로 저장하면 됩니다.

UWB range baseline은 Node 서버에서 별도로 저장합니다.

---

### 6.2 UWB time slot

다중 Tag 충돌 방지용 선택 기능입니다.

```json
{
  "cmd": "uwb_slot",
  "active": true
}
```

또는:

```json
{
  "cmd": "uwb_slot",
  "active": false
}
```

`UWB_TDMA=1`로 서버를 시작한 경우 사용합니다.

Tag는:
- `active:true` → DW3000 ranging 수행
- `active:false` → DW3000 ranging 시작/송신 정지

로 처리하면 됩니다.

ADXL345 측정과 Wi-Fi 상태 전송은 슬롯과 무관하게 계속 유지하는 것을 권장합니다.

---

## 7. Node 서버가 Δz를 계산하는 방식

Tag가 보내야 하는 것은 **Anchor별 현재 거리**입니다.

Tag가 직접 `z`를 계산할 필요는 없습니다.

Node 서버는 웹에서 알고 있는:

```text
Tag 추천 위치
Anchor 추천 위치
```

와 Zero 시점 UWB 거리를 사용합니다.

실제 단위 GLB/DXF일 경우:

```text
현재 range
- Zero range
+ 모델상의 초기 Anchor↔Tag 거리
        ↓
수평거리 제거
        ↓
수직 성분 계산
        ↓
Δz
```

형태로 상대 높이 변화를 추정합니다.

따라서 이후 DW3000 펌웨어는 **안정적인 DS-TWR range를 meter 단위로 전달하는 것**에 집중하면 됩니다.

---

## 8. 전송 주기 권장

프로토타입 시작 권장:

### ADXL / Tag packet

```text
5 ~ 10 Hz
```

### DW3000 range

처음에는 안정성을 위해 그보다 느려도 됩니다.

예:

```text
각 Tag가 0.5 ~ 2 Hz 수준으로 4 Anchor ranging 완료
```

다중 Tag 안정화 후 주기를 올리는 방식이 좋습니다.

---

## 9. 온라인 판정

서버 기본값:

```text
4초 이상 packet 없음 → Offline
```

변경:

```powershell
$env:NODE_OFFLINE_MS="6000"
```

---

## 10. 서버 API

실제 모듈 목록:

```text
GET /api/modules
```

센서 현재값:

```text
GET /api/sensors/state
```

실시간 스트림:

```text
GET /api/sensors/stream
```

브라우저는 SSE(Server-Sent Events)로 이 endpoint를 사용합니다.

---

## 11. 중요한 점

이 통신 규격은 DW3000의 ranging 구현과 분리되어 있습니다.

즉 아래 부분은 향후 ESP 펌웨어에서 자유롭게 결정할 수 있습니다.

- Qorvo DW3000 driver
- DS-TWR initiator/responder 구현
- ranging schedule
- antenna delay calibration
- UWB channel / preamble 설정

단, 측정 결과를 이 문서의 UDP JSON 규격으로 넘기면
현재 Node.js + Web Digital Twin 코드는 그대로 사용할 수 있습니다.
