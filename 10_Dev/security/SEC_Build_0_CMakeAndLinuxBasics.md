---
tags: [#security, #edr, #build-system, #cmake, #linux, #proc-filesystem, #cross-platform]
---

# SEC_Build_0_CMakeAndLinuxBasics

## 🧠 핵심 개념 한 줄 요약
CMake는 OS 공통 설계도로 각 플랫폼용 빌드 지시서를 생성해주는 도구이고, Linux는 "모든 것은 파일이다" 철학 위에서 `/proc` 같은 가상 파일시스템만으로 프로세스·시스템 상태를 파악할 수 있다 — 이 둘이 크로스플랫폼 EDR 개발의 최소 기반이다.

---

## 📦 1. CMake 빌드 시스템

### 왜 필요한가
- Visual Studio의 `.vcxproj`는 Windows 전용. EDR/보안 SW는 Windows + Linux 양쪽에서 동작하는 경우가 많아 OS 무관 공통 빌드 설정 방식이 필요함.

### 동작 흐름
```
CMakeLists.txt (공통 설계도, OS 무관)
        ↓ cmake 실행
Makefile (Linux/macOS)  또는  .sln (Windows)
        ↓ 빌드 실행 (make / MSBuild)
실행 파일
```

### 비유
CMake = 건축 설계도(CMakeLists.txt) + 현장 감독(cmake 명령어).
설계도는 공통이지만 실제로 짓는 인부(컴파일러: GCC/Clang, MSVC)는 지역마다 다르다. CMake는 컴파일러가 아니라 "설계도를 각 지역 인부가 알아들을 작업 지시서로 번역해주는 감독"이다.

### 실습 예제
**`main.cpp`**
```cpp
#include <iostream>

int main() {
    // 나중에 이 부분이 실제 프로세스 목록 조회 API 호출로 대체될 자리
    std::cout << "EDR 학습 프로젝트: 빌드 시스템 테스트\n";
    return 0;
}
```

**`CMakeLists.txt`**
```cmake
cmake_minimum_required(VERSION 3.20)   # CMake 최소 요구 버전
project(EDRPractice CXX)               # 프로젝트 이름, 언어 선언

set(CMAKE_CXX_STANDARD 17)             # C++17 사용 (회사 표준과 통일)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(EDRPractice main.cpp)   # 실행 파일 생성 지시
```

**빌드 명령어**
```bash
brew install cmake      # macOS 설치
mkdir build && cd build
cmake ..                # Makefile 생성
make                    # 실제 컴파일 (clang++ 호출)
./EDRPractice
```

---

## 📦 2. Linux / Ubuntu 기초

### Windows vs Linux 대응 관계
| 개념 | Windows | Linux |
|---|---|---|
| 프로세스 목록 | `CreateToolhelp32Snapshot()` | `/proc` 파일시스템 읽기, `ps`/`top` |
| 권한 체계 | ACL (Access Control List) | 소유자/그룹/기타 + `chmod` 퍼미션 |
| 커널 진입 | 드라이버(.sys), WDK | LKM (Loadable Kernel Module) |
| 실행 파일 포맷 | PE (.exe) | ELF |

### "Everything is a file" 철학
- 프로세스 정보: `/proc/[pid]/status`, `/proc/[pid]/maps` 등
- 디바이스: `/dev/` 아래 파일로 노출
- 별도 API 세트 없이 파일 읽기만으로 시스템 상태 파악 가능 → Linux 보안 도구가 가벼운 이유 중 하나

### 학습 환경: Docker
- Docker Desktop for Mac은 내부적으로 경량 Linux VM 위에서 컨테이너를 실행함.
- 0~2단계(사용자 모드 프로그래밍, `/proc` 파싱, `ptrace`, `inotify`)는 Docker로 충분.
- 3단계(커널 드라이버)·5단계(eBPF)는 호스트 커널 공유 특성상 제한적 → VM(UTM 등)으로 전환 필요.

### 실습 명령어
```bash
# Ubuntu 컨테이너 실행
docker run -it --name edr-ubuntu ubuntu:24.04 bash

# 개발 도구 설치 (컨테이너 내부)
apt update && apt install -y build-essential cmake git vim

# 재접속
docker start -ai edr-ubuntu

# 프로세스 상태 확인 (Windows 프로세스 속성 창에 대응)
echo $$
cat /proc/$$/status
ls -l /proc/$$/fd
```

`/proc/[pid]/status` 주요 필드:
- `State`: 프로세스 상태 (Windows Running/Suspended와 유사 개념)
- `PPid`: 부모 프로세스 ID → 부모-자식 프로세스 트리 구성의 핵심
- `VmRSS`: 실제 사용 중인 메모리 (Windows 작업관리자 메모리 사용량과 대응)

---

## 🔗 EDR과의 연관성
- CMake: 프로세스 모니터·FIM 프로젝트를 Windows/Linux 양쪽 빌드 가능하게 만드는 표준 도구. osquery, Sysmon for Linux 등 실제 오픈소스 EDR도 CMake 기반 → 4단계 코드 리딩 시 필수.
- `/proc` 파싱: 커널 모듈 없이 가능한 가장 기초적인 Linux 프로세스 감시 방법 — 추후 FIM·프로세스 모니터 프로젝트의 Linux 쪽 기반.
- `ptrace()`, `inotify`는 2단계 실습에서 직접 코드로 사용 예정.
- Linux 권한 모델(소유자/그룹/setuid)을 모르면 MITRE ATT&CK 권한 상승(Privilege Escalation, T1068) 이해 어려움.

## ⚠️ 자주 하는 오해/실수
- "CMake = 컴파일러다" → 틀림. 빌드 설정 생성 도구일 뿐, 실제 컴파일은 GCC/Clang/MSVC가 수행.
- "CMakeLists.txt 하나면 코드 수정 없이 아무 OS에서나 빌드된다" → 틀림. 빌드 설정은 공통화되지만 Win32 API vs POSIX처럼 OS별 API 차이는 여전히 `#ifdef _WIN32` 같은 코드 분기가 필요.
- "macOS 터미널 = Linux" → 틀림. macOS는 Unix 계열이지만 커널(XNU)이 다르고 `/proc` 파일시스템 자체가 없음.
- "Docker면 커널 실습도 다 된다" → 틀림. 컨테이너는 호스트 커널을 공유하므로 커널 모듈/eBPF 실습은 VM 환경이 별도로 필요.

## 🔨 컴파일/실행 명령어 모음
```bash
# CMake
brew install cmake
mkdir build && cd build && cmake .. && make

# Docker Ubuntu
docker run -it --name edr-ubuntu ubuntu:24.04 bash
apt update && apt install -y build-essential cmake git vim
docker start -ai edr-ubuntu
```

## 🔗 연결 노트
- 이전: [[SEC_Theory_6_CryptoBasics]]
- 다음: SEC_OSInternals_1_ProcessMemory (1단계 시작)
