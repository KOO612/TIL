# CPP Exception 1 — 예외처리 & 디버깅

#cpp #exception #debugging #RAII #cpp17

---

## 🧠 핵심 개념 한 줄 요약

> **예외처리 = 문제 발생 시 프로그램이 죽지 않고 "안내하고 정상 흐름 복귀"하는 안전장치**

---

## 📦 기본 문법: try / throw / catch

```cpp
#include <stdexcept>

double divide(double a, double b) {
    if (b == 0.0)
        throw std::runtime_error("0으로 나눌 수 없습니다."); // 예외 던지기

    return a / b;
}

try {
    double result = divide(10.0, 0.0);
}
catch (const std::runtime_error& e) {
    std::cerr << "예외 발생: " << e.what() << std::endl; // 예외 잡기
}
```

---

## 📋 표준 예외 클래스 계층

```
std::exception
 ├── std::logic_error         // 사전에 방지 가능한 논리 오류
 │    ├── std::invalid_argument
 │    ├── std::out_of_range    // vector.at() 범위 초과
 │    └── std::length_error
 └── std::runtime_error        // 실행 중에만 알 수 있는 오류
      ├── std::overflow_error
      └── std::underflow_error
```

```cpp
std::vector<int> v = {1, 2, 3};

try {
    v.at(10); // 범위 초과 → std::out_of_range 던짐
}
catch (const std::out_of_range& e) {      // 파생 클래스 먼저
    std::cerr << "범위 오류: " << e.what() << std::endl;
}
catch (const std::exception& e) {         // 기반 클래스 나중
    std::cerr << "기타 예외: " << e.what() << std::endl;
}
```

> **실무 핵심 규칙**
> - 범위 초과 위험 있는 접근 → `.at()` (예외로 명확히 감지)
> - 성능이 중요하고 범위가 검증된 경우 → `[]` (범위 검사 없음, 대신 UB 위험)

---

## 🛡️ RAII와 예외 안전성 (⭐ 매우 중요)

```cpp
class Resource {
public:
    Resource()  { std::cout << "자원 획득\n"; }
    ~Resource() { std::cout << "자원 해제\n"; } // 예외 발생해도 반드시 호출
};

void riskyFunction() {
    Resource res; // 스택에 생성
    throw std::runtime_error("에러 발생!");
    // res의 소멸자는 스택 언와인딩 과정에서 반드시 호출됨
}
```

- **RAII (Resource Acquisition Is Initialization)**: 자원 획득을 객체 생성과 묶고 소멸자에서 자동 해제
- 예외가 발생해도 스택에 있는 지역 객체의 소멸자는 반드시 호출됨 (스택 언와인딩)
- `std::unique_ptr`, `std::shared_ptr` 등이 대표적 RAII 도구

---

## 🔍 디버깅 기초 도구

```cpp
#include <cassert>

void setAge(int age) {
    // 디버그 빌드에서만 검사, Release 빌드(NDEBUG)에서는 제거됨
    assert(age >= 0 && "나이는 음수일 수 없습니다.");
}
```

| 상황 | Windows (Visual Studio) | macOS |
|------|--------------------------|-------|
| 중단점 디버깅 | F9 브레이크포인트, F5 디버그 실행 | Xcode / `lldb` |
| 콘솔 디버거 | `windbg`, VS 내장 디버거 | `lldb`, `gdb` |
| 어설션 | `assert(condition)` | 동일 |
| 로그 출력 | `OutputDebugString()` (MFC/Win32 전용) | `std::cerr`, `NSLog` |

---

## ⚖️ Orthodox C++ vs Modern C++

| 항목 | Orthodox C++ | Modern C++11~17 |
|------|---------------|------------------|
| 오류 전달 | 반환값, `errno`, `GetLastError()` | `throw`, `std::optional` (C++23: `std::expected`) |
| 자원 정리 | `catch` 안에서 수동 `delete`/`CloseHandle()` | RAII로 자동 정리 |
| 실무 예시 | MFC/Win32 API (`INVALID_HANDLE_VALUE` + `GetLastError()`) | STL, 표준 라이브러리 |
| 성능 | 오버헤드 없음 | 예외 미발생 시 거의 zero-cost, 발생 시 상대적으로 느림 |

---

## 🏭 MFC/Win32 실무 연관

```cpp
// Win32 스타일 (Orthodox) — 반환값 + GetLastError()
HANDLE hFile = CreateFile(L"test.txt", GENERIC_READ, 0, NULL,
                           OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
if (hFile == INVALID_HANDLE_VALUE) {
    DWORD err = GetLastError(); // 예외 대신 에러 코드로 확인
}
```

- MFC/Win32 코드를 다룰 때는 예외보다 **반환값 + `GetLastError()`** 패턴이 훨씬 흔함
- STL이나 최신 C++ 라이브러리는 예외 기반 설계가 기본
- 두 스타일이 섞인 코드베이스를 다루는 게 실무에서 매우 흔하므로 둘 다 능숙해야 함

---

## 🌐 크로스플랫폼 주의사항

```cpp
// cl.exe에서 /EHsc 누락 시 예외처리가 정상 작동하지 않음
// Windows(MSVC)와 macOS(clang++/g++)는 컴파일 옵션이 다르므로
// 빌드 스크립트/CMake에서 플랫폼별 분기 필요

#ifdef _WIN32
    // Windows 전용 예외 처리 (구조적 예외 처리 SEH 등)
#else
    // macOS/Linux 표준 C++ 예외만 사용
#endif
```

---

## ⚠️ 자주 하는 실수

| 실수 | 올바른 방법 |
|------|------------|
| `[]`로 접근 후 범위 오류로 디버깅 시간 낭비 | `.at()` 사용 시 명확한 예외 메시지로 즉시 원인 파악 |
| cl.exe에서 `/EHsc` 옵션 누락 | 반드시 `/EHsc` 추가 (빠뜨리면 예외 정상 작동 안 함) |
| 예외 발생 시 자원 해제를 수동으로 처리하려다 누락 | RAII(스마트 포인터, 래퍼 클래스)로 원천 차단 |
| `catch` 순서를 기반 클래스 먼저 배치 | 파생 클래스 예외를 기반 클래스 예외보다 먼저 배치 |
| `assert`를 런타임 입력 검증용으로 착각 | Release 빌드에서 제거되므로 실제 오류 처리는 `throw` 사용 |

---

## 🔨 컴파일 명령어

```bash
# Windows (MinGW)
g++ -std=c++17 -o main main.cpp

# Windows (MSVC) — /EHsc 필수
cl /std:c++17 /EHsc main.cpp

# macOS
clang++ -std=c++17 -o main main.cpp
```

---

## 🔗 연결 노트

- 이전: [[CPP_STL_3_Algorithm_Containers]] — 알고리즘 & 나머지 컨테이너
- 다음: 2단계 — Win32 API 기초
