# UPS 시험 구성

1차 수량 변경 기록: 2026-10-30. 당시 범위는 **시험 수량·연결 구성만 반영**이었다. 이후 사용자의 진행 요청으로 UPS·BMS 드라이버 및 UPS 전용 현장 프로필을 추가했다. **현재 실행 구성과 남은 범위는 [UPS 드라이버 안내](README_UPS_DRIVER.md)를 우선한다.** 아래 반영·미반영 목록은 1차 변경 당시의 기록이다.

| 항목 | 설정 | 위치 |
| --- | --- | --- |
| UPS | 3대 | GVL_PARAM.UPS_COUNT |
| 배터리 시스템 / BMS | 3대 | GVL_PARAM.BMS_COUNT |
| 배터리 그룹 | 3개 | GVL_PARAM.BAT_COUNT |
| 배터리당 랙 | 10개 | GVL_PARAM.RACKS_PER_BMS |
| 총 랙 | 30개, BMS_COUNT × RACKS_PER_BMS | GVL_PARAM.RACK_COUNT |
| 랙당 배터리 모듈 | 10개 | GVL_PARAM.MODULES_PER_RACK |
| 총 배터리 모듈 | 300개 | GVL_PARAM.BAT_MODULE_COUNT |

배터리 모듈 수는 PLC 기능 모듈 수인 MODULE_MAX와 다른 개념이다.

## 연결 구성

GVL_SITE.UPS_BMS는 UPS 번호(1부터)에 대응하는 BMS 등록 칸(1부터)을 정의한다. 현재는 [1, 2, 3]으로 UPS1 ↔ BMS1, UPS2 ↔ BMS2, UPS3 ↔ BMS3이다. 실제 UPS 드라이버가 사용하는 통신 연결은 아직 구현하지 않았다.

| UPS 번호 | BMS 등록 칸 | 배터리 그룹 | MQTT BAT DeviceIndex | MQTT RACK DeviceID | 랙 ModuleCount |
| --- | --- | --- | --- | --- | --- |
| 1 | 1 | 1 | 0 | 0~9 | 10 |
| 2 | 2 | 2 | 1 | 10~19 | 10 |
| 3 | 3 | 3 | 2 | 20~29 | 10 |

랙의 BatID는 0~2다. PRG_PublishDeviceNo가 DeviceID·BatID·ModuleCount를 함께 채운다. 더미 측정값을 꺼도 이 구성 정보는 유지되며, 값이 있다는 이유로 통신 정상이나 실측 데이터라고 판단하지 않는다.

## 반영 범위

- BMS3 가상 FB와 등록·배터리 그룹·시뮬레이션 연결을 추가했다. 가상 용량 4000kWh는 기존 시험값을 따른 것이며 실제 장비 정격이 아니다.
- MQTT의 BAT·RACK 배열과 패킷 크기는 기존 GVL_PARAM 참조를 통해 3대·30랙으로 확장된다. 토픽과 패킷 레코드 형식은 유지한다.
- 더미 랙 데이터는 모듈 수 10개와 위치 1~10에 맞췄다. 배터리 요약의 기존 모듈 위치11도 설정 범위 안으로 제한했다. 셀 수는 이번에 확정하지 않았고 기존 더미 생성 규칙을 유지한다.
- 기존 PCS 두 대와 제어 모드·유닛 구성은 유지했다. UPS는 PCS와 별개이므로 PCS_COUNT를 3으로 바꾸지 않았다. 배터리 그룹3에 해당하는 PCS 유닛은 없어 그 그룹의 PCS 연계 가상 전력은 0이다.
- 10_Core와 장비 쓰기 허가 로직은 변경하지 않았다. GVL_SITE.DEVICE_WRITE=FALSE, 기존 MQTT 브로커 bUse=FALSE 상태를 유지한다.

## 남은 범위

실제 UPS/BMS 드라이버, 모듈·셀 상세 읽기, UPS MQTT 실측 보고, 충방전 시퀀스, MQTT 설정 적용은 이번 작업 범위가 아니다. UPS_COUNT와 UPS_BMS는 다음 드라이버 연결을 위한 구성 정의이며 UPS 세 대의 통신이 구현됐다는 뜻이 아니다. 300개 모듈의 상세 계측 배열·발행을 추가한 것도 아니다.

실제 통신은 UPS std v3.2 TCP와 System v1.3 맵을 함께 적용한다. 실제 IP·Port·Unit ID, 모듈 인덱스 및 미설치 영역 처리, 장비 정격은 드라이버 작업 때 확인한다.

## 검증 및 다운로드

정적 점검 통과: XML 소스 346개 파싱 및 GUID 중복 없음, 수정 PLC 파일의 BOM·CRLF, 프로젝트 등록 경로, 3배터리×10랙의 ID 및 모듈 위치 범위를 확인했다. 백업과 비교해 기존 파일 6개만 변경됐고 10_Core는 변경되지 않았음을 확인했다. 실제 쓰기와 브로커 접속도 비활성화 상태다.

TwinCAT 빌드·다운로드·온라인 확인·브로커 수신 시험은 수행하지 않았다. 해당 검증은 사용자가 수행한다.

BMS_COUNT와 BAT_COUNT 변경으로 PERSISTENT 배열의 크기도 달라진다. 다운로드 전 기존 운영 설정을 보관하고, 다운로드 후 실제 유지·초기화된 설정값을 확인한다. 기존 값을 확인 없이 Reset Origin으로 지우지 않는다.

백업: C:/gil/UPS/tmp/pms_before_ups_3x10x10_20261030_135553/source.zip 및 같은 폴더의 manifest.json.
