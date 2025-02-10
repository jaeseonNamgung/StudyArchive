# systemd 란
- `systemd`는 리눅스에서 부팅 프로세스 및 시스템 서비스를 관리하는 init 시스템이다.
- 우리가 흔히 쓰는 `systemctl` 명령어가 바로 `systemd`를 이용한 롼리 도구이다.

# systemd의 역할

1. 부팅 시 서비스 관리(init 시스템)
- 시스템이 부팅될 때 필요한 모든 서비스(예: 네트워크, SSH, Docker 등)를 자동으로 실행
- `systemctl start` / `systemctl stop` 명령어로 개별 서비스 관리 가능
1. 프로세스 모니터링 및 재시작
- 서비스가 비정상 적으로 종료되면 자동으로 재시작

```bash
sudo systemctl restart kubelet
```

1. 로그 관리 (journalctl)
- `journalctl` 명령어로 시스템 관리 확인 가능

```bash
sudo journalctl -u kubelet -f
```

## systemd 주요 명령어

- 서비스 시작: `systemctl start <서비스 명>`
- 서비스 중지: `systemctl stop <서비스 명>`
- 서비스 재시작: `systemctl restart <서비스 명>`
- 서비스 상태 확인: `systemctl status <서비스 명>`
- 부팅 시 자동 실행 설정: `systemctl enable <서비스 명>`
- 부팅 시 자동 실행 해제: `systemctl disable <서비스 명>`
