# C++ 객체지향 3강 — 다형성 심화 / 추상 클래스

#cpp #oop #polymorphism #abstract-class #virtual

---

## 📌 핵심 개념 한 줄 요약

> 추상 클래스는 **"반드시 이걸 구현해라"** 는 계약서
> 파생 클래스는 계약 항목을 **반드시** 구현해야 함

---

## 1. 순수 가상 함수 (Pure Virtual Function)

```cpp
class Shape {
public:
    virtual ~Shape() {}

    // 순수 가상 함수 — = 0 이 핵심
    virtual double calcArea()      = 0;
    virtual double calcPerimeter() = 0;
    virtual void   draw()          = 0;

    // 일반 함수는 구현 가능
    void printColor() {
        std::cout << "색상 출력" << std::endl;
    }
};
```

### 규칙
- `= 0` 을 붙이면 순수 가상 함수
- 순수 가상 함수가 하나라도 있으면 → **추상 클래스**
- 추상 클래스는 **객체 생성 불가**

```cpp
Shape shape;  // ❌ 컴파일 에러 — 추상 클래스는 객체 생성 불가
```

---

## 2. 추상 클래스 구현

```cpp
class Circle : public Shape {
private:
    double m_radius;

public:
    Circle(std::string color, double radius)
        : Shape(color), m_radius(radius) {}

    // 순수 가상 함수는 반드시 override
    double calcArea() override {
        return 3.14159 * m_radius * m_radius;
    }

    double calcPerimeter() override {
        return 2 * 3.14159 * m_radius;
    }

    void draw() override {
        std::cout << "원 그리기 (반지름: " << m_radius << ")" << std::endl;
    }
};
```

---

## 3. virtual vs 순수 가상 함수 비교

| 종류 | 문법 | 재정의 | 기본 동작 | 추상 클래스 여부 |
|------|------|--------|-----------|----------------|
| 일반 함수 | `void func()` | 선택 | 있음 | ❌ |
| 가상 함수 | `virtual void func()` | 선택 | 있음 | ❌ |
| 순수 가상 함수 | `virtual void func() = 0` | **필수** | 없음 | ✅ |

---

## 4. 인터페이스 패턴

> 인터페이스 = **순수 가상 함수만** 있는 추상 클래스
> 멤버변수, 생성자 없음
> `I` 접두사 관례 사용

```cpp
// ✅ 올바른 인터페이스
class IDrawable {
public:
    virtual ~IDrawable() {}  // 가상 소멸자 필수
    virtual void draw()   = 0;
    virtual void resize() = 0;
    // 멤버변수 없음, 생성자 없음
};

// ❌ 잘못된 인터페이스
class IAnimal {
protected:
    std::string m_type;  // 멤버변수 → 인터페이스에 부적절
public:
    IAnimal(std::string type) : m_type(type) {}  // 생성자 → 부적절
};
```

### 멤버변수는 파생 클래스에서 관리

```cpp
class Dog : public IAnimal {
private:
    std::string m_type;  // ✅ 파생 클래스에서 관리
public:
    Dog(std::string type) : m_type(type) {}
};
```

---

## 5. 다중 인터페이스 구현

```cpp
class IDrawable {
public:
    virtual ~IDrawable() {}
    virtual void draw() = 0;
};

class ISerializable {
public:
    virtual ~ISerializable() {}
    virtual void serialize() = 0;
};

// C++는 다중 상속 가능
class Document : public IDrawable, public ISerializable {
public:
    void draw()      override { std::cout << "문서 그리기" << std::endl; }
    void serialize() override { std::cout << "문서 저장"   << std::endl; }
};
```

---

## 6. 다형성 활용 패턴

```cpp
// 부모 포인터로 자식 객체 관리
std::vector<std::unique_ptr<Shape>> shapes;
shapes.push_back(std::make_unique<Circle>("빨강", 5.0));
shapes.push_back(std::make_unique<Rectangle>("파랑", 4.0, 6.0));

// 같은 함수 호출 → 각자 다른 동작
for (const auto& shape : shapes) {
    shape->draw();
    std::cout << "넓이: " << shape->calcArea() << std::endl;
}
```

---

## 7. 자식 전용 함수 호출 방법

부모 포인터로 관리할 때 자식에만 있는 함수 호출 방법

```cpp
// 방법 1 — Shape에 virtual로 추가 (권장)
class Shape {
    virtual void printInfo() {}  // 기본 빈 함수
};

// 방법 2 — dynamic_cast로 타입 확인
Circle* circle = dynamic_cast<Circle*>(shape.get());
if (circle != nullptr) {
    circle->printInfo();  // Circle 전용 함수 호출
}

// 방법 3 — 벡터 분리
std::vector<std::unique_ptr<Circle>> circles;
```

| 방법 | 사용 시점 |
|------|----------|
| virtual 추가 | 모든 자식에게 의미 있는 함수 |
| dynamic_cast | 특정 자식에게만 필요한 경우 |
| 벡터 분리 | 타입별로 완전히 다른 처리 |

---

## 8. 가상 소멸자 — 반드시 선언

```cpp
class Shape {
public:
    virtual ~Shape() {}  // ✅ 필수!
};

// 없으면?
Shape* ptr = new Circle();
delete ptr;  // ❌ Circle 소멸자 미호출 → 메모리 누수
```

---

## 9. MFC 실무 연관성

```cpp
// MFC CView — OnDraw가 순수 가상 함수
class CView : public CWnd {
public:
    virtual void OnDraw(CDC* pDC) = 0;  // 반드시 구현해야 함
};

// 우리가 만드는 뷰
class CMyView : public CView {
public:
    void OnDraw(CDC* pDC) override {
        pDC->TextOut(100, 100, "Hello MFC");
    }
};
```

---

## 10. 크로스플랫폼 활용 (ImGui 전환 시)

```cpp
// 인터페이스로 플랫폼별 구현 분리
class IRenderer {
public:
    virtual ~IRenderer() {}
    virtual void init()     = 0;
    virtual void render()   = 0;
    virtual void shutdown() = 0;
};

#ifdef _WIN32
class Win32Renderer : public IRenderer { /* Win32 구현 */ };
#else
class MacRenderer   : public IRenderer { /* Mac 구현 */ };
#endif
```

---

## ⚠️ 자주 하는 실수

```cpp
// ❌ override에 괄호 붙이기
void speak() override() { }   // 컴파일 에러

// ✅ override는 키워드, 괄호 없음
void speak() override { }

// ❌ 순수 가상 함수 구현 누락
class Dog : public IAnimal {
    // speak() 구현 안 함 → 컴파일 에러
};

// ❌ 추상 클래스 직접 생성
IAnimal animal;  // 컴파일 에러
```

---

## 🔗 연관 개념

- [[CPP_OOP_1_Class]] — 클래스 기초
- [[CPP_OOP_2_Inheritance]] — 상속
- [[CPP_OOP_4_OperatorOverloading]] — 연산자 오버로딩
- [[CPP_STL_Vector]] — vector (다음 단원)

---

## 컴파일 명령어

```bash
# Windows (MinGW)
g++ -std=c++17 -o main main.cpp

# macOS
clang++ -std=c++17 -o main main.cpp
./main
```
