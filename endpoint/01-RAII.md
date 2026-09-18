# RAII Resource Acquisition Is Initialization
- 객체의 생명주기에 자원의 생명주기를 묶는다
- 생성자에서 자원을 획득하고, 소멸자에서 반드시 해제한다


### 필요성
- GC가 없기 때문에 직접 자원 해제를 모든 경로에서 일일이 챙겨야 한다
- C 스타일: 자원이 늘어날수록 정리 코드가 조합폭발로 늘어남 (goto cleanup 패턴)
- 특히 예외가 던져지면 수동 정리 코드는 건너뛰어짐 → 자원(핸들) 누수

### C++가 이걸 해결하는 방법 — 스택 언와인딩
- 스코프를 벗어나는 로컬 객체는 정상 return이든 예외로 인한 탈출이든 **반드시 소멸자 호출**이 보장됨
- RAII는 이 보장을 이용: 자원을 객체에 담아두면 객체가 죽을 때 자원도 자동 정리
- C++에 `finally`가 없는 이유이기도 함 (소멸자가 그 역할을 대신함)

```cpp
class FileHandle {
public:
    explicit FileHandle(const char* path) : f_(fopen(path, "r")) {
        if (!f_) throw std::runtime_error("open failed");
    }
    ~FileHandle() { if (f_) fclose(f_); }  // 예외가 나도 반드시 호출됨
    FILE* get() const { return f_; }
private:
    FILE* f_;
};
```

### 다른 언어와 비교
| 언어 | 자원 정리 방식 |
|---|---|
| C | 수동 (goto cleanup) |
| Java/C# | GC는 메모리만. 나머지는 `try-finally`/`using` |
| Node.js | GC + 이벤트 기반. `close()` 수동 호출 필요 |
| C++ | 소멸자 자동 호출 → 별도 문법 불필요 |

---

## Rule of Three
> 클래스가 **자원을 직접 소유**하면, 아래 셋 중 하나라도 직접 정의해야 할 이유가 있으면 **셋 다** 정의해야 한다.
1. 소멸자 (destructor)
2. 복사 생성자 (copy constructor)
3. 복사 대입 연산자 (copy assignment operator)

### 왜 하나만 정의하면 위험한가 — 얕은 복사 문제
- 소멸자만 정의하고 복사를 막지 않으면, 컴파일러가 만든 기본 복사 생성자가 멤버(핸들 값)를 그대로 복사(shallow copy)
- 두 객체가 같은 핸들을 들고 있다가, 각자 소멸자에서 `CloseHandle()`을 호출 → **Double Close**
  - 최악의 경우 그 사이 다른 스레드가 같은 핸들 번호를 재사용했다면 엉뚱한 자원을 닫는 버그로 이어짐

### 자원 소유권 처리 전략
| 전략 | 설명 | 예시 |
|---|---|---|
| 복사 금지 | 소유권이 유일 | `unique_ptr`, `UniqueHandle` |
| 깊은 복사 | 복사마다 새 자원 생성 | 커스텀 버퍼 클래스 |
| 참조 카운팅 | 여러 객체가 공유, 마지막 소유자가 해제 | `shared_ptr` |

### Rule of Zero
- 가장 좋은 설계는 세 함수를 아예 안 쓰는 것
- 클래스가 직접 자원을 들고 있지 않고 `unique_ptr`/`vector` 등 이미 RAII된 멤버만 가지면, 컴파일러 기본 소멸자/복사만으로 충분히 안전

---

## 실습: Windows HANDLE RAII 래퍼

```cpp
class UniqueHandle {
public:
    explicit UniqueHandle(HANDLE h = nullptr) noexcept : h_(h) {}
    ~UniqueHandle() { if (valid()) CloseHandle(h_); }

    // 복사 금지 (이동은 다음 주차에서 추가)
    UniqueHandle(const UniqueHandle&) = delete;
    UniqueHandle& operator=(const UniqueHandle&) = delete;

    bool valid() const noexcept { return h_ != nullptr && h_ != INVALID_HANDLE_VALUE; }
    HANDLE get() const noexcept { return h_; }

private:
    HANDLE h_;
};
```

## 체크포인트 (자가 점검)
1. `CloseHandle`을 소멸자에서 호출하는 클래스에서 복사 생성자를 막지 않으면 어떤 버그가 발생하는가?
2. C++에는 왜 `finally` 없이도 자원 정리가 자동으로 되는가?

