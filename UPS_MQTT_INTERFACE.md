# UPS 공통 MQTT 인터페이스 틀

> 2026-11-02 UPS 보고 매핑 추가: 아래의 기존 미연결·발행 차단 설명보다 이 내용을 우선한다.
>
> `FB_RptSebangUps`로 UPS 계측값 41개를 연결했다. 경로는 `MAIN.A_Report → GVL_IF.g_stReport.astUps → PRG_Mqtt → g_stUpsPublish.Devices`다. `UPS_REPORT_ENABLED=TRUE`, 브로커는 `stLocal`(192.168.0.194:1883)만 사용한다. `g_fbUps1`은 UPS 시뮬레이터 보드(192.0.0.98:23, Unit 1)에 접속한다.
>
> Status는 b0 통신에러, b1 run(인버터 운전), b2 stop, b3 bypass, b4 inverter, b7 e-stop을 채운다. b5·b6과 Warning·Fault는 비트 배치 확정 전이라 0이며 고장 없음을 뜻하지 않는다. `FB_UpsModbusJob`의 타임아웃이 0으로 고정되던 오류도 고쳤다. 쓰기 차단은 유지한다. TwinCAT 빌드·보드 통신·브로커 시험은 이 갱신 시점에 하지 않았다.

> 2026-10-30 BAT 보고 매핑 추가: 아래의 기존 미연결 설명 중 BAT 계측값 부분은 이 갱신 내용을 우선한다.
>
> `FB_RptEsolutionBat`를 추가하여 BMS 3개의 Heartbeat, SOC, SOH, 전체 전압·전류, 충전·방전 허용 전류·전력, 최대·최소 셀 전압(11개 계측값)과 장치 번호·전체 랙 수를 연결했다. 경로는 `MAIN.A_Report → GVL_IF.g_stReport.astBat → PRG_Mqtt → g_stBatPublish.Devices`다.
>
> 유효한 장비 표본에만 계측값을 반영하고, 무효 시 계측 필드를 초기화한다. Status는 기존 미가용 표시 1을 유지한다. EOL·위치·모듈 온도·온라인 랙 수·경보/고장은 미연결 기본값이며 실제 측정된 0을 뜻하지 않는다. 단위 변환은 셀 mV→V, 전체 전압·전류 및 SOC ×0.1, SOH ×1을 적용했다. 전류 부호는 장비 값을 유지한다.
>
> 공통 BAT 레코드의 필드·순서·크기는 바꾸지 않았다. 제어 차단과 MQTT 접속 비활성은 유지한다. XML·프로젝트 등록·GUID·BOM/CRLF 및 실제 대입식을 이용한 숫자 변환을 확인했다. TwinCAT 빌드·브로커·실장비 시험은 수행하지 않았다.

갱신일: 2026-10-30. 사용자가 제공한 `C:/gil/UPS/인터페이스`의 장치별 최신 시트를 기준으로 한다. 이 단계는 **공통 바이너리 패킷·토픽·수신 원본 보관·수신 응답**을 연결한 틀이다. 장비 운전 설정 적용·영구 저장·실행 명령은 연결하지 않았다.

## 적용 원본

| 원본 | 시트 | 코드 |
| --- | --- | --- |
| 08.UPS.xlsx | V1 | ST_UpsPublish_V1, 필드49개 |
| 03.BAT.xlsx | V1 | ST_BatPublish_V1, Warning/Fault 1~10 확장 포함 |
| 04.Rack.xlsx | V1 | ST_RackPublish_V1, DeviceIndex·BatIndex 사용 |
| 10. Module.xlsx | V1 | ST_ModulePublish_V1, 인덱스3·온도3·셀전압20 |
| 101.Config Project.xlsx | UPS_V1, COMMON_VALUE_TYPE | ST_UpsConfig_Rx, 필드19개 |
| 103.Config BAT.xlsx | UPS_V1 | ST_BatConfig_Rx, DeviceId·SocMin REAL·SocMax REAL |
| 105.Config Protection.xlsx | UPS_V1 | 기존 ST_ProtectionConfig_Rx와 일치 |
| 104.Config Schedule.xlsx | UPS_V1 | Work/Holiday/Daily/Weekly의 타입 틀만 정의 |

파일별 버전은 서로 독립적이다. PCS 설정 파일의 UPS_V3를 UPS 충방전 Config V3로 간주하지 않는다. BAT는 개별 `103.Config BAT.xlsx`의 UPS_V1을 우선했다. `101.Config Project.xlsx`의 BAT_CONFIG는 wrapper 형식이라 상충한다. 기존 BAT Config의 value/is_available 묶음을 제거하고 별도 BAT 파일의 REAL 형식으로 맞췄으므로 EMS도 이 정의와 맞아야 한다. 이 변경으로 BAT Config 레코드는20바이트에서12바이트로 바뀐다. UPS Config는 value/is_available 묶음을 유지한다.

원본 XLSX는 수정하지 않았다. 공통 헤더와 필드의 대소문자·순서는 기존 프로토콜 및 제공된 레코드 정의를 따른다. 오타 BatWarrning 등도 wire 필드명은 원본을 보존한다.

## 토픽과 크기

사이트 번호는 기존3, solution은 ups를 유지했다. 아래는 단일 바이너리 메시지의 크기이며 JSON이 아니다. 장치 헤더40바이트, Config는 기존 ST_MqttCommandHeader 64바이트다. HeaderVersion=1, DeviceVersion=1, Config 헤더 CommandVersion=2는 기존 전송 계약을 유지한다. 본문 시트 V1과 헤더의 CommandVersion은 다른 버전이다.

| 방향 | 토픽 | 수량 | 크기 |
| --- | --- | --- | --- |
| PMS → EMS | dgspms/ups/3/ups | UPS3개 | 40 + 3×196 = 628바이트 |
| PMS → EMS | dgspms/ups/3/bat | BAT3개 | 40 + 3×200 = 640바이트 |
| PMS → EMS | dgspms/ups/3/rack | 랙30개 | 40 + 30×184 = 5560바이트 |
| PMS → EMS | dgspms/ups/3/module | 모듈300개 | 40 + 300×104 = 31240바이트 |
| EMS → PMS | dgsems/ups/3/config/ups | 최대 UPS3개 | 64 + N×148, N=1~3 |
| PMS → EMS | dgspms/ups/3/config/ups | UPS3개 수신 원본 echo | 64 + 3×148 = 508바이트 |
| 양방향 | dgsems 또는 dgspms/ups/3/config/bat | BAT3개 기준 | 64 + 3×12 = 100바이트 |
| 기존 | config/protection | 단일 레코드 | 64 + 24 = 88바이트 |

UPS Config의 토픽은 원본에 명시돼 있다. UPS·Module 보고 파일에는 토픽 행이 없어 기존 장치 토픽 규칙을 따라 각각 ups·module로 구성했다. 실제 EMS의 해당 토픽 이름은 연결 전에 확인해야 한다.

UPS·Module 보고는 GVL_MQTT.UPS_REPORT_ENABLED 및 MODULE_REPORT_ENABLED를 FALSE로 두어 발행하지 않는다. 유효한 실측 매핑 없이 300개 셀·온도0을 장비 데이터처럼 보내지 않는다. 전체 MQTT 브로커 접속 역시 기존대로 FALSE다. BAT·Rack의 수량·ID 틀은 유지하며, 미매핑 상태는 Status bit0으로 표시한다. 실제 드라이버와의 측정값·상태·경보 매핑은 다음 단계다.

## 수신 경로

```text
EMS config/ups 바이너리
 → 기존 FB_MqttSubscribe / F_MqttDistribute
 → GVL_MQTT.g_astUpsConfig[0..2]
 → GVL_IF.g_stUpsConfigRx (g_nUpsConfigSeq 홀수/짝수)
 → MAIN.stUpsConfigReceived
```

MqttTask가 수신함을 쓰고 PlcTask가 번호 전후를 확인해 일관된 원본 사본만 읽는다. MAIN은 이 값으로 설정 변경이나 장비 명령을 하지 않는다. PERSISTENT에도 저장하지 않는다. 장비 적용 단계가 없으므로 수신 echo를 적용 완료 응답으로 설명하지 않는다.

온라인 확인 예: GVL_MQTT.g_astUpsConfig[0].TrickleChargeSetParameter.value 및 is_available, MAIN.stUpsConfigReceived.astConfig[0], MAIN.stUpsConfigReceived.nRxCount. 마지막 것은 기존 MQTT의 config/ups 수신 누계이며 장비 적용 횟수가 아니다.

Config는 필드 단위 패치라고 확정하지 않았다. 현재 분배기는 한 장치 레코드를 통째로 복사하고, 다른 장치 레코드는 유지한다. is_available=0은 기능 Disable 명령이나 장비 쓰기 허가가 아니다. 해당 항목의 실제 지원·사용 가능 여부다. 간헐충전 선택값 value=0(Disable)과 구분한다.

기존 수신 누계는 헤더 TimeStamp 변경으로 새 수신을 구분한다. EMS가 같은 타임스탬프를 재사용하면 레코드 복사는 일어나도 새 수신 누계·MAIN 수신함 갱신은 발생하지 않을 수 있다. 메시지마다 TimeStamp를 갱신해야 한다. 브로커별 전역 순서·중복 제거·재전송 처리·적용 응답 규칙은 추가 설계 대상이다.

길이가 짧거나 수신 버퍼보다 크거나 헤더 DeviceCount에 필요한 레코드가 부족한 패킷은 장치 수신 배열에 복사하지 않는다. 정상 복사된 레코드가 없는 메시지는 수신 누계도 올리지 않는다. 범위 밖 DeviceId는 기존 분배기 규칙으로 버린다. 수신 구조 검사는 운전값의 의미·상하한 검증을 대신하지 않는다. 버전·site 인증/검증 정책 역시 기존 계약에 맞춰 별도 확정해야 한다.

## 번호

> 2026-11-10 변경: 모든 장치 번호는 1부터 (현장 5: UPS 1대, BAT 1대, 랙 8개, 랙당 모듈 9개). 배열 칸은 0부터 그대로(메시지 안의 몇 번째 레코드).

- 발행: UPS DeviceIndex 1, BAT DeviceIndex 1, Rack DeviceIndex 1~8 + BatIndex 1, Module BatIndex 1·RackIndex 1~8·DeviceIndex(랙 안 모듈 번호) 1~9. pms·pv_meter·ess_meter·temperature_sensor·airconditioner DeviceIndex도 1.
- 랙의 최고·최저 셀전압·모듈 온도 위치(모듈·셀 번호)도 1부터.
- 수신: command(pcs·bat·ups)와 config(pcs·bat·ups)의 DeviceIndex/DeviceId는 1부터 보내야 한다 (`F_MqttDistribute.DEVICE_ID_BASE = 1`). 0은 범위 밖으로 버린다.
- config echo의 DeviceId도 1부터.

## 확인할 문서 항목

- TrickleChargeSetStartTime은 시작 시간이라는 이름이며 추가 충전 시간인지 확정되지 않았다. 전류·시간 필드의 빠진 단위와0.1 스케일도 장비 맵과 대조 후 변환해야 한다.
- TempCompAppliedChargeVoltage는 사용자 입력·계산 목표·실제 적용값 중 역할이 미정이다. 지금은 수신·echo만 한다.
- UPS OutputActivePower의 예시는 kVA로 적혀 있지만 UPS std 맵은 유효전력 kW다. 실측 매핑 때 kW를 사용한다.
- Config Schedule의 Holiday/Weekly/Day 최상위 배열 수량·바이너리 포장과 STRING[20]의 종료문자 포함 여부가 명시되지 않았다. 타입의 Name STRING(20)은 TwinCAT 기준21바이트이며 확정된20바이트 wire 문자열이라고 취급하지 않는다. 따라서 Schedule 패킷·토픽은 등록하지 않았다. Daily Works10·Weekly Works70의 타입 틀만 제공한다.
- Module에는 유효성·통신 상태 필드가 없다. 미설치·미수신·stale 값을 어떻게 표시할지 정하기 전에는 기본 발행 차단을 유지한다.

## 검증 범위

정적 검증 통과: 원본7개 정의와 필드 순서·타입을 대조했다(UPS49·BAT50·Rack46·Module26·UPS Config19·BAT Config3·Protection6필드). 공통 헤더40/64바이트와 각 레코드 크기·패킷 크기를 계산했고, 별도 little-endian 바이너리 예제로 UPS Config의 DeviceId·criterion·전압·available 위치를 확인했다. 300개 모듈의 인덱스 조합도 중복이 없음을 확인했다.

신규 인터페이스 타입11개를 프로젝트에 등록했으며 XML·GUID·수정 파일 BOM/CRLF·ST 제어 블록 균형·코어 무변경·쓰기 및 보고 차단을 확인했다. 이번 틀 추가로 기존 파일13개가 변경됐다. TwinCAT 빌드·EMS 브로커 수신·실장비 설정 적용은 수행하지 않았다. 이 검증은 실제 PLC 실행 결과를 뜻하지 않는다. 새 패킷의 브로커/PLC 메시지 크기 한도는 실제 연결 때 확인한다.

제어 차단은 GVL_PARAM.UPS_CONTROL_ENABLED=FALSE 및 GVL_SITE.DEVICE_WRITE=FALSE로 유지하며, 이번 수신함에는 이 값을 바꾸는 기능이 없다.
