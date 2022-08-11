# 개념 숙지

- Debian: 리눅스 커널을 쓰기 좋게 여러 소프트웨어와 함께 묶어 배포하는 운영 체제
- sda 및 파티션
  - SCSI Drive [a-z] = sda (물리)
    - sda1, sda2, ... = 파티션
  - 왜 부트로더를 `/dev/sda`에 설치해도 `/dev/sda1`에 설치되는가
- AppArmor: 시스템 관리자가 프로그램 프로필 별로 프로그램의 역량을 제한할 수 있게 해주는 리눅스 커널 보안 모듈
  - 강제적 접근 통제(MAC)
  - 네트워크 액세스
  - raw 소켓 액세스
  - 파일의 읽기, 쓰기, 실행
- `apt`: 로우 레벨 패키지 매니저
- `aptitude`: 하이 레벨에서 `apt`를 사용하여 패키지를 좀 더 지능적으로 관리해주는 유틸리티 (충돌 해결 등 더 복잡한 기능 보유)
- LVM: Logical Volume Manager
- TTY: Teletypewriter, 콘솔의 일종
- UFW: Uncomplicated Firewall

# 운영 체제

- hostname 변경: `hostnamectl set-hostname ${NEWNAME}`
- 파티션 정보 확인: `lsblk`
- `sysstat` 패키지 설치 이유: `/proc/cpuinfo`를 통해 논리 프로세서의 개수를 파악하기 위함

## 정기 메시지

- `uname`: 시스템 정보 출력, `-a`: 모든 정보 출력
- `nproc`: 물리 프로세서 개수 출력, `--all`: 모든 프로세서 산입
- `/proc/cpuinfo`: 논리 프로세서 정보
    - `grep`으로 `processor n`행만 추출 후 wc -l로 줄 개수를 산입
- `free`: 메모리 사용량 출력, `-m`: 메비바이트 단위 출력
  - `grep Mem`으로 `swap`을 제외한 RAM 정보 행만 추출
  - `awk '{printf"%d/%dMB (%.2f%%)\n", $3, $2, $3/$2 * 100}'`
    - 2번 필드: 총 메모리
    - 3번 필드: 사용된 메모리
- `df`: 디스크 사용량 출력, `-a`: 정규 파일이 아닌 것까지 모두 포함
  - **(검증 필요)** `grep /dev/mapper`로 실제 사용중인 파일 시스템 행만 추출
  - `awk '{sum += $3} END {print int(sum / 1024)}'`
    - 모든 행의 3번째 (사용중) 필드의 값을 sum 변수에 더한 뒤 1024로 나눈 값을 정수로 출력
    - 4번째 필드는 사용 가능한 바이트, 즉 $3 + $4 = 총 바이트
  - `tr -d '\n'`과 조합하면 개행 없이 다음 문자 출력 가능
- `mpstat`: 모든 프로세서의 활성 상태 출력
  - `grep all`로 총 사용량 행 추출
  - `awk '{printf "%.2f%%\n", 100-$13}'`
    - 13번 필드가 사용 가능량이므로 100에서 뺄셈함
- `who`: 로그온한 사용자 정보 표시, `-b`: 마지막 시동 시각
  - `awk '{printf $3" "$4"\n"}'`
    - 3번 필드, 공백, 4번 필드, 개행 문자를 차례로 출력
- `lsblk`: 블록 장치 (파티션)의 목록 표시
  - `grep lvm`으로 `TYPE`에 `lvm`이 지정된 행 추출 후 `wc -l`로 행의 개수 출력
- `ss`: 소켓 상태 정보 표시
  - `grep -i tcp | wc -l`으로 대소문자 구분 없이 `tcp`가 포함된 행 개수 파악하여 네트웍 접속 정보만 표시
- `who`: 로그온한 사용자 정보 표시
  - `wc -l`으로 사용자 행 개수 추출
- `hostname`: 시스템의 `hostname` 표시, `-I`: 모든 네트웍 주소 표시
- `ip`: ip 관련 명령 모음, `link`: 네트웍 장비에 대하여, `show`: 정보 표시
  - `awk '$1 == "link/ether" {print $2}'`
    - 1번 필드가 일치할 경우 2번 필드 출력
  - `head -n 1`로 첫째 줄만 추출 (`link/ether`가 여러 개일 경우)
- `journalctl`: `systemd`가 작성한 일지를 조회, `_COMM=sudo`: `sudo`가 작성한 일지 조회
  - `grep COMMAND | wc -l`로 메시지 자체의 개수만 추출

### 정기 작업

- `crontab -e` (`root`로서): `cron` 유틸리티의 `root`의 table을 조작, `-e`: 편집기 실행
- 파일에 `*/10 * * * * /root/monitoring.sh | wall` 추가
  - 분, 시, 일, 월, 요일, 명령 순서
  - `*/10`: 매 10분마다, `*`: 매번
  - `sleep30;`을 앞에 추가하여 30초 유예를 부여 가능

# 사용자 계정

## 사용자 및 그룹 생성 및 조작

- 새 사용자 추가: `useradd ${USER}`
- 암호 변경: `passwd ${USER}`
- 새 그룹 생성: `groupadd ${GROUP}`
- 사용자를 그룹에 추가: `usermod -aG ${GROUP} ${USER}`

## 암호 정책

암호 시한 관련 정책은 `/etc/login.defs`에 정의됨

- 암호 만료 시한: `PASS_MAX_DAYS`
- 최소 암호 변경 주기: `PASS_MIN_DAYS`
- 암호 만료 전 경고일: `PASS_WARN_AGE`

- `chage -l ${USER}`: 현 정책 확인
- `chage -M ${N} ${USER}`: 암호 만료 시한 N일
- `chage -m ${N} ${USER}`: 최소 암호 변경 주기 N일
- `chage -W ${N} ${USER}`: 암호 만료 N일 전 경고

`libpam-cracklib`를 설치하는 이유: 

암호 복잡도 관련 정책은 `/etc/pam.d/common-password`에 정의됨

- root에게도 규칙을 적용함: `enforce_for_root`
- 최소 길이: `minlen`
- 대문자 크레딧: `ucredit`
- 소문자 크레딧: `lcredit`
- 숫자 크레딧: `dcredit`
- 기타 문자군 크레딧: `ocredit`
- 동일 연속 글자 제한: `maxrepeat`
- 사용자명 정방향/역방향 포함 금지: `reject_username`
- 이전 암호와 중복되는 글자 허용량: `difok`

> 1. The following rule does not apply to the root password: 
>     1. The password must have at least 7 characters that are not part of the former password.
> 1. Of course, your root password has to comply with this policy.
> 
> 2항을 단서로 하여 `/etc/pam.d/common-password`에 `enforce_for_root`을 추가한다. 다만 `root`로서 `passwd`를 실행할 경우 직전 암호를 요구하지 않기 때문에 1-1항은 `root`에게는 적용되는 것이 불가능하다. 따라서 1항을 만족시키기 위해서 별도의 조치가 필요하지 않고, 불만족시키는 것이 불가능하다.

> 암호에 사용되는 문자는 소문자, 대문자, 숫자, 기타 문자로 구분된다. 이때 각 군의 문자가 암호에 포함되어 있을 시 그에 대한 크레딧을 획득하고, 암호의 물리적 길이에 크레딧을 합산하여 총 길이를 산출하게 된다. 이때 각 군별 획득할 수 있는 크레딧의 최대값을 설정할 수 있다.
> 
> `lcredit=3`, `dcredit=1`, `ocredit=1`이면 암호 `abcdefg01!`은 15자로 판정한다.
> 
> - 물리적 길이 10
> - 소문자를 7개 사용했으나 `lcredit=3`으로 제한하였으므로 3자 가산
> - 숫자를 2개 사용했으나 `dcredit=1`으로 제한하여 1자 가산
> - 기타 문자를 1개 사용하였고 `ocredit=1`이므로 1자 가산
> 
> 따라서 이를테면 `minlen=12`으로 제한하였더라도 그 물리적 길이에도 불구하고 해당 암호를 사용할 수 있다.
> 
> 이때 크레딧을 음수로 설정하면 가산점은 없어지고 해당 군의 문자를 N개 사용하는 것이 강제된다. 크레딧이 0일 경우 해당 군의 문자를 사용하거나 사용하지 않아도 전체 복잡도에 영향이 없다.

## sudo

- sudo 관련 정책 편집: `visudo`

# 네트웍

- UFW 상태 확인: `ufw status verbose`
- 내부 IP 확인: `hostname -I`

## SSH 서버

`/etc/ssh/sshd_config`을 편집
- SSH 서버 상태 확인: `systemctl status ssh`
- SSH 서버 재시작: `systemctl restart ssh`
