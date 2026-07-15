---
tags: [security, edr, windows-internals, dll-enumeration, registry-monitoring, mitre-attack]
---

# SEC_EDR_4 — DLL 열거 및 레지스트리 모니터링

## 🧠 핵심 개념 한 줄 요약
공격자는 주로 **정상 프로세스에 악성 DLL을 몰래 로드**시키거나 **레지스트리를 건드려 지속성(Persistence)을 확보**하므로, EDR은 DLL 목록과 레지스트리 변경을 상시 감시한다.

---

## 📦 핵심 개념 상세

### 1. DLL 열거 (Module Enumeration)
- `CreateToolhelp32Snapshot`을 프로세스 감시와 동일하게 사용하되 플래그만 다르게 지정.

```
TH32CS_SNAPPROCESS  → 프로세스 목록 스냅샷
TH32CS_SNAPMODULE   → 특정 프로세스가 로드한 모듈(DLL) 목록 스냅샷
```

- 비유: 프로세스가 "직원"이라면 DLL은 그 직원이 "빌려 쓰는 도구함". 정상 회계 프로그램 직원이 갑자기 해킹 도구함을 빌려 쓰면 의심스러움.

**EDR 활용 방식**
- 화이트리스트/블랙리스트 경로 비교: `%TEMP%`, 다운로드 폴더 등 이상 경로에서 로드된 DLL은 의심 신호.
- 디지털 서명 검증과 결합 (서명 검증은 이후 단계에서 학습 예정).
- DLL 사이드로딩 탐지: `version.dll`, `dbghelp.dll` 등 자주 악용되는 이름으로 위장한 악성 DLL 탐지에 활용.

### 2. 레지스트리 모니터링 (Registry Monitoring)
- 레지스트리 = Windows 설정을 저장하는 거대한 데이터베이스.
- `RegOpenKeyEx`, `RegQueryValueEx`로 값 조회, `RegNotifyChangeKeyValue`로 **이벤트 기반** 변경 감시 가능.
- 비유: 특정 서랍(레지스트리 키)에 손을 대면 알람이 울리는 센서를 달아두는 것. 폴링처럼 매 순간 확인할 필요 없음.

**EDR이 주목하는 주요 경로**

| 경로 | 의미 |
|---|---|
| `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` | 로그인 시 자동 실행 (지속성 확보 대표 위치) |
| `HKLM\Software\Microsoft\Windows\CurrentVersion\Run` | 위와 동일, 모든 사용자 대상 |
| `HKLM\System\CurrentControlSet\Services` | 서비스 등록 (서비스 위장 지속성) |
| `HKCU\...\Explorer\Shell Folders` | 탐색기 경로 변조 (지속성/은닉 악용) |

- 이 경로들은 MITRE ATT&CK **T1547 (Boot or Logon Autostart Execution)**과 직접 연결됨.

### 3. 두 기능의 결합 (Correlation의 기초)
```
1. 레지스트리 Run 키에 새 값 등록 감지 (RegNotifyChangeKeyValue)
2. 해당 값이 가리키는 실행 파일 경로 확인
3. 그 실행 파일이 로드하는 DLL 목록 확인 (TH32CS_SNAPMODULE)
4. 서명 안 된 DLL / 이상 경로 DLL 발견 시 → 경고/차단
```
- "이벤트들을 연결해 하나의 공격 스토리로 재구성"하는 것을 EDR 업계에서 **Correlation(상관관계 분석)**이라 부름 (추후 심화 예정).

---

## 🔗 EDR과의 연관성
- DLL 열거 + 레지스트리 감시는 각각 **Defense Evasion(방어 회피)** 및 **Persistence(지속성)** 전술 탐지의 핵심 축.
- 지속성 확보 방법은 Run 키 외에도 예약 작업, 서비스 등록, WMI 이벤트 구독 등 T1547 하위에 10개 이상 세부 기법 존재 — 단일 경로 감시로는 불충분.

---

## ⚠️ 자주 하는 오해/실수
1. "레지스트리 Run 키만 감시하면 지속성 공격을 다 잡는다" → **아님**. 예약 작업/서비스/WMI 등 다양한 경로 존재.
2. "DLL 경로가 System32면 무조건 안전하다" → **아님**. 관리자 권한 탈취 시 System32에도 악성 DLL 심을 수 있음 — 경로 검사는 1차 필터일 뿐, 서명 검증이 더 중요.

---

## 💻 코드 예제
(다음 세션에서 DLL 열거 + 레지스트리 Run 키 감시 코드 예제 제공 예정)

---

## 🔗 연결 노트
- 이전: [[SEC_EDR_3_ProcessSnapshotMonitoring]]
- 다음: (DLL 열거 + 레지스트리 감시 코드 실습 — 예정)
- 연관: [[SEC_EDR_2_APIHooking]]
