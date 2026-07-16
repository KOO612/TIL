---
tags: [security, edr, windows-internals, dll-enumeration, registry-monitoring, cpp]
---

# SEC_EDR_5 — DLL 열거 + 레지스트리 감시 코드 실습

## 🧠 핵심 개념 한 줄 요약
`CreateToolhelp32Snapshot(TH32CS_SNAPMODULE, pid)`로 특정 프로세스의 로드 DLL을 열거하고, `RegNotifyChangeKeyValue`로 레지스트리 Run 키의 변경을 **이벤트 기반(폴링 아님)**으로 실시간 감시할 수 있다.

---

## 📦 핵심 개념 상세

### 1. DLL 열거 — `TH32CS_SNAPMODULE`
- 프로세스 스냅샷(`TH32CS_SNAPPROCESS`)과 동일한 API를 쓰되, **대상 PID를 반드시 함께 넘겨야** 함 (`TH32CS_SNAPPROCESS`는 PID=0으로 전체 스캔 가능하지만 모듈 스냅샷은 "어느 프로세스"인지 지정 필요).
- 각 모듈의 이름, 전체 경로, 베이스 주소, 크기를 확인 가능.
- 실전 EDR에서는 여기서 얻은 경로로 ① 신뢰 경로 여부 확인 ② 디지털 서명 검증(`WinVerifyTrust`, 추후 단계) ③ 화이트리스트 대조를 이어서 수행.

### 2. 레지스트리 감시 — `RegNotifyChangeKeyValue`
- `RegOpenKeyExW`로 키를 열 때 `KEY_NOTIFY`(감시용) + `KEY_READ`(조회용) 권한을 함께 요청.
- `RegNotifyChangeKeyValue`는 **호출 시점에 값이 바뀔 때까지 블로킹 대기**하는 방식이라 폴링이 아니며, 변화가 실제로 일어난 순간에만 깨어남 → CPU 낭비 없음.
- `REG_NOTIFY_CHANGE_LAST_SET` 옵션으로 값 추가/삭제/수정을 감지.
- `bWatchSubtree` 인자를 `TRUE`로 주면 하위 키까지 재귀적으로 감시 (`FALSE`는 지정한 키 자체만).

### 3. 두 기능을 잇는 실전 흐름
```
1. Run 키 변경 감지 (registry_watch)
2. 변경된 값에서 실행 파일 경로 추출
3. 그 파일이 이미 실행 중이라면 PID 확인 후 dll_enum으로 로드 모듈 검사
4. 이상 경로/미서명 DLL 발견 시 경고
```

---

## 💻 코드 예제 1: 특정 프로세스의 로드된 DLL 목록 확인

```cpp
// dll_enum.cpp
// 특정 PID가 로드한 DLL 목록을 나열하는 예제
// 컴파일: cl.exe /EHsc /std:c++17 dll_enum.cpp
//        또는 g++ -std=c++17 dll_enum.cpp -o dll_enum.exe

#include <windows.h>
#include <tlhelp32.h>
#include <iostream>
#include <string>

// 지정한 PID가 로드한 모든 DLL(모듈) 목록을 출력하는 함수
void ListModulesForProcess(DWORD pid)
{
    // TH32CS_SNAPMODULE: 지정한 PID의 모듈(DLL) 목록만 스냅샷
    // 반드시 PID를 함께 넘겨야 함 (프로세스 스냅샷과의 차이점)
    HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPMODULE | TH32CS_SNAPMODULE32, pid);

    if (hSnapshot == INVALID_HANDLE_VALUE)
    {
        // 실패 원인 대부분: 대상 프로세스가 이미 종료됐거나, 접근 권한 부족
        std::wcerr << L"모듈 스냅샷 실패 (PID=" << pid << L"), 오류코드=" << GetLastError() << L"\n";
        return;
    }

    MODULEENTRY32W me32{};
    me32.dwSize = sizeof(MODULEENTRY32W); // 구조체 크기 설정 필수

    if (Module32FirstW(hSnapshot, &me32))
    {
        do
        {
            // 모듈 이름, 로드된 전체 경로, 로드 시작 주소, 크기를 출력
            std::wcout << L"  [모듈] " << me32.szModule
                       << L" | 경로: " << me32.szExePath
                       << L" | 베이스주소: 0x" << std::hex << (uintptr_t)me32.modBaseAddr
                       << std::dec << L" | 크기: " << me32.modBaseSize << L" bytes\n";

            // 실전 EDR이라면 여기서:
            // 1) me32.szExePath가 System32/신뢰 경로인지 확인
            // 2) 디지털 서명 검증 (WinVerifyTrust API, 추후 단계에서 학습)
            // 3) 화이트리스트에 없는 경로면 경고 처리

        } while (Module32NextW(hSnapshot, &me32));
    }
    else
    {
        std::wcerr << L"모듈 열거 실패, 오류코드=" << GetLastError() << L"\n";
    }

    CloseHandle(hSnapshot);
}

int main(int argc, char* argv[])
{
    if (argc < 2)
    {
        std::wcout << L"사용법: dll_enum.exe <PID>\n";
        std::wcout << L"예시: dll_enum.exe 1234\n";
        return 1;
    }

    DWORD pid = static_cast<DWORD>(std::stoul(argv[1]));
    std::wcout << L"=== PID " << pid << L" 의 로드된 모듈 목록 ===\n";
    ListModulesForProcess(pid);

    return 0;
}
```

**사용법**: 작업관리자에서 PID 확인 후 `dll_enum.exe 1234` 형태로 실행.

---

## 💻 코드 예제 2: 레지스트리 Run 키 변경 감시

```cpp
// registry_watch.cpp
// HKCU\Software\Microsoft\Windows\CurrentVersion\Run 키의 변경을 실시간 감시
// 컴파일: cl.exe /EHsc /std:c++17 registry_watch.cpp
//        또는 g++ -std=c++17 registry_watch.cpp -o registry_watch.exe -ladvapi32

#include <windows.h>
#include <iostream>
#include <string>

// 지정한 레지스트리 키의 현재 값들을 모두 출력하는 함수 (변경 감지 시 호출)
void PrintRunKeyValues(HKEY hKey)
{
    DWORD index = 0;
    wchar_t valueName[256];
    DWORD valueNameSize;
    wchar_t valueData[1024];
    DWORD valueDataSize;
    DWORD type;

    std::wcout << L"  --- 현재 Run 키 값 목록 ---\n";

    // RegEnumValueW로 키 안의 모든 값(자동 실행 등록된 프로그램들)을 순회
    while (true)
    {
        valueNameSize = sizeof(valueName) / sizeof(wchar_t);
        valueDataSize = sizeof(valueData);

        LONG result = RegEnumValueW(hKey, index, valueName, &valueNameSize,
                                     nullptr, &type, (LPBYTE)valueData, &valueDataSize);

        if (result == ERROR_NO_MORE_ITEMS) break; // 더 이상 값 없으면 종료
        if (result != ERROR_SUCCESS)
        {
            std::wcerr << L"  값 열거 실패, 오류코드=" << result << L"\n";
            break;
        }

        // 값 이름 = 임의의 라벨, 값 데이터 = 실제로 실행될 파일 경로+인자
        std::wcout << L"  [자동실행] " << valueName << L" = " << valueData << L"\n";

        index++;
    }
}

int main()
{
    HKEY hKey;

    // HKCU\...\Run 키 열기 (감시 + 값 조회를 위한 권한 요청)
    LONG openResult = RegOpenKeyExW(
        HKEY_CURRENT_USER,
        L"Software\\Microsoft\\Windows\\CurrentVersion\\Run",
        0,
        KEY_NOTIFY | KEY_READ,   // KEY_NOTIFY: 변경 감시용, KEY_READ: 값 조회용
        &hKey);

    if (openResult != ERROR_SUCCESS)
    {
        std::wcerr << L"레지스트리 키 열기 실패, 오류코드=" << openResult << L"\n";
        return 1;
    }

    std::wcout << L"=== Run 키 감시 시작 (Ctrl+C로 종료) ===\n";

    // 감시 시작 전, 초기 상태를 한 번 보여줌
    PrintRunKeyValues(hKey);

    while (true)
    {
        // RegNotifyChangeKeyValue: 이 키에 변화가 생길 때까지 여기서 "대기"함 (폴링 아님!)
        // REG_NOTIFY_CHANGE_LAST_SET: 값이 추가/삭제/수정될 때 알림
        LONG notifyResult = RegNotifyChangeKeyValue(
            hKey,
            FALSE,                          // 하위 키까지 감시할지 여부
            REG_NOTIFY_CHANGE_LAST_SET,      // 값 변경 감지 옵션
            nullptr,                         // 이벤트 핸들 (동기 대기이므로 nullptr)
            FALSE);                          // 비동기 여부 (FALSE = 동기, 여기서 블로킹됨)

        if (notifyResult != ERROR_SUCCESS)
        {
            std::wcerr << L"감시 실패, 오류코드=" << notifyResult << L"\n";
            break;
        }

        // 여기 도달했다는 건 = 실제로 Run 키에 변화가 생겼다는 뜻
        std::wcout << L"\n[변경 감지!] Run 키에 변화가 발생했습니다.\n";
        PrintRunKeyValues(hKey); // 변경 후 최신 상태 다시 출력
    }

    RegCloseKey(hKey);
    return 0;
}
```

### 🔨 컴파일 방법

```powershell
# MSVC
cl.exe /EHsc /std:c++17 dll_enum.cpp
cl.exe /EHsc /std:c++17 registry_watch.cpp

# MinGW
g++ -std=c++17 dll_enum.cpp -o dll_enum.exe
g++ -std=c++17 registry_watch.cpp -o registry_watch.exe -ladvapi32
```

**테스트 방법**: `registry_watch.exe`를 실행해두고, 아무 프로그램의 "시작 시 자동 실행" 옵션을 켜서 `[변경 감지!]` 로그 확인.

---

## ⚠️ 자주 하는 오해/실수
1. `RegNotifyChangeKeyValue`가 한 번 감지 후 자동으로 계속 감시할 거라 생각 → **아님**. 알림을 받은 후에는 **다시 호출**해야 그 다음 변경도 감지됨 (코드에서 while 루프로 재호출하는 이유).
2. 모듈 스냅샷 실패 시 권한 문제만 의심 → 대상 프로세스가 이미 종료되었거나 32/64비트 아키텍처 불일치(예: 32비트 프로세스가 64비트 프로세스의 모듈을 열람하려는 경우)도 흔한 원인.

---

## 🔗 연결 노트
- 이전: [[SEC_EDR_4_DLLEnumRegistryMonitoring]]
- 다음: [[SEC_EDR_6_KernelModeConcepts]]
- 연관: [[SEC_EDR_3_ProcessSnapshotMonitoring]], [[SEC_EDR_2_APIHooking]]
