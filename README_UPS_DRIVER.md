# UPS·배터리 드라이버

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

갱신일: 2026-10-30. 현재 현장은 **UPS 3대 + 배터리 시스템 3대**, 배터리당 10랙, 랙당 10모듈이다. 별도 PCS는 설치하지 않는다. UPS 내부 정류기·인버터·배터리 컨버터(BBC)는 UPS 통신맵으로 다룬다.

## 현재 실행 구성

| 대상 | 코드 | 상태 |
| --- | --- | --- |
| UPS1~3 | GVL_SITE.g_fbUps1~3 / FB_SebangUps | I_Ups 배열에 등록. 주소 미정으로 접속 비활성 |
| 배터리1~3 | GVL_SITE.g_fbBms1~3 / FB_EsolutionBms | I_Bms 배열에 등록. 주소 미정으로 접속 비활성 |
| UPS–BMS 소속 | GVL_SITE.UPS_BMS=[1,2,3] | 같은 번호끼리 연결 |
| 랙·모듈 | BMS당 10랙, 랙당 10모듈 | 총 30랙·300모듈. 상세 읽기는 별도 선택 |
| 별도 PCS | g_aipPcs 비움, INSTALLED_PCS_COUNT=0 | 실제 호출 없음. MQTT pcs와 config/pcs echo 발행 제외 |
| ESS 지령원 | g_aipSource 비움 | 기존 ESS P·Q 제어 경로에 UPS 명령을 넣지 않음 |
| 별도 계측기 | g_aipMeter 비움, nPccPoint=0 | 가상 PCC를 실제 계측처럼 사용하지 않음 |
| 시험 데이터 | TEST_MODE=FALSE | 가상 연결·자동 시험·더미 발행 실행 안 함 |

기존 PCS·유닛 배열 크기 2는 표준 소스·시험의 호환성을 위한 미설치 칸이다. UPS 대수를 뜻하지 않는다. 기존 Plaspo·가상 FB와 테스트 소스는 보관하되 실행 등록에서 제외했다. 기존 BMS 가상 FB 인스턴스 대신 실제 맵 드라이버를 등록했다. 배터리 용량은 미정이므로 BMS_CFG.fCapacity=0이며, 표준 코어에서는 SOC 집계 시 동일 가중치로 처리한다.

원래 모듈 설정·제어 모드 및 공통 코어를 삭제하거나 UPS 전용으로 바꾸지 않았다. 10_Core는 변경하지 않았다.

## 통신 및 명령 기준

- [UPS std TCP 시험용 맵](../../프로토콜%20맵%20v3/UPS_std_modbus_tcp_map_ver_32_261022%20대건%20시험용.xlsx): Input 0~34, 38~53을 읽고, Holding 0~11과 50~66을 읽는다. 예약 영역35~37,12~49는 일괄 읽기에 포함하지 않는다. 명령은 주소100~125 중 정의된 항목에 0x55를 쓴다. 주기적 시작115·펄스 시작116·공통 방전 취소117을 사용한다.
- [System v1.3 맵](../../프로토콜%20맵%20v3/UPS_System_MODBUS_TCP_Protocol%20v1.3.xlsx): Input 0~24,50~89,90~179를 분리해 읽는다. 명령40011은 0 기준 주소10이며 한 비트씩 전송한다.
- System v1.3 전용 제어: 비상차단40012/주소11, 시스템 접촉기40013/주소12, 충전 접촉기40014/주소13, 방전 접촉기40015/주소14. 각각 uint16 값으로 UPS의 0x55와 구분한다. 비상차단의 0은 해제 명령으로 구현하지 않는다.
- I_Bms의 Close/Open은 System Contactor Control, EStop은 Emergency Shutdown, Reset은 Reset System(40031/주소30)에 연결한다. 개별 랙 비트맵 제어와는 별개다. 반환 TRUE는 대기열 접수이며 실제 적용·접촉기 투입 완료를 뜻하지 않는다.
- 상세 읽기를 켜면 랙별406개를 125+125+125+31개로 나누며 랙 간 주소 간격500을 유지한다. 10모듈 설치라고 맵의 고정17모듈 배치를 10모듈 간격으로 줄이지 않는다.

공통 Modbus 작업 FB는 트랜잭션 시작 시 IP·포트·Unit ID·주소·길이·값을 고정하고 응답 길이도 확인한다. 읽기는 FC03/04, 쓰기는 FC06만 사용한다. 호출 시그니처는 [Beckhoff 공식 Modbus TCP 문서](https://download.beckhoff.com/download/document/automation/twincat3/TF6250_TC3_Modbus_TCP_EN.pdf)와 대조했다.

## 제어 차단과 상태 확인

- GVL_PARAM.UPS_CONTROL_ENABLED=FALSE이며 GVL_SITE.DEVICE_WRITE=FALSE다. 모든 신규 UPS/BMS 장비 쓰기는 드라이버 접수와 실제 FC06 시작에서 차단한다. 웹/MQTT로 이 상수를 변경하는 경로는 추가하지 않았다.
- MAIN의 bWriteEnable은 표준 설정 정상·이중화 쓰기 허가·DEVICE_WRITE·UPS 구성 점검을 합친다. 시작·설정·취소·부동충전 복귀 명령도 같은 허가를 따른다. 이미 전송한 쓰기를 취소할 수 있다고 표시하지 않는다.
- UPS 상태: GVL_SITE.g_fbUps1.q_stState(2·3도 동일), 원본 레지스터: q_stRaw, 통신 오류: q_nTransportError.
- BMS 상태: GVL_SITE.g_fbBms1.q_stState, 원본 레지스터: q_stRaw. q_stState.bValid는 기본 읽기 묶음 완성과 heartbeat 신선도를 포함한다. 상세 데이터는 abDetailValid와 anDetailSeq를 따로 확인한다.
- 새 장비는 최초 정상 응답 전까지 정상으로 표시하지 않는다. Disable 후 재활성화하면 기본 읽기 묶음을 다시 완성해야 유효해진다. 현재 비활성 초기 상태의 0은 실측 0이 아니다.
- 물리 접촉기 피드백은 미확정이므로 bContactorFeedbackKnown=FALSE, 표준 bContactorClosed=FALSE로 둔다. 명령·일반 운전 모드로 투입 완료를 추정하지 않는다.
- UPS 전용 인터페이스는 공통 코어의 ModuleList에 자동 등록되는 종류가 아니다. MAIN.awsUpsConfigError와 bUpsConfigOk로 설정을 확인한다. UPS 통신 경보의 표준 이벤트 로그·장비별 MQTT 실측 매핑은 아직 연결하지 않았다.

## 읽기 시험 준비

1. UPS1~3과 BMS1~3의 실제 IP·Port·Unit ID를 확정한다. 지금은 i_sIpAddr가 빈 문자열이고 i_bEnable=FALSE다. Unit ID 기본1은 미확정 초기값이지 현장 확정값이 아니다.
2. 각 GVL_SITE 인스턴스의 통신 입력을 설정하고, 비활성 상태에서만 통신 대상 주소를 변경한다. 접속할 장비만 i_bEnable=TRUE로 한다. 읽기 확인 중에는 두 쓰기 상수를 FALSE로 유지한다.
3. BMS 랙 수10·모듈 수10을 확인한다. 맵의 모듈 슬롯은 0~16이며 실제 설치 슬롯과 모듈0 사용 여부는 확인 대상이다. i_bReadDetails는 기본 FALSE이며, 미설치 모듈 영역을 읽었을 때 응답하는지 확인한 후 상세 읽기를 켠다. 기본 상태·랙 상태 읽기는 별도로 동작한다.
4. 사용자가 TwinCAT 빌드 후 활성화·온라인 상태를 확인한다. 초기 통신·스케일·부호·0 기준 주소 정렬과 최신 장비 펌웨어의 명령 지원은 실기 확인 대상이다.

브로커 접속은 기존대로 모두 비활성이다. 배터리·랙 MQTT 패킷은 구성 번호 외 실측 보고를 아직 채우지 않으며, 0을 실제 장비값으로 설명하지 않는다. UPS를 기존 pcs 패킷에 임의로 넣지 않았다.

## 남은 작업

이 단계는 장비별 드라이버·주소 매핑과 현장 연결이다. 간헐충전·온도보상 반복 제어 수행 주체, BMS 방전 설정 전달 경로·수신 확인, 실제 접촉기 피드백, LiB 운전값을 확정하기 전까지 충방전 자동 시퀀스에 직접 연결하지 않는다. 최신 자료의 주 운전 충돌 시 시작 거부·온도보상 병행 및 주기적/펄스 방전 중 보정 적용 보류를 기준으로 한다.

자동 충방전 로직, 웹 설정의 PLC 적용, UPS·BMS 실측 MQTT 매핑·이벤트 로그는 완료되지 않았다. 기존 ESS의 보호·출력 분배를 UPS 고장 대응 시퀀스로 간주하지 않는다.

백업: C:/gil/UPS/tmp/pms_before_ups_drivers_20261030/source.zip.

정적 검증: XML·GUID·수정 파일 BOM/CRLF·프로젝트 등록·ST 제어 블록 균형을 확인했다. UPS 명령26개와 System v1.3 신규 제어4개의 주소를 최신 XLSX와 대조했다. 기본 읽기의 예약 영역 분리, 랙10개 상세4060개 주소의 분할·중복 없음, 물리 접촉기 상태를 추정하지 않는 경로, PCS 미설치·양쪽 쓰기 차단 및 10_Core 무변경을 확인했다. Disable 전 시작된 읽기 응답이 재활성화 후 새 스냅샷에 들어가지 않도록 폐기 경로를 추가했다.

TwinCAT 빌드·장비 통신·명령 실행 시험은 하지 않았다. 이 정적 점검이 런타임 검증을 대신하지 않는다.

현재 공통 MQTT 인터페이스의 패킷·Config 수신함 틀은 [UPS MQTT 인터페이스](UPS_MQTT_INTERFACE.md)에 추가했다. 실측 매핑·운전 설정 적용은 아직 미연결이다.
