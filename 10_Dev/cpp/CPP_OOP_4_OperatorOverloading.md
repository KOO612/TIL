# CPP OOP 4 — 연산자 오버로딩 (Operator Overloading)

#cpp #oop #operator-overloading #cpp17

---

## 🧠 핵심 개념 한 줄 요약

> **기존 연산자(`+`, `-`, `<<` 등)에게 내 클래스만의 동작을 직접 가르치는 것**

---

## 📐 기본 문법

```cpp
반환타입 operator연산자(매개변수) { ... }
```

---

## 📋 오버로딩 방식 두 가지

| 방식 | 언제 사용 | 예시 |
|------|-----------|------|
| **멤버 함수** | 왼쪽 피연산자가 내 클래스일 때 | `+`, `-`, `*`, `==`, `[]`, `++` |
| **전역 함수 + friend** | 왼쪽 피연산자가 내 클래스가 아닐 때 | `<<`, `>>`, `숫자 * 객체` |

---

## ✅ 패턴별 구현 정리

### 1. 산술 연산자 `+`, `-`

```cpp
// ✅ 권장 패턴: += 먼저 구현 → + 는 재활용
Vector2D& operator+=(const Vector2D& other) {
    x += other.x;
    y += other.y;
    return *this;
}

Vector2D operator+(const Vector2D& other) const {
    Vector2D result = *this;
    result += other;
    return result;
}
```

### 2. 출력 연산자 `<<` — 반드시 전역 함수 + friend

```cpp
// 클래스 내부에 선언
friend std::ostream& operator<<(std::ostream& os, const Vector2D& v);

// 클래스 외부에 정의
std::ostream& operator<<(std::ostream& os, const Vector2D& v) {
    os << "(" << v.x << ", " << v.y << ")";
    return os; // 체이닝(cout << a << b)을 위해 반드시 반환
}
```

### 3. 비교 연산자 `==`, `!=`

```cpp
bool operator==(const Vector2D& other) const {
    const double EPSILON = 1e-9; // 부동소수점 안전 비교
    return (std::abs(x - other.x) < EPSILON) &&
           (std::abs(y - other.y) < EPSILON);
}

bool operator!=(const Vector2D& other) const {
    return !(*this == other); // == 재활용
}
```

### 4. 첨자 연산자 `[]`

```cpp
// 수정 가능 버전
double& operator[](int index) {
    if (index < 0 || index >= 3)
        throw std::out_of_range("인덱스 범위 초과!");
    return data[index];
}

// 읽기 전용 버전 (const 객체용) — 두 버전 모두 필요
const double& operator[](int index) const {
    if (index < 0 || index >= 3)
        throw std::out_of_range("인덱스 범위 초과!");
    return data[index];
}
```

### 5. 전위 / 후위 증가 `++`

```cpp
// 전위 ++v : 증가 후 자신 반환
Vector2D& operator++() {
    ++x; ++y;
    return *this;
}

// 후위 v++ : int 더미 매개변수로 구별, 이전 값 반환
Vector2D operator++(int) {
    Vector2D temp = *this;
    ++(*this);
    return temp;
}
```

---

## ⚖️ Orthodox C++ vs Modern C++

| 항목 | Orthodox (C++03 이전) | Modern (C++11~17) |
|------|----------------------|-------------------|
| 비교 연산자 | `==`, `!=`, `<`, `>` 전부 직접 구현 | C++20: `<=>` 하나로 전부 자동 |
| 복사 대입 | 직접 구현 필수 | `= default` 자동 생성 |
| 출력 | `print()` 멤버 함수 | `operator<<` 전역 함수 |
| 이동 대입 | 없음 | `operator=(T&&)` 추가 가능 |

---

## 🚫 오버로딩 불가 연산자

| 연산자 | 이유 |
|--------|------|
| `::` | 스코프 해석자 |
| `.` | 멤버 접근자 |
| `.*` | 멤버 포인터 접근 |
| `?:` | 삼항 연산자 |
| `sizeof` | 컴파일 타임 연산 |

---

## 🏭 MFC/Win32 실무 연관

- `CPoint`, `CSize`, `CRect` 내부에 `operator+` 등이 이미 구현됨
- `CString`의 `+` 연결도 동일 원리
- 커스텀 MFC 데이터 클래스 설계 시 직접 적용 가능

---

## 🌐 크로스플랫폼 주의사항

```cpp
// ❌ 위험: 부동소수점 직접 비교 (플랫폼마다 결과 다를 수 있음)
return x == other.x;

// ✅ 안전: EPSILON 허용 오차 사용
return std::abs(x - other.x) < 1e-9;
```

---

## 🔑 핵심 규칙 5가지

1. `+=`, `-=` 먼저 구현 → `+`, `-` 는 이를 재활용 (코드 중복 제거)
2. `<<`, `>>` 는 반드시 전역 함수 + `friend`
3. `[]` 는 `const` / 비`const` 두 버전 모두 구현
4. 후위 `++`/`--` 는 `int` 더미 매개변수로 전위와 구별
5. 부동소수점 `==` 비교는 `EPSILON` 사용

---

## ⚠️ 자주 하는 실수

| 실수 | 올바른 방법 |
|------|------------|
| `<<` 를 멤버 함수로 구현 시도 | 전역 함수 + `friend` 로 구현 |
| `[]` const 버전 누락 | const / 비const 두 버전 모두 작성 |
| `+` 먼저 구현 후 `+=` 따로 구현 | `+=` 먼저 → `+` 가 재활용 |
| 후위 `++` 에서 이전 값 반환 누락 | `temp` 에 저장 후 반환 |
| 부동소수점 `==` 직접 비교 | EPSILON 허용 오차 사용 |
| `<<` 에서 `os` 반환 누락 | `return os;` 필수 (체이닝) |

---

## 🔨 컴파일 명령어

```bash
# Windows (MinGW)
g++ -std=c++17 -o main main.cpp

# Windows (MSVC)
cl /std:c++17 /EHsc main.cpp

# macOS
clang++ -std=c++17 -o main main.cpp
```

---

## 🔗 연결 노트

- 이전: [[CPP_OOP_3_Polymorphism]] — 다형성 / 추상 클래스
- 다음: [[CPP_STL_1_Vector]] — STL vector
