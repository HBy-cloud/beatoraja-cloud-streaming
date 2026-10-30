## 2026-10-31 — M0 실제 마운트·재생 관찰 및 실험 중단 (KST)

> 이 기록은 앞선 'M0 시작 전' 기록을 삭제하거나 소급 수정하지 않는 후속 이력이다. **현재 상태: M0 부분 검증, 추가 개발·실험 중단.** M1~M6은 완료되지 않았다.

### 환경 및 연결 확인

- Windows 11 Home 26H2; Java Temurin JDK 25.0.4.1(사용자 제공 버전); rclone v1.75.1.
- WinFsp 설치 경로 확인 및 `WinFsp.Launcher` Running 확인.
- beatoraja 0.8.8 → 0.8.9 업데이트.
- 마운트 전 C: 여유 29.1GiB, D: 여유 222.9GiB(NTFS, Healthy). 실험 후 사용량 미측정.
- 개인 Google Cloud 프로젝트/데스크톱 OAuth 클라이언트 생성, Drive API 활성화, 외부·테스트 앱과 본인 테스트 사용자 등록, `drive.readonly` 범위 사용.
- rclone remote `gdrive` 인증 및 폴더 목록 조회 성공; 테스트 경로 `gdrive:han/BMS-Test`. 테스트 상태의 인증 만료 및 지속 운영은 미검증.
- OAuth 토큰, client secret, `rclone.conf` 등 인증 비밀정보는 기록·수집·커밋하지 않는다.

### 실제 실행 (Windows PowerShell)

```powershell
rclone mount "gdrive:han/BMS-Test" B: --read-only --vfs-cache-mode full --cache-dir "D:\BMS-Cloud\cache" --vfs-cache-max-size 30G --vfs-cache-min-free-space 100G
```

- `B:` 파일 목록 확인 성공. 시작 시 `ERROR : symlinks not supported without the --links flag: /` 및 `The service rclone has been started.` 출력. 이후 목록 조회·게임 재생 가능. 오류 원인 미확인, `--links` 미추가.
- `30G`는 엄격한 사용량 상한이 아니라 캐시 정리 목표이며, `--vfs-cache-min-free-space 100G`만으로 100GiB 여유를 항상 보장하지 않는다.
- `--read-only` 및 OAuth `drive.readonly` 설정은 확인했으나 실제 쓰기 거부 시험은 미실행.
- 마운트 종료 및 캐시 정리 여부 미확인.

### 테스트 자료 및 beatoraja 설정

- 최초 확인한 압축 파일: `2r_bbkkbkk.rar`, `[Feryquitous]Distorv.rar`, `[catastrophe]Sakuzyo_Altale.zip`, `conflict_ogg_20110917.zip`, `electricity_ogg.zip`.
- 이후 Altale 압축 해제 폴더를 업로드. 업로드 시간이 길어 사용자 요청에 따라 폴더 검증은 우선 1곡으로 축소하고 ZIP 직접 실행 관찰을 앞당김. 원본 압축 파일 삭제·변환 없음.
- beatoraja 0.8.9: `Scan song archives` 체크, `Archive extraction: No temporary files` 확인. 기존 로컬 BMS Path를 유지하면서 `B:\` 추가. `Update BMS database` 후 테스트 곡 인식 확인.
- 사용자 요청에 따라 이번 DB·설정 백업 생략. 백업 완료·복구 검증 완료 아님. 향후 백업 원칙은 유지.

### 재생 관찰 (정밀 측정 아님)

| 대상 | 확인된 결과 | 한계 |
|---|---|---|
| `[catastrophe]Sakuzyo_Altale` 압축 해제 폴더 | 첫 관찰 로딩 60초 이상; 사용자 플레이 정상 확인; 나갔다가 재선택 시 체감상 즉시 로딩 | 첫 로딩의 정확한 종료 시간 미측정. 인덱싱 중 캐시 가능성 때문에 엄밀한 Cold 아님 |
| ZIP 직접 실행 | conflict ZIP을 안내한 뒤 사용자가 '되긴 된다'고 재생 가능 보고 | 실제 선택 ZIP·채보 경로 미확인; 정확한 로딩 시간, 소리·BGA 세부 검증 미실행 |
| 별도 느린 곡(이름 미확인) | 곡마다 최초 로딩 편차가 크고 7분 초과 사례 보고 | 곡 이름과 폴더/ZIP/RAR 형태 미확인; Altale 또는 특정 ZIP에 연결 금지 |

### 해석·미검증·중단 판단

- `Google Drive → rclone/WinFsp → B: → beatoraja`의 곡 인식 및 재생 가능성 확인. 15초 목표 미달 사례 확인.
- 재로딩 개선이 rclone·OS·게임 캐시 중 무엇에 기인하는지 분리하지 못함. ZIP이 폴더보다 빠르다는 결론 불가.
- 작은 파일 요청, API 지연, 압축 랜덤 읽기는 원인 후보일 뿐 검증된 원인이 아님.
- p50/p95, 실패율, 전송량, 캐시 점유량, Google Drive 실제 전송 속도, 정확한 Cold/Warm 조건 미측정.
- 실제 쓰기 거부, 원본 해시 비교, DB 복구, 5~10곡 개별 재생 완료, 실험 후 디스크 여유 및 장애 복구 미검증.
- **잠정 평가:** 최초 로딩 지연으로 현재 구성의 실사용성 부족. **M0 부분 검증**, M1~M6 미완료. 사용자 결정으로 추가 개발·성능 실험 중단.
- **재개 시 첫 단일 작업(이번에는 실행하지 않음):** 느린 곡 1개의 정확한 경로·저장 형태 확인 후 전송량과 로딩 병목 측정.
- 본 기록 작성은 문서 변경안 준비이며, 실제 저장소 수정·커밋·푸시 여부는 별도 확인이 필요하다.
