# Table of Contents

- [파일 시스템](#파일-시스템)
  - [파일 접근 방법](#파일-접근-방법)
  - [메모리 맵 파일](#메모리-맵-파일)
  - [일반적인 파일 시스템](#일반적인-파일-시스템)
  - [쿼터(용량 제한)](#쿼터(용량 제한))
  - [파일 시스템 정합성 유지](#파일-시스템-정합성-유지)
    - [저널링을 사용한 오류 방지](#저널링을-사용한-오류-방지)
    - [카피 온 라이트로 오류 방지](#카피-온-라이트로-오류-방지)
    - [뭐니 뭐니 해도 백업](#뭐니-뭐니-해도-백업)
  - [Btrfs가 제공하는 파일시스템의 고급 기능](#btrfs가-제공하는-파일시스템의-고급-기능)
    - [스냅샷](#스냅샷)
    - [멀티 볼륨](#멀티-볼륨)
  - [결국 어떤 파일 시스템을 사용하면 좋은가?](#결국-어떤-파일-시스템을-사용하면-좋은가)
  - [데이터 손상 감지와 복구](#데이터-손상-감지와-복구)
  - [기타 파일 시스템](#기타-파일-시스템)
    - [메모리 기반의 파일 시스템](#메모리-기반의-파일-시스템)
    - [네트워크 파일 시스템](#네트워크-파일-시스템)
    - [procfs](#procfs)
    - [sysfs](#sysfs)

---

# 파일 시스템
- 대부분의 저장 장치는 파일 시스템(file system)을 사용하여 장치에 접근
  - 일반적인 장치들은 디바이스 파일로 접근
- 파일 시스템의 역할
  - 디스크의 데이터 저장 위치
  - 데이터 훼손 방지를 위한 빈 영역 관리
  - 쓰기 이후, 읽기를 위한 메타데이터(위치, 크기, 데이터종류)
- 사용자 직접 관리 대신 저장장치의 관리 영역에 기록

- 파일 시스템
  - 관리 영역(파일 형식으로 데이터를 관리하는 저장 장치의 영역)
  - 파일 시스템 코드 (해당 영역을 다루는 처리)

- 디렉터리(Directory)
  - 리눅스 파일 시스템은 각 파일은 디렉터리라는 특수 파일을 통해 분류

- 파일시스템의 데이터 종류
  - 데이터
    - 실제 사용하는 데이터(문서, 영상, 동영상, 프로그램)
  - 메타 데이터
    - 파일 관리 목적의 부가적 정보 (종류, 권한, 등)

![Unix File Structure](https://teaching.healthtech.dtu.dk/unix/images/3/3e/File_structure2.png)      
[The Unix File system](https://teaching.healthtech.dtu.dk/unix/index.php/Unix_architecture_and_file_system)
- 사전 정의된 directory와 사용자가 생성가능한 directory

[Linux File System Directory Architecture](https://thesagediary.wordpress.com/2020/04/14/linux-file-system-directory-architecture/)
- 각 시스템 디렉토리의 역할 및 기능 설명
- 간단하게 [이런것](https://substackcdn.com/image/fetch/w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F9ee1bb1e-32cf-4b6a-96ac-878f652ac53d_1200x1581.gif)도 있음

## 파일 접근 방법
- POSIX에 정의된 함수로 접근 가능 > 파일시스템 종류 상관없이 사용 가능
- 파일 조작
  - 작성, 삭제: creat(), unlink() 등
  - 열고 닫기: open(), close() 등
  - 읽고 쓰기: read(), write(), mmap() 등
- 디렉터리 조작
  - 작성, 삭제: mkdir(), rmdir()
  - 현재 디렉터리 변경: chdir()
  - 열고 닫기: opendir(), closedir()
  - 읽기: readdir() 등

- bash 등과 같은 셸로 파일시스템 접근시 내부적으로 위의 함수 호출
- 처리 과정
  1. 파일시스템 조작용 함수가 내부적으로 파일시스템 조작 시스템콜 호출
  2. 커널 내부 가상 파일 시스템(Virtual File System, VFS) 동작 및 각 처리 호출
  3. 파일 시스템 처리가 디바이스 드라이버 호출      
    ㄴ 파일 시스템 처리 <> 디바이스 드라이버 간 블록 계층 존재(9장)
  4. 디바이스 드라이버가 장치 조작

![image](https://github.com/user-attachments/assets/1989168b-b30e-4faf-8429-0e0e9cb77deb)
- [Block Diagram from Anatomy of the Linux File System](https://embedkari.com/linux-filesystem/)
- [Anatomy of the Linux file system](https://developer.ibm.com/tutorials/l-linux-filesystem/?mhsrc=ibmsearch_a&mhq=Anatomy%20of%20the%20Linux%20File%20System&mhp=2)

## 메모리 맵 파일
- 메모리 맵 파일(memory-mapped file): 파일 영역을 가장 주소 공간에 매핑
- 가상 메모리와 비슷하게, 파일 영역도 가상 주소에 매핑하여 처리
  - mmap() 함수를 특정 방법으로 호출하여
  - 파일 내용을 메모리로 읽고 그 영역을 가상 주소 공간에 매핑
  - 메모리의 데이터 변경 시 저장 장치의 파일도 특정 타이밍에 변경(8장)

./filemap.go
- testfile을 열어서 메모리 공간 매핑 확인
  1. 프로세스 메모리 맵 상황(/proc/\<pid\>/maps) 출력
  2. testfile 열어서 파일을 mmap()으로 메모리 공간 매핑
  3. 프로세스 메모리 맵 상황 다시 출력
  4. 매핑된 영역의 데이터를 hello > HELLO 변경
- 0x7f78066cb000 가상 주소에 testfile이 매핑되고, 파일이 수정됨

```
echo hello > testfile
cat testfile
hello

go build filemap.go
./filemap

*** testfile 메모리 맵 이전의 프로세스 가상 주소 공간 ***
00400000-004aa000 r-xp 00000000 08:50 168401                             /home/nasir/2025-linux-for-beginner/07/filemap
004aa000-00583000 r--p 000aa000 08:50 168401                             /home/nasir/2025-linux-for-beginner/07/filemap
00583000-0059a000 rw-p 00183000 08:50 168401                             /home/nasir/2025-linux-for-beginner/07/filemap
0059a000-005b8000 rw-p 00000000 00:00 0 
c000000000-c004000000 rw-p 00000000 00:00 0 
7f78066cc000-7f78089bd000 rw-p 00000000 00:00 0 
7ffc4f892000-7ffc4f8b4000 rw-p 00000000 00:00 0                          [stack]
7ffc4f986000-7ffc4f98a000 r--p 00000000 00:00 0                          [vvar]
7ffc4f98a000-7ffc4f98c000 r-xp 00000000 00:00 0                          [vdso]

testfile을 매핑한 주소: 0x7f78066cb000

*** testfile 메모리 맵 이후의 프로세스 가상 주소 공간 ***
00400000-004aa000 r-xp 00000000 08:50 168401                             /home/nasir/2025-linux-for-beginner/07/filemap
004aa000-00583000 r--p 000aa000 08:50 168401                             /home/nasir/2025-linux-for-beginner/07/filemap
00583000-0059a000 rw-p 00183000 08:50 168401                             /home/nasir/2025-linux-for-beginner/07/filemap
0059a000-005b8000 rw-p 00000000 00:00 0 
c000000000-c004000000 rw-p 00000000 00:00 0 
7f78066cb000-7f78066cc000 rw-s 00000000 08:50 58137                      /home/nasir/2025-linux-for-beginner/07/testfile
7f78066cc000-7f78089bd000 rw-p 00000000 00:00 0 
7ffc4f892000-7ffc4f8b4000 rw-p 00000000 00:00 0                          [stack]
7ffc4f986000-7ffc4f98a000 r--p 00000000 00:00 0                          [vvar]
7ffc4f98a000-7ffc4f98c000 r-xp 00000000 00:00 0                          [vdso]

cat testfile
HELLO
```

## 일반적인 파일 시스템
| 파일 시스템 | 특징 | 비고 |
|:------------|:-----|:-----|
| **ext4** | 가장 널리 쓰이는 기본 파일 시스템. 빠르고 안정적. | 특별한 이유 없으면 이걸 사용 |
| **xfs** | 대용량 파일, 대규모 디렉터리에 강함. | RHEL, CentOS 계열 기본 선택지 |
| **btrfs** | 스냅샷, 롤백, 압축, 체크섬 등 고급 기능. | 아직 100% 신뢰하기엔 이른 편 |
| **vfat / exfat** | USB, SD 카드 등 이동식 매체용. | 리눅스 고유 파일 시스템 아님 |
| **ntfs** | 윈도우 파일 시스템. 리눅스에서 읽고 쓸 수 있음. | 주로 데이터 교환용 |
| **zfs** | 스냅샷, 풀링, 자동복구 기능 제공. | 메모리 많이 필요, 라이선스 주의 |
| **tmpfs** | 메모리 기반 임시 파일 시스템. 부팅 후 초기화됨. | `/tmp`, `/run` 등에 사용 |
| **nfs** | 네트워크를 통한 파일 공유용 파일 시스템. | 서버 간 디스크 공유할 때 사용 |

- 리눅스에서는 ext4, XFS, Btrfs 같은 파일 시스템(이하 fs) 주로 사용
- 각 fs는 저장 장치에서의 데이터 구조와 관리 처리방식이 다름, 대표적으로..
  - 파일 시스템 최대 크기
  - 파일 최대 크기
  - 최대 파일 개수
  - 파일명 최대 길이
  - 동작별 처리 속도
  - 표준 기능 외 추가 기능 유무

![image](https://github.com/user-attachments/assets/71a136e9-d96a-41f6-af9a-19d23841dba9)
- [Virtual filesystems in Linux: Why we need them and how they work](https://opensource.com/article/19/3/virtual-filesystems-linux)
- 각 파일시스템별 윗단/아랫단과의 연결 관계
  
## 쿼터(용량 제한)
- 쿼터(quota): 시스템에서 용도별로 사용 가능한 fs 용량 제한 기능 필요
  - 다양한 용도 시스템 사용 시, 특정 기능으로 인해 다른 기능 실행에 필요한 용량 부족 문제 발생 가능
  - 시스템 관리용 처리를 위한 용량이 부족하면 전체가 불안정
  - 특정 사용자/프로그램이 과도하게 사용하지 않도록 제어 필요

- 종류 
  - 사용자 쿼터
    - 파일 소유자인 사용자마다 용량을 제한
    - 일반 사용자로 인한 /home/ 디렉터리 풀 방지
    - ext4/XFS 에서 사용 가능
  - 디렉터리 쿼터(프로젝트 쿼터)
    - 특정 디렉터리마다 용량 제한
    - 프로젝트 멤버가 공유하는 디렉터리에 용량 제한
    - ext4/XFS 에서 사용 가능  
  - 서브 볼륨 쿼터
    - fs 내부의 서브볼륨 단위마다 용량 제한
    - 디렉터리 쿼터와 사용법 비슷
    - Btrfs 에서 사용 가능

## 파일 시스템 정합성 유지
- 시스템 사용시 fs 내용에 오류가 생길 수도 있음
  - 저장장치 RW 도중 전원이 강제로 끊김 / 프로그램 강제 종료
- 이후 fs이 오류 감지시 다음과 같은 문제 발생 가능
  - 마운트 작업 > fs 마운트 불가능 or RO로 mount
  - 시스템 패닉(system panic) 발생 가능
- 오류 방지 기능의 대표적 2가지 방식
  - 저널링 (ext4,XFS)
  - 카피 온 라이트 (Btrfs)

### 저널링을 사용한 오류 방지
- 저널링(journaling): fs 내부의 저널영역이라는 별도 메타데이터영역 준비
- journaling의 갱신 방법
  1. 갱신에 필요한 아토믹한 처리 목록을 저널영역에 기록 > 저널로그 journal log
  2. 저널 영역의 내용에 따라 실제로 fs 내용 갱신
    - 갱신 완료 시 (성공) > 저널로그 파기하여 완료
    - 저널 로그 갱신 중 (1) 에러 > 저널로그 drop
    - 실제 fs 갱신 중 (2) 에러 > 저널로그 재시작

![journaling](https://i0.wp.com/foxutech.com/wp-content/uploads/2017/03/Journaling-FileSystem.png?fit=640%2C287&ssl=1)
- [Journaling FileSystem & Its 3 types](https://foxutech.com/journaling-filesystem/)



### 카피 온 라이트로 오류 방지
- fs 간 저장 방법의 차이
  - 저널링(ext,XFS) > 파일 갱신 시 동일 위치에 데이터 갱신
  - 카피온라이트(Btrfs) > 새로운 장소에 데이터 작성 이후 링크 변경
- COW의 갱신 방법
  1. 파일 내부의 변경된 부분만 다른 장소에 복사 및 변경
  2. 갱신한 데이터 작성 이후 링크 재작성 하여 완료
    - 링크 완료시 (성공) > 기존 파일 제거
    - 변경 처리중 에러 (1) > 중간데이터 삭제 하고 재생성
 
![image](https://github.com/user-attachments/assets/63386e1c-bbb1-4ae8-91d1-4ad3567b2148)
- [COW and clones: how they save space and SSD wear](https://eclecticlight.co/2023/05/04/cow-and-clones-how-they-save-space-and-ssd-wear/)
- 실제 데이터(FA,FB,FC,FD)는 그대로 두고 변경되는 AB데이터만 변경 후 메타데이터 수정


### 뭐니 뭐니 해도 백업
- 위의 방법을 통해 오류의 빈도를 줄일 수 있으나 궁극적인 제거는 불가
  - 파일 시스템의 버그 or 하드웨어 장애의 가능성
- fs 백업을 통해 마지막 백업 시점으로의 복원 필요성
- 정기 백업이 어렵다면, 복구용 명령어로 정합성 회복 시도 가능
  - fsck
    - ext4: fsck.ext4
    - XFS: xfs_repair
    - Btrfs: btrfs check
  - fsck가 적절하지 않은 경우
    1. 필요 시간 김
      - 정합성 확인 및 복구를 위해 fs 전체를 검사하기에, 대용량 데이터의 경우 몇십시간~며칠 단위 소요
    2. 복구 검사 결과 실패할 가능성 존재
    3. 시점 설정 불가
      - 데이터 오류가 발생한 파일을 마운트(사용) 가능하게 만들 뿐, 오류가 생긴 데이터/메타데이터는 삭제됌

## Btrfs가 제공하는 파일시스템의 고급 기능

### 스냅샷
- 파일시스템의 스냅샷(snapshot) 사용 가능
  - 백업과 달리, 데이터 전체가 아닌 메타데이터만 작성
  - 일반 복사 작업 대비 빠른 속도
  - 원본 파일시스템과 데이터 공유하므로 차지하는 공간 절감
  - COW 데이터 갱신 특정 최대 활용

- 처리 과정
  1. A 데이터를 다른 새로운 영역에 복사
  2. 새로운 영역 데이터를 갱신

- 원본 시스템과 데이터를 공유 > 공유 데이터 손실 시 스냅샷도 손실
- 백업 작업시 입출력을 멈춰야함, 스냅샷 사용시 스냅샷 중간의 짧은 시간만 입출력 멈추고 스냅샷 대상 백업할 수 있어 장점

![image](https://github.com/user-attachments/assets/512f6401-b9c7-4339-b1bf-6b896b737377)
[Exploiting Point in Time Copy Services](https://technoscooop.wordpress.com/tag/redirect-on-write/)
- COW와 비슷하게 동일 데이터를 바라보는 메타데이터(스냅샷) 생성 이후 변경이 필요한 데이터만 스냅샷에 연결


### 멀티 볼륨
- ext4/XFS > 1개의 파티션에 단일 파일 시스템 생성
- Btrfs > 다수 저장장치/파티션으로 저장소 풀(storage pool) 생성후 마운트 가능한 서브 볼륨(subvolume) 생성
  - LVM에서 다루는 볼륨 그룹과 유사
  - 서브 볼륨은 LVM의 논리볼륨+파일시스템과 유사
- RAID 구성 가능
  - RAID 0, 1, 10, 5, 6, dup(이중화)


![image](https://github.com/user-attachments/assets/c5fb9f82-28e2-4513-8736-ddebdffa6309)


#### 결국 어떤 파일 시스템을 사용하면 좋은가?
- 모든 상황에서 만족스러운 파일시스템은 X, 장단점 존재
  - 기능이 많으면 느리고 vs 빠르면 기능이 적고
- 조건에 따라서 좋은 파일 시스템을 결정할 수 있음
  - 특정 필수 기능이 있고, 특정 fs에서만 지원 조건 > 간단
  - 일반적으로는 조건이 복잡함
  - 단순히 대량 파일 연속 작업하는 벤치마크로는 평가가 어려움, 직접 조건 상황의 테스트 필요

## 데이터 손상 감지와 복구
- Btrfs에는 모든 데이터에 체크섬checksum이 있어 데이터 손상 감지 가능 
- 레이드 구성등을 통해 이중화 해두었으면 정상 데이터 기반으로 손상 데이터 복구 가능

## 기타 파일 시스템
### 메모리 기반의 파일 시스템
- tmpfs: 저장 장치 대신 메모리를 사용
  - 전원이 꺼지면 사라짐 / 접근 속도가 빠름
  - /tmp, /var/run등에서 주로 사용

```
mount | grep tmpfs
none on /mnt/wsl type tmpfs (rw,relatime)
none on /mnt/wsl/docker-desktop/shared-sockets/guest-services type tmpfs (rw,nosuid,nodev,mode=755)
none on /mnt/wsl/docker-desktop/shared-sockets/host-services type tmpfs (rw,nosuid,nodev,mode=755)
none on /mnt/wsl/docker-desktop-bind-mounts/Ubuntu/docker.sock type tmpfs (rw,nosuid,nodev,mode=755)
none on /mnt/wslg type tmpfs (rw,relatime)
none on /dev type devtmpfs (rw,nosuid,relatime,size=8094896k,nr_inodes=2023724,mode=755)
none on /run type tmpfs (rw,nosuid,nodev,mode=755)
none on /run/lock type tmpfs (rw,nosuid,nodev,noexec,noatime)
none on /run/shm type tmpfs (rw,nosuid,nodev,noatime)
none on /dev/shm type tmpfs (rw,nosuid,nodev,noatime)
none on /run/user type tmpfs (rw,nosuid,nodev,noexec,noatime,mode=755)
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,mode=755)
none on /tmp/.X11-unix type tmpfs (ro,relatime)
none on /run/user type tmpfs (rw,relatime)
none on /run/snapd/ns type tmpfs (rw,nosuid,nodev,mode=755)
```

- free 명령어 출력에서 shared 필드값이 tmpfs 등에서 사용된 메모리 용량 의미
- OS 뿐만 아니라, 사용자도 mount 명령어를 통해 생성 가능
- 생성시가 아닌, 실제 사용시 메모리 사용 값 증가
```
free
              total        used        free      shared  buff/cache   available
Mem:       16196780     2410684    12792952        6900      993144    13463296
Swap:       4194304           0     4194304

sudo mount -t tmpfs tmfps /mnt -osize=1G
mount | grep /mnt
...
tmfps on /mnt type tmpfs (rw,relatime,size=1048576k)

free
              total        used        free      shared  buff/cache   available
Mem:       16196780     2418764    12778552        6904      999464    13455192
Swap:       4194304           0     4194304

sudo dd if=/dev/zero of=/mnt/testfile bs=100M count=1
1+0 records in
1+0 records out
104857600 bytes (105 MB, 100 MiB) copied, 0.0556458 s, 1.9 GB/s

free
              total        used        free      shared  buff/cache   available
Mem:       16196780     2428984    12665852      109304     1101944    13342312
Swap:       4194304           0     4194304

sudo umount /mnt
free
              total        used        free      shared  buff/cache   available
Mem:       16196780     2429916    12766372        7124     1000492    13443824
Swap:       4194304           0     4194304
```

### 네트워크 파일 시스템
- Network File System(NFS), Common Internet File System(CIFS)
  - 일반적인 파일시스템은 로컬 저장 장치의 데이터를 다룸
  - 네트워크로 연결된 원격 호스트의 데이터에 파일시스템 인터페이스를 사용하여 접근
  - 원격 호스트의 파일 시스템을 로컬 머신의 파일 시스템처럼 조작 가능

### procfs
- procfs: 시스템의 프로세스 관련 정보 확인
  - /proc 아래 마운트 되어있음
  - /proc/pid/ 하위에 각 pid에 대응하는 프로세스 정보 확인 가능
- ps, sar, free 등의 명령어를 procfs에서 데이터를 가져와 출력

### sysfs
- 커널의 정보를 저장하기 위한 장소
  - 초기에 프로세스 관련 정보외에 커널 정보도 procfs에 저장
  - procfs 남용을 방지하기 위해 /sys/ 아래 sysfs 저장
- 예시
  - /sys/block


```
ll /sys/block
total 0
lrwxrwxrwx 1 root root 0 Apr 29 12:55 loop0 -> ../devices/virtual/block/loop0
lrwxrwxrwx 1 root root 0 Apr 29 12:55 loop1 -> ../devices/virtual/block/loop1
lrwxrwxrwx 1 root root 0 Apr 29 12:55 loop2 -> ../devices/virtual/block/loop2
lrwxrwxrwx 1 root root 0 Apr 29 12:55 loop3 -> ../devices/virtual/block/loop3
lrwxrwxrwx 1 root root 0 Apr 29 12:55 loop4 -> ../devices/virtual/block/loop4
lrwxrwxrwx 1 root root 0 Apr 29 12:55 loop5 -> ../devices/virtual/block/loop5
lrwxrwxrwx 1 root root 0 Apr 29 12:55 loop6 -> ../devices/virtual/block/loop6
lrwxrwxrwx 1 root root 0 Apr 29 12:55 loop7 -> ../devices/virtual/block/loop7
lrwxrwxrwx 1 root root 0 Apr 29 12:55 ram0 -> ../devices/virtual/block/ram0
lrwxrwxrwx 1 root root 0 Apr 29 12:55 ram1 -> ../devices/virtual/block/ram1
lrwxrwxrwx 1 root root 0 Apr 29 12:55 ram10 -> ../devices/virtual/block/ram10
lrwxrwxrwx 1 root root 0 Apr 29 12:55 ram11 -> ../devices/virtual/block/ram11
lrwxrwxrwx 1 root root 0 Apr 29 12:55 ram12 -> ../devices/virtual/block/ram12
lrwxrwxrwx 1 root root 0 Apr 29 12:55 ram13 -> ../devices/virtual/block/ram13
lrwxrwxrwx 1 root root 0 Apr 29 12:55 ram14 -> ../devices/virtual/block/ram14
lrwxrwxrwx 1 root root 0 Apr 29 12:55 ram15 -> ../devices/virtual/block/ram15
lrwxrwxrwx 1 root root 0 Apr 29 12:55 ram2 -> ../devices/virtual/block/ram2
lrwxrwxrwx 1 root root 0 Apr 29 12:55 ram3 -> ../devices/virtual/block/ram3
lrwxrwxrwx 1 root root 0 Apr 29 12:55 ram4 -> ../devices/virtual/block/ram4
lrwxrwxrwx 1 root root 0 Apr 29 12:55 ram5 -> ../devices/virtual/block/ram5
lrwxrwxrwx 1 root root 0 Apr 29 12:55 ram6 -> ../devices/virtual/block/ram6
lrwxrwxrwx 1 root root 0 Apr 29 12:55 ram7 -> ../devices/virtual/block/ram7
lrwxrwxrwx 1 root root 0 Apr 29 12:55 ram8 -> ../devices/virtual/block/ram8
lrwxrwxrwx 1 root root 0 Apr 29 12:55 ram9 -> ../devices/virtual/block/ram9
lrwxrwxrwx 1 root root 0 Apr 29 12:55 sda -> ../devices/LNXSYSTM:00/LNXSYBUS:00/ACPI0004:00/VMBUS:00/fd1d2cbd-ce7c-535c-966b-eb5f811c95f0/host0/target0:0:0/0:0:0:0/block/sda
lrwxrwxrwx 1 root root 0 Apr 29 12:55 sdb -> ../devices/LNXSYSTM:00/LNXSYBUS:00/ACPI0004:00/VMBUS:00/fd1d2cbd-ce7c-535c-966b-eb5f811c95f0/host0/target0:0:0/0:0:0:1/block/sdb
lrwxrwxrwx 1 root root 0 Apr 29 12:55 sdc -> ../devices/LNXSYSTM:00/LNXSYBUS:00/ACPI0004:00/VMBUS:00/fd1d2cbd-ce7c-535c-966b-eb5f811c95f0/host0/target0:0:0/0:0:0:2/block/sdc
lrwxrwxrwx 1 root root 0 Apr 29 12:55 sdd -> ../devices/LNXSYSTM:00/LNXSYBUS:00/ACPI0004:00/VMBUS:00/fd1d2cbd-ce7c-535c-966b-eb5f811c95f0/host0/target0:0:0/0:0:0:3/block/sdd
lrwxrwxrwx 1 root root 0 Apr 29 12:55 sde -> ../devices/LNXSYSTM:00/LNXSYBUS:00/ACPI0004:00/VMBUS:00/fd1d2cbd-ce7c-535c-966b-eb5f811c95f0/host0/target0:0:0/0:0:0:4/block/sde
lrwxrwxrwx 1 root root 0 Apr 29 13:15 sdf -> ../devices/LNXSYSTM:00/LNXSYBUS:00/ACPI0004:00/VMBUS:00/fd1d2cbd-ce7c-535c-966b-eb5f811c95f0/host0/target0:0:0/0:0:0:5/block/sdf


cat /sys/block/sda/dev
8:0
ll /dev/sda
brw-rw---- 1 root disk 8, 0 Apr 29 15:24 /dev/sda
```
