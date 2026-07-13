---
tags: [security, mitre-attack, tactics, techniques, edr, fundamentals]
---

# SEC_Theory_5: MITRE ATT&CK 프레임워크 입문

## 🧠 핵심 개념 한 줄 요약
MITRE ATT&CK은 실제 관찰된 공격 사례를 바탕으로 공격자의 **Tactic(전술, Why)**과 **Technique(기법, How)**을 체계적으로 정리한, 보안 업계의 공통 언어이자 EDR 탐지 설계의 기준점이다.

## 📦 핵심 개념 상세

### 비유: 요리 레시피 백과사전
- 요리(공격)마다 재료(도구)와 조리 순서(기법)가 있음
- 상상이 아니라 **실제 침해사고에서 관찰된 기법들의 집합체**라는 게 핵심

### 구조: Tactic(전술) vs Technique(기법)

| 개념 | 의미 | 비유 |
|---|---|---|
| Tactic (전술) | 공격자가 달성하려는 "목표" (Why) | 요리의 "단계" (전채 → 메인 → 디저트) |
| Technique (기법) | 그 목표를 달성하는 구체적 "방법" (How) | 각 단계의 "구체적 조리법" (볶기, 찌기, 굽기) |

이전 시간에 배운 공격 5단계는 사실 Tactic(전술)들이다.

| Tactic (전술) | 이전 시간 개념 |
|---|---|
| Initial Access | 피싱 |
| Execution | 익스플로잇 |
| Privilege Escalation | 권한 상승 |
| Lateral Movement | 측면 이동 |
| Command and Control | C2 통신 |

### 전체 Tactic 흐름 (Enterprise Matrix 기준)
```
Reconnaissance (정찰) → Resource Development (자원 개발) → Initial Access (초기 접근)
→ Execution (실행) → Persistence (지속성 유지) → Privilege Escalation (권한 상승)
→ Defense Evasion (방어 회피) → Credential Access (자격 증명 접근) → Discovery (탐색)
→ Lateral Movement (측면 이동) → Collection (수집) → Command and Control (C2)
→ Exfiltration (탈취) → Impact (영향)
```

EDR 개발에서 특히 중요한 Tactic:
- **Persistence (지속성 유지)**: 재부팅 후에도 계속 실행되게 만드는 기법 (레지스트리 Run 키 등록, 서비스 등록) → EDR 레지스트리 모니터링과 직결
- **Defense Evasion (방어 회피)**: 백신/EDR 탐지 회피·무력화 기법 (루트킷, 프로세스 인젝션) → 이전 시간의 루트킷이 여기 속함
- **Credential Access (자격 증명 접근)**: 비밀번호/토큰 탈취 (LSASS 메모리 덤프 등) → CIA 기밀성 침해와 직결

### Technique ID 체계
각 기법에는 T번호가 부여된다.

```
T1566        = Phishing (피싱) — Tactic: Initial Access
T1203        = Exploitation for Client Execution — Tactic: Execution
T1068        = Exploitation for Privilege Escalation — Tactic: Privilege Escalation
T1021        = Remote Services — Tactic: Lateral Movement
T1071        = Application Layer Protocol — Tactic: Command and Control
T1071.004    = DNS (Sub-technique, 하위 기법)
T1014        = Rootkit — Tactic: Defense Evasion
T1003        = OS Credential Dumping — Tactic: Credential Access
```

> T1071.004처럼 점(.) 뒤 숫자가 붙은 건 Sub-technique(하위 기법) — 상위 기법을 더 구체적으로 나눈 것.

## 🔗 EDR과의 연관성
1. **탐지 룰 설계의 기준**: EDR 회사들은 "우리 제품이 몇 개의 T번호 기법을 탐지 가능한가"를 커버리지 지표로 사용 (제품 스펙에 "MITRE ATT&CK 커버리지 92%" 같은 표현 등장)
2. **공통 언어**: "이건 T1055(Process Injection)입니다"라고 하면 보안 엔지니어끼리 바로 통함 — 면접에서 이 용어를 쓸 줄 알면 실무 감각이 있다는 인상을 줌
3. **MITRE ATT&CK Evaluation**: MITRE 재단이 매년 EDR 제품 대상 실제 공격 시나리오 재현 평가 진행
4. **탐지 로직 설계**: "이 T번호에 해당하는 행위 패턴을 코드로 어떻게 구현할까"가 실제 개발 작업. 예: T1055(Process Injection) 탐지 → `CreateRemoteThread`, `WriteProcessMemory` 등 API 호출 시퀀스 감시 코드 작성
   → 포트폴리오 EDR 프로젝트도 T번호 단위로 설계하면 체계적임

### 참고
- attack.mitre.org에서 모든 Tactic/Technique 무료 공개
- 프로젝트 문서에 "이 프로젝트는 T1055, T1003을 탐지 목표로 한다"는 식으로 명시하면 포트폴리오 완성도 상승

## ⚠️ 자주 하는 오해/실수
- Tactic과 Technique을 같은 레벨로 혼동하기 쉬움 — Tactic은 "목표(Why)", Technique은 "방법(How)"으로 계층이 다름
- MITRE ATT&CK을 "탐지 규칙 자체"로 오해하기 쉬운데, 실제로는 **분류 체계(Taxonomy)**일 뿐이며 탐지 로직은 별도로 구현해야 함
- T번호가 절대적/고정된 게 아니라 MITRE가 지속적으로 업데이트하는 살아있는 문서라는 점 — 최신 버전 확인 필요

## 🔗 연결 노트
- 이전: [[SEC_Theory_4_AttackTechniques]]
- 다음: [[SEC_Theory_6_CryptoBasics]]

#security #edr #mitre-attack #tactics #techniques #fundamentals
