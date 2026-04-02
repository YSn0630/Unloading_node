# Unloading_node
Turtlebot3 AFL Unloading_node

FSM Align Line-Tracing Unload Node  v7.0
TurtleBot3 Waffle Pi + Forklift Add-on
ROS2 Humble | Remote PC Control

■ v7.0 리팩토링 요약 (v6.7 → v7.0) — 모든 기능 동일, 구조 개선
  ──────────────────────────────────────────────────────────────────────────
  [R1]  dataclass 도입 — 분산된 상태 필드를 논리 단위로 묶음
        · MarkerState       : marker_found / id / top_y / bot_y / cx / tl / tr
        · LidarState        : front_dist / narrow_dist / obstacle_front / pre_estop_state
        · OdomState         : x / y / oz / ow
        · SlipState         : turn_correction / turn_slip_measured /
                              forward_measured / forward_t / forward_d / forward_ratio

  [R2]  _angular() 통합 — _line_angular / _marker_angular 중복 제거
        두 함수가 동일한 수식이었으므로 단일 정적 메서드 _angular()로 통합

  [R3]  _publish_string() 헬퍼 도입
        String 메시지 생성·발행 3줄 패턴을 1줄 호출로 단축
        (pub_fork 'DOWN', pub_done 'Done' 두 곳에서 반복되던 패턴)

  [R4]  _detect_line() 마스크 선택 로직 정리
        단일 구간·복수 구간 분기를 list comprehension + argmax로 단일 경로로 통합
        (동작 동일, 가독성 향상)

  [R5]  _s_approach() 가드 클로즈 구조화
        반환 조건을 위에서 차례로 처리하도록 가드 클로즈 순서 명시적 정렬
        (로직 변경 없음, 흐름 가독성 향상)

  [R6]  _wait_event_frames() 헬퍼 도입
        RETURN / UNLOAD_ALIGN 두 곳에서 반복된 _frame_event.clear() +
        _frame_event.wait() 루프를 단일 메서드로 추출

  [R7]  _unload_phase2_cross() P3/P4 분기 명시적 분리
        시간 초과 후 odom 부족/충분 분기를 elif로 명확히 구분
        (동작 동일, 의도 명확화)

  [R8]  _s_return() 보정 루프를 _trim_rotation() 헬퍼로 추출
        yaw 보정 while 루프를 별도 메서드로 분리하여 _s_return() 길이 단축

  [R9]  상수 명명 일관성 통일
        · TURN_CORRECTION (fallback 상수) / turn_correction (런타임 값)
          → 대문자/소문자 구분은 유지하되 SlipState 필드로 이동하여 혼동 제거
        · _init_params 내 순수 튜닝 상수와 런타임 초기값 분리

  [R10] 모듈 상단 QoS 상수 3개 → _make_qos() 팩토리 함수로 가독성 향상
        반복되는 QoSProfile(...) 키워드 인수를 축약 호출로 정리

■ 핵심 우선순위 원칙 (변경 없음)
  1. 엔드포인트 ArUco 마커 인식 → 즉시 완전 정지 → UNLOAD_ALIGN
  2. 마커 미인식 상태에서만 라인트레이싱 수행
  3. RETURN 진입 시 0·1·2번 마커를 엔드포인트 목록에서 완전 제거
  4. returning=True 구간에서는 라인트레이싱 우선, 마커 무시

■ 마커 ID 등록 규칙
  BLUE → 0 / RED → 1 / YELLOW → 2 / 100 은 항상 엔드포인트



  ## **유한 상태별 동작 요약 (ver.7.0 기준)**

### **[STATE] WAIT**

1. `std_msgs/msg/String` 타입의 `/get_unload` 토픽으로 `"{data: 'BLUE'}"`, `"{data: 'RED'}"`, `"{data: 'YELLOW'}"` 중 하나 **수신 대기**

*WAIT 상태가 아닌 경우, 미션 진행 중첩을 막기 위해 `/get_unload` 토픽을 무시함*

2. 유효한 색상 수신 시, **타겟 마커 ID를 등록**하고 SEARCH 상태로 전환

*( BLUE : ArUco_4x4_0, RED : ArUco_4x4_1, YELLOW : ArUco_4x4_2 )*

### **[STATE] IDLE**

1. **(동작 초기화)** `geometry_msgs/msg/Twist` 타입의 `/cmd_vel` 토픽으로 `"{linear: {x: 0.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}"` 발행하여 로봇 완전 정지 대기

### **[STATE] SEARCH**

1. 시계방향(CW), 반시계방향(CCW)으로 각 최대 1바퀴 회전하며 데이터에 해당하는 색의 라인 탐색
2. 최소 90° 이상 회전 시 **노면의 회전 슬립을 계측하여 보정 계수(`_slip.turn_correction`) 갱신**
2-1. 슬립 측정은 **노드 실행 후 최초 1회에 한하여 실행**되며, 이후 회전 동작에 지속 적용

*회전각이 90° 미만인 상태에서 조기 라인 발견 시, 데이터 부족을 보완하기 위해 CW 45°, CCW 45° 도합 2회 추가 회전하여 계측(`_measure_turn_slip_supplement` 메서드)*

3. 라인 발견 시 즉시 정지 후 APPROACH 상태로 전환

### **[STATE] APPROACH**

1. **귀환 모드(`returning=True`)**인 경우:
1-1. 마커를 무시하고 **라인트레이싱 최우선** 수행
1-2. 중앙 편차가 화면의 10% 초과 시 APPROACH_ALIGN 상태로 전환, **라인 소실 시 DONE 상태**로 전환
2. **마커 우선 모드(`returning=False`)**인 경우:
2-1. 지정된 엔드포인트 ArUco 마커가 인식되면 라인보다 **마커 중앙으로 전진을 우선**함
2-2. 마커 정렬 범위(`bot_y >= MARKER_STOP_BOT_Y`) 도달 시 UNLOAD_ALIGN 상태로 전환
2-3. 마커에 너무 가깝게 붙은 경우(`bot_y >= MARKER_TOO_CLOSE_Y`) 후진(1.0초) 수행
2-4. 마커 미인식 시에는 라인트레이싱 폴백(Fallback) 동작 수행 (편차 10% 초과 시 APPROACH_ALIGN, 라인 소실 시 SEARCH 전환)

### **[STATE] APPROACH_ALIGN**

1. **극저속(`SPD_ALIGN`)으로 회전**하며 라인을 화면 중앙(편차 5% 이내)에 정밀하게 맞춤
2. 귀환 중이 아닐 때 마커가 감지되거나, 중앙 정렬이 완료되면 다시 APPROACH 상태로 복귀
3. 정렬 중 **라인을 완전히 소실하면 귀환 여부에 따라 DONE 또는 SEARCH 상태로 전환**

### **[STATE] EMERGENCY_STOP**

1. `/scan` 토픽 감시 중 **전방 유효 장애물(0.35m 이내) 감지 시 비상 정지** (`_cb_lidar` 콜백에서 트리거)
2. **정지 후 대기**하다가, 장애물이 사라지면 진입하기 직전의 상태(예: APPROACH)로 즉시 복귀

### **[STATE] UNLOAD_ALIGN**

1. ArUco 마커를 기준으로 포크리프트 하역을 위한 **정밀 정렬** 수행 (최대 40회 반복)

*만약 과접근으로 인해 마커가 프레임 바깥까지 빠져나간 경우, 일정한 거리 유지를 위해 0.2초 후진을 반복하며 마커 재탐색 (이 때엔 roi를 사용하지 않고 전체 프레임 이용)*

2. 틸트 오차(TL, TR 마커 꼭짓점의 Y축 단차)와 중앙 위치(CX) 오차를 계산하여 각각 **독립적으로 회전 보정**
3. **두 조건이 모두 허용 오차 내에 들어오면 정렬 성공**으로 간주하고 UNLOAD_STANDBY 상태로 전환

### **[STATE] UNLOAD_STANDBY**

1. **전진 슬립 계측 여부(`_slip.fwd_measured`)**에 따라 1단계 또는 2단계 전진 분기 수행
1-1. 1단계 (초기): **LiDAR ±5° 중앙값을 측정**하여 목표 거리(28cm)까지 전진. **실제 소요 시간(`fwd_t`)과 이동 거리(`fwd_d`)를 저장하여 슬립을 계측**
1-2. 2단계 (이후): 1단계에서 **측정한 시간 및 Odom 데이터를 바탕으로 교차 감시 전진** 수행. 이상 주행 시 강제 정지(E-STOP) 또는 조기 정지 처리
2. 지정 거리 도달 완료 후 `/fork_cmd` 토픽으로 'DOWN' 명령 발행
3. `/fork_done` 응답 수신 대기 (타임아웃 10초) 후, 완료 시 안전거리를 확보하기 위해 5초간 후진
4. 후진 완료 후 RETURN 상태로 전환

### **[STATE] RETURN**

1. 완료된 마커(0, 1, 2)를 엔드포인트에서 완전히 제거하고 귀환 플래그(`returning=True`) 활성화
2. 1차: **측정된 슬립 보정 계수(`_slip.turn_correction`)를 반영**하여 시간 기반으로 180° 회전
3. 2차: `_trim_rotation` 헬퍼 메서드를 통해 **Odom Yaw 변화량을 교차 검증**
3-1. 회전 오차가 허용 범위(±5°) 이내면 보정 통과
3-2. 부족하거나 과도하게 회전된 경우 **최대 15초 동안 잔여 오차만큼 역/순방향 추가 보정 회전 수행**
4. 회전 완료 후 **카메라 프레임 안정화를 위해 2프레임 대기** 후 APPROACH 상태로 복귀

*180° 회전 시 불필요한 외부 영상처리를 줄이기 위해 `/image_compressed` 토픽 구독을 일시적으로 중단*

### **[STATE] DONE**

1. 목표 색상, 엔드포인트, 귀환 상태, 마커 감지 등 **런타임 변수 모두 초기화**
2. 모터 정지 후 **`/done` 토픽으로 'Done' 메시지 발행**
3. IDLE 상태를 거쳐 최종적으로 WAIT 상태로 복귀하여 다음 명령을 대기
