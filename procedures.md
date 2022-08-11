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
- LVM: Logical Volume Manager
- TTY: Teletypewriter, 콘솔의 일종
- UFW: Uncomplicated Firewall

# 데이터 준비

https://www.debian.org/distrib/ 에서 완전한 설치 이미지, amd64 체계용 이미지 다운로드

# 가상 기기 생성

- 표준적 절차 및 기본 설정값에 의거하여 가상 기기 생성
- 저장소 - IDE 컨트롤러에 디스크 이미지 탑재

# 운영 체제 설치

1. 기기 시동
1. Install
1. 위치 및 로케일 설정
1. 계정 설정
    1. hostname: `donghyle42`, domain: 임의
    1. Full name for the new user: 임의, Username: `donghyle`
1. Partitioning
    1. Guided, encrypted LVM
    1. 따로 파티션 만들기 (`/home`)
    1. 볼륨 그룹을 전부 사용 (최대 용량)
1. 기본값으로 계속 진행
2. GRUB을 `/dev/sda1`에 설치(?)

---

# 운영 체제 설정

## 사용자 계정 설정

### 암호 관리 정책

#### 암호 시한

`/etc/login.defs` 파일을 편집한다.

- 암호 만료 시한 30일: `PASS_MAX_DAYS 30`
- 최소 암호 변경 주기 2일: `PASS_MIN_DAYS 2`
- 암호 만료 7일 전 경고: `PASS_WARN_AGE 7`

#### 암호 복잡도

`libpam-cracklib` 패키지를 `apt`를 통해 설치한다. 암호 및 인증 관련 파일은 `/etc/pam.d`에 저장된다. 암호 관리 정책은 `/etc/pam.d/common-password`에 정의되어 있다.
```
password	requisite	pam_cracklib.so retry=3
```
상기된 것과 비슷한 행을 찾은 후 다음 내용을 추가한다.

- root에게도 규칙을 적용함: `enforce_for_root`
- 최소 길이: `minlen=10`
- 대문자를 포함: `ucredit=-1`, 숫자를 포함: `dcredit=-1`
- 기타 문자군에 대하여 크레딧 비활성화: `lcredit=0`, `ocredit=0`
- 동일 연속 글자 3개 이하: `maxrepeat=3`
- 사용자명 포함 금지: `reject_username`
- 이전 암호와 중복되지 않는 글자 최소 7개: `difok=7`

```
enforce_for_root minlen=10 ucredit=-1 dcredit=-1 lcredit=0 ocredit=0 maxrepeat=3 reject_username difok=7
```

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

### sudo 설치 및 설정

#### 설치 및 사용자 추가

1. `apt`를 사용하여 `sudo`를 설치한다.
2. `groupadd user42`: `user42` 그룹을 생성한다.
3. `donghyle` 사용자를 `user42`와 `sudo` 그룹에 추가한다.
    - `usermod -aG user42,sudo donghyle`
        - `-a`: 추가, `-G`: 그룹명 지정
    - `usermod -g user42 donghyle`
        - 사용자의 초기 그룹을 변경 **(필수적인지 불확실)**

#### 규칙 설정

`visudo` 명령을 이용하여 `/etc/sudoers` 파일을 편집한다.

- 3번까지 인증 시도 허용: `Defaults	passwd_tries=3`
- 인증 실패 시 메시지 현시: `Defaults	authfail_message="메시지"`
- 암호 실패 시 메시지 현시: `Defaults	badpass_message="메시지"`
- `sudo`를 이용한 모든 행동의 입출력을 `/var/log/sudo/`에 기록하도록 설정:
  - `Defaults	log_input`
  - `Defaults	log_output`
  - `Defaults	iolog_dir="/var/log/sudo/"`
- TTY 모드를 활성화: `Defaults	requiretty`
- `sudo`가 사용할 수 있는 경로를 제한
  - `Defaults	secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"`

```
Defaults	passwd_tries=3
Defaults	authfail_message="메시지"
Defaults	badpass_message="메시지"
Defaults	log_input
Defaults	log_output
Defaults	iolog_dir="/var/log/sudo/"
Defaults	requiretty
Defaults	secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
```

- `mkdir /var/log/sudo`: 디렉토리가 없을 시 생성

**암호 및 sudo 관련 설정 후 모든 사용자의 암호를 변경한다.**

## 네트웍 설정

### SSH 서버

1. `apt`를 사용하여 `openssh-server`를 설치한다.
1. `/etc/ssh/sshd_config` 파일에서
    - `4242`번 포트에 실행: `Port 4242`
    - `root` 계정으로는 접속 불가: `PermitRootLogin no`
1. 필요시 서비스를 재시작: `systemctl restart ssh`


### 포트포워딩

1. `hostname -I`로 내부 IP를 확인 (예: `10.0.2.15`)
1. 가상 기기를 완전히 정지 및 종료
2. VirtualBox의 메뉴에서 Tools - Network 진입
   1. Create로 새로운 네트웍 생성
3. 가상 기기의 설정 창에서 네트웍 메뉴 진입
   1. 포트포워딩 대화 상자에서 Host IP에 네트웍 IP를, Guest IP에 내부 IP 입력
   2. 어댑터 2를 Host-only Adapter로 변경 후 3-1에서 생성한 네트웍 지정
4. 호스트에서 3-1에서 생성한 IP로 ssh 접속

### 방화벽

1. `apt`를 사용하여 `ufw`를 설치한다.
1. UFW 활성화 및 시동 시 실행: `ufw enable`
1. 기본적으로 모든 접속 차단: `ufw default deny`
1. 4242번 포트 허용: `ufw allow 4242`

## 시동 메시지 설정

### 필요 유틸리티

`apt`를 사용하며 `sysstat`을 설치한다.

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
  - `grep sudo | wc -l`로 메시지 자체의 개수만 추출

`vi /root/monitoring.sh`
```bash
#!/usr/bin/bash
ARCHI=$(uname -a | tr -d '\n')
N_CPU_P=$(nproc --all | tr -d '\n')
N_CPU_V=$(cat /proc/cpuinfo | grep processor | wc -l | tr -d '\n')
MEM_USED=$(free -m | grep Mem | awk '{print $3}' | tr -d '\n')
MEM_TOTAL=$(free -m | grep Mem | awk '{print $2}' | tr -d '\n')
MEM_PCENT=$(echo "${MEM_USED} ${MEM_TOTAL}" | awk '{printf"%.2f", 100 * $1 / $2}' | tr -d '\n')
DISK_USED=$(df -a | grep /dev/map | awk '{sum += $3} END {print int(sum / 1024)}' | tr -d '\n')
DISK_AVAIL=$(df -a | grep /dev/map | awk '{sum += $4} END {print int(sum / 1024)}' | tr -d '\n')
DISK_TOTAL=$(echo "${DISK_USED} ${DISK_AVAIL}" | awk '{print $1 + $2}' | tr -d '\n')
DISK_PCENT=$(echo "${DISK_USED} ${DISK_TOTAL}" | awk '{printf"%.2f", 100 * $1 / $2}' | tr -d '\n')
CPU_PCENT=$(mpstat | grep all | awk '{printf "%.2f", 100 - $13}' | tr -d '\n')
TIME_LAST_BOOT=$(who -b | awk '{printf $3" "$4"\n"}' | tr -d '\n')
if [ "$(lsblk | grep lvm | wc -l)" -eq 0 ] ; then USAGE_LVM="no" ; else USAGE_LVM="yes" ; fi
N_TCP_CON=$(ss | grep -i tcp | wc -l | tr -d '\n')
N_USER_CON=$(who | wc -l | tr -d '\n')
ADDR_IP=$(hostname -I | tr -d '\n' | tr -d ' ')
ADDR_MAC=$(ip link show | awk '$1 == "link/ether" {print $2}' | head -n 1 | tr -d '\n')
N_SUDO=$(journalctl _COMM=sudo | grep COMMAND | wc -l | tr -d '\n')

echo "#Architecture: ${ARCHI}"
echo "#CPU physical : ${N_CPU_P}"
echo "#vCPU : ${N_CPU_V}"
echo "#Memory Usage: ${MEM_USED}/${MEM_TOTAL}MB (${MEM_PCENT}%)"
echo "#Disk Usage: ${DISK_USED}/${DISK_TOTAL}MB (${DISK_PCENT}%)"
echo "#CPU load: ${CPU_PCENT}%"
echo "#Last boot: ${TIME_LAST_BOOT}"
echo "#LVM use: ${USAGE_LVM}"
echo "#Connections TCP : ${N_TCP_CON} ESTABLISHED"
echo "#User log: ${N_USER_CON}"
echo "#Network: IP ${ADDR_IP} (${ADDR_MAC})"
echo "#Sudo : ${N_SUDO} cmd"
```

### 정기 작업 등록

- `chmod +x /root/monitoring.sh`
- `crontab -e` (`root`로서): `cron` 유틸리티의 `root`의 table을 조작, `-e`: 편집기 실행
- 파일에 `*/10 * * * * /root/monitoring.sh | wall` 추가
  - 분, 시, 일, 월, 요일, 명령 순서
  - `*/10`: 매 10분마다, `*`: 매번

# 작업 후 가상 기기 조작

1. 가상 기기의 상태 저장
1. 가상 기기의 스냅샷 찍기
1. `shasum ~/VirtualBox VMs/$(가상 기기 이름).vdi` 실행하여 시그니쳐 확보
