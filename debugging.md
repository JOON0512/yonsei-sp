# 반복 조회와 모듈 제거 중 커널이 죽던 문제

`/proc`에서 프로세스 정보를 읽는 모듈을 Linux 6.8.0-147-generic / x86-64, QEMU 2 vCPU에서 확인했습니다.

## 원인

여러 파일과 `open`이 같은 데이터를 공유하는데, `close`할 때마다 그 데이터를 해제하고 있었습니다. 반복 조회에서 `__slab_free → kfree → proc_release` 경로의 kernel BUG가 발생했습니다.

또한 타이머·spinlock 문맥에서 메모리 할당, mmap 잠금, proc 엔트리 제거처럼 잠들 수 있는 작업을 수행했습니다. `rmmod`에서는 `scheduling while atomic`이 발생했습니다. proc 콜백이 끝나기를 기다리면서 같은 잠금을 잡고 있을 수도 있었습니다.

task, mm, 부모 프로세스의 참조 수명을 보장하지 않았고, PID별 hash 엔트리도 중복해서 쌓였습니다.

## 수정

- 수집을 `delayed_work`로 옮기고 task/mm 참조를 확보한 뒤 접근
- PID별 자료를 갱신하고 종료한 PID 정리
- `open`마다 데이터와 조회 시각을 고정하고 `close`에서는 그 복사본만 해제
- work와 proc 콜백이 끝난 뒤 모듈 메모리 해제

메모리 조회는 일반 x86-64 PTE 매핑으로 제한했습니다. huge/devmap/비상주 페이지는 주소 0으로 표시합니다.

## 확인

수정본에서 부분 읽기·seek, 접근 권한, 동시 reader 4개의 400회 조회, 200회 프로세스 생성·종료, 열린 FD를 유지한 제거, 50회 load/read/unload를 확인했습니다. 이 실행에서 kernel BUG/Oops/WARN/panic은 발생하지 않았습니다.

KASAN과 lockdep 검사는 아직 하지 않았습니다. 당시 수업 VM과 커널 설정이 같지는 않습니다.

과제 소스는 수업 공개 제한으로 포함하지 않았습니다. 별도 구현은 [proc-lens](https://github.com/JOON0512/proc-lens)에서 볼 수 있습니다.
