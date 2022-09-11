# 프로젝트 수행 지침

# 데이터 준비

https://www.debian.org/distrib/ 에서 완전한 설치 이미지, amd64 체계용 이미지 다운로드

# 가상 기기 생성

- 표준적 절차 및 기본 설정값에 의거하여 가상 기기 생성
- 저장소 - IDE 컨트롤러에 디스크 이미지 탑재

# 운영 체제 설치

1. 기기 시동
2. Install
3. 위치 및 로케일 설정
4. 계정 설정
    1. hostname: `donghyle42`, domain: 임의
    2. Full name for the new user: 임의, Username: `donghyle`
5. Partitioning
    1. Manual
    2. SCSI3 (0,0,0) (주 드라이브) 선택
    3. pri/log 빈 공간에 파티션 2회 생성
       1. Create a new partition, 500M, Primary, Beginning, Mount point: `/boot`
       2. Create a new partition, max, Logical, Mount point: none (Do not mount it)
    4. Configure encrypted volumes
       1. Create encrypted volumes
       2. `/dev/sda5` 선택하여 진행
       3. passphrase 설정
    5. Configure the Logical Volume Manager
       1. Create volume group
          1. Volume group name: LVMGroup
          2. `/dev/mapper/sda5_crypt` 선택하여 진행
       2. Create logical volume 2회
          1. name: root, size: 2G
          2. name: swap, size: 1G
          3. name: home, size: 1G
          4. name: var, size: 1G
          5. name: srv, size: 1G
          6. name: tmp, size: 1G
          7. name: var-log, size: 남은 것 전부 (기본값)
    6. Partition disks 메뉴에서 각 볼륨 설정
       1. home: Use as Ext4, mount point `/home`
       2. root: Use as Ext4, mount point `/`
       3. srv: Use as Ext4, mount point `/srv`
       4. swap: Use as swap area
       5. tmp: Use as Ext4, mount point `/tmp`
       6. var: Use as Ext4, mount point `/var`
       7. var-log: Use as Ext4, mount point `/var/log`
    7. Finish Partitioning
6. Software selection: 모두 제외

---

# 운영 체제 설정

## sudo 설치 및 설정

### 설치 및 사용자 추가

1. `apt`를 사용하여 `sudo`를 설치한다.
2. `groupadd user42`: `user42` 그룹을 생성한다.
3. `donghyle` 사용자를 `user42`와 `sudo` 그룹에 추가한다.
    - `usermod -aG user42,sudo donghyle`
        - `-a`: 추가, `-G`: 그룹명 지정
    - `usermod -g user42 donghyle`
        - 사용자의 초기 그룹을 변경 **(필수적인지 불확실)**

### 규칙 설정

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

## 네트웍 설정

### SSH 서버

1. `apt`를 사용하여 `openssh-server`를 설치한다.
1. `/etc/ssh/sshd_config` 파일에서
    - `4242`번 포트에 실행: `Port 4242`
    - `root` 계정으로는 접속 불가: `PermitRootLogin no`
1. 필요시 서비스를 재시작: `systemctl restart ssh`


### 포트포워딩

1. `hostname -I`로 내부 IP를 확인 (예: `10.0.2.15`)
1. **가상 기기를 완전히 정지 및 종료**
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

## 사용자 계정 설정

### 암호 관리 정책

#### 암호 시한

`/etc/login.defs` 파일을 편집한다.

```
sudo nano /etc/login.defs
```

- 암호 만료 시한 30일: `PASS_MAX_DAYS 30`
- 최소 암호 변경 주기 2일: `PASS_MIN_DAYS 2`
- 암호 만료 7일 전 경고: `PASS_WARN_AGE 7`

암호 시한 정책은 정책 시행 이후 생성된 계정에만 적용되기 때문에 `chage` 명령으로 기존 계정의 정책을 변경한다.

- `chage -l ${USER}`: 현 정책 확인
- `chage -M 30 ${USER}`: 암호 만료 시한 30일
- `chage -m 2 ${USER}`: 최소 암호 변경 주기 2일
- `chage -W 7 ${USER}`: 암호 만료 7일 전 경고

```
sudo chage -M 30 root && sudo chage -m 2 root && sudo chage -W 7 root && sudo chage -M 30 donghyle && sudo chage -m 2 donghyle && sudo chage -W 7 donghyle
```

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
- 사용자명 정방향/역방향 포함 금지: `reject_username`
- 이전 암호와 중복되지 않는 글자 최소 7개: `difok=7`

```
sudo nano /etc/pam.d/common-password
enforce_for_root minlen=10 ucredit=-1 dcredit=-1 lcredit=0 ocredit=0 maxrepeat=3 reject_username difok=7
```

**암호 및 sudo 관련 설정 후 모든 사용자의 암호를 변경한다.**

```
sudo passwd
```

## 정기 메시지 설정

### 필요 유틸리티

`apt`를 사용하며 `sysstat`을 설치한다.

```
sudo apt install sysstat -y
sudo nano /root/monitoring.sh
sudo chmod +x /root/monitoring.sh
```

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

- `crontab -e` (`root`로서): `cron` 유틸리티의 `root`의 table을 조작, `-e`: 편집기 실행
- 파일에 `*/10 * * * * /root/monitoring.sh | wall` 추가
  - 분, 시, 일, 월, 요일, 명령 순서
  - `*/10`: 매 10분마다, `*`: 매번
  - `sleep30;`을 앞에 추가하여 30초 유예를 부여 가능

# 서비스 설치

## Lighttpd, MariaDB, PHP 설치

### Lighttpd 설치

1. `apt`를 통해 `lighttpd`를 설치한다.
2. 80번 포트를 통한 연결을 허가한다.

```
sudo apt install lighttpd -y && sudo ufw allow 80
```

### MariaDB 설치

1. `apt`를 통해 `mariadb-server`를 설치한다.
2. `mysql_secure_installation`을 실행하여 MariaDB의 초기 설정을 변경한다.
   1. 초기 root 암호는 없다 (운영 체제의 root 사용자와 다름) `Enter`
   2. unix_socket을 사용하지 않는다. `n`
   3. root 암호를 설정하지 않는다. `n`
   4. 익명 사용자를 제거한다. `Y`
   5. 원격 root 접속을 불허한다. `Y`
   6. 테스트 DB를 제거한다. `Y`
   7. 권한 테이블을 다시 로드한다. `Y`
3. `mariadb`를 실행하여 MariaDB에 접속하고 `exit`로 종료한다.

```
sudo apt install mariadb-server -y
sudo mysql_secure_installation
```
```
sudo mariadb
CREATE DATABASE wordpress_db
GRANT ALL ON wordpress_db.* TO 'wordpress_user'@'localhost' IDENTIFIED BY 'wordpresspass42' WITH GRANT OPTION;
FLUSH PRIVILEGES;
exit
```
```
mariadb -u wordpress_user -p
SHOW DATABASES;
exit
```

### PHP 설치

1. `apt`를 통해 `php-cgi`, `php-mysql`와 `wget`를 설치한다.
2. `http://wordpress.org/latest.tar.gz`를 `/var/www/html`에 다운로드한다.
3. 위 파일의 압축을 해제하고 원본을 삭제한다.
4. 압축을 해제한 내용을 `/var/www/html`에 옮기고 원본을 삭제한다.
5. 샘플 설정 파일 `/var/www/html/wp-config-sample.php`을 주 설정 파일 `/var/www/html/wp-config.php`로 설정한다.
6. 위 파일을 편집하여 MariaDB를 참조하도록 설정한다.
   1. `database_name_here`을 `wordpress_db`로
   2. `username_here`을 `wordpress_user`로
   3. `password_here`을 `wordpresspass42`로

```
sudo apt install php-cgi php-mysql wget -y && sudo wget http://wordpress.org/latest.tar.gz -P /var/www/html && sudo tar -xzvf /var/www/html/latest.tar.gz && sudo rm /var/www/html/latest.tar.gz && sudo cp -r /var/www/html/wordpress/* /var/www/html && sudo rm -rf /var/www/html/wordpress && sudo cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
sudo vi /var/www/html/wp-config.php
```

### 필요 모듈 활성화

- `fastcgi`, `fastcgi-php` 모듈을 활성화한다.

```
sudo lighty-enable-mod fastcgi && sudo lighty-enable-mod fastcgi-php && sudo service lighttpd force-reload
```

## FTP 설치

1. `apt`를 통해 `vsftpd`를 설치한다.
2. 21번 포트를 통한 연결을 허가한다.
3. `/etc/vsftpd.conf`를 편집한다.
   1. `#write_enable=YES`의 주석을 해제한다.
   2. `#chroot_local_user=YES`의 주석을 해제한다.
4. FTP로 접속한 자가 사용할 디렉토리를 생성한다.
   1. `/home/donghyle/ftp` 디렉토리를 생성한다.
   2. `/home/donghyle/ftp/files` 디렉토리를 생성한다.
   3. 위 디렉토리의 소유권을 nobody:nogroup에 할당한다.
   4. 위 디렉토리에 쓰기 권한을 모든 유저에게서 없앤다.

다음 줄을 추가한다.
```
user_sub_token=$USER
local_root=/home/$USER/ftp
```

1. `/etc/vsftpd.userlist`에 `donghyle42`를 추가한다.

다음 줄을 추가한다.
```
userlist_enable=YES
userlist_file=/etc/vsftpd.userlist
userlist_deny=NO
```

# 작업 후 가상 기기 조작

1. 가상 기기의 상태 저장
1. 가상 기기의 스냅샷 찍기
1. `shasum ~/VirtualBox VMs/$(가상 기기 이름).vdi` 실행하여 시그니쳐 확보
