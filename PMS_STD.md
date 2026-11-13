# PMS_STD

이 문서는 PMS 표준 PLC 코드(PmsStd)를 처음 보는 사람과, 이어서 작업하는 AI·채팅이 함께 읽는 참고 자료다. 처음 보는 사람은 **1장(안내) → 3장(구조) → 4장(폴더와 파일)** 순서로 읽으면 큰 그림이 잡힌다. 그 뒤는 필요할 때 찾아보는 자료다: 원칙(2장), 설정 위치(5장), 확장과 자주 하는 작업(6장), 동작 규칙(7장), 이름 규칙(8장), 코드 작업 방법(9장), 남은 일(10장), 자주 받는 질문(11장), 인터페이스·구조체 참조(12장). AI는 질문에 답할 때 11장 "답하는 방법"을 따른다. 0장은 이 코드를 고치는 사람·AI가 지킬 작업 규칙이다. 현장별 요구사항(특정 현장의 지령원 우선순위, 특정 장비 맵 등)은 이 문서에 넣지 않는다.

- 대상: TwinCAT 3.1.4024 기반 PMS 표준 PLC 코드. 보호 조건·대응 모듈, SOC 제한, EMS 지령원, 장비 드라이버와 접속점 그룹 단위 제어 모드를 다룬다. 제어 모드는 7장, 시험 기록과 남은 검증은 10장을 참고한다.
  - 기록된 가상 시험 결과는 제어 모드 자동 시험 42건과 실경로 시험 3종이다 (10장). 이 기록은 이후 변경된 코드 전체의 빌드·동작 검증을 뜻하지 않는다.
  - 실제 장비 등록·시험 모드·장비 쓰기 여부는 `90_Project/GVL_SITE`를 확인한다. 이 작업본의 현장별 장비 구성과 검증 범위는 별도 `README_UPS_DRIVER.md`에 기록한다. 표준의 가상 PCS·BMS·계측기와 제어 모드 코드는 재사용을 위해 유지하며, 선언만 남아 있는 장비를 설치된 장비로 간주하지 않는다.
  - MQTT 로거 발행(`pms_log` 토픽, `20_Module/Mqtt/Logger`)과 로그 문구 길이 규칙(7장)을 더했다. 토픽은 `dgspms/ups/{사이트}/pms_log` (`GVL_MQTT.SOLUTION_NAME = 'ups'`, 사이트는 `GVL_SITE.MQTT_SET.nSiteId`, 지금 3). 이중화면 Active PLC만 로그를 보낸다.
  - **로거 MQTT(`pms_log`)를 쓰려면 제어기 TwinCAT 런타임이 3.1.4024.17 이상이어야 한다** (`FB_ListenerBase2`). 낮으면 구독이 `16#98110701`(서비스 미지원)로 실패해 로그가 하나도 나가지 않는다. 메인 머지 전 코드에서 4024.17로 리스너 수신·발행을 확인했다 (4장 "로그가 EMS까지 가는 길"). 머지 뒤 빌드·시험은 아직이다.
  - **이중화는 PLC 2대 실기로 시험했다** (2026-10-28, `DEVICE_WRITE = FALSE`): 통신 단선·상대 정지 판정, 시작 순서, 역할 충돌, 절체(Primary 정지 → Secondary가 약 1 s에 Active), 복귀(약 10 s 뒤 Failback), 설정 동기화, 멈췄다 이어짐, 이벤트 로그. 7장 "이중화". 램프 시작값·Manual 유닛 승격 동작과 MQTT 이중화 대응은 시험하지 않았다 (10장).

---

## 0. 작업 규칙 (반드시 지킴)

| 항목 | 내용 |
|---|---|
| 커밋 | 파일만 고친다. 커밋·푸시는 사용자가 한다 (git 이력이 백업) |
| 지울 파일 | 직접 삭제하지 않고 따로 옮겨 둔다. 사용자가 직접 지운다 |

- **코드는 사용자가 "수정해"(또는 같은 뜻으로 분명히 요청)할 때만 고친다.** 확인할 것은 먼저 말하고, 확인받은 뒤 고친다.
- 빌드는 사용자가 한다. 빌드 오류가 없으면 다음 단계로 넘어간다.
- 설명은 직관적·간단·명확하게. 제목은 명사로 끝낸다. 약어는 `약어(한국어, English)`.
- **단어는 듣기만 해도 뜻을 알 수 있게 고른다** (2026-11-09). 결과 보고·문서·코드 주석을 새로 쓸 때 적용한다.
  - "버스" → **배터리 그룹** (같은 DC 버스에 물린 배터리 묶음, `nBatGroup`)
  - "그룹"·"제어 그룹" → **접속점 그룹** (같은 접속점 하나를 함께 맞추는 유닛 묶음, `nGroup`)
  - 그 밖에도 두 가지 이상으로 읽힐 수 있는 줄임말은 풀어 쓴다.
  - **이미 있는 이름은 바꾸지 않는다**: 코드 식별자(`GROUP_CFG`·`nGroup`·`nBatGroup`·`g_astGroupSet` 등), 기능 이름(버스 제한 모듈 등), 이미 쓴 문서 문장. 이런 이름을 인용할 때는 뜻을 같이 적는다 (예: "접속점 그룹(`nGroup`)", "배터리 그룹 한도를 줄이는 버스 제한 모듈(`I_BatLimitRule`)").
- 이름은 8장 규칙을 따른다 (특히 Auto·Manual 이름 원칙).
- 하지 말 것 (요청 전까지): `POINT_TERM_MAX` 단순화, 보고 매핑 리팩터.

---

## 1. 처음 보는 사람을 위한 안내

이 장은 PmsStd를 처음 여는 사람이 30분 안에 큰 그림을 잡도록 쓴 것이다. 자세한 규칙은 뒤 장에 있다.

### PMS가 하는 일

PMS(전력 관리 시스템, Power Management System)는 ESS(에너지 저장 장치, Energy Storage System) 현장에서 **"누가 시킨 출력을, 장비가 낼 수 있는 만큼, 안전하게 나눠서 내보내는"** PLC 프로그램이다.

```
 위에서 시키는 쪽                    PMS (이 코드)                         아래 장비
┌──────────────┐        ┌───────────────────────────────┐        ┌─────────────────┐
│ EMS (상위)   │──지령─▶│ 1. 장비 상태 읽기              │──P·Q──▶│ PCS (전력변환)  │
│ 운전원 (HMI) │        │ 2. 보호·가용 판단              │──명령─▶│ BMS (배터리)    │
│ 스케줄       │        │ 3. 지령 고르기·다듬기          │◀─상태──│ 계측기 (전력계) │
│ 알고리즘     │        │ 4. 유닛마다 나누기             │──DO───▶│ 보조 I/O (VCB,  │
└──────────────┘        │ 5. 장비에 쓰기                 │◀─DI/AI─│  팬, 화재 접점) │
        ▲               │ 6. 기록·진단·보고              │        └─────────────────┘
        └───── 보고 ────┴───────────────────────────────┘
```

- **P**는 유효전력(kW), **Q**는 무효전력(kvar)이다. 방전·공급 방향이 +다.
- PMS는 10 ms마다 위 1~6을 한 번씩 반복한다. 한 번 도는 것을 **스캔**이라고 한다.

### 이 코드의 생각: 코어 + 모듈 + 설정

현장마다 장비 제조사, 대수, 운전 방식, 보호 항목이 다르다. 예전에는 현장마다 코드를 복사해서 고쳤다. PmsStd는 이것을 세 층으로 나눈다.

| 층 | 폴더 | 비유 | 현장마다 바꾸는가 |
|---|---|---|---|
| 코어 | `10_Core` | 자동차의 엔진·변속기. 모든 현장이 같은 것을 쓴다 | 바꾸지 않는다 |
| 모듈 | `20_Module` | 끼웠다 빼는 부품 (타이어, 내비게이션). 정해진 모양(인터페이스)에 맞으면 무엇이든 끼운다 | 필요한 것만 고르거나 새로 만든다 |
| 설정 | `90_Project` | 차량 등록증·옵션표. 몇 대인지, 무엇을 끼웠는지, 숫자 값 | 현장마다 고친다 |

- 코어는 "흐름"만 안다. 예를 들어 코어는 "PCS"라는 모양(`I_Pcs`)만 알고, 플라스포 PCS인지 가상 PCS인지는 모른다.
- 모듈은 코어가 정해 둔 자리(확장 지점)에 끼운다. 모듈은 장비에 직접 쓰지 않고 "요청"이나 "한도"만 낸다. 최종 결정과 장비 쓰기는 코어가 한 곳에서 한다.
- 설정은 `GVL_` 파일들이다. 장비 대수, 장비끼리 어떻게 묶였는지, 어떤 모듈을 끼웠는지를 적는다.

### 용어

| 용어 | 뜻 | 코드에서 |
|---|---|---|
| PCS (전력변환장치, Power Conversion System) | 배터리 직류를 교류로 바꿔 계통에 내보내거나 반대로 충전하는 장비 | `I_Pcs`, `g_aipPcs` |
| BMS (배터리 관리 장치, Battery Management System) | 배터리 SOC·전압·온도를 감시하고 충·방전 한도를 알려 주는 장비. 접촉기(배터리 투입·개방)를 가진다 | `I_Bms`, `g_aipBms` |
| SOC (충전 상태, State Of Charge) | 배터리에 남은 에너지 비율 (%) | `ST_BatState` |
| 유닛 | PMS가 출력 지령을 따로 주는 가장 작은 단위. 보통 PCS 1대 = 유닛 1개. PCS 1대가 출력을 2개로 나눠 받으면 유닛 2개 | `UNIT_CFG`, 유닛 번호 `u` |
| 제어 그룹 (새 글에서는 **접속점 그룹**) | 같은 접속점 하나를 함께 맞추는 유닛 묶음. 그룹 지령(예: 그룹 전체 1000 kW)은 그룹 단위로 받는다. 새로 쓰는 글에서는 "접속점 그룹"이라고 쓴다 (0장) | `GROUP_CFG`, 그룹 번호 `g` |
| 접속점 (PCC, Point of Common Coupling) | 그룹이 계통과 만나는 지점. 피크저감·역송 방지 같은 제한은 이 지점의 전력으로 판단한다 | `ST_GroupCfg.nPccPoint` |
| 계측기 / 계측점 | 계측기 = 실제 전력계 장비. 계측점 = 계측기 값을 부호와 함께 더해 만든 "보고 싶은 지점의 값" | `I_Meter` / `POINT_CFG` |
| 배터리 그룹 (DC 버스) | 직류 쪽에서 같은 버스에 물린 배터리 묶음. 같은 버스의 유닛은 충·방전 방향이 같아야 하고 한도를 나눠 쓴다. 새로 쓰는 글에서는 "버스" 대신 "배터리 그룹"이라고 쓴다 (0장) | `nBatGroup`, 번호 `b` |
| 보조 장비 | 디지털·아날로그 점 몇 개만 주고받는 장비 (VCB 접점, 팬, 화재 감지기, 온도 센서) | `I_AuxDevice`, `FB_AuxIo` |
| 지령원 | 출력·상태를 시키는 쪽 하나. EMS, 운전원, 스케줄, 알고리즘이 각각 지령원이다 | `I_CommandSource`, `g_aipSource` |
| 운전 모드 (Auto / Manual) | 유닛이 어느 지령원의 말을 듣는지. Auto = 자동 지령원(EMS·스케줄·알고리즘), Manual = 운전원 | `E_OpMode`, `fbModeSwitch.q_aeOpMode` |
| 경로 | Auto 유닛과 Manual 유닛은 같은 로직을 따로 한 번씩 돈다. 이 두 줄기를 경로라고 한다 | `fbAutoShape` / `fbManualShape` |
| 운전 상태 | 장비의 실제 상태. Stop(정지) / Standby(대기: 활성인데 출력 0) / Run(활성: 출력 지령을 받음, 크기 무관) / Fault(고장) / Unknown(모름). Total Interface 정의와 같다 | `E_RunState` |
| 상태 요청 | "Run 해라", "Standby 해라" 같은 상태 바꾸기 요청. 출력 값(kW)과 따로 다룬다 | `ST_PcsReq`, `FB_StateArbiter` |
| 출력 지령 | "몇 kW 내라"는 값 | `ST_SourceOutput` |
| 제어 모드 | Auto에서 "무엇을 기준으로 출력을 정하나"의 운전 방식 하나 (정출력, 스케줄, 피크컷, 통합 등). 한 관리자 안에서 한 번에 하나만 켠다 | `I_ControlMode`, `20_Module/ControlMode` |
| 제어 모드 관리자 | 제어 모드를 등록받아 하나만 켜고 전환을 맡는 지령원. 코어에는 지령원 하나로 보인다. 기본은 접속점 그룹마다 하나 (2026-11-09부터. 예전에는 PCS마다 하나) | `FB_CtrlModeMgr`, `g_fbCtrlMode1` |
| 보호 | 조건이 걸리면 장비 상태·배터리·모드를 바꾸는 판단. 칸마다 조건 모듈 + 대응 모듈 (기본 5종 조건과 표준 대응은 표준 모듈로 기본 등록) | `FB_Protection`, `I_ProtectionRule`, `I_ProtResponse` |
| 제한 | 출력 크기(P·Q 범위)만 줄이는 것. 장비 명령이 없고 범위가 열리면 바로 돌아온다 (SOC 감발, 사용자 한도, 그룹 제한) | `FB_EquipmentLimits`, 성형 |
| 가용 | 유닛이 지금 쓸 수 있는 상태인가 (통신·고장·로컬·배터리·보호 이상 없음) | `FB_UnitAvail` |
| 출력 가능 | 가용 + 장비 활성(Run·대기) + 정지 진행 중 아님 + 모드 전환 아님 + Manual 새 지령 대기 아님. 이것이 참이어야 출력을 준다 | `FB_OutputReady` |
| 활성·대기 | Run = 활성(출력 지령을 받음, 크기 무관). 대기 = 활성 + 출력 0. PMS 안에 대기 명령은 따로 없다 (0을 받은 드라이버가 장비 맵에 맞게 씀) | — |
| 설비 한도 | 장비가 물리적으로 낼 수 있는 범위 (정격, 보호 감발, BMS 한도, SOC 정책) | `FB_EquipmentLimits` |
| Derate (감발) | 보호 때문에 정격을 일부만 쓰게 줄이는 것 (예: 온도 높음 → P 정격 50 %). P·Q 따로 | `ST_ProtCfg.fPPct`·`fQPct` |
| 성형 | 그룹 전체가 "얼마" 낼지 정하는 단계 (램프 → 접속점 제한 → 보정 → 설비 제한) | `FB_ShapePath` |
| 분배 | 성형된 그룹 값을 유닛마다 "누가 얼마" 낼지 나누는 단계 | `FB_Distributor` |
| 램프 | 출력을 한 번에 바꾸지 않고 정해진 속도(kW/s)로 바꾸는 것 | `ST_GroupSet.fRampUp` |
| 모듈 ID | 모듈마다 붙이는 번호. 로그에 "누가 막았는지"를 남길 때 쓴다. 0은 코어 | `i_nModuleId` |
| 등록 배열 | 모듈 인스턴스를 넣는 배열. 칸에 넣으면 동작하고, 비워 두면 그 기능은 없다 | `g_aip…` (`GVL_SITE`) |
| 등록 설정 | 등록 배열의 같은 칸 번호에 대응하는, 운전 중 바꾸는 설정 (켜기·그룹·Auto/Manual 적용) | `g_ast…Reg`, `g_ast…Cfg` (`GVL_PERSIST`) |

### 처음 읽는 순서

1. **`90_Project/GVL_PARAM`** — 장비가 몇 대이고 모듈 칸이 몇 개인지. 숫자만 있어 가장 짧다.
2. **`90_Project/GVL_SITE`** — 유닛·BMS·계측점·그룹이 어떻게 묶였는지(`UNIT_CFG` 등), 어떤 장비·모듈 인스턴스를 만들어 어느 등록 배열에 넣었는지.
3. **`90_Project/MAIN`** — 한 스캔에 어떤 FB가 어떤 순서로 불리는지. 주석의 ⓪~⑥ 번호가 이 문서 3장 "한 스캔"과 같다.
4. 이 문서 **1장 "한 스캔 따라가기"** 예시를 보면서, 궁금한 FB를 하나씩 연다. FB 첫 줄 주석에 그 FB가 하는 일이 한 줄로 적혀 있다.
5. 동작이 왜 그런지는 **7장 동작 규칙**, 모듈을 붙이려면 **6장 확장**을 본다.

온라인으로 볼 때는 `MAIN`의 인스턴스(`fbProtection`, `fbDistributor`, `fbOutputDiag` 등)를 펼치면 된다. 출력이 왜 줄었는지는 `fbOutputDiag.q_astAutoPTrace[그룹]`(출력 추적표)에서 한 줄씩 보인다.

### 한 스캔 따라가기 (예시)

현장: 제어 그룹 1개, 유닛 2개(각 정격 1000 kW, 같은 배터리 버스), BMS 1대, 접속점 계측기 1대. 두 유닛 모두 Auto이고 Run 중이다. EMS가 그룹 방전 1500 kW를 보냈다. BMS는 방전 한도 1200 kW를 알려 온다.

| 단계 | FB | 이 스캔에 하는 일 | 결과 |
|---|---|---|---|
| ⓪ | `FB_ConfigCheck` | 설정 점검 | 오류 없음 → 장비 쓰기 허가 |
| ① | 드라이버 `M_Cycle` → `FB_Snapshot` → `FB_Measure` | 장비를 읽고 사본을 만든다. 접속점 값 계산 | 유닛 1·2 Run, BMS 방전 한도 1200 kW, SOC 50 % |
| ② | `FB_ModeSwitch` | 운전 모드 전환 중인지 | 둘 다 Auto, 전환 없음 |
| ② | `FB_Protection` | 고장·통신·현장 보호 판단 | 보호 없음 |
| ② | `FB_BatGroup` → `FB_UnitAvail` | BMS 쓸 수 있는지, 유닛 가용인지 | 둘 다 가용 |
| ② | `FB_EquipmentLimits` | 설비 한도 계산 | 유닛 한도 각 ±1000, 버스 방전 한도 1200 → 그룹 설비 범위: 정격 줄 2000, 배터리 줄 1200 |
| ③ | `FB_GroupView` → `FB_SourceHub` | 그룹 요약을 만들고 지령원을 부른다 | EMS 지령원: 그룹 P 1500 |
| ④ | `FB_StateArbiter` · `FB_Sequencer` | 상태 요청 처리 (받은 명령은 한 번 보냄) | 새 상태 요청 없음. 두 유닛 활성이라 지령만 씀 |
| ④ | `FB_Select` | 지령원 중에서 고른다 (Auto 경로) | Auto 그룹 P 1500 |
| ④ | `FB_Gate`(Auto) → `FB_OutputReady` | 지령 거르기, 출력 가능 판단 | 통과, 두 유닛 출력 가능 |
| ④ | `FB_LimitEval` | 제한·보정 모듈 평가 | 등록된 모듈 없음 |
| ④ | `fbManualShape` → `fbAutoShape` | 성형: 램프 → 접속점 제한 → 보정 → 설비 제한 | Manual 몫 0. Auto: 1500 → (램프 꺼짐) 1500 → 1500 → 1500 → **설비 제한 1200** (배터리 줄) |
| ⑤ | `FB_Distributor` | 유닛에 나눈다 (`eDistBasis = Rating`) | 유닛 1: 600, 유닛 2: 600. 버스 남은 한도 1200 안 |
| ⑥ | `FB_Sequencer` → `FB_Dispatcher` | 상태 명령, P·Q 쓰기 | Auto라 200 ms마다 600 kW씩 씀 |
| 옆 | `FB_OutputDiag` · `FB_EventLog` | 왜 줄었는지 남긴다 | 막은 단계 `DeviceLimit`, 이유 "배터리 충·방전 한도 합". 10 s 넘게 유지되면 "출력 제한" 로그 1건 |

이어서 몇 가지가 바뀌면 이렇게 달라진다.

- **유닛 2의 PCS 통신이 끊김** → `FB_UnitAvail`에서 유닛 2 가용 아님 (통신 끊김은 사실), PCS 통신 보호(표준 대응 대기)가 알람 → 그룹 정격 줄 1000 → 성형 결과 1000 → 분배: 유닛 1에 1000, 유닛 2는 0 (이유 `NotAvail`).
- **운전원이 유닛 2를 Manual로 바꾸고 300 kW 지령** → `FB_ModeSwitch`가 유닛 2를 출력 0 → 대기(출력 0) 확인 → Manual로 전환. 전환이 끝나면 유닛 2는 Manual 경로, 운전원이 새 지령을 보내야 출력한다 (장비는 활성 그대로). Manual 경로를 먼저 계산(300)한다. Auto 그룹 지령은 접속점 기준(Manual 몫까지 포함한 그룹 전체 값)이라, Auto 경로는 1500을 그룹 전체 목표로 보고 제한한 뒤 Manual 몫을 뺀다 → 배터리 줄 1200 − Manual 300 = Auto(유닛 1) 900 (Manual 우선). 배터리 한도가 없었다면 Auto 목표는 1500 − 300 = 1200이지만, 실제 유닛 1 출력은 정격 한도 1000 kW까지다.
- **SOC가 5 % 아래로 내려감** → 버스 제한 `FB_SocLimit`가 방전 한도를 0으로 막음 → 그 버스 유닛은 방전 출력 0 (= 대기, 장비 명령 없음). SOC가 재개 마진(2 %)만큼 돌아오면 다시 방전한다. "다시 방전하지 않음"이 필요하면 보호(조건 SOC 하한 + Manual 전환 또는 정지)로 붙인다.
- **설정 실수 (`UNIT_CFG`에 같은 PCS·nSub 두 번)** → `FB_ConfigCheck`가 오류 → 쓰기 허가 FALSE → 읽기·보호·진단·로그는 계속하지만 PCS·BMS 쓰기와 보조 DO 켜기는 막는다 (DO 끄기는 허용). "설정 오류로 장비 제어 중지" 알람이 뜬다.

### 저장소 폴더 맨 위

```
pms-standard/
├─ PMS_STD.md              이 문서
├─ PmsStd.sln               Visual Studio(TwinCAT XAE) 솔루션. 이것을 연다
└─ PmsStd/
   ├─ PmsStd.tsproj         TwinCAT 시스템 프로젝트 (태스크, I/O, 이벤트 클래스 정의)
   ├─ _Config/              TwinCAT이 만드는 I/O 설정 폴더
   ├─ _Boot/                빌드·활성화 때 TwinCAT이 만드는 부트 파일 (공유하지 않음)
   └─ PmsStd/               PLC 프로젝트
      ├─ PmsStd.plcproj     PLC 프로젝트 파일. 모든 소스 파일이 여기 등록돼야 빌드에 들어감
      ├─ PlcTask.TcTTO 외   태스크 3개 정의 (PlcTask, MqttTask, RedundancyTask)
      ├─ 10_Core/           코어 (4장)
      ├─ 20_Module/         모듈 (4장)
      ├─ 90_Project/        현장 설정과 MAIN (4장)
      ├─ DUTs/ GVLs/ POUs/ VISUs/   TwinCAT 기본 폴더. 비어 있음 (쓰지 않음)
      ├─ _Libraries/        라이브러리 캐시 (TwinCAT 관리)
      └─ _CompileInfo/      빌드 중간 파일 (공유하지 않음)
```

---

## 2. 원칙

- **출력이 막히면 왜 막혔는지 바로 보여야 한다.** 가장 중요한 원칙이다.
- 현장 차이는 설정(`90_Project`의 GVL), MAIN(`90_Project`), 모듈(`20_Module`)로만 처리한다. 코어(`10_Core`)는 모든 현장에서 고치지 않는다. 코어를 복사하거나 현장별로 분기하지 않는다.
- **MAIN은 현장 수정 자리다.** 새 장비 종류의 `M_Cycle`, 현장 모듈 설정 넘기기(`A_PassModuleSet`), 중계처럼 현장마다 다른 일은 MAIN(또는 MAIN에 선언한 모듈)에서 한다.
- **경로는 지령 주체(Auto/Manual) 하나로만 가른다.** GFL/GFM은 PCS 제어 방식이라 유닛 속성(`ST_UnitState.eGridMode`)으로 두고, 계통 연계/독립은 나중에 그룹 상태로 둔다. 그래서 기본 상태가 Auto/Manual × GFL/GFM 네 조합으로 늘지 않는다. 기본은 GFL이고, GFM은 PCS에 맡긴다 (계통 연계 중 GFM은 PMS 계산이 GFL과 같음: P·Q 지령, 전압·주파수 드룹은 PCS가 처리).
- **칸을 비우면 미설치다.** 유닛의 PCS 칸, BMS 칸이 비어 있으면 보호·가용·설정 점검·배터리 집계에서 뺀다 (모니터링 전용 납품 등).
- 기능은 모듈로 붙였다 뗀다. 알고리즘·제한·보호 모듈이 하나도 없어도 코어는 그대로 돈다.
- **보호와 제한은 결과로 가른다.** 장비 상태·배터리·모드를 바꾸면 보호, 출력 크기(P·Q 범위)만 줄이면 제한이다. 원인(이상 상황인지)으로 가르지 않는다. 다만 이상 상황이라 알람을 남겨야 하는 감발은 보호의 범위 요청으로 붙인다. 출력 크기는 설비 한도 한 곳, 장비 상태는 중재 한 곳에서 만난다.
- 설정이 틀리면 PCS·BMS 쓰기와 보조 DO 켜기를 막는다. 읽기·보호·진단·로그는 계속하고, 보조 DO 끄기는 허용한다. 쓰기 허가의 전체 조건은 3장 `bWriteEnable`을 따른다. 대기 PLC에도 같은 DO 끄기 예외가 적용된다. 운전 중 HMI로 바꾸는 설정 값은 잘못 넣으면 마지막 정상값으로 되돌린다 (`FB_SettingGuard`). 되돌릴 정상값이 없을 때만 설정 오류로 쓰기를 막는다.
- Auto와 Manual은 같은 로직을 두 경로로 돈다. Manual은 그룹 정책(`ST_ManualPolicy`)으로 단계를 끌 수 있을 뿐, 따로 만든 코드가 아니다.
- 한 FB는 한 역할만 한다. 같은 계산·판단은 한 곳에서만 하고, 필요한 FB는 그 결과를 받아 쓴다 (예: 가용 한도 = `FB_EquipmentLimits`, 출력 가능 = `FB_OutputReady`, 등록 모듈 목록 = `FB_ModuleList`).
- 처음 보는 사람이 이해할 수 있게 쓴다. 이름은 뜻이 드러나게 짓고, 쓰이는 변수만 두고, 같은 데이터를 중복해 두지 않는다. 진단 값으로 제어하지 않는다 (제어에 쓰는 값은 따로 낸다).
- **코드는 사람이 읽기 쉽게 단순하게 쓴다.** 현장에 문제가 생기면 사람이 코드를 열어 원인을 찾는다. 같은 동작이면 짧고 곧은 쪽을 고르고, 길어지는 본문은 액션으로 나눈다. 꼭 필요하지 않은 기능·설정·기록은 넣지 않는다.
- 로컬 장비는 기다리지 않는다. PCS가 로컬(PMS 제어를 안 받음)이면 가용이 아니다. 출력은 0이 되고, 기동 요청은 거부하며, 모드 전환은 Standby를 기다리지 않고 끝낸다. 리모트 신호가 없는 장비는 리모트로 본다 (`ST_UnitState.bCtrlReady` 기본 TRUE).

| 규칙 | 내용 |
|---|---|
| R1 | 출력을 정하는 곳은 제어 그룹·경로마다 한 곳이다 (중재·성형). |
| R2 | 장비에 쓰는 곳은 장비 종류마다 한 곳이다. PCS·BMS는 `FB_Dispatcher`, 보조 장비 DO는 `FB_AuxOutput`, 새 장비 종류는 그 장비 모듈(드라이버 `M_Cycle`). |
| R3 | 보호·알고리즘·외부 명령은 "요청"만 낸다. |
| R4 | 현장 차이는 설정으로 처리한다. |
| R5 | 출력이 요청보다 작으면 막은 단계·주체·이유가 남는다. 로그는 이유가 바뀔 때만 남긴다. |
| R6 | 상위 인터페이스는 보고 전용이다. 제어 판단에 쓰지 않는다. |

---

## 3. 구조

### 장비 계층

```
PCS ──(1:n)── 유닛 ──(n:1)── 제어 그룹 ── 접속점(계측점)
                │
                └──(n:1)── 배터리 그룹(DC 버스) ──(1:m)── BMS

계측기 ──(합, 부호 ±1)── 계측점 ── 그룹 접속점 / 기타 계측값
```

- 유닛은 출력 지령을 따로 받는 제어 단위다. 기본은 PCS다. `UNIT_CFG`의 PCS 번호와 PCS 안 번호(`nSub`)로 정한다.
- 제어 그룹은 접속점 1개와 소속 유닛으로 이뤄진다.
- 배터리 그룹은 같은 DC 버스에 물린 배터리 묶음이다. 유닛 n : BMS m을 지원한다. 유닛은 `UNIT_CFG.nBatGroup`, BMS는 `BMS_CFG.nBatGroup`으로 같은 번호를 가리킨다. 0은 배터리 없음이다.
- 계측점은 계측기 값의 부호 있는 합이다 (`POINT_CFG`). 계측기 설치 방향 차이는 `fSign`으로 맞춘다.
- 유닛의 경로(Auto/Manual)는 유닛의 실제 운전 모드(`fbModeSwitch.q_aeOpMode`)로 정해진다. 한 그룹에 두 경로 유닛이 섞일 수 있다.

### 한 스캔 (PlcTask 10 ms, `MAIN`)

```
⓪ 시험 연결·설정 점검   PRG_SimLink·PRG_Test… (TEST_MODE만) → fbClock(현지 시각) → A_PassModuleSet → FB_ModuleList → FB_SettingGuard → FB_ConfigCheck (매 스캔)
① 읽기·스냅샷·계측      장비 M_Cycle → FB_Snapshot → FB_Measure
② 모드·보호·가용·한도    FB_ModeSwitch → FB_Protection → FB_BatGroup → FB_UnitAvail    ◀ I_ProtectionRule, I_ProtResponse
                        → FB_EquipmentLimits                                            ◀ I_UnitLimitRule, I_BatLimitRule
③ 요청                  FB_GroupView → FB_UnitView → FB_SourceHub                       ◀ I_CommandSource
                          (지령원 중 FB_CtrlModeMgr가 켜진 제어 모드 하나를 부름)           ◀ I_ControlMode
④ 상태 요청             FB_PermitEval → fbSequencer.M_Prepare → fbStateArb(후보) → fbSequencer.M_CheckRequest
                        → fbStateArb.M_CommitRequest                                    ◀ I_StartPermit
  출력 지령 (그룹마다)    FB_Select                                                       ◀ I_SelectPolicy
                        → FB_Gate (Auto·Manual 인스턴스 둘)                              ◀ I_Gate
                        → FB_OutputReady → FB_LimitEval
                        → fbManualShape → fbAutoShape (둘 다 FB_ShapePath)              ◀ I_LimitRule, I_Compensator
⑤ 분배 (그룹마다)        FB_Distributor (Manual 경로 → Auto 경로 → 배터리 버스 → 피상전력) ◀ I_DistPolicy
⑥ 쓰기                  fbSequencer(본체, 안에서 FB_BatRequest) → FB_GroupState → FB_Dispatcher → FB_AuxOutput                ◀ I_OutputRule
옆                      FB_AliveRecorder · FB_PersistSave · FB_OutputDiag · FB_EventLog · FB_Report
```

MAIN이 한 번 만들어 여러 FB에 같이 넘기는 값:

| 값 | 내용 |
|---|---|
| `bTestReset` | 시험 리셋 = `TEST_MODE` AND `GVL_TEST.g_bTestReset`. 시퀀서가 모든 PCS·BMS에 고장 리셋 명령 (지령원 리셋은 시퀀서가 장비별로 봄) |
| `bClearHoldDo` | 유지 DO 풀기 = 시험 리셋 OR 지령원 PCS 리셋 요청(`fbSources.q_bReset`, 지난 스캔). 보조 출력이 씀. 고장 리셋과 따로 정하는 것은 DO 작업 때 (10장) |
| `bWriteEnable` | 장비 쓰기 허가 = `fbConfigCheck.q_bConfigOk` AND `GVL_IF.g_bRedWrite` AND `GVL_SITE.DEVICE_WRITE`. `g_bRedWrite`는 Active가 되고 쓰기 대기 시간이 지난 뒤 TRUE다 (이중화 미사용 시 TRUE). FALSE면 PCS·BMS 쓰기와 시퀀서 명령 생성을 막고, 기동 요청은 `WriteDisabled`로 거부한다 (설정 오류는 `ConfigError` 우선). 보조 DO는 켜는 쓰기만 막고 끄는 쓰기는 허용한다. |
| `afZeroGrp` | Manual 성형에 넘기는 "다른 경로 몫" (항상 0). |

FB마다 하는 일은 4장 폴더별 표에 있다.

### 성형과 분배

```
FB_ShapePath  (fbManualShape 먼저 → fbAutoShape)
  유닛별 값 : 램프만 (유닛 램프율 = 그룹 램프율 × 유닛 정격 / 그룹 정격 합). 유닛 한도는 분배에서
             출력을 못 내는 유닛(멈춤·모드 전환·Manual 새 지령 대기)은 0 → 기동하면 0부터 램프
  P        : 목표 → 램프 → 접속점 제한 → +보정 → 설비 제한
             설비 제한 = 그룹 설비 범위(정격 줄·배터리 줄) + 설비 기준 제한 모듈
             목표 = 이 경로 그룹 지령 (접속점 기준 = 그룹 전체 값). 없으면 이 경로 유닛 지령 합 (유닛별 램프 후) + 다른 경로 몫
             Auto  : 그룹 전체 값으로 제한하고, 결과에서 Manual 결과(iq_afOtherGrpP)를 뺀 것이 Auto 몫
             Manual: 다른 경로 몫 0. g_astManualPolicy에서 켠 단계만
  Q        : 고르기 → 응답 필터 → 제한 (그룹 Q 범위 + 설비 기준 모듈 Q + 접속점 기준 모듈 Q)
             목표 = 그룹 지령(필터 후 + 보정). 없으면 유닛 지령 합 + 다른 경로 몫

FB_Distributor  (그룹마다, P·Q 각각. M_DistPath는 두 경로가 같은 순서)
  버스 남은 한도 = 버스 한도 (방전·충전 쪽 따로). 유닛에 P를 줄 때마다 줄어듦 (여러 그룹이 같은 버스를 써도 한 번만)
  Manual 경로 → Auto 경로
    ① 유닛별 값 (유닛 한도 → 버스 남은 한도 안으로)
    ② 그룹 몫 = 경로 목표 − ①   그룹 지령이 있을 때만. 목표와 반대 방향 금지
                               유닛 한도·버스 남은 한도에 걸린 몫은 다른 유닛에 재분배
                               한 버스로 가는 몫이 남은 한도보다 크면 그 버스 유닛 몫을 같은 비율로 줄임
    ③ 그래도 넘치면 유닛별 값을 비율로 축소
    Auto 목표 = Auto 성형 결과 + Manual 부족분 (Auto 그룹 지령이 있을 때만)
  ④ 배터리 버스 마지막 확인: 순 방향과 반대인 유닛 0 → 순 한도로 비율 축소 (보통은 줄일 것 없음. BMS 한도를 끈 Manual 유닛은 빠짐)
  ⑤ 피상전력: 우선이 아닌 쪽을 줄임 (재분배 없음, 정격을 끈 Manual 유닛은 건너뜀)
```

---

## 4. 폴더와 파일

폴더 번호는 스캔 순서와 같다. 위에서 아래로 읽으면 한 스캔을 따라가는 셈이다. 파일 이름 앞글자로 종류를 알 수 있다.

| 앞글자 | 종류 | 파일 확장자 |
|---|---|---|
| `FB_` | 기능 블록 (상태를 가진 부품) | `.TcPOU` |
| `F_` | 함수 (상태 없음, 같은 입력 → 같은 결과) | `.TcPOU` |
| `PRG_` | 프로그램 (태스크가 부름) | `.TcPOU` |
| `I_` | 인터페이스 (끼우는 자리의 모양) | `.TcIO` |
| `ST_` | 구조체 (값 묶음) | `.TcDUT` |
| `E_` | 열거형 (정해진 선택지) | `.TcDUT` |
| `GVL_` | 전역 변수 목록 (설정·인스턴스·등록) | `.TcGVL` |

### 한눈에 보기

```
10_Core/              코어. 현장마다 고치지 않는다
├─ 00_Type            공통 타입 (구성·설정·상태·요청 구조체, 열거형, 코어 이유 번호)
├─ 01_Interface       끼우는 자리의 모양 (장비·지령원·제어 모드·제한·보호 등 인터페이스 18개)
├─ 02_Util            작은 도구 (필터, 램프 한 걸음, 읽기 주기, PERSISTENT 저장, HHMM 시각 계산)
├─ 03_Base            모듈이 상속하는 기반 FB 4개 (모듈·장비·지령원·제어 모드)
├─ 10_Snapshot        ① 읽기: 장비 상태 사본, 계측점 계산
├─ 20_Protection      ② 보호·가용·설비 한도
├─ 30_Request         ③ 요청 모으기: 그룹·유닛 요약, 지령원 부르기, 제어 모드 관리자
├─ 40_Arbitration     ④ 판단: 운전 모드 전환, 상태 중재, 지령 고르기·거르기, 출력 가능, 성형
├─ 50_Distribution    ⑤ 분배: 그룹 값을 유닛에 나눔
├─ 60_Dispatch        ⑥ 쓰기: 시퀀스(기동·정지 순서), 장비 쓰기, 보조 DO 쓰기
├─ 70_Diag            진단·로그·설정 점검·모듈 목록·재기동 기록
├─ 80_Report          보고 자리 (상위 인터페이스로 내보내기)
└─ 90_Redundancy      이중화 (상대 PLC 판정, 상태 기계, 설정 동기화)
20_Module/            모듈. 필요한 것만 골라 쓴다
├─ Common             여러 현장이 같이 쓰는 모듈 (보조 I/O, 접점·아날로그 규칙, SOC 정책, 현지 시각)
├─ ControlMode        제어 모드 (정출력, 스케줄, 피크컷, 통합)
├─ Standard           표준 보호 모듈 5종
├─ Driver             실제 장비 드라이버 (제조사별 하위 폴더)
├─ Mqtt               MQTT 보고·명령 (PMS ↔ EMS)
│  ├─ Publish         발행 패킷
│  ├─ Config          config 패킷
│  └─ Logger          이벤트 로그 발행 (pms_log 토픽)
├─ Report             보고 매핑 (장치 값 → 발행 구조체)
└─ Virtual            사무실 시험용 가상 장비·지령원
90_Project/           현장 설정과 MAIN. 현장마다 고친다
99_Test/              시험 프로그램 (TEST_MODE일 때 MAIN이 부름, 로거 시험 PRG_TestLogger는 PRG_Mqtt가 부름. 시험 버튼·bStart로만 동작). 현장 빌드에서 뺀다
```

### 10_Core/00_Type — 공통 타입

코어와 모듈이 같이 쓰는 데이터 모양이다. 여기 있는 구조체는 "무엇을 주고받는지"의 약속이라 함부로 바꾸면 모듈이 깨진다.

| 묶음 | 파일 | 내용 |
|---|---|---|
| 구성 (재빌드로 바꿈) | `ST_UnitCfg` `ST_BmsCfg` `ST_PointCfg` `ST_PointTerm` `ST_GroupCfg` | 유닛이 어느 PCS·버스·그룹에 속하는지와 정격, BMS가 어느 버스인지, 계측점을 어떤 계측기로 만드는지, 그룹 접속점이 어느 계측점인지 |
| 운영 설정 (운전 중 바꿈) | `ST_GroupSet` `ST_ManualPolicy` `ST_ModuleReg` `ST_RuleTargetReg` `ST_SourceReg` `ST_StateModeSet` `ST_PcsCmdMode` `ST_BatCmdMode` `ST_ProtCfg` `ST_RuleCfg` `ST_OutputCfg` | 그룹 램프·분배 기준, Manual 정책, 모듈 등록(켜기·그룹·Auto/Manual), 지령원 등록(등급·받는 모드·만료), 보호 항목 설정, 출력 규칙 설정 |
| 현장 상수 설정 | `ST_ModeSet` `ST_LogSet` `ST_RedSet` | 모드 전환 단계, 로그, 이중화. `GVL_SITE`에서 정하고 재빌드 |
| 운전 중 변경 설정 | `ST_CmdFilterSet` `ST_ManualCmdSet` `ST_SeqSet` `ST_PcsSeqSet` `ST_BatSeqSet` | 이상값·Manual 지령 거르기, 시퀀스 시간·재시도. `GVL_PERSIST`에서 관리 |
| 장비 상태 (표준 모양) | `ST_UnitState` `ST_BatState` `ST_MeterState` `ST_PccMeas` `ST_Val` | 드라이버가 채우는 표준 상태. 코어는 이것만 본다 (제조사 차이는 드라이버가 흡수) |
| 요청·명령 | `ST_SourceOutput` `ST_PcsReq` `ST_BatReq` `ST_UnitReq` `ST_PcsCmd` `ST_BatCmd` `ST_PQ` | 지령원이 내는 출력(P·Q·상태 요청·P·Q 제한), 장비에 보낼 명령 |
| 모듈에 넘기는 요약 (읽기 전용) | `ST_GroupView` `ST_UnitView`(유닛 하나 기준, 유닛 단위 제어 모드용) `ST_UnitLimitView` `ST_BatLimitView` `ST_SelectView`(+ 후보 `ST_SelectCand`) `ST_DistView` `ST_GateView` `ST_ProtView`, Gate 결과 `ST_GateOut`, 보호 대응 요청 `ST_ProtReq` | 모듈이 판단할 때 보는 값. 모듈은 이것만 보고 다른 모듈을 직접 부르지 않는다 |
| 모듈 결과 | `ST_Limit` `ST_LimitOut` `ST_BatLimitOut` | 제한 모듈이 돌려주는 상·하한과 이유 |
| 제어 모드 | `ST_CtrlModeSet` `ST_CtrlModeView` `ST_CtrlModeOut` `ST_LocalTime` `E_CtrlModeStep` `E_CtrlModeRefuse` `E_PccShare` | 관리자 설정(전환·대체 모드·접속점 몫), 모드에 넘기는 값(그룹 또는 유닛 요약 + 현지 시각), 모드가 돌려주는 P·Q와 스스로 끝냄, 관리자 단계, 요청 거부 이유, 유닛 단위 접속점 몫 방식 |
| 열거형 | (제어 모드 열거형은 위 줄) `E_OpMode`(Auto/Manual) `E_GridMode`(GFL/GFM, 유닛 속성) `E_RunState`(운전 상태) `E_PcsCmd` `E_BatCmd`(명령) `E_PcsAction` `E_BatAction`(보호 조치) `E_StartMode`(기동 방식) `E_BatClose`(배터리 투입 시점) `E_DistBasis`(분배 기준) `E_SelectMode`(지령원 고르는 방식) `E_SourceMode`(받는 모드) `E_LimitBasis`(접속점 기준/설비 기준) `E_GroupState` `E_ProtScope` `E_RedRole`(이중화 역할) | 선택지 |
| 이유 번호 | `GVL_REASON` | 코어가 출력을 막았을 때의 이유 번호 (`RSN_RATING` 정격, `RSN_BAT` 배터리, `RSN_RATING_Q`, `RSN_DERATE`) |
| 코어 상수 | `GVL_CORE` | 코어 내부 상수 (`NO_LIMIT`: 정격·배터리가 없는 유닛·버스의 대용 한도). 현장에서 바꾸지 않는다 (현장 대수·크기는 `GVL_PARAM`) |

### 10_Core/01_Interface — 끼우는 자리

모든 모듈은 `I_Module`(모듈 ID·이름·이유 문구)을 기본으로 갖고, 역할에 따라 아래 중 하나 이상을 구현한다. 이 표가 확장 지점 목록이다 (붙이는 순서는 6장, 메서드는 12장).

| 인터페이스 | 무엇을 끼우나 | 코어가 부르는 곳 | 기반 FB |
|---|---|---|---|
| `I_Module` | 모든 모듈의 공통 (ID, 이름, `M_ReasonText`, 설정 점검 `M_CheckSet`) | `FB_ModuleList`, `FB_ConfigCheck`, `FB_SettingGuard`, `FB_EventLog` | `FB_ModuleBase` |
| `I_Pcs` `I_Bms` `I_Meter` `I_AuxDevice` | 장비 드라이버, 보조 I/O | MAIN ① `M_Cycle`, `FB_Snapshot`(읽기), `FB_Dispatcher`·`FB_AuxOutput`(쓰기) | `FB_DeviceBase` |
| `I_CommandSource` | 지령원 (외부 명령, 운전원, 스케줄, 알고리즘 묶음) | `FB_SourceHub` | `FB_SourceBase` |
| `I_ControlMode` | 제어 모드 하나 (정출력·스케줄·피크컷·통합 등). 관리자가 한 번에 하나만 켬 | `FB_CtrlModeMgr` (지령원) | `FB_CtrlModeBase` |
| `I_LimitRule` | 그룹 출력 제한 (접속점 기준 / 설비 기준: 피크저감, 역송 방지 등) | `FB_LimitEval` | `FB_ModuleBase` |
| `I_Compensator` | 보정량 (그룹 출력에 더함) | `FB_LimitEval` | `FB_ModuleBase` |
| `I_StartPermit` | 기동 조건 (Run 허가) | `FB_PermitEval` | `FB_ModuleBase` |
| `I_ProtectionRule` | 보호 조건 (기본 5종은 표준 모듈, 현장 보호: 화재·온도 등). 대상마다 판단 | `FB_Protection` | `FB_ModuleBase` |
| `I_ProtResponse` | 보호 대응: 같은 칸 조건이 걸리면 무엇을, 누구에게, 어떤 순서로 요청할지 (`ST_ProtReq`). 기본은 표준 대응 `FB_ProtRespStd` | `FB_Protection` | `FB_ModuleBase` |
| `I_ReqDevice` | 요청을 받는 새 장비 종류 (모양만. 등록·코어 연결은 장비가 생길 때. 6장 "기존 종류에 안 맞는 새 장비") | — | `FB_ModuleBase` |
| `I_UnitLimitRule` | 유닛 P·Q 한도 (사용자 한도 등) | `FB_EquipmentLimits` | `FB_ModuleBase` |
| `I_BatLimitRule` | 배터리 버스 충·방전 한도 (SOC 감발 등) | `FB_EquipmentLimits` | `FB_ModuleBase` |
| `I_SelectPolicy` | 지령원 후보 중 값 하나 정하기 (고르기·합치기). 그룹마다 한 칸 `g_aipSelectPolicy[그룹]`, 비우면 코어 기본 | `FB_Select` | `FB_ModuleBase` |
| `I_DistPolicy` | 그룹 몫을 유닛에 나눌 무게. 그룹마다 한 칸 `g_aipDistPolicy[그룹]`, 비우면 코어 기본 | `FB_Distributor` | `FB_ModuleBase` |
| `I_Gate` | 경로별 지령 거르기: 접수·보류·무시·새 지령 대기 (상태를 가져도 됨). `g_ipAutoGate`·`g_ipManualGate`, 비우면 코어 기본 | `FB_Gate` | `FB_ModuleBase` |
| `I_OutputRule` | 보조 DO를 켤지 판단 | `FB_AuxOutput` | `FB_ModuleBase` |

### 10_Core/02_Util — 작은 도구

| 파일 | 하는 일 |
|---|---|
| `F_CycleSec` | 지금 태스크 주기(초). 램프·필터 계산에 쓴다 |
| `F_RampStep` | 기준값에서 한 스캔 동안 갈 수 있는 만큼만 목표 쪽으로 옮긴다 |
| `F_IsStopped` | 장비 상태가 멈춘 상태인가 (Stop 또는 Fault). 통신 여부는 부르는 쪽이 따로 본다 |
| `F_ModeApplies` | 등록(`bAuto`·`bManual`)이 대상에 걸리는가 (그 운전 모드 유닛이 하나라도 있으면). 보호 대응·유닛 제한·버스 제한이 같이 씀 |
| `F_IsLocal` | PCS가 로컬(PMS 제어 안 받음)인가: 통신 있음 + 리모트 아님. 통신이 끊긴 건 로컬이 아니다 |
| `F_IsFinite` | 실수가 보통 숫자인가 (NaN·무한대 아님). 비트로만 본다. 바깥에서 온 실수는 크기 비교 전에 이것으로 거른다 (9장 "컴파일 함정") |
| `F_TimeHhmm` `F_HhmmOk` `F_InHhmmWindow` `F_MinutesTo` | 시각을 HHMM 정수로, HHMM 값이 맞는지(0000~2359, 끝 시각은 2400도), 지금이 [시작, 끝) 안인지(시작 > 끝이면 자정을 넘김, 같으면 빈 구간), 끝 시각까지 남은 분 (제어 모드 시간표·시간 구간) |
| `FB_Lpf` | 1차 지연 필터 (Q 응답 필터 등) |
| `FB_ReadTimer` | 읽기 주기 타이머. 읽을 때가 된 스캔만 TRUE |
| `FB_PersistSave` | PERSISTENT 값을 파일에 저장 (변경 후 1 s 동안 추가 변경이 없으면, 그리고 주기마다). PERSISTENT는 정상 종료 때만 저장되기 때문에 필요하다. 바뀐 설정을 저장할 때 무엇이 바뀌었는지(`q_bSetChanged`, `q_sSetChanged`)를 내고 `FB_EventLog`가 기록한다 |

### 10_Core/03_Base — 기반 FB

모듈을 만들 때 상속한다. 공통 기능을 여기 두어 모듈마다 다시 쓰지 않는다.

| 파일 | 상속하는 쪽 | 주는 것 |
|---|---|---|
| `FB_ModuleBase` | 모든 모듈 | 모듈 ID·이름(`i_nModuleId`, `i_sName`), 기본 이유 문구, 기본 설정 점검(점검할 것 없음) |
| `FB_DeviceBase` | 장비 드라이버 | 위 + 읽기 주기(`M_IsReadTime`), 통신 판정(`M_MarkAlive`, `M_UpdateComm`, `i_tCommTimeout`), 쓰기 실패 판정(`M_MarkWriteResult`: 잇달아 3번 실패하면 `_bWriteFail`, 한 번 성공하면 풀림) |
| `FB_SourceBase` | 지령원 | 위 + 출력 칸(`_stOut`), 구현할 메서드 `M_UpdateOutput`, 지령원 P·Q 제한의 이유 문구("지령원 제한"). 끊기면 지령·요청은 무효, P·Q 제한은 마지막 값 유지 |
| `FB_CtrlModeBase` | 제어 모드 (ABSTRACT) | `FB_ModuleBase` + `I_ControlMode`. 켜짐 표시(`_bActive`)를 `M_Enter`·`M_Exit`가 맞추고, 기본 진입 검사(늘 됨)를 준다. 파생 FB는 `M_Calc`만 구현하면 된다 |

### 10_Core/10_Snapshot — ① 읽기

| 파일 | 하는 일 |
|---|---|
| `FB_Snapshot` | 이번 스캔의 장비 상태를 한 번에 복사한다. 뒤 단계는 전부 이 사본만 본다 (스캔 도중 값이 바뀌어 단계마다 다른 값을 보는 일을 막음). 빈 칸(장비 없음)은 "통신 없음, Unknown"으로 채운다 |
| `FB_Measure` | 계측점(계측기 값 × 부호의 합)을 계산하고, 그룹마다 접속점 값을 만든다. `TEST_MODE`면 가상신호를 덮어쓴다 |
| `ST_VirtualSet` | 가상신호 설정 (시험용) |

### 10_Core/20_Protection — ② 보호·가용·설비 한도

"지금 무엇을 할 수 있는가"를 정하는 곳이다. 여기서는 요청과 한도만 만들고 장비에 직접 명령하지 않는다.

| 파일 | 하는 일 |
|---|---|
| `FB_Protection` | 보호 칸·대상마다(대상 번호 0 = 범위 안 전부) 조건 모듈 판단 → 같은 칸 대응 모듈(`I_ProtResponse`)이 요청(`ST_ProtReq`) → 유닛·BMS마다 합친다: PCS·배터리 조치는 센 쪽, P·Q 상한은 낮은 쪽, Manual 전환은 하나라도 (`ST_UnitProt.bModeManualReq`). 건 모듈은 조건 모듈 ID. 지연 없음 (조건이 TRUE면 바로) |
| `F_PcsRank` `F_BatRank` | 조치 세기 비교 (None < Standby < Stop < EStop). 여러 보호가 걸리면 센 쪽 |
| `E_ModeAction` `ST_UnitProt` `ST_BmsProt` | 보호의 모드 조치(None/ToManual), 유닛·BMS별 보호 결과 (조치·P 상한·Q 상한·모드 조치를 건 모듈 ID) |
| `FB_BatGroup` | BMS마다 쓸 수 있는지(설치·통신·고장·스스로 정지·보호) 판단하고, 배터리 그룹마다 쓸 수 있는 BMS의 한도·SOC·투입 완료를 모은다 (`ST_BmsAvail`, `ST_BatGroupState`) |
| `FB_UnitAvail` | 유닛마다 가용인지와 이유(통신·고장·로컬·배터리·보호)를 BOOL로 펼쳐 낸다 (`ST_UnitAvail`) |
| `FB_EquipmentLimits` | 설비 한도를 스캔당 한 번 계산한다. 유닛 한도(`ST_UnitEnvelope`), 버스 한도(`ST_BatEnvelope`), 그룹 설비 범위(`ST_GroupEnvelope`). 보호 P·Q 상한은 따로 적용 (피상전력 한도 = MIN(정격 S, √(P 상한² + Q 상한²)): 한쪽만 줄여도 다른 쪽을 묶지 않음). 성형·분배·그룹 요약이 모두 이 결과만 쓴다 |

### 10_Core/30_Request — ③ 요청 모으기

| 파일 | 하는 일 |
|---|---|
| `FB_GroupView` | 모듈에 넘길 그룹 요약(`ST_GroupView`)을 만든다: 정격 합, 가용 한도(설비 한도 결과를 옮김), 실측 출력 합, 가용 유닛 수, SOC, 배터리 용량(가용 유닛이 쓰는 버스의 SOC 유효 BMS 용량 합) |
| `FB_UnitView` | 유닛마다 같은 모양의 요약(`ST_UnitView`)을 만든다: 정격·가용 한도·출력은 그 유닛, 버스 충·방전 한도와 용량은 버스 가용 유닛 정격 비율로 나눈 몫, SOC는 연결 버스, 접속점·상태는 소속 그룹 그대로 + 그룹 출력 합과 그룹 가용 한도 중 이 유닛 몫(`fRatingShare`). 유닛 단위 제어 모드 관리자가 주소로 받아 본다 |
| `FB_SourceHub` | 등록된 지령원을 부르고 출력을 모은다. 고르지는 않는다 (고르는 것은 ④ `FB_Select`). 리셋 요청도 모은다 |
| `FB_CtrlModeMgr` | 제어 모드 관리자 (지령원, `FB_SourceBase` 상속). 등록한 제어 모드 중 하나만 켜고 전환(진입 검사 → 이전 모드 끝 → 출력 0 지연 → 다시 진입 검사 → 진입)을 맡는다. `i_nUnit` 0 = 그룹 지령, n = 유닛 n의 유닛별 값만 낸다. 모드는 `FB_ModuleList`에 없으므로 모드의 설정 점검(`M_CheckSet`)을 대신 부른다 (7장 "제어 모드") |

### 10_Core/40_Arbitration — ④ 판단

여러 요청 중 무엇을 따를지, 얼마를 낼지 정하는 곳이다. 파일이 가장 많다. 크게 네 갈래다.

**운전 모드**

| 파일 | 하는 일 |
|---|---|
| `FB_ModeSwitch` | 요청 모드(`GVL_PERSIST.g_aeOpMode`)를 받아 전환 단계(출력 0 → 지연 → 대기 확인: 장비가 대기·정지를 보고하거나 실측 |P|·|Q| ≤ 정격 × `g_stSeqSet.stPcs.fStopDonePct`)를 거친 뒤 실제 모드를 낸다. 코어는 실제 모드만 본다. ②의 맨 앞에서 부른다. 보호의 모드 조치(`ToManual`)가 걸려 있으면 요청 모드(`g_aeOpMode`)를 Manual로 바꿔 쓴다 (`q_abModeForced`) |
| `E_ModeSwitch` `E_ModeStep` | 전환 방식(Direct/Zero/Standby), 전환 단계 |

**상태 요청 (Run·Standby·Stop)**

| 파일 | 하는 일 |
|---|---|
| `FB_StateArbiter` | 본체: 유닛마다 이번 스캔 지령원 상태 요청 하나를 고른다 (같은 스캔이면 정지 쪽, 보호 조치보다 기동 쪽은 버림). `M_CommitRequest`: 접수된 것만 이번 스캔 요청(`q_abNewReq`, `q_aeReq`). 저장하는 목표는 없다 (받은 명령은 시퀀서가 한 번 보냄) |
| `F_StateRank` | 상태 요청 세기 (정지 쪽이 큼) |
| `F_PcsCmdAllowed` `F_BatCmdAllowed` `F_SourceAllowed` | 지령원 요청을 이 운전 모드에서 받는지 (등록값 기준) |
| `FB_PermitEval` | 기동 조건 모듈을 그룹마다 스캔당 한 번 평가 (모두 허가해야 기동). `fbSequencer.M_Prepare` 앞에 부르고, 결과는 시퀀서 접수 검사·기동 단계가 씀 |

**출력 지령 고르기·거르기**

| 파일 | 하는 일 |
|---|---|
| `FB_Select` | 지령원 출력 중에서 경로(Auto/Manual)마다 유닛 값과 그룹 값의 후보를 모아 고르기 정책에 넘긴다. 지령원 P·Q 제한은 전부 겹친다. 이상값(NaN, 정격 × 1.5 초과)은 후보에서 빼고, 정책이 낸 값이 이상값이어도 버린다 (`q_anPolicyBad`) |
| `FB_SelectStd` | 코어 기본 고르기 정책 (그룹 칸이 비었을 때): 유닛별 값은 등급, 그룹 값은 `eSelP`·`eSelQ`(등급 높은 것 / 절대값 작은 것). 같으면 먼저 등록된 칸 |
| `F_Better` `F_CmdOk` `ST_Pick` | 후보 비교(기본 정책이 씀), 지령값이 쓸 만한지, 고른 결과 |
| `FB_Gate` | 경로별 지령 거르기 자리 (`fbAutoGate`·`fbManualGate`가 같은 FB를 경로 `i_ePath`만 바꿔 씀): 등록한 Gate 모듈(`GVL_SITE.g_ipAutoGate`·`g_ipManualGate`)을 부르거나, 비우면 코어 기본을 부른다. 결과를 고정된 출력으로 내고, 다른 경로 유닛 값은 비운다 (Gate가 잘못 채워도 새지 않게) |
| `FB_GateAuto` | 코어 기본 Auto Gate: 그대로 통과 |
| `FB_GateManual` | 코어 기본 Manual Gate: 방향 반대 새 지령 무시, 새 지령 표시, Run 뒤 새 지령 대기 (`g_stManualCmdSet`) |
| `E_CmdRefuse` | 무시한 지령의 이유 |
| `FB_OutputReady` | 유닛마다 지금 출력을 낼 수 있는지 한 번 정한다: 가용 + 장비 활성(Run·대기) + 정지 진행 중 아님(`fbSequencer.q_abStopping`, 지난 스캔) + 모드 전환 아님 + Manual 새 지령 대기 아님. 결과 `q_abReady`, 이유 `q_aeNotReadyBy`(NotAvail / ModeSwitch / WaitManualCmd / NotRun). 성형(램프 기준)·분배(대상·이유)·디스패처(Manual 쓰기 시점)가 이 결과만 쓴다 |

**성형 (그룹이 "얼마")**

| 파일 | 하는 일 |
|---|---|
| `FB_LimitEval` | 제한·보정 모듈을 그룹마다 한 번 평가한다 (두 경로가 같은 결과를 써서 모듈 타이머가 두 번 돌지 않게) |
| `FB_ShapePath` | 성형, 경로 하나. MAIN에 `fbManualShape`(`i_bManual = TRUE`) 먼저, `fbAutoShape`가 Manual 결과를 받아 돈다. P: 램프 → 접속점 제한 → 보정 → 설비 제한. Q: 필터 → 제한. 설비 제한은 그룹 설비 범위(`iq_astGroupEnv`)를 쓴다 |
| `F_RangeOpen` `F_RangeAdd` `F_RangeAnd` `F_RangeClamp` `F_MakeLimit` `F_LimitKeepZero` `ST_Range` `ST_Clamp` | 범위 계산 도구: 제한 없는 범위, 제한 하나만큼 좁히기, 두 범위 겹치기, 값을 범위 안으로, 코어 제한 한 줄 만들기, 모듈 제한에 0 포함시키기. 상·하한마다 "누가 정했는지"를 같이 들고 다닌다 |
| `ST_ShapeDiag` | 성형 진단 (경로·그룹 하나) |

### 10_Core/50_Distribution — ⑤ 분배

| 파일 | 하는 일 |
|---|---|
| `FB_Distributor` | 그룹 값을 유닛에 나눈다 (`M_DistPath`, Manual 경로 먼저 → Auto). 유닛별 지령 → 그룹 몫 나누기(무게는 분배 정책 `I_DistPolicy`, 비우면 `eDistBasis`: 정격 비율 / SOC / 균등) → 한도에 걸린 몫 재분배 → 배터리 버스 확인 → 피상전력. 유닛·버스 한도는 `FB_EquipmentLimits` 결과를 쓴다. 제어용으로 `q_abUnitLimited`(유닛 지령이 줄었거나 0으로 묶임)를 낸다 |
| `FB_DistStd` | 코어 기본 분배 정책 (그룹 칸이 비었을 때): `eDistBasis` 정격 비율 / SOC(방전은 SOC 높은 쪽, 충전은 낮은 쪽, P만. 기본) / 균등 |
| `E_UnitLimitBy` | 유닛 값이 줄어든 이유 (NotAvail, Rating, BatLimit, SLimit, UnitRule 등) |
| `ST_UnitDiag` `ST_DistDiag` `ST_DistPathDiag` `ST_BatDistDiag` | 유닛별·그룹별·경로별·버스별 분배 진단 |

### 10_Core/60_Dispatch — ⑥ 쓰기

장비에 실제로 쓰는 곳이다. PCS·BMS는 `FB_Dispatcher`, 보조 DO는 `FB_AuxOutput`만 쓴다 (R2).

| 파일 | 하는 일 |
|---|---|
| `FB_Sequencer` | 유닛·BMS 시퀀서를 묶는다. `M_Prepare`: 이번 스캔 조건 모으기 (유닛 상태, 배터리 투입 완료 = `FB_BatGroup`, 기동 조건 = `FB_PermitEval`). `M_CheckRequest`: 상태 요청 접수 검사 (판단만). 본체: 유닛·BMS 명령 생성 |
| `FB_BatRequest` | 배터리 그룹마다 투입·개방 요청 정리: 지령원 BatClose·BatOpen, 정지 시 개방, 운전원이 연 배터리 유지, 거부 이유(UnitRunning). 버스 정지 확인 두 가지: `q_abBusStopped`(통신 있고 Stop·Fault, 정지 시 개방 기준), `q_abBusStopOk`(정지 또는 통신 끊김, 보호 개방·운전원 BatOpen 기준). 시퀀서 본체가 부름 |
| `FB_GroupState` | 유닛 시퀀스 상태로 그룹 상태를 요약 (그룹 보기에 다음 스캔부터) |
| `FB_UnitSeq` | 유닛 하나의 시퀀스: 받은 명령을 한 번 보내고 장비 상태로 확인. 정지 쪽은 지령원 Stop·EStop과 보호 정지 중 센 쪽, 기동은 멈춘 유닛에 Run 1회(단계별 또는 순서 기동). 보호 명령만 재시도, 실패는 알람만 |
| `FB_BmsSeq` | BMS 하나의 접촉기: 투입은 요청·기동이 있을 때만, 보호 개방·비상정지는 버스 PCS 정지 뒤. 보호 명령만 재시도, 실패는 알람만 (잠그지 않음) |
| `F_CmdReached` | 장비 상태가 명령대로 바뀌었는지 |
| `E_UnitSeq` `E_SeqRefuse` | 시퀀스 단계, 상태 요청 거부 이유 |
| `FB_Dispatcher` | PCS·BMS에 상태 명령(새 명령일 때 1회)과 P·Q(Auto는 쓰기 주기마다, Manual은 새 지령일 때)를 쓴다. 쓰기 허가가 없으면 아무것도 안 쓴다. 출력을 다시 0으로 막지 않는다 (출력 못 내는 유닛은 분배에서 이미 0). 마지막 안전장치로 NaN·무한대 지령은 0으로 바꿔 쓴다 (`q_abBadSet`, 로그 "지령값 이상") |
| `FB_AuxOutput` | 출력 규칙 → 켜짐·꺼짐 지연 → 유지(latch) → 같은 점은 OR → 보조 DO에 쓴다. 쓰기 허가가 없으면 켜는 쓰기만 막고 끄는 쓰기는 한다 (7장 "쓰기") |

### 10_Core/70_Diag — 진단·로그·점검

| 파일 | 하는 일 |
|---|---|
| `FB_ModuleList` | 등록 배열 15종과 Gate 2칸을 하나의 모듈 목록(`q_aipModule`)으로 만든다. 모듈 ID 점검과 로그의 이름·이유 문구 찾기는 이 목록만 본다. 같은 인스턴스를 여러 배열에 등록하면 한 번만 넣는다 (인스턴스 주소로 비교). 유닛·BMS 설치 여부(`q_abUnitInstalled`, `q_abBmsInstalled`: 칸이 비면 미설치)도 낸다 |
| `FB_ConfigCheck` | 매 스캔 고정 설정(`GVL_SITE`)을 점검한다: 구성 번호 범위, 모듈 ID 겹침·0, 같은 PCS·`nSub` 중복, 정격 0 이하(미설치 유닛 제외), 모듈 설정 묶음 크기, 되돌릴 정상값이 없는 운전 설정(`FB_SettingGuard.q_bNoGood`). 오류가 있으면 `q_bConfigOk = FALSE` → PCS·BMS 쓰기와 보조 DO 켜기를 막는다 (DO 끄기는 허용). 오류 문구(앞 10건, `q_asErrors`)만 내고 기록은 `FB_EventLog` |
| `FB_SettingGuard` | 운전 중 바꾸는 설정 값을 지킨다: 코어 설정(`g_stCmdFilter`·`g_stManualCmdSet`·`g_stSeqSet`), 등록·출력·규칙 설정의 번호 범위(`g_astOutputCfg`, `g_astRuleCfg.nTarget`, 등록의 `nGroup`·`nTarget`, 그룹이 2개 이상일 때 `nGroup` 0), 모듈 설정(각 모듈 `M_CheckSet`). 잘못된 값이면 받지 않고 마지막 정상값을 `GVL_PERSIST`에 다시 써 되돌린다 (등록·모듈 설정은 묶음 통째, 되돌린 스캔은 MAIN이 모듈에 다시 넘김). 시작부터 잘못이면 `q_bNoGood` → 설정 오류. 모듈 설정 묶음 크기 초과(`q_bModuleSetTooBig`)도 여기서 낸다 |
| `FB_OutputDiag` | 그룹마다 출력을 막은 단계·주체·이유(`ST_OutputDiag`)와 P 추적표(`ST_PTrace`, `ST_TraceStep`)를 낸다 |
| `E_DiagStage` `E_TraceStep` | 막은 단계(Ramp, PccLimit, DeviceLimit, UnitLimit 등), 추적표 줄 종류 |
| `FB_EventLog` | 기록하는 유일한 곳. 결과가 바뀔 때만 TwinCAT 이벤트 로거에 남긴다. 본문은 기록 구간별 액션(`A_LogSystem` `A_LogRedundancy` `A_LogProtection` `A_LogDeviceFail` `A_LogSource` `A_LogRefused` `A_LogOutput`)을 차례로 부른다 |
| `FB_LogAlarm` `FB_LogMsg` | 알람(발생·해소 한 쌍), 메시지(한 번) |
| `FB_AliveRecorder` | 마지막 동작 시각을 1 s마다 갱신, 시작할 때 "직전 동작 시각·꺼져 있던 시간"을 `FB_EventLog`에 알린다 (`q_bStartEvent`). 기록은 `FB_EventLog`가 한다 |
| `F_Txt…` (11개) | 로그·화면용 문구 만들기 (조치, 명령, 코어 이유, 축소 이유, 모드, 거부 이유, 시퀀스, 단계, "앞말 + 번호") |

### 10_Core/80_Report — 보고

| 파일 | 하는 일 |
|---|---|
| `FB_Report` | 코어 결과를 상위 인터페이스(EMS 토탈 인터페이스)로 내보내는 자리. 보고 전용이고 제어는 이 값을 읽지 않는다 (R6). 매핑 모듈화는 10장 남은 일 |

### 10_Core/90_Redundancy — 이중화

두 PLC에 같은 프로그램을 넣고, 이 폴더의 FB를 `PRG_Redundancy`(RedundancyTask)가 부른다. 동작 규칙은 7장 "이중화".

| 파일 | 하는 일 |
|---|---|
| `FB_RedLink` | I/O 연결: EAP 하트비트 보내기·받기(`_stTx`·`_stRx`·`_nEapQuality`), 접점(alive 펄스·Active 레벨) 보내기·받기, 상대 상태 판정 호출. I/O(`AT %I*`·`%Q*`)는 설치할 때 XAE에서 연결 |
| `FB_RedPeerMonitor` | 순수 로직: EAP 정상(카운터가 바뀜 + Quality 정상)과 접점 정상(alive 값이 바뀜)을 따로 보고 `E_RedPeer`로 합친다. 받은 역할이 내 역할과 같으면 되돌아온 내 프레임으로 무시, 같은 역할로 살아 있으면 `q_bRoleConflict` |
| `FB_RedState` | 순수 로직: 이중화 상태 기계(`E_RedState`). 역할 + 상대 상태 → Active 여부(`q_bActive`)·장비 쓰기 허가(`q_bWrite`). 절체·복귀·양보·멈췄다 이어짐(`i_bReset`)·쓰기 대기 |
| `FB_RedSync` | 운영 설정(브로커 감시 설정 포함)과 제어 모드 요청을 EAP로 매 스캔 보내고, 내가 Active가 아니고 상대가 Active일 때 받은 값으로 바뀐 필드만 덮어쓴다 |
| `ST_RedHeartbeat` | EAP 하트비트 구조체 (형식 번호, 카운터, 역할, Active, 상태) |
| `ST_RedSyncCfg` | EAP 동기화 구조체 (`GVL_PERSIST`의 운영 설정과 같은 모양 + 제어 모드 요청). 모듈 설정 묶음은 없다 (현장이 필요하면 7장 "현장 모듈 설정 동기화"로 더한다) |
| `E_RedPeer` `E_RedState` `E_RedSwitch` | 상대 상태(Unknown·Alive·EapLost·WireLost·Lost), 이중화 상태(Init·Standby·Active·Failed·Handover), Active가 바뀐 이유(Start·PeerLost·Failback·Yield) |

### 20_Module/Common — 공통 모듈

| 파일 | 구현 | 하는 일 |
|---|---|---|
| `FB_AuxIo` | `I_AuxDevice` | 벡호프 단자 I/O (EL1xxx DI, EL2xxx DO, EL3xxx AI, EL4xxx AO). 점은 TwinCAT I/O 화면에서 연결 |
| `FB_RuleBit` | `I_ProtectionRule`, `I_OutputRule` | 디지털 접점 하나 → 조건 (화재, 연기, 비상정지, 문 열림). 반전·통신 끊김 시 동작 선택 |
| `FB_RuleValue` | `I_ProtectionRule`, `I_OutputRule` | 아날로그 값 하나 ↔ 임계값 (온도, 습도). 히스테리시스. 보호용·출력용 상태를 따로 가짐 |
| `F_SocDerate` | 함수 | SOC 감발 비율 계산 (상태 없음) |
| `F_SocLimitSetErr` | 함수 | SOC 정책 설정 점검 ("" = 정상). `FB_SocLimit.M_CheckSet`과 `FB_EmsConfigApply`가 같이 씀 |
| `FB_SocLimit` | `I_BatLimitRule` | SOC 감발·막음·재개 마진 (버스 제한). 기본 등록 (`g_fbSocLimit`, 모든 버스, Auto·Manual) |
| `FB_BreakerCtrl` | `I_OutputRule`, `I_ProtectionRule` | 차단기(VCB) 운전원 투입·개방: HMI 지령 → 코일 펄스(DO) → 상태 접점 확인 → 투입·개방 실패 감시 (6장 "자주 하는 작업") |
| `ST_BreakerSet` | 설정 구조체 | `FB_BreakerCtrl` 펄스 폭·확인 시간 |
| `FB_UnitLimit` | `I_UnitLimitRule` | 사용자 한도: 유닛마다 운전원이 정한 P(kW)·Q(kvar) 상·하한. 정격·Derate와 겹쳐 좁은 쪽. 출력 제한 이유 UnitRule + 모듈 ID, 문구 "사용자 한도". 기본 등록 (`g_fbUnitLimit` 230, 설정 `g_stModuleSet.stUnitLimit` 기본 모두 끔, 등록 `g_astUnitRuleReg[1]`). EMS config pcs 충·방전 한도도 여기에 씀 (2026-11-04) |
| `ST_UnitLimitItem` `ST_UnitLimitSet` | 설정 구조체 | `FB_UnitLimit` 유닛별 한도 (`bPEnable`, `fPMax`, `fPMin`, `bQEnable`, `fQMax`, `fQMin`, 기본 모두 끔). 켠 한도는 상한 ≥ 0 ≥ 하한이어야 함 |
| `ST_SocLimitSet` `ST_RuleBitSet` `ST_RuleValueSet` | 설정 구조체 | 위 모듈의 설정 (운전 중 바꿈). 모듈은 `i_stSet`으로 받고, 값은 `GVL_PERSIST.g_stModuleSet` 칸에 둔다 |
| `FB_LocalClock` | FB | PLC 현지 시각(`ST_LocalTime`: 연·월·일·요일(0 = 일)·시·분·초, `bValid`). `FB_LocalSystemTime`(Tc2_Utilities)으로 Windows 시간대 기준. MAIN `fbClock`이 매 스캔 부르고 `A_PassModuleSet`이 제어 모드 관리자에 넘긴다 |

### 20_Module/ControlMode — 제어 모드

제어 모드 관리자(`FB_CtrlModeMgr`)에 등록하는 모드들이다. 한 관리자 안에서 한 번에 하나만 켜진다. 스케줄·피크컷·통합은 Messis(`Messis.Pms_v1(TC3)`)의 계산을 옮겼고, 같은 계산은 같은 메서드·FB를 쓴다. 동작 규칙은 7장 "제어 모드".

| 파일 | 하는 일 |
|---|---|
| `FB_CmConstPower` | 정출력: 정한 P(필요하면 Q)를 그대로 낸다. `tDuration`이 지나면 스스로 끝낸다 (0 = 끝없이). 새 모드를 만들 때 본보기 |
| `FB_CmStdSchedule` | 스탠다드 스케줄: 시간표를 PMS에 두지 않고 EMS가 시각마다 보내는 P 지령을 그대로 따른다 (7장 "제어 모드"). 입력 `i_stCmd : ST_CmExtCmd` |
| `ST_CmExtCmd` | 바깥(EMS 등) 지령 한 대 몫: `bAlive`, `nPSeq`·`fP`(P 지령 번호·값), `nStateSeq`·`eState`(마지막 상태 명령). MAIN `A_PassEmsCmd`가 EMS 명령 칸에서 채움 (모드는 통신 형식을 모름) |
| `FB_CmTariffBase` | 요금제 기반 모드의 공통 부모 (ABSTRACT): 시각 확인, 휴일, 계절, 시간표 칸 찾기, 방전 허용 시간, 역송 방지, 잔량 방전·자동 충전 계산, 설정 점검, 이유 문구. 상태·이유 코드는 `GVL_CM_RSN` |
| `FB_CmSchedule` | 스케줄: 시간표 [계절, 요일, 칸]에서 지금 칸의 값. 충전(−)은 그대로, 방전(+)은 방전 허용 시간·역송 방지. 맞는 칸 없음·휴일 → 0 |
| `FB_CmPeakCalc` | 15분 피크 제어량 계산 (피크컷·통합이 같이 씀). 구간은 시계 기준 15분(00·15·30·45분), 구간 경계에서 0부터 다시. 상한은 가용 출력, 30초 평균 부하 × `fLoadRatio`, 30초 평균 부하 − 역송 여유 |
| `FB_CmPeakCut` | 피크컷: 자동 충전 시간 → 잔량 방전 시간(잔량 방전과 피크 제어 중 큰 값) → 피크 제어(방전 허용 시간에만) 순서 |
| `FB_CmSchedPeak` | 통합(스케줄 + 피크컷): 잔량 방전 시간 → 피크 제어가 필요하면 피크 제어량과 시간표 방전 중 큰 값 → 아니면 스케줄과 같음. 시간표는 스케줄과 같은 표 |
| `F_CmVelocity` `F_CmRoundTens` | 피크 제어 속도(남은 분에 따른 배율), 10 단위 올림 |
| `GVL_CM_RSN` | 요금제 모드 상태·이유 코드 (`NO_TIME` 1 … `NEW_INTERVAL` 16, 12장) |
| `ST_CmCommonSet` `ST_CmDisWindow` `ST_CmNoReverse` | 공통 설정: 월 → 계절, 휴일 [월, 일], 방전 허용 시간(기본 08~22시, 일요일 금지), 역송 방지(여유 10 kW, 비율 0.95) |
| `ST_CmScheduleSet` `ST_SchedSlot` | 시간표 [계절 0..2, 요일 0..6, 칸 1..`SCHED_SLOT_COUNT`], 칸 = 시작·끝(HHMM)·P(kW, 방전 +) |
| `ST_CmPeakCalcSet` `ST_CmPeakCutSet` `ST_CmSchedPeakSet` `ST_CmWindowSet` `ST_CmConstPowerSet` | 피크 목표·감도·부하 비율·최소 출력, 피크컷(자동 충전·잔량 방전 시간), 통합(잔량 방전 시간), 시간 구간 + SOC, 정출력 |

### 20_Module/Standard — 표준 보호 모듈

기본으로 등록하는 보호 조건 5종과 표준 대응이다. 조건 칸에서 빼면 그 보호는 없다. 현장마다 판단·대응이 다르면 복사해 이름을 바꾸고 그 칸에 대신 등록한다 (원본은 고치지 않음). 적용 모드·표준 대응 조치는 `GVL_PERSIST.g_astRuleCfg` 같은 칸 번호에서 정한다.

- `FB_ProtRespStd` (900, `I_ProtResponse`): 표준 대응. `g_aipResp` 1~5칸에 같은 인스턴스(`g_fbProtRespStd`)를 등록 (상태가 없어 하나로 충분). 칸 설정(`stProt`)의 조치를 걸린 동안 한 단계로 요청한다: 범위 안 유닛 중 그 운전 모드에 거는 것(`bAuto`·`bManual`)에 PCS 조치·P·Q 상한, `bAuto` 항목의 `ToManual`은 범위 안 유닛 전부, Bms·BatGroup 범위 배터리 조치는 그 버스에 조치를 받는 유닛이 있을 때 (유닛이 없으면 그대로), Unit·Group·Site 범위는 조치를 받는 유닛의 배터리 그룹 BMS 전부. 대응 칸을 비우면 그 보호는 알람만.

| 파일 | 모듈 ID | 범위 | 조건 | 기본 조치 (`g_astRuleCfg`) |
|---|---|---|---|---|
| `FB_ProtPcsFault` | 901 | Unit | 통신이 있고 PCS 고장 | 1칸: 그 유닛 Stop |
| `FB_ProtPcsComm` | 902 | Unit | PCS 통신 끊김 | 2칸: 그 유닛 Standby |
| `FB_ProtBmsFault` | 903 | Bms | 통신이 있고 BMS 고장 | 3칸: BMS Stop + 같은 버스 유닛 Stop |
| `FB_ProtBmsComm` | 904 | Bms | BMS 통신 끊김 | 4칸: 같은 버스 유닛 Standby |
| `FB_ProtPccInvalid` | 905 | Group | 접속점 유효전력 값 무효 | 5칸: 그룹 Standby |

- 미설치 장비(칸 비어 있음)는 판단하지 않는다 (`i_stView.bInstalled`). 범위를 잘못 등록하면 보호가 걸리지 않는다.
- 보호를 빼도 통신이 끊긴 PCS·고장 PCS는 출력에서 빠진다 (가용 판단: 사실). 보호가 정하는 것은 장비 명령·모드·알람이다.

### 20_Module/Driver — 장비 드라이버

제조사마다 하위 폴더를 둔다. 드라이버는 통신(Modbus 등)과 "제조사 값 ↔ 표준 상태" 변환만 한다.

| 파일 | 하는 일 |
|---|---|
| `Plaspo/FB_PlaspoPcs` | 플라스포 PCS (Modbus TCP, 유닛 1개) |
| `Plaspo/ST_PlaspoPcsRaw` | 플라스포 PCS 읽기 레지스터 맵 |
| `Autonics/FB_AutonicsThd` | 오토닉스 THD 온습도계 (Modbus RTU, 국번 1..n), 보조 장비. AI 2n-1 온도·2n 습도, DI n 센서 통신 이상 |
| `Bender/FB_BenderIso685` | Bender ISO685 절연감시장치 (Modbus TCP), 보조 장비. AI 1 절연저항, DI 1 경고·2 고장·3 시험 중 |
| `Rootech/FB_RootechAcuraIo` | 루텍 ACURA3700 원격 I/O (Modbus TCP), 보조 장비. DI 1..8 접점(화재·연기·가스 감지기), AI 1..6 |
| `../Common/F_WordsToReal` | 레지스터 2개(앞 = 상위 워드) → REAL. NaN·무한은 0 |

위 세 보조 장비 드라이버는 Helios 드라이버의 동작을 옮긴 것이다 (맵 미확보, 2026-11-09). 읽기 전용이라 `M_SetBit`·`M_SetValue`는 아무것도 하지 않는다. 통신 이상이면 DI는 FALSE, AI는 `bValid` FALSE (규칙은 `Comm`으로 따로 판단). 시리얼(RTU) 드라이버는 라이브러리 `Tc2_ModbusRTU`의 `ModbusRtuMaster_KL6x22B`를 쓴다. 시리얼 단자는 EL6022(2채널, 회사 표준, 필요하면 3대까지 늘림)이고 장치 하나에 채널 하나를 쓴다 (드라이버마다 통신 블록이 따로라 한 채널을 나눠 쓰지 않음). 채널 연결은 XAE I/O 화면에서 한다.

### 20_Module/Mqtt — MQTT 보고·명령 (PMS ↔ EMS)

상위 플랫폼(EMS)과 MQTT로 주고받는다. 브로커 3개(Cloud·Local·Azure)에 발행·구독한다. 장치 발행 토픽 표는 `pms` `pcs` `bat` `rack` `pv_meter` `ess_meter` `temperature_sensor` `airconditioner` 및 `ups` `module` 10칸(사용하지 않는 칸은 발행 수0으로 제외)(`dgspms/{solution}/{사이트}/{장치}`), 구독은 command `pcs` `bat` `ups`과 config `pcs` `bat` `protection` `ups`(`dgsems/{solution}/{사이트}/{command|config}/{장치}`)이다. `{solution}`은 `GVL_MQTT.SOLUTION_NAME`(지금 `ups`). config는 pcs(충·방전 전력 한도)·bat(SOC 최소·최대)를 운영 설정에 반영하고, 실제 반영된 설정을 되돌려 발행(echo)한다 (아래 "config가 설정까지 가는 길", 2026-11-04 9-21). UPS 현장의 config/bat 레코드는 `SocMin`·`SocMax` REAL 형식(103.Config BAT UPS_V1, is_available 없음)이라 `FB_EmsConfigApply`가 받은 BMS 값만 계산에 넣는다 (2026-11-10 STD 반영). protection(배터리·배터리실 온도·습도 경고·고장)과 pcs `PvMarginRate`는 반영하지 않는다 (값의 뜻·센서 대기). UPS config는 태스크 간 수신함(`GVL_IF.g_stUpsConfigRx`·`g_nUpsConfigSeq`)까지 전달하며, 받은 원본 사본을 실제 적용 설정으로 간주하지 않는다. 이벤트 로그는 로거 토픽(`dgspms/{solution}/{사이트}/{LOGGER_TOPIC}`, 지금 `pms_log`)으로 생길 때마다 1건씩 보낸다 (아래 "로그가 EMS까지 가는 길"). 라이브러리 `Tc3_IotBase`가 필요하다 (로거는 `Tc3_EventLogger`도 씀).

| 파일 | 하는 일 |
|---|---|
| `FB_MqttMain` | MQTT 로직 전체. 입력 `i_stSet`(접속 설정). `PRG_Mqtt`가 부름. 로거 리스너(`fbLogger`)를 돌리고 브로커마다 로거 이벤트를 보냄(`A_PublishLogger`) |
| `ST_MqttSet` · `ST_MqttBroker` | 접속 설정 구조체 (사이트 번호, 브로커 3곳, 계정) |
| `FB_MqttPublish` · `FB_MqttSubscribe` | 브로커 1개 발행(1 s마다) · 구독. 토픽 표(`ST_MqttPubEntry`·`ST_MqttSubEntry`)대로 동작. 접속 값은 입력으로 받음. 로거 이벤트는 `FB_MqttPublish.M_PublishLogger`로 1건씩 (QoS 1) |
| `F_MqttTimeText` | 시각 → `'YYYY-MM-DD hh:mm:ss.xxx'` (헤더 TimeStamp 형식). 장치 토픽 헤더와 로거가 같이 씀 |
| `Logger/FB_MqttLoggerListener` | 로거 리스너 (`FB_ListenerBase2` 상속, MqttTask). 이벤트 로거에 남은 PMS 이벤트(클래스 4개)를 받아 시각·문구를 큐(`LOGGER_QUEUE` 64칸)에 쌓음. `M_Cycle` 매 스캔, `M_GetPacket` 브로커별 다음 패킷 |
| `Logger/ST_LoggerHeader` · `ST_LoggerPublish_V1` | 로거 토픽 패킷 (헤더 36 byte + `Contents : WSTRING(100)` 202 byte = 238 byte) |
| `Logger/ST_LoggerSlot` · `F_Utf8ToWstring` | 큐 한 칸, 이벤트 문구(UTF-8) → WSTRING 변환 |
| `F_MqttDistribute` | 받은 메시지(헤더 + 레코드 n개)를 토픽 표대로 수신 배열 칸에 복사 (DeviceIndex 0부터) |
| `FB_EmsCommand` | **명령 핸들러** (MqttTask). 받은 command 레코드를 한 번 해석하고 `CommandType`을 0으로 지움 → `ST_IfEmsCmd`(Std 번호 1부터, 명령 번호 `nSeq`·`anPcsPSeq`). 이상값(NaN·무한대) P 지령은 버리고 셈 |
| `ST_IfEmsCmd` | MQTT → MAIN 명령 칸 (`GVL_IF.g_stEmsCmd`) |
| `FB_EmsConfigApply` | **config 반영기** (PlcTask, MAIN `fbEmsCfg`). 받은 config 레코드 → 운영 설정 (`stUnitLimit`·`stSocLimit`), 실제 설정 → echo 칸 (`M_BuildEcho`). 아래 "config가 설정까지 가는 길" |
| `ST_IfEmsCfg` · `ST_IfEmsCfgEcho` | MQTT → MAIN config 칸 (`GVL_IF.g_stEmsCfg`, 칸마다 번호 `anPcsSeq`·`anBatSeq`) · MAIN → MQTT echo 칸 (`GVL_IF.g_stEmsCfgEcho`) |
| `FB_SrcEms` | **EMS 지령원** (MAIN, `GVL_SITE.g_fbSrcEms`, 모듈 ID 104). 명령 칸을 요청으로 바꿈 (아래) |
| `Publish/` · `Config/` | 발행 패킷(`ST_*Publish_V1`·`ST_*PublishData`)과 config 패킷 정의 |
| `PRG_PublishDeviceNo` | 발행 패킷 장치 번호(DeviceIndex·DeviceID), 랙의 배터리 소속(BatID)·모듈 수(ModuleCount) 채움. 시험 측정값과 분리 |

접속 설정(사이트 번호·브로커 3곳의 사용 여부·주소·포트·계정)은 `GVL_SITE.MQTT_SET`(`ST_MqttSet`)에만 있고, 로직은 `FB_MqttMain`(토픽 표, 시각·헤더, 브로커 3개 발행·구독, 수신 합산)이 한다. 토픽 수·버전·발행/수신 버퍼는 `90_Project/GVL_MQTT`에 있다. 접속 값은 첫 호출 때 한 번 읽으므로 바꾸면 재빌드한다. 브로커 계정은 저장소에 올리지 않으려고 비워 뒀다. 시험 빌드 전에 `MQTT_SET.sUserName`·`sUserPassword`에 적고 커밋하지 않는다.

**명령이 장비까지 가는 길**

```
EMS ─MQTT→ FB_MqttSubscribe → g_astPcsCommand / g_astBatCommand (MqttTask)
        → FB_EmsCommand 해석 → GVL_IF.g_stEmsCmd (번호 g_nCmdSeq: 홀수 쓰는 중, 짝수 끝)
        → MAIN.A_PassEmsCmd 가 일관되게 복사 → FB_SrcEms → 다른 지령원과 같은 중재·성형·분배
```

`FB_SrcEms` 규칙:

| 항목 | 내용 |
|---|---|
| 상태 명령 | PCS Run·Standby·Stop·EStop·Reset. 그 PCS의 유닛마다 요청(`nUnit`). 리셋은 전체 한 번. 스캔마다 1개씩 큐(16칸)에서 냄. 받는 운전 모드는 등록 `stStateMode` (기본: Run·Standby·배터리 투입/개방은 Auto, Stop·EStop은 Auto·Manual) |
| 배터리 | 투입(Close)·개방(Open)은 그 BMS의 배터리 그룹 요청. 리셋은 운전자 리셋(PCS Reset). **비상정지는 받지 않음** (요청 종류에 없음, `q_nUnsupported`로 셈) |
| 충전·방전 P | PCS 한 대의 kW(방전 +, 충전 −)를 그 PCS의 유닛에 정격 비율로 나눠 유닛 지령으로 냄. **`FB_SrcEms.i_bPEnable`이 TRUE일 때만** (기본 FALSE. 상태 명령은 무관). 제어 모드와 같이 쓸 때는 끈 채로 두고, 관리자에서 스탠다드 스케줄 모드(5)를 켜면 EMS P 지령이 그 모드를 거쳐 나간다 (`A_PassEmsCmd`가 모드에 넘김). 둘을 같이 켜지 않는다 |
| 만료 | 새 메시지나 새 명령이 `tExpire`(기본 60 s) 안에 오지 않으면 끊김: P 지령과 큐를 버림 (돌아와도 옛 지령이 되살아나지 않음) |
| 시작 | PMS가 켜진 뒤 처음 복사본(번호 0, 아직 받은 것 없음)으로 번호만 맞춤. 그 뒤 처음 받은 명령부터 실행 |
| retain | **EMS는 command 토픽을 retain=false로 발행한다** (운영 규칙, 2026-11-04). 브로커에 retained 명령이 남아 있으면 PLC가 재시작·재접속해 다시 구독할 때 브로커가 그 명령을 새 메시지로 보내고, PLC는 실행한다 (`Tc3_IotBase` 4024는 받은 메시지의 retain 표시를 주지 않아 PLC가 가릴 수 없음). 구독은 QoS 0이라 꺼져 있던 동안의 명령이 쌓였다 오지는 않는다. 같은 메시지를 두 번 받으면(EMS QoS 1 재전송·여러 브로커에 같은 명령·EMS 재전송) 두 번 실행한다 (9-11, 그대로 둠) |
| 그룹 | 등록 `nGroup = 0`이면 그룹 1 기준. 제어 그룹이 여럿이면 그룹마다 인스턴스 |

**config가 설정까지 가는 길** (2026-11-04 9-21)

```
EMS ─MQTT→ FB_MqttSubscribe → F_MqttDistribute (받기 전 버퍼 0) → g_stPcsConfigWireIn / g_stBatConfigWireIn (MqttTask)
        → PRG_Mqtt 5절: 새 메시지의 레코드를 칸(DeviceId + 1)에 넣고 칸 번호 +1 → GVL_IF.g_stEmsCfg (번호 g_nCfgSeq: 홀수 쓰는 중, 짝수 끝)
        → MAIN.A_PassEmsCfg (A_PassModuleSet 앞) 일관 복사 → FB_EmsConfigApply → GVL_PERSIST.g_stModuleSet.stUnitLimit / stSocLimit
        → 같은 스캔에 모듈 설정 넘기기·설정 보호(FB_SettingGuard)·저장(FB_PersistSave)
        → MAIN 본문(설정 보호 뒤) fbEmsCfg.M_BuildEcho → GVL_IF.g_stEmsCfgEcho → PRG_Mqtt 1-3절 → GVL_MQTT.g_ast…ConfigApplied → config echo 발행
```

| 항목 | 내용 |
|---|---|
| pcs 충·방전 전력 한도 (kW, UDINT) | 사용자 한도 `FB_UnitLimit`(230, 기본 등록)의 `stUnitLimit.astUnit[u]`에 씀. 그 PCS의 유닛들에 정격 비율로 나눔. 충전 한도는 `fPMin`(−), 방전 한도는 `fPMax`. 유닛 P 한도가 꺼져 있었으면 켜면서 안 받은 방향은 정격(= 제한 없음). Q 한도는 건드리지 않음 |
| bat SOC 최소·최대 (%) | SOC 정책 `stSocLimit.fSocMin`·`fSocMax`. 버스 전체에 한 세트라 받아 둔 BMS 값 중 가장 엄격한 값 (하한 = 가장 큰 값, 상한 = 가장 작은 값) |
| `is_available` 0 | **기본값으로 반영** (2026-11-09, 14번 탭 21번): pcs 충전·방전 한도 → 그 방향 정격(제한 없음), 둘 다 0 → 그 PCS 유닛의 사용자 P 한도 끔(`bPEnable` FALSE). bat SOC 하한 → 0 %, 상한 → 100 % (감발 폭·재개 마진은 HMI 값). 그래서 EMS가 한도를 풀 수 있다. 모든 값이 0인 레코드도 반영하고, 빈 칸은 실제로 받은 레코드 수(`GVL_MQTT.g_anCfgRxRecords`, `F_MqttDistribute.q_nReceived`)로 거른다. (예전: 기존 값 유지, 모두 0인 레코드는 건너뜀) |
| 범위 검사 | 쓰기 전에 먼저: 한도 1E6 kW 초과, 유닛 정격 합 0, SOC NaN·0~100 밖, 합친 SOC 설정이 `F_SocLimitSetErr`(= `FB_SocLimit.M_CheckSet`과 같은 규칙)에 걸리면 그 값을 버림 (SOC는 이번에 받은 BMS 값도 버림). 로그 "잘못된 설정 값을 받지 않음 (마지막 정상값 유지): EMS config …". 설정 보호가 한 번 더 지킴 |
| 기록 | 바꾸면 "운영 설정 바뀜: EMS config: PCS 충·방전 전력 한도 / SOC 상·하한" (+ 저장 때 "모듈 설정") |
| 영속 | GVL_PERSIST라 재시작 뒤에도 남음. EMS가 다시 보내지 않아도 마지막 값으로 운전 |
| echo | 받은 값이 아니라 **실제 설정**: pcs 충·방전 한도 = 그 PCS 유닛 사용자 한도 합 (모두 켜져 있으면 `is_available` 1, 아니면 0), `PvMarginRate` = 받은 값·`is_available` 0 (쓰지 않음), bat = SOC 정책 상·하한 (BMS마다 같은 값). MAIN이 아직 안 채웠으면 받은 값. protection echo는 받은 값 그대로 |
| 반영 안 함 | protection 6개, `PvMarginRate`. 붙일 곳: 배터리실 온도·습도 → `FB_RuleValue` 보호 칸(경고 = 조치 없음, 고장 = 정지·대기, 임계값 `g_astRuleCfg[n].fThreshold`, 보조 IO 센서 필요) |
| 확인할 곳 | `MAIN.fbEmsCfg`: `q_nApplied`, `q_nRejected`, `q_sRejected`. 코어 반영은 `MAIN.fbEquipLimits.q_astUnitEnv[u].stP` (모듈 230) |

**브로커 끊김 감시** (2026-11-09, 통합시험 14번 탭 9번)

| 항목 | 내용 |
|---|---|
| 하는 곳 | `FB_BrokerWatch` (PlcTask, MAIN `fbBrokerWatch`, `fbModeSwitch` 앞). 접속 상태는 `PRG_Mqtt`가 `GVL_IF.g_abBrokerUse`·`g_abBrokerConn`(1 Cloud, 2 Local, 3 Azure, 구독 클라이언트 기준)에 올림 |
| 설정 | `GVL_PERSIST.g_stBrokerWatch` (`ST_BrokerWatchSet`): `eBroker` 감시할 브로커 (기본 `Local`), `tLostDelay` 판정 시간 (기본 30 s). 운전 중 바꿈, 저장(`FB_PersistSave` "브로커 감시"), 이중화 동기화 |
| 끊김 | 고른 브로커가 `MQTT_SET`에서 쓰는 브로커이고 접속이 `tLostDelay` 넘게 이어서 끊기면 알람 `PmsOutput.BrokerLost` "MQTT 브로커 끊김: {브로커} ({시간} 넘게 접속 안 됨) - Auto 유닛을 Manual로 바꿈". 다시 접속하면 해소. 쓰지 않는 브로커를 고르면 감시하지 않음. PMS 시작 뒤 접속 전도 끊김으로 셈 |
| Manual 전환 | 끊김이 확정되면 한 번만 운전 모드 요청(`g_aeOpMode`)을 모든 유닛 Manual로 (`FB_ModeSwitch` 전환 단계를 그대로 거침). 끊긴 동안 운전원이 Auto로 되돌려도 다시 바꾸지 않음. 다시 접속하면 다시 무장 → 다음 끊김에 다시 한 번. 이중화면 Active PLC만 바꿈 (대기 PLC는 동기화로 맞춰짐) |
| 지령원 끊김 | EMS 지령원 등록 `bLostAlarm` FALSE: EMS command가 `tExpire`(60 s) 동안 없어도 "지령원 끊김" 알람은 내지 않음 (끊김 처리 — P 지령·큐 버림 — 는 그대로). 다른 지령원은 `bLostAlarm` TRUE (기본) |

**로그가 EMS까지 가는 길**

```
FB_EventLog → TwinCAT 이벤트 로거 ─(리스너 콜백)→ FB_MqttLoggerListener (MqttTask)
        → 문구 요청(비동기, UTF-8 → WSTRING) · 시각(UTC → 현지) → 큐 64칸
        → FB_MqttMain.A_PublishLogger → 브로커마다 FB_MqttPublish.M_PublishLogger → EMS
```

| 항목 | 내용 |
|---|---|
| 토픽 | `dgspms/{solution}/{사이트}/{LOGGER_TOPIC}` (`GVL_MQTT.LOGGER_TOPIC`, 지금 `pms_log`). 메시지 1건 = 이벤트 1건. 주기 발행 없이 생길 때만, QoS 1 |
| 패킷 | `ST_LoggerPublish_V1`: HeaderVersion·DeviceVersion(`LOGGER_VERSION`)·SiteId·TimeStamp(24 byte) + Contents(`WSTRING(100)`, UTF-16LE, null 포함 202 byte) = 238 byte. 보낼 때마다 0으로 비우고 채움 문자열은 길이 + null만 복사한다 (`M_GetPacket`): 문자열 대입 `:=`는 선언 크기 전체를 복사해서 큐 칸을 돌려 쓰는 동안 null 뒤에 옛 문구가 남는다 (PC 도구 재pack 비교 "불일치"로 보임) |
| TimeStamp | 이벤트가 생긴 시각(로거 기록 시각)을 현지 시각으로. 보낸 시각이 아니다. 현지 − UTC는 `FB_MqttMain.A_MakeTimeStamp`가 15분 단위로 구함 |
| Contents | 이벤트 로거 문구 그대로 (인자 채움). 알람 해소는 앞에 `[해소] `. 100자를 넘으면 99자 + `…` (`fbLogger.q_nTruncated`). 문구 길이 규칙은 7장 "로그 문구 길이" |
| 요건 | **제어기 TwinCAT 런타임 3.1.4024.17 이상** (`FB_ListenerBase2`, `Tc3_EventLogger` 3.1.27 이상). 빌드 PC의 라이브러리 버전이 아니라 제어기 런타임 버전이 기준이다. 낮으면 `fbLogger._hrSubscribe = 16#98110701`(서비스 미지원), `q_bSubscribed` FALSE, `q_nReceived` 0 → 런타임을 올린다 |
| 받는 이벤트 | `PmsSystem`·`PmsProt`·`PmsOutput`·`PmsSeq` 클래스의 메시지·알람 발생·알람 해소. TwinCAT 시스템 이벤트는 안 받음. 구독은 클래스 단위(`TC_EVENT_CLASSES`)라 클래스 안 이벤트를 추가·삭제·이름 변경해도 리스너는 고치지 않는다. **새 이벤트 클래스를 만들면** `FB_MqttLoggerListener.M_Cycle` 필터에 한 줄 추가 |
| 큐 | `GVL_MQTT.LOGGER_QUEUE`(64)칸. 한 스캔에 생길 수 있는 로그 수보다 커야 한다 (`PRG_TestLogger.bSendAll`은 스캔마다 32건씩). 브로커마다 다음 번호(`FB_MqttMain.anLoggerNext`)를 따로 가져 접속이 끊긴 동안 쌓인 것을 이어서 보냄. 기동 직후 접속 지연(10 s) 동안의 로그도 큐에 남음. 64건보다 밀리면 건너뛰고 셈(`q_nLost`) |
| 이중화 | 발행은 Active PLC만 한다 (`i_bPublish = GVL_IF.g_bRedActive`, 장치 토픽과 같음). 대기 PLC의 로그는 큐에 쌓지 않고 버린다 (`A_PublishLogger`): Active가 된 뒤 옛 로그를 한꺼번에 보내거나 두 PLC가 같은 로그를 이중으로 보내지 않게. 그래서 대기 PLC에서만 생긴 로그(예: Active를 내려놓은 뒤의 로그)는 EMS로 가지 않는다. 리스너와 시험 프로그램은 두 PLC 모두에서 돈다 |
| 문구 실패 | 5 s 안에 문구를 못 받으면 "로그 문구를 가져오지 못함: 이벤트 ID n" (`q_nTextError`) |
| 확인할 곳 | `PRG_Mqtt.fbMqttMain.fbLogger`: `q_bSubscribed`, `q_nReceived`, `q_nTruncated`, `q_nTextError`, `q_nLost`. 기록하는 곳은 그대로 `FB_EventLog` 하나 (이 리스너는 옮기기만) |
| 시험 | `99_Test/PRG_TestLogger` (`TEST_MODE`일 때 `PRG_Mqtt`가 부름). **브로커 대신 큐에서 패킷을 꺼내** 확인하므로 브로커 접속 없이 된다. **등록된 이벤트 자동 찾기**: 시작할 때 클래스 4개(`TC_EVENT_CLASSES`) × 번호 1~64를 이벤트 로거에 문구 요청해, 문구가 오면 등록된 이벤트로 담는다 (약 5 s, `bScanDone`·`nFound`, 메인 기준 34종). 인자 자리에 `#0#`~`#5#`를 넣어 요청하므로 이벤트마다 인자 수·고정 글자·남는 글자(95 − 고정)·인자 하나당 남는 글자가 `astReg[n]`에 나오고 (TwinCAT은 문구 자리보다 많이 넘긴 인자를 문구 끝에 ` (값)`으로 덧붙이므로 끝의 ` (#5#)`, ` (#4#)` …는 떼어 내고 센다. 그래서 문구 틀이 맨 끝에서 ` ({n})`으로 끝나면 안 된다), 고정 50자 초과 또는 인자 하나당 10자 미만이면 확인 필요(`bCheck`, `nCheck`). **이벤트를 추가·삭제·변경해도 시험 프로그램은 고치지 않는다** (새 클래스만 `A_InitScan`에 한 줄). **보내기**: 인자 자리에 문구 틀에 적힌 그대로 `{0}`, `{1}` …를 넣어 (Error List·EMS에 문구 틀 모양 그대로 보임) `bSendAll` 전부(스캔마다 32건) → `bAllDone` 뒤 `bAllOk`(다 받음·잘린 수 = 예상 `nExpectTrunc`·문구 오류 0), `bSendOne` `nSendNo`번 한 건, `bSendNext` 보내고 다음 번호로. 문구가 실제 로그와 같게 Error List·EMS에 찍히고, 심각도는 모두 Info(원래 심각도는 PLC에서 못 읽음). **로거 경로 시험**(시험 이벤트 `PmsSystem.LoggerTest`): `bAlarm` 켜면 알람 발생·끄면 `[해소] `, `bSendLong` 127자 → 99자 + `…`·`q_nTruncated` 1 증가. 받은 로그는 `awsAllRx`·`asAllRxTime`. 브로커 쪽 확인은 PC 도구 `PMS_Log_Tester`(수신 → unpack → pack → 원본 비교) |

### 20_Module/Report — 보고 매핑 (장치 값 → MQTT 발행 직전 구조체)

| 파일 | 하는 일 |
|---|---|
| `FB_RptPlaspoPcs` | 플라스포 PCS의 읽기 맵(`ST_PlaspoPcsRaw`, 드라이버 속성 `RawMap`)과 코어 결과(유닛 상태·운전 모드·PMS 지령)를 `ST_PcsPublish_V1`에 채운다. 필드 대응표는 FB 머리말에 있다. 읽지 않는 필드는 0, 통신이 끊기면 레코드 전체가 0. 배율 입력(`i_f…Scale`)은 레지스터 맵으로 확인되지 않아 기본 1.0 |

흐름: 드라이버 읽기 → `FB_PlaspoPcs.RawMap` → MAIN 액션 `A_Report`(현장마다 한 줄)가 `GVL_SITE.g_fbRptPcs1`을 불러 `GVL_IF.g_stReport`에 씀(번호 `g_nReportSeq` 홀수/짝수) → `PRG_Mqtt`가 복사해 `GVL_MQTT.g_stPcsPublish.Devices[i-1]`에 덮어씀(시험 모드의 더미 위) → 발행. 확인할 곳: `GVL_IF.g_stReport.astPcs[1]`(매핑 직후)와 `GVL_MQTT.g_stPcsPublish.Devices[0]`(발행 직전). 보고 전용이며 제어는 이 값을 읽지 않는다 (R6).

### 20_Module/Virtual — 시험용

사무실에서 실제 장비 없이 돌려 보기 위한 것이다. `TEST_MODE = TRUE`일 때 쓴다. 현장 빌드에서는 `GVL_SITE` 인스턴스를 실제 드라이버로 바꾼다.

| 파일 | 하는 일 |
|---|---|
| `FB_PcsSim` | 가상 PCS |
| `FB_BmsSim` | 가상 BMS |
| `FB_MeterSim` | 가상 계측기 |
| `FB_SrcTest` | 시험 지령원. 온라인으로 값을 써서 지령·상태 요청·P·Q 제한(`i_bPLimActive`·`i_fPMax`·`i_fPMin`, `i_bQLimActive`·`i_fQMax`·`i_fQMin`)을 넣는다 |
| `PRG_SimLink` | 가상 장비끼리 연결 (사무실 시험 전용, `TEST_MODE`일 때 MAIN ⓪에서 부름). 등록된 가상 PCS 출력 → 배터리 그룹마다 그 그룹 BMS 직류 전력(`BMS_CFG`), 전체 합 → 계측기1 P·Q |
| `PRG_MqttTestData` | MQTT 시험용 더미 발행 값 (8개 토픽 전부, 1 s마다, 10분 주기 삼각파)과 수신 기록 `A_HandleRx`(브로커별 수신 횟수·마지막 헤더·받은 레코드 사본·`sLastRx`). `TEST_MODE`일 때만 `PRG_Mqtt`가 부름. 발행 값은 PLC 상태와 무관한 가짜 값이다 |

### 99_Test — 시험 프로그램

`TEST_MODE`일 때 MAIN ⓪에서 부른다. 자동 시험은 따로 둔 인스턴스만 써서 장비 지령에 영향이 없고, 실경로 시험은 `bStart := TRUE`를 써야 시작한다 (끝나면 바꾼 설정·운전 모드·가상신호를 되돌리고 PCS 정지). 결과는 `bDone`·`bPass`·`wsFail`, 자동 시험은 `fbTally.q_nPass`·`q_nFail`·`q_anFail`.

| 파일 | 하는 일 |
|---|---|
| `FB_TestTally` | 시험 결과 세기 (통과·실패 수, 실패한 시험 번호) |
| `PRG_TestCtrlMode` | 자동 시험: 스케줄 모드 단위(1~19), 관리자 전환·거부·재시도·설정 점검(20~33) |
| `PRG_TestUnitMode` | 자동 시험: 유닛 단위 관리자 출력 칸·접속점 몫·설정 점검, `FB_UnitView` 버스 몫 (40~50), 스탠다드 스케줄 모드 (51~57: 켜지기 전 지령 무시, 방전 100 유지, 정지 → 0, Run만으로는 안 되살아남, 이상값 버림, EMS 끊김), NaN·무한대 거르기 (58~63: `F_IsFinite`·`F_CmdOk`·모드·`FB_EmsCommand`에 NaN을 넣어도 PLC가 멈추지 않음) |
| `PRG_TestSchedLive` | 실경로: PCS1 스케줄 (지금 시각부터 −200 / +100 / 칸 없음 0) |
| `PRG_TestPeakLive` | 실경로: PCS1 피크컷 4단계(P1~P4)·통합 4단계(I1~I4), 부하 모델 |
| `PRG_TestUnitLive` | 실경로: PCS1·PCS2에 다른 시간표·모드를 넣고 PCS마다 실제 출력 판정, 접속점 몫(`ByRating`·`Actual`) |
| `PRG_TestStdLive` | 실경로: EMS 명령 레코드(`GVL_MQTT.g_astPcsCommand[1]` = PCS2)를 넣어 `FB_EmsCommand` → `GVL_IF` → MAIN → 스탠다드 스케줄 모드 → 관리자 2 → 코어 고르기(`MAIN.fbSelect`)까지 판정. 켠 직후 0 / 방전 100 / 충전 −50 / 정지 0. 장비 쓰기(`DEVICE_WRITE`)와 무관 |

### 90_Project — 현장 설정과 MAIN

현장 차이는 이 폴더에만 둔다.

GVL 파일(`GVL_PARAM` `GVL_SITE` `GVL_PERSIST` `GVL_TEST` `GVL_IF`)이 담는 것은 5장 표에 있다. 그 밖의 파일:

| 파일 | 내용 | 바꾸는 방법 |
|---|---|---|
| `ST_ModuleSet` | 현장 모듈 설정 묶음의 타입 (모듈마다 칸 하나) | 현장에 모듈을 추가할 때 칸 추가 |
| `MAIN` | 한 스캔 (⓪~⑥). 모든 코어 FB 인스턴스가 여기 있고, `GVL_SITE`·`GVL_PERSIST` 값을 코어에 넘긴다. 액션 `A_PassModuleSet`에서 `g_stModuleSet` 칸을 각 모듈 `i_stSet`에 넘기고, 제어 모드 관리자에는 모드 등록 배열·설정·현지 시각(`fbClock`)·유닛 요약 주소(`ADR(fbUnitView.q_astView)`)를 넘긴다 | 새 장비 종류·새 확장 지점, 모듈 설정 한 줄을 추가할 때 |
| `PRG_Mqtt` | MqttTask: 장치 번호 → 시험 더미 → `GVL_IF.g_stReport`의 매핑된 PCS 레코드 덮어쓰기 → 발행·구독 → 수신 기록 → 명령 핸들러 → `GVL_IF.g_stEmsCmd`. 현재 PCS1은 실값 매핑을 사용하며, 시험 모드의 미매핑 레코드는 더미를 유지한다 | 보고·명령 연동을 확장할 때 |
| `PRG_Mqtt` (로거) | 2. 발행·구독 안에서 `FB_MqttMain`이 로거 이벤트를 `pms_log`로 발행 (Active PLC만) → 2-1. (시험 모드) 로거 발행 시험 `PRG_TestLogger` (버튼을 누를 때만 로그를 남김, 3. 수신 기록과 따로) | 로거 발행·시험을 바꿀 때 |
| `PRG_Fast` | FastTask(선택): 빠른 계측이 필요한 현장만 | 필요할 때 |
| `PRG_Redundancy` | RedundancyTask: 이 PLC의 역할(`g_eRedRole`) → 상대 PLC 판정(`FB_RedLink`) → 동기화(`FB_RedSync`) → 상태 기계(`FB_RedState`) → Active(`g_bRedActive`)·장비 쓰기 허가(`g_bRedWrite`). 멈췄다 이어짐도 여기서 감지 | 이중화 현장은 `GVL_SITE.RED_SET`과 XAE의 EAP·I/O 연결 (7장 "이중화") |

---

## 5. 설정 위치

| 파일 | 담는 것 | 바꾸는 방법 |
|---|---|---|
| `GVL_PARAM` | 대수와 등록 칸 수 (배열 크기), 계산값(`MODULE_MAX`, `TRACE_STEP_MAX`) | 고치고 재빌드한다 |
| `GVL_SITE` (상수) | 구성(`UNIT_CFG`·`BMS_CFG`·`POINT_CFG`·`GROUP_CFG`), 현장 맞춤 상수(아래 표) | 고치고 재빌드한다 |
| `GVL_SITE` (일반 변수) | 장비·모듈 인스턴스(연결만), 등록 배열 | 고치고 재빌드한다 |
| `GVL_PERSIST` | 운전 모드 요청, 그룹 설정, Manual 정책, 지령원·제한·기동 조건·보정 등록, 보호 설정, 출력 규칙 설정, 명령 필터, Manual 지령 거르기, 시퀀스 설정, 현장 모듈 설정 묶음, 재기동 기록 (아래 표) | 운전 중 HMI·온라인으로 바꾼다. 변경 후 1 s 동안 추가 변경이 없으면 저장하며, 저장 성공한 값은 재기동해도 유지된다 |
| `GVL_TEST` | 가상신호, 시험 리셋 (`g_bTestReset`: 모든 PCS·BMS 고장 리셋 + 유지 DO 풀기) | 시험용이다. `TEST_MODE`가 FALSE면 안 쓴다 |
| `GVL_IF` | 태스크 사이에 주고받는 값 (이중화 활성 `g_bRedActive`·쓰기 허가 `g_bRedWrite`·상태 `g_eRedState`·상대 상태, MQTT 명령 칸 `g_stEmsCmd`·`g_nCmdSeq`, 보고 칸 `g_stReport`·`g_nReportSeq`) | 태스크 PRG와 MAIN이 쓴다 |

`GVL_SITE` 현장 맞춤 상수:

| 이름 | 타입 | 내용 |
|---|---|---|
| `TEST_MODE` | BOOL | 사무실 가상 시험. FALSE(현장)면 `PRG_SimLink`, 가상신호, `GVL_TEST.g_bTestReset`을 안 쓴다 |
| `START_MODE` | `E_StartMode` 유닛 배열 | 기동 방식. `Step`(단계별, 기본) / `Sequence`(순서 기동) |
| `MODE_SET` | `ST_ModeSet` | 운전 모드 전환 단계 |
| `LOG_SET` | `ST_LogSet` | 로그 유지 시간·저장 주기·자세한 기록 켜기 |
| `RED_SET` | `ST_RedSet` | 이중화: 사용 여부(`bUse`, 기본 FALSE), Primary·Secondary PLC의 AMS Net ID, 판정·절체·복귀 시간 (7장 "이중화") |

`GVL_PERSIST` 지령·시퀀스·모듈 설정 (운전 중 변경, 칸은 12장):

| 이름 | 타입 | 내용 |
|---|---|---|
| `g_stCmdFilter` | `ST_CmdFilterSet` | 이상값 거르기 (Auto·Manual 공통) |
| `g_stManualCmdSet` | `ST_ManualCmdSet` | Manual 지령 거르기 (방향 반대 무시, Run 뒤 새 지령만) |
| `g_stSeqSet` | `ST_SeqSet` | 시퀀스: 쓰기 주기, PCS 명령 확인·재시도·정지 램프, 배터리 투입 시점·확인·재시도 |
| `g_stModuleSet` | `ST_ModuleSet` (`90_Project`) | 현장 모듈 설정 묶음. 모듈마다 칸 하나 (기본: `stSocLimit : ST_SocLimitSet`, 제어 모드는 아래 표) |

- 모듈 설정은 모듈 인스턴스 선언이 아니라 `g_stModuleSet` 칸에 둔다. MAIN 액션 `A_PassModuleSet`이 매 스캔 맨 앞에서 `GVL_SITE.g_fbXxx.i_stSet := GVL_PERSIST.g_stModuleSet.stXxx;`로 넘긴다. 줄이 없으면 그 모듈은 설정 구조체의 기본값으로 돈다.
- 인스턴스 선언에는 연결만 남긴다: 모듈 ID, 이름, 읽을 장비(`i_ipAux`), 점 번호(`i_nPoint`).
- `g_stModuleSet`은 `FB_PersistSave`가 주소·크기로 통째로 비교해 변경 후 1 s 동안 추가 변경이 없으면 저장한다. 크기가 `GVL_PARAM.MODULE_SET_BYTES`(8192, PCS 2대 제어 모드 설정 때문에 4096에서 늘림)보다 크면 설정 오류다 (`fbSettingGuard.q_bModuleSetTooBig` → `FB_ConfigCheck`). `MODULE_SET_BYTES`를 늘린다.
- 제어 모드 설정 (`g_stModuleSet`, PCS마다 한 세트. N = PCS 번호):

| 칸 | 타입 | 내용 |
|---|---|---|
| `stCmCommon` | `ST_CmCommonSet` | 공통 하나: 월 → 계절, 휴일, 방전 허용 시간, 역송 방지 (모든 PCS의 스케줄·피크컷·통합이 같이 씀) |
| `stCtrlModeN` | `ST_CtrlModeSet` | 관리자 N: 전환 때 출력 0(`bZeroOnSwitch`, `tZeroDelay` 1 s), 스스로 끝날 때 넘어갈 모드(`nFallback`, 0 = 없음), 접속점 몫(`ePccShare` 기본 ByRating, `fPccRatio`) |
| `stCmConstPN` | `ST_CmConstPowerSet` | 정출력 값 (kW, 방전 +) |
| `stCmScheduleN` | `ST_CmScheduleSet` | PCS N 시간표. 스케줄·통합 모드가 같이 씀. 예: 봄·가을 월요일 1칸 = `GVL_PERSIST.g_stModuleSet.stCmSchedule1.astSlot[1, 1, 1]` (`nStart` 1000, `nEnd` 1200, `fP` 200) |
| `stCmPeakCutN` | `ST_CmPeakCutSet` | 피크컷: 목표 피크(PCS N 몫), 자동 충전·잔량 방전 시간 |
| `stCmSchedPeakN` | `ST_CmSchedPeakSet` | 통합: 목표 피크(PCS N 몫), 잔량 방전 시간 |

- 켤 모드는 설정이 아니라 관리자 입력이다: `GVL_SITE.g_fbCtrlMode1.i_nModeReq` (모드 칸 번호, 0 = 없음). PERSISTENT가 아니라서 재기동하면 0(모드 없음)이다 (10장, 2026-11-09 14번 탭 2번: 이대로 유지로 결정).
- 새 PERSISTENT 변수는 처음 내려받을 때 선언 초기값으로 시작한다 (이미 저장된 파일이 있어도 새 칸은 초기값).
- PERSISTENT 변수의 모양(배열 크기·구조체 칸)이 바뀌면 내려받을 때 "partly restored"(일부만 되살림) 또는 "not restored" 경고가 뜬다. 예전 칸은 예전 값으로 남으므로, 그 변수를 온라인에서 확인하거나 Reset Origin으로 기본값을 다시 넣는다. 예: 보호 모듈화(2026-10-22)로 `g_astRuleCfg`가 1칸 → 5칸이 되어, 그 전 버전에서 올리면 1칸(PCS 고장)이 예전 빈 칸 값(조치 None)으로 남는다.

`GVL_PERSIST` 그룹 설정:

| 이름 | 내용 |
|---|---|
| `g_astGroupSet[그룹]` (`ST_GroupSet`) | 고르기 방식, 램프(Auto·Manual 공통), Q 필터, 분배 기준, 피상전력 우선 |
| `g_astManualPolicy[그룹]` (`ST_ManualPolicy`) | Manual 경로에 적용할 단계 (7장). Auto는 항상 전부 적용(`FB_ShapePath.APPLY_ALL`) |

- 모듈 ID 범위: 1~99 장비, 100~199 지령원(제어 모드 관리자 102·103), 200~899 기능 모듈(제어 모드 240~: PCS1 240~243·248, PCS2 244~247·249), 900~ 표준 모듈(`20_Module/Standard`, 보호 901~905). 시험 프로그램 인스턴스는 9000~. 0은 코어 예약이다. 선언 때 `(i_nModuleId := n, i_sName := '영문')`로 준다. 이름은 15자 이내 (로그 문구 길이, 7장).
- 등록 칸은 최소 1개다. 빈 칸은 등록하지 않은 것으로 본다 (`<> 0`으로 확인).
- 모듈 등록(`ST_ModuleReg`): `bEnable`, `nGroup`(0 = 모든 그룹), `bManual`(제한 모듈만: 접속점 기준 모듈을 Manual 경로에도 적용. **기본 TRUE**, 7장 "접속점 기준 제한과 Manual"). 그룹이 2개 이상이면 제한·보정·기동 조건 모듈은 `nGroup`을 지정하고 그룹마다 인스턴스를 둔다 (0이면 설정 오류).
- 유닛 제한·버스 제한 등록(`ST_RuleTargetReg`, `GVL_PERSIST.g_astUnitRuleReg`·`g_astBatRuleReg`): `bEnable`, `nTarget`(유닛 제한 = 유닛, 버스 제한 = 배터리 그룹. 0 = 전체, 모듈이 대상별 상태를 따로 가짐), `bAuto`·`bManual`(그 운전 모드 유닛에 적용, `F_ModeApplies`). 칸 수는 `GVL_PARAM`의 `UNIT_RULE_COUNT`·`BAT_RULE_COUNT`.
- 보호 칸(`RULE_COUNT`)은 조건(`g_aipRule`)·대응(`g_aipResp`)·설정(`g_astRuleCfg`)이 같은 칸 번호다. 현장 보호 칸을 늘리면 대응 칸에도 `g_fbProtRespStd`(또는 현장 대응)를 등록한다.
- 코어와 모듈은 `GVL_PARAM`만 본다. `GVL_SITE`·`GVL_PERSIST` 값은 MAIN이 넘긴다.
- 배터리 그룹별로 시퀀스 설정을 다르게 할 요청이 오면 `GVL_PERSIST.g_stSeqSet.stBat`을 배터리 그룹 배열로 바꾼다.

---

## 6. 확장

### 확장 지점

인터페이스·끼우는 것·기반 FB는 4장 "10_Core/01_Interface" 표에 있다.

모듈을 붙이는 순서는 다음과 같다.
1. `20_Module`에 FB를 만든다 (기반 FB 상속).
2. `GVL_PARAM`의 칸 수를 늘린다.
3. `GVL_SITE`에 인스턴스를 선언하고(연결만) 등록 배열에 넣는다.
4. `GVL_PERSIST`에 같은 칸 번호로 등록 설정을 넣는다.
5. 모듈에 운전 중 바꿀 설정이 있으면: 설정 구조체(`ST_XxxSet`, 모듈 옆)를 만들고 모듈은 `i_stSet : ST_XxxSet`으로 받는다. `ST_ModuleSet`에 칸을 추가하고, MAIN 액션 `A_PassModuleSet`에 한 줄 추가한다.
6. 빌드하고 온라인으로 확인한다.

### 모듈 약속

- 장비에 직접 쓰지 않는다. 요청·제한·판단만 낸다.
- 다른 모듈을 직접 부르지 않는다. 필요한 값은 코어가 `ST_GroupView`로 넘긴다.
- 제한·보정 모듈은 `FB_LimitEval`이, 기동 조건 모듈은 `FB_PermitEval`이 그룹마다 한 번 평가한다. 모듈 안 타이머·상태는 한 그룹 것이어야 하므로 그룹이 여러 개면 그룹마다 인스턴스를 둔다.
- 유닛 제한·버스 제한 모듈은 `FB_EquipmentLimits`가 대상마다 스캔당 한 번 부른다. `nTarget` 0(전체)으로 등록하면 한 인스턴스가 여러 대상을 보므로, 모듈은 상태를 대상 번호(`i_stView.nUnit`·`nBatGroup`) 배열로 따로 둔다.
- 유닛·버스 제한 모듈과 그룹 제한 모듈(`I_LimitRule`), 지령원 P·Q 제한은 범위를 좁히기만 한다. 코어가 0을 항상 범위 안에 두므로 제한이 출력을 강제할 수 없다 (상한 < 0이나 하한 > 0은 0으로 바꿈). 강제로 내야 하는 출력(역송 방지 충전 등)은 지령원으로 만든다.
- 보호 조건 모듈은 판단(조건 있음/없음)만 한다. 무엇을 할지는 같은 칸의 대응 모듈이 정한다. 지연은 없다 (통신 에러·고장은 드라이버가 판정해 올림, 떨림 방지가 필요하면 조건 모듈 설정에). 보호 해제를 위한 리셋은 없다 (조건이 있는 동안만). 대상 번호 0으로 등록하면 범위 안 대상마다 한 번씩 불리므로(`i_stView.nTarget`), 모듈 안 상태는 대상 번호 배열로 따로 둔다. 운영 방식·모드에 따라 다르게 판단해야 하면 표준 모듈을 복사해 `i_stView`를 보고 판단한다.
- 보호 대응 모듈(`I_ProtResponse.M_Respond`)은 범위 안 대상마다 매 스캔 불린다 (조건이 없을 때도 `i_bActive = FALSE`로: 단계 대응이 타이머·상태를 풀 수 있게). 요청은 `iq_stReq`(코어가 비워서 넘김)에 쓰고, 코어가 장비마다 합친다. 장비 상태·소속·운전 모드는 읽기 전용 참조로 받는다 (대상 밖 장비도 볼 수 있음). 장비 사이 순서(PCS 정지 확인 → 다음 단계 등)는 장비 상태를 보고 진행한다. 배터리 개방·비상정지가 PCS 정지 확인 뒤에 나가는 것은 시퀀서가 지킨다.
- 대응이 요청할 수 있는 것은 `ST_ProtReq` 칸뿐이다 (PCS 조치·Manual 전환·P·Q 상한·배터리 조치). 목록 밖 동작(보조 DO, 새 장비 명령)이 필요하면 표준으로 칸을 늘린다 (새 칸 기본값은 요청 없음).
- 지령원은 새 지령을 받은 스캔에만 `_stOut.bNewSet := TRUE`로 한다 (Manual 즉시 쓰기에 쓰인다). 새 지령은 값이 바뀐 것이 아니라 보낸 것이다. 운전자가 같은 값을 다시 보내도 `bNewSet`을 세운다 (Run 전에 넣어 버려진 값을 Run 뒤 다시 보낼 수 있어야 한다). 상태 요청도 새 명령일 때 한 스캔만 낸다. 구현할 메서드는 `M_UpdateOutput`이다.
- 이유 문구는 모듈이 `M_ReasonText`로 직접 돌려준다. 이유 문구는 20자 이내, 모듈 이름(`i_sName`)은 15자 이내로 한다. 둘 다 로그 문구 인자로 들어가기 때문이다 (7장 "로그 문구 길이").
- 제어 모드는 지령원 안의 부품이라 `FB_ModuleList`에 넣지 않는다. 관리자가 `M_CheckSet`을 대신 부른다. 모드 ID는 `FB_ConfigCheck`가 따로 점검한다 (`iq_aipCtrlMode` = `g_aipCtrlMode1`): 0, 제어 모드끼리 겹침, 등록 모듈과 겹침이면 설정 오류 (2026-11-09, 14번 탭 2번).
- 운전 중 바꾸는 설정(한계값·지연·켜기 등)은 VAR_INPUT 낱개로 두지 않고 설정 구조체 하나(`i_stSet`)로 받는다. 설정이 있는 모듈은 `M_CheckSet`을 구현해 잘못된 값(상·하한 뒤집힘, 음수 폭 등)이면 짧은 오류 문구를 돌려준다. `FB_SettingGuard`가 그 값을 받지 않고 마지막 정상값으로 되돌린다 (모듈 설정 묶음 통째, 기록 "모듈 이름: 문구"). 시작부터 잘못이면 설정 오류가 된다. 값은 `GVL_PERSIST.g_stModuleSet`에 있다 (5장). 연결(장비·점 번호)만 VAR_INPUT 낱개로 둔다.
- 장비 드라이버는 `IF M_IsReadTime() THEN` 안에서 읽고, 정상 응답이면 `M_MarkAlive()`, 매 스캔 `M_UpdateComm()`을 부른다. 쓰기(명령·지령) 응답을 받을 때마다 `M_MarkWriteResult(성공 여부)`를 부르고, 상태의 `bWriteFail`에 `_bWriteFail`을 넣는다 (로그 "장비 쓰기 실패" 알람). 출력·명령은 받으면 드라이버 안 목록에 넣고, 실제 전송은 `M_Cycle`에서 한다. 목록은 가장 최근 것만 유지한다.
- 드라이버는 표준 약속(방전·공급 +, kW·kvar)에 맞춰 값을 낸다. 리모트 신호가 있으면 `bCtrlReady`에 매번 넣는다.
- 한 인스턴스를 보호 규칙과 출력 규칙에 같이 등록할 수 있다. 판단 상태(히스테리시스 등)는 두 용도를 따로 둔다 (`FB_RuleValue`: `_bProtCond`·`_bOutCond`).

### 기능별로 붙이는 곳

| 기능 | 붙이는 곳 |
|---|---|
| 운전 방식 (스케줄·피크컷·피크저감·태양광 연계 등, 한 번에 하나) | 제어 모드 `I_ControlMode` (`FB_CtrlModeBase` 상속) → 관리자 등록 배열 `g_aipCtrlModeN` (7장 "제어 모드") |
| 알고리즘 여러 개를 동시에 경합 | 알고리즘마다 지령원 하나 (등급 = 우선순위). 등급·절대값으로 안 되는 경합 규칙은 고르기 정책 `I_SelectPolicy` |
| 현장 분배 규칙 (SOH·운전 시간·우선 유닛 등) | 분배 정책 `I_DistPolicy` |
| 피크저감 / 역송 방지 (출력을 정하는 쪽) | 제어 모드 (피크컷·통합, 모드 안 역송 방지) |
| 피크저감 / 역송 방지 (어떤 지령이든 막는 쪽) | 접속점 기준 제한 (하한 / 상한). Manual에도 걸림 (등록 `bManual` 기본 TRUE + 정책 `bPccLimit` 기본 TRUE). 7장 "접속점 기준 제한과 Manual" |
| SOC 감발·막음·재개 (버스별) | 버스 제한 `I_BatLimitRule` (공통 `FB_SocLimit`) |
| SOC 한계 뒤 다시 충·방전하지 않음 (제한은 늘 돌아오므로) | 보호: 조건 모듈(SOC 한계) + 대응(Manual 전환 또는 정지) |
| 조건에 따라 단계로 대응 (예: 대기 → 10 s → 정지 → 배터리 개방) | 대응 모듈 `I_ProtResponse` (표준 대응을 복사해 고침). 장비 상태를 보고 단계 진행 |
| PCS별 사용자 한도 | 유닛 제한 `I_UnitLimitRule` (공통 모듈 `FB_UnitLimit`) |
| 그룹 합계 한도 (설비 기준) | 설비 기준 제한 `I_LimitRule` |
| 자동 지령 접수·보류·재개 정책 | Gate 모듈 `I_Gate` → `GVL_SITE.g_ipAutoGate` (비우면 통과) |
| Manual 지령 거르기를 현장 규칙으로 | Gate 모듈 `I_Gate` → `g_ipManualGate` (비우면 코어 기본: 방향 반대 무시·Run 뒤 새 지령 대기) |
| 접점 입력 (화재·비상정지 등) | `FB_AuxIo` + `FB_RuleBit` → 보호 규칙 |
| 아날로그 입력 (온도·습도 등) | `FB_RuleValue` → 보호 규칙 또는 출력 규칙 |
| DO 출력 (팬·경광등·트립) | 출력 규칙 + `g_astOutputCfg`. 보호 동작 출력은 기본으로 두지 않는다 |

### 기존 종류에 안 맞는 새 장비

점 몇 개만 주고받는 장비는 보조 장비(`I_AuxDevice`)로 처리한다. 상태가 구조화돼 있거나, 명령을 주고받거나, 이중 연결이 있으면 새 장비 종류로 만든다. 코어는 고치지 않는다.

| 할 일 | 위치 |
|---|---|
| 인터페이스 `I_Xxx EXTENDS I_Module` (`M_Cycle`, 상태 속성, 필요한 쓰기 메서드) | `20_Module/<현장>` (여러 현장이 쓰면 `Common`) |
| 상태 구조체 `ST_XxxState` | 같은 폴더 |
| 드라이버 `FB_<현장>Xxx EXTENDS FB_DeviceBase` | 같은 폴더 |
| 대수 `XXX_COUNT` | `GVL_PARAM` |
| 인스턴스, 등록 배열 `g_aipXxx` | `GVL_SITE` |
| 매 스캔 `M_Cycle` 호출 | `MAIN` ①단계 |
| 보호 대응·지령원이 이 장비에 명령을 요청해야 하면 `I_ReqDevice`도 구현 | 같은 드라이버 (모양만 정해 둠: 명령 코드·세기·확인·재시도는 장비 모듈이, 코어는 요청을 모아 센 쪽 하나를 넘김. 코어 연결은 처음 쓰는 장비가 생길 때) |

```
새 장비 ──▶ 드라이버 (I_Xxx) ──┬─▶ 지령원 모듈          장비가 내리는 명령·셋포인트 → 요청
                               ├─▶ 보호 조건·대응        셧다운·트립 → 보호 조치 / 장비에 명령 요청 (I_ReqDevice)
                               ├─▶ 출력 제한·기동 조건    장비 상태로 제한·허가
                               └─◀ 보고 쪽              장비에 써 줄 값
```

- 모듈은 선언 때 장비를 가리킨다 (`i_ipXxx := g_fbXxx1`).
- 한 인스턴스를 여러 등록 배열에 넣을 수 있다 (`FB_ModuleList`가 한 번만 셈). 예: 차단기 모듈(보호 규칙 + 출력 규칙 2칸), 보호계전기 드라이버 하나가 `I_Meter` + `I_AuxDevice`, 플랜트 컨트롤러 드라이버 하나가 `I_Pcs` + `I_Bms`("유닛 1개 + 배터리 그룹 1개"). 장비 드라이버를 두 장비 배열에 넣으면 MAIN이 `M_Cycle`을 배열마다 부르므로, 드라이버는 스캔당 한 번만 통신하도록 막아 둔다 (`FB_BreakerCtrl.M_Update`처럼 태스크 사이클 번호로 확인).
- 새 등록 배열은 `FB_ModuleList`와 `GVL_PARAM.MODULE_MAX`에 추가한다. 그러면 모듈 ID 점검과 로그 이름 찾기에 자동으로 들어간다.
- 배터리 없는 출력 장치(`nBatGroup = 0`)는 ESS 유닛과 같은 그룹에 넣어도 된다. 그룹 배터리 줄은 버스별로 계산하고, 배터리 없는 유닛은 자기 한도로 더한다.

### 자주 하는 작업

**새 현장 설정 (코어는 그대로)**

1. `GVL_PARAM`: 장비 대수와 등록 칸 수를 현장에 맞춘다. 칸은 최소 1개다 (배열 크기라 0이면 빌드가 안 된다).
2. `GVL_SITE` 구성표: `UNIT_CFG`(유닛마다 PCS 번호·PCS 안 번호·버스·그룹·정격), `BMS_CFG`, `POINT_CFG`(계측기 부호 포함), `GROUP_CFG`(접속점), `START_MODE`.
3. `GVL_SITE` 인스턴스: 가상 장비(`FB_PcsSim` 등)를 실제 드라이버로 바꾸고, 모듈 ID·이름·IP·읽기 주기를 준다. 등록 배열 `g_aip…`에 넣는다.
4. `GVL_SITE.TEST_MODE := FALSE`.
5. `GVL_PERSIST`: 지령원 등록(`g_astSourceReg`: 등급·받는 모드·만료), 그룹 설정, 보호 설정, 모듈 등록 설정을 같은 칸 번호로.
6. 빌드 → 온라인으로 `fbConfigCheck.q_bConfigOk`가 TRUE인지 먼저 본다. FALSE면 오류 문구(최대 10건)를 보고 고친다.
7. 시운전 때는 `LOG_SET`의 자세한 기록(`bUnitLimited`, `bReqRefused`, `bSeqChange`, `bModeChange`)을 켜 두면 원인 추적이 쉽다.

**이중화에서 현장 모듈 설정을 두 PLC에 맞추기**

- 표준은 모듈 설정 묶음(`g_stModuleSet`)을 동기화하지 않는다. 두 PLC의 값이 달라지면 안 되는 현장은 조각 전송 FB(`FB_RedSyncMod`)를 `90_Project`에 더한다. 코드·XAE 설정·시험은 7장 "현장 모듈 설정 동기화".
- 더하지 않는 현장은 HMI가 두 PLC 모두에 쓴다 (7장 "운영 규칙").

**알고리즘을 여러 개 경합시키기**

- 알고리즘마다 지령원 하나(`I_CommandSource`)를 만들어 `g_aipSource`에 등록하고, `g_astSourceReg[n]`에 등급(`nGrade`), 받는 모드(`ePowerMode`), 그룹(`nGroup`), 만료(`tExpire`)를 준다.
- 그룹마다 `FB_Select`가 `eSelP`·`eSelQ` 방식(등급 높은 것 / 절대값 작은 것)으로 하나를 고른다. 지령원 P·Q 제한은 고르지 않고 전부 겹친다.
- 조건별 우선순위·합치기처럼 규칙이 복잡하면 고르기 정책 모듈(`I_SelectPolicy`)을 만들어 `GVL_SITE.g_aipSelectPolicy[그룹]`에 넣는다. 코어는 이상값을 뺀 후보(`ST_SelectView.astCand`: 지령원 칸·모듈 ID·등급·값·새 지령)를 넘기고, 정책이 낸 `ST_Pick` 하나를 쓴다. 합칠 때 `nModule`은 대표 지령원 하나를 넣는다 (로그·진단용). 정책은 같은 입력에 같은 결과를 내야 한다 (한 스캔에 유닛·그룹·P·Q·경로마다 여러 번 부름). 한 번에 하나만 켜는 운전 방식이면 경합시키지 말고 제어 모드로 둔다 (7장 "제어 모드").
- 분배 규칙을 바꾸려면 분배 정책 모듈(`I_DistPolicy`)을 `g_aipDistPolicy[그룹]`에 넣는다. 유닛마다 무게(0 이상)만 돌려주면 한도·재분배·버스·피상전력은 코어가 그대로 한다. 음수·이상값 무게는 0으로 보고, 무게가 모두 0이면 그룹 몫을 아무 유닛에도 안 준다.

**제어 모드 쓰기 (PCS마다)**

1. 온라인으로 설정을 넣는다: 시간표 `GVL_PERSIST.g_stModuleSet.stCmScheduleN.astSlot[계절, 요일, 칸]`, 피크 목표 `stCmPeakCutN.stPeak.fTarget` 등 (5장 표). 공통(계절·휴일·방전 허용 시간·역송 방지)은 `stCmCommon`.
2. 유닛을 Auto로 (`GVL_PERSIST.g_aeOpMode[유닛]`), 배터리 투입·Run 요청.
3. `GVL_SITE.g_fbCtrlMode1.i_nModeReq`(접속점 그룹 1)에 모드 칸 번호를 쓴다 (1 정출력, 2 스케줄, 3 피크컷, 4 통합, 5 스탠다드 스케줄, 0 없음).
4. 확인: `g_fbCtrlMode1.q_nActive`(켜진 칸, 전환 중 0), `q_eStep`, `q_eRefuse`·`q_nRefuseReason`(거부), `q_nExitSlot`·`q_nExitReason`(스스로 끝냄), `q_fPccP`(모드가 본 접속점 P). 모드 안은 `g_fbCmScheduleN.q_nState`·`q_nSlot`·`q_fOutP`, 피크는 `g_fbCmPeakCutN.q_fPeakCtrl`·`q_nRemainMin`·`q_fExceeded`. 이유 문구는 `g_fbCtrlModeN.M_ModeReasonText(칸, 이유)`.
5. 같은 값을 다시 처리하게 하려면(거부·스스로 끝난 뒤) `i_bRetry := TRUE`.

**접속점 그룹 늘리기 (제어 모드 한 세트 더)** — 2026-11-09부터 제어 모드는 접속점 그룹 단위라 PCS를 늘려도 관리자는 늘리지 않는다 (`UNIT_CFG`·`g_aipPcs`만). 아래는 접속점 그룹(접속점)을 하나 더 둘 때

1. `GVL_PARAM`: `PCS_COUNT`·`UNIT_COUNT`, `SOURCE_COUNT` +1.
2. `GVL_SITE`: `UNIT_CFG`·`START_MODE`에 줄, PCS 인스턴스와 `g_aipPcs`, 관리자 `g_fbCtrlModeN := (i_nModuleId, i_sName, i_nUnit := N)`와 모드 4개(새 모듈 ID), `g_aipSource`에 관리자, `g_aipCtrlModeN` 등록 배열.
3. `GVL_PERSIST.g_astSourceReg`에 같은 칸 (관리자 칸과 같은 값: 등급 2, `ePowerMode` Auto, 상태 요청 None).
4. `ST_ModuleSet`에 `stCtrlModeN`·`stCmConstPN`·`stCmScheduleN`·`stCmPeakCutN`·`stCmSchedPeakN` 칸, 크기가 `MODULE_SET_BYTES`를 넘지 않는지 (PCS 한 세트 약 1.8 KB).
5. MAIN `A_PassModuleSet`에 관리자 4줄 + 모드 설정 8줄.
6. 그룹 하나로 제어할 현장은 관리자 하나(`i_nUnit := 0`)와 모드 한 세트만 둔다 (그룹 지령을 분배가 나눔).

**새 제어 모드 만들기**

1. `20_Module/ControlMode`(현장 전용이면 현장 폴더)에 `FB_CmXxx EXTENDS FB_CtrlModeBase`. 시간표·요금제가 있으면 `FB_CmTariffBase`를 상속해 공통 계산을 쓴다.
2. `M_Calc`(필수): `ST_CtrlModeOut`(P·Q 지령, 스스로 끝냄 `bExitReq`·이유)을 돌려준다. 필요하면 `M_CheckEnter`(진입 조건, 상태를 바꾸지 않음), `M_Enter`(내부 상태 초기화, `SUPER^.M_Enter` 부름), `M_Exit`, `M_ReasonText`, `M_CheckSet`.
3. 설정은 `ST_CmXxxSet` + `i_stSet` (모듈 약속과 같음). 모드에 넘어오는 값은 `i_stView`(`stGroup` = 그룹 또는 유닛 요약, `stTime` = 현지 시각)뿐이다.
4. `GVL_PARAM.CTRL_MODE_COUNT` 늘리고, 관리자마다 인스턴스를 따로 선언해 `g_aipCtrlModeN`에 넣는다 (모드는 상태를 가지므로 관리자끼리 같이 쓰지 않는다). 설정 칸 + `A_PassModuleSet` 줄.

**PCS·배터리 없이 모니터링·VCB 제어만**

- 대수 상수(`PCS_COUNT`, `BMS_COUNT`, `UNIT_COUNT`, `BAT_COUNT`)는 배열 크기라 0으로 둘 수 없다. 1로 두고 `g_aipPcs`, `g_aipBms`를 빈 칸으로 둔다. 빈 칸은 미설치로 본다.
  - 미설치 유닛: PCS 표준 보호(고장·통신)가 안 걸리고, 가용 이유는 `bNotInstalled`만, 설정 점검에서 정격을 안 본다. `UNIT_CFG`에는 형식상 한 줄만 있으면 된다 (`nPcs` 1, 정격은 0이어도 됨).
  - 미설치 BMS: 배터리 그룹 집계에서 빠지고 BMS 표준 보호가 안 걸린다.
  - 그래서 설정만 하면 "통신 끊김" 경보·로그가 생기지 않는다.
- 계측기·계측점·보조 장비 입력은 그대로 동작한다.
- VCB 자동 개방(보호 트립 등): 출력 규칙(`FB_RuleBit`·`FB_RuleValue`) + `g_astOutputCfg`의 DO 점. 펄스 코일이면 `tOffDelay`를 펄스 폭으로 쓴다 (조건이 한 번 켜지면 그 시간만큼 유지).
- VCB 운전원 투입·개방: 공통 모듈 `FB_BreakerCtrl` (코어 수정 없음).
  1. 인스턴스 선언: `g_fbVcb1 : FB_BreakerCtrl := (i_nModuleId := 220, i_sName := 'VCB1', i_ipAux := 상태 접점 장비, i_nClosedPoint := 투입 상태 DI)`.
  2. 출력 규칙 두 칸에 같은 인스턴스를 등록: `g_astOutputCfg` 투입 코일 칸 `fThreshold := 1`, 개방 코일 칸 `fThreshold := 2`, `nAux`·`nPoint` = 코일 DO, 지연 0·유지 끔.
  3. 보호 규칙 한 칸에도 등록: `g_astRuleCfg` 조치 None = 투입·개방 실패 알람만 (정지가 필요하면 조치 Stop).
  4. 설정 `ST_BreakerSet`(펄스 폭 1 s, 확인 시간 5 s)은 `g_stModuleSet` 칸 + `A_PassModuleSet` 한 줄.
  5. HMI는 `g_fbVcb1.i_bCloseCmd`·`i_bOpenCmd`(상승 에지 1회), `i_bResetFail`을 켜고, `q_bClosed`·`q_bBusy`·`q_bCloseFail`·`q_bOpenFail`을 본다.
  - 개방은 언제나 받고(조작 중이어도 끼어듦), 투입은 상태 접점이 정상이고 개방 상태일 때만 받는다. 같이 오면 개방이 이긴다. 확인 시간 안에 상태 접점이 바뀌지 않으면 실패.
  - 보호 트립 같은 자동 개방은 다른 출력 규칙을 같은 개방 코일 점에 등록한다 (같은 점은 OR).
  - 계측기(보호계전기)가 차단기 제어를 겸하면 드라이버 하나가 `I_Meter` + `I_AuxDevice`를 구현해 두 배열에 등록한다 (위 "한 인스턴스를 여러 등록 배열에" 참고).

**출력이 왜 안 나가는지 찾기**

1. `fbConfigCheck.q_bConfigOk` — FALSE면 PCS·BMS 쓰기와 보조 DO 켜기가 막힌다 (DO 끄기는 허용).
2. `fbOutputDiag.q_astAutoPDiag[그룹]` (Manual이면 `q_astManualPDiag`) — 막은 단계(`eStage`)·모듈(`nModule`, 0 = 코어)·이유(`nReason`).
3. `fbOutputDiag.q_astAutoPTrace[그룹]` — 요청에서 쓴 값까지 단계별로 값이 어떻게 줄었는지.
4. 유닛 하나만 이상하면 `fbDistributor.q_astUnitDiag[유닛]`(줄어든 이유 `E_UnitLimitBy`)과 `fbOutputReady.q_aeNotReadyBy[유닛]`.
5. 상태가 안 바뀌면 `fbSequencer.q_aeRefuse[유닛]`(거부 이유), `fbSequencer.q_aeUnitSeq[유닛]`(시퀀스 단계), `fbSequencer.q_aeFailCmd[유닛]`(확인 안 된 명령), `fbProtection.q_astUnitProt[유닛].ePcsAction`(보호 조치: 걸려 있으면 기동 쪽 요청은 버림).

---

## 7. 동작 규칙

### 운전과 분배

- Auto와 Manual은 같은 순서의 두 경로다. 지령 거르기(`fbAutoGate`/`fbManualGate`, 둘 다 `FB_Gate`) → 성형(`fbAutoShape`/`fbManualShape`) → 분배(`M_DistPath`). Manual을 먼저 계산하고 Auto가 그 결과를 받는다.
- 각 경로의 목표는 그 경로 그룹 지령이다. 그룹 지령은 접속점 기준이라 Auto 그룹 지령에는 Manual 몫이 포함된다 (Auto 몫 = 제한 후 그룹 값 − Manual 결과). 없으면 그 경로 유닛 지령 합(유닛별 램프 후)을 목표로 같은 제한을 건다. Manual 그룹 지령도 Auto와 똑같이 처리한다.
- 그룹 지령이 있으면 유닛 지령은 그 합계 안에 포함된다 (더해지지 않는다). 나머지를 같은 경로의 지령 없는 유닛이 `eDistBasis`로 나눈다. 그룹 지령이 없으면 지령 없는 유닛은 0이다.
- 제한에 걸리면 그 경로의 그룹 몫 유닛부터 줄이고, 그다음 유닛별 값을 비율로 줄인다.
- Auto 경로는 접속점 제한을 Manual 몫까지 합쳐서 본다. 깎이는 양은 Auto가 부담한다 (Manual 우선).
- Manual이 한도에 걸려 덜 나간 만큼은 Auto 그룹 지령(접속점 기준)이 있을 때만 Auto가 채운다. Auto에 유닛 지령만 있으면 그대로 둔다.
- 보정은 Auto 경로에 더한다. Manual은 정책 `bComp`가 켜져 있고 같은 그룹에 Auto 유닛이 없을 때만 더한다 (두 번 더하지 않게).
- 같은 배터리 버스의 유닛은 같은 방향으로만 출력한다. 분배는 버스 남은 한도를 보며 나누고, 넘친 몫은 다른 버스 유닛에 재분배한다 (이유 `BatLimit`). 한 버스로 가는 몫이 남은 한도보다 크면 그 버스 유닛 몫을 같은 비율로 줄인다.
  - 버스 방향은 Manual 유닛 쪽 순 방향이 먼저다 (Manual 우선). Manual 유닛이 없거나 순 합이 0이면 전체 순 방향을 쓰고, 그것도 0이면 정할 수 없으므로 반대 방향끼리 모두 0으로 한다 (버스 안에서 전력이 도는 것을 막음).
  - 버스 비율로 몫을 줄였는데 같은 버스의 다른 유닛이 자기 한도에 먼저 걸려 버스 한도가 남으면, 남은 한도는 다음 라운드에 그 버스 유닛에 더 나눈다. 버스 한도를 다 쓴 버스의 유닛만 빠진다.
  - 출력을 못 내는 유닛(멈춤·통신 끊김·새 지령 대기)의 유닛별 Q 지령은 P와 같이 0으로 보고 그룹 Q 합에 넣지 않는다.
  - 보정 모듈 값이 NaN·무한대면 그 값은 버린다 (램프 기준·Q 필터에 남지 않게).
- 분배 대상은 출력 가능 유닛(`FB_OutputReady.q_abReady`)이다: 가용(통신·고장·로컬·배터리·보호 이상 없음), 장비 활성(Run·대기), 정지 진행 중 아님, 모드 전환 아님, Manual 새 지령 대기 아님. 아니면 출력은 0이고 몫은 다른 유닛으로 간다.
- 램프 기준은 지난 스캔 결과다. 그룹마다 `bRamp`로 켜고 끈다 (기본 꺼짐, 두 경로 공통. Manual은 정책 `bRamp`도 켜져 있어야 한다). 제한 모듈이 재개 중이면 오르는 쪽은 `fRampResume`(0이면 `fRampUp`)이다.
- 출력을 못 내는 유닛(멈춤·전환·새 지령 대기)의 성형 값은 0이다. 기동하면 0부터 램프한다. 경로에 출력 낼 유닛이 하나도 없으면 그 경로 그룹 램프 기준을 "이 경로 몫 0"으로 둔다.
- 그룹 값은 `eSelP`·`eSelQ`로 고른다 (`Grade` 기본, `MinAbs`). 지령원 P·Q 제한은 전부 겹친다 (Manual은 정책 `bSrcLimit`). 지령원 Q 제한은 접속점 기준이고, 성형에서 보정만큼 옮겨 정격·제한 모듈 Q 범위와 겹친다.
- 제한 모듈 Q 제한도 P와 같은 규칙이다: 설비 기준 모듈은 `bDevLimit`, 접속점 기준 모듈은 Auto는 항상, Manual은 정책 `bPccLimit` + 모듈 등록 `bManual`. Q 범위는 설비 기준(접속점 + 보정)이라, 접속점 기준 모듈과 지령원의 Q 제한은 보정만큼 옮겨 겹친다.

### Manual 정책 (`GVL_PERSIST.g_astManualPolicy[그룹]`)

| 칸 | 기본 | 끄면 |
|---|---|---|
| `bRamp` | TRUE | Manual은 램프·Q 필터 없이 바로 목표 |
| `bSrcLimit` | TRUE | 지령원 P·Q 제한을 Manual에 안 씀 |
| `bPccLimit` | TRUE | 접속점 기준 모듈을 Manual에 안 씀 (켜도 모듈 등록 `bManual`인 것만) |
| `bComp` | FALSE | 보정을 Manual에 안 더함 |
| `bDevLimit` | TRUE | 설비 기준 모듈(P·Q)을 Manual에 안 씀 |
| `bRating` | TRUE | 이 그룹 Manual 유닛에 정격·Derate·피상전력 한도를 안 씀 |
| `bBatLimit` | TRUE | 이 그룹 Manual 유닛에 BMS 충방전 한도(유닛·배터리 버스)를 안 씀 |

- `bRating`을 끄면 그 유닛은 유닛 제한 모듈(등록 `bManual`)만 쓴다. `bBatLimit`을 끄면 BMS 한도와 버스 제한 모듈 모두 그 유닛에 안 걸리고, 버스 남은 한도에서도 빠진다.
- `bRating`·`bBatLimit`를 꺼도 장비 자체 보호는 그대로다. 하나라도 꺼지면 `fbDistributor.q_abManualLimitOff[그룹]`이 켜진다 (HMI 표시용).
- 유닛·버스 제한 모듈을 Manual에 걸지는 정책이 아니라 모듈 등록 `bManual`로 정한다.
- 보호는 정책이 아니라 보호 항목마다 `ST_ProtCfg.bManual`로 켜고 끈다.
- `ST_ManualPolicy`에 칸을 더하면 `FB_ShapePath.APPLY_ALL`에도 더한다.

### 접속점 기준 제한과 Manual (2026-11-04 5-16 결정)

접속점(수전점)에는 같은 그룹의 Auto·Manual 유닛 출력이 모두 합쳐진다. 접속점 기준 제한 모듈(역송 방지, 계약 전력, 피크저감 상한 등)은 이렇게 건다.

| 단계 | 내용 |
|---|---|
| Manual 경로 (먼저) | 정책 `g_astManualPolicy[그룹].bPccLimit` AND 모듈 등록 `g_astLimitReg[n].bManual`이면 Manual 그룹 값도 상한으로 자른다. 하나라도 FALSE면 Manual은 지령대로 |
| Auto 경로 (다음) | 항상 적용. 목표는 접속점 기준(그룹 전체 값)이라 Manual 결과까지 합쳐 제한하고, 넘치는 양은 Auto 유닛이 줄인다 (Auto 몫 = 제한 후 그룹 값 − Manual 결과) |
| Auto로 못 맞추는 경우 | Manual 몫이 상한보다 크면 Auto 몫이 음수가 되어야 한다. 고정 지령(정출력 등) Auto 유닛은 0까지만 줄이고 반대 방향(충전)으로 메우지 않는다 → 접속점이 상한 위에 남는다. 반대 방향으로 메우지 않는 이유: 방전 지령 유닛이 갑자기 충전하면 운전원 의도와 맞지 않고 배터리 사용이 늘어남 |

- **기본은 Manual도 지킨다**: `ST_ModuleReg.bManual` 기본 TRUE, `bPccLimit` 기본 TRUE. 넘으면 안 되는 상한을 Manual 운전(시운전·점검·복구 뒤 시험·계통 요청 수동 운전) 중에도 지키려고.
- **현장 시험 때만 푼다**: 그 모듈 칸의 `g_astLimitReg[n].bManual`을 HMI·XAE에서 FALSE로 (빌드 없이 다음 스캔부터, "운영 설정 바뀜" 로그). 그룹의 접속점 기준 모듈 전부를 풀려면 정책 `bPccLimit` FALSE. 푼 동안은 상한이 Manual을 막지 않으므로 운전원이 직접 지킨다.
- **끝나면 TRUE로 되돌린다**: 영속값이라 저절로 안 돌아오고 재시작 뒤에도 FALSE 그대로다. 시험 절차에 넣는다.
- **업데이트 주의**: 기본값을 바꾼 것은 처음 내려받는 현장에만 적용된다. 이미 운영 중인 현장은 저장된 값(이전 기본 FALSE)이 남으므로 업데이트 때 `bManual`을 TRUE로 한 번 쓴다.
- 확인: `MAIN.fbLimitEval.q_abPccManual[n]`, 출력 추적표(`fbOutputDiag`)의 접속점 제한 줄.


### 지령원이 받는 운전 모드

- 지령원마다 `g_astSourceReg[n]`에 정한다. 값은 `Auto` / `Manual` / `Both` / `None`이다.
  - `ePowerMode`: P·Q 지령과 P·Q 제한을 받는 운전 모드.
  - `stStateMode` (`ST_StateModeSet`): 상태 요청을 명령마다 정한다. `stPcs`(`eRun`, `eStandby`, `eStop`, `eEStop`)와 `stBat`(`eClose`, `eOpen`)으로 나뉜다. 기본은 모두 `Manual`이다. 판단은 `F_PcsCmdAllowed`·`F_BatCmdAllowed`가 한다.
  - 이 설정은 지령원 요청에만 걸린다. 보호 조치와 모드 전환은 이 설정과 관계없이 동작한다.
  - 여기에는 코어 표준 운영 명령만 둔다. 장비마다 다른 세부 명령(운전 모드 레지스터, 계통연계/독립 전환 등)은 드라이버 메서드와 사이트 모듈로 처리한다.
- `E_PcsCmd`·`E_BatCmd`에 명령을 추가하면 `ST_PcsCmdMode` 또는 `ST_BatCmdMode`에 칸, `F_PcsCmdAllowed` 또는 `F_BatCmdAllowed`에 줄을 같이 추가한다. 빠뜨리면 그 명령은 모든 모드에서 받지 않는다 (안전 쪽 기본).
- 상태 요청은 대상 유닛의 실제 운전 모드로 판단한다. 배터리 요청은 그 버스의 유닛 중 하나라도 맞는 모드면 받는다. 리셋은 모드와 관계없이 받는다.
- 모드 전환 중인 유닛은 지령원의 정지 쪽 요청(Standby·Stop·EStop)만 받고 Run은 받지 않는다.
- 지령원 출력의 상태 요청은 `stPcsReq`(`eCmd : E_PcsCmd`, `nUnit`)와 `stBatReq`(`eCmd : E_BatCmd`, `nBatGroup`)로 나뉜다. 0은 전체다.
- 사이트별 운영 방식은 이 등록값으로 정한다. 예: "Auto에서는 정지 쪽만 허용"이면 운전자 지령원의 `stPcs.eRun`·`stBat.eClose` = `Manual`, `stPcs.eStandby`·`eStop`·`eEStop`·`stBat.eOpen` = `Both`.

### 상태 요청 (후보 → 접수 검사 → 이번 스캔 요청)

```
FB_PermitEval              기동 조건 모듈 평가 (그룹마다 1회)
fbSequencer.M_Prepare      이번 스캔 조건 모으기 (배터리 투입 완료, 기동 조건, 통신·운전 상태·가용)
fbStateArb (본체)          지령원 요청 후보: 같은 스캔이면 정지 쪽. 보호 조치보다 기동 쪽 후보는 버림
fbSequencer.M_CheckRequest 접수 검사 (판단만. 타이머·재시도·명령은 안 건드림)
fbStateArb.M_CommitRequest 접수된 것만 이번 스캔 요청 (q_abNewReq, q_aeReq)
```

- 저장하는 목표 상태는 없다. 상태 명령(Run·Stop·EStop·배터리 투입·개방·고장 리셋)은 일회성이다: 접수되면 시퀀서가 한 번 보내고 장비 상태로 확인한다.
- 정지 쪽(운전 중 Standby·Stop·EStop)은 항상 받는다. 운전 중 Standby 요청은 할 일이 없다 (대기 = 출력 0이고, 출력은 지령으로 정함).
- 기동 쪽은 Run, 또는 정지 상태(통신 있고 Stop·Fault)에서 Standby(= 활성)다. 기동 쪽만 아래 순서로 검사하고, 처음 걸린 이유로 거부한다 (`fbSequencer.q_aeRefuse`).

| 순서 | 이유 (`E_SeqRefuse`) | 조건 |
|---|---|---|
| 1 | `ConfigError` | 설정 오류 |
| 2 | `WriteDisabled` | 장비 쓰기 불가 (`g_bRedWrite` 또는 `DEVICE_WRITE`가 FALSE) |
| 3 | `CommLost` | 통신 없음 |
| 4 | `StopInProgress` | 정지 진행 중 (보낼 차례·정지 램프·확인 중). 정지는 끝까지 하고, 끝난 뒤 새 요청이 필요하다 |
| 5 | `NotAvail` | 가용 아님 |
| 6 (단계별만) | `NoBattery` | 정지 상태에서 대기·Run 요청인데 배터리 미투입 |
| 7 (단계별만) | `NoPermit` | 정지 상태에서 Run 요청인데 기동 조건 미충족 |

- 순서 기동(`Sequence`)은 배터리 투입·기동 조건을 기다리는 단계라 그것으로 거부하지 않는다.
- 보호 조치(Standby·Stop·EStop)가 걸린 유닛은 그보다 기동 쪽인 요청을 접수 검사 전에 버린다 (보호가 풀려도 저절로 기동하지 않게). 버린 요청은 따로 기록하지 않는다 (보호 알람과 겹침).
- PMS 시작 때 장비에 명령을 스스로 보내지 않는다 (운전 중 장비를 멈추지 않는다).

### 시퀀스

- Run = 활성, 대기 = 활성 + 출력 0. Run은 기동 때 한 번 보내고, 활성(Run·대기)이면 지령(P·Q)만 쓴다. PMS가 Run을 다시 보내거나 대기 명령을 만들지 않는다. 0을 받은 드라이버가 장비 맵에 맞게 쓴다 (대기 주소·부호별 명령이 필요한 맵이면 그 명령, 아니면 0).
- 정지 쪽: 지령원 Stop·EStop(접수된 스캔)과 보호 Stop·EStop(걸려 있는 동안) 중 센 쪽 하나를 보낸다. 진행 중인 기동을 끊는다. 정지 램프는 `g_stSeqSet.stPcs.bStopRamp`로 켠다 (출력이 `fStopDonePct` 이하로 내려가거나 `tStopRampMax`가 지나면 Stop).
- 기동 방식은 유닛마다 정한다 (`START_MODE`). `Step`(단계별, 기본): 요청마다 한 단계 (BatClose → Run. 정지에서 Standby 요청도 Run으로 활성). `Sequence`(순서 기동): Run 하나로 배터리 투입 → 기동 조건 → Run. 보호 조치나 정지가 걸리면 진행 중인 기동을 멈춘다.
- PCS Fault는 정지로 본다. 통신 끊김·Unknown은 정지 확인이 아니다.
- 재전송:
  - 지령원 명령(운전원·EMS)은 운전 모드와 관계없이 한 번만 보낸다. 확인 시간(기동 쪽 `tCmdConfirm` 30 s, 정지 쪽 `tStopRetry` 5 s)이 지나면 실패 알람(`fbSequencer.q_aeFailCmd`)만 남긴다.
  - 보호 정지 쪽 명령만 `tStopRetry` 간격 × `nStopRetry`(3)회까지 보낸다. 넘으면 실패 알람만 남기고 멈춘다 (잠그지 않음). 그 보호가 걸려 있는 동안은 다시 보내지 않고, 보호가 약해지거나 새 요청이 오면 실패 표시를 지운다.
  - 한 번만 보낸 지령원 명령을 확인하는 중에 보호가 **같은 명령**을 원하면 재시도로 바뀐다.
  - 재시도 중 통신이 끊기면 멈추고 기다린다. 통신이 돌아오면 이어서 보내고, 기다린 동안은 횟수를 세지 않는다 (PCS·BMS 같음).
  - 장비 통신이 없을 때 생긴 보호 명령은 통신이 돌아오면 1회만 보낸다 (재시도 없음). 그 사이 보호가 풀렸어도 보낸다 (예: PCS 통신 에러 보호를 정지로 두면, 통신이 돌아올 때 Stop 1회).
- 고장 리셋은 장비에 내리는 명령일 뿐이다 (PMS 상태는 바꾸지 않음). 지령원 PCS 리셋(`stPcsReq.eCmd = Reset`, `nUnit` 0 = 대상 그룹 전체) → 그 PCS, 배터리 리셋(`stBatReq.eCmd = Reset`, `nBatGroup` 0 = 전체) → 그 배터리 그룹 BMS. 장비가 고장일 때만 1회 보낸다 (상승 에지, 보호 중이어도 보냄). 같은 스캔에 다른 명령을 보냈거나 정지·개방이 진행 중이면 그것이 끝난 뒤 보낸다. 시험 리셋(`GVL_TEST.g_bTestReset`)은 모든 PCS·BMS.

### 배터리

- 투입 시점 `g_stSeqSet.stBat.eClose`: `OnStart`(순서 기동 중인 유닛이 배터리를 원할 때, 기본) / `OnRequest`(지령원 BatClose로만). 새 기동·요청이 있어야 투입한다 (보호·고장으로 열린 뒤 저절로 다시 넣지 않음). 이미 투입돼 있으면 건너뛴다.
- 투입 완료 = 그 배터리 그룹의 쓸 수 있는 BMS가 하나 이상이고 전부 투입됨.
- 버스 충·방전 한도는 투입된 BMS가 있으면 투입된 BMS 한도만 합친다 (열린 BMS는 전류를 못 받음). 하나도 투입되지 않았으면(기동 전) 쓸 수 있는 BMS 전부를 합친다.
- 보호 개방·비상정지는 같은 버스 PCS가 모두 정지(Stop·Fault 보고)했거나 통신이 끊겼을 때 보낸다 (`q_abBusStopOk`). 비상정지도 같다. 보호 중에는 투입하지 않는다.
- 지령원 BatOpen도 같은 기준으로 받는다 (아니면 `UnitRunning` 거부). 그렇게 연 배터리는 BatClose 또는 그 버스 유닛의 새 기동 요청(Run·Standby)이 올 때까지 자동 투입하지 않는다 (`fbSequencer.q_abBatHoldOpen`).
- 정지 시 개방(`bOpenOnStop`, 기본 FALSE)은 실제 정지(통신 있고 Stop·Fault)만 본다 (`q_abBusStopped`).
- 보호 조치(개방·비상정지)만 `tOpenRetry` × `nOpenRetry`(3)회 재시도한다. 나머지(지령원 요청·자동 투입·정지 시 개방)는 한 번만 보낸다. 확인되지 않으면 실패 알람만 (`fbSequencer.q_aeBmsFailCmd`, 잠그지 않음). 같은 명령은 투입 요청·새 기동·보호 변경으로 실패가 지워질 때까지 다시 보내지 않는다.
- 배터리 요청 거부 이유는 배터리 그룹별이다 (`fbSequencer.q_aeBatRefuse[b]`: `UnitRunning`).

### 쓰기

| 대상 | 언제 쓰는가 |
|---|---|
| 상태 명령 | 새 명령일 때 한 번. 쓰기 허가가 없는 동안 생긴 명령은 보내지 않고 버린다 |
| Auto 유닛 P·Q | 쓰기 주기(`g_stSeqSet.tWritePeriod`, 200 ms)마다 |
| Manual 유닛 P·Q | 새 명령이 들어온 스캔(`fbManualGate.q_abManualNew`)에 바로 한 번, Auto→Manual 전환, 출력 가능 들어감·나옴(`fbOutputReady.q_abReady`) 때 한 번. 그 뒤에는 램프 중이거나 성형 제한에 걸렸을 때(`fbManualShape.q_abFollow`), 또는 유닛 지령이 줄었거나 0으로 묶였을 때(`fbDistributor.q_abUnitLimited`) 쓰기 주기마다, 풀린 값까지 쓰면 멈춘다 |
| PMS 시작 첫 스캔 | Manual 유닛에는 쓰지 않는다 |

- 쓰기 허가(`bWriteEnable`)는 설정 정상 AND 이중화 쓰기 허가(`GVL_IF.g_bRedWrite`: Active가 되고 `tWriteHold`가 지남, 이중화를 안 쓰면 항상 TRUE) AND `GVL_SITE.DEVICE_WRITE`다. 허가가 없으면 디스패처는 아무것도 쓰지 않고, 보조 DO는 켜는 쓰기만 막는다 (끄는 쓰기는 한다: 펄스 출력이 켜진 채 남지 않게).
- 시퀀서는 쓰기 허가가 없으면 장비 명령을 만들지 않는다. 기동 요청은 `WriteDisabled`로 거부하고, 설정 오류가 있으면 `ConfigError`를 우선한다. 정지 쪽 요청은 접수하되 명령은 생성하지 않는다.

### 명령 필터

| 설정 | 경로 | 내용 |
|---|---|---|
| `g_stCmdFilter` (`FB_Select`) | 공통 | NaN·무한대는 항상 무시. `bBadValue`면 정격 × `fBadFactor`(1.5) 초과도 무시 (그룹 값은 그룹 정격 합, 유닛별 값은 유닛 정격 기준) |
| `g_stManualCmdSet.bReverse` (`FB_GateManual`) | Manual | 실제 출력 방향과 반대인 새 명령은 버리고 앞 값을 유지한다. 실제 \|P\|가 정격의 `fDirBand`(1 %) 이하면 멈춘 것으로 본다 |
| `g_stManualCmdSet.bClearOnStop` | Manual | Run을 확인한 뒤 들어온 지령만 쓴다. Run이 아니거나 Manual로 막 들어오면 새 지령이 올 때까지 출력 0 (`q_abManualWaitNewCmd`, 분배 이유 `WaitManualCmd`). 정지 중에 넣은 지령은 버린다. PMS 시작 첫 스캔은 기준으로만 삼는다 |

- Auto 지령은 지령원이 계속 최신 값을 보내므로 버리지 않는다. 지령원이 끊기면(`tExpire`) 요청이 없어져 0이 된다.
- 배터리 한계는 명령 필터가 아니라 버스 제한 모듈(`FB_SocLimit`)이 맡는다 (아래 "설비 한도와 SOC 정책").

### 설비 한도와 SOC 정책

```
FB_EquipmentLimits (② 끝, 스캔당 한 번)
  유닛 한도  = 정격 × 보호 상한 (P·Q 따로) ∩ 유닛 제한 모듈 (유닛 실제 운전 모드에 맞는 등록만)
  버스 한도  = BMS 충·방전 한도 합 ∩ 버스 제한 모듈 (그 버스에 대상 모드 유닛이 하나라도 있으면 버스 전체)
  그룹 범위  정격 줄 = 가용 유닛 한도 합
            배터리 줄 = 버스마다 min(버스 한도, 그 버스 가용 유닛 한도 합) + 배터리 없는 유닛 한도
            Q = 가용 유닛 Q 한도 합
      → 성형(그룹 범위), 분배(유닛·버스 한도)
```

- 같은 한도를 두 곳에서 따로 계산하지 않는다. 성형과 분배는 이 결과만 쓴다.
- 0은 항상 범위 안이다 (6장 "모듈 약속"). 지령원·그룹 제한 모듈의 범위는 `F_LimitKeepZero` 한 곳에서 맞춘다 (2026-10-21 결정: 제한은 줄이기만, 강제 출력은 지령으로).
- 상·하한마다 정한 주체가 남는다: 유닛 한도는 `ePMaxBy`·`ePMinBy`(Rating / Derate / UnitRule)와 모듈 ID, 버스 한도는 모듈 ID(0 = BMS 한도). 그룹 범위는 처음 찾은 모듈, 없으면 코어 이유(`RSN_RATING`·`RSN_DERATE`·`RSN_BAT`).
- 유닛·버스 제한 모듈이 재개 중(`bResume`)이면 그룹 램프의 오르는 쪽은 `fRampResume`이다 (그룹 제한 모듈의 재개와 같음).
- 버스 하나를 여러 그룹이 같이 쓰면 그룹 범위는 그룹마다 버스 전체 한도를 본다. 실제 나눔은 분배의 버스 남은 한도가 맞춘다.

SOC 정책은 제한이다 (범위만 줄이고 장비 명령은 없다. 한도가 0이면 그 방향 출력 0 = 대기):

| 모듈 | 인터페이스 | 하는 일 | 기본 설정 |
|---|---|---|---|
| `F_SocDerate` | 함수 (상태 없음) | SOC·한계·폭 → 충전·방전 한도 비율 (0~1) | — |
| `FB_SocLimit` | `I_BatLimitRule` | 최대 SOC가 상한에 다가가면 충전 한도, 최소 SOC가 하한에 다가가면 방전 한도를 선형으로 줄이고 한계에서 막음. 막은 뒤 `fResumeMargin`만큼 돌아와야 다시 허용, 그동안 `bResume`. SOC 무효면 막음(`bBlockOnInvalid`). 설정 `ST_SocLimitSet` | 95 / 5 %, 폭 5 %, 재개 마진 2 %. 기본 등록 (`g_fbSocLimit`, 설정 `g_stModuleSet.stSocLimit`, 등록 `g_astBatRuleReg[1]`: 모든 버스, Auto·Manual) |

- 제한은 범위가 열리면 늘 다시 출력한다. "만충 뒤 SOC가 내려와도 다시 충전하지 않음"처럼 그대로 두려면 보호로 붙인다 (조건 모듈: SOC 한계, 대응: Manual 전환 또는 정지).
- 정격·배터리가 없어 한도를 걸 수 없는 유닛·버스의 대용 한도는 코어 상수 `GVL_CORE.NO_LIMIT`(1E9, `10_Core/00_Type`) 한 곳에 둔다. 열린 범위·무한대 판정(`F_RangeOpen`·`F_CmdOk`)의 1E30과는 뜻이 다르다.

### 제어 모드

```
FB_CtrlModeMgr (지령원, 접속점 그룹마다 하나: g_fbCtrlMode1 → 접속점 그룹 1, i_nUnit = 0)
  i_nModeReq ─▶ 진입 검사 ─▶ 이전 모드 M_Exit ─▶ (이전 모드가 내던 P·Q를 0으로, tZeroDelay) ─▶ 진입 검사 다시 ─▶ 새 모드 M_Enter
  켜진 모드 M_Calc(매 스캔) ─▶ 유닛 단위: astUnit[i_nUnit] / 그룹 단위(i_nUnit = 0): 그룹 fPSet
```

- 관리자 하나 안에서는 모드 하나만 켠다. 모드 사이 경합(등급)은 없다. 관리자는 지령원 등록(등급 2, `ePowerMode` Auto)을 따라 **Auto 유닛에만** 들어간다. 상태 요청(Run 등)과 P·Q 제한은 내지 않는다.
- 요청(`i_nModeReq`)이 바뀌면 한 번 처리한다. 진입 검사에 걸리면 지금 모드를 그대로 두고 `q_eRefuse = NotReady`, 이유 `q_nRefuseReason`. 없는 칸이면 `BadSlot`. 처리한 값은 다시 처리하지 않는다 (같은 값 재시도는 `i_bRetry`). 시작한 전환은 끝까지 간다 (전환 중 온 요청은 끝난 뒤).
- 켜진 모드가 스스로 끝내면(`bExitReq`, 예: 정출력 시간 끝) 대체 모드(`nFallback`)로, 그 모드 진입 검사에 걸리거나 0이면 모드 없음. 모드 없음이면 관리자는 지령을 내지 않는다 (등급이 낮은 다른 지령원 값이 들어옴).
- 전환 중 출력 0은 이전 모드가 내던 쪽(P·Q)만 낸다 (비우면 그 사이 다른 지령원 값이 들어오므로). `bZeroOnSwitch = FALSE`면 바로 진입하고 출력 변화는 그룹 램프가 맡는다.
- **현장 구성 (2026-11-09, 14번 탭 16번)**: 접속점 그룹마다 관리자 하나(`g_fbCtrlMode1`, `i_nUnit = 0`)와 모드 한 세트. 접속점 그룹 안의 Auto 유닛은 모두 같은 모드로 돌고, 모드는 접속점 그룹 값(접속점·정격·SOC)을 보고 접속점 그룹 지령을 내며 분배가 Auto 유닛에 나눈다. Manual 유닛은 제어 모드를 받지 않는다. 피크 목표·정출력 값은 접속점 그룹 전체 값이다 (PCS 몫으로 나누지 않음 → 속도 계수가 접속점 그룹 1대 운전과 같음). 스탠다드 스케줄(5)만 EMS가 PCS마다 보낸 P를 유닛별 값(`ST_CtrlModeOut.bUnitOut`·`astUnit`)으로 낸다. 예전 PCS별 관리자(`g_fbCtrlMode2`, 지령원 칸 3)와 PCS2 설정 칸은 없앴다 (`SOURCE_COUNT` 4 → 3, EMS는 칸 3).
- **PCS마다 따로 (유닛 단위, 코어 기능)**: 관리자 `i_nUnit = n`은 유닛 n의 유닛별 값만 낸다. PCS마다 관리자·모드 인스턴스·설정(시간표 등)이 따로라 같은 시각에 PCS1은 스케줄, PCS2는 정출력처럼 다르게 돌 수 있다. 유닛별 값이라 그룹 몫을 나누지 않고, 램프(그룹 램프율 × 유닛 정격 비율)·유닛 한도·접속점 그룹의 접속점 제한(유닛 값 합에 걸고 넘치면 비율로 축소)은 코어가 그대로 건다.
- **그룹 하나로 (그룹 단위)**: 관리자 `i_nUnit = 0`은 그룹 지령(접속점 기준)을 내고 분배가 유닛에 나눈다. 그룹이 2개 이상이면 그룹마다 관리자, 등록 `nGroup`.
- **접속점 값 (유닛 단위)**: 모드는 부하를 "출력 − 접속점 P"로 계산한다. 같은 접속점을 여러 PCS가 쓰면 관리자 설정 `ePccShare`로 이 PCS가 볼 부하를 정한다.

| `ePccShare` | 모드가 보는 부하 | 쓰는 때 |
|---|---|---|
| `ByRating` (기본) | 그룹 부하(그룹 출력 합 − 접속점 P) × 그룹 가용 한도 중 이 유닛 몫 | 피크컷·역송 방지가 겹치지 않음. 목표 피크도 PCS 몫으로 넣는다 (예: 두 대 같은 정격, 목표 1000 → 각 500) |
| `Actual` | 접속점 실측 그대로 = 그룹 부하 − 다른 유닛 출력 | 다른 PCS 출력을 실측으로 반영. 같은 피크를 두 PCS가 따로 보고 반응하므로 겹칠 수 있음 |
| `Fixed` | 그룹 부하 × `fPccRatio` (0~1) | 정한 비율 |

  - 접속점이 PCS마다 따로인 현장은 PCS마다 제어 그룹을 따로 둔다 (`GROUP_CFG`). 그룹에 유닛이 하나이고 가용이면 ByRating과 Actual의 부하가 같고, Fixed는 `fPccRatio = 1`일 때만 같다.
  - 예: 부하 1000, PCS1 +200, PCS2 시간표 +600. `ByRating`(몫 0.5) → PCS2 부하 500 → 역송 방지 (500 − 10) × 0.95 = 465.5. `Actual` → PCS2 부하 1000 − 200 = 800 → 상한 750.5 → 600 그대로.
- 유닛 요약(`FB_UnitView`): 버스 한도·용량은 같은 버스 가용 유닛 정격 비율로 나눈다 (몫 합 = 버스 값). 그래서 같은 버스의 PCS 두 대가 자동 충전·잔량 방전을 쓰면 각자 용량의 반을 보고 계산한다. 가용이 아닌 유닛은 가용 유닛 0이라 모드 진입이 거부된다.
- **같은 배터리 버스의 PCS는 같은 시각에 충전과 방전을 섞을 수 없다.** 시간표를 따로 넣어도 분배가 버스 순 방향과 반대인 유닛을 0으로 둔다 (`E_UnitLimitBy.BatDir`, 7장 "운전과 분배"). 섞어야 하면 PCS마다 배터리 그룹을 따로 둔다.
- **스탠다드 스케줄 (`FB_CmStdSchedule`, 칸 5)**: 시간표를 PMS에 저장하지 않고 EMS가 시각마다 지령을 보낸다. 예) 09~10시 방전 100 kW → EMS가 9시에 방전 100 kW(CommandType 5), 10시에 정지(CommandType 2)를 보냄.
  - Manual 지령과 같은 구조다: 새 지령(번호가 바뀜)만 받고, 받은 P는 다음 지령까지 유지한다. 정지 쪽 상태 명령(Stop·Standby·EStop)이 오면 받아 둔 P를 버리고 새 P 지령이 올 때까지 0 (Run만 다시 와도 옛 값은 되살아나지 않음).
  - 모드가 켜지기 전에 온 지령은 쓰지 않는다 (켜진 뒤 새 지령부터). 켜져 있는 동안 P는 늘 낸다 (지령이 없으면 0 → 등급이 낮은 지령원 값이 새지 않게).
  - EMS가 끊기면(EMS 지령원 무효, `tExpire` 60 s) 받아 둔 P를 버리고 0. 살아 있음은 지난 스캔 값이라, 새 지령이 온 스캔은 살아 있는 것으로 본다 (조용하던 EMS의 첫 지령이 버려지지 않게). 돌아와도 옛 지령은 되살아나지 않는다. NaN·무한대 P는 버리고 앞 값 유지 (`q_nBadCount`). 정격 대비 상한은 코어 명령 필터.
  - 상태 명령(Run·Stop 등) 자체는 EMS 지령원(`FB_SrcEms`)이 장비에 낸다. 이 모드는 P만 내고, 역송 방지·방전 허용 시간은 걸지 않는다 (EMS 몫. 코어 접속점 제한은 그대로).
  - 연결: MAIN `A_PassEmsCmd`가 EMS 명령 칸(`stEmsCopy`)의 PCS N 값을 `g_fbCmStdSchedN.i_stCmd`에 넘긴다 (PCS 한 대 = 유닛 하나. 한 PCS에 유닛이 여럿이면 유닛 몫으로 나눠 넘김). 설정 없음.
  - 확인: `g_fbCmStdSchedN.q_nState`(0 = 받은 P를 냄, 2 = EMS 끊김, 3 = 새 P 지령 대기), `q_bHold`·`q_fPCmd`(받아 둔 P), `q_nCmdCount`.
- 시각은 `fbClock`(PLC Windows 시간대의 현지 시각)이다. 시각이 무효면 요금제 모드는 진입을 거부하고 0을 낸다. 실경로 시험도 PLC의 현지 시각과 시간대를 확인한 뒤 그 기준으로 시간표를 넣는다.
- 요금제 모드 공통 (`FB_CmTariffBase`): 휴일 → 0, 방전(+)은 방전 허용 시간(기본 월~토 08~22시, 피크컷·통합의 잔량 방전 시간도 포함: 2026-11-09 14번 탭 14번)과 역송 방지(부하 − 여유 10 kW를 넘으면 가능량 × 0.95)로 줄인다. 충전(−)은 줄이지 않는다. 접속점 계측 무효면 방전 0. 시간표 칸은 자정을 넘지 않는다 (시작 > 끝은 설정 오류, 두 칸으로 나눔). 맞는 칸이 여럿이면 번호가 큰 칸. 지금 상태·이유는 모드의 `q_nState`(`GVL_CM_RSN`).
- 피크 제어 (`FB_CmPeakCalc`): 시계 기준 15분 구간, 1초마다 수전·부하 평균, 분이 바뀔 때 제어량 갱신(목표를 넘은 수전 누적 ÷ 남은 분 × 속도 × 지난 분/15), 구간 경계에서 0. 모드가 켜질 때 처음부터 센다.
- 설정 점검: 관리자 `M_CheckSet`이 자기 설정(대체 모드 칸, `i_nUnit` 범위, 유닛 요약 연결, `fPccRatio`)과 등록한 모드마다 `M_CheckSet`(HHMM 범위, 시작 > 끝, SOC·비율 범위 등)을 부른다. 잘못된 값은 `FB_SettingGuard`가 받지 않고 되돌린다.

### 운전 모드 전환 (`MODE_SET`)

```
요청(GVL_PERSIST.g_aeOpMode) ─▶ FB_ModeSwitch ─▶ 실제(fbModeSwitch.q_aeOpMode)

→ Manual (기본 Standby) : 출력 0 → 400 ms → 운전 중이고 리모트면 대기(출력 0) 확인 → 전환
→ Auto   (기본 Zero)    : 출력 0 → 400 ms → 전환
```

- 단계는 방향별로 `Direct` / `Zero` / `Standby` 중에서 고른다.
- 전환 중에는 출력이 0이고, 지령원의 기동 쪽 요청(Run)은 받지 않는다. 정지 쪽 요청은 받는다.
- 대기 확인: 대기 명령은 따로 없다 (대기 = 출력 0). 장비가 대기·정지를 보고하거나, 실측 |P|·|Q|가 정격 × `g_stSeqSet.stPcs.fStopDonePct`(1 %) 이하면 끝. 로컬이 되면 기다리지 않는다.
- 시작한 전환은 요청이 바뀌어도 끝까지 진행한다 (목표 고정). 끝난 뒤 요청이 다르면 다시 전환한다. 예외: 보호 `ToManual`이 걸리면 Auto로 가던 전환의 목표를 Manual로 바꾼다 (출력 0 유지, Auto로 한 스캔도 넘어가지 않게).
- PMS 시작 때(전원·다운로드 뒤 첫 스캔)는 기본으로 모든 유닛을 Manual로 시작한다. 요청 모드(`g_aeOpMode`)도 Manual로 써서 화면·저장값과 맞춘다. 현장 전체 정전 뒤처럼 PCS·배터리를 다시 투입해야 하는 경우가 대부분이라, 장비 투입과 Auto 전환은 운전원이 확인하고 한다. `MODE_SET.bKeepOnRestart = TRUE`면 저장된 요청 모드를 그대로 쓴다 (전환 단계 없음).
- 시작 때 Run·배터리 투입 명령을 스스로 보내지 않는다. 장비 투입은 운전원 요청(또는 순서 기동 Run)이 있어야 한다.
- 보호가 `eModeAction = ToManual`로 요청하면 요청 모드가 Manual로 바뀌고 (7장 "보호"), 이후는 위 전환 단계를 그대로 따른다.

### 보호

- 보호 칸 = 조건 모듈(`I_ProtectionRule`, `g_aipRule`) + 대응 모듈(`I_ProtResponse`, `g_aipResp`) + 설정(`g_astRuleCfg`), 셋 다 같은 칸 번호. 기본 5종 조건(PCS 고장·통신, BMS 고장·통신, 접속점 계측 무효)과 표준 대응(`g_fbProtRespStd`)이 1~5칸에 기본 등록돼 있고, 현장 보호는 6칸부터 붙인다.
- 보호를 빼는 방법: 조건 칸을 비우고 재빌드(그 현장에서 안 씀), 또는 `g_astRuleCfg[칸].stProt`의 `bAuto`·`bManual`을 둘 다 FALSE(운전 중 끔). 어느 쪽이든 알람·조치가 없어진다. 대응 칸만 비우면 알람만 남는다.
- 규칙 칸 하나에 범위(`eScope`)와 대상 번호(`nTarget`)를 준다. `nTarget` 0이면 범위 안 대상을 모두 하나씩 판단한다 (칸 하나로 "모든 PCS 고장"). 기록도 대상마다 남는다.
- 지연은 없다: 조건이 TRUE면 바로 걸린다. 통신 에러·고장은 장비 쪽이 판정해 올린다 (통신 에러 판정 시간 = 드라이버 통신 타임아웃 `i_tCommTimeout`). 현장 접점·아날로그의 떨림 방지는 조건 모듈 설정에 둔다 (`FB_RuleValue` 히스테리시스 등).
- 보호는 조건이 있는 동안만 걸린다 (래치 없음). 보호를 풀려고 리셋할 필요는 없다. 그대로 두려면 정지로 두기 = PCS 정지, 대기로 두기 = PCS 대기 + Manual 전환. "풀린 뒤 대기 유지" 설정은 두지 않는다: Auto 상위 지령과 PMS가 묶은 출력 0이 부딪히고 상위는 이유를 모른다. Manual 전환은 상위 지령을 멈추고 모드가 보고되므로 충돌이 없다.
- 대응이 낼 수 있는 요청(`ST_ProtReq`): PCS(None / Standby = 출력 0, 명령 없음 / Stop / EStop), 배터리(None / Stop / EStop), Manual 전환, P·Q 상한(정격 대비 %). 코어가 장비마다 합친다: PCS·배터리는 센 쪽, 상한은 낮은 쪽, Manual 전환은 하나라도. 건 모듈은 조건 모듈 ID다.
- 조치가 장비에 가는 길: PCS Standby는 가용 아님으로 출력 0 (명령 없음, 풀리면 다시 출력). Stop·EStop은 시퀀서가 명령을 보내고 장비가 보고하는 상태로 확인한다 (재시도·통신 복구 1회는 7장 "시퀀스"). 정지로 멈춘 장비는 조건이 풀려도 다시 돌지 않는다 (다시 운전하려면 Run 요청). 배터리 조치는 버스 PCS 정지 확인 뒤에 나간다 (7장 "배터리"). P·Q 상한은 설비 한도에서 줄인다.
- 표준 대응 설정 (`ST_ProtCfg`): `bAuto`·`bManual`(그 운전 모드 유닛에 조치를 걸지. 둘 중 하나라도 TRUE면 감지·발생 표시·로그, 둘 다 FALSE면 이 보호를 안 씀), `ePcsAction`(None/Standby/Stop/EStop), `eBatAction`(None/Stop/EStop), `eModeAction`(None/ToManual), `fPPct`·`fQPct`(P·Q 상한 %, 100 = 없음).
  - 고를 수 있는 조합: PCS None·Standby·Stop·EStop × 배터리 None·Stop·EStop. 배터리 조치는 PCS가 Stop·EStop일 때만 (아니면 설정 값 받지 않음 → 마지막 정상값). Manual 전환은 모든 조합에서 선택.
  - BMS 조치는 같은 버스에 조치를 받는 유닛이 있을 때 건다 (버스에 유닛이 없으면 그대로 건다).
- 현장 대응이 다르면(단계 대응, 다른 장비까지 조치 등) 표준 대응 `FB_ProtRespStd`를 복사해 이름을 바꾸고 그 칸 `g_aipResp`에 대신 등록한다. 대응은 장비 상태를 읽기 전용으로 보고(대상 밖 장비 포함), 조건이 이어진 시간(`i_tActive`)을 받는다.
- 리셋은 장비 고장 리셋 명령일 뿐이다 (7장 "시퀀스"). 장비가 고장을 붙잡고 있으면 리셋으로 장비 고장이 풀리고, 그러면 고장 보호도 조건이 없어져 풀린다.
- BMS 고장의 기본 조치는 BMS 정지 + 같은 버스 유닛 정지다. 남은 BMS로 계속 돌리려면 PCS 조치와 배터리 조치를 둘 다 None으로 바꾼다 (배터리 조치는 PCS 정지와만).
- **모드 조치(`eModeAction`)**: `ToManual`이면 그 보호가 걸려 있는 동안, 범위 안 유닛(`bAuto`가 켜진 항목)의 **요청 모드**(`g_aeOpMode`)를 `FB_ModeSwitch`가 Manual로 바꾼다.
  - 보호가 풀려도 Manual을 유지한다. Auto로 돌아가려면 운전원이 다시 요청한다 (재기동 안전). 걸려 있는 동안 Auto를 다시 요청해도 Manual로 되돌린다.
  - 전환 자체는 `MODE_SET` 단계(출력 0 → 지연 → 대기 확인)를 그대로 따른다. 보호 결과는 지난 스캔 값이라 반영이 1스캔 늦다.
  - `ePcsAction = Standby` + `ToManual`이 "Manual 전환 + 대기", Fault 보호 + `ToManual`이 "고장이면 Manual 전환"에 해당한다.
  - `bAuto`가 꺼진 항목에는 효과가 없다. 무해한 설정이라 설정 오류(장비 제어 중지)로는 막지 않는다.
  - 기록: 보호가 걸릴 때마다 `PmsSeq.ModeForced` 경고 1건 (보호 이름, 모듈 ID). 걸려 있는 동안 Auto 재요청을 되돌려도 다시 남기지 않는다.
- 보호 알람의 심각도·조치 문구는 칸 설정(`stProt`)의 조치로 정한다. 표준 대응이 아닌 대응을 붙인 칸은 실제 조치와 다를 수 있다 (10장).

### 장비 읽기

- 읽기 주기 `i_tReadPeriod`는 기본 200 ms이고, 장비마다 선언에서 바꾼다.
- 통신 이상 판정 `i_tCommTimeout`은 기본 10 s다. 읽기 주기보다 길어야 한다.
- 여러 영역을 읽는 장비는 드라이버 안에서 돌려 읽거나, 영역마다 `FB_ReadTimer`를 둔다.

### 진단과 로그

- `FB_OutputDiag`는 그룹마다 경로·P/Q별로 4개를 낸다: `q_astAutoPDiag`, `q_astManualPDiag`, `q_astAutoQDiag`, `q_astManualQDiag` (`ST_OutputDiag`).
  - `eStage`(단계), `nModule`(모듈, 0 = 코어), `nReason`(모듈 이유 또는 코어 이유 `GVL_REASON`), `eUnitBy`(UnitLimit 단계일 때 유닛 한도 이유, 그 경로 유닛 중 처음 찾은 것), `eGroupState`, `fReq`, `fOut`(그 경로 유닛에 쓴 값 합).
  - 판단 순서: 요청 없음(`NoRequest`) → 이 경로 유닛 없음(`NoUnit`, 정상) → 그룹 운전 아님(`NotRunning`) → 막은 단계.
  - P(두 경로): 막은 단계 = **최종 값을 정한 곳**, 곧 추적표에서 값을 바꾼 마지막 줄이다 (보정 제외). Manual P는 추적표 값이 안 바뀌었어도 Manual 유닛이 유닛 한도에 걸렸으면 UnitLimit으로 표시한다.
  - Q(두 경로): 설비 제한 → 유닛 한도·분배 순으로 처음 막은 곳 (Q는 램프 없음).
  - 그룹 지령 없이 유닛 지령만 있으면 P 요청값은 유닛 지령 합이다. `ST_ShapeDiag`의 `bPGroupCmd`·`bPUnitCmd`로 어느 쪽인지 본다.
  - 이벤트 로그(출력 제한·해소)와 보고는 Auto P만 쓴다 (`iq_astAutoPDiag`).
- P 추적표 `fbOutputDiag.q_astAutoPTrace[그룹]`, `q_astManualPTrace[그룹]` (`ST_PTrace`): 지나간 단계마다 값·자름·모듈·이유를 남긴다 (예: 1000 → 700 → 500 → 100 → 0).
  - 요청 → 램프 → 접속점 제한(지령원 P 제한, 접속점 기준 모듈 하나씩) → 보정 → 설비 제한(가용 정격, 배터리 한도, 설비 기준 모듈 하나씩) → 유닛 한도·분배 후 합 → 쓴 값 합.
  - 같은 단계 안의 모듈은 등록 순서대로 하나씩 자른 값이고, 다음 단계는 실제 계산 결과에서 이어간다 (보기용, 출력 계산에는 안 쓴다). 꺼진 단계(Manual 정책)는 줄이 없다.
  - 마지막 두 줄: Auto는 그룹 전체 유닛(Auto+Manual) 합(접속점 기준), Manual은 Manual 유닛 합.
- 유닛 한도 이유(`E_UnitLimitBy`): `NotAvail`, `NotRun`(활성 아님·정지 중), `Rating`, `Derate`(보호 P·Q 상한), `Scaled`, `BatDir`, `BatLimit`(버스 한도·버스 제한 모듈), `SLimit`, `ModeSwitch`, `WaitManualCmd`, `UnitRule`(유닛 제한 모듈). 유닛별 결과는 `fbDistributor.q_astUnitDiag`이고, P를 줄인 모듈은 `nPModule`(0 = 코어)이다.
- 메서드나 임시 변수 안에 있어 온라인으로 안 보이는 중간값은 진단 칸으로 남긴다 (제한 범위, 램프율, 분배 경로별 값, 배터리 버스, 유닛 한도·무게·배터리 후 값).
- 진단 값은 제어에 쓰지 않는다. 제어에 필요한 값은 따로 낸다 (`fbDistributor.q_abUnitLimited`, `fbAutoShape/fbManualShape.q_abGrpPCmd·q_abGrpQCmd`).
- 로그는 꼭 필요한 것만 남긴다 (이벤트 로거 보관 한도 때문이다). 출력 제한은 이유가 바뀌고 `LOG_SET.tHold`(10 s) 유지될 때만 남긴다.

| 이벤트 클래스 | 기본으로 기록 | 설정으로 켬 |
|---|---|---|
| PmsProt | 보호 동작(1), BMS 명령 실패(2: 이벤트 이름은 `BatLocked`, 문구 "BMS {0} - {1}" (2026-11-09 14번 탭 8번: 예전 "배터리 잠금: BMS {0} - {1}") 보호 개방·비상정지 재시도를 다 쓰거나 투입 확인 안 됨, 알람만) | — |
| PmsOutput | 출력 제한(1)·해소(2), 지령원 끊김(4: 등록 `bLostAlarm` TRUE인 지령원만. EMS는 FALSE), 가상신호(5, `TEST_MODE`), 지령값 이상(6: 디스패처가 NaN·무한대를 0으로 바꾼 유닛, 생길 때 1건), MQTT 브로커 끊김(7: 알람, 7장 "브로커 끊김 감시") | 유닛 축소(3, `bUnitLimited`) |
| PmsSeq | 상태 요청(1), 명령 실패(4: 확인되지 않은 PCS 명령, 알람만), 지령 무시(6: 이상값·방향 반대·운전 모드 설정, `bCmdRefused`. 보호가 요청을 버린 것은 보호 알람과 겹쳐 기록하지 않음), 배터리 요청 거부(2, `bBatRefused`), 기동 조건 대기(7: 그룹·막은 모듈·이유, 대기가 시작되거나 막은 모듈·이유가 바뀔 때 1건), 제어 모드 바뀜(9: 관리자·이전·새 모드, 전환이 끝난 뒤 1건), 제어 모드 요청 거부(10: 요청 칸·이유), 제어 모드 스스로 끝남(11: 모드·이유·넘어간 모드) (9~11은 2026-11-09 14번 탭 7번) | 유닛 요청 거부(2, `bReqRefused`), 시퀀스 변화(3, `bSeqChange`), 모드 변경(5, `bModeChange`) |
| PmsSystem | PMS 시작(1), 설정 오류(2), 설정 오류로 장비 제어 중지(3, 알람), 운영 설정 바뀜(4: 바뀐 설정 이름, 저장할 때 1건), 설정 저장 실패(5, 알람: 오류 번호), 잘못된 설정 값 받지 않음(6: 설정·이유, 마지막 정상값 유지), 장비 쓰기 실패(7, 알람: 유닛·BMS마다, 드라이버가 `bWriteFail`을 채울 때), 이중화(8~17, 아래), 로거 시험(18, `PRG_TestLogger`만 씀: 메시지·알람 둘 다) | — |

- 이중화 이벤트(`PmsSystem` 8~17, `A_LogRedundancy`). 이중화를 쓸 때만 나온다:
  - 메시지: 시작 Active(`RedStart`, 8), 절체(`RedFailover`, 9, Warning), 복귀(`RedFailback`, 10), Active 양보(`RedYield`, 11), PLC 멈췄다 이어짐(`RedResume`, 12, Warning), 동기화로 설정 맞춤(`RedSyncApplied`, 17: 건수)
  - 알람(발생·해소): 상대 PLC 끊김(`RedPeerLost`, 13, `Lost`일 때), 통신 한쪽 고장(`RedLinkFault`, 14: EAP 또는 접점), 동기화 값 못 받음(`RedSyncLost`, 15: EAP는 정상인데 5 s 넘게 안 옴), 이중화 설정 오류(`RedFailed`, 16, Error: 역할 같음 또는 Net ID 불일치)
  - Active가 바뀐 이유는 횟수가 바뀐 다음 스캔에 읽는다 (횟수와 이유가 다른 태스크에서 따로 쓰임). 멈췄다 이어진 직후 0.5 s는 절체 기록을 건너뛴다 (상태가 `Init`으로 돌아가며 횟수가 오르지만 이유는 옛 값이기 때문)
- 인자는 전부 WSTRING이고 최대 6개다.
- **로그 문구 길이**: 인자를 채운 로그 문구는 **95자 이내**로 한다. 로그는 MQTT 로거 토픽(`pms_log`)으로도 나가는데 Contents가 `WSTRING(100)`이고, 알람 해소는 앞에 `[해소] `(5자)를 붙이기 때문이다. 넘으면 99자 + `…`로 잘려 나간다 (`fbLogger.q_nTruncated`가 늘면 규칙 위반).
  - 이벤트 문구 틀(`PmsStd.tsproj` 이벤트 클래스)의 고정 글자는 50자 이내를 권한다.
  - 인자에 들어가는 문구: 모듈 이름 `i_sName` 15자 이내, 모듈 `M_ReasonText` 20자 이내, `FB_EventLog`가 직접 만드는 설명 인자 35자 이내, `F_Txt…` 문구 15자 이내.
  - 문구 틀이나 인자를 바꾸면 "고정 글자 + 인자마다 최대 길이"가 95자 이내인지 계산한다. 2026-10-28 기준 인자를 한도까지 채운 가장 긴 것은 `PmsSystem.SettingRejected` 93자, `PmsOutput.OutputLimited` 91자, `PmsSeq.CmdRefused` 89자다. `PRG_TestLogger`가 시작할 때 등록된 이벤트를 모두 찾아 문구 틀의 고정 글자와 인자 하나당 남는 글자를 보여 준다 (`astReg[n]`, 고정 50자 초과 또는 인자 하나당 10자 미만이면 `bCheck`). 그 남는 글자 안에 실제 인자(위 한도)가 드는지는 사람이 본다. 이벤트를 추가해도 시험 프로그램은 고치지 않는다.
- 재기동 기록: `FB_AliveRecorder`가 마지막 동작 시각을 1 s마다 갱신하고, 시작할 때 직전 시각과 꺼져 있던 시간을 `FB_EventLog`에 알려 남긴다. 시각은 글자·숫자(`GVL_PERSIST.g_sLastAlive`·`g_nLastAliveLocal`) 모두 `FB_LocalSystemTime`의 현지 시각 하나에서 얻는다 (`F_GetSystemTime`을 쓰면 재기동 직후 꺼져 있던 시간이 0으로 나온 일이 있었음). 지금 시각이 직전 동작보다 앞서면 "알 수 없음".

### PERSISTENT 저장

- PERSISTENT는 정상 종료 때만 저장되므로 `FB_PersistSave`(`10_Core/02_Util`)가 직접 저장한다.
- 운영 설정이 바뀌면 1 s 동안 더 안 바뀔 때 바로 저장하고, 그 외에는 `LOG_SET.tPersistSave`(60 s)마다 저장한다 (재기동 기록 시각 포함).
- 바뀜은 지난 스캔 사본과 메모리 비교(`MEMCMP`)로 찾는다. `GVL_PERSIST`에 코어 설정을 추가하면 `FB_PersistSave`의 입출력·사본·`A_CopyShadow`·비교와 MAIN 호출에도 추가한다. 빠뜨리면 주기 저장만 된다. 지금 연결된 것: `g_aeOpMode`, `g_astGroupSet`, `g_astManualPolicy`, `g_astSourceReg`, `g_astLimitReg`, `g_astPermitReg`, `g_astCompReg`, `g_astRuleCfg`, `g_astOutputCfg`, `g_astUnitRuleReg`, `g_astBatRuleReg`, `g_stCmdFilter`, `g_stManualCmdSet`, `g_stSeqSet`. 현장 모듈 설정 묶음 `g_stModuleSet`은 주소·크기(`i_pModuleSet`, `i_nModuleSetSize`)로 받아 통째로 비교하므로, 현장에서 칸을 늘려도 `FB_PersistSave`는 고치지 않는다.
- 바뀐 설정을 저장할 때 "운영 설정 바뀜: 이름, 이름" 이벤트를 1건 남긴다 (마지막 저장 뒤 바뀐 것 모두, 길면 잘림). 연속 입력은 1 s 잠잠해진 뒤 한 번에 기록한다.
- 저장이 실패하면 "설정 저장 실패" 알람이 뜨고, 다음 저장이 성공하면 풀린다. 실패한 동안 재기동하면 마지막으로 저장된 값으로 돌아간다.

### 태스크

```
PlcTask        (10 ms, 우선순위 20)              MAIN             제어 전부
MqttTask       (PlcTask보다 낮게, 우선순위 30)    PRG_Mqtt         보고·명령
FastTask       (선택, 높게, 우선순위 2)           PRG_Fast         빠른 계측
RedundancyTask (선택, 가장 높게, 우선순위 1)      PRG_Redundancy   이중화 (활성 g_bRedActive / 쓰기 허가 g_bRedWrite, 보조 DO 끄기 예외는 7장)
        └──────────── 주고받기는 GVL_IF만 ────────────┘
```

- 우선순위 숫자는 작을수록 높다 (`PmsStd.tsproj` Tasks와 각 `.TcTTO`에 같은 값). 2026-11-03까지는 MqttTask 1·RedundancyTask 3으로 위 표와 반대였음 (통합시험 결함 3).

- 제어 흐름은 PlcTask의 MAIN에만 둔다.
- `GVL_IF`의 칸마다 쓰는 태스크는 하나다. 쓰는 쪽은 쓰기 전에 번호(`…Seq`)를 1 올리고(홀수 = 쓰는 중) 다 쓴 뒤 다시 1 올린다(짝수). 읽는 쪽은 번호(짝수일 때만) → 내용 복사 → 번호를 다시 읽어 같을 때만 쓴다. 우선순위가 높은 태스크가 쓰는 중에 끼어들어도 읽는 쪽이 반만 바뀐 값을 거른다.
- 태스크 전용 FB 인스턴스는 그 태스크 PRG 안에 선언한다. I/O가 있는 인스턴스를 GVL에 두면 `{attribute 'TcContextName' := '태스크 이름'}`을 붙인다.
- 이중화를 안 쓰면(`RED_SET.bUse = FALSE`) 역할은 Primary, `g_bRedActive`와 `g_bRedWrite`는 TRUE다.
- 이중화를 쓰면 두 PLC에 **같은 프로그램**을 넣는다. `PRG_Redundancy`가 시작할 때 이 PLC의 AMS Net ID를 읽어(`PRG_Redundancy.sMyNetId`) `RED_SET`의 Primary·Secondary와 비교해 역할(`GVL_IF.g_eRedRole`)을 정한다. PLC마다 따로 빌드·설정하지 않는다.
  - Net ID를 읽는 동안(`Reading`)은 쓰기 허가가 FALSE다 (보조 DO 끄기 예외는 7장). 어느 쪽과도 맞지 않거나(`NoMatch`) 두 Net ID가 비었거나 같으면 설정 오류다.
  - 누가 Active인지는 역할이 아니라 상태 기계가 정한다 (아래 "이중화").

### 이중화

두 PLC(Primary·Secondary)에 **같은 프로그램**을 넣는다. 평소 Primary가 Active(장비에 씀), Secondary는 Standby(읽기·계산만, PCS·BMS 쓰기와 보조 DO 켜기 금지, DO 끄기는 허용)다. 이중화를 안 쓰면(`RED_SET.bUse = FALSE`, 기본) 아래 전부 꺼지고 항상 Active다.

**구성**

| 통로 | 내용 |
|---|---|
| EAP 하트비트 (`ST_RedHeartbeat`) | 매 스캔(10 ms): 카운터, 역할, Active, 상태. 상대는 카운터가 바뀌는지로 살아 있음을 본다 |
| EAP 동기화 (`ST_RedSyncCfg`) | 매 스캔: 운영 설정과 제어 모드 요청 (아래 "설정 동기화") |
| 접점 (EL2004 → 상대 EL1004) | 채널 1 = alive(100 ms마다 반전), 채널 2 = Active 레벨. 배선은 두 PLC가 서로 마주 본다 |
| 장비 | 각 PLC가 직접 연결해 읽는다. PCS·BMS 쓰기는 `bWriteEnable` 허가 때만 (보조 DO 끄기 예외는 7장) |
| MQTT | 두 PLC 모두 접속·구독, 발행은 Active만 (`PRG_Mqtt`가 `g_bRedActive`를 `FB_MqttMain.i_bPublish`로 넘김). ClientId 뒤에 역할(`-P`/`-S`)을 붙인다: 같은 사이트 ClientId로 둘이 접속하면 브로커가 서로 끊기 때문 |

**상대 PLC 판정** (`FB_RedPeerMonitor`, `GVL_IF.g_ePeerState`)

| EAP | 접점 | 판정 | 처리 |
|---|---|---|---|
| 정상 | 정상 | `Alive` | 정상 |
| 끊김 | 정상 | `EapLost` | 통신 고장. 절체 안 함 |
| 정상 | 끊김 | `WireLost` | 배선 고장. 절체 안 함 |
| 끊김 | 끊김 | `Lost` | 상대 정지로 봄 = 절체 조건 |
| 미확인 | 미확인 | `Unknown` | 시작 후 `tStartGrace`(3 s) 동안 대기 |

- EAP 정상: 카운터가 `tEapTimeout`(500 ms) 안에 바뀌고, Quality(= 마지막 갱신 뒤 지난 시간, 100 µs 단위)가 그 시간 이하. 접점 정상: 상대 alive 입력이 `tWireTimeout`(500 ms) 안에 바뀜.
- 한쪽만 끊겼을 때 절체하지 않는 이유: 네트워크나 배선만 끊긴 경우 양쪽이 Active가 될 수 있다.
- 받은 역할이 내 역할과 같으면 내 프레임이 되돌아온 것(루프)으로 보고 무시한다. 같은 역할 프레임이 접점도 살아 있는 채 계속 오면 역할 충돌(`g_bRedRoleConflict`).

**상태 기계** (`FB_RedState`, `GVL_IF.g_eRedState`)

| 상태 | 뜻 |
|---|---|
| `Init` | 역할을 읽는 중이거나 상대 상태를 아직 모름. 쓰지 않음 |
| `Standby` | 대기: 계산만, 쓰지 않음 |
| `Active` | 활성 상태 (`g_bRedActive`). 쓰기 대기 뒤 `g_bRedWrite`가 TRUE가 돼야 장비 쓰기 허가 |
| `Handover` | Primary가 복귀하면서 Secondary가 Active를 내려놓기를 기다림. 아직 쓰지 않음 |
| `Failed` | 설정 오류(Net ID 불일치·두 PLC 역할 같음). 둘 다 쓰지 않음 |

- Primary: 시작하면 상대 상태를 본 뒤, 상대가 Active가 아니면 Active, 이미 Active이면 뺏지 않고 Standby. Secondary는 Standby로 시작한다.
- **절체**: Secondary는 상대가 `Lost`이고 `tTakeoverDelay`(500 ms)가 지나면 Active (`g_eRedSwitch = PeerLost`). 실측: Primary 정지 후 약 1 s.
- **복귀**: Primary는 Standby에서 상대(Secondary)가 `Alive`·Active이고 동기화 완료가 `tFailbackStable`(10 s) 이어지면 `Handover`로 간다. Secondary가 `Handover`를 보고 Active를 내려놓으면(`Yield`) Primary가 Active가 된다 (`Failback`). `tHandoverTimeout`(3 s) 안에 안 내려놓으면 Standby로 돌아가 다시 기다린다. 둘 다 안 쓰는 틈(약 1.3 s)은 있지만 정상 통신 상태의 복귀에서는 둘 다 쓰지 않는다 (통신 이중 단절은 아래 스플릿 브레인 한계 참고).
- **동시 Active 방지**: 둘 다 Active로 보이면 Secondary가 Standby로 물러난다 (Primary 우선).
- **쓰기 대기**: Active가 된 뒤 `tWriteHold`(1 s)가 지나야 `g_bRedWrite`가 켜진다 (승격 직후 계산이 안정되기를 기다림). Active를 잃으면 바로 꺼진다. 상대에게 알리는 Active는 대기 없이 상태 전환 즉시 켜진다.
- **멈췄다 이어짐**: XAE·ADS의 Stop/Run은 변수를 초기화하지 않는다. 스캔 사이 시스템 시각(`TIME()`)이 1 s 넘게 벌어지면 `PRG_Redundancy`가 Active를 내리고 `Init`부터 다시 정한다 (`g_nRedResumeCount`). 안 그러면 멈춘 사이 Secondary가 Active를 넘겨받았는데도 옛 Active를 이어 쓰고 Secondary의 설정을 덮어쓴다.

**설정 동기화** (`FB_RedSync`, `g_bSyncFresh`·`g_bSyncInSync`·`g_nSyncApplied`)

- 동기화: `GVL_PERSIST`의 운영 설정(운전 모드 요청, 그룹 설정, Manual 정책, 지령원·제한·기동 조건·보정 등록, 보호 규칙, 출력 규칙, 유닛·버스 제한 등록, 명령 필터, Manual 지령 거르기, 시퀀스 설정), 브로커 감시 설정(`g_stBrokerWatch`)과 제어 모드 요청(`g_fbCtrlMode1.i_nModeReq`). 구조체 하나(2026-11-09 변경 뒤 크기는 빌드 후 `PmsStd.tmc`에서 확인)로 EAP 한 쌍. 대상을 늘리면 `ST_RedSyncCfg`와 `FB_RedSync`의 보내기·맞추기에 추가한다.
- 동기화하지 않음: 장비에서 읽는 값(두 PLC가 각자 읽음), 보호·시퀀서 진행 상태(매 스캔 다시 계산), 재기동 기록, **모듈 설정 묶음**(`g_stModuleSet`: SOC 제한, 제어 모드 전환 설정, 스케줄 시간표 등. 칸이 현장마다 다르고 한 프레임(1514 B)에 안 들어가서 표준은 동기화하지 않는다. 필요한 현장이 아래 "현장 모듈 설정 동기화"로 추가한다. 추가하지 않으면 HMI가 두 PLC 모두에 쓴다).
- 적용: 내가 Active가 아니고, 상대가 Active이며, EAP가 정상일 때만. 값이 다른 필드만 덮어써서 저장(`FB_PersistSave`)이 불필요하게 일어나지 않는다. Standby에서 설정을 바꿔도 Active 값으로 되돌아온다.
- 복귀 조건: Primary는 Standby에서 `g_bSyncInSync`(내 설정이 상대 Active와 같음)가 TRUE여야 `Handover`로 간다.
- 상대 값이 `i_tTimeout`(2 s) 안에 안 오면 낡은 것으로 본다 (`g_bSyncFresh = FALSE`).

**운영 규칙**

- **Manual 지령은 HMI가 두 PLC 모두에 쓴다.** 모듈 설정 묶음도 "현장 모듈 설정 동기화"를 추가하지 않은 현장은 같다: PLC가 꺼져 있던 동안 바꾼 모듈 설정은 그 PLC를 복귀시키기 전에 HMI에서 다시 쓴다. PLC를 교체·다시 다운로드했을 때도 같다. 추가하지 않으면 시스템이 이 값의 일치를 검사하거나 맞춰 주지 않는다.
- 운영 설정은 Active에서만 바꾼다 (Standby 값은 Active 값으로 덮인다).
- MQTT 명령은 두 PLC가 모두 받지만(구독은 QoS 0, 보존 없음) 쓰기는 Active만 한다. 꺼져 있거나 Standby가 놓친 일회성 명령은 되살리지 않는다. 반복되는 지령은 `tExpire` 안에 다시 오므로 곧 따라잡는다.
- 장비가 Modbus TCP 동시 접속을 두 개까지 허용해야 한다 (각 PLC가 직접 연결). 하나만 허용하는 장비는 Standby가 연결을 못 맺는다.
- EAP는 전용 NIC 또는 직결 케이블로 쓰는 것이 낫다 (멀티캐스트가 다른 장비로 새지 않고 루프 위험이 없다).

**설치** (현장마다. 코드는 건드리지 않는다)

1. `GVL_SITE.RED_SET`: `bUse := TRUE`, `sPrimaryNetId`·`sSecondaryNetId`. 시간 값(`tEapTimeout` 등)은 기본값으로 둔다.
2. 두 PLC에 같은 프로그램을 다운로드한다. 역할은 PLC의 AMS Net ID로 정해진다 (`PRG_Redundancy.sMyNetId`).
3. 접점 배선: PLC A의 EL2004 채널 1 → PLC B의 EL1004 채널 1 (alive), 채널 2 → 채널 2 (Active). 반대 방향도 같다.
4. XAE의 EAP 장치(두 PLC 대칭)에 Publisher·Subscriber를 **두 쌍** 만든다. 쌍마다 Publisher와 Subscriber의 변수 ID를 **같은 값**으로 맞춘다 (두 PLC가 같은 설정을 쓰려면 같아야 한다). 하트비트와 동기화는 다른 ID를 쓴다.

| EAP 쌍 | 변수 타입 | Publisher ← | Subscriber → |
|---|---|---|---|
| 하트비트 | `ST_RedHeartbeat` | `PRG_Redundancy.fbRedLink._stTx` | `fbRedLink._stRx`, Quality는 `fbRedLink._nEapQuality` |
| 동기화 | `ST_RedSyncCfg` | `PRG_Redundancy.fbRedSync._stTx` | `fbRedSync._stRx` |

5. I/O 연결: `fbRedLink._bDoAlive`·`_bDoActive`(EL2004 채널 1·2), `_bDiPeerAlive`·`_bDiPeerActive`(EL1004 채널 1·2).
6. `PRG_Redundancy`의 EAP 변수 크기는 유닛·그룹·지령원 수(`GVL_PARAM`)에 따라 바뀐다. 개수를 바꾸면 XAE에서 EAP 변수 크기를 다시 확인한다.

**확인용 변수** (`GVL_IF`)

| 변수 | 뜻 |
|---|---|
| `g_eRedRole` `g_eRedState` | 이 PLC의 역할, 이중화 상태 |
| `g_bRedActive` `g_bRedWrite` | Active, 장비 쓰기 허가(Active + 쓰기 대기 지남) |
| `g_ePeerState` `g_bPeerEapOk` `g_bPeerWireOk` | 상대 상태와 두 통로 |
| `g_nPeerRole` `g_nPeerState` `g_bPeerActive` `g_bPeerWireActive` | 상대가 보낸 역할·상태·Active (EAP / 접점) |
| `g_bRedRoleConflict` | 두 PLC 역할이 같음 |
| `g_bSyncFresh` `g_bSyncInSync` `g_nSyncApplied` | 동기화 값 수신·일치·덮어쓴 횟수 |
| `g_nRedSwitchCount` `g_eRedSwitch` `g_nRedResumeCount` | Active가 바뀐 횟수·마지막 이유, 멈췄다 이어진 횟수 |
| `g_nRedSeq` | 보낸 하트비트 수 |

**알려진 한계**

- **스플릿 브레인**: EAP와 접점이 둘 다 끊기면 Secondary는 상대가 죽었는지 자기 연결이 끊겼는지 구분할 수 없어 승격한다. Primary가 살아 있으면 양쪽이 Active가 된다. 통신이 돌아오면 Secondary가 양보한다. 막으려면 장비 쪽 증거(PCS가 어느 PLC의 명령을 받는지)가 필요하다.
  - 4회차 실측 (2026-11-05, CX5130 두 대): B(Secondary) 쪽에서 EAP와 A→B 접점이 끊기자 0.53 s 뒤 승격, 약 23 s 동안 두 PLC 모두 쓰기 허가. A는 B→A 접점이 살아 있어 `EapLost`만 보고 Active 유지. 접점을 다시 꽂고 4.3 s 뒤 B 양보. Secondary 입력 쪽 한 방향만 끊겨도 생긴다.
  - 결정 (2026-11-05): 코드는 그대로 두고 **현장 운영 규칙**으로 관리한다. ① EAP 케이블과 접점 배선은 서로 다른 경로로 깔고 커넥터를 고정 ② `RedLinkFault`(통신 한쪽 고장) 알람이 뜨면 바로 복구 (한쪽만 남은 채 운전하지 않음) ③ EAP·접점을 함께 건드리는 작업은 운전 정지 또는 한 PLC를 멈춘 상태에서만 ④ `RedPeerLost`와 양쪽 Active가 보이면 Secondary를 멈춘다.
- **램프 기준**: 램프 기준값은 그 PLC가 지난 스캔에 계산한 값이다 (장비에서 읽은 P가 아니다). Standby도 같은 입력으로 계속 계산하므로 오래 켜져 있었다면 Active와 같지만, Standby를 켜거나 멈춘 직후에 절체되면 램프가 0(또는 옛 값)에서 올라가 출력이 잠깐 떨어질 수 있다 (미측정).
- **Manual 유닛**: Manual 지령은 들어온 스캔에 한 번만 쓰고 장비가 유지한다. 승격 자체는 Manual 유닛에 쓰지 않는다. 하지만 Standby가 그 지령을 받지 못했다면 승격 뒤 램프·장비 제한으로 "따라가는" 상태가 될 때 Standby의 계산값(0)이 나갈 수 있다. 그래서 Manual 지령도 HMI가 두 PLC 모두에 쓴다.
- 멈췄다 이어짐 감지는 `TIME()` 간격 1 s에 의존한다.

**현장 모듈 설정 동기화** (현장에서 추가. 표준 코드는 고치지 않는다)

표준은 모듈 설정 묶음(`g_stModuleSet`)을 동기화하지 않는다. 묶음에 어떤 칸이 있는지가 현장마다 다르기 때문이다. 동기화가 필요한 현장은 아래 조각 전송 FB를 `90_Project`에 더한다. 이 FB는 묶음의 내용을 모르고 **주소와 크기**만 쓴다 (`FB_PersistSave`가 묶음을 비교하는 방식과 같다). 그래서 현장이 `ST_ModuleSet`에 칸을 더해도 FB는 고치지 않는다.

*언제 더하는가*

- 두 PLC의 값이 다르면 절체·복귀 때 지령이 달라지는 설정이 묶음에 있을 때: 스케줄 시간표, 정출력 값, 피크 목표, SOC 제한 등.
- 더하지 않으면 위 "운영 규칙"대로 HMI가 두 PLC 모두에 쓴다. 이때 시스템은 두 값의 일치를 검사하지 않는다.

*왜 조각으로 보내는가*

- EAP 텔레그램 하나(머리 + 같은 Publisher 박스에 든 변수 전부)는 1514 B를 넘을 수 없다 (Beckhoff 문서 "EAP telegram structure"). 넘으면 Publisher의 `FrameState` 0x0002(frame oversized)가 켜진다.
- 기본 구성의 크기 (`PmsStd.tmc` 기준. 칸 수·배열 크기에 따라 바뀐다):

| 항목 | 크기 |
|---|---|
| `ST_ModuleSet` 전체 (PCS 2대분) | 3996 B |
| `ST_CmScheduleSet` (PCS마다 1칸) | **1680 B** (변수 하나에 못 들어간다) |
| `ST_CmCommonSet` | 416 B |
| 그 밖의 칸 (`ST_SocLimitSet` `ST_CtrlModeSet` `ST_CmConstPowerSet` `ST_CmPeakCutSet` `ST_CmSchedPeakSet`) | 각 16~40 B |

- 그래서 묶음을 1000 B 조각으로 나누어 스캔마다 하나씩 돌려 보낸다. 3996 B면 4조각이고 한 바퀴에 약 40 ms다.

*동작*

- **보내기**: 한 바퀴를 시작할 때(0번 조각) 묶음을 통째로 한 번 복사해 두고 세대 번호(`nGen`)를 올린다. 그 바퀴의 조각은 모두 이 사본에서 보낸다. 바퀴 도중 설정이 바뀌어도 조각마다 값이 섞이지 않는다.
- **받기**: 같은 세대의 조각이 모두 모이면 한 벌로 확정한다. 세대가 바뀌면 처음부터 다시 모은다.
- **맞추기**: `FB_RedSync`와 같은 조건(`i_bApply`)일 때만, 내 묶음이 받은 한 벌과 다르면 통째로 덮어쓴다. 같으면 `q_bInSync`.
- **복귀 조건**: `PRG_Redundancy`가 `g_bSyncInSync`에 이 FB의 `q_bInSync`를 AND로 묶는다. 모듈 설정까지 같아야 Primary가 복귀를 시작한다.

*추가 순서*

1. `90_Project`에 파일 셋을 만든다: `GVL_RedSyncMod`, `ST_RedSyncChunk`, `FB_RedSyncMod` (아래 코드).
2. `PRG_Redundancy`에 인스턴스를 선언하고 호출한다 (아래 코드). I/O(`AT %I*`·`%Q*`)가 있으므로 인스턴스는 RedundancyTask를 도는 `PRG_Redundancy` 안에 둔다.
3. XAE의 EAP 장치(두 PLC 대칭)에 **Publisher 박스를 하나 더** 만들고 변수를 하나 넣는다. 변수 ID는 표준(1401·1402)과 다른 값(예 1403), 크기는 `SIZEOF(ST_RedSyncChunk)` (정렬 때문에 합계와 다를 수 있으니 `PmsStd.tmc`에서 확인), `fbRedSyncMod._stTx`에 연결한다. Subscriber는 같은 ID로 `_stRx`에 연결한다. 박스를 따로 두면 프레임이 따로라서 표준 하트비트(16 B)·동기화(384 B)와 크기를 다투지 않는다. 변수 ID는 망 안에서 유일해야 한다 (Beckhoff 문서 "EAP telegram structure").
4. 두 PLC에 같은 프로그램을 다운로드한다. 묶음 크기가 다르면 `q_bSizeMismatch`가 켜지고 적용하지 않는다.
5. 아래 "시험"을 한다.
6. 현장이 `ST_ModuleSet`을 키워 `GVL_PARAM.MODULE_SET_BYTES`(8192)를 올리면 `GVL_RedSyncMod.CHUNK_COUNT_MAX`는 자동으로 따라간다. 변수 ID·크기는 그대로다.

*코드 (예시. 현장에서 빌드·시험한다)*

`GVL_RedSyncMod`:

```
VAR_GLOBAL CONSTANT
	CHUNK_BYTES		: UDINT := 1000;	// 조각 하나의 데이터 크기 (ST_RedSyncChunk.abData와 같아야 함)
	CHUNK_COUNT_MAX	: UDINT := (GVL_PARAM.MODULE_SET_BYTES + CHUNK_BYTES - 1) / CHUNK_BYTES;	// 조각 수 상한
END_VAR
```

`ST_RedSyncChunk`:

```
TYPE ST_RedSyncChunk :		// 모듈 설정 묶음의 조각 하나 (EAP로 보냄)
STRUCT
	nProtoVer	: UINT := 1;	// 형식 번호
	nGen		: UDINT;		// 세대: 한 바퀴마다 1 증가. 같은 세대의 조각만 모은다
	nIdx		: UINT;			// 조각 번호 (0부터)
	nCount		: UINT;			// 조각 수
	nSize		: UDINT;		// 묶음 전체 크기 (B)
	abData		: ARRAY[0..GVL_RedSyncMod.CHUNK_BYTES - 1] OF BYTE;
END_STRUCT
END_TYPE
```

`FB_RedSyncMod`:

```
FUNCTION_BLOCK FB_RedSyncMod		// 모듈 설정 묶음을 조각으로 상대에게 보내고, 상대가 Active이면 받은 값으로 맞춘다 (FB_RedSync와 같은 규칙)
VAR_INPUT
	i_bUse		: BOOL;
	i_bApply	: BOOL;				// 받은 값을 덮어써도 됨 (FB_RedSync의 i_bApply와 같은 조건)
	i_pData		: PVOID;			// 동기화할 메모리 주소: ADR(GVL_PERSIST.g_stModuleSet)
	i_nSize		: UDINT;			// 그 크기: SIZEOF(GVL_PERSIST.g_stModuleSet)
	i_tTimeout	: TIME := T#2S;		// 새 조각이 이 시간 안에 안 오면 낡은 것
END_VAR
VAR_OUTPUT
	q_bInSync		: BOOL;			// 받은 값이 낡지 않았고 내 묶음이 그와 같음 (i_bApply일 때만 TRUE)
	q_bFresh		: BOOL;			// 상대 조각이 계속 오고 있음
	q_bSizeMismatch	: BOOL;			// 상대 묶음 크기가 내 것과 다름 (두 PLC 프로그램이 다름): 적용 안 함
	q_nApplied		: UDINT;		// 받은 값으로 덮어쓴 횟수
END_VAR
VAR
	(* I/O: XAE에서 연결 *)
	_stTx		AT %Q* : ST_RedSyncChunk;
	_stRx		AT %I* : ST_RedSyncChunk;

	_abSnap		: ARRAY[0..GVL_PARAM.MODULE_SET_BYTES - 1] OF BYTE;	// 보내는 쪽 사본
	_abAsm		: ARRAY[0..GVL_PARAM.MODULE_SET_BYTES - 1] OF BYTE;	// 받은 조각을 모으는 곳
	_abRx		: ARRAY[0..GVL_PARAM.MODULE_SET_BYTES - 1] OF BYTE;	// 한 벌이 다 모인 값
	_abGot		: ARRAY[0..GVL_RedSyncMod.CHUNK_COUNT_MAX - 1] OF BOOL;
	_nTxIdx		: UDINT;
	_nTxGen		: UDINT;
	_nAsmGen	: UDINT;
	_nAsmCount	: UINT;
	_nAsmSize	: UDINT;
	_nRxSize	: UDINT;
	_bRxValid	: BOOL;
	_nLastGen	: UDINT;
	_nLastIdx	: UINT;
	_tonFresh	: TON;
END_VAR
VAR_TEMP
	nCount		: UDINT;
	nOff		: UDINT;
	nLen		: UDINT;
	i			: UDINT;
	bNew		: BOOL;
	bAll		: BOOL;
END_VAR
```

```
IF NOT i_bUse OR i_pData = 0 OR i_nSize = 0 OR i_nSize > GVL_PARAM.MODULE_SET_BYTES THEN
	q_bInSync := FALSE;
	RETURN;
END_IF

(* 1) 보내기: 스캔마다 조각 하나. 0번 조각에서 묶음을 한 번 복사해 한 바퀴를 같은 시점의 값으로 보냄 *)
nCount := (i_nSize + GVL_RedSyncMod.CHUNK_BYTES - 1) / GVL_RedSyncMod.CHUNK_BYTES;
IF _nTxIdx >= nCount THEN
	_nTxIdx := 0;
END_IF
IF _nTxIdx = 0 THEN
	MEMCPY(ADR(_abSnap), i_pData, i_nSize);
	_nTxGen := _nTxGen + 1;
END_IF
nOff := _nTxIdx * GVL_RedSyncMod.CHUNK_BYTES;
nLen := MIN(GVL_RedSyncMod.CHUNK_BYTES, i_nSize - nOff);
_stTx.nProtoVer	:= 1;
_stTx.nGen		:= _nTxGen;
_stTx.nIdx		:= UDINT_TO_UINT(_nTxIdx);
_stTx.nCount	:= UDINT_TO_UINT(nCount);
_stTx.nSize		:= i_nSize;
MEMCPY(ADR(_stTx.abData), ADR(_abSnap[nOff]), nLen);
_nTxIdx := _nTxIdx + 1;

(* 2) 받기: 새 조각이 오는지 *)
bNew := _stRx.nGen <> _nLastGen OR _stRx.nIdx <> _nLastIdx;
_nLastGen := _stRx.nGen;
_nLastIdx := _stRx.nIdx;
_tonFresh(IN := NOT bNew, PT := i_tTimeout);
q_bFresh := _stRx.nProtoVer = 1 AND _stRx.nCount > 0 AND NOT _tonFresh.Q;

(* 3) 모으기: 같은 세대의 조각이 다 모이면 한 벌로 확정. 값이 이상한 조각은 버림 *)
IF bNew AND q_bFresh
	AND _stRx.nCount <= GVL_RedSyncMod.CHUNK_COUNT_MAX AND _stRx.nIdx < _stRx.nCount
	AND _stRx.nSize <= GVL_PARAM.MODULE_SET_BYTES
	AND _stRx.nIdx * GVL_RedSyncMod.CHUNK_BYTES < _stRx.nSize THEN
	IF _stRx.nGen <> _nAsmGen OR _stRx.nCount <> _nAsmCount OR _stRx.nSize <> _nAsmSize THEN	// 새 세대: 처음부터
		_nAsmGen	:= _stRx.nGen;
		_nAsmCount	:= _stRx.nCount;
		_nAsmSize	:= _stRx.nSize;
		FOR i := 0 TO GVL_RedSyncMod.CHUNK_COUNT_MAX - 1 DO
			_abGot[i] := FALSE;
		END_FOR
	END_IF
	nOff := _stRx.nIdx * GVL_RedSyncMod.CHUNK_BYTES;
	nLen := MIN(GVL_RedSyncMod.CHUNK_BYTES, _stRx.nSize - nOff);
	MEMCPY(ADR(_abAsm[nOff]), ADR(_stRx.abData), nLen);
	_abGot[_stRx.nIdx] := TRUE;
	bAll := TRUE;
	FOR i := 0 TO _stRx.nCount - 1 DO
		IF NOT _abGot[i] THEN
			bAll := FALSE;
		END_IF
	END_FOR
	IF bAll THEN
		MEMCPY(ADR(_abRx), ADR(_abAsm), _stRx.nSize);
		_nRxSize	:= _stRx.nSize;
		_bRxValid	:= TRUE;
	END_IF
END_IF

(* 4) 맞추기: i_bApply이고 한 벌이 있을 때만. 다르면 통째로 덮어씀 *)
q_bInSync := FALSE;
q_bSizeMismatch := _bRxValid AND _nRxSize <> i_nSize;
IF i_bApply AND q_bFresh AND _bRxValid AND NOT q_bSizeMismatch THEN
	IF MEMCMP(i_pData, ADR(_abRx), i_nSize) <> 0 THEN
		MEMCPY(i_pData, ADR(_abRx), i_nSize);
		q_nApplied := q_nApplied + 1;
	ELSE
		q_bInSync := TRUE;
	END_IF
END_IF
```

`PRG_Redundancy` (선언에 한 줄, 동기화 블록 바로 뒤에 호출):

```
// VAR
	fbRedSyncMod	: FB_RedSyncMod;		// 현장 모듈 설정 동기화 (조각 전송)

// 3) 동기화 블록(fbRedSync) 바로 뒤, 4) 상태 기계 앞
IF GVL_SITE.RED_SET.bUse THEN
	fbRedSyncMod(	i_bUse		:= TRUE,
					i_bApply	:= GVL_IF.g_eRedState <> E_RedState.Active AND GVL_IF.g_bPeerEapOk AND GVL_IF.g_bPeerActive,
					i_pData		:= ADR(GVL_PERSIST.g_stModuleSet),
					i_nSize		:= SIZEOF(GVL_PERSIST.g_stModuleSet));
	GVL_IF.g_bSyncInSync	:= GVL_IF.g_bSyncInSync AND fbRedSyncMod.q_bInSync;		// 복귀 조건에 모듈 설정 포함
END_IF
```

*주의*

- 묶음에 **포인터·인터페이스 칸을 두지 않는다.** PLC마다 주소가 달라 복사하면 안 된다.
- 받은 값이 이 PLC의 `FB_SettingGuard`에서 거부되어 정상값으로 되돌려지면 `q_nApplied`가 계속 늘고 `q_bInSync`가 맞지 않는다. 두 PLC가 같은 프로그램인지, 그 값이 Active에서 정상인지 본다.
- 방향은 Active → Standby 한 방향이고, 설정에 버전이 없다. 먼저 Active가 된 쪽의 값이 다른 쪽을 덮는다. 두 PLC를 모두 끈 뒤에는 **마지막에 Active였던 PLC를 먼저 켠다.** (Primary를 먼저 켜면 Primary가 Active가 되어, Secondary에서만 바꾼 최신 값이 옛 값으로 덮인다.)
- 동기화하는 묶음은 **Active에서만 바꾼다.** Standby 값은 Active 값으로 덮인다. Manual 지령은 동기화하지 않으므로 여전히 HMI가 두 PLC 모두에 쓴다.
- 대역은 약 1 KB × 100회/s ≈ 0.8 Mbit/s다. EAP 전용 NIC나 직결 케이블에서 쓴다.
- 칸 일부를 두 PLC가 **다르게** 가져야 하면 이 방식(통째 복사)을 쓰지 않는다. 그때는 `FB_RedSync`·`ST_RedSyncCfg`를 본보기로 필요한 칸만 현장 구조체에 담아 보내는 FB를 만든다. 변수 하나의 크기는 1514 B 아래로 한다.

*시험*

1. 두 PLC를 온라인으로 놓고 Active에서 모듈 설정 한 칸을 바꾼다. Standby의 같은 칸이 수십 ms 안에 따라오고 Standby의 `fbRedSyncMod.q_nApplied`가 1 오른다.
2. Standby에서 같은 칸을 바꿔 본다. Active 값으로 되돌아온다.
3. Primary를 정지하고 Secondary(Active)에서 값을 바꾼 뒤 Primary를 시작한다. Primary가 Standby에서 값을 받아 `g_bSyncInSync`가 TRUE가 된 뒤 10 s 지나서 복귀한다. 복귀 전에 두 PLC의 값이 같은지 본다.
4. 한쪽 프로그램만 `ST_ModuleSet`을 다르게 만들어 본다. `q_bSizeMismatch`가 켜지고 `g_bSyncInSync`가 FALSE라 복귀하지 않는다.
5. XAE에서 새 Publisher 박스의 `FrameState`가 0인지 확인한다 (0x0002 = 프레임 초과).

---

## 8. 이름 규칙

```
범위 접두 + 타입 접두 + 뜻         예) iq_astCompReg = 입출력 · 구조체 배열 · 보정 등록
```

| 범위 | 뜻 | 타입 | 뜻 |
|---|---|---|---|
| `g_` | 전역 | `b n f s t e st` | BOOL·정수·실수·문자열·시간·열거·구조체 |
| `i_` `q_` `iq_` | 입력·출력·입출력 | `fb ip ton rt` | FB·인터페이스·TON·R_TRIG |
| `_` | FB 내부 | `a` + 타입 | 배열 (`ab af ast aip`) |
| 없음 | 임시 | 대문자 | 상수 |

- `q_`는 출력이다 (IEC 주소 `%Q`에서 왔다). 무효전력 Q는 이름 가운데에 대문자로 쓴다 (예: `q_afQUnit` = 출력 · REAL 배열 · 유닛 Q).
- 타입 이름은 `ST_ E_ FB_ I_ F_ GVL_ PRG_`로 시작한다. 열거형에는 `qualified_only`와 `strict`를 붙인다.
- 이름은 뜻이 드러나게 짓는다. 숫자·약어로 줄이지 않는다. 메서드·액션은 동사 + 대상, 변수는 무엇의 값인지를 쓴다. 예외는 반복 변수 `u g b`와 표준 줄임 `Cfg Prot Bat Avail`이다.
- **Auto·Manual에서 쓰임이 다르면 이름에 `Auto`·`Manual`을 넣는다.** 한쪽만 쓰는 값은 그쪽 이름을 붙이고(`iq_abManualWaitNewCmd`, `g_stManualCmdSet`), 두 쪽을 따로 켜고 끄는 설정은 `bAuto`·`bManual`로 나란히 둔다(`ST_ProtCfg`). 경로가 같은 코드를 쓰면 인스턴스 이름으로 구분한다(`fbAutoShape`·`fbManualShape`). 두 쪽이 똑같이 쓰는 값은 붙이지 않는다(`ST_GroupSet.bRamp`, `g_stCmdFilter`). 줄여 쓰지 않는다 (`Man` 아님).
- 액션을 먼저 쓴다. 본문을 단계로 나누는 건 액션(`A_`)으로 한다. 메서드(`M_`)는 인터페이스 구현, 인자·반환값이 필요할 때, 상속 기반 FB의 PROTECTED 기능일 때, 한 FB를 스캔 중 여러 단계로 나눠 부를 때(`fbSequencer.M_Prepare`·`M_CheckRequest`)만 쓴다. 액션이 본문의 `VAR_TEMP`를 못 볼 수 있으므로, 본문과 액션이 함께 쓰는 반복 변수·임시값은 `VAR`에 `_`를 붙여 두고 매 스캔 쓴 뒤에 읽는다 (`FB_EventLog`·`FB_EquipmentLimits`·`FB_UnitSeq`).
- 배열은 1부터 센다. 전력은 REAL kW·kvar이고 방전·공급이 +다. 계산은 끝까지 실수로 하고, 장비 레지스터에 맞춘 정수 변환은 드라이버에서만 한다.

---

## 9. 코드 작업 방법 (AI용)

### 설계 점검 (고치기 전·후 반드시)

기능을 넣을 때 요청한 기능만 보면 중복·역할 겹침·코어 경계 위반이 쌓인다. 형식 점검(아래 "파일 형식")만으로는 걸러지지 않으므로, 코드를 고칠 때마다 아래를 한다.

**고치기 전**
1. 바꿀 FB·파일 목록을 먼저 뽑는다.
2. 같은 일(계산·판단·기록·설정 값)을 이미 하는 코드가 있는지 전체에서 찾는다 (이름·주석·비슷한 식으로 검색). 있으면 그 결과를 받아 쓰고, 새로 만들지 않는다.
3. 넣을 자리를 정한다: 현장마다 다를 수 있는 선택이면 모듈(`20_Module`)·설정(GVL)·MAIN, 모든 현장이 같은 흐름이면 코어(`10_Core`). 코어에 넣는다면 왜 모듈·설정·MAIN이 아닌지 말할 수 있어야 한다.
4. 고칠 FB가 이미 역할이 많으면, 기능을 더하지 말고 먼저 나눌지 사용자에게 묻는다.

**고친 뒤 (형식 점검과 같이 하고, 결과를 보고에 적음)**

| 점검 | 볼 것 |
|---|---|
| 중복 | 같은 계산·판단·기록·사본이 두 곳 이상에 생겼는가 (의도한 안전장치면 이유를 주석·문서에) |
| 한 FB 한 역할 | 고친 FB가 역할을 하나 더 떠안았는가. 첫 줄 주석 한 문장으로 하는 일을 설명할 수 있는가 |
| 기록 | 이벤트 기록이 `FB_EventLog` 밖에서 생겼는가. 바뀐 로그 문구(인자 포함)가 95자 이내인가 (7장 "로그 문구 길이") |
| 장비 쓰기 | 장비·DO에 쓰는 곳이 R2에 정한 곳(`FB_Dispatcher`·`FB_AuxOutput`·새 장비 종류의 모듈) 밖에서 생겼는가 |
| 코어 경계 | 코어에 현장별 선택지·현장 이름·장비 맵이 들어갔는가 |
| 설정 | 운전 중 바꾸는 값이 PERSISTENT·값 점검(`M_CheckSet` 또는 `FB_SettingGuard`)·저장 연결을 갖췄는가 |
| 원칙 | 2장 원칙에 하나씩 비춰 어긋나는 곳이 없는가 (출력 제한 이유가 보이는가, 요청만 내는가, 경로는 Auto/Manual만인가, 미설치 처리 등) |
| 안 쓰는 것 | 쓰이지 않는 변수·입력·파일이 남았는가 |
| 단순 | 처음 보는 사람이 FB를 열어 흐름을 따라갈 수 있는가. 더 짧고 곧게 쓸 수 있는가 |
| 문서 | 바뀐 동작·이름·설정이 이 문서(3·4·5·7·11·12장)에 반영됐는가 |

- 보고에는 "문제없음"만 쓰지 말고 무엇을 봤는지와 남은 것을 적는다.
- 큰 변경 뒤나 몇 단계를 이어서 고친 뒤에는 코어 전체를 한 번 다시 읽어 위 표로 검토한다.

### 파일 형식

- `.TcPOU` `.TcDUT` `.TcGVL` `.TcIO`는 XML이다. UTF-8 BOM과 CRLF를 지키고, 코드는 CDATA 안에 있다. 고칠 때는 파이썬으로 읽고 → 바꾸고 → 쓴다. 바꿀 문자열이 정확히 한 번 나오는지 확인한다.
- 새 파일은 `Id="{GUID}"`를 새로 만들고, `PmsStd.plcproj`에 `<Compile Include="...">`를 추가한다. 새 폴더면 `<Folder Include="...">`도 추가한다.
- 메서드는 `<Method Name=… Id=…>`, 액션은 `<Action Name=… Id=…>`, 속성은 `<Property>` + `<Get>`으로 쓴다.
- 라이브러리는 `plcproj`의 `<PlaceholderReference>`로 추가한다 (현재: Tc2_Standard, Tc2_System, Tc3_Module, Tc3_EventLogger, Tc2_Utilities, Tc2_ModbusSrv, Tc3_IotBase).
- 이벤트 클래스는 `PmsStd.tsproj`의 DataTypes에 있다. 이벤트를 추가할 때는 같은 형식(`<EventId><Name Id=…>`)으로 덧붙인다.
- 지울 파일은 직접 삭제하지 않고 따로 옮겨 둔다 (0장).
- 고친 뒤 점검: XML 파싱, UTF-8 BOM·CRLF, `plcproj` 등록(빠진 파일·남은 항목), GUID 중복, 지운 이름이 남았는지.
- 공유할 때는 백업 폴더, 지울 파일을 옮겨 둔 폴더, `.vs`, `*.~u`, `*.bak`, `_CompileInfo`, `_Boot`를 뺀다.

### 컴파일 함정

- IL 연산자 이름(`s r st ld` 등)을 변수명으로 쓰면 오류가 수십 개 난다.
- **NaN을 크기 비교(`<` `>` `<=` `>=`)하면 실행 중 부동소수점 예외(SSE invalid operation)로 PLC가 멈춘다** (빌드는 됨). 바깥에서 온 실수(EMS 명령, 장비 값, 모듈 결과)는 비교하기 전에 `F_IsFinite`(비트 검사)로 거른다. `x = x` 같은 자기 비교로 NaN을 찾지 않는다. AND·OR는 단락 평가가 아니므로 `NOT F_CmdOk(x) OR x < 0`처럼 한 줄에 쓰지 말고 IF를 나눈다. 시험에서 NaN은 비트(`16#7FC00000`)를 `MEMCPY`로 넣어 만든다.
- 예약어(`Limit Internal Any`)를 열거값으로 쓰지 않는다.
- 메서드가 FB의 `VAR_IN_OUT`을 직접 읽으면 경고가 난다. 필요한 값은 멤버로 복사해 두거나 메서드 입력으로 넘긴다.
- 문자열 상수가 받는 쪽 길이보다 길면 경고가 난다 (`WSTRING(n)` 길이를 확인한다).
- 메서드·임시 변수는 온라인에서 안 보인다. 따라가야 하는 중간값은 FB 변수나 진단 칸으로 뺀다. 매 스캔 새로 계산하는 값은 사용 전에 초기화한다.

### 이름 바꾸기

- 단어 경계로만 바꾼다. 구조체 필드(`.이름`)와 구분한다. `CDATA`와 `Method/Action Name` 속성만 바꾼다.
- 바꾼 뒤 옛 이름이 남았는지 전체를 다시 찾아본다. `LineIds` 안의 이름은 TwinCAT이 다시 만든다.

---

## 10. 표준에 남은 것

- EMS config: protection 6개(배터리·배터리실 온도·습도 경고·고장)와 pcs `PvMarginRate`는 반영하지 않음 (값의 뜻은 EMS 사양, 온도·습도는 보조 IO 센서·보호 칸 필요). SOC는 버스별 설정이 없어 BMS 값 중 가장 엄격한 값 (2026-11-04)
- 접속점 기준 제한 `bManual` FALSE 유지 경고 없음 (현장 시험 뒤 TRUE로 되돌리는 것은 절차로) (2026-11-04)

- [ ] 가상 시험
  - 보호 재설계 (2026-10-26): 보호 정지(PCS 고장 Stop) → 출력 0 → Stop 확인 / 확인 안 되면 `nStopRetry`회 뒤 명령 실패 알람만 (기동 요청은 받음) / PCS 통신 끊김 + 통신 보호를 Stop으로 → 통신 복구 때 Stop 1회 / BMS 고장(PCS Stop + 배터리 Stop) → PCS 정지 확인 뒤 배터리 개방, PCS 통신 끊김이면 바로 개방 / 배터리 비상정지도 PCS 정지 뒤 / 개방 확인 안 되면 BMS 명령 실패 알람, 잠금 없음 / 보호 대기 → 출력 0(명령 없음) → 풀리면 다시 출력 / `ToManual` + 대기 → Manual 전환 뒤 그대로 / 표준 대응 칸을 비우면 알람만 / 배터리 조치 + PCS None 설정 → 값 받지 않음 / P·Q 상한 따로(`fPPct`·`fQPct`) / 운전원 Run(멈춘 유닛 활성) · Standby(멈춘 유닛이면 활성, 활성이면 할 일 없음) · Stop, 활성 유닛에 Run을 다시 보내지 않는지 / 순서 기동(배터리 투입 → 기동 조건 → Run), 보호가 걸리면 멈춤 / 모드 전환 대기 확인(장비 Run 그대로 + 실측 0이면 끝) / PCS·BAT 리셋이 그 장비에만 가고 다른 상태는 그대로 / 시험 리셋 / SOC 한계 → 한도 0 → 마진 회복 뒤 재개 (Auto·Manual 둘 다)
  - 설정 값 지키기·기록: 운전 중 SOC 하한 ≥ 상한, 음수 마진·해소 폭, `fBadFactor` ≤ 1, `tWritePeriod`·확인 시간 0을 넣으면 값이 바로 마지막 정상값으로 돌아가고(화면에서도) "설정 값 받지 않음" 1건, 쓰기는 계속됨 / 모듈 설정 두 개를 같이 바꿨는데 하나가 잘못이면 둘 다 되돌아감 / 저장된 값이 처음부터 잘못이면 설정 오류(쓰기 차단), 고치면 풀림 / 설정을 바꾸면 "운영 설정 바뀜" 1건(바뀐 이름), 저장 실패 알람
  - 설정 PERSISTENT: `g_stModuleSet`·`g_stCmdFilter`·`g_stManualCmdSet`·`g_stSeqSet`을 온라인으로 바꾸면 그 스캔부터 적용되고 1 s 뒤 저장(`fbPersistSave.q_nSaveCount` 증가), 재기동 뒤 유지, 처음 내려받을 때 초기값 확인, `A_PassModuleSet` 줄을 빼면 기본값으로 동작
  - 이번 정리: 미설치(PCS·BMS 칸 비움 → 통신 경보 없음, 정격 0 허용, 버스 집계 제외), 시퀀서 나눔 뒤 동작이 같은지(배터리 BatClose·BatOpen·UnitRunning 거부, 운전원이 연 배터리 유지, 정지 시 개방, 기동 조건 대기, 그룹 상태), 설정 오류 기록·알람이 전과 같은지, P 추적표가 전과 같은지, 같은 인스턴스 여러 칸 등록(모듈 ID 겹침 오류 없음), `FB_BreakerCtrl`(투입·개방 펄스, 개방 우선, 상태 접점 없음·통신 끊김에서 투입 거부, 확인 시간 초과 → 실패 알람, 실패 풀기)
  - Gate: 칸을 비우면 전과 같은지(Manual 방향 반대 무시·앞 값 유지, Run 뒤 새 지령 대기, 첫 스캔에 운전 중 출력 안 바꿈, Auto 통과), 시험 Gate(예: Auto 그룹 P 보류)가 따르는지, Gate가 다른 경로 유닛 값을 채워도 나가지 않는지
  - 정책 모듈: 칸을 비우면 전과 같은지(등급·MinAbs·정격·SOC·균등), 시험 정책(예: 후보 합 / 유닛 1만 무게)을 넣으면 따르는지, 정책이 NaN을 내면 버리고(`q_anPolicyBad`) 무게 음수·NaN은 0인지
  - 기록·한도 추가: 기동 조건 대기 로그(막은 모듈이 바뀌면 다시 1건, 풀리면 다음 대기에 1건), 지령값 이상 로그, `FB_UnitLimit`(P·Q 한도 켜고 끄기, 0 넘는 하한 → 설정 값 되돌림, UnitRule 이유·"사용자 한도"), 지령원 Q 제한(`FB_SrcTest`로 Q 상·하한 → Auto Q가 잘림, Manual은 `bSrcLimit` 따라, 끊김 뒤 마지막 제한 유지, 보정을 켜도 접속점 Q가 지령원·접속점 기준 모듈 Q 한도에 맞는지)
  - 상태·시퀀스: 접수 검사(정지 중 Run 거부, 단계별 거부 이유), 리셋 순서, PCS Fault = 정지, 통신 끊김 = 정지 아님(BMS 일반 개방 대기), Auto/Manual 경로 분리, Manual 정책, 램프 0 시작, BatOpen 유지, 재시도 중 통신 대기, 보호 bAuto/bManual, 새 설정 점검
  - 역할 정리: 출력 가능 들어감·나옴 때 Manual 0 쓰기(가용 잃음·전환 시작 포함), 그룹 제한 모듈이 보는 가용 한도가 Derate·SOC 감발을 반영하는지, 모듈 ID 겹침 점검·로그 모듈 이름(지령원 제한 문구 포함)
  - 설비 한도·SOC: 보호 P·Q 상한이 그룹 정격 줄에 들어감, 배터리 없는 유닛이 0으로 잘리지 않음, 버스 2개에서 한 버스 한도 초과분이 다른 버스로 재분배, 같은 버스 유닛 사이 비율, Manual `bRating`·`bBatLimit` 끔, `FB_SocLimit` 감발·막음·재개(bResume 램프)·SOC 무효, 유닛 제한 모듈(UnitRule 이유·모듈 ID)
  - 기본 동작: 기동·정지·모드 전환·Manual·보호·설정 오류, 유닛 지령 기본(그룹 P만 / 유닛 P만 / 둘 다, 제한 걸림), 명령 재전송 규칙, 보호 중 요청 버림, `g_stSeqSet` 온라인 변경, 출력 추적표·로그 이유
  - 보호 모드 전환(`eModeAction`): 항목을 `ToManual`로 두면 Auto 유닛이 보호 발생 때 요청·실제 모드 모두 Manual로 바뀌는지(MODE_SET 단계 포함), 보호가 풀려도 Manual 유지, 걸려 있는 동안 화면에서 Auto를 다시 요청하면 되돌아가는지, 이미 Manual인 유닛은 변화 없음, `bAuto` 끔·`None`이면 효과 없음, `PmsSeq.ModeForced` 기록 1건(보호 중 Auto를 여러 번 요청해도 1건, 풀렸다 다시 걸리면 또 1건), Manual→Auto 전환 중 보호가 걸리면 Auto를 거치지 않고 Manual로 끝나는지, `ePcsAction = Standby` + `ToManual`(Manual 전환 + 대기)
  - MQTT 로거(2026-10-28): **확인함** (메인 머지 전 코드, 런타임 4024.17): 리스너 수신(`OnMessageSent`), `A_PublishLogger` 발행(`anLoggerNext` 증가). **남음**: 메인 머지 뒤 빌드 (`TC_EVENT_CLASSES` 필터 포함)·수신 확인 / `PRG_TestLogger` 자동 찾기 `nFound` 34·`nCheck` 0·`nScanOverflow` 0 (없는 번호 요청이 오류·빈 문구로 오는지) / `bSendAll` → `bAllOk` / `bAlarm` 발생·`[해소] ` 쌍 / `bSendLong` → `…`·`q_nTruncated` 증가 / 브로커를 켜고 PC 도구(`PMS_Log_Tester`)로 받아 재pack "일치", 한글·TimeStamp(현지 시각, 해소는 해소 시각) 확인 / 브로커 끊김 → 재접속 뒤 밀린 것 전송, 64건 넘게 밀리면 `q_nLost` / 이중화: 대기 PLC는 로그를 보내지 않고 절체 뒤 새 Active가 이어서 보내는지 / 기동 직후 "PMS 시작" 로그가 접속 뒤 나가는지 / 줄인 이벤트 문구 3개(출력 제한·보호로 Manual 전환·지령값 이상)
  - 이번 정리(2026-10-20): Gate 합침(`FB_Gate` 두 인스턴스) 뒤 Auto·Manual 동작이 같은지, `FB_EventLog` 액션 분리 뒤 기록 내용·순서가 같은지(PMS 시작 기록 포함), `FB_EquipmentLimits`·`FB_UnitSeq` 분리 뒤 한도·시퀀스가 같은지, SOC 재개 마진 이름 변경 뒤 값 유지
- [x] 제어 모드 가상 시험 (기존 가상 장비 시험 기록)
  - 자동 시험: `PRG_TestCtrlMode` 31건, `PRG_TestUnitMode` 11건 모두 통과
  - 실경로 (가상 PCS 실제 출력까지): 스케줄 −200 / +100 / 0, 피크컷 P1 피크 방전 91.6·P2 0·P3 자동 충전 −1000(상한)·P4 잔량 방전 465.5(역송 방지 상한), 통합 I1 100·I2 108.3(피크 > 시간표)·I3 −200·I4 465.5 (PCS 1대 때 값)
  - PCS별: PCS1 +200 / PCS2 +100 (다른 시간표) → PCS2만 정출력 150 (PCS1 그대로) → PCS2 +600 `ByRating` 465.5 → `Actual` 600
- [x] 스탠다드 스케줄 모드 시험 (2026-10-28): 자동 시험 51~57 통과(다시 돌려도 18/18), 실경로 `PRG_TestStdLive` 통과 (모드·관리자·코어 고르기 100 → −50 → 0). PCS 실제 출력은 `DEVICE_WRITE = FALSE`라 보지 않음
- [x] NaN 거르기 (2026-10-28): 코어 `F_IsFinite` 추가, `F_CmdOk`·`FB_Distributor` 무게·`FB_EventLog` 이상값 문구·`FB_EmsCommand` P 값이 비교 전에 비트로 거름. 자동 시험 58~63 통과 (24/24). MQTT 설정 값(`20_Module/Mqtt/Config`) 등 다른 바깥 실수는 쓰는 곳이 생길 때 같은 규칙으로
- [ ] 제어 모드 남은 것
  - 15분 구간 경계 리셋을 실경로로 시험 안 함
  - 요청 모드(`i_nModeReq`)가 PERSISTENT가 아님 (재기동하면 모드 없음, 2026-11-09 이대로 유지로 결정). HMI·MQTT 연결 없음. 모드 ID 점검은 2026-11-09 추가
  - 시간대별 모드 자동 전환, 비상 충전(Messis) 없음
  - 모드 모듈 ID 겹침 점검 없음 (`FB_ModuleList`에 안 들어감)
- [ ] 실제 장비 시험 (드라이버 통신, 명령 접수 확인·재시도, 통신 끊김 보호)
- 결정 기록 (2026-10-26): 보호 재설계 (코드 반영함. 동작은 2·4·6·7·11·12장). 현장에서 검증된 기존 로직과 같게 동작하는 것이 기준이다.
  - 나누는 기준: 가용(코어) = 장비가 지금 출력할 수 있는가라는 사실 (BMS 통신 끊김도 정상 BMS 판단에). 보호 = 장비 상태·배터리·모드를 바꿈 (현장마다 다름). 제한 = 크기만 줄이고 늘 돌아옴. 보호와 제한은 합치지 않는다 (모양이 다르고, 따로 있어야 찾기 쉬움). 같은 판단(등록 모드 적용)만 `F_ModeApplies`로 합침.
  - 보호 칸 = 조건 모듈 + 대응 모듈. 표준 대응은 `20_Module/Standard`(`FB_ProtRespStd`)에 두고 칸마다 기본 등록 (코어가 모듈을 직접 부르지 않게). 대응 칸을 비우면 알람만.
  - 지연 없음 (장비가 판정해 올림). 래치·풀린 뒤 유지 없음 (Auto 상위 지령과 부딪힘): 그대로 두려면 정지나 Manual 전환. 고장 리셋은 장비 명령일 뿐.
  - 표준 대응 조합: PCS None·Standby·Stop·EStop × 배터리 None·Stop·EStop, 배터리는 PCS Stop·EStop일 때만. 배터리 조치(비상정지 포함)는 PCS 정지 확인 뒤, PCS 통신 에러면 건너뜀. 확인 없이 배터리를 여는 동작은 두지 않음.
  - 목표 상태 없앰: Run = 활성, 대기 = 활성 + 출력 0 (회사 정의). 상태 명령은 일회성, 보호 명령만 N회 재시도 후 알람 (잠금 없음), 통신 없을 때 생긴 보호 명령은 복구 때 1회. 2026-10-19 결정 "SOC 한계는 실제 Standby로 보내고 자동 재개"는 이것으로 없앰 (`FB_SocStandby`·`FB_StandbyEval`·`I_StandbyRule` 제거, SOC는 `FB_SocLimit` 기본 등록).
  - 배터리: 투입 `Always` 없앰, 배터리 잠금 없앰 (새 기동·요청이 있어야 투입). 운전원 BatOpen도 PCS 정지 또는 통신 끊김이면 받음.
  - 보고: 장비가 보고한 상태를 매핑해 올림. PCS는 출력 0이면 대기로 통일하고, 대기 명령·부호별 명령이 필요한 장비 맵은 드라이버가 처리.
  - 바꾼 계약 (다른 담당에 알림): 모드 — 대기 요청 없음, 출력 0 확인(`fStopDonePct`)으로 전환 끝. MQTT — `ST_ProtCfg`의 `tDelay` 없앰·`fDeratePct` → `fPPct`·`fQPct`, 리셋은 장비별 요청(`stPcsReq`·`stBatReq` Reset), BMS 실패 알람(`PmsProt.BatLocked` 항목, 문구 "BMS {0} - {1}").
  - 내려받을 때: `ST_ProtCfg`·`ST_UnitProt`·`ST_ModuleSet`·`E_BatClose` 모양이 바뀌고 `g_astStandbyRuleReg`가 없어져 PERSISTENT 값이 일부만 되살아난다. Reset Origin 또는 재설정.
- [ ] 보호 남은 것: DO 한 점에 요청이 부딪힐 때의 규칙 (대응 요청에 DO를 넣을 때), 유지 DO 해제 방법 (지금은 지령원 PCS 리셋 요청과 같은 신호 `bClearHoldDo`), 새 장비 요청(`I_ReqDevice`) 코어 연결 (처음 쓰는 장비가 생길 때), 대응 모듈을 붙인 칸의 알람 조치 표시 (지금은 칸 설정 `stProt` 기준), BAT 리셋을 BMS 하나 단위로 (지금은 지령원 요청이 배터리 그룹 단위)
- [ ] MQTT 표준 인터페이스 확장: **구현됨** — 전송(`20_Module/Mqtt`), 명령 핸들러(`FB_EmsCommand`), EMS 지령원(`FB_SrcEms`), 명령 칸, 플라스포 PCS1 실값 매핑(`FB_RptPlaspoPcs` → MAIN `A_Report` → `GVL_IF.g_stReport` → `PRG_Mqtt`), 이벤트 로그 발행(`pms_log` 토픽, `20_Module/Mqtt/Logger`, 머지 뒤 빌드·시험 전). **남음** — 나머지 장치의 실값 매핑, config의 PLC 설정 적용, 배터리 비상정지 명령, 명령 수신 이벤트 로그. 시험 모드의 미매핑 레코드는 더미다. 보고 매핑은 장치별 모듈에 두고 현장 연결은 MAIN에서 구성하며, 보고 값은 제어에 사용하지 않는다 (R6). `I_Report` 기반 공통 계약과 코어 보고 재료 분리는 향후 설계 방향이다.
- [ ] 버스를 여러 그룹이 같이 쓸 때: 그룹 범위가 그룹마다 버스 전체 한도를 봄 (분배는 버스 남은 한도로 맞춤). 그룹 사이 배분 정책이 필요하면 정한다
- [ ] 모드 전환 때 Gate·SOC 정책 상태 인계 (Gate 교체 자리 `I_Gate`는 됨)
- [ ] 정책 인터페이스 (성형·전달·보호. 고르기·분배는 됨), 설정 확정(commit). 확장 장비(DCS 등)는 필요할 때
- [ ] GFM: 지금은 PCS에 맡김 (드라이버 설정). 표준 운영이 필요하면 `I_Pcs`에 그리드 모드 명령을 두고 Standby·Stop에서만 바꾸는 전환 단계. 독립 운전(계통 형성으로 부하를 받음)은 그룹 상태(연계/독립)로 따로 설계
- [ ] 중계 모듈 확장 지점: EMS 명령을 ESS가 아닌 장비(DCS 운전 명령·PV SCADA·OOS 등)에 그대로 넘기는 정해진 자리. `I_Relay` 같은 인터페이스와 등록 배열을 두고 MAIN ⑥ 뒤에서 부른다. 실제 전송은 그 장비 드라이버 `M_Cycle`. 지금은 현장마다 MAIN에 직접 넣어야 한다.
- [ ] 드라이버 쓰기 실패 연결: 기반(`M_MarkWriteResult`, 상태 `bWriteFail`, 로그)은 있고, 실제 드라이버(Plaspo `_bWriteError` 등)는 아직 연결하지 않음
- [ ] Plaspo Standby 판단: 장비 상태가 아니라 마지막 명령으로 추정함 (`_bStandbyMode`). 맵에 Standby 상태 비트가 있는지 확인 후 수정
- [ ] 이중화 (구현됨: 상대 판정, 상태 기계, 절체·복귀, 설정 동기화, 이벤트 로그, MQTT 구분. 7장 "이중화"). 남은 것:
  - 4회차 시험 (2026-11-05, CX5130 두 대)에서 확인함: 역할·기동, 절체(ADS 정지·전원 차단·Config 모드, 절체 약 0.5 s + 쓰기 1 s), 복귀(10 s), 50회 반복, EAP·접점 단선과 양보, 동기화, ClientId -P/-S, 발행 게이트(장치 토픽). 남은 시험은 아래
  - 시험: Manual 유닛·램프 시작값 측정(`DEVICE_WRITE = TRUE`, 시뮬레이터), MQTT 로거 `pms_log` 발행 게이트 (4024.17 이상 런타임 필요), 회사 브로커가 같은 계정 두 접속을 허용하는지, 핸드오버 타임아웃
  - 램프 기준을 승격 때 장비 현재 P로 맞출지 (측정 뒤 정한다)
  - 보조 DO: 끄는 쓰기 예외가 대기 PLC에도 걸리는 것 (대기 PLC는 보조 DO도 켜는 쓰기만 막는지 확인)
  - 장비 연결이 하나만 허용되는 현장 대응 (Standby 재연결 방식)
  - ~~스플릿 브레인을 장비 쪽 증거로 막을지~~ → 2026-11-05 결정: 막지 않고 현장 운영 규칙 (7장 이중화 알려진 한계)
  - 보조 DO 끄기 쓰기는 대기 PLC에도 허용되므로, 현장 출력 구성에 맞는지 확인한다.
- 결정 기록 (2026-10-24~27): 제어 모드. Auto 안의 운전 방식은 한 번에 하나만 켜는 제어 모드로 두고, 틀(관리자·인터페이스·기반 FB)은 모든 현장이 쓰므로 코어, 모드는 모듈로 둔다. 관리자는 지령원 하나라 코어 흐름(고르기·성형·분배)은 고치지 않았다. `I_Algorithm`·`ST_AlgoOut`은 제어 모드로 대신해 지웠다. Messis와 같은 계산은 같은 메서드·FB(`FB_CmTariffBase`, `FB_CmPeakCalc`)를 쓴다.
- 결정 기록 (2026-11-09, 통합시험 14번 탭 16번): 제어 모드를 접속점 그룹 단위로 바꿈 (접속점 그룹 안 Auto 유닛은 같은 모드, Manual 제외). 스탠다드 스케줄만 PCS별 값 유지. 아래 2026-10-27 결정을 대신함.
- 결정 기록 (2026-10-27, 2026-11-09에 바뀜): PCS별 제어 모드. PCS마다 관리자·모드·설정을 완전히 따로 둔다 (사용자 선택). 같은 접속점이면 피크컷·역송 방지가 겹칠 수 있어 `ePccShare`로 몫을 정한다. 제어 그룹은 없애지 않는다: 접속점 한도는 같은 접속점 유닛 합에 한 번만 걸어야 하고, 접속점 기준 외부 지령의 분배·고장 유닛 몫 재분배·그룹 보호가 그룹에 있기 때문이다. 접속점이 PCS마다 따로인 현장은 PCS마다 그룹 하나로 두면 그룹이 없는 것과 같다. 이름은 "제어 그룹" 그대로.
- 남겨 둔 것 (의도): `I_AuxDevice.M_SetValue`(AO, 쓸 때 `FB_AuxOutput` 경로로), `ST_GroupView`의 안 쓰는 칸(모듈 약속), `FB_PersistSave` HMI 입출력
- 결정 기록 (2026-10-19): 한 FB 한 역할, 같은 계산은 한 곳(가용 한도·출력 가능·모듈 목록). 설정 오류 때 보조 DO는 켜는 쓰기만 막고 끄는 쓰기는 함 (2026-10-21 바꿈: 차단기 코일 펄스가 켜진 채 남는 문제). Manual 정책 `bRating`·`bBatLimit`(물리 한도 끄기)는 유지. 유닛·버스 제한 모듈의 Manual 적용은 모듈 등록 `bAuto`·`bManual`로 정함. SOC는 `FB_SocLimit` 하나 (2026-10-26). 버스 한도 초과분은 분배에서 재분배

---

## 11. 자주 받는 질문 (AI 답변 지침)

사람이 AI에게 PmsStd를 물어볼 때 실제로 나왔던 질문과, 답할 때 기준이 되는 내용이다. 비슷한 질문이 오면 여기 답을 출발점으로 삼는다.

### 답하는 방법

- **먼저 코드를 확인한다.** 이 문서는 요약이다. 문서와 코드가 다르면 코드가 맞고, 문서 수정을 제안한다.
- **결론부터 말한다.** "된다 / 안 된다 / 일부만 된다"를 먼저 쓰고, 방법은 다음 순서로 구분한다: ① 설정만으로 됨(`90_Project`) → ② 모듈 추가(`20_Module`) → ③ 코어 변경(`10_Core`). ③이면 2장 원칙에 어긋나지 않는지도 말한다.
- **FB 이름과 온라인 변수 경로를 같이 쓴다** (예: `fbSequencer.q_aeRefuse[유닛]`). 사람이 바로 찾아볼 수 있게 한다.
- **숫자 예시를 든다** (예: "그룹 1500 kW → 배터리 한도 1200 → 유닛 600 / 600").
- 헷갈리기 쉬운 두 개념을 물으면 표로 나란히 비교한다 (하는 일, 다루는 값, 부르는 곳).
- 질문만 받았으면 답만 한다. 코드는 "수정해"라고 할 때만 고친다 (0장).

### 구조와 역할

**Q. `FB_Protection`은 무엇을 하나?**
보호 칸마다 조건 모듈(`I_ProtectionRule`)이 판단하고, 같은 칸 대응 모듈(`I_ProtResponse`, 기본 표준 대응 `FB_ProtRespStd`)이 유닛·BMS에 **요청**(PCS: None/Standby/Stop/EStop, 배터리: None/Stop/EStop, Manual 전환, P·Q 상한 %)을 낸다. `FB_Protection`은 이것을 장비마다 합친다. 장비에 직접 명령하지 않는다: PCS Standby는 가용 아님으로 출력 0, Stop·EStop은 시퀀서가 명령, 배터리 조치는 시퀀서가 버스 PCS 정지 뒤 명령, P·Q 상한은 `FB_EquipmentLimits`가 유닛 한도에 넣는다. 기본 5종 조건과 표준 대응은 1~5칸에 기본 등록돼 있고, 현장 보호는 그 뒤 칸이다. 설정은 같은 칸의 `g_astRuleCfg`다 (적용 운전 모드 `bAuto`·`bManual`, 대상 범위 `eScope`, 표준 대응 조치). 지연은 없다.

**Q. Manual Gate(`FB_GateManual`)가 예전 "Manual 보호", Auto Gate(`FB_GateAuto`)가 "Auto 보호" 같은 것인가?**
아니다. Gate는 **지령 값 거르기**이고 보호가 아니다. `FB_GateManual`은 Manual 지령 중 방향 반대 새 지령을 무시하고, Run 뒤 새 지령을 기다리게 한다. `FB_GateAuto`는 기본으로 그대로 통과시킨다. 바깥 FB는 `FB_Gate` 하나를 경로만 바꿔 두 번 쓰고, 경로마다 다른 동작은 안쪽 Gate 모듈이 갖는다. 둘 다 Gate 모듈(`I_Gate`)을 넣어 현장 규칙으로 바꿀 수 있다. 보호는 Auto·Manual 모두 `FB_Protection` 하나가 하고, 항목마다 `bAuto`·`bManual`로 어느 모드 유닛에 걸지 정한다.

**Q. `FB_UnitAvail`(가용)과 `FB_OutputReady`(출력 가능)는 왜 따로인가? 가용에서 이미 판단해야 하지 않나?**
묻는 것이 다르다.

| | 가용 (`FB_UnitAvail`) | 출력 가능 (`FB_OutputReady`) |
|---|---|---|
| 질문 | 이 장비를 지금 쓸 수 있나 | 지금 이 유닛에 kW를 줘도 되나 |
| 보는 것 | 통신, 고장, 로컬, 배터리, 보호 조치 | 가용 + 장비 활성(Run·대기) + 정지 진행 중 아님 + 모드 전환 아님 + Manual 새 지령 대기 아님 |
| 쓰는 곳 | 기동 접수 검사(`NotAvail` 거부), 설비 범위(가용 유닛만 합), 그룹 요약 | 성형(램프 기준), 분배(대상), 디스패처(Manual 쓰기 시점) |

멈춘(Stop) 유닛은 가용이지만 출력 가능은 아니다. 가용이어야 Run 요청을 받을 수 있으므로, 가용 판단에 "활성"을 넣으면 기동 자체가 안 된다.

**Q. `FB_ShapePath`(성형)가 분배를 하는 것 아닌가?**
아니다. 성형은 **그룹이 총 얼마**를 낼지(경로마다 값 하나) 정하고, 분배(`FB_Distributor`)는 **누가 얼마씩** 낼지 정한다. 성형도 유닛별 지령을 다루지만 램프만 걸 뿐 나누지 않는다. 한도도 성형은 그룹 합계 한도(가용 정격 합, 버스별 배터리 합), 분배는 유닛 하나하나의 한도와 버스 남은 한도를 본다. 두 곳 모두 `FB_EquipmentLimits` 결과를 쓰므로 같은 계산을 두 번 하지 않는다.

**Q. 분배는 한 유닛에 문제가 생겼을 때 다른 유닛이 받게 하려고 있는 것인가?**
그것도 하지만 그것만은 아니다. 분배는 매 스캔 그룹 몫을 `eDistBasis`(정격 비율 / SOC / 균등)로 나누는 것이 기본 일이다. 한 유닛이 가용을 잃거나 한도에 걸리면 그 몫을 남은 유닛에 **재분배**하는 것이 그 안에 들어 있다. 그룹 지령 없이 유닛별 지령만 있으면 나눌 몫이 없어 유닛 값만 한도 안으로 자른다.

**Q. ④ 판단 단계에서는 무엇을 하나?**
두 갈래다. ① 상태(Run·Stop 등): `fbSequencer.M_Prepare`(조건 평가) → `fbStateArb`(요청 후보, 같은 스캔은 정지 쪽) → `fbSequencer.M_CheckRequest`(기동 쪽 접수 검사) → `fbStateArb.M_CommitRequest`(이번 스캔 요청) → 시퀀서가 한 번 보냄. 저장하는 목표는 없다. ② 출력 값(kW): `FB_Select`(지령원 고르기) → Gate(거르기) → `FB_OutputReady` → `FB_LimitEval`(제한·보정 모듈) → `fbManualShape` → `fbAutoShape`(성형). 3장 "한 스캔"과 7장 "상태 요청"에 자세히 있다.

**Q. 한 그룹에 Auto 유닛과 Manual 유닛이 섞이면? 다 Auto이거나 다 Manual이면?**
두 경로를 다 돈다. Manual을 먼저 계산하고, Auto는 Manual 결과를 그룹 전체에 포함해 접속점·설비 제한을 본 뒤 Manual 몫을 뺀다 (Manual 우선, 깎이는 양은 Auto 부담). 모두 같은 모드면 다른 경로는 유닛이 없어 진단에 `NoUnit`(정상)으로 나오고 결과는 0이라 영향이 없다.

### 상태와 Standby

**Q. Standby와 Stop은 무엇이 다른가?**
Run = 활성(출력 지령을 받음, 크기 무관), Standby = 대기(활성인데 출력 0), Stop = 정지. 대기는 따로 명령하는 상태가 아니라 출력이 0인 활성이다 (드라이버가 장비 맵에 맞게 씀). 보호 조치로 보면 Standby(출력 0)는 조건이 풀리면 다시 출력하고, Stop·EStop은 조건이 풀려도 정지로 남아 Run 요청이 있어야 다시 돈다 (`E_PcsAction`).

**Q. 운전원이 Standby를 누르면?**
멈춘 유닛이면 활성으로 기동한다 (Run 1회, 출력 0). 이미 활성이면 할 일이 없다: 출력을 0으로 하려면 그 지령원 출력을 0으로 둔다 (대기 = 출력 0). 장비를 멈추려면 Stop. 보호 뒤 그대로 두려면 정지나 Manual 전환을 쓴다 (7장 "보호").

**Q. SOC 한계로 출력이 0이 되면 자동으로 다시 나가나?**
된다. SOC 한계는 제한(`FB_SocLimit`)이라 장비 명령 없이 그 방향 한도만 0이 되고(= 대기), SOC가 재개 마진(기본 2 %)만큼 돌아오면 다시 허용한다 (재개 램프 `fRampResume`). 다시 나가지 않게 하려면 보호로 붙인다 (조건 SOC 한계 + Manual 전환 또는 정지). 7장 "설비 한도와 SOC 정책".

**Q. 통신이 끊겼을 때 Standby로 할지 Stop으로 할지 정할 수 있나?**
된다, 설정만으로. `GVL_PERSIST.g_astRuleCfg[2]`(PCS 통신 끊김, 기본 Standby)와 `[4]`(BMS 통신 끊김 → 같은 버스 유닛, 기본 Standby)의 `stProt.ePcsAction`을 None / Standby / Stop / EStop 중에서 고르고, `bAuto`·`bManual`로 적용 모드를 정한다. 통신이 끊긴 PCS는 가용 아님이라 출력에서 빠지고, 명령은 통신이 돌아오면 1회 나간다 (정지로 두면 복구 때 Stop 1회). 판정 시간(지연)은 드라이버의 `i_tCommTimeout`(기본 10 s)이다. 이보다 복잡한 대응(대기 뒤 N초 지나면 정지 등)은 대응 모듈로 붙인다.

**Q. 통신 에러 보호를 빼 달라고 하면?**
된다. 그 현장에서 아예 안 쓰면 `GVL_SITE.g_aipRule`에서 `g_fbProtPcsComm`(또는 `g_fbProtBmsComm`) 칸을 비우고 재빌드한다. 운전 중 잠깐 끄려면 `g_astRuleCfg[2]`(또는 `[4]`)의 `stProt.bAuto`·`bManual`을 FALSE로 둔다. 알람과 대기 조치가 없어진다. 다만 통신이 끊긴 PCS는 명령을 받을 수 없어 출력에서는 빠진다 (다른 유닛은 계속 돈다).

**Q. Auto 모드에서 운전원이 Stop을 누를 수 있나?**
기본은 안 된다. 지령원 상태 요청은 `g_astSourceReg[n].stStateMode`의 명령별 모드를 따르고 기본은 모두 `Manual`이다. 운전원 지령원의 `stPcs.eStop`·`eEStop`(필요하면 `eStandby`, `stBat.eOpen`)을 `Both`로 바꾸면 Auto에서도 받는다. 보호와 코어 요청은 이 설정과 관계없이 항상 동작한다.

**Q. Manual로 지령을 넣었는데 출력이 안 나간다.**
흔한 이유 순서: ① 유닛이 Run이 아님(Manual은 Run 요청을 따로 해야 함) → ② `g_stManualCmdSet.bClearOnStop`: Run을 확인한 뒤 들어온 지령만 쓴다. Run 전에 넣은 값은 버려지므로 Run 뒤 다시 보낸다 (`fbManualGate.q_abManualWaitNewCmd`, 분배 이유 `WaitManualCmd`) → ③ `bReverse`: 지금 출력 방향과 반대인 새 지령은 무시된다 (먼저 0 근처로) → ④ 한도(`fbOutputDiag.q_astManualPDiag`).

**Q. PMS를 재시작하면 돌고 있던 장비가 멈추나?**
아니다. 시작 때 장비에 명령을 스스로 보내지 않는다. 기본으로 모든 유닛이 Manual로 시작하고(`MODE_SET.bKeepOnRestart`), Manual 유닛에는 새 지령이 올 때까지 값을 쓰지 않으므로 장비는 이전 출력을 유지한다. `bKeepOnRestart`로 Auto를 유지하면 활성인 Auto 유닛에는 Auto 지령을 그대로 쓴다. 정전 뒤 장비 투입은 운전원이 한다.

### 이중화

**Q. 이중화를 켜려면?**
`GVL_SITE.RED_SET`에 `bUse := TRUE`와 두 PLC의 AMS Net ID를 쓰고, 같은 프로그램을 두 PLC에 넣고, 접점 배선과 XAE의 EAP 두 쌍·I/O 연결을 설정한다 (7장 "이중화" 설치). 역할은 PLC의 Net ID로 정해지므로 PLC마다 따로 빌드하지 않는다.

**Q. Primary가 멈추면 얼마 만에 Secondary가 쓰나?**
상대가 `Lost`로 판정된 뒤 `tTakeoverDelay`(500 ms)를 더 기다려 Active가 된다. 실측은 정지 후 약 1 s, 쓰기 허가는 `tWriteHold`(1 s) 뒤라 약 2.3 s다.

**Q. Primary가 돌아오면 바로 되돌아가나?**
아니다. Primary는 Standby로 들어와 Secondary(Active)의 설정을 받고, 통신과 동기화가 10 s(`tFailbackStable`) 이어지면 `Handover`로 넘겨받는다. 실측 약 10 s. Secondary가 Active인 동안 바꾼 설정은 복귀 뒤에도 남는다.

**Q. EAP 케이블만 빠지면 절체되나?**
안 된다. 접점이 살아 있으면 `EapLost`(통신 고장)로 보고 절체하지 않는다. EAP와 접점이 둘 다 끊겨야 `Lost`다.

**Q. 두 PLC가 동시에 장비에 쓸 수 있나?**
정상 동작에서는 없다. 복귀 때는 Secondary가 내려놓은 뒤 Primary가 올라가 잠깐 아무도 쓰지 않는 구간이 있다. 예외는 EAP와 접점이 둘 다 끊긴 스플릿 브레인이다 (7장 한계).

**Q. HMI가 설정을 한 PLC에만 쓰면?**
운영 설정(`GVL_PERSIST`, 모듈 설정 묶음 제외)은 Active에서 Standby로 맞춰진다. 모듈 설정 묶음(SOC 제한, 스케줄 시간표 등)은 칸이 현장마다 달라 표준이 동기화하지 않는다. 현장이 7장 "현장 모듈 설정 동기화"를 더했으면 Active에서 Standby로 맞춰지고, 더하지 않았으면 HMI가 두 PLC 모두에 써야 한다. Manual 지령은 어느 쪽이든 동기화하지 않으므로 HMI가 두 PLC 모두에 쓴다.

**Q. XAE에서 PLC를 Stop했다가 Run하면?**
이중화를 쓸 때는 멈춘 시간이 1 s를 넘으면 상태를 `Init`부터 다시 정한다 (`g_nRedResumeCount`가 올라가고 이벤트 로그 "멈췄다 이어짐"). Secondary가 이미 Active이면 Primary는 Standby로 들어와 복귀 절차를 밟는다.

**Q. MQTT 명령은 어느 PLC가 받나?**
두 PLC 모두 구독하고, 장비 쓰기는 Active만 한다. 발행도 Active만 한다. ClientId에 `-P`/`-S`가 붙어 브로커가 두 접속을 구분한다.

### 구성과 설정

**Q. 제어 모드(스케줄·피크컷 등)는 어떻게 고르나? 여러 개를 같이 켤 수 있나?**
관리자 하나 안에서는 하나만 켠다. `GVL_SITE.g_fbCtrlModeN.i_nModeReq`에 칸 번호(1 정출력, 2 스케줄, 3 피크컷, 4 통합, 5 스탠다드 스케줄, 0 없음)를 쓰면 진입 검사 → 이전 모드 끝 → 1 s 출력 0 → 새 모드로 바뀐다. 거부되면 지금 모드를 유지하고 `q_eRefuse`·`q_nRefuseReason`에 이유가 남는다. 스케줄과 피크컷을 같이 쓰려면 통합 모드(4)를 고른다. 관리자는 Auto 유닛에만 들어간다 (7장 "제어 모드").

**Q. 스케줄은 어디에 넣나?**
`GVL_PERSIST.g_stModuleSet.stCmScheduleN.astSlot[계절, 요일, 칸]` (N = PCS 번호). 계절 0 여름·1 봄가을·2 겨울(월 → 계절은 `stCmCommon.anSeasonOfMonth`), 요일 0 = 일요일, 칸 1~10. 한 칸은 `nStart`·`nEnd`(HHMM, 끝 미포함, 2400 = 자정까지)·`fP`(kW, 방전 +, 충전 −, Messis와 부호 반대). 방전은 방전 허용 시간(기본 08~22시, 일요일 금지)과 역송 방지로 줄어든다. 휴일은 `stCmCommon.abHoliday[월, 일]`.

**Q. PCS마다 다른 모드를 쓸 수 있나?**
코어는 된다 (관리자 `i_nUnit := n`, 유닛 단위). 하지만 표준 현장 구성은 2026-11-09부터 접속점 그룹 단위다 (통합시험 14번 탭 16번: 접속점 그룹 안 Auto 유닛은 같은 모드, Manual 제외, 스탠다드 스케줄만 PCS별 값). PCS마다 다르게 돌려야 하는 현장은 접속점이 따로면 PCS마다 접속점 그룹을 두고, 같은 접속점이면 유닛 단위 관리자를 현장에서 추가한다 (`ePccShare`로 부하 몫).

**Q. 제어 그룹을 없애고 PCS마다 모드만 두면 안 되나?**
권하지 않는다. 그룹은 "이 PCS들이 어느 접속점을 같이 쓰나"라는 전기적 사실이다. 접속점 한도(역송 방지·계약전력)는 같은 접속점 유닛 합에 한 번만 걸어야 하고, 접속점 기준 외부 지령을 나눌 곳, 고장 PCS 몫을 남은 PCS가 채우는 분배, "접속점 계측 이상 → 그룹 대기" 보호가 그룹에 있다. 접속점이 PCS마다 따로인 현장은 PCS마다 그룹 하나로 설정하면 그룹이 없는 것과 똑같이 동작한다.

**Q. PCS 두 대를 유닛 하나로 묶을 수 있나?**
안 된다. `UNIT_CFG`의 유닛은 PCS 번호(`nPcs`) 하나만 가진다. 방법은 두 가지다. ① 두 PCS를 유닛 2개로 두고 같은 제어 그룹에 넣는다 (그룹 지령을 분배가 나눔, 보통 이것으로 충분). ② 두 PCS를 묶어 하나처럼 보이는 드라이버(`I_Pcs` 하나)를 만든다 (장비 쪽에서 하나로 다뤄야 할 때). 반대로 PCS 한 대가 출력을 둘로 나눠 받으면 `nSub`로 유닛 2개를 만든다.

**Q. PCS·배터리 없이 모니터링하고 VCB만 제어할 수 있나?**
된다. 대수는 0으로 둘 수 없어 1로 두고 PCS·BMS 칸을 비우면 미설치로 보고, 통신 경보·보호·정격 점검에서 뺀다 (설정만으로 됨). 조건에 따른 VCB 자동 개방은 출력 규칙, 운전원 투입·개방과 실패 감시는 공통 모듈 `FB_BreakerCtrl`로 붙인다. 계측기가 차단기 제어를 겸하면 드라이버 하나를 `I_Meter`·`I_AuxDevice` 두 배열에 등록한다. 6장 "자주 하는 작업"에 자세히 있다.

**Q. 운전원이 유닛마다 출력 한도(kW)를 정하게 하려면?**
공통 모듈 `FB_UnitLimit`을 쓴다. `GVL_SITE` 주석 예 `g_fbUnitLimit`을 풀어 `g_aipUnitRule` 한 칸에 넣고(`GVL_PARAM.UNIT_RULE_COUNT` 맞춤), `g_astUnitRuleReg` 그 칸에 `nTarget := 0`(전체 유닛)과 적용 모드(`bAuto`·`bManual`)를 준다. 설정은 `ST_ModuleSet`에 `stUnitLimit : ST_UnitLimitSet` 칸을 풀고 `A_PassModuleSet`에 한 줄을 넣는다. HMI는 `GVL_PERSIST.g_stModuleSet.stUnitLimit.astUnit[n]`의 `bPEnable`·`fPMax`(방전 상한, 0 이상)·`fPMin`(충전 하한, 0 이하, kW)을 바꾼다. Q도 같다(kvar). 잘못된 값(상한 < 0, 하한 > 0)은 받지 않고 마지막 정상값으로 돌아간다. 걸리면 유닛 한도 이유가 `UnitRule`, 모듈 이유가 "사용자 한도"다.

**Q. 상위(EMS)가 무효전력 상·하한을 보내면?**
지령원이 `ST_SourceOutput`의 `bQLimActive`·`fQMax`·`fQMin`(kvar, 접속점 기준)을 채운다. P 제한과 같이 모든 지령원 것이 겹치고, Manual에는 정책 `bSrcLimit`이 켜졌을 때만 쓴다. 지령원이 끊겨도 마지막 제한은 유지된다.

**Q. 김천처럼 알고리즘 여러 개가 동시에 경합하면 어디에 넣나? (한 번에 하나만 켜는 운전 방식이면 제어 모드)**
알고리즘마다 지령원(`I_CommandSource`) 하나로 `g_aipSource`에 넣고, `g_astSourceReg`의 등급(`nGrade`, 1이 가장 높음)으로 우선순위를 준다. 그룹마다 `FB_Select`가 고른다 (`eSelP`: `Grade` 또는 `MinAbs`). 규칙이 더 복잡하면 고르기 정책 모듈(`I_SelectPolicy`)을 만들어 `g_aipSelectPolicy[그룹]`에 넣는다 (코어 수정 없음). 6장 "자주 하는 작업".

**Q. 설정이 틀리면 어떻게 되나?**
`FB_ConfigCheck`가 매 스캔 점검해서 오류가 있으면 `q_bConfigOk = FALSE`가 된다. PCS·BMS에는 아무것도 쓰지 않고, 보조 DO는 켜는 쓰기만 멈춘다 (끄는 쓰기는 한다). 읽기·보호 판단·진단·로그는 계속된다. 기동 요청은 `ConfigError`로 거부된다. 장비 쓰기 스위치(`GVL_SITE.DEVICE_WRITE`)가 꺼져 있거나 이 PLC가 대기(Standby)일 때도 시퀀서는 명령을 만들지 않고 기동 요청을 `WriteDisabled`로 거부한다. 오류 문구는 최대 10건 로그에 남고 "설정 오류로 장비 제어 중지" 알람이 뜬다. 다만 운전 중 HMI로 바꾸는 값(모듈 설정·명령 필터·Manual 지령 거르기·시퀀스)을 잘못 넣으면(예: SOC 하한 ≥ 상한, 재시도 간격 0) 설정 오류로 멈추지 않는다. `FB_SettingGuard`가 그 값을 받지 않고 마지막 정상값을 다시 써 넣고(화면에도 정상값이 보임), "잘못된 설정 값을 받지 않음 (마지막 정상값 유지): SocLimit: SOC 상·하한 …"을 기록한다. 운전은 계속된다. 저장된 값이 PMS 시작부터 잘못돼 되돌릴 정상값이 없을 때만 설정 오류가 된다.

**Q. 리셋은 무엇을 하나?**
장비에 고장 리셋 명령을 보낼 뿐이다. 지령원 PCS 리셋(`stPcsReq.eCmd = Reset`, `nUnit`)은 그 PCS에, 배터리 리셋(`stBatReq.eCmd = Reset`, `nBatGroup`)은 그 배터리 그룹 BMS에, 장비가 고장일 때 1회 간다. PMS 상태는 바꾸지 않는다 (PMS 쪽 잠금은 없다: 명령 실패·BMS 실패는 알람만). 보호는 리셋이 없다 (조건이 풀리면 저절로 풀림). 시험 때는 `GVL_TEST.g_bTestReset`(`TEST_MODE`만, 모든 PCS·BMS). 리셋해도 저절로 기동하지 않는다. 유지 DO 풀기는 지금 지령원 PCS 리셋 요청과 같은 신호를 쓴다 (MAIN `bClearHoldDo`, 따로 정하는 것은 10장).

**Q. SOC 감발을 Auto나 Manual 한쪽에만 쓰려면?**
기본은 `g_fbSocLimit` 하나가 모든 버스에 Auto·Manual 둘 다 걸린다. 한쪽만 쓰려면 `GVL_PERSIST.g_astBatRuleReg[1]`의 `bAuto`·`bManual` 중 하나를 끈다 (운전 중 변경). 모드마다 다른 한계를 쓰려면 인스턴스를 하나 더 두고(`g_aipBatRule` 칸·`BAT_RULE_COUNT` 늘림), `ST_ModuleSet`에 설정 칸과 `A_PassModuleSet`에 한 줄을 더한다.

**Q. 같은 배터리 버스에서 한 유닛은 충전, 한 유닛은 방전할 수 있나?**
안 된다. 같은 버스 유닛은 같은 방향만 낸다. 반대 방향 유닛은 0이 된다 (`E_UnitLimitBy.BatDir`).

**Q. GFM/GFL은 Auto/Manual처럼 따로 갈라야 하나?**
아니다. Auto/Manual은 "누가 시키나"(지령원)라서 PMS 계산이 달라지지만, GFL/GFM은 "PCS가 어떻게 따라가나"라서 계통 연계 중이면 PMS 계산이 같다 (GFM PCS도 P·Q 지령을 받고, 전압·주파수 드룹은 PCS가 처리). 그래서 경로를 늘리지 않고 유닛 속성(`ST_UnitState.eGridMode`, 드라이버가 채움, 기본 GFL)으로 둔다. GFM 운전이 필요한 현장은 드라이버 설정으로 PCS에 맡긴다. 독립 운전은 그룹 상태(연계/독립)로 따로 설계할 일이다 (10장).

**Q. 램프가 안 걸린다.**
그룹 설정 `g_astGroupSet[그룹].bRamp` 기본이 FALSE다. 켜고 `fRampUp`·`fRampDn`(kW/s)을 준다. Manual은 정책 `g_astManualPolicy[그룹].bRamp`도 TRUE여야 한다.

---

## 12. 데이터 참조 (인터페이스·구조체)

모듈을 만들거나 "이 값은 어디서 오나"를 답할 때 보는 표다. 단위: P kW(방전·공급 +), Q kvar(공급 +), SOC %. `ST_Val`은 `fValue` + `bValid`(FALSE면 값을 쓰지 않음)다. 코드가 바뀌면 이 장도 같이 고친다.

### 인터페이스 메서드

| 인터페이스 | 메서드·속성 | 코어가 넘기는 것 → 돌려받는 것 |
|---|---|---|
| `I_Module` | `ModuleId : UINT`, `ModuleName : STRING(31)`, `M_ReasonText(i_nReason) : WSTRING(63)`, `M_CheckSet() : WSTRING(63)` | 이유 코드 → 로그 문구 (기본 "이유 n") |
| `I_Pcs` | `M_Cycle()`, `M_GetUnitState(i_nSub) : ST_UnitState`, `M_SetPower(i_nSub, i_fP, i_fQ) : BOOL`, `M_SetStateCmd(i_nSub, i_stCmd : ST_PcsCmd) : BOOL` | PCS 안 유닛 번호마다 상태 읽기, P·Q 쓰기, 상태 명령 |
| `I_Bms` | `M_Cycle()`, `BatState : ST_BatState`, `M_SetStateCmd(i_stCmd : ST_BatCmd) : BOOL` | 상태 읽기, Close/Open/EStop/Reset |
| `I_Meter` | `M_Cycle()`, `MeterState : ST_MeterState` | 계측 읽기 |
| `I_AuxDevice` | `M_Cycle()`, `Comm : BOOL`, `M_GetBit(i_nPoint)`, `M_GetValue(i_nPoint) : ST_Val`, `M_SetBit(i_nPoint, i_bValue)`, `M_SetValue(i_nPoint, i_fValue)` | DI·AI 읽기, DO 쓰기(`FB_AuxOutput`만), AO 쓰기(아직 쓰는 곳 없음) |
| `I_CommandSource` | `M_Cycle(i_stReg : ST_SourceReg, i_stView : ST_GroupView)`, `Output : ST_SourceOutput` | 등록 한 줄 + 그룹 요약 → 지령 출력. `FB_SourceBase`를 상속하면 `M_UpdateOutput`만 구현 |
| `I_ControlMode` (`EXTENDS I_Module`) | `M_CheckEnter(i_stView : ST_CtrlModeView) : UINT`, `M_Enter(i_stView)`, `M_Calc(i_stView) : ST_CtrlModeOut`, `M_Exit()` | 진입 검사(0 = 됨, 그 외 이유 코드. 상태를 바꾸지 않음), 켜질 때 한 번, 켜진 동안 매 스캔, 꺼질 때 한 번. 관리자가 부름 |
| `I_LimitRule` | `Basis : E_LimitBasis` (`Pcc` 접속점 기준·보정 전 / `Device` 설비 기준·보정 후), `M_Evaluate(i_stView : ST_GroupView) : ST_LimitOut` | 그룹 P·Q 상·하한 |
| `I_Compensator` | `M_Calc(i_stView : ST_GroupView) : ST_PQ` | 목표에 더할 보정량 |
| `I_StartPermit` | `M_Evaluate(i_stView : ST_GroupView, i_bRequest : BOOL) : BOOL`, `Reason : UINT` | TRUE = 기동 허가. `i_bRequest` = 그룹이 기동을 기다리는 중 |
| `I_ProtectionRule` | `M_Evaluate(i_stView : ST_ProtView) : BOOL` | 대상 하나의 정보 → TRUE = 보호 조건. 조치는 같은 칸 대응. 대상 번호 0이면 대상마다 부름 |
| `I_ProtResponse` | `M_Respond(i_bActive, i_tActive, i_nTarget, i_stRule : ST_RuleCfg; IN_OUT CONSTANT iq_astUnit, iq_astBms, iq_astUnitCfg, iq_astBmsCfg, iq_aeOpMode; IN_OUT iq_stReq : ST_ProtReq)` | 보호 대응: 무엇을, 누구에게 요청할지 `iq_stReq`에 씀 (코어가 비워서 넘기고 장비마다 합침). 범위 안 대상마다 매 스캔 |
| `I_ReqDevice` | `M_CmdRank(i_nCmd) : INT`, `M_Request(i_nCmd, i_nOrigin)` | 요청을 받는 새 장비 (모양만). 세기·명령·확인은 장비 모듈 |
| `I_OutputRule` | `M_Check(i_fThreshold : REAL) : BOOL` | TRUE = DO 켬. 지연·유지는 코어(`g_astOutputCfg`) |
| `I_UnitLimitRule` | `M_Evaluate(i_stView : ST_UnitLimitView) : ST_LimitOut` | 유닛 P·Q 한도 (좁히기만, 0은 항상 허용) |
| `I_BatLimitRule` | `M_Evaluate(i_stView : ST_BatLimitView) : ST_BatLimitOut` | 버스 충·방전 한도 (좁히기만) |
| `I_SelectPolicy` | `M_Select(i_stView : ST_SelectView) : ST_Pick` | 후보(`nCand`, `astCand[1..nCand]`: `nSource`·`nModule`·`nGrade`·`fValue`·`bNew`)와 `nGroup`·`nUnit`(0 = 그룹 값)·`bForQ`·`eOpMode`·`eSel` → 값 하나. 상태를 바꾸지 않음 |
| `I_Gate` | `M_Filter(i_stView : ST_GateView, iq_astUnitP·iq_astUnitQ·iq_astGrpP·iq_astGrpQ : ST_Pick 배열, iq_aeOpMode, iq_astUnit, iq_astUnitCfg, iq_stOut : ST_GateOut)` | 경로마다 스캔당 한 번. `ST_GateView`: `ePath`, `stManualCmdSet`. `ST_GateOut`: 거른 `astUnitP/Q`·`astGrpP/Q`, `abNew`, `aeRefuse`, `anRefuseModule`, `abWaitNewCmd` (지난 스캔 값이 남아 있어 보류는 그대로 두면 됨) |
| `I_DistPolicy` | `M_Weight(i_stView : ST_DistView) : REAL` | `nGroup`·`nUnit`·`bManual`·`bForQ`·`bPositive`·`eBasis`·`fPRated`·`fQRated`·`fSoc`(-1 = 모름) → 무게 (0 이상). 상태를 바꾸지 않음 |

### 구성 (`GVL_SITE`, 재빌드)

| 구조체 | 칸 |
|---|---|
| `ST_UnitCfg` (`UNIT_CFG[유닛]`) | `nPcs`(PCS 칸), `nSub`(PCS 안 번호, 1..`MAX_SUB`), `nBatGroup`(버스, 0 = 배터리 없음), `nGroup`(제어 그룹), `fPRated` kW, `fQRated` kvar, `fSRated` kVA |
| `ST_BmsCfg` (`BMS_CFG[BMS]`) | `nBatGroup`, `fCapacity` kWh (SOC 가중 평균 무게, 0이면 1) |
| `ST_PointCfg` (`POINT_CFG[계측점]`) | `astTerm[1..POINT_TERM_MAX]` = (`nMeter` 계측기 칸 0 = 빈 칸, `fSign` +1/−1). 전압·주파수는 첫 계측기 값 |
| `ST_GroupCfg` (`GROUP_CFG[그룹]`) | `nPccPoint` 접속점 계측점 (0 = 없음) |
| `ST_RedSet` (`RED_SET`) | `bUse`(기본 FALSE), `sPrimaryNetId`, `sSecondaryNetId` (AMS Net ID 문자열) |

### 장비 상태 (드라이버가 채움)

| 구조체 | 칸 |
|---|---|
| `ST_UnitState` | `bComm`, `eRunState`(Unknown/Stop/Standby/Run/Fault), `bCtrlReady`(리모트, 기본 TRUE), `bFault`, `bWriteFail`(쓰기 잇달아 실패), `stP`, `stQ`, `eGridMode`(Unknown/Gfl/Gfm, 기본 Gfl, 화면·로그용) |
| `ST_BatState` | `bComm`, `bContactorClosed`, `bWarn`, `bFault`, `bSysStop`(배터리 스스로 정지), `stSoc`, `stSoh`, `fPChgMax`·`fPDisMax`(양수, 통신 없으면 0으로 봄), `bWriteFail` |
| `ST_MeterState` | `bComm`, `stP`, `stQ`, `stV`(상전압 평균), `astV[1..3]`, `stF` |

### 지령원 출력 (`ST_SourceOutput`)

| 칸 | 뜻 |
|---|---|
| `bValid` | 사용 가능한 출력 또는 유지할 제한이 있음. FALSE면 나머지 무시. 만료 뒤에도 유지할 P·Q 제한이 있으면 TRUE를 유지하고 지령·상태 요청만 비운다 |
| `bPSetActive`, `fPSet` | 그룹 P 지령 (접속점 기준 = 그룹 전체 값) |
| `bPLimActive`, `fPMax`, `fPMin` | 지령원 P 제한 (모든 지령원 것이 겹침) |
| `bQLimActive`, `fQMax`, `fQMin` | 지령원 Q 제한 (kvar, 접속점 기준, 모든 지령원 것이 겹침) |
| `bQSetActive`, `fQSet` | 그룹 Q 지령 |
| `bNewSet` | 새 지령을 보낸 스캔만 TRUE (같은 값을 다시 보내도 TRUE). Manual 즉시 쓰기에 씀 |
| `stPcsReq` (`eCmd : E_PcsCmd`, `nUnit` 0 = 그룹 전체) | 상태 요청 Run/Standby/Stop/EStop, 고장 리셋(Reset: 그 PCS에만). 새 요청일 때 1스캔 |
| `stBatReq` (`eCmd : E_BatCmd`, `nBatGroup` 0 = 전체) | 배터리 요청 Close/Open, 고장 리셋(Reset: 그 배터리 그룹 BMS). 새 요청일 때 1스캔 |
| `astUnit[유닛]` (`bPSetActive`, `fPSet`, `bQSetActive`, `fQSet`) | 유닛별 값. 있으면 그 유닛은 그룹 몫 대신 이 값 |

### 등록·운영 설정 (`GVL_PERSIST`, 운전 중 변경)

| 구조체 | 칸 |
|---|---|
| `ST_SourceReg` (`g_astSourceReg`) | `bEnable`, `nGrade`(기본 9, 1이 가장 높음), `stStateMode`, `ePowerMode`(기본 Auto), `nGroup`(0 = 모든 그룹), `tExpire`(기본 10 s) |
| `ST_StateModeSet` | `stPcs`(`eRun`, `eStandby`, `eStop`, `eEStop`), `stBat`(`eClose`, `eOpen`). 값은 `E_SourceMode` Auto/Manual/Both/None, 기본 모두 Manual |
| `ST_ModuleReg` (제한·기동 조건·보정) | `bEnable`(기본 TRUE), `nGroup`(0 = 모든 그룹, 그룹 1개일 때만), `bManual`(제한 모듈만: 접속점 기준 모듈을 Manual에도) |
| `ST_RuleTargetReg` (유닛 제한·버스 제한) | `bEnable`, `nTarget`(유닛 또는 버스, 0 = 전체), `bAuto`, `bManual` (기본 모두 TRUE). 기본 `g_astBatRuleReg[1]` = SOC 정책 |
| `ST_ProtCfg` | `bAuto`, `bManual`(둘 다 FALSE면 보호 안 씀). 표준 대응 조치: `ePcsAction`(None/Standby/Stop/EStop), `eBatAction`(None/Stop/EStop, PCS Stop·EStop일 때만), `eModeAction`(None/ToManual), `fPPct`·`fQPct`(P·Q 상한 %, 기본 100) |
| `ST_RuleCfg` (`g_astRuleCfg`) | `stProt`, `eScope`(Unit/Bms/BatGroup/Group/Site), `nTarget`(0 = 범위 안 전부), `fThreshold`. 기본값 1~5칸 = 표준 보호: PCS 고장 → Stop, PCS 통신 → Standby, BMS 고장 → 유닛 Stop + 배터리 Stop, BMS 통신 → 같은 버스 유닛 Standby, 접속점 무효 → 그룹 Standby |
| `ST_OutputCfg` (`g_astOutputCfg`) | `bEnable`, `nAux`(0 = 판단만), `nPoint`, `fThreshold`, `tOnDelay`, `tOffDelay`, `bLatch` |
| `ST_GroupSet` (`g_astGroupSet`) | `eSelP`, `eSelQ`(Grade/MinAbs), `bRamp`(기본 FALSE), `fRampUp`, `fRampDn`, `fRampResume`(kW/s), `tQFilter`, `eDistBasis`(Rating/Soc/Equal, 기본 Soc. 현장에서 바꾸려면 `GVL_PERSIST.g_astGroupSet` 초기값), `bQPriority`(피상전력 초과 시 Q 우선) |
| `ST_ManualPolicy` (`g_astManualPolicy`) | `bRamp`, `bSrcLimit`, `bPccLimit`, `bComp`(기본 FALSE), `bDevLimit`, `bRating`, `bBatLimit` (나머지 기본 TRUE) |
| `ST_CmdFilterSet` (`g_stCmdFilter`) | `bBadValue`(기본 TRUE), `fBadFactor`(기본 1.5) |
| `ST_ManualCmdSet` (`g_stManualCmdSet`) | `bReverse`(기본 TRUE), `fDirBand`(기본 1 %), `bClearOnStop`(기본 TRUE) |
| `ST_SeqSet` (`g_stSeqSet`) | `tWritePeriod`(200 ms), `stPcs`(`tCmdConfirm` 30 s, `tStopRetry` 5 s, `nStopRetry` 3, `bStopRamp`, `fStopDonePct`, `tStopRampMax`), `stBat`(`eClose` OnStart, `bOpenOnStop`, `tConfirm` 60 s, `tOpenRetry` 5 s, `nOpenRetry` 3) |
| `ST_ModuleSet` (`g_stModuleSet`, `90_Project`) | 현장 모듈마다 칸 하나. 기본: `stSocLimit : ST_SocLimitSet`, 제어 모드 `stCmCommon` + PCS마다 `stCtrlModeN`·`stCmConstPN`·`stCmScheduleN`·`stCmPeakCutN`·`stCmSchedPeakN` |
| `ST_CtrlModeSet` | `bZeroOnSwitch` TRUE, `tZeroDelay` 1 s, `nFallback` 0, `ePccShare` ByRating, `fPccRatio` 1.0 |
| `ST_CmCommonSet` | `anSeasonOfMonth[1..12]`(기본 2,2,1,1,1,0,0,0,1,1,2,2), `abHoliday[1..12, 1..31]`, `stDis`(`bUse` TRUE, `nStart` 800, `nEnd` 2200, `bSunday` FALSE), `stNoRev`(`bUse` TRUE, `fMargin` 10 kW, `fScale` 0.95) |
| `ST_CmScheduleSet` | `astSlot[0..2, 0..6, 1..SCHED_SLOT_COUNT]` of `ST_SchedSlot`(`nStart`, `nEnd` HHMM, `fP` kW) |
| `ST_CmPeakCalcSet` | `fTarget` kW (0 = 피크 제어 안 함), `fBandSens` 0, `fLoadRatio` 0.8, `fMinOut` 10 kW |
| `ST_CmPeakCutSet` / `ST_CmSchedPeakSet` | `stPeak`, `stAutoChg`(피크컷만, `fSoc` 100 = 채울 목표), `stRemain`(`fSoc` 10 = 남길 하한). 시간 구간 `ST_CmWindowSet`: `bUse`, `nStart`, `nEnd`(이 시각까지 맞춤), `fSoc` |
| `ST_CmConstPowerSet` | `fPSet` kW, `bQSet`, `fQSet` kvar, `tDuration`(0 = 끝없이) |
| `ST_CmExtCmd` (스탠다드 스케줄 입력, 설정 아님) | `bAlive`, `nPSeq`, `fP` kW(방전 +), `nStateSeq`, `eState : E_PcsCmd` |
| `ST_SocLimitSet` | `fSocMax` 95, `fSocMin` 5, `fBand` 5 %, `fResumeMargin` 2 %, `bBlockOnInvalid` TRUE |
| `ST_RuleBitSet` | `bInvert`(b접점), `bOnCommLoss`(통신 끊김 시 조건) |
| `ST_RuleValueSet` | `bBelow`(이하면 조건), `fHyst`(해소 폭), `bOnCommLoss` |
| `ST_BreakerSet` | `tPulse` 1 s (코일 펄스 폭), `tConfirm` 5 s (상태 접점 확인 시간) |

### 모듈에 넘기는 요약 (읽기 전용)

| 구조체 | 칸 |
|---|---|
| `ST_GroupView` | `nGroup`, `eState`(그룹 상태), `stPcc`(접속점 계측), `fPRatedSum`·`fQRatedSum`(전체 정격 합), `nUnitAvail`, `fPRatedAvail`·`fQRatedAvail`·`fSRatedAvail`(가용 한도 합, 설비 한도 결과), `fPChgMax`·`fPDisMax`(배터리 기준 그룹 한도), `stSocMin`·`stSocMax`·`stSocAvg`, `fCapacity`(kWh, 가용 유닛 버스의 SOC 유효 BMS 용량 합), `fPOut`·`fQOut`(실측 합) |
| `ST_UnitView` (`FB_UnitView`) | `nUnit`, `bAvail`, `stView : ST_GroupView`(유닛 기준: 정격·가용 한도·출력 = 이 유닛, 버스 한도·용량 = 정격 비율 몫, SOC = 연결 버스, 접속점·상태 = 소속 그룹), `fGroupPOut`·`fGroupQOut`(그룹 출력 합), `fRatingShare`(그룹 가용 한도 중 몫 0~1) |
| `ST_CtrlModeView` | `stGroup : ST_GroupView`(그룹 단위 = 그룹 요약, 유닛 단위 = 유닛 요약에 접속점 몫을 적용한 값), `stTime : ST_LocalTime`(`bValid`, `nYear`, `nMonth`, `nDay`, `nDayOfWeek` 0 = 일, `nHour`, `nMinute`, `nSecond`) |
| `ST_UnitLimitView` | `nUnit`, `eOpMode`, `bAvail`, `eRunState`, `stP`, `stQ`, `fPRated`, `fQRated`, `nBatGroup`, `stSoc` |
| `ST_BatLimitView` | `nBatGroup`, `bUsable`, `stSoc`(용량 가중 평균), `stSocMin`, `stSocMax`, `fPChgMax`·`fPDisMax`(BMS 한도 합), `fPOut`, `nUnitAuto`, `nUnitManual` |
| `ST_ProtView` | `eScope`, `nTarget`(지금 판단하는 대상), `fThreshold`, `bInstalled`, `stUnit`·`eOpMode`(Unit), `stBms`(Bms), `stPcc`(Group), `nGroup`, `nBatGroup` |

### 모듈 결과

| 구조체 | 칸 |
|---|---|
| `ST_Limit` | `bActive`, `fMax`(상한, 방전 쪽), `fMin`(하한, 충전 쪽), `nModule`(0 = 코어), `nReason` |
| `ST_LimitOut` | `stP`, `stQ`, `bResume`(재개 중 → 재개 램프율) |
| `ST_BatLimitOut` | `stP`(`fMax` 방전 상한 ≥ 0, `fMin` 충전 하한 ≤ 0), `bResume` |
| `ST_CtrlModeOut` | `bPSetActive`, `fPSet`, `bQSetActive`, `fQSet`(그룹 단위 = 접속점 기준, 유닛 단위 = 유닛 출력), `bExitReq`(스스로 끝냄, 그 스캔 값은 냄), `nReason` |
| `ST_ProtReq` (보호 대응 요청) | 유닛마다 `aePcs`(E_PcsAction), `abToManual`, `afPPct`·`afQPct`(100 = 없음), BMS마다 `aeBat`(E_BatAction). 칸은 추가만 |
| `ST_UnitProt` `ST_BmsProt` (합친 보호 결과) | 유닛: `ePcsAction`·`nModule`, `fPPct`·`nPModule`, `fQPct`·`nQModule`, `bModeManualReq`·`nModeModule`. BMS: `eBatAction`·`nModule`. 모듈 = 조건 모듈 ID |

### 이유·진단 열거형

| 열거형 | 값 |
|---|---|
| `E_PcsCmd` | None, Run, Standby, Stop, Reset(그 장비에 고장 리셋 명령. PMS 상태는 바꾸지 않음), EStop |
| `E_BatCmd` | None, Close, Open, EStop(보호), Reset |
| `E_PcsAction` | None(알람만), Standby(출력 0, 명령 없음: 조건 동안만, 풀리면 다시 출력), Stop·EStop(정지 명령: 풀려도 Run 요청 필요) |
| `E_BatAction` | None, Stop(접촉기 개방, 버스 PCS 정지 뒤), EStop(비상정지, 버스 PCS 정지 뒤). 열린 뒤 저절로 다시 넣지 않음 (새 기동·투입 요청이 있어야) |
| `E_StartMode` | Step(단계별: BatClose → Run 하나씩, 기본), Sequence(Run 하나로 전부) |
| `E_BatClose` | OnStart(순서 기동 유닛이 원할 때, 기본), OnRequest(지령원 BatClose로만) |
| `ST_UnitAvail` 이유 | `bNotInstalled`(PCS 칸 비어 있음), `bNoPcsComm`, `bPcsFault`, `bPcsLocal`, `bNoBat`, `bProt` (Derate는 가용에 영향 없음) |
| `E_SeqRefuse` (기동·배터리 요청 거부) | NotAvail, NoBattery, NoPermit, CommLost, UnitRunning, StopInProgress, ConfigError, WriteDisabled |
| `E_UnitLimitBy` (유닛 값이 줄어든 이유) | NotAvail, NotRun(활성 아님·정지 중), Rating, Derate(보호 P·Q 상한), Scaled, BatDir, BatLimit, SLimit, ModeSwitch, WaitManualCmd, UnitRule |
| `E_DiagStage` (출력을 막은 단계) | None, NoRequest, NotRunning, Ramp, PccLimit, DeviceLimit, UnitLimit, NoUnit |
| `ST_OutputDiag` | `eStage`, `nModule`, `nReason`, `eUnitBy`, `eGroupState`, `fReq`, `fOut` |
| `E_CtrlModeStep` (관리자 단계) | Idle(모드 없음), Active, Leaving(전환 중 출력 0) |
| `E_CtrlModeRefuse` (요청 결과) | None(접수), BadSlot(칸 범위 밖·빈 칸), NotReady(진입 검사 걸림, 이유 `q_nRefuseReason`) |
| `E_PccShare` (유닛 단위 접속점 값) | ByRating(기본), Actual, Fixed |
| `GVL_CM_RSN` (요금제 모드 상태·이유) | NO_TIME 1, NO_UNIT 2, HOLIDAY 3, NO_SLOT 4, DIS_TIME 5, REV_LIMIT 6, NO_PCC 7, NO_CAP 8, PEAK 11, BELOW_PEAK 12, AUTO_CHG 13, REMAIN_DIS 14, SCHED_PEAK 15, NEW_INTERVAL 16 |
