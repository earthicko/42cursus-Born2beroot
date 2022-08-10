# 42cursus-Born2beroot

# 개요

- 본 프로젝트에서는 VirtualBox를 활용한다.
- Git 저장소의 최상위 경로에 가상 디스크의 시그니쳐를 `signature.txt`에 작성하여 제출한다.
  - 다음 디렉터리에 접근:
    - Windows: `%HOMEDRIVE%%HOMEPATH%\VirtualBox VMs\`
    - Linux: `~/VirtualBox VMs/`
    - MacOS: `~/VirtualBox VMs/`
  - `.vdi` 파일에서 sha1 형식의 시그니쳐를 추출:
    - Windows: `certUtil -hashfile ???.vdi sha1`
    - Linux: `sha1sum ???.vdi`
    - MacOS: `shasum ???.vdi`
  - `6e657c4619944be17df3c31faa030c25e43e40af`와 같은 형태의 시그니쳐 추출됨

# 동료 평가 지침

- `aptitude`와 `apt`의 차이점, SELinux, AppArmor가 무엇인지 등을 숙지한다.
- `SSH`를 통하여 새 계정을 생성하고 접속할 수 있다.
- hostname을 변경할 수 있다.
- 새 계정을 생성하고 그룹에 지정할 수 있다.
- 서버 접속 시 현시될 메시지 (`monitoring.sh`)의 원리를 숙지한다.
- Git 저장소에 제출된 시그니쳐와 평가 시 기기의 시그니쳐가 다를 경우 0점이 부여된다.
  - 평가 과정에서 기기의 시그니쳐가 변경될 수 있으므로 기기를 복제하거나 상태 저장 기능을 활용한다.

# 제출 사항

- 본 프로젝트에서는 X.org와 같은 그래픽 사용자 인터페이스의 사용을 금지한다.
- Debian 운영체제의 최신 안정판 버전을 설치한다. 
  - CentOS에서는 SELinux가, Debian에서는 AppArmor가 시스템 시동 시 실행되도록 해야 한다.
- LVM을 사용하여 최소 2개의 암호화된 파티션을 생성한다. 

예시
```
$>lsblk
NAME					MAJ:MIN	RM	TYPE	MOUNTPOINT
sda						8:0		0	disk
-sda1					8:1		0	part	/boot
-sda2					8:2		0	part
-sda5					8:5		0	part
  -sda5_crypt			254:0	0	crypt
    -wil--vg-root		254:1	0	lvm		/
	-wil--vg-swap_1		254:2	0	lvm		[SWAP]
	-wil--vg-home		254:3	0	lvm		/home
sr0						11:0	1	rom
```

## 네트웍

- `4242`번 포트에 SSH 서비스가 실행되어야 한다. `root` 계정으로는 SSH에 접속할 수 없도록 한다.
- UFW 방화벽을 설치하고 `4242`번 포트만 개방한다.
  - 가상 기기 시동 시 방화벽이 실행 및 활성화되어야 한다. DNF를 사용하여 방화벽을 설치한다.

## 사용자

- 가상 기기의 hostname은 `$(계정명)42`의 형태로 한다. 
- `root` 사용자와 이름이 `$(계정명)`인 사용자를 구비한다.
  - 강력한 암호 관리 정책을 수립한다.
    - 암호 만료 시한: 30일
    - 최소 암호 변경 주기: 2일
    - 암호 만료 경고일: 만료 7일 전
    - 10글자 이상, 대문자와 숫자 포함, 동일 연속 글자 3개 이하
    - 사용자명 포함 금지
    - (`root`에게는 제외) 이전 암호와 중복되지 않는 글자를 7개 이상 포함
  - 엄격한 규칙을 따라 `sudo`를 설치하고 설정한다.
    - `$(계정명)` 사용자는 `user42`와 `sudo` 그룹에 지정
    - `sudo`를 이용한 인증 시도는 3번까지 허용
    - `sudo` 인증 실패시 임의의 메시지 현시 가능
    - `sudo`를 이용한 모든 행동의 입출력을 `/var/log/sudo/`에 기록하도록 설정
    - TTY 모드를 활성화
    - `sudo`가 사용할 수 있는 경로를 제한
      - 예시: `/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin`
- configuration file을 만든 후 가상 기기에 있는 모든 사용자의 암호를 변경한다.

## 스크립트

- `monitoring.sh`이라는 스크립트를 `bash`로 작성한다.
  - 서버 시동 시 해당 정보를 10분마다 현시 (`wall` 참조)
  - 배너 현시는 선택 사항
  - 에러 현시는 금지
    - 운영 체제의 아키텍쳐와 커널 버전
    - 물리적 / 가상 프로세서의 개수
    - 현재 메모리 가용량과 사용 비율 (퍼센트로)
    - 현재 기억장치 가용량과 사용 비율 (퍼센트로)
    - 현재 프로세스 사용 비율 (퍼센트로)
    - 최근 재시동 일시
    - LVM의 활성화 여부
    - 활성 세션의 개수
    - 서버에 로그온 중인 사용자의 개수
    - IPv4 및 MAC 주소
    - `sudo`를 통해 실행된 명령어의 개수

예시
```
Broadcast message from root@wil (tty1) (Sun Apr 25 15:45:00 2021):

	#Architecture: Linux wil 4.19.0-16-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64 GNU/Linux
	#CPU physical : 1
	#vCPU : 1
	#Memory Usage: 74/987MB (7.50%)
	#Disk Usage: 1009/2Gb (39%)
	#CPU load: 6.7%
	#Last boot: 2021-04-25 14:45
	#LVM use: yes
	#Connections TCP : 1 ESTABLISHED
	#User log: 1
	#Network: IP 10.0.2.15 (08:00:27:51:9b:a5)
	#Sudo : 42 cmd
```

## 주 점검사항

다음 명령을 예상
```
#> head -n 2 /etc/os-release
PRETTY_NAME="Debian GNU/Linux 10 (buster)"
NAME="Debian GNU/Linux"
#> /usr/sbin/aa-status
apparmor module is loaded.
#> ss -tunlp
Netid	State		--- 생략

tcp		LISTEN		--- 생략
#> /usr/sbin/ufw status
Status: active

To			Action		From
--			------		----
4242		ALLOW		Anywhere
4242 (v6)	ALLOW		Anywhere (v6)
```
