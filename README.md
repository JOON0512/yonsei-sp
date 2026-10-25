# Linux kernel debugging notes

시스템프로그래밍 수업에서 만든 `/proc` 모니터가 반복 조회와 모듈 제거 중 죽는 문제를 정리했습니다.

[디버깅 노트](debugging.md)에서는 공유 데이터의 중복 해제, 잠들 수 없는 문맥에서의 메모리 할당, task/mm 참조 수명 문제를 다룹니다. 재현 환경은 Linux 6.8 / QEMU입니다.

과제 코드는 수업 공개 제한으로 포함하지 않았습니다. 같은 주제를 별도로 구현한 프로세스 모니터는 [proc-lens](https://github.com/JOON0512/proc-lens)에 있습니다.
