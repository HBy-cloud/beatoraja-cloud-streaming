**2026.10.30 \| 설계 검토안 v3.0 \| Windows · beatoraja · Google Drive**

beatoraja Cloud VFS Streaming

**개발 명세서 v3 — 용량 절약 우선 / 미캐시 로딩 15초 목표**

기준: 첨부 v2 개발 명세서 + 사용자가 확정한 PC/인터넷 조건 + 2026-10-30 공식 문서 검증

| **우선순위** | **최종 요구사항**                                                               |
|--------------|---------------------------------------------------------------------------------|
| P0 저장 공간 | SSD 500GB / 현재 여유 약 300GB / Google Drive 5TB / 전체 라이브러리 로컬 미복제 |
| P0 안정성    | BMS가 정상 플레이되어야 하며 원본 Google Drive는 읽기 전용                      |
| P1 로딩      | 미캐시 곡 플레이 시작부터 화면 준비까지 15초 이내 목표(보장 아님)               |
| P1 캐시      | 초기 30GiB 목표, 50GiB 확장은 측정 후 결정, SSD 여유 100GiB 이상 보호           |
| P2 자동화    | Java CLI prewarm 및 모니터링 → 플레이 기록 자동화는 후순위                      |

# 핵심 결론

기본안은 폴더 기반 rclone read-only mount + VFS full 캐시다. beatoraja 0.8.9의 실험적 ZIP 직접 실행은 대안으로 비교하되 기본 구조로 확정하지 않는다. 먼저 5~10곡을 실제 플레이하고, 15초 목표 달성을 방해하는 병목을 측정한다.

## 문서의 검증 수준

> • 확정된 요구: 사용자 제공 SSD·인터넷·스토리지 조건과 우선순위.
>
> • 공식 문서로 확인: rclone VFS full, 캐시 한계, Google Drive 파일 요청 병목, beatoraja 0.8.9 실험적 ZIP 지원.
>
> • 실측 필요: 사용자 설치 버전, Google Drive 원격 지연, beatoraja 첫 로딩, BMS별 파일 수, 캐시 효과, 실제 DB 스키마.

# 1. 목표, 측정 정의, 비범위

## 1.1 성공 조건

| **지표**  | **MVP 인수 기준**                                            | **측정법**                                         |
|-----------|--------------------------------------------------------------|----------------------------------------------------|
| 원본 보존 | 마운트에서 새 파일 쓰기/변경이 실패함                        | B: 쓰기 거절 확인 + 원본 샘플 해시/목록 비교       |
| 용량      | 설정 캐시 목표 30GiB, 실제 사용량 기록, 여유 100GiB 보호     | 캐시 실제 디스크 할당량 + Get-Volume/드라이브 여유 |
| 플레이    | 테스트 20곡에서 곡 로딩 및 재생 결과 기록; 실패 시 원인 분류 | 곡별 pass/fail 및 로그                             |
| 로딩 시간 | 15초 이내 비율과 p50·p95를 측정; 15초 전곡 보장은 미확정     | 선택 후 플레이 시작 요청 → 플레이 준비까지         |
| 안정성    | 403/429/5xx·인터넷 끊김 발생 시 실패 원인과 복구 가능        | rclone/Java/게임 로그                              |

15초는 목표 기준이다. “모든 미캐시 BMS가 반드시 15초 이하”라는 강제 약속은 현재 증거로 할 수 없다. 초기에는 미캐시 성공률과 15초 이내 비율을 모은 다음 현실적인 SLO를 정한다.

## 1.2 포함 / 제외

> • 포함: Drive 읽기 전용 가상 드라이브, 작은 범위 인덱싱, Java 목록 prewarm, 디스크 가드, 로딩 로그, 성능 비교.
>
> • MVP 제외: beatoraja 소스 수정, 선택 화면 실시간 감지, GUI, 전체 ZIP 일괄 변환, 원본 파일 이동/삭제, AI 런타임.
>
> • 다음 단계: DB 자동 우선순위, 선택 시점 선행 로딩, 선택적 ZIP 보관 정책은 검증 후 제안.

# 2. 시스템 구성

> Google Drive 5TB (BMS 원본 / 접근 read-only)
> ↓ Google Drive API
> rclone mount ── WinFsp ── B:\\ (고정 경로)
> │ │
> └─ VFS full cache └─ beatoraja (플레이/검색)
> │ │
> SSD Cache 로컬 DB/설정
> ↑ │
> Java CLI prewarm (선택) ← 읽기 전용 DB 조회(추후)

## 2.1 로컬 저장 정책

> • Google Drive: 원본 보관. beatoraja가 직접 원격을 수정할 경로는 제공하지 않음.
>
> • B: 가상 드라이브: 곡 경로로 사용하며 재부팅 뒤에도 드라이브 문자가 유지되도록 실행 절차 고정.
>
> • 캐시: NTFS 로컬 SSD의 전용 폴더에 배치. rclone VFS full의 sparse file 특성을 고려하여 “표시 파일 크기”와 실제 할당 디스크 사용량을 구분.
>
> • 로컬 DB: songdata.db, player별 score.db 등은 가상 드라이브 밖에 유지. 경로 변경 시 인덱싱/DB 영향을 검토.
>
> • 클라우드 폴더가 너무 크면 소수의 서브폴더를 먼저 인덱싱. 전체 라이브러리 자동 스캔을 기본값으로 두지 않음.

# 3. rclone 마운트 초안(M0)

## 3.1 사전 점검

> • rclone version, rclone config show(민감정보 출력 주의), java -version, beatoraja 버전, WinFsp 설치 상태 확인.
>
> • Google Drive 개인 OAuth Client ID/Secret 준비. rclone 공식 문서는 공유 Client ID가 2026년 중 중단된다고 안내함.
>
> • 캐시가 실제로 SSD(권장 NTFS)에 위치하는지 확인. B:가 이미 사용 중이라면 별도 문자로 변경 후 DB 경로 정책 동기화.
>
> • 설치 프로그램의 antivirus 예외는 보안 위험을 고려해 측정 후 필요한 최소 범위에서만 적용. 기본 단계에서는 제외.

## 3.2 실행 배치(초안: CMD)

> @echo off
> setlocal
> set "REMOTE=gdrive_bms:BMS"
> set "DRIVE=B:"
> set "CACHE=%LOCALAPPDATA%\BmsCloudVfs"
> if not exist "%CACHE%" mkdir "%CACHE%"
>
> rclone mount "%REMOTE%" "%DRIVE%" ^
> --read-only ^
> --vfs-cache-mode full ^
> --vfs-cache-max-size 30G ^
> --vfs-cache-min-free-space 100G ^
> --vfs-cache-max-age 336h ^
> --vfs-cache-poll-interval 1m ^
> --dir-cache-time 12h ^
> --poll-interval 5m ^
> --buffer-size 4M ^
> --vfs-read-ahead 8M ^
> --cache-dir "%CACHE%" ^
> --log-level INFO ^
> --log-file "%CACHE%\rclone.log"

위는 실행 출발점이며 사용자 rclone 버전, Google Drive remote 이름/경로 및 Windows 환경을 확인한 후 실제 실행한다. --network-mode는 호환성 문제가 있을 경우 A/B 시험하는 선택 옵션이다. --vfs-refresh는 대규모 라이브러리에서 전체 디렉터리 열거를 유발하므로 초기 설정에서 제외한다.

## 3.3 매개변수 정책

| **설정**                           | **초기값**          | **이유/검증**                                        |
|------------------------------------|---------------------|------------------------------------------------------|
| --read-only                        | 활성                | 원본 쓰기 차단. 테스트에서 검증                      |
| --vfs-cache-mode                   | full                | 랜덤 읽기를 포함한 읽기 데이터를 로컬 디스크에 캐시  |
| --vfs-cache-max-size               | 30G                 | rclone 관리 목표. 절대 상한은 아니므로 모니터링 필요 |
| --vfs-cache-min-free-space         | 100G                | 디스크 보호 기준. 정리 지연/열린 파일 예외 주의      |
| --vfs-cache-max-age                | 336h                | 오래 사용하지 않은 캐시 제거 후보                    |
| --vfs-cache-poll-interval          | 1m                  | 주기적 정리. 잦은 변경보다 초기 상태를 관찰          |
| --dir-cache-time / --poll-interval | 12h / 5m            | 탐색 메타데이터 캐시; 변경 인식 지연 체크            |
| --buffer-size / --vfs-read-ahead   | 4M / 8M             | 작은 키음 다수 사용 시 메모리·사전읽기 균형          |
| --network-mode                     | 기본 off, 실험용 on | Windows에서 beatoraja 인식·접근 호환성 비교          |
| --vfs-refresh                      | off                 | 전체 재귀 스캔 회피                                  |
| chunk 튜닝                         | 기본값 우선         | 첫 테스트 이전 과도한 옵션 최적화 금지               |

## 3.4 안전한 실행·종료

> • M0: 새 CMD에서 배치 실행 → B: 경로 확인 → 작은 곡 폴더만 인덱싱 → 정상 플레이 → 게임 종료 → rclone 정상 종료.
>
> • M0 쓰기 금지 시험은 B: 내 임시 이름으로 쓰기 시도를 하되, --read-only 확인 후 진행. Google Drive 원본에 변형이 없어야 함.
>
> • 장애 시 rclone 및 beatoraja를 정상 종료하고, 원격 설정과 로컬 DB 백업은 유지한다. 활성 캐시 내부 파일 삭제 금지.

# 4. Java prewarm CLI 설계(M2/M3)

## 4.1 책임 분리

| **모듈**      | **역할**                      | **필수 제약**                                                  |
|---------------|-------------------------------|----------------------------------------------------------------|
| ListSource    | 사용자 지정 폴더 목록 읽기    | B: 아래의 허용 폴더로 한정, 잘못된 경로 무시                   |
| FolderScanner | 곡 리소스 파일 열거/용량 예상 | 파일/폴더 중복 제외, 심볼릭 링크/경로 탈출 제한                |
| BudgetPlanner | 우선순위와 사전읽기 예산 결정 | 기본 5GiB 단일 실행 예산; 캐시 상한과 분리                     |
| DiskGuard     | SSD 사용량/여유 확인          | 여유 110GiB 이하이면 새 작업 보류, 100GiB 미만 즉시 중단       |
| WarmRunner    | 파일 끝까지 읽기(쓰기 없이)   | 처음 2개 동시, 차후 4개 비교; 제한된 재시도                    |
| StatsReporter | CSV/JSON 로그                 | 읽은 파일 수, 논리 바이트, 시간, 실패 유형; hit 추정 표시 금지 |
| DbSource (M5) | 플레이 기록 기반 대상 목록    | 실제 SQLite 스키마 확인 후 read-only 구현                      |

## 4.2 CLI 계약

> bms-warm warm --list today.txt --dry-run
> bms-warm warm --list today.txt --threads 2 --budget 5G
> bms-warm status --cache-dir "%LOCALAPPDATA%\BmsCloudVfs"
> bms-warm verify --list today.txt (M3 이후 선택 기능)

CLI 명령과 옵션은 계약 초안이다. M2에서는 warm --list, --dry-run, --threads, --budget 및 CSV 출력까지만 구현하고, status/verify는 M3에서 추가한다.

## 4.3 동작 순서

> • 입력 폴더를 검증하고 중복을 제거한다. 모든 경로는 마운트 루트 B: 아래여야 한다.
>
> • 곡별 파일 수·합산 바이트를 조사한다. dry-run도 폴더 열거 API를 호출할 수 있다.
>
> • 남은 SSD 여유 공간과 prewarm 예산을 확인한다. 안전 한계에서 새 읽기를 시작하지 않는다.
>
> • 각 폴더의 파일을 읽고 내용을 버린다. 실제 다운로드/캐시는 rclone이 수행한다.
>
> • 403/429/일시적 네트워크/5xx는 제한된 횟수에 걸쳐 backoff로 재시도한다. 영구적인 404는 기록 후 넘긴다.
>
> • 시작·종료 시각, 논리 읽기량, 파일 수, 결과, 오류를 CSV로 남기고 종료 코드를 반환한다.

## 4.4 리소스 필터링 주의

BMS는 키음만이 아니라 배경 이미지·영상, 각종 참조 파일이 필요할 수 있다. 초기 prewarm에서 영상 확장자를 무조건 제외하면 플레이 중 추가 원격 읽기/지연이 발생할 수 있다. 전체 리소스 포함을 기본으로 하고 영상 생략은 재생 무결성을 검증한 선택 옵션으로만 제공한다.

## 4.5 예외 정책

| **상황**                      | **대응**                                                                |
|-------------------------------|-------------------------------------------------------------------------|
| Google Drive 403 / rate limit | 요청률 감소, 지수 백오프, 최대 시도 후 실패 목록 기록                   |
| 네트워크 잠시 끊김            | 다른 작업 중단하지 않되 재시도 횟수 제한, 최종 실패 분리                |
| 원격 파일이 삭제됨            | skip + 404 기록. DB의 stale path를 삭제·수정하지 않음                   |
| SSD 여유 부족                 | 신규 prewarm 시작 중지, 사용자 경고. 활성 VFS 캐시를 직접 삭제하지 않음 |
| rclone 마운트 종료            | 즉시 읽기 중단, 종료 코드/원인 기록                                     |
| DB 잠금/스키마 불일치         | M5에서 안전한 읽기 전용 접근, 마운트·플레이는 독립 유지                 |

# 5. DB 연계(M5: 나중에)

v2에는 score.db와 songdata.db의 테이블/컬럼이 가정으로 표기되어 있다. beatoraja 공개 소스에는 song 테이블의 sha256/path/folder 등이 확인되지만, 사용자 설치 버전의 실제 DB와 score의 playcount/date 구조는 직접 조회한 다음 확정해야 한다.

> • 사전 점검 SQL: PRAGMA database_list; SELECT name FROM sqlite_master WHERE type='table'; PRAGMA table_info(song);
>
> • score.db는 플레이어별 위치, 실제 스키마, 타임스탬프 단위, 플레이 횟수의 누적 의미를 확인한다.
>
> • 필요하다면 DB 읽기 전용 연결에서 ATTACH하며, SQLite 잠금이 있으면 공식 backup API 또는 안전한 일관성 있는 스냅샷을 사용한다. 단순 파일 복사는 WAL 동기화 위험을 검토한다.
>
> • SHA-256 일치 → 곡 파일 경로 → 상위 폴더(또는 ZIP 단위) → 최근 활동/횟수에 따른 선정. 차분 중복 제거.
>
> • 사용자가 선택한 곡을 즉시 파악하는 기능은 DB 자동 선정과 별개이다. 인게임 선곡 감지는 M5 범위 밖.

# 6. ZIP 직접 실행 비교(M4: 실험)

beatoraja 0.8.9 릴리스 노트는 ZIP 및 일부 RAR 파일을 임시 추출 없이 읽는 기능을 실험적으로 추가했다고 안내하며, archive detection은 기본 비활성이다. 사용자 설치 버전에서 옵션과 실제 호환성을 확인한 후 진행한다.

| **A: 폴더 원본**   | **B: ZIP 원본**            | **검증 내용**                    |
|--------------------|----------------------------|----------------------------------|
| 원격 파일 수 다수  | 곡별 압축 파일 1개         | 스캔/첫 로딩 시간과 API 요청 수  |
| 변환 불필요        | ZIP 변환 및 관리 필요      | 실제 변환 비용·용량 이득         |
| 키음 개별 다운로드 | ZIP 내부 random seek 가능  | 부분 읽기 횟수·실제 전송량       |
| 기존 DB 경로 유지  | ZIP 경로로 재인덱싱 가능성 | 미리 복제한 테스트 DB에서만 비교 |

ZIP 하나로 묶는다고 곧바로 “요청 1회”가 되는 것은 아니다. 중앙 디렉터리 및 랜덤 액세스로 원격 범위 요청이 발생할 수 있다. 실제 측정에서 의미 있는 개선이 확인되지 않으면 기존 폴더 방식을 유지한다.

# 7. 실험과 통과 기준

## 7.1 테스트 데이터셋

> • 기능 검증: 최소 5~10곡(작은 곡, 키음 많은 곡, BGA 포함, 이미지 다수, 차분 포함).
>
> • 성능 검증: 서로 다른 20곡 이상. 각 곡의 BMS 크기, 키음 수, 동영상 유무, 폴더 파일 수를 기록한다.
>
> • 네트워크 조건: 테스트 시간대, 유선/무선, 다른 대역폭 점유 프로그램을 기록한다.

## 7.2 시간 측정 표준

| **항목**  | **측정 정의**                                                                           |
|-----------|-----------------------------------------------------------------------------------------|
| 시작 t0   | beatoraja에서 플레이 시작 명령이 발생한 순간                                            |
| 종료 t1   | 실제로 플레이 화면이 준비되어 게임이 진행 가능한 순간                                   |
| 로딩 시간 | t1 - t0. 인덱싱/앱 실행 시간은 별도 지표                                                |
| Cold      | 테스트 전 해당 곡이 VFS 데이터 캐시에 없음을 확보한 상태. 안전한 격리/새 캐시 환경 이용 |
| Warm      | 동일 곡을 직후 다시 로딩한 상태                                                         |
| Prewarm   | CLI로 미리 읽은 뒤 처음 플레이한 상태                                                   |
| 지표      | p50/p95, 15초 이내 비율, 실패율, 실제 캐시 디스크 사용량                                |

cold 검증에서 현재 마운트가 사용하는 캐시 파일을 직접 지우지 않는다. 테스트 전 rclone을 정상 종료하고 격리된 cache-dir을 사용하거나 새 샘플 곡을 선택한다.

## 7.3 테스트 케이스

| **ID** | **시나리오**            | **확인**                         |
|--------|-------------------------|----------------------------------|
| T01    | B: read-only 쓰기 거부  | 원격 원본 변경 없음              |
| T02    | 미캐시 일반 곡          | 정상 재생 및 15초 판정 기록      |
| T03    | 캐시 재로딩             | 시간·원격 요청 변화 기록         |
| T04    | prewarm 적용            | 바로 재로딩과 지연 비교          |
| T05    | 키음 다수/영상 포함     | 누락 자산·음성 지연·오류 확인    |
| T06    | 인터넷 끊김/복귀        | 오류 분류와 정상 재시도          |
| T07    | 30GiB 캐시 근접         | rclone 정리 정책과 초과폭 관찰   |
| T08    | SSD 여유 110GiB 임계    | 새로운 prewarm 작업 차단         |
| T09    | 한글/특수문자/중첩 경로 | 파일 탐색 및 정확한 로딩         |
| T10    | ZIP 비교(옵션)          | 10개 짝지은 곡 비교; 재생 무결성 |

# 8. 개발 단계 / 완료 조건(DoD)

| **단계**       | **핵심 산출물**                     | **완료 조건**                                |
|----------------|-------------------------------------|----------------------------------------------|
| M0 환경·마운트 | 진단 기록, mount_bms.bat, 복원 절차 | 5~10곡 play OK; 원본 read-only; DB 로컬 보존 |
| M1 베이스라인  | cold/warm 성능 CSV, 로그            | 20곡 이상 기준 로딩 분포·원인 기록           |
| M2 prewarm MVP | CLI 목록/스캔/병렬 읽기/dry-run     | 지정 폴더 prewarm, 재로딩과 비교 가능한 수준 |
| M3 운영 안정화 | DiskGuard, retry, status/log        | 100GiB 보호, 원인별 실패 로그, 안전 종료     |
| M4 ZIP A/B     | 폴더 대 ZIP 비교 리포트             | 고객 데이터 일괄 변환 없이 도입 여부 결정    |
| M5 DB 자동화   | SQLite 스키마 리포트, DB source     | 읽기 전용 조회와 중복 제거 검증              |
| M6 선택 고도화 | 선곡 감지/GUI/알림 등 필요 시       | 측정상 필요성이 입증된 기능만 채택           |

## 8.1 최초 실행 체크리스트

> • □ Windows 디스크 문자/캐시 저장 볼륨 확인
>
> • □ rclone/WinFsp/beatoraja/JDK 버전 기록
>
> • □ Google Drive 인증 remote 만들기(비밀정보는 공유하지 않기)
>
> • □ 로컬 beatoraja DB/설정 백업
>
> • □ 소수 테스트곡을 Drive에 준비
>
> • □ 마운트 실행·읽기·쓰기 거부 확인
>
> • □ beatoraja 소수 곡만 스캔 후 첫 로딩 기록

# 9. 개발 프로젝트 구성과 산출물

> bms-cloud-streaming/
> ├─ README.md
> ├─ docs/
> │ ├─ specification-v3.md
> │ ├─ decisions.md \# ADR 설계 의사결정
> │ └─ benchmark-template.csv
> ├─ scripts/
> │ ├─ mount_bms.bat
> │ └─ check_environment.ps1
> ├─ app/
> │ ├─ build.gradle
> │ └─ src/main/java/.../ \# CLI / scanner / guard / runner / report
> ├─ app/src/test/java/.../
> └─ logs/ \# gitignore 대상

권장 빌드: Java 17 이상 + Gradle 또는 Maven + picocli + sqlite-jdbc(M5에서 도입) + JUnit 5. 런타임 서버/LLM 프레임워크는 필요하지 않다. 캐시와 인증 파일, DB 백업, 로그는 Git에 커밋하지 않는다.

## 9.1 기록 형식(CSV 제안)

> timestamp,chart_id,folder,storage_mode,cache_state,load_seconds,
> files_count,logical_bytes_read,free_space_gib,result,error_class,notes

chart_id는 실제 BMS SHA-256을 확인한 후 채운다. 캐시 상태는 실측/검증 방법을 명확히 기록하며, 단순히 두 번째 재생이라는 이유만으로 완전 hit라고 판단하지 않는다.

# 10. 미결정 사항과 위험

| **항목**                         | **현재 상태**    | **확정 방법**                   |
|----------------------------------|------------------|---------------------------------|
| beatoraja 설치 버전              | 미확정           | 실행 화면/릴리스 확인           |
| rclone/WinFsp/JDK 버전           | 미확정           | 명령 및 설치 기록               |
| 캐시 설치 볼륨/NTFS              | 미확정           | Windows 볼륨 정보 확인          |
| 실제 Google Drive 접근/전송속도  | 미측정           | 소수 곡 cold 실험과 로그        |
| B: 드라이브 접근 호환성          | 미측정           | fixed / network 모드 비교       |
| songdata.db·score.db 실제 스키마 | 미확정           | 실제 사본에서 PRAGMA 확인       |
| ZIP 기능의 실효성                | 실험 기능/미측정 | 20곡 중 짝지은 10곡 A/B         |
| 보관할 곡 총량/패키징            | 미확정           | 실제 라이브러리 통계            |
| 15초 이내 달성 범위              | 미측정           | 20곡 이상 성능 분포와 병목 분류 |

# v2 → v3 주요 변경

> • 100GB 기본 캐시 → 30GiB 시작, 최대 50GiB 확장 검토. 용량 사용 최소화를 더 높은 우선순위로 조정.
>
> • “캐시 30G = 엄격한 한도”가 아님을 명시; 디스크 보호 기준 및 안전 여유 추가.
>
> • 미캐시 15초 목표를 정량 측정(SLO 후보)으로 구체화; 모호한 “로컬 수준” 대신 p50/p95, 이내 비율 기록.
>
> • 전체 디렉터리 재귀 사전 적재(--vfs-refresh) 제거; 시작 시 API 폭주 위험 감소.
>
> • 영상 확장자 기본 제외를 재검토; 완전한 BMS 로딩이 먼저라는 원칙 반영.
>
> • ZIP 직접 실행을 “실험적 선택지”로 등록하고 실제 A/B 테스트 전 일괄 변환 금지.
>
> • DB 자동화는 뒤로 미루고 M0~M3에서 실효성과 안전성부터 검증.

# 참고 자료(2026-10-30 확인)

> • rclone VFS Mount 문서: https://rclone.org/commands/rclone_mount/
>
> • rclone Google Drive 백엔드 문서: https://rclone.org/drive/
>
> • beatoraja 공식 릴리스 노트(0.8.9): https://github.com/exch-bms2/beatoraja/releases
>
> • beatoraja Song Database 코드: https://github.com/exch-bms2/beatoraja/blob/master/src/bms/player/beatoraja/song/SQLiteSongDatabaseAccessor.java
>
> • 원본 문서: 사용자 첨부 “beatoraja Cloud VFS Streaming v2 개발 명세서.docx” (2026-10-30)

이 문서는 구현 전 검증을 포함한 설계안이다. 사용자의 실제 PC 또는 Google Drive에 대한 명령 실행이나 성능 벤치마크를 수행한 문서는 아니다.
