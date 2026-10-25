# 반복 조회와 모듈 제거 오류 분석

2026-10-25, Linux 6.8.0-147-generic x86-64, QEMU 2 vCPU / 768 MiB RAM에서 원본과 수정본을 각각 실행했습니다. 호스트 커널에는 실험 모듈을 적재하지 않았고, 게스트는 디스크·네트워크 없이 실행했습니다.

## 원본에서 확인한 문제

- 여러 proc 파일과 여러 open이 공유하는 데이터를 close마다 해제해 use-after-free / 중복 해제가 발생했습니다.
- 타이머·spinlock 문맥에서 잠들 수 있는 메모리 할당, 프로세스 메모리 잠금, proc 엔트리 제거를 수행했습니다.
- 프로세스·mm·부모 프로세스의 참조 수명을 보장하지 않고, 중복 hash 엔트리가 쌓이는 구조였습니다.
- 엔트리 제거가 진행 중인 callback을 기다리는 동안 같은 잠금을 유지할 수 있었습니다.

`panic_on_warn=0`인 게스트에서 반복 조회는 `__slab_free -> kfree -> proc_release` 경로의 kernel BUG와 cat 실패를 재현했습니다. rmmod에서는 `scheduling while atomic` 오류가 발생했습니다. 별도의 `panic_on_warn=1` 실험에서는 처음부터 존재하지 않는 proc 엔트리를 지우는 경고가 panic으로 전환됐습니다. 두 결과를 구분하며, 과거 VM의 정확한 커널·설정과 동일한 재현이라고 주장하지 않습니다.

## 로컬 수정 방향

수집을 delayed work로 옮기고, task/mm 참조를 확보한 뒤 접근하도록 했습니다. PID별 자료를 갱신하고 종료한 PID를 정리하며, open마다 별도 스냅샷을 만들어 close에서 그 복사본만 해제하도록 변경했습니다. read/seek 중 출력이 바뀌지 않도록 open 시각도 고정했습니다. work 취소와 proc callback 종료를 기다린 뒤 메모리를 해제합니다. 메모리 조회는 x86-64의 일반 PTE 매핑으로 범위를 제한하고 huge/devmap/비상주 페이지는 주소 0으로 표시합니다.

수정본에서 확인한 항목은 다음과 같습니다.

- 한 바이트 단위 read와 seek 후 동일한 출력
- 비관리자 접근 거부
- 4개 동시 reader의 총 400회 조회
- 200회 fork/exec/mmap/mprotect/munmap
- 종료한 PID의 proc 파일 정리
- 열린 FD를 유지한 상태에서 모듈 제거
- 50회 적재·조회·제거 반복, kernel BUG/Oops/WARN/panic 없음

이 검증은 특정 Linux 6.8 환경의 스트레스 테스트이며 모든 커널 버전·아키텍처·장기 실행 안전성을 증명하지 않습니다. KASAN·lockdep 검증을 했다는 의미도 아닙니다. 원본과 파생 수정 소스는 수업의 공개 제한에 따라 로컬에 보관하며, 별도 공개 구현은 [proc-lens](https://github.com/JOON0512/proc-lens)에서 볼 수 있습니다.
