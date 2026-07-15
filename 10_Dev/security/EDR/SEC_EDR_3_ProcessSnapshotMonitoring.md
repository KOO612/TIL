---
tags: [security, edr, windows-internals, process-monitoring, mitre-attack]
---

# SEC_EDR_3 — 프로세스/스레드 생성 감시 (CreateToolhelp32Snapshot)

## 🧠 핵심 개념 한 줄 요약
`CreateToolhelp32Snapshot`은 특정 순간의 프로세스 목록을 "사진 찍듯" 통째로 가져오는 **폴링(polling) 방식** 감시 기법이며, ETW(실시간 push 방식)보다 구현은 쉽지만 정교함은 떨어진다.

---

## 📦 핵심 개념 상세

### 1. 스냅샷 방식의 동작 원리
```
1. CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0)
   → 그 순간의 모든 프로세스 목록을 스냅샷으로 생성

2. Process32First() / Process32Next()
   → 스냅샷 안의 프로세스를 하나씩 순회 (PID, 이름, 부모 PID 등)
```
- "실시간 이벤트 알림"이 아니라 **찍는 순간의 상태**만 알려줌.
- 신규 프로세스 생성을 감지하려면 **일정 주기로 반복해서 이전 스냅샷과 비교**해야 함.

```
1초 후: 스냅샷 A → PID {100, 200, 300}
2초 후: 스냅샷 B → PID {100, 200, 300, 400}
                                     ↑
                          비교 결과 400이 신규 → "새 프로세스 생성!"
```

### 2. ETW와의 비교

| 항목 | ETW | Toolhelp 스냅샷 |
|---|---|---|
| 방식 | 이벤트 발생 즉시 통지(push) | 주기적으로 확인(polling) |
| 실시간성 | 높음 | 낮음 (폴링 간격 사이 이벤트 놓칠 수 있음) |
| 구현 난이도 | 상대적으로 복잡 | 매우 쉬움 |
| 실전 활용 | EDR 코어 엔진 | 보조 도구, 초기 인벤토리 조회 |

### 3. 실전 함정
- 폴링 간격보다 짧게 **생성 후 즉시 종료**되는 프로세스(예: 악성코드가 실행 직후 스스로 종료해 흔적을 지우는 경우)는 스냅샷 방식으로 **놓칠 수 있음**.
- 그래서 실전 EDR은 이 방식을 단독으로 쓰지 않고, ETW/커널 콜백(3단계 예정)과 함께 사용.

---

## 🔗 EDR과의 연관성
- EDR 에이전트 시작 시 **현재 실행 중인 전체 프로세스 인벤토리** 파악 용도로 자주 사용.
- `th32ParentProcessID`로 **부모-자식 프로세스 트리 추적** → 이상 패턴 탐지의 기초.
  - 예: `winword.exe`(정상 문서 프로그램)의 자식으로 `powershell.exe`가 실행되는 경우 → 매크로 기반 공격 의심 (MITRE ATT&CK **T1059.001 PowerShell**).

---

## ⚠️ 자주 하는 오해/실수
1. 스냅샷 방식이면 실시간 감시가 충분하다 → **아님**. 폴링 간격 사이의 짧은 생성/종료는 놓칠 수 있어 ETW/커널 콜백과 병행 필요.
2. 프로세스 개수만 세면 된다 → **아님**. PID는 재사용될 수 있으므로 이름+생성시각 등 추가 식별자로 교차 검증 필요 (실전 구현 시 유의).

---

## 💻 코드 예제: 프로세스 목록 + 신규 생성 감시

```cpp
// process_watch.cpp
// CreateToolhelp32Snapshot을 이용한 프로세스 목록 조회 및 신규 프로세스 감시
// 컴파일: cl.exe process_watch.cpp (Windows, MSVC)
//        또는 g++ process_watch.cpp -o process_watch.exe (MinGW)

#include <windows.h>
#include <tlhelp32.h>
#include <iostream>
#include <unordered_map>
#include <string>
#include <thread>
#include <chrono>

// PID -> 프로세스 이름 저장용 (이전 스냅샷 상태 기억)
using ProcessMap = std::unordered_map<DWORD, std::wstring>;

// 현재 시점의 프로세스 목록을 스냅샷으로 가져오는 함수
ProcessMap TakeSnapshot()
{
    ProcessMap result;

    // TH32CS_SNAPPROCESS: 프로세스 목록만 스냅샷에 포함
    HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    if (hSnapshot == INVALID_HANDLE_VALUE)
    {
        std::wcerr << L"스냅샷 생성 실패\n";
        return result;
    }

    PROCESSENTRY32W pe32{};
    pe32.dwSize = sizeof(PROCESSENTRY32W); // 구조체 크기 반드시 설정 (Win32 API 관례)

    // 스냅샷의 첫 번째 프로세스부터 순회 시작
    if (Process32FirstW(hSnapshot, &pe32))
    {
        do
        {
            // PID를 key로, 프로세스 이름을 value로 저장
            result[pe32.th32ProcessID] = pe32.szExeFile;

            // 부모 PID도 함께 확인 가능 (프로세스 트리 추적에 활용)
            // pe32.th32ParentProcessID
        } while (Process32NextW(hSnapshot, &pe32));
    }

    CloseHandle(hSnapshot); // 스냅샷 핸들은 반드시 해제 (핸들 누수 방지)
    return result;
}

int main()
{
    std::wcout << L"=== 프로세스 감시 시작 (Ctrl+C로 종료) ===\n";

    // 최초 스냅샷: 현재 실행 중인 프로세스 목록 저장
    ProcessMap previous = TakeSnapshot();
    std::wcout << L"[초기 상태] 현재 프로세스 개수: " << previous.size() << L"\n";

    while (true)
    {
        // 1초 대기 후 새 스냅샷 (폴링 간격 - 짧을수록 정확하지만 CPU 부하 증가)
        std::this_thread::sleep_for(std::chrono::seconds(1));

        ProcessMap current = TakeSnapshot();

        // 신규 생성된 프로세스 탐지: current에는 있는데 previous에는 없는 PID
        for (const auto& [pid, name] : current)
        {
            if (previous.find(pid) == previous.end())
            {
                std::wcout << L"[생성 감지] PID=" << pid
                           << L" 이름=" << name << L"\n";
            }
        }

        // 종료된 프로세스 탐지: previous에는 있는데 current에는 없는 PID
        for (const auto& [pid, name] : previous)
        {
            if (current.find(pid) == current.end())
            {
                std::wcout << L"[종료 감지] PID=" << pid
                           << L" 이름=" << name << L"\n";
            }
        }

        previous = std::move(current); // 다음 비교를 위해 현재 상태를 이전 상태로 갱신
    }

    return 0;
}
```

### 🔨 컴파일 방법 (Windows)

```powershell
# MSVC (Visual Studio Developer Command Prompt에서)
cl.exe /EHsc /std:c++17 process_watch.cpp

# 또는 MinGW
g++ -std=c++17 process_watch.cpp -o process_watch.exe
```

실행 후 메모장을 켜고 끄면서 `[생성 감지]`, `[종료 감지]` 로그 확인.

---

## 🔗 연결 노트
- 이전: [[SEC_EDR_2_APIHooking]]
- 다음: [[SEC_EDR_4_DLLEnumRegistryMonitoring]]
- 연관: [[SEC_EDR_1_ETW]]
