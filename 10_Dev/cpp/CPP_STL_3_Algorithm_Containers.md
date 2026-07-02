# CPP STL 3 — 알고리즘 & 나머지 컨테이너

#cpp #stl #algorithm #deque #stack #queue #lambda #cpp17

---

## 🧠 핵심 개념 한 줄 요약

> **STL 알고리즘 = 검증된 도구 세트 — 직접 루프 짜는 대신 표준 함수 활용**

---

## 📋 `<algorithm>` 핵심 함수

```cpp
#include <algorithm>

// 정렬
std::sort(v.begin(), v.end());                       // 오름차순
std::sort(v.begin(), v.end(), std::greater<int>());  // 내림차순
std::stable_sort(v.begin(), v.end());                // 안정 정렬 (순서 보장)

// 탐색
std::find(v.begin(), v.end(), 3);                    // O(n), 정렬 불필요
std::binary_search(v.begin(), v.end(), 3);           // O(log n), 반드시 정렬 후 사용

// 최솟값 / 최댓값
auto minIt = std::min_element(v.begin(), v.end());   // 이터레이터 반환 → * 로 역참조
auto maxIt = std::max_element(v.begin(), v.end());

// 개수 세기
std::count(v.begin(), v.end(), 2);                   // 값이 2인 원소 수

// 뒤집기
std::reverse(v.begin(), v.end());

// 변환
std::transform(v.begin(), v.end(), out.begin(), [](int n){ return n * 2; });

// 순회
std::for_each(v.begin(), v.end(), [](int n){ std::cout << n; });
```

---

## 🗑️ Erase-Remove 패턴 (조건부 삭제)

```cpp
// ✅ 조건에 맞는 원소 삭제 — 반드시 두 단계로
v.erase(
    std::remove_if(v.begin(), v.end(), [](int n){
        return n % 2 == 0; // 짝수 제거
    }),
    v.end()
);

// ⚠️ remove_if 단독 사용 금지 — 실제 삭제 안 됨
// remove_if 후: [홀수들, ?, ?, ?]  ← 뒤는 쓰레기값
// erase 후:     [홀수들]           ← 실제 삭제 완료
```

---

## 📋 `<numeric>` 핵심 함수

```cpp
#include <numeric>

// 합계
std::accumulate(v.begin(), v.end(), 0);

// 곱
std::accumulate(v.begin(), v.end(), 1, std::multiplies<int>());

// 연속값 채우기
std::vector<int> seq(5);
std::iota(seq.begin(), seq.end(), 1); // [1, 2, 3, 4, 5]
```

---

## 🎯 람다(Lambda) 캡처 정리

```cpp
// 람다 = 이름 없는 함수 — [](){}
int sum = 0;
std::for_each(v.begin(), v.end(), [&sum](int n){
    sum += n;
});
```

| 캡처 | 의미 |
|------|------|
| `[]` | 캡처 없음 |
| `[&]` | 모든 외부 변수 참조로 캡처 |
| `[=]` | 모든 외부 변수 복사로 캡처 |
| `[&sum]` | sum만 참조로 캡처 |
| `[sum]` | sum만 복사로 캡처 |
| `[&, sum]` | sum은 복사, 나머지는 참조 |

---

## 📦 나머지 컨테이너

### deque — 양방향 큐

```cpp
#include <deque>

std::deque<int> dq;
dq.push_back(1);   // 뒤에 추가
dq.push_front(0);  // 앞에 추가 ← vector에는 없는 기능 O(1)
dq.pop_front();    // 앞에서 제거
dq.pop_back();     // 뒤에서 제거
```

### list — 이중 연결 리스트

```cpp
#include <list>

std::list<int> lst = {1, 2, 3};
auto it = lst.begin();
std::advance(it, 2);  // 이터레이터 이동 (인덱스 접근 불가)
lst.insert(it, 99);   // 중간 삽입 O(1)
lst.erase(it);        // 중간 삭제 O(1)
// ❌ lst[2] 불가 — 순차 접근만 가능
```

### stack — LIFO

```cpp
#include <stack>

std::stack<int> stk;
stk.push(1); stk.push(2); stk.push(3);
stk.top();   // 3 — 맨 위 확인 (제거 안 함)
stk.pop();   // 3 제거
stk.empty(); // 비었는지 확인 — top() 전에 반드시 체크
```

### queue — FIFO

```cpp
#include <queue>

std::queue<int> q;
q.push(1); q.push(2); q.push(3);
q.front(); // 1 — 맨 앞 확인 (제거 안 함)
q.pop();   // 1 제거
q.empty(); // 비었는지 확인 — front() 전에 반드시 체크
```

---

## 🗂️ 컨테이너 선택 가이드

| 상황 | 컨테이너 |
|------|----------|
| 임의 접근(인덱스)이 자주 필요 | `vector` |
| 앞/뒤 삽입/삭제가 자주 필요 | `deque` |
| 중간 삽입/삭제가 자주 필요 | `list` |
| key-value, 정렬 필요 | `map` |
| key-value, 속도 중요 | `unordered_map` |
| 중복 없는 집합, 정렬 필요 | `set` |
| 중복 없는 집합, 속도 중요 | `unordered_set` |
| LIFO (undo, 재귀 대체) | `stack` |
| FIFO (작업 대기열, BFS) | `queue` |

---

## ⚖️ Orthodox C++ vs Modern C++

| 항목 | Orthodox | Modern C++11~ |
|------|----------|---------------|
| 조건 삭제 | 직접 루프 + erase | Erase-Remove 패턴 |
| 순회 | 직접 for 루프 | `for_each` + 람다 |
| 변환 | 직접 루프 | `transform` + 람다 |
| 비교 함수 | 함수 포인터 / functor | 람다 `[](){}` |
| 안정 정렬 | `stable_sort` | 동일 |

---

## 🏭 MFC/Win32 실무 연관

```cpp
// MFC 레거시 → STL 마이그레이션
// CList → std::list
// CArray → std::vector
// CMap → std::map

// Undo/Redo 시스템 → stack 활용
std::stack<std::string> undoHistory;
undoHistory.push("텍스트 입력");
// Ctrl+Z → top() 실행 후 pop()

// Win32 메시지 대기열 → queue 개념과 동일
```

---

## 🌐 크로스플랫폼 주의사항

```cpp
// sort는 동일한 값의 순서를 보장하지 않음 (플랫폼마다 다를 수 있음)
std::sort(v.begin(), v.end());         // ❌ 순서 보장 없음
std::stable_sort(v.begin(), v.end());  // ✅ 순서 보장

// STL 알고리즘 자체는 플랫폼 무관하게 동작
```

---

## ⚠️ 자주 하는 실수

| 실수 | 올바른 방법 |
|------|------------|
| `remove_if` 단독 사용 | 반드시 `erase` 와 함께 사용 |
| 정렬 없이 `binary_search` | 먼저 `sort` 후 사용 |
| 람다에서 외부 변수 캡처 누락 | `[&변수명]` 또는 `[&]` 명시 |
| `list` 에 인덱스 접근 | `list` 는 순차 접근만 가능 |
| `stack` / `queue` 비어있을 때 `top()` / `front()` | `empty()` 체크 먼저 |
| `min/max_element` 결과 역참조 없이 사용 | `*minIt` 으로 역참조 |

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

- 이전: [[CPP_STL_2_Map_Set]] — map / set
- 다음: [[CPP_ExceptionHandling_Debugging]] — 예외처리 & 디버깅
