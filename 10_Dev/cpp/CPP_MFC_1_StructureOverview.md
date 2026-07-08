# CPP MFC 1 — 구조 이해 (CWnd, CDocument, CView)

#cpp #mfc #windows #cwnd #cdocument #cview #document-view

---

## 🧠 핵심 개념 한 줄 요약

> **MFC = Win32 API를 객체지향으로 감싼 프레임워크, 핵심 철학은 "데이터(Document)와 화면(View)의 분리"**

---

## 📦 MFC 클래스 계층 구조

```
CObject                          // MFC 모든 클래스의 최상위 (직렬화, RTTI 지원)
 └── CCmdTarget                  // 메시지(명령) 처리 가능한 객체
      ├── CWnd                   // 모든 윈도우(창)의 기반 클래스
      │    ├── CFrameWnd         // 메인 프레임 창 (제목표시줄, 메뉴 포함)
      │    │    └── CMDIFrameWnd // 다중 문서 인터페이스 프레임
      │    └── CView             // 문서를 화면에 표시하는 뷰
      │         ├── CScrollView
      │         ├── CFormView
      │         └── CEditView
      └── CDocument              // 데이터를 관리하는 문서 클래스
```

| 클래스 | 역할 | Win32 대응 개념 |
|------|------|-----------------|
| `CWnd` | 모든 윈도우 UI의 기반. `HWND`를 멤버로 감싸 관리 | `HWND` + `WindowProc` |
| `CDocument` | 데이터 저장/불러오기, 직렬화 담당 | (Win32에는 대응 개념 없음 — MFC 고유 추상화) |
| `CView` | Document의 데이터를 화면에 그리는 역할 | `WM_PAINT` 처리 로직 |
| `CFrameWnd` | 메인 창(제목표시줄, 메뉴, 툴바 포함) | `CreateWindowEx` + 메뉴/툴바 API |

---

## 🪟 CWnd — 모든 윈도우의 기반 클래스

```cpp
class CWnd : public CCmdTarget {
protected:
    HWND m_hWnd; // Win32 HWND를 캡슐화

public:
    HWND GetSafeHwnd() const { return m_hWnd; }
    virtual void OnPaint();        // WM_PAINT 메시지에 대응
    void Invalidate();             // Win32 InvalidateRect 감쌈 — 다시 그리기 요청
    BOOL ShowWindow(int nCmdShow); // Win32 ShowWindow API 감쌈
};
```

> **핵심 연결**: `HWND`, `WM_PAINT`, `ShowWindow`는 Win32 API 강의에서 배운 것과 동일한 개념. **MFC = Win32 API를 클래스로 감싼 객체지향 래퍼**

---

## 📄 CDocument — 데이터 관리

```cpp
class CMyDocument : public CDocument {
protected:
    CMyDocument() {} // MFC 프레임워크가 생성 → protected
    DECLARE_DYNCREATE(CMyDocument)

private:
    CString m_strContent; // 실제 데이터

public:
    virtual void Serialize(CArchive& ar) override {
        if (ar.IsStoring())  ar << m_strContent; // 저장
        else                 ar >> m_strContent; // 불러오기
    }

    void SetContent(const CString& str) {
        m_strContent = str;
        SetModifiedFlag(); // "수정됨" 표시 → 저장 안내창 트리거
    }
};
```

**핵심 역할**
- 실제 데이터 저장 (메모리 상의 상태)
- `Serialize()`를 통한 파일 입출력 (직렬화)
- `SetModifiedFlag()` / `IsModified()`로 "저장 여부" 추적

---

## 🖼️ CView — 화면 표시

```cpp
class CMyView : public CView {
public:
    CMyDocument* GetDocument() const {
        return (CMyDocument*)m_pDocument; // CView가 내부에 CDocument* 보유
    }

    virtual void OnDraw(CDC* pDC) override {
        CMyDocument* pDoc = GetDocument();
        if (!pDoc) return;
        pDC->TextOut(10, 10, pDoc->GetContent()); // Document 데이터를 화면에 출력
    }
};
```

**핵심 역할**
- `OnDraw(CDC* pDC)`: Document 데이터를 실제 화면에 그림 (Win32 `WM_PAINT` + `TextOut` 흐름과 동일)
- `GetDocument()`: View는 항상 자신과 연결된 Document에 접근 — **View는 Document 없이 존재 불가**

---

## 🔗 Document-View 아키텍처

```
┌─────────────┐   데이터 요청/알림   ┌─────────────┐
│  CDocument  │ ◄─────────────────► │   CView     │
│ (데이터 보관) │                     │ (화면 표시)  │
└─────────────┘                     └─────────────┘
       ▲                                   │
       │ 소유                              │ 소유
       ▼                                   ▼
┌───────────────────────────────────────────────┐
│               CFrameWnd (메인 창)                │
└───────────────────────────────────────────────┘
```

- 하나의 `CDocument`가 여러 개의 `CView`를 가질 수 있음 (예: 워드 "창 나누기")
- Document 변경 시 `UpdateAllViews()` 호출 → 연결된 모든 View의 `OnUpdate()` 호출되어 화면 갱신

```cpp
void CMyDocument::SetContent(const CString& str) {
    m_strContent = str;
    SetModifiedFlag();
    UpdateAllViews(nullptr); // 모든 View에 "다시 그려라" 알림
}
```

---

## ⚖️ Orthodox C++ vs Modern C++

| 항목 | Orthodox (MFC 전통 방식) | Modern C++ 관점 |
|------|---------------------------|-----------------|
| 클래스 등록 | `DECLARE_DYNCREATE`/`IMPLEMENT_DYNCREATE` 매크로 | 템플릿/팩토리로 대체 가능하나, MFC 특성상 매크로 유지가 실무적으로 안전 |
| 메시지 처리 | 메시지 맵(`BEGIN_MESSAGE_MAP`/`ON_WM_PAINT()`) | 내부적으로 함수 포인터 테이블 — `std::function` 콜백과 개념 유사, MFC는 매크로 기반 |
| 데이터-화면 분리 | Document/View 아키텍처, 수동 `UpdateAllViews()` | 최신 UI 프레임워크(React 등)의 상태 기반 자동 리렌더링과 철학 유사, 구현은 수동적 |
| 메모리 관리 | `CObject` 기반, 참조 카운팅 없음, 수동 관리 | 스마트 포인터 사용 제한적 — MFC 객체 생명주기는 프레임워크가 관리 |

> 💡 **ImGui와의 대비**: MFC Document/View는 "데이터 변경 → 알림(`UpdateAllViews`) → 다시 그리기" 구조. 반면 ImGui는 매 프레임 다시 그리는 Immediate Mode 방식이라 "변경 알림" 개념 자체가 없음 (3단계에서 재등장)

---

## 🏭 MFC/Win32 실무 연관

- `CWnd`는 `HWND`를 멤버로 감싼 것 → Win32 API 지식이 MFC 이해의 기반
- Document/View 분리 구조는 실무에서 "저장 파일 포맷"과 "화면 표시 로직"을 독립적으로 유지보수할 때 유용
- 다중 뷰(멀티 뷰) 구조는 동일 데이터를 여러 창/패널에서 동시에 보여줘야 하는 실무 요구사항에 대응

---

## 🌐 크로스플랫폼 주의사항

```cpp
// MFC는 Windows 전용 프레임워크 - CWnd, CDocument, CView 모두 windows.h 의존
// macOS/Linux 이식 시 인터페이스로 추상화 필요

class IDocument {  // CDocument 역할을 인터페이스로 추상화
public:
    virtual ~IDocument() {}
    virtual void save() = 0;
    virtual void load() = 0;
};

class IView {       // CView 역할을 인터페이스로 추상화
public:
    virtual ~IView() {}
    virtual void render() = 0; // ImGui라면 매 프레임 호출
};
```

> 3강(다형성/추상 클래스)의 인터페이스 패턴을 그대로 재사용 — MFC 개념을 이해해두면 ImGui 전환 시 "Document/View 역할을 누가 대신하는가" 설계가 빨라짐

---

## ⚠️ 자주 하는 실수

| 실수 | 올바른 방법 |
|------|------------|
| Document 데이터 변경 후 `UpdateAllViews()` 호출 누락 | 데이터 변경 시 반드시 호출하여 화면 갱신 보장 |
| `SetModifiedFlag()` 호출 누락 | 데이터 변경 시마다 호출해야 저장 안내창이 정상 작동 |
| View에서 Document 없이 데이터 접근 시도 | `GetDocument()`로 항상 유효성(`nullptr`) 체크 후 접근 |
| MFC 코드를 그대로 macOS에서 컴파일 시도 | 인터페이스(`IDocument`, `IView`)로 추상화 후 플랫폼별 구현 분리 |
| `DECLARE_DYNCREATE` 매크로를 임의로 생략 | MFC 프레임워크의 동적 생성 메커니즘에 필요하므로 유지 |

---

## 🔨 컴파일 명령어

```bash
# MFC는 Visual Studio 프로젝트(MFC 앱 마법사) 기반이 일반적
# 직접 커맨드라인 컴파일 시:
cl /std:c++17 /EHsc /MD /D_AFXDLL main.cpp /link mfc140u.lib

# macOS: 컴파일 불가 (MFC는 Windows 전용) — 개념 학습 및 인터페이스 설계 참고용
```

---

## 🔗 연결 노트

- 이전: [[CPP_Win32_1_HandleHwnd]] — Win32 API 기초
- 다음: 멀티스레딩 (Windows 스레드)
