# CPP Win32 1 — API 기초 (HWND, HANDLE)

#cpp #win32 #windows #handle #hwnd #raii

---

## 🧠 핵심 개념 한 줄 요약

> **HANDLE = OS가 관리하는 자원을 가리키는 불투명한 번호표 — 내부 구조는 몰라도 됨**

---

## 📦 HANDLE / HWND 개념

```cpp
HANDLE hFile;    // 파일, 프로세스, 스레드, 뮤텍스 등 모든 커널 객체의 핸들
HWND   hWnd;     // 윈도우(창)를 가리키는 핸들 (HANDLE의 특수화된 버전)
HINSTANCE hInst; // 실행 중인 프로그램 인스턴스(모듈)를 가리키는 핸들
HDC    hDC;      // 디바이스 컨텍스트(그리기 대상) 핸들
```

| 타입 | 의미 | 실무 사용처 |
|------|------|------------|
| `HANDLE` | 커널 객체(파일, 프로세스, 스레드, 뮤텍스, 이벤트) 범용 핸들 | `CreateFile`, `CreateThread`, `CreateMutex` |
| `HWND` | 윈도우(창) 핸들 — 모든 UI 요소는 각각 HWND를 가짐 | `CreateWindow`, `FindWindow`, `SendMessage` |
| `HINSTANCE` | 실행 파일(.exe/.dll) 모듈 핸들 | `WinMain` 매개변수, 리소스 로딩 |
| `HDC` | 그리기 작업 대상(화면/프린터/메모리) 핸들 | `GetDC`, `BeginPaint`, `TextOut` |

```cpp
// Windows SDK 내부 정의 (참고용)
typedef void* HANDLE;
typedef HANDLE HWND;  // HWND도 결국 HANDLE의 한 종류
```

> **실무 핵심 규칙**: HANDLE 값을 직접 해석하려 하지 말 것 — 블랙박스로 취급하고 오직 API 함수를 통해서만 사용

---

## 🪟 최소 Win32 윈도우 프로그램 구조

```cpp
#include <windows.h>

// 윈도우 프로시저 - 모든 이벤트(클릭, 닫기, 그리기 등)를 처리하는 콜백 함수
LRESULT CALLBACK WindowProc(HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam) {
    switch (msg) {
    case WM_DESTROY:
        PostQuitMessage(0); // 창이 닫힐 때 → 프로그램 종료 요청
        return 0;
    case WM_PAINT:
        {
            PAINTSTRUCT ps;
            HDC hdc = BeginPaint(hwnd, &ps);
            TextOutW(hdc, 50, 50, L"Hello Win32!", 12);
            EndPaint(hwnd, &ps);
        }
        return 0;
    }
    return DefWindowProc(hwnd, msg, wParam, lParam); // 미처리 메시지는 기본 처리기로
}

int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance,
                    LPSTR lpCmdLine, int nCmdShow) {
    const wchar_t CLASS_NAME[] = L"MyWindowClass";

    WNDCLASSW wc = {};
    wc.lpfnWndProc   = WindowProc;   // 메시지 처리 함수 연결
    wc.hInstance     = hInstance;
    wc.lpszClassName = CLASS_NAME;
    RegisterClassW(&wc);             // 1. 윈도우 클래스 등록 (설계도)

    HWND hwnd = CreateWindowExW(     // 2. 실제 윈도우 생성 → HWND 반환
        0, CLASS_NAME, L"내 첫 Win32 프로그램", WS_OVERLAPPEDWINDOW,
        CW_USEDEFAULT, CW_USEDEFAULT, 640, 480,
        nullptr, nullptr, hInstance, nullptr
    );

    if (hwnd == nullptr) return 0; // Orthodox 스타일: 반환값 확인

    ShowWindow(hwnd, nCmdShow);

    MSG msg = {};
    while (GetMessage(&msg, nullptr, 0, 0)) { // 3. 메시지 루프
        TranslateMessage(&msg);
        DispatchMessage(&msg); // WindowProc으로 전달
    }
    return 0;
}
```

---

## 🔁 메시지 루프 (이벤트 기반 프로그래밍의 핵심)

```
[사용자 입력] → [OS가 메시지 큐에 넣음] → [GetMessage로 꺼냄]
      → [DispatchMessage로 WindowProc 호출] → [메시지 종류별 분기 처리]
```

- `GetMessage`: 콜센터 상담원이 전화를 기다리는 것과 동일 (블로킹 대기)
- `DispatchMessage`: 전화 내용에 따라 담당 부서(`WindowProc`의 case문)로 연결
- MFC의 메시지 맵(`ON_WM_PAINT()` 등)도 내부적으로 이 구조를 감싼 것

---

## ⚖️ Orthodox C++ vs Modern C++

| 항목 | Orthodox C++ (Win32 스타일) | Modern C++ |
|------|------------------------------|------------|
| 자원 관리 | `HANDLE`을 직접 `CloseHandle()`로 해제 | RAII 래퍼 클래스로 자동 해제 |
| 콜백 | 함수 포인터 (`WNDPROC`) | `std::function`/람다 (단, Win32 API 자체는 여전히 함수 포인터 요구) |
| 오류 처리 | 반환값 확인 + `GetLastError()` | 예외 기반이지만 Win32 API를 감쌀 때만 적용 가능 |
| 문자열 | `LPCWSTR`, `wchar_t*` (수동 관리) | `std::wstring` (자동 관리, 필요 시 `.c_str()`로 변환) |

### RAII로 HANDLE 감싸기 (실무 권장 패턴)

```cpp
class HandleGuard {
    HANDLE m_handle;
public:
    explicit HandleGuard(HANDLE h) : m_handle(h) {}
    ~HandleGuard() {
        if (m_handle != nullptr && m_handle != INVALID_HANDLE_VALUE)
            CloseHandle(m_handle); // 소멸자에서 자동 해제 (8강 RAII와 직결)
    }
    HANDLE get() const { return m_handle; }
    HandleGuard(const HandleGuard&) = delete;            // 복사 금지
    HandleGuard& operator=(const HandleGuard&) = delete; // 소유권은 하나만
};
```

---

## 🏭 MFC/Win32 실무 연관

- MFC의 `CWnd` 클래스는 내부적으로 `HWND`를 멤버 변수로 감싸서 관리 (Win32 API 위에 얹은 객체지향 래퍼)
- `HANDLE` 누수는 MFC/Win32 코드에서 흔한 버그 → `HandleGuard` 같은 RAII 래퍼로 원천 차단
- 문자열 API는 W(유니코드)/A(ANSI) 두 버전 존재 — 신규 코드는 W 버전 권장

---

## 🌐 크로스플랫폼 주의사항

```cpp
// Windows는 W(유니코드)/A(ANSI) 두 버전 API 제공
CreateWindowExW(...);  // 유니코드(UTF-16) — 권장
CreateWindowExA(...);  // ANSI(멀티바이트) — 레거시
CreateWindowEx(...);   // UNICODE 매크로에 따라 W/A로 자동 치환
```

- `windows.h`는 Windows 전용 헤더 → macOS/Linux에서 컴파일 불가
- `wchar_t` 크기가 플랫폼마다 다름 (Windows 2바이트 UTF-16 / macOS·Linux 4바이트 UTF-32)
- 크로스플랫폼 코드에서는 `wchar_t` 대신 `std::u16string`/UTF-8 기반 `std::string` 표준화 권장

---

## ⚠️ 자주 하는 실수

| 실수 | 올바른 방법 |
|------|------------|
| `HANDLE` 값을 직접 해석/연산하려 함 | 블랙박스로 취급, API를 통해서만 사용 |
| `CloseHandle()` 호출 누락 (특히 예외/조기 return 시) | RAII 래퍼 클래스(`HandleGuard`)로 자동 해제 |
| ANSI(`A`)/유니코드(`W`) 버전 혼용 | 신규 코드는 W 버전으로 통일 |
| `windows.h` 코드를 macOS에서 그대로 컴파일 시도 | Windows 전용 헤더이므로 크로스플랫폼 코드에서 분리 필요 |
| `wchar_t` 크기가 플랫폼마다 같다고 가정 | Windows(2바이트) vs macOS/Linux(4바이트) 차이 인지 |

---

## 🔨 컴파일 명령어

```bash
# Windows (MinGW) - GUI 앱이므로 -mwindows 필요
g++ -std=c++17 win32_test.cpp -o win32_test.exe -mwindows

# Windows (MSVC)
cl /std:c++17 /EHsc win32_test.cpp user32.lib gdi32.lib

# macOS
# 컴파일 불가 (windows.h는 Windows 전용) — 개념 비교 학습용으로만 활용
```

---

## 🔗 연결 노트

- 이전: [[CPP_Exception_1_HandlingDebugging]] — 예외처리 & 디버깅
- 다음: MFC 구조 이해 (CWnd, CDocument, CView)
