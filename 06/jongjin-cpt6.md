## Table of Contents

- [장치 접근](#장치-접근)
  - [디바이스 파일](#디바이스-파일)
    - [캐릭터 장치](#캐릭터-장치)
    - [블록 장치](#블록-장치)
      - [루프 장치](#루프-장치)
  - [디바이스 드라이버](#디바이스-드라이버)
    - [메모리맵 입출력(MMIO)](#메모리맵-입출력mmio)
    - [폴링](#폴링)
    - [인터럽트](#인터럽트)
      - [일부러 폴링을 사용하는 경우](#일부러-폴링을-사용하는-경우)
  - [디바이스 파일명은 바뀌기 마련](#디바이스-파일명은-바뀌기-마련)

---

# 장치 접근
프로세스가 디바이스 파일을 사용해 물리 장치에 접근하는 방법을 설명

- 프로세스는 장치에 직접 접근 불가
  - 다수 프로그램이 동시 장치 조작 시, 예상못하는 방식으로 작동할 가능성
  - 접근해서는 안되는 데이터 조회 및 훼손 가능성

- 프로세스 대신, 커널이 장치에 접근
  - 디바이스 파일 이라는 특수 파일 매개
  - 블록 장치에 구축한 파일시스템 조작 (>7장)
  - NIC은 속도등의 문제로 인해 디바이스 파일 대신 소켓 구조 사용
    - 네트워크 분야라 디테일 설명 X

## 디바이스 파일
> 리눅스에서 드라이버 파일은 커널이 하드웨어 장치를 제어할 수 있게 해주는 모듈 또는 코드로, /dev 아래의 디바이스 파일과 연결되어 장치와 사용자 공간 간의 인터페이스를 제공합니다.

![DDD](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbfsZNd%2FbtqvZYequdx%2FyUiCn9pUymPLaJ1wn5Knm1%2Fimg.png)
[디바이스 드라이버 개요, 디바이스 드라이버 종류](https://butter-shower.tistory.com/29)
- 지금까지 배웠던건 유저스페이스 <> 커널스페이스 <> 하드웨어
- 그사이에 (장치에 따라) 디바이스 드라이버 추가

디바이스 드라이버(device driver)
- 장치마다 존재함
  - 예를들어 저장 장치라면 /dev/sda, /dev/sdb와 같은 방식
- 프로세스는 일반 파일과 같은 방식으로 디바이스 파일 조작
  - 시스템콜 사용) ex. open(), read(), write(), ioctl()
  - 디바이스 파일에 접근 가능한것은 보통 루트뿐
- 디바이스 파일에 저장되는 정보
  - 파일종류: 캐릭터장치(character device) 또는 블록장치(block device)
  - 디바이스 메이저번호, 마이너 번호: major,minor 번호 조합이 동일하다면 동일장치, 다르면 다른 장치
- 보통 /dev/ 디렉토리 하위에 존재
  - c라면 캐릭터 장치, b라면 블록 장치
  - 5번째 필드가 메이저번호, 6번째 필드가 마이너 번호
  - /dev/loop*는 블록 장치, /dev/tty는 캐릭터 장치

```
ls -l /dev/
...
crw-rw---- 1 root disk     10, 237 Apr 20 23:30 loop-control
brw-rw---- 1 root disk      7,   0 Apr 20 23:30 loop0
brw-rw---- 1 root disk      7,   1 Apr 20 23:30 loop1
...
crw-rw-rw- 1 root tty       5,   0 Apr 20 23:30 tty
crw--w---- 1 root tty       4,   0 Apr 20 23:30 tty0
crw--w---- 1 root tty       4,   1 Apr 20 23:30 tty1
crw--w---- 1 root tty       4,  10 Apr 20 23:30 tty10
crw--w---- 1 root tty       4,  11 Apr 20 23:30 tty1
```


### 캐릭터 장치
> 리눅스에서 캐릭터 장치는 데이터를 바이트 단위로 순차적 입출력하는 장치로, 예를 들어 터미널, 키보드, 마우스 등이 이에 해당합니다.  

[문자 특수 장치(character special file) 또는 문자 장치(character device)는 버퍼링되지 않은, 직접 접근을 하드웨어 장치에 제공한다.](https://ko.wikipedia.org/wiki/%EC%9E%A5%EC%B9%98_%ED%8C%8C%EC%9D%BC#%EB%AC%B8%EC%9E%90_%EC%9E%A5%EC%B9%98)

캐릭터장치(character device) > 읽고 쓰기는 가능, 장치 내부의 접근 장소 변경하는 탐색(seek) 불가
- 대표적인 캐릭터 장치
  - 단말
  - 키보드
  - 마우스
- 단말의 디바이스파일의 역할(조작)
  - write() 시스템콜 > 단말에 데이터 출력
  - read() 시스템콜 > 단말에서 데이터 입력
- 디바이스 파일에 접근해서 현재 단말 장치를 조작하기
  1. 현재 프로세스에 대응하는 단말 및 대응하는 디바이스 파일 찾기
    - ps ax의 두번째 필드, pts/4 사용 중
  2. pts/4 디바이스 파일에 적당한 문자열 입력
    - 단말 장치에 문자열을 입력하면 write() 시스템 콜 호출 > 단말에 문자열 출력됌
    - = echo hello와 동일한 결과
    - echo 명령어는 stdout에 hello를 쓰고, 리눅스에서 stdout은 터미널과 연결되어있기 때문
- 디바이스 파일에 접근해서 다른 단말 장치를 조작하기
  1. 다른 단말을 기동한뒤 디바이스 파일 찾기
  2. 다른 단말의 디바이스 파일에 문자열 입력 > 해당 단말에서 문자열 출력
```
❯ echo $SHELL
/usr/bin/zsh
❯ ps ax | grep zsh
    943 pts/1    S+     0:00 -zsh
   1209 pts/1    S      0:00 -zsh
   1210 pts/1    S      0:00 -zsh
   1213 pts/1    S      0:00 -zsh
   1324 pts/4    Ss     0:01 /usr/bin/zsh -i
   1521 pts/4    S      0:00 /usr/bin/zsh -i
   1522 pts/4    S      0:00 /usr/bin/zsh -i
   1525 pts/4    S      0:00 /usr/bin/zsh -i
   4279 pts/4    S+     0:00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn --exclude-dir=.idea --exclude-dir=.tox --exclude-dir=.venv --exclude-dir=venv zsh

echo hello | sudo tee /dev/pts/4
또는
sudo su
echo hello > /dev/pts/4  
```

### 블록 장치
> 리눅스에서 블록 장치는 데이터를 블록 단위로 임의 접근할 수 있는 장치로, 하드디스크, SSD, USB 드라이브 등이 이에 해당합니다.

[블록 특수 파일(block special file) 또는 블록 장치(block device)는 버퍼링된 접근을 하드웨어 장치에 제공하며, 이들의 세부 사항에 따라 어느 정도의 추상화를 제공한다](https://ko.wikipedia.org/wiki/%EC%9E%A5%EC%B9%98_%ED%8C%8C%EC%9D%BC#%EB%B8%94%EB%A1%9D_%EC%9E%A5%EC%B9%98)

블록장치(block device)는 파일 읽기/쓰기 뿐만 아니라 탐색도 가능!
- 대표적인 블록 장치
  - 저장장치(HDD, SSD)
- 블록 장치에 데이터 읽고쓰면 > 일반 파일처럼 저장 장치 특정 위치의 데이터 접근 가능
- 블록 디바이스 파일을 사용하여 블록 장치 조작
  - 일반적으로는 파일시스템을 통해서 데이터 읽고쓰기 / 일반적이지 않은 use case
  - 블록디바이스 파일의 ext4 파일시스템 내용을 fs 안거치고 블록디바이스파일을 통한 직접 변경
  1. 적당히 비어있는 파티션 찾기
    - 없으면 루프장치 컬럼을 참조하여 루프장치 사용
    - 기존 데이터가 있는 파티션 대상이면 데이터 손상 위험성 O
  2. 비어있는 파티션에 ext4 파일시스템 생성 
  3. 생성된 파일시스템을 마운트한 뒤, testfile 파일명의 hello world 문자열 기록

이거 빈파티션 없어서 못하는데> 루프장치로

#### 루프 장치
루프 장치는 리눅스에서 일반 파일을 마치 블록 장치처럼 마운트할 수 있게 해주는 가상 장치.
- fallocate > 빈파일을 빠르게 생성할때 사용하는 명령어
- losetup > loopback 디바이스와 파일을 연결
  - 디스크 이미지 파일을 마치 실제 블록 장치처럼 사용할 수 있게 해줌
```
❯ fallocate -l 1G loopdevice.img
❯ sudo losetup -f loopdevice.img
❯ losetup -l
NAME SIZELIMIT OFFSET AUTOCLEAR RO BACK-FILE                                                                               DIO LOG-SEC
/dev/loop1
             0      0         1  1 /mnt/docker-desktop-disk/isocache/entries/docker-desktop.iso/a9f8cc7082f33c3c8efdc39f0fcc7f7cfc5dc0711601f3ead58063717971fdb0
                                                                                                                             0     512
/dev/loop2
             0      0         0  0 /home/nasir/2025-linux-for-beginner/06/loopdevice.img                                     0     512
/dev/loop0
             0      0         1  1 /mnt/docker-desktop-disk/isocache/entries/docker-wsl-cli.iso/20df88aaf6922239590204cfbb43807f8a73845c1a7e9e7f582990f5ced90e33

❯ sudo mkfs.ext4 /dev/loop2
mke2fs 1.45.5 (07-Jan-2020)
Discarding device blocks: done                            
Creating filesystem with 262144 4k blocks and 65536 inodes
Filesystem UUID: 8234e8c8-570c-4630-a81d-7bba2cd60a28
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done

❯ mkdir mnt
❯ sudo mount /dev/loop0 mnt
mount: /home/nasir/2025-linux-for-beginner/06/mnt: WARNING: device write-protected, mounted read-only.
❯ sudo mount /dev/loop2 mnt
❯ mount
...
/dev/loop2 on /home/nasir/2025-linux-for-beginner/06/mnt type ext4 (rw,relatime)

```                       



## 디바이스 드라이버
> 디바이스 드라이버는 리눅스 커널이 하드웨어 장치와 직접 통신할 수 있도록 중개하는 소프트웨어 계층입니다.

디바이스 드라이버device driver > 프로세스가 디바이스 파일에 접근할때 동작
- 장치 직접 조작시 각 장치에 내장된 레지스터 영역 읽고 쓰기 필요
- 구체적인 레지스터 종류 및 조작법 정보는 각 장치 사양에 따라 달라짐
- 디바이스 레지스터는 CPU 레지스터와 이름은 같지만 다른 동작을 함
- 프로세스 입장에서 보는 장치 조작
  1. 프로세스가 디바이스 파일을 사용해 디바이스 드라이버에 장치 조작 요청
  2. CPU의 커널 모드 전환, 디바이스 드라이버가 레지스터를 사용하여 장치에 요청 전달
  3. 장치가 요청에 따라 처리
  4. 디바이스 드라이버가 장치의 처리 완료 확인 및 결과 수신
  5. CPU의 사용자 모드 복귀, 프로세스가 디바이스 드라이버 처리 완료 확인

![dd](https://docs.oracle.com/cd/E19120-01/open.solaris/819-3159/images/driver.overview.gif)    
[Device Drivers](https://docs.oracle.com/cd/E19120-01/open.solaris/819-3159/emjjs/index.html)


### 메모리맵 입출력(MMIO)
> 메모리맵 입출력(MMIO)은 하드웨어 장치의 레지스터를 메모리 주소 공간에 매핑해 CPU가 메모리 접근 방식으로 제어하는 방식입니다.

메모리맵입출력(memory-mapped I/O,MMIO): 현대적 장치는 MMIO 구조를 사용하여 디바이스 레지스터에 접근
- x86_64 아키는 리눅스 커널이 자신의 모든 가상 주소 공간에 물리메모리를 매핑 
- MMIO를 통해 장치를 조작한다면 주소 공간에 메모리 뿐만 아니라 레지스터도 매핑

그림06-05/06 다시그리기

- 처리 완료 확인을 위해 폴링 혹은 인터럽트 활용

### 폴링
> 폴링은 드라이버가 장치 레지스터를 반복적으로 읽어 상태 변화를 감지하는 방식,
폴링polling은 디바이스 드라이버가 능동적으로 장치에서 처리를 완료했는지 확인
- 장치는 디바이스 드라이버가 요청한 처리 완료 시 처리 완료 통지용 레지스터 값 변화
- 디바이스 드라이버는 이 값을 주기적으로 읽어서 처리 완료 확인
- 스마트폰 채팅앱(=카카오톡) 예시
  - 상대에게 질문 (처리 요청) 시
  - 주기적으로 앱을 열어서 답변 확인(폴링)
- 가장 단순한 폴링 구조
  - 디바이스 드라이버가 장치에 처리 요청
  - 처리 완료할때 까지 확인용 레지스터 계속 확인
  - 두개의 프로세스 p0, p1이 존재할 때,
    - p0가 디바이스 드라이버에 처리를 요청
    - 디바이스 드라이버가 정기적으로 실행되서 장치 처리 완료 대기
  - 이때 디바이스 드라이버의 완료 확인 전까지, CPU는 확인 외 다른 작업 수행 불가
    - 장치 처리를 요청한 p0는 대기해도 무방, 처리와 별개인 p1 동작 불가는 손해
      - 장치에 처리 요청 및 완료까지의 시간은 밀리초/마이크로초 단위
      - CPU 명령 실행의 시간은 나노초 또는 그 이하
- 일정 간격을 두고 사용하는 폴링 구조
  - 장) CPU에 다른 처리를 실행할 수 있도록 함
  - 단) 디바이스 드라이버가 복잡해짐
    - 다른 프로세스인 p1에는 처리 중간중간 레지스터값 읽는 코드 삽입 필요
    - 확인 간격을 얼마나 줄지의 문제
      - 간격 김 > p0 처리완료의 시간이 늦어짐
      - 간격 짧음 > 원점회귀, 자원낭비 커짐


![interruptoverpoliing](https://i.ytimg.com/vi/7F4qQOSJGDw/sddefault.jpg)      
[Operating Systems Lecture 17: Communication with I/O devices](https://youtu.be/7F4qQOSJGDw?si=xdoGDsznnfS2sJgC&t=495)      


### 인터럽트
> 인터럽트는 장치가 이벤트 발생 시 커널에 인터럽트를 발생시켜 드라이버가 처리하도록 하는 방식.
인터럽트interrupt 컨트롤러의 인터럽트 핸들러를 호출하여, 처리 완료를 확인
- 디바이스 드라이버가 장치 처리 요청, CPU는 다른 작업 수행
- 장치 처리 완료 시 인터럽트 방식으로 CPU에 알림
- CPU는 사전에 디바이스 드라이버가 인터럽트 컨트롤러 하드웨어에 등록해둔 인터럽트 핸들러 처리 호출
- 인터럽트 핸들러가 장치의 처리 결과 수신
- 스마트폰 채팅앱(=카카오톡) 예시
  - 상대에게 질문 (처리 요청) 시
  - 완료되면 알림(인터럽트 수신)
- 주요 차이점
  - 장치 처리 완료까지 CPU는 다른 프로세스 실행 가능(p1)
  - 장치 처리 완료를 즉시 확인 가능
  - 장치에서 처리가 이뤄지는 동안 다른 프로세스는 장치에 신경쓸필요X
- 따라서 폴링보다 다루기 쉬워, 인터럽트를 더 많이 활용함
- 인터럽트의 확인은 /proc/interrupts 파일을 통해 확인 가능

```
❯ cat /proc/interrupts
           CPU0       CPU1       CPU2       CPU3       CPU4       CPU5       CPU6       CPU7       CPU8       CPU9       CPU10      CPU11      
  8:          0          0          0          0          0          0          0          0          0          0          0          0   IO-APIC   8-edge      rtc0
  9:          0          0          0          0          0          0          0          0          0          0          0          0   IO-APIC   9-fasteoi   acpi
 24:          0          1          0          0          0          0          0          0          0          0          0          0  Hyper-V PCIe MSI 1703491403776-edge      virtio0-config
 25:          0          0       1760          0          0          0          0          0          0          0          0          0  Hyper-V PCIe MSI 1703491403777-edge      virtio0-virtqueues
 26:          0          0          0          0          1          0          0          0          0          0          0          0  Hyper-V PCIe MSI 7177964093440-edge      virtio1-config
 27:          0          0          0          0          0          1          0          0          0          0          0          0  Hyper-V PCIe MSI 7177964093441-edge      virtio1-hiprio
 28:          0          0          0          0          0          0         15          0          0          0          0          0  Hyper-V PCIe MSI 7177964093442-edge      virtio1-requests.0
NMI:          0          0          0          0          0          0          0          0          0          0          0          0   Non-maskable interrupts
LOC:          0          0          0          0          0          0          0          0          0          0          0          0   Local timer interrupts
SPU:          0          0          0          0          0          0          0          0          0          0          0          0   Spurious interrupts
PMI:          0          0          0          0          0          0          0          0          0          0          0          0   Performance monitoring interrupts
IWI:          1          0          0          1          0          0          0          0          0          0          0          0   IRQ work interrupts
RTR:          0          0          0          0          0          0          0          0          0          0          0          0   APIC ICR read retries
RES:       2414       1769       2401       1381       3006       1228       2815        912       1992       1126       3015       1153   Rescheduling interrupts
CAL:     162477     196247     221006     121702     233388      97035     187840      63121     163497      69566     167543      70789   Function call interrupts
TLB:          0          0          0          0          0          0          0          0          0          0          0          0   TLB shootdowns
TRM:          0          0          0          0          0          0          0          0          0          0          0          0   Thermal event interrupts
HYP:     128684      10761      11977        839        269       1888        914        263        556        627        581          0   Hypervisor callback interrupts
HRE:          0          0          0          0          0          0          0          0          0          0          0          0   Hyper-V reenlightenment interrupts
HVS:     182236     106466     194126      86078     178003      63475     204102      42372     194346      80860     189229      61902   Hyper-V stimer0 interrupts
ERR:          0
MIS:          0
PIN:          0          0          0          0          0          0          0          0          0          0          0          0   Posted-interrupt notification event
NPI:          0          0          0          0          0          0          0          0          0          0          0          0   Nested posted-interrupt event
PIW:          0          0          0          0          0          0          0          0          0          0          0          0   Posted-interrupt wakeup event
```


#### 일부러 폴링을 사용하는 경우
본문에서는 폴링보다 인터럽트를 권장했지만, 특수 경우에는 폴링의 사용이 권장되기도 한다
- 인터럽트 방식의 단점
  - 인터럽트 핸들러 호출은 오버헤드 발생
  - 장치 처리가 너무 빠르면 인터럽트 핸들러를 호출하는 사이에 차례로 인터럽트가 발생해 처리를 따라잡지 못할 수 있다
- 장치 처리가 빠르고, 처리 빈도가 높다면 예외적으로 폴링을 사용하기도 함
  - 평소에는 인터럽트 방식, 빈도가 높아지면 폴링으로 전환하는 디바이스 드라이버도 있음

> 사용자 공간 입출력은 응용 프로그램이 시스템 콜을 통해 커널을 거쳐 디바이스에 데이터를 읽고 쓰는 방식입니다.
사용자 공간 입출력(userspace I/O, UIO) > 디바이스 레지스터를 매핑한 메모리 영역을 프로세스 가상 주소 공간에 매핑해, 프로세스에서 장치를 조작
- 파이썬과 같은 userspace process로 디바이스 드라이버 생성 가능
- UIO를 통해 디바이스 파일에 접근할 때 마다 CPU 모드가 전환되는걸 막아 장치 접근속도 빨라짐 기대가능
- 처리 고속화 목적으로 UIO 사용하는 디바이스 드라이버는... 다양한 기법활용
  - 폴링을 사용하여 장치와 상호작용
  - 디바이스 드라이버 전용으로 논리 CPU 할당 등

## 디바이스 파일명은 바뀌기 마련
여러 장치에 연결되어있을 시, 각 장치와 디바이스 파일의 대응은 PC를 기동할때마다 변경됨
- 커널은 일정 규칙에 따라 장치를 각 다른 이름으로 디바이스 파일에 매핑
  - SATA/SAS > /dev/sda, /dev/sdb, sdc, ...
  - NVMe SSD > /dev/nvme0n1, /dev/nvme1n1, etc...
- SATA 접속방식의 저장장치 A,B (=HDD 2개)를 연결할 경우
  - 무엇이 /dev/sda, /dev/sdb될지는 장치 인식 순서에 따라 달라짐
- 재시작 시 순서변경의 가능성, 변경 이유
  1. 다른 저장 장치의 추가
    - 저장장치 C를 추가하여 A>C>B순으로 인식시 B는 /dev/sdc
  2. 저장 장치 위치의 변경
    - 예를 들어 A와 B의 위치 변경시 A가 /dev/sdb, B가 /dev/sda
  3. 저장 장치 고장으로 인식 불가
    - A가 고장, B가 /dev/sda
- 순서 변경시 부팅 실패부터 데이터 초기화 까지 파장은 다양함
  - 기존 데이터 있는 장치에 신규 파티션 생성시 초기화 동반
- systemd의 udev를 사용해 영구장치명 persistent device name을 이용
  - udev는 장치 구성의 변경과 별개로 /dev/disk 아래 자동 생성
  - 영구 장치명은 /dev/disk/by-path 디렉토리 아래에 디바이스가 설치된 경로위치 바탕의 디바이스 파일 존재
  - 파일시스템의 레이블, UUID가 있다면 /dev/disk/by-label 또는 /dev/disk/by-uuid 디바이스 파일 생성
- 단순 파일시스템의 실수 방지라면, mount 옵션에 레이블 또는 UUID를 사용하여 방지 가능
  - /etc/fstab에 /dev/sda같은 커널이름이 아닌 UUID 지정을 통해 지정 마운트 가능

---

anaconda.cfg    
https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/6/html/installation_guide/sn-automating-installation#sn-automating-installation    
https://yongbi.tistory.com/3    

리눅스 설치과정을 자동화하는 도구     
여러개의 장치디바이스있을때 sda,sdb 지정해서 돌게설정       

[mainboard](https://img.danawa.com/prod_img/500000/534/813/desc/prod_4813534/add_1/0.994463001484716641.jpg)
SATA 꽂는 위치차이인줄알았는데 아니였음
