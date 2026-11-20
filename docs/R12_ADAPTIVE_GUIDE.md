# SOC R12 Adaptive Quality-Aware UWB 최신 가이드

## 기준본과 수정 범위
R11.5.8(Dual-Pair) 버전을 기준으로 수정했다.
기존 3D 모델/UI/AI 추천/Surface Normal 변위 계산/고정 bias 보정/필터/ADXL345/UDP/SSE는 유지하고, UWB ranging 선택과 TDMA 완료 조건을 R12 Adaptive 구조로 교체했다.

중요: R11.5.8 UI는 `dual_pair_ab`를 요구했지만 Tag 펌웨어는 실제로 R11.5.7 4-Anchor full-ranging 형식이라 서로 프로토콜이 맞지 않았다. R12에서는 서버와 Tag를 `slot_protocol:3`, `ranging_mode:adaptive_quality_v1`로 통일했다.

## R12 동작
- Anchor 4개는 모두 물리적으로 유지한다. Anchor 펌웨어는 Wi-Fi 없이 기존 UWB responder 그대로 사용한다.
- NORMAL: 서버가 현재 Tag 좌표 + Surface Normal + Anchor 좌표 + 최근 성공률 + 거리 잡음 + residual + CIR/NLOS 품질을 점수화하여 가장 좋은 2개 Anchor를 선택한다.
- VERIFY3: 품질이 약간 나빠지거나 VERIFY4 이후 회복 구간이면 3개를 사용한다.
- VERIFY4: ADXL 진동/가속도 이벤트, 측정 실패, residual/uncertainty 증가, 강한 NLOS 의심, 또는 주기적 health sweep에서 4개를 사용한다.
- 주기적 Full Sweep: 정상 상태여도 약 2.6초마다 한 번 4개를 모두 확인해 link health와 bias 적용 상태를 갱신한다.
- 로컬 즉시 escalation: 서버가 2개를 지시했더라도 Tag가 같은 cycle 안에서 ADXL 이벤트, 60 mm 이상의 갑작스러운 range jump, 강한 CIR NLOS 의심, 2-link 확보 실패를 보면 3\~4개로 즉시 확대한다.
- 정기 Full Sweep에서 나쁜 Anchor가 발견돼도 VERIFY4에 영구 고정되지 않는다. 서버는 마지막 실제 CIR/NLOS 진단을 기억해 그 Anchor 점수를 낮추고, 다음 NORMAL에서는 다른 조합을 선택한다.
- 실제 사용하지 않은 Anchor는 `scheduled:false`로 보내므로 실패 통계에 포함되지 않는다.

## Anchor 선택 점수
서버는 각 Anchor에 다음 요소를 합친 점수를 준다.
1. Surface Normal 방향 Jacobian |dr/dd|: 변위를 얼마나 민감하게 보는가.
2. 최근 UWB 성공률.
3. 보정/실시간 거리 잡음.
4. DW3000 CIR 기반 NLOS score.
5. 직전 기하 residual / outlier 패널티.
2개 또는 3개를 선택할 때는 단일 점수뿐 아니라 선택 Anchor 사이의 Jacobian 다양성도 함께 평가한다.

## DW3000 CIR 품질
Tag는 성공한 ranging 직후 `dwt_readdiagnostics()`로 Ipatov CIR 정보를 읽는다.
- First Path amplitude 힌트
- peak amplitude
- First Path ↔ peak sample gap
- 보수적인 `nlos_score` / `nlos_suspect`
이 값은 거리를 강제로 보정하는 용도가 아니다. 신뢰도를 낮추거나 추가 Anchor 검증을 트리거하는 용도로만 사용한다. Surface-Normal solver에도 NLOS score 기반 de-weight를 적용해 의심 링크가 최종 Δn을 지배하지 않게 했다.

## TDMA 변경
- Protocol 3 START 명령에 `anchor_mask`, `target_links`, `range_profile`을 포함한다.
- live 상태는 유효 packet 1개를 받으면 다음 Tag로 handoff하여 고정 6 packet 대기보다 갱신 지연을 줄였다.
- calibration 상태는 각 Tag 차례에 4-Anchor packet을 최대 4개 모아 보정 속도와 Tag 간 공정성을 같이 유지한다.
- 기본 timing: prepare 140 ms, live max 320 ms, calibration max 680 ms, handoff guard 35 ms.
- Tag 내부 목표 cycle: 2 Anchor 70 ms / 3 Anchor 85 ms / 4 Anchor 105 ms.

## 펌웨어 업로드
- TAG-01 → `ESP_control/ESP_control.ino`
- TAG-02 → `ESP_control2/ESP_control2.ino`
- TAG-03 → `ESP_control3/ESP_control3.ino`
- ANCHOR-01\~04 → 기존 `if_ANCHOR1~4` 파일 그대로

Tag 시리얼 정상 예시:
- `[TDMA] START ...`
- `[UWB] ADAPTIVE requested=2 actual=2 mask=0x... event=NO`
- `[UDP] tag_data SENT ... adaptive=2 | ranges=2/4 ...`
이벤트 시 `adaptive=3` 또는 `adaptive=4`, reason이 `imu_event_verify`, `range_jump_verify`, `nlos_backup`, `link_recovery` 등으로 바뀐다.

## 실행
1. UI 폴더에서 `npm install`
2. `npm start`
3. `http://localhost:8080`
4. Tag Wi-Fi SSID/PASSWORD와 SERVER_IP 확인
5. 센서 매핑 → 추천 좌표 동기화 → Zero/보정 → 실시간 모니터링

## 이번 버전에서 아직 하지 않은 것
IEEE 802.15.4z one-to-many DS-TWR로 MAC 자체를 재설계하는 단계는 포함하지 않았다. 현재는 기존 responder 기반 ranging을 유지한 채 Anchor 선택/검증/TDMA를 적응형으로 바꾼 버전이다. 이 편이 기존 하드웨어/검증 코드에 대한 변경 위험이 훨씬 작다.
