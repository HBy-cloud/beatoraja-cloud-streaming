# 개발 진행 기록

> 프로젝트: beatoraja Cloud VFS Streaming  
> 기준 명세: [`SPEC.md`](SPEC.md) v3  
> 기록 기준일: 2026-10-31

## 현재 상태

- **단계:** M0 — 환경·소규모 마운트 검증 **시작 전**
- **직전 완료:** GitHub 저장소 및 기본 문서 구성
- **최신 사용자 확인 커밋:** `2b0bb68` (`main`, `origin/main`)
- **작업 트리:** 사용자가 마지막으로 출력한 `git status --short` 결과는 비어 있었음. 이 문서를 복사하면 신규 변경사항이 생김.
- **미검증:** rclone / WinFsp / Java / beatoraja 설치 버전, Drive 마운트, 실제 곡 플레이, 성능

## 확정된 환경과 목표

| 항목 | 현재 정보 |
|---|---|
| 운영체제 / 게임 | Windows / beatoraja |
| 저장소 | Google Drive 5TB |
| 로컬 SSD | 500GB, 여유 약 300GB |
| 인터넷 측정 | 다운로드 365Mbps, 업로드 223.8Mbps, 지연 37ms (Google Drive 직접 측정 아님) |
| 캐시 목표 | 30GiB 초기, 필요 시 50GiB 검토 |
| 디스크 보호 | 가용 공간 100GiB 이상 목표 |
| 곡 로딩 | 미캐시 15초 이내 목표, 보장 불가 |

## 마일스톤

| 단계 | 목표 | 상태 | 완료 증거 |
|---|---|---|---|
| 준비 | Git 저장소, README, AGENTS, SPEC | 완료 | `2b0bb68`, push 성공 및 clean status 사용자 확인 |
| M0 | 환경 진단, B: 읽기 전용 마운트, 5~10곡 플레이 | 시작 전 | 버전·설정 점검표, 마운트·플레이 로그 |
| M1 | 20곡 이상 cold/warm 로딩 측정 | 대기 | 로딩 CSV, p50/p95, 실패율 |
| M2 | Java 목록 기반 prewarm MVP | 대기 | JUnit 및 실제 캐싱 결과 |
| M3 | 디스크·오류 가드 | 대기 | 보호 동작 및 오류 복구 로그 |
| M4 | 폴더와 ZIP 방식 비교 | 대기 | A/B 테스트 기록 및 ADR |
| M5 | SQLite 기반 자동 대상 선정 | 대기 | 실제 DB 스키마 확인 및 읽기 전용 테스트 |
| M6 | 선택적 고도화 | 대기 | 필요성 검증 후 결정 |

## 작업 로그

### 2026-10-31 — 프로젝트 초기 구성

- GitHub 저장소: https://github.com/HBy-cloud/beatoraja-cloud-streaming
- 확인된 파일: `README.md`, `.gitignore`, `AGENTS.md`, `docs/SPEC.md`
- `28ef6eb`: 개발 명세서 및 Codex 지침 추가
- `2b0bb68`: 구형 DOCX 명세서 제거
- Git push 성공. `git status --short` 출력 없음(정상).
- 이번 작업: `docs/DECISIONS.md`, `docs/PROGRESS.md` 신규 초안 작성. **아직 사용자 PC에 배치되거나 커밋되었다고 간주하지 않음.**

## 다음 작업 — M0 환경 사전 확인

Windows **PowerShell 5.1**에서 아래 항목을 확인하되, 인증 정보는 공유하지 않는다.

```powershell
# 저장소 작업 위치
cd D:\games\beatoraja_Cloud_VFS_Streaming

# 각 실행 파일이 PATH에서 발견되는지 확인
Get-Command rclone, java, javac -ErrorAction SilentlyContinue |
    Select-Object Name, Source

# 설치된 경우에만 버전 확인
rclone version
java -version
javac -version

# B: 드라이브가 이미 사용 중인지 확인
Get-PSDrive -Name B -ErrorAction SilentlyContinue

# WinFsp 설치 위치 확인(일반적인 기본 경로)
Test-Path 'C:\Program Files (x86)\WinFsp\bin'
```

명령이 없으면 그 사실만 기록하고, **Google OAuth 토큰, client secret, `rclone.conf` 전체 내용은 공유하지 않는다.** 정확한 beatoraja 버전은 사용 중인 프로그램의 표시 또는 설치 파일에서 별도로 확인한다.

### M0 완료 조건

- [ ] rclone 버전 확인
- [ ] WinFsp 설치 및 동작 확인
- [ ] Java/JDK 버전 확인 (Java CLI 개발 전까지 미필수)
- [ ] beatoraja 실제 버전 확인
- [ ] 인증 정보 유출 없이 Google Drive 연결 구성
- [ ] B: 읽기 전용 마운트 및 캐시 디렉터리 확인
- [ ] 게임 DB/설정 백업 후 테스트곡 5~10개 인덱싱·플레이
- [ ] 실패/오류/디스크 사용량 기록

## 기록 양식

```markdown
### YYYY-MM-DD — 작업명
- 현재 단계:
- 변경한 파일:
- 실행한 명령 / 테스트:
- 결과 (성공/실패/미실행):
- 미검증 사항:
- 결정 변경(ADR 링크):
- 다음 작업:
```
