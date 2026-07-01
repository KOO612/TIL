# CPP STL 1 — vector

#cpp #stl #vector #cpp17

---

## 🧠 핵심 개념 한 줄 요약

> **크기가 동적으로 변하는 배열 — 가장 자주 쓰는 STL 컨테이너**

```
일반 배열: [고정 크기] → 초과하면 끝
vector:   [동적 크기] → 자동으로 늘어남
```

---

## 📦 선언 방법

```cpp
#include <vector>

std::vector<int> v1;                    // 빈 벡터
std::vector<int> v2(5);                 // 크기 5, 전부 0
std::vector<int> v3(5, 42);             // 크기 5, 전부 42
std::vector<int> v4 = {1, 2, 3, 4, 5}; // 초기값 직접 지정 (C++11)
```

---

## 📋 주요 멤버 함수 총정리

### 추가 / 삭제

```cpp
v.push_back(10);             // 끝에 추가 (복사)
v.emplace_back(10);          // 끝에 직접 생성 (복사 없음 — 권장)
v.pop_back();                // 마지막 제거
v.insert(v.begin(), 0);      // 맨 앞에 삽입
v.insert(v.begin() + 2, 99); // 인덱스 2에 삽입
v.erase(v.begin());          // 첫 번째 제거
v.erase(v.begin(), v.begin() + 2); // 범위 삭제
v.clear();                   // 전체 삭제 (size=0, capacity 유지)
```

### 접근

```cpp
v[0];       // 인덱스 접근 — 범위 초과 시 미정의 동작(UB)
v.at(0);    // 안전한 접근 — 범위 초과 시 예외 발생 (디버깅 시 권장)
v.front();  // 첫 번째 요소
v.back();   // 마지막 요소
```

### 크기 / 용량

```cpp
v.size();        // 현재 원소 수
v.capacity();    // 할당된 메모리 크기 (항상 size 이상)
v.empty();       // 비었는지 확인
v.reserve(100);  // 미리 공간 확보 (재할당 방지 — 성능 최적화)
v.resize(5);     // 크기 조정 (줄이면 잘리고, 늘리면 0으로 채움)
v.shrink_to_fit(); // capacity를 size에 맞게 줄임
```

---

## 🔁 순회 방법

```cpp
// 방법 1: 인덱스 (Orthodox)
for (int i = 0; i < (int)v.size(); ++i) { ... }

// 방법 2: 범위 기반 for (Modern C++11 — 가장 권장)
for (const auto& elem : v) { ... }

// 방법 3: 이터레이터
for (auto it = v.begin(); it != v.end(); ++it) {
    std::cout << *it; // * 로 역참조
}
```

---

## ⚖️ Orthodox C++ vs Modern C++

| 항목 | Orthodox | Modern C++11~ |
|------|----------|---------------|
| 초기화 | `push_back` 반복 | `{1, 2, 3}` 초기화 리스트 |
| 추가 | `push_back` | `emplace_back` (복사 없이 효율적) |
| 순회 | 인덱스 for | 범위 기반 for |
| 타입 | `std::vector<int>::iterator` | `auto` |

---

## 📐 size vs capacity 내부 동작

```
push_back 시 capacity 초과 → 새 공간 2배 할당 → 전체 복사

size     = 실제 들어있는 원소 수
capacity = 현재 할당된 메모리가 담을 수 있는 최대 수
capacity >= size 항상 성립
```

```cpp
// ✅ 크기를 미리 알면 reserve로 재할당 방지
std::vector<int> v;
v.reserve(1000); // 재할당 없이 1000개 추가 가능
```

---

## 🔑 push_back vs emplace_back

```cpp
// push_back: 객체를 만들고 → 복사/이동해서 넣음
students.push_back(Student("kim", 90));

// emplace_back: vector 내부에서 직접 생성 → 복사 없음 → 더 빠름
students.emplace_back("kim", 90); // Modern C++ 권장
```

---

## 🗂️ 2차원 vector

```cpp
int rows = 3, cols = 4;
std::vector<std::vector<int>> matrix(rows, std::vector<int>(cols, 0));

matrix[0][0] = 1;
matrix[1][2] = 5;
```

---

## 🏭 MFC/Win32 실무 연관

```cpp
// MFC CArray 스타일 (Orthodox)
// CArray<int, int> arr;
// arr.Add(10);

// Modern C++ 스타일
std::vector<int> arr;
arr.emplace_back(10);

// Win32 핸들 목록 관리
// std::vector<HWND> windowHandles;
// windowHandles.push_back(hwnd);
```

---

## 🌐 크로스플랫폼 주의사항

```cpp
// size() 반환 타입은 size_t (플랫폼마다 크기 다름)
// ❌ int와 비교 시 경고 가능
for (int i = 0; i < v.size(); ++i) { ... }

// ✅ 안전한 방법
for (size_t i = 0; i < v.size(); ++i) { ... }

// ✅ 더 간결한 방법
for (const auto& e : v) { ... }
```

---

## ⚠️ 자주 하는 실수

| 실수 | 올바른 방법 |
|------|------------|
| `v[n]` 범위 초과 접근 | `v.at(n)` 으로 예외 처리 |
| 빈 벡터에서 `max_element` 역참조 | `empty()` 체크 먼저 |
| 대량 추가 전 `reserve` 생략 | 미리 `reserve(n)` 호출 |
| `size()` 를 `int` 와 비교 | `size_t` 사용 또는 범위 for |
| 순회 중 `erase` 호출 | `erase` 반환값(이터레이터)을 다음으로 사용 |
| `erase` 전 범위 체크 생략 | `index >= 0 && index < size()` 확인 |
| 공백 포함 문자열 `cin >>` 로 읽기 | `std::getline(std::cin >> std::ws, str)` |

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

- 이전: [[CPP_OOP_4_OperatorOverloading]] — 연산자 오버로딩
- 다음: [[CPP_STL_2_Map_Set]] — map / set
