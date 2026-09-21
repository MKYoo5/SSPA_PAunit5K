# PAunit_Firm 작업 계획서

**작성일**: 2026-09-16 시작 → 2026-09-17 정리본 → 2026-09-19 갱신 → **2026-09-21 갱신**(이전 버전: `PAunit_Firm작업_계획서_0919.md`, 상세 로그: `이전/PAunit_Firm작업_계획서_0917_상세로그.md`)
**작업 위치**: `E:\01_Claude_Work\SSPA160K_PRJ\10_PAunit5k설계\50_HPA_Coding`
**코드 위치**: `Bootload_Firm/`(Bootloader), `PAunit_Firm/`(Application), `50_HPA_Coding.ioc`(MX 프로젝트)
**참조 파일**: `26_PAunit_Firm작업_0629/0916/0916_2~4/0917_1/0918.xlsx`, `26_PAunit_Coding작업_0919.xlsx`, **`26_PAunit_Coding작업_0921.xlsx`(현행 원본)**, `26_PAunit_Coding작업_0921_Anl수정안.xlsx`(2.3_Anl_Process 정리안), `26_HPA설계_160kSSPA_진행0915.xlsx`, 사용자 제공 회로도·MX 스크린샷

---

## 0. 갱신 요약

### 2026-09-21 (이번 갱신)

| 구분 | 내용 |
|---|---|
| **Analog Data Process** | 신규 `2.3_Anl_Process` 시트(구 `2.3_ADC읽기` 대체)를 검토해 1차 확정 (→ 6-12절, 7절 13번) |
| **처리 흐름** | `Source → ADCinAry → Avlt(V) → 5차 다항식 → Offset → PAnal → PA_Sts(12+N)` |
| **ADC 채널 배치** | Pin Map 기준 8채널: ADC1 = FWD·RFL, ADC2 = 6채널. DMA scan·연속 변환 (MX 사양 → 6-12절) |
| **DC Pack 6항목** | 이미 환산된 값 → **BYPASS**(Avlt·다항식 생략, Offset만 적용) |
| **처리 주기** | FWD/RFL은 3mS마다 매번, 나머지 29항목은 순환(87mS/loop) |
| **다항식 표** | 입력 X = Avlt(V) 통일, 배율을 계수에 반영, 계수 **Master = EEPROM 시트** |
| **Array Index** | 0부터 (`n = Proc NO − 1`), `Aoffset(n) = Pary(0,n)` |
| **MUX 송신 주기** | **150mS 확정** (`3.인터벌동작` 시트의 250mS는 원본 정정 필요) |
| **EEPROM 메모리 맵** | **구획1(공통 정보, 1KB) 신설** — 기존 구획1·2·3을 구획2·3·4로 shift. PA unit Address(예: 132)를 IF board에서 수신해 저장 (→ 4절) |

### 2026-09-19 (유지)
- **Command Code**: `2.2.1SysCMDcode` 기준(PA unit = 24xx~54xx), PA 알람 항목 34 = 실측 31 + Spare 3 (→ 4절, 6-13절)
- **Cmd_Control 5.2 Code Command in**: Target 888·자기 Address, RF On 2초 지연, Fault 우선 (→ 6-11절)
- **DC Pack 통신 Alarm**: `PA_Sts(4).0~2`/`(7).0~2`/`(10).0~2`, `444` 미사용

---

## 1. Bootloader / Firm Download — 코드 완료

**역할**: Bootloader는 "Application을 올리는 것"만 담당(Download 플래그 확인 + Application Jump). RF 제어·통신·Alarm 등 PA unit 동작 로직은 전부 Application 몫.

| 항목 | 내용 |
|---|---|
| 설치 | SWD(SWDIO=PA13, SWCLK=PA14)로 최초 1회 설치. JTDI(PA15)/JTDO(PB3)/TRST(PB4)는 미사용 NC → MX Debug 설정 = **Serial Wire** |
| BOOT0 | Pin94, **GND 고정**(Pull-down) — User Flash Boot 보장 |
| Flash 배치 | Bootloader: `0x08000000~0x08003FFF`(16KB, Sector0) / Application: `0x08004000~`(Sector1~11) |
| 동작 | Cold Start 시 대기 없이 즉시 Application Jump. Download는 RTC Backup Register(`RTC->BKP0R`) 플래그로 진입 판단 |
| Download 경로 | UART1(전면, RS232 115200bps) — 동일 포트로 PC Monitor 상태 모니터링도 겸함 |
| 패킷 포맷 | `STX(0xF2)+CMD+LEN(2)+DATA+XOR` — CMD: START(Size+CRC32)/DATA(최대256B)/END(CRC32검증) |
| Fail-safe | CRC32 불일치·Flash 오류 시 플래그 유지, 재시도 대기(미완성 Application으로 Jump 방지) |

**남은 작업**: ① Application 측 "Download 진입" 트리거 코드(PC 커맨드 수신 시 `BKP0R` Set 후 Reset) ② PC측 Download Tool ③ Bootloader 전용 빌드 프로젝트 구성(CMSIS startup 별도) ④ UART Timeout 재보정

---

## 2. 개발환경 / System Clock — 확정

| 항목 | 값 |
|---|---|
| Tool | Claude Code + VS Code, MCU: STM32F407VGT6_LQFP100 |
| HSE | 25MHz(외부 크리스탈, OSC_IN/OSC_OUT) |
| PLL | M=25, N=336, P=2 |
| SYSCLK/HCLK | **168MHz** (PLLCLK 소스) |
| APB1 | 42MHz (Prescaler /4, Max) |
| APB2 | 84MHz (Prescaler /2, Max) |

`.ioc`에서 최종 확정 완료 — 모든 시간 기준 계산은 168MHz 전제.

---

## 3. Coldstart Init — 부분 확정 (나머지 보류)

원본(`1.1ColdStart`) 순서 중 아래만 확정, 나머지(전역변수/Array 정의, setting 값 호출)는 **세부 항목 미확정으로 보류**.

1. **EEPROM Parameter 로드**: 구획1 공통 정보(PA unit Address 등) + 구획2 Alarm 배열 272byte 1회 Read (4번 항목 참조). 읽은 PA Address는 `PA_Sts(0)`(전송 Data 선두)에 기록
2. **ADC 환산식 Array 로드**: 채널별 다항식 계수를 RAM 변수영역에 적재 (13번 항목 참조)

---

## 4. EEPROM Read/Write — 정책 확정, 시트 갱신 대기

**HW 사양**

| 항목 | 내용 |
|---|---|
| EEPROM | 24FC128-I/SN (128Kbit, SOIC-8, U2) |
| 통신 | Software Bit-bang I2C, **ST HAL 함수** 기반, 100kbps + **SCL Read-back**(Clock Stretching 지원, 10kΩ Pull-up 대응) |
| 핀 | SCL=PE14(Pin45), SDA=PE15(Pin46) — 일반 GPIO, Open-Drain |
| I2C 주소 | 0xA0(Write)/0xA1(Read) — A0/A1/A2 전부 GND 접지(회로도 확인) |
| Pull-up | 외부 10kΩ 기실장 |

**데이터 구조**: 34개 항목(**실측 31 + Spare 3**) × 4그룹(Offset/ENB·Dis/Level/Delay) = 항목당 8byte, 전체 **272byte** (항목 목록·순서는 `2.2.1SysCMDcode` 4.2.4, EEPROM 시트, `2.3_Anl_Process` Proc NO가 동일)

| 그룹 | 의미 | Command Code (SysCMDcode 기준, PA unit = 24xx~54xx) |
|---|---|---|
| Offset | Analog 값 배율(1~200%, Default 100=1.0) | 2401~2434 |
| ENB/Dis | Alarm 감시 On/Off | 3401~3434 |
| Level | Alarm 임계값 | 4401~4434 |
| Delay | 임계값 초과 지속시간(10mS 단위) | 5401~5434 |

- **Index 규칙(0921)**: Array Index는 **0부터**. `n = 항목번호 − 1`(0~33), `Pary(g,n)`, 코드 끝 두 자리 = n+1 (예: 2401 ↔ n=0). `PA_Sts(0)`=Unit Address, `MxAry(0)`=Spare는 종전대로. EEPROM 시트·SysCMDcode 4.2.4의 `Pary(g,1..34)` 표기는 사용자 갱신 시 0부터로 정리.
- **EEPROM 시트 1차원화(사용자 작업 예정)**: Alarm 배열을 1차원 배열로 수정 예정. **1차원 순서(항목별 4그룹/그룹별 항목)는 시트 갱신 후 확정**하여 바이트 주소 계산에 반영 — **보류**.
- **다항식 계수 Master = EEPROM 시트**(0921). `2.3_Anl_Process`·SysCMDcode 5.4의 표는 참조·검증용.
- **ENB/Dis 방식 변경(반영 대기)**: 기존 = Enable/Disable 코드 2개 + Value 항상 0. SysCMDcode 초안 = 코드 1개(34xx) + **Value 1/0 저장**. EEPROM 시트 갱신 후 확정, 초기값(1=Enable 여부)은 **보류**.
- **Spare(항목 32~34)**: 기존 확정은 Level/Delay=0. SysCMDcode 표의 Spare 기본값(200/10)과 불일치 → **보류**. 항목명도 Spare1~3으로 정리 필요.
- **메모리 맵**(24FC128 16KB 전체, 타 Unit 공용 설계, **0921 변경: 4구획**):

| 구획 | 용도 | 주소 | 크기 |
|---|---|---|---|
| **1 (신설)** | **공통 정보** — PA unit이 공통으로 저장해야 할 사항(PA unit Address 등) | `0x0000~0x03FF` | 1KB |
| 2 (구 1) | Alarm + Spare (272byte 사용) | `0x0400~0x07FF` | 1KB |
| 3 (구 2) | ADC 환산식 + Spare (float32 31채널×6계수 = 744byte 필요 / 8.3배 여유) | `0x0800~0x1FFF` | 6KB |
| 4 (구 3) | 나머지(예비) | `0x2000~0x3FFF` | 8KB |

  (기존 구획3 9KB에서 1KB가 구획1로 넘어가 구획4는 8KB. 합계 1+1+6+8 = 16KB)
- **구획1 — PA unit Address 저장 (0921 추가)**
  - 예) PA unit Address = `132` (RCU1 · IF board 3 · PA 2). 전면 Switch는 0~9까지만 설정 가능해 전체 Address를 물리적으로 설정할 수 없음.
  - **IF board가 Address를 송신 → PA unit이 EEPROM 구획1에 저장 → Cold Start 시 읽어와 `PA_Sts(0)`(전송 Data 선두)에 기록**하고, Command 수신 시 Target 판정에도 사용(6-11절).
  - **보류**: ① IF board가 Address를 전달하는 방법(Command Code/필드 정의, Address 미설정 상태에서 수락할 Target) ② EEPROM 공백(미설정) 시 초기값·유효성 판정 ③ 구획1의 그 외 공통 항목 목록 ④ Default(777)·Factory Set 시 구획1(Address)을 유지할지(유지 권장)

**Read/Write 동작**

- Cold Start: 구획1(공통 정보/PA Address) 읽기 후 구획2(Alarm 배열 272byte) 1회 Read
- 운용 중: 5~10초 주기 Round-robin(1항목씩) 검증, 불일치 시 **EEPROM이 우선**(ROM→RAM 덮어씀)
- Write 트리거: ①IF board 명령(개별 8byte) ②PC Monitor 값변경(개별) ③PC Monitor **"Default"** 명령(구획2 전체 272byte, Factory Default 복원) 및 Code **777**(Factory Set) ④**IF board의 PA Address 수신 → 구획1 저장(0921 추가)** — 최초 부팅 판별은 포기, Default 명령으로 단일화

**남은 작업**: 6번(PC Monitor Default 기능)·8번(IF board Command→EEPROM Write 매핑) 상세화 시 연동 반영. **777 Factory Set 범위**(Alarm 배열 구획2만 Default, ADC 환산식 구획3·공통 정보 구획1(Address)은 유지 여부)는 **보류**.

---

## 5. 통신모듈

### 5-6. UART1 PC Monitor 통신
Dummy Data 수신, Status Data Table 200mS 전송, Parameter Change 처리. **"Default" 명령 추가 필요**(수신 시 EEPROM 34항목 전체를 Factory Default로 기록) — 아직 미구현.
**보류**: PC Mon 경로의 Target 값(`2.5PC_Mon통신`에 500 "임의값"). **PA 자기 Address는 EEPROM 구획1에 저장**(0921 결정, 4절)하며 IF board가 전달하는 방법은 보류.

### 5-7. MUX CPU 통신 — 완료

| 항목 | 내용 |
|---|---|
| 방식 | RS232, 115200,8,N,1, USART2(TX=PD05, RX=PD06), MAX3232CD(U3) 트랜시버 |
| 커넥터 | Z5(5267-8P): Pin2=RX, Pin3=TX, Pin6=Status_in(PE00 추정), Pin8=PA_ON_out(PE02 추정) |
| 주기 | **150mS 송수신 확정**(0921). `3.인터벌동작` 시트의 250mS는 **원본 정정 필요** |
| 수신 | DMA |
| Data 형식 | 16bit Word, Low byte 먼저 |
| Error Check | XOR + Modbus CRC16 (의도된 이중 방어) |

**TX 패킷**(Main→MUX, 35byte): `STX(0xF2)|Counter(0x1C)|Target|Group|CMDCode|Data1|Data2|spare×4|AGC Gain/Phase Target|spare×4|DummyData|XOR|CRC16|ETX(0xF3)|CR|LF`

**RX 패킷**(MUX→Main, 58byte): `STX|Counter(50)|MUX_Ary(1~25, 50byte: Pallet Status1/2·RF Signal·Driver AMP·Heatsink Temp·Gain/Phase Adjust·Pallet1~4×4종·Load Power×4·Spare×2)|XOR|CRC16|ETX|CR|LF`

### 5-8. IF board 통신 — 완료

| 항목 | 내용 |
|---|---|
| 방식 | RS422, UART6(TX=PC06, RX=PC07), 230400,8,N,1 |
| Data 요청 | Master(IF board)가 PB10(Pin47)을 Low로 내리면 PA unit이 Status 전송 |
| Command 수신 | IF board가 200mS 간격 일방 송신(26byte) |
| Data 형식 | 16bit Word, Low byte 먼저 |
| Error Check | XOR + Modbus CRC16 + **1st/2nd 이중 헤더**(Data Counter/Address 앞뒤 중복 기재로 프레임 정렬 검증 — 통신 Error 시 오기록 방지) |

**공통 구조**: `STX(0xF2)|Target Addr|Counter(N)|Source Addr|Data[N]|XOR|CRC16(L/H)|ETX(0xF3)|CR|LF` (총 N+10byte)

- **Command 패킷**(N=26, 36byte, 13 Word): D01 Source / **D02 Command Code** / D03·D04 Command Data L/H / D05~D06 Spare / D07~D09 PA unit Target FWD Power·Gain Volt·Phase Volt(SysCMDcode 표기) / D10~D11 Spare / D12 2nd Counter / D13 2nd Target. *(D07 명칭이 IF board 시트("System 동작상태")와 다름 — 확인 필요)*
- **Status 패킷**(N=116, 126byte): PA Address + D01~D53(53항목: Status Bit7 + RF Signal/FWD·RFL Power/Heatsink Temp/DC Total·Pack1~3 Volt·Current/Dri AMP·Driver AMP/Pallet1~4×4종/Offset류/Interlock류/Spare/Dummy) + 2nd Counter/2nd Address. 수신측 배열명(`P111` 등)은 IF board 내부 명칭이라 PA unit 코딩 범위 밖.

### 5-9. DC Pack I2C 통신 — 완료

| 항목 | 내용 |
|---|---|
| 대상 | CP3500AC65TEZ(OmniOn Power) × 3, PA unit당 내장 |
| 통신 | 하드웨어 I2C3, SCL=PA08(Pin67), SDA=PC09(Pin66), **PD0=I2C3_ENB**(PCA9515A 버퍼 Enable, Active Level 확인 필요) |
| 진행 | 100mS 간격 Pack1→2→3 순차 요청(Pack별 갱신주기 300mS), 요청 Code=Status_summary(0xD0) |
| ⚠ | RF 잡음으로 통신 Error 잦음(원본 명시) — 20회 연속 무응답 시 Error Bit |

**Data 변환**: 수신 = IEEE-754 Half float(16bit) → Voltage×10/Current×1/Temperature×10 정수 변환

- 복원식: `실수값 = (-1)^S × 2^(E-15) × (1+M/1024)` (S=1bit부호, E=5bit지수, M=10bit가수). E=31(NaN/Inf)은 Error 처리
- 검증: 65.2V → Raw16=0x5413 → 역복원 65.1875 → ×10 반올림 = 652 (시트 예시 일치)
- 유의: Half-float 유효숫자 3~4자리 → ×10 결과 최하위 자리 ±1 오차는 정상

**저장 변수**: Volt→`DcPk1V/2V/3V`, Current→`DcPk1A/2A/3A`(→ Proc NO 5~10, `PA_Sts(17~22)`), Status/Alarm/Temp→`DCpack{1,2,3}{Sts1,Sts2,Alm1,Alm2,Alm3,Temp}`(전용 변수).
**Comm Error Alarm**: `PA_Sts(4).0/.1/.2` = Pack1/2/3 **Latch**, `PA_Sts(7).0~2` = Enable, `PA_Sts(10).0~2` = Unlatch (기존 `PA_Sts(3).8~10` 방식 폐기, 다른 Latch Alarm과 동일 체계)

---

## 6. 입출력 처리

### 6-10. GPIO 상태입력 처리 — 설계초안 (미착수)

### 6-11. Command in / Control out — 코드 작성 완료(0917분), **0919 확정 사항 반영 대기**

**공통 규칙**: Status/Command in 핀 = Pull-up + Active Low. Control Out 핀 = Open-Drain + Active Low. 내부 변수는 전부 **정논리(Active High)**로 정규화.

| 기능 | 입력 | 조건 | 출력 |
|---|---|---|---|
| Standby | PB01(STBcmdi) | (`IO_STBcmdIn` or `Cod_STBcmdIn`) & `PAfaultSts`=L | **PC10**(DC Pack On) Low |
| RF On | PE07(RFONcmdi) | (`IO_RFonCMDin` or `Cod_RFONcmdIn`) & `PAfaultSts`=L & **STB_CNTR_out=H 2초 이상 유지** | **PE2**(MUX_RFonO) Low |
| Reset | PB02(RSTcmdi) | `IO_RSTcmdIn` or `Cod_RSTcmdIn` | 물리 출력 없음 — **내부 로직 전용**(Fault Latch Clear 등) |
| Off | (파생) | 원시 핀 조건 — **확인 필요**(아래) | `IO_OffcmdIn` |

**5.2 Code Command in — 확정 사항 (2026-09-19)**

- 수신 경로 2가지(① RCU→IF board→PA ② PC Mon→UART1 직접)는 UART Port만 다르고 **Frame 검증·Command 처리 루틴 공용**. Frame 검증 통과 패킷만 처리.
- **Target = PA 자기 Address(EEPROM 구획1 저장값) 또는 888**만 수락(444 미사용). **Source는 발신처 식별용**(수락 제한 없음; RCU 100/200, IF board WEB 1100~1400)이며 마지막 Command의 Source를 `Cod_SrcAddr`에 저장.
- IF board/PC Mon 명령은 같은 `Cod_` 변수를 세움(동시 수신 시 마지막 수신 명령 우선).
- 상태 명령(750/720/710)은 **마지막 명령 유지(Latch)**, Cold Start 초기값 = Off(`Cod_STB`/`Cod_RFON`=L). 790(Reset)/777(Factory Set)은 **1회성**(처리 후 자동 L, Reset은 Fault Latch Clear 1주기 후). System 명령의 Data(L/H)는 미사용.
- **Fault 우선**: `PAfaultSts`=H(Shut Down)이면 RCU가 명령을 계속 유지해도 Control Out 차단. `Cod_` 상태는 내리지 않고 유지, Reset으로 Fault 해제 시 명령이 유지 중이면 Control Out 재개(RF On은 2초 지연부터 다시).
- **RF On 2초 지연**: 720(RF On) 수신 시 Standby 출력 후 2초 뒤 RF On 출력(SSPA 안정 동작). STB_CNTR_out=L이 되면 RF On 즉시 L, `RFON_DlyTmr` Reset. IO 명령 경로에도 동일 적용. 신규 변수: `Cod_FactSetReq`, `RFON_DlyTmr`, `Cod_SrcAddr`.
- 통신 두절 시 `Cod_` 변수는 유지, 두절 자체는 통신 Alarm 항목에서 처리. *(주의: Control Out이 IO or Code 합성이라 IO로 Code 유지 출력을 끌 수는 없음)*

**코드 반영 필요**(`PAunit_Firm/cmd_control.c/h`는 0917 기준): `PAfaultSts` 조건, RF On 2초 지연, Code 명령(750/720/710/790/777) set/clear 처리 추가. 코딩은 지시 시 진행.

**남은 확인**: ① `2.2Cmd_Control` 37행 OFF 조건이 `STBcmdi=H & RFONcmdi=H`로 적혀 있으나 기존 확정은 "둘 다 Low(Active)일 때 Off" — 원본 정정 필요 ② 시트의 "5.2 Control Out 처리"(88행) 번호가 "5.2 Code Command in"과 중복 → 5.3으로 정정 ③ **5.3 Status in 처리**가 원본에 "추가 예정"으로 완전히 비어 있음

### 6-12. Analog Data Processing (ADC in) — 1차 확정 (2026-09-21), 코딩 대기
원본: `2.3_Anl_Process` (구 `2.3_ADC읽기`, Maj/Min 구분은 "FWD/RFL 우선 처리"로 대체). 목적: ADC/MUX/DC Pack 값을 사람이 보는 단위 값으로 변환.

**처리 흐름**: `Source → ADCinAry(n) → Avlt(n)[V] → 5차 다항식 → Aoffset 적용 → PAnal(n) → PA_Sts(12+N)` (n = Proc NO − 1)

| Source | 항목 수 | 처리 |
|---|---|---|
| **ADC**(MCU) | 8 — FWD, RFL, DC Total Volt, Driver AMP DC Current, Pallet1~4 DC Current | ADCinAry → Avlt → 다항식 → Offset |
| **MUX**(`MxAry`, 원시 ADC값) | 17 — Heatsink, RF Signal in, Driver AMP Power, Pallet1~4 FWD·RFL·Temp(12), Gain/Phase Adjust Volt | 동일(MxAry 번호는 MUX 시트와 일치 확인: (3)RF Signal, (4)Driver, (5)Heatsink, (6)(7)Gain/Phase, (8)~(19)Pallet) |
| **DC Pack**(I2C, 환산 완료값) | 6 — Pack1~3 Volt·Current | **BYPASS**: Avlt·다항식 생략, Offset만 적용 |

**처리 규칙 (확정)**
- `Avlt = ADCinAry × 3.3 / 4095` (V, float32, 예: 1.375V). **다항식 입력 X = Avlt(V)**.
- `PAnal = round( 다항식(Avlt) × Aoffset / 100 )`, **uint16, 0~65535 clip(음수는 0)**. Aoffset = EEPROM Offset Word(100 = 1.0, 범위 1~200, **0은 무효값**). Offset은 다항식 결과에 곱함(Alarm은 PAnal로 판정).
- 계산은 float32 + Horner 방식(FPU 사용). 호출 = `AnalProc_Call`(3mS Interval).
- **처리 주기**: 3mS마다 ① PA out FWD ② PA out RFL 매번 처리 + ③ 나머지 29항목 중 1개 순환 처리 → 29×3mS = **87mS/loop**. (MUX 150mS, DC Pack Pack별 300mS로 갱신되므로 해당 항목은 원천 갱신주기가 한계.)
- 변수명 통일: `ADCinAry` / `Avlt` / `Aoffset` / `PAnal`. 계수 표는 `2.3_Anl_Process`(수정안 파일 참조)에서 Master(EEPROM 시트)로 이관.

**MCU ADC 채널 / MX 설정 사양** (Pin Map 기준, MX 수정은 사용자 작업)

| ADC | 채널(Rank 순) | 변환 수 | DMA |
|---|---|---|---|
| ADC1 | 1=CH15(PC5) FWD, 2=CH14(PC4) RFL | 2 | DMA2 Stream0 Ch0, Circular, Half-Word |
| ADC2 | 1=CH8(PB0) DC Total, 2=CH11(PC1) Driver DC, 3=CH12(PC2) Pallet1 DC, 4=CH13(PC3) Pallet2 DC, 5=CH0(PA0) Pallet3 DC, 6=CH3(PA3) Pallet4 DC | 6 | DMA2 Stream3 Ch1, Circular, Half-Word |

- 공통: Independent mode, 12bit 우측정렬, Scan/Continuous/DMA Continuous Requests **Enable**, EOC=End of sequence, **Sampling 144 cycles**(ADC clock 21MHz = PCLK2/4, 채널당 약 7.4µs), 채널당 **16샘플 버퍼 평균**(ISR 불필요). DMA2 Stream2는 USART1/UART6 수신 후보라 회피.
- **현재 `.ioc`(0917 기준)와 차이**: ADC1=CH15 1개·ADC2=CH8 1개만 설정, RFL이 ADC2_IN14(Pin Map은 ADC1_IN14), Sampling 3 cycles → 위 사양으로 변경 필요.
- Gain/Phase Adjust Volt는 MCU ADC가 아니라 **MUX 수신값**(`MxAry(6)/(7)`)임(구 `2.3_ADC읽기`와 다름).

### 6-12-1. DAC out — 설계완료(코딩 대기)

| 항목 | 내용 |
|---|---|
| IC | DAC7512E × 2(U5=CH1, U6=CH2), Write-only 3-wire(SYNC+SCLK+DIN), 16bit MSB First |
| 출력 | RC(1KΩ+0.1uF) → `DAC5V_CH1`(Z4-13)/`DAC5V_CH2`(Z4-15). Gain/Phase Adjust Volt 출력용으로 추정(AGC Control, 18번과 직결) |

| 핀 | 신호 | MX 설정 |
|---|---|---|
| PD9 | SCLK(공유) | Output, **Push-Pull**, No pull, Medium |
| PD10 | DIN(공유) | Output, **Push-Pull**, No pull, Medium |
| PD11 | ENB1(U5 CS, Active Low) | Output, Push-Pull, **Pull-up**, Medium, 초기 High |
| PC8 | ENB2(U6 CS, Active Low) | Output, Push-Pull, **Pull-up**, Medium, 초기 High |

(EEPROM과 달리 Point-to-Point라 Open-Drain 아닌 Push-Pull. ENB Pull-up은 Reset 직후 오동작 래치 방지용)

**확인 필요**: ① CH1/CH2 = Gain/Phase 매핑 미확정 ② SCLK 최대 주파수/Setup·Hold Time 확인 후 Bit-bang Delay 결정

### 6-13. System Command Code (`2.2.1SysCMDcode`, PA unit 범위) — 0919

**코드 해독**: `그룹 = code/1000`(2=Offset, 3=ENB/Dis, 4=Level, 5=Delay), `Unit = (code%1000)/100`(**4=PA**), `항목 = code%100`(1~34). 예) 4407 → Level, 항목 7(DC Pack3 Volt). 범위 검사만으로 파싱 가능. Data는 Offset/ENB·Dis/Level/Delay = 16bit Word(Data L), ADC 다항식 계수 = 32bit float(L Word+H Word).

**System Control Code** (Target 888, Data 0):

| Code | 명령 | PA unit 처리 |
|---|---|---|
| 750 | Standby | `Cod_STBcmdIn`=H, `Cod_RFONcmdIn`=L, `Cod_OffcmdIn`=L |
| 720 | RF On | `Cod_STBcmdIn`=H, `Cod_RFONcmdIn`=H (RF On 출력은 2초 지연) |
| 710 | Off | `Cod_STBcmdIn`=L, `Cod_RFONcmdIn`=L, `Cod_OffcmdIn`=H |
| 790 | Reset | `Cod_RSTcmdIn`=H (1회성) |
| 777 | Factory Set | `Cod_FactSetReq`=H (1회성, EEPROM Default 기록 — 범위 보류) |
| 875/850/820, 300/400 | 75·50·25% Set, Power 증/감 | **PA unit 해당 없음**(수신 시 무시) |

**다항식 Command Code 없음**: 표에는 `Ofset(n)`만 있고 계수를 쓰는 Command Code가 없음 → 코드 정의 필요(예: 6000+n) — **보류**. (EEPROM 1차원화 후 Index 규칙과 함께 정리)

**Alarm 기본값 점검(원본 수정 필요)**: Pallet 2~4의 Level/Delay가 Pallet1 대비 한 칸씩 밀려 있음, 5429(Pallet4 DC Current Delay) 공란, Level이 1개뿐이라 **상한 초과만 알람인지**(하한 알람 필요 여부) 확인 필요.

---

## 7. 데이터 처리

| No | 항목 | 상태 |
|---|---|---|
| 13 | Analog 값 환산 처리 | **1차 확정(0921, 6-12절)**. 31항목 × 6계수 float32, Cold Start 시 계수 RAM 로드. 계수 표 정리안: X=Avlt(V) 통일 / 수식의 ×10·×0.05·×1.01·×0.925 배율을 계수에 반영 / DC Pack V·I는 BYPASS / 부호·오타 정정(out FWD 3차 162.7, Pallet FWD 3차 −511.93). **Master = EEPROM 시트**. 값은 실측 전 임의값. **Y 단위·스케일은 Alarm 항 정리 시 확정(보류)** — 확인 사항: DC Pack 전류가 `3.3DCpack통신`에서 ×1 정수인데 Alarm Level 기본값(500대)은 ×10 스케일로 보임 |
| 14 | Analog 환산 구현안(LUT) | 소스 3종(ADC/MUX/DCPACK) × 처리 2종(POLY5/BYPASS) 구조체 Lookup Table + 공통 계산 루틴으로 통합 예정. 구 `3.1AnalProSS`(25채널)는 `2.3_Anl_Process`(31항목)로 대체된 것으로 보이며 정리 필요 |
| 15 | Alarm Setting | 기준값 이상/이하 지속시간 판정, Minor/Major, Latch/Unlatch — 설계진행중, **다음 작업 대상**(아래 "Alarm 항 정리 시 함께 확정" 참조) |
| 16 | Status Bit 출력 생성 | PA# Status Word bit 정의 — 설계 상세완료 |

**Alarm 항 정리 시 함께 확정할 것**: ① `PAnal`의 Y 단위·스케일 ② Alarm Latch 비트 순서(Heatsink는 Latch 13번째 비트, Proc 3번)↔Proc NO 매핑표 ③ `PA_Sts(45~54)` Offset·Interlock(4항목분) 유지 여부 ④ MUX Load Power×4·Pallet Status 미사용 확인 ⑤ Level 방향(상한만/하한 필요)

---

## 8. 인터벌 동작

| 주기 | 내용 |
|---|---|
| 3mS | Analog 처리 Call (FWD/RFL 매번 + 1항목 순환) |
| 100mS | DC Pack 통신 Call |
| 150mS | MUX board 송수신 Call — **확정**(원본 `3.인터벌동작` 시트 250mS → 정정 필요) |
| 200mS | IF board 전송 Call |

부하 분산 목적, 150mS는 Test 시 조정 가능. 모듈 결합(실제 Call 연결)은 아직 미완료. RF On 2초 지연 Timer의 틱 기준도 여기에 맞춰 정할 것.

---

## 9. AGC Control

PID 제어: Target/Current Volt 차이의 1/2씩 접근(2차 함수 근사), 10mS 간격, 5mV Threshold — 알고리즘 확정, 코딩 전.

---

## 10. 데이터/설정 테이블

| No | 항목 | 상태 |
|---|---|---|
| 19 | PA Data List | 설계완료 |
| 20 | PA Status Table | File1/File2 항목수(Word6~7) 차이 — 반영 여부 확인 필요 |
| 21 | CPU Pin Table | PB12(SPI CS) MX setting 미지정 확인 필요. EEPROM 핀 변경(PE14/15)이 원본 핀맵에 반영 안 됨. **ADC 채널 배치는 Pin Map 기준(0921)**, `.ioc`는 6-12절 사양으로 수정 필요 |
| 22 | Command Code 체계 | **`2.2.1SysCMDcode` 기준**(Offset 2xxx/ENB·Dis 3xxx/Level 4xxx/Delay 5xxx, PA unit=24xx~54xx). 기존 22번 체계·`2.5PC_Mon통신`의 별도 번호는 **정렬 필요**. `00_작업진행`의 `444` 등 옛 코드 잔재는 사용자가 정리 |

---

## 11. 상위설계 (참고)

| No | 항목 | 요지 |
|---|---|---|
| 23 | PA Address Setting | RCU/Rack No, 전면 Dip/로타리 SW(0~9만 설정 가능) → **0921: 전체 Address는 IF board가 전달, EEPROM 구획1에 저장**(4절) |
| 24 | Nor/War/Alarm 설계 | 평균화·최종 판정은 IF board/RCU 레벨 — **PA unit Firmware 범위 아님**(12/15/16번이 대응분) |
| 25 | Main CPU HW 설계 | 설계초안 |
| 26 | Code Tree | 초안 |

---

## 전체 미해결 항목 요약

1. **6번 PC Monitor**: "Default" 명령 기능 미구현, Target 값(500 임의값) 확정. **PA Address 전달 방법**(IF board → PA, 미설정 시 수락 Target, EEPROM 공백 초기값·유효성)은 4절 구획1 보류 항목과 함께 확정
2. **10번 GPIO 상태입력 처리**: 설계초안 단계, 미착수
3. **11번 Command/Control**: `cmd_control.c`에 PAfaultSts·RF On 2초 지연·Code 명령 처리 반영 / OFF 조건(H&H vs L&L) 정정 / **5.3 Status in** 원본 미작성
4. **12번 Analog Process**: 코딩 대기 — MX ADC 설정 수정(사용자), `.ioc` RFL 채널·Sampling 정정. **Y 단위·스케일은 Alarm 항 정리 시 확정**
5. **12-1 DAC out**: CH1/CH2 Gain/Phase 매핑, SCLK 타이밍 스펙 확인
6. **EEPROM 시트 갱신(사용자 작업)**: 1차원화(순서 확정 후 바이트 주소 반영 — 구획2 기준 주소는 `0x0400~`), 4구획 메모리 맵(구획1 공통 정보 신설) 반영, Index 0부터 정리, ENB/Dis Value 1/0 방식·초기값, Spare 32~34 기본값, 777 범위, 다항식 계수 Master 반영
7. **다항식 계수 쓰기용 Command Code 정의**(예: 6000+n), Command Code 체계 정렬(22번): SysCMDcode ↔ 2.5PC_Mon통신 ↔ 00_작업진행, Offset 범위(UI 1~100% vs 사양 1~200%)
8. **Alarm 항 정리(15번, 다음 작업)**: 7절 하단 확정 목록 ①~⑤
9. **원본 시트 정정(사용자)**: `3.인터벌동작` MUX 250mS→150mS, `2.2Cmd_Control` OFF 조건·번호 중복, `00_작업진행` 444 잔재
10. **Command 패킷 D07 명칭**(Target FWD Power vs 동작상태) 및 2nd Target 비교 대상 확인
11. **9번 DC Pack**: I2C3_ENB(PD0) Active Level 확인
12. **20번 PA Status Table**: File1/File2 Word 항목수 차이 반영 여부
13. **21번 CPU Pin Table**: PB12 배선 확정, EEPROM 핀 변경 원본 반영
14. **버전 조회 프로토콜 없음**(Monitor/Boot/App Ver), PC Mon 5.1 응답/ACK 형식 미작성
15. PE14/PE15 GPIO Speed(Very High로 설정됨, Low/Medium 권장) — 사소, 성능에 큰 영향 없음
