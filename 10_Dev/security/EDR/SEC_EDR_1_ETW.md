


# SEC_EDR_1_ETW

## 🧠 핵심 개념 한 줄 요약
ETW(Event Tracing for Windows)는 커널 드라이버 없이도 Windows가 이미 만들어둔 프로세스/파일/네트워크 이벤트 스트림을 유저 모드에서 구독(subscribe)해서 실시간으로 받아볼 수 있는 이벤트 추적 인프라다.

---

## 📦 핵심 개념 상세

### ETW란 (비유)
Windows 내부에는 이미 깔려 있는 **CCTV 배선망**이 있다.
- **Provider(제공자)** = CCTV 카메라 — 커널/시스템 서비스가 이벤트를 발생시키는 주체 (예: `Microsoft-Windows-Kernel-Process`)
- **Session(세션)** = 녹화 채널 — 어떤 Provider의 어떤 이벤트를 수집할지 설정한 통로
- **Consumer(소비자)** = 모니터 보는 사람 — 세션에 연결해서 실시간으로 이벤트를 받는 프로그램 (우리가 만든 코드)

핵심: **코드를 주입하거나 후킹하는 게 아니라, 이미 존재하는 이벤트 스트림을 구독만 하는 것**. 그래서 커널 드라이버보다 개발/배포가 훨씬 쉽다 (서명 요구사항, 블루스크린 리스크 없음).

### ETW의 한계
- **관찰(observe)만 가능, 차단(block) 불가능** — "프로세스 실행을 막아라" 같은 실시간 차단은 3단계 커널 드라이버(미니필터, `PsSetCreateProcessNotifyRoutine`)의 영역
- 그래서 실제 EDR은 **ETW(관찰) + 커널 드라이버(차단)** 조합으로 구성됨

### 실습 흐름 (2단계)
1. **`logman`으로 커맨드라인 실습**: 코드 없이 ETW 세션을 만들고 `.etl` 파일로 저장 → `tracerpt`로 XML 변환해서 눈으로 이벤트 구조 확인
   ```cmd
   logman create trace MyProcTrace -p "Microsoft-Windows-Kernel-Process" 0x10 0x5 -o C:\temp\proctrace.etl
   logman start MyProcTrace
   :: (여기서 프로세스 실행/종료시켜서 이벤트 발생)
   logman stop MyProcTrace
   tracerpt C:\temp\proctrace.etl -o C:\temp\proctrace.xml -of XML
   ```
   - **EventID 1** = 프로세스 생성, **EventID 2** = 프로세스 종료
   - 핵심 필드: `ProcessID`, `ParentProcessID`, `ImageName`

2. **Raw Win32 ETW API로 C++ Consumer 작성**
   - `OpenTrace` (채널 맞추기) → `ProcessTrace` (재생 시작, 블로킹 함수) → 콜백(`EventRecordCallback`)이 이벤트마다 자동 호출
   - 필드 파싱은 `TdhGetEventInformation` + `TdhGetProperty` (TDH = Trace Data Helper)
   - **중요**: 필드마다 실제 타입(`InType`)이 다름 (`UINT32`, `UNICODESTRING` 등) — 전부 문자열로 캐스팅하면 `ProcessID` 같은 정수 필드가 빈 값/깨진 값으로 출력됨. 반드시 `TDH_INTYPE_*`로 분기 처리 필요

### 링크(link) 개념 정리 (이번 실습에서 파생된 핵심 인사이트)
- `#include`는 **선언**만 가져올 뿐 절대 링크를 하지 않는다 (컴파일 단계 vs 링크 단계는 별개)
- MSVC는 `kernel32.lib`, `advapi32.lib` 등 핵심 시스템 lib 일부를 **기본 링크 목록**에 포함하지만, `tdh.lib`는 포함되지 않아 명시적 링크 필요
- 다만 기본 링크 목록 포함 여부는 **빌드 환경(버전, 아키텍처)에 따라 달라질 수 있음** — 실제로 `advapi32.lib`도 이번 환경(MSVC 19.51, x86)에서는 명시적 링크가 필요했음 (`OpenTraceW`, `ProcessTrace`, `CloseTrace` 미해결 심볼 에러)
- **라이브러리 소속 확인 방법**: MSDN(Microsoft Learn) 문서의 함수별 "Requirements" 섹션에서 `Library:` 항목 확인이 가장 확실

---

## 🔗 EDR과의 연관성
- 실제 EDR 제품(Sysmon, Microsoft Defender for Endpoint의 텔레메트리 일부 등)이 프로세스 생성/이미지 로드 이벤트를 수집하는 핵심 방식 중 하나가 ETW다
- **Sysmon = ETW를 잘 정리해서 보여주는 도구 + 자체 커널 콜백**의 조합 (4단계 오픈소스 리딩에서 다시 연결됨)
- `ParentProcessID` + `ImageName` 조합은 "정상 프로세스로 위장한 공격" 탐지의 출발점 (예: 워드 문서 매크로에서 `cmd.exe`가 실행되는 패턴 — MITRE ATT&CK `T1204` 사용자 실행 기법과 연결)
- `ProcessTrace`는 블로킹 함수라서 실무에서는 **별도 스레드**에서 돌리고 메인 스레드는 제어 로직 담당 — 멀티스레딩 지식과 직결 (C++ 학습 프로젝트 내용과 연결)

---

## ⚠️ 자주 하는 오해/실수
- ETW가 코드 인젝션/후킹이라는 오해 → ❌ 이미 존재하는 이벤트 스트림 구독일 뿐
- ETW로 프로세스 실행을 막을 수 있다는 오해 → ❌ 순수 관찰용, 차단은 커널 드라이버 영역
- 관리자 권한 없이 커널 Provider 구독 가능하다는 오해 → ❌ 대부분 관리자 권한 필요
- 이벤트 필드를 전부 문자열로 캐스팅해서 파싱 → ❌ 타입별(`InType`) 분기 처리 필수, 안 그러면 정수 필드가 빈 값으로 나옴
- `#include`만 하면 자동으로 링크된다는 오해 → ❌ 헤더(선언)와 lib(구현)는 완전히 별개 메커니즘
- 실습 삽질 기록: `PTRACE_EVNET_INFO`(오타) / `eventId`·`eventID` 대소문자 불일치 / `EVENT_TRACE_LOGFILEW`의 실제 필드명은 `LogFileName`(W 접미사 없음) / `advapi32.lib` 명시적 링크 누락 → 전부 실제로 만난 에러들, 향후 유사 에러 시 참고

---

## 🔨 컴파일 명령어
```cmd
cl /EHsc /std:c++17 main.cpp tdh.lib advapi32.lib
```
- 한글 주석 포함 시 파일을 **UTF-8 with BOM**으로 저장할 것 (안 그러면 C4819 경고)
- 관리자 권한 명령 프롬프트에서 실행 필요 (`logman`, 실시간 세션 구독 시)

---

## 🔗 연결 노트
- 이전: [[SEC_OSInternals_4_ProcessInjectionConcepts]]
- 다음: [[SEC_EDR_1_ETW]]

---

#security #edr #windows-internals #etw #cpp
