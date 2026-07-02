# CPP STL 2 — map / set

#cpp #stl #map #set #cpp17

---

## 🧠 핵심 개념 한 줄 요약

> **`map` = 사전 (key → value) / `set` = 중복 없는 자동 정렬 집합**

---

## 📦 map

### 선언

```cpp
#include <map>

std::map<std::string, int> scores; // key: 이름, value: 점수
```

### 삽입

```cpp
scores["김철수"] = 90;          // [] 연산자 — 수정/삽입 시 사용
scores.insert({"이영희", 85});  // insert
scores.emplace("박민수", 78);   // emplace — 직접 생성, 권장
```

### 접근 방법 3가지 비교 (중요)

```cpp
// ❌ [] — key 없으면 기본값(0)으로 자동 생성 (의도치 않은 삽입 주의)
scores["없는사람"]; // map에 추가됨

// ✅ at() — key 없으면 예외 발생 (안전한 읽기)
scores.at("김철수");

// ✅ find() — 가장 실무적, 찾은 후 값도 사용할 때
auto it = scores.find("김철수");
if (it != scores.end()) {
    std::cout << it->first << ": " << it->second << std::endl;
}
```

> **실무 핵심 규칙**
> - 읽기 전용 → `find()`
> - 값 수정/삽입 → `[]`
> - 안전한 읽기 → `at()` + 예외 처리

### 존재 여부 확인

```cpp
scores.count("김철수");  // 0 또는 1 반환 — C++17 이하
// scores.contains("김철수"); // C++20 only — C++17에서는 사용 불가
```

### 삭제

```cpp
scores.erase("박민수");       // key로 삭제
scores.erase(scores.begin()); // 이터레이터로 삭제
scores.clear();               // 전체 삭제
```

### 순회

```cpp
// Orthodox 스타일
for (const auto& pair : scores) {
    std::cout << pair.first << ": " << pair.second << std::endl;
}

// ✅ Modern C++17: 구조화 바인딩 (가독성 향상)
for (const auto& [name, score] : scores) {
    std::cout << name << ": " << score << std::endl;
}
```

---

## 📦 set

### 선언 및 기본 사용

```cpp
#include <set>

std::set<std::string> names;

names.insert("김철수");
names.insert("김철수"); // 중복 → 무시됨

std::cout << names.size(); // 1

// 존재 여부
if (names.count("김철수")) { ... }

auto it = names.find("김철수");
if (it != names.end()) { std::cout << *it; }

// 삭제
names.erase("김철수");
```

---

## ⚡ map / set vs unordered 계열

| | `map` / `set` | `unordered_map` / `unordered_set` |
|---|---|---|
| 내부 구조 | 레드-블랙 트리 | 해시 테이블 |
| 정렬 | 자동 정렬 유지 | 정렬 없음 |
| 탐색/삽입/삭제 | O(log n) | 평균 O(1) |
| 플랫폼 순서 보장 | ✅ 동일 | ❌ 플랫폼마다 다름 |

```cpp
// 속도 중요, 정렬 불필요
#include <unordered_map>
std::unordered_map<std::string, int> fastScores;
```

### 선택 기준

```
정렬된 순서 필요         → map / set
속도 중요, 정렬 불필요   → unordered_map / unordered_set
key 범위 순회 필요       → map / set
크로스플랫폼 순서 보장   → map / set
```

---

## ⚖️ Orthodox C++ vs Modern C++

| 항목 | Orthodox | Modern C++17 |
|------|----------|--------------|
| 순회 | `pair.first` / `pair.second` | 구조화 바인딩 `[key, value]` |
| 삽입 | `insert(make_pair(...))` | `insert({...})` / `emplace(...)` |
| 존재 확인 | `count()` | `count()` (C++20: `contains()`) |

---

## 🏭 MFC/Win32 실무 연관

```cpp
// Win32: 윈도우 핸들 → 이름 매핑
std::map<HWND, std::string> windowNames;
windowNames[hwnd] = "메인 윈도우";

// 에러 코드 → 메시지 매핑
std::map<int, std::string> errorMessages;
errorMessages[404] = "찾을 수 없음";
errorMessages[500] = "서버 오류";
```

---

## 🌐 크로스플랫폼 주의사항

```cpp
// unordered_map 순서는 플랫폼/컴파일러마다 다름
// 순서가 중요하면 반드시 map 사용

// ✅ 순서 보장
std::map<std::string, int> ordered;

// ⚠️ 순서 보장 없음
std::unordered_map<std::string, int> unordered;
```

---

## ⚠️ 자주 하는 실수

| 실수 | 올바른 방법 |
|------|------------|
| 읽기에 `[]` 사용 → 의도치 않은 삽입 | `find()` 또는 `at()` 사용 |
| `find()` 후 `end()` 체크 생략 | 반드시 `it != end()` 확인 |
| C++17에서 `contains()` 사용 | `count()` 또는 `find()` 로 대체 |
| 순회 중 `erase` 호출 | `erase` 반환 이터레이터 사용 |
| 정렬 기대하며 `unordered_map` 사용 | 정렬 필요하면 `map` 사용 |

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

- 이전: [[CPP_STL_1_Vector]] — vector
- 다음: [[CPP_STL_3_Algorithm]] — 나머지 컨테이너 & 알고리즘
