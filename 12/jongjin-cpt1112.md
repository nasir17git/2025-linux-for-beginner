# 컨테이너

- 컨테이너(Container) 기술
  - 컨테이너 어플리케이션을 관리하는 도커, 컨테이너 오케스트레이션 시스템은 쿠버네티스
  - 단순 사용은 쉽지만 문제시 조사 또는 구조 이해는 어렵다

## 가상머신과의 차이점
- 유사점
  - 각각 독립된 프로세스 실행환경을 제공
- 차이점
  - 가상머신
    - 각 가상머신 전용의 가상 H/W 및 커널 사용
    - 윈도우에서 리눅스 실행 등 완전히 다른 커널의 호스트 OS 사용 가능
  - 컨테이너
    - 호스트 OS와 모든 컨테이너가 하나의 커널을 공유
    - 리눅스 컨테이너는 리눅스 커널에서만 동작하는 시스템만 사용 가능

- VM에서의 OS(우분투)실행 과정
  1. 호스트의 가상화 SW가 VM 기동
  2. GRUB 같은 부트로더 기동
  3. 부트로더가 커널 기동
  4. 커널이 init 프로그램 기동
  5. init(systemd)가 여러 서비스 기동

- 컨테이너에서의 OS(우분투)실행 과정
  1. 컨테이너 런타임 프로세스가 컨테이너 작성 후 최초 프로세스 기동

- 따라서 VM보다 처리가 가벼움
  - 기동속도: 1~3까지의 과정 생략 가능
  - HW 접근속도: VM처럼 하드웨어 접근에 의해 물리기기에 제어 넘기는 과정 X

## 컨테이너 종류
- 크게 2가지 종류의 컨테이너로 구분 가능
  - 시스템 컨테이너
    - 일반 리눅스 환경처럼 다양한 앱 실행을 위한 컨테이너
    - 최초 프로세스로 init 프로세스 실행 > 이후 각종 서비스 (VM과 유사)
    - 도커 이전의 컨테이너 대부분이 시스템 컨테이너(LXD,Linux Container Daemon)
  - 애플리케이션 컨테이너
    - 특정 앱 실행을 위한 컨테이너
    - 최초 프로세스로 앱 실행 > 이후 각종 서비스 실행 X
    - 도커 이후의 컨테이너 대부분이 애플리케이션 컨테이너(도커)

## 네임스페이스
- 커널에는 컨테이너라고 부르는 자체적인 기능은 없음
- 컨테이너는 커널의 네임스페이스(namespace) 기능등을 사용하여 구현

- 네임스페이스: 시스템의 다양한 자원을 공유사용 대신 각 소속 프로세스에 독립적으로 제공하는 기능
  - 프로세스ID 네임스페이스(pid namespace): 독립된 pid 이름 공간 제공
  - 사용자 네임스페이스(user namespace): 독립된 uid, gid 제공
  - 마운트 네임스페이스(mount namespace): 독립된 fs 마운트 포인트 제공

### 프로세스ID 네임스페이스
- 시스템 시작 시 모든 프로세스가 소속된 루트 프로세스ID 네임스페이스(root pid ns) 존재
  - 루트 프로세스ID 네임스페이스는 모든 프로세스가 소속되어 있음
  - 각 프로세스는 pid를 통해 다른 프로세스를 식별 가능(A>B,C)
- 다른 프로세스ID 네임스페이스 생성 시 root pid ns 조회 불가
  - foo pid ns 생성시, root pid ns 내부에 자식으로 생성되며 부모(root pid ns) 조회 불가 (A>B가능, B>A불가능)

- 실습
  - pid는 `ls -l /proc/<pid>/ns/pid` 로 확인 가능
  - unshare 명령어로 새로운 ns에서 프로세스 실행 가능
  - root pid 4026532219, 새로운 ns pid 4026532221
  - root에서 조회하는 bash의 pid는 11030, 새로운 ns에서 조회하는 bash의 pid는 1

```
ls -l /proc/$$/ns/pid
lrwxrwxrwx 1 nasir nasir 0 Jun  3 18:30 /proc/10289/ns/pid -> 'pid:[4026532219]'

sudo unshare --fork --pid --mount-proc bash

root@nasir-pc:/home/nasir/2025-linux-for-beginner# echo $$
1

root@nasir-pc:/home/nasir/2025-linux-for-beginner# ls -l /proc/1/ns/pid
lrwxrwxrwx 1 root root 0 Jun  3 18:30 /proc/1/ns/pid -> 'pid:[4026532221]'

pstree -p
init-systemd(Ub(2)─┬─SessionLeader(10106)───Relay(10108)(10107)─┬─sh(10108)───sh(10109)───sh(10114)───node(10118)─┬─node(10142)─┬─zsh(10289)───sudo(11028)───unshare(11029)───bash(11030)

 sudo ls -l /proc/11030/ns/pid
lrwxrwxrwx 1 root root 0 Jun  3 18:32 /proc/11030/ns/pid -> 'pid:[4026532221]'
```

### 컨테이너 정체
- 컨테이너: 독립된 네임스페이스를 가지고 다른 프로세스와 실행 환경이 나뉘는 하나 또는 여러 프로세스
  - 각 컨테이너마다 서로다른 pid ns, user ns, mount ns 등 존재
- 이러한 ns 분리 방식은 각 컨테이너 런타임마다 다름 + ns는 계속해서 늘어남
- 특정 컨테이너(foo) 입장에서, 다른 ns(root/bar/etc..)의 문제 조회 불가

## 보안 위험성
- 일반적으로 컨테이너는 VM 대비 보안 위험성이 큼
  - 컨테이너는 호스트 시스템 및 모든 컨테이너가 커널을 공유
  - 악의적인 컨테이너 사용자가 호스트의 커널에 침범 가능
  - VM의 경우 게스트 내부의 커널선에서 그침

# cgroup
- cgroup(control group): CPU/Mem 같은 시스템 자원을 특정 프로세스에 얼마나 제공할지 세밀한 제어
  - 프로세스를 그룹(group)으로 나누어 각종 자원을 제어함(control)

- 시스템 안정적인 사용을 위해 특정사용자의 독점 방지
  - IaaS) 공유 공간의 VM 리소스 사용량 제어
  - DB) 일반작업과 백업작업 실행 중, 일반 작업에 영향주지않도록 백업작업의 자원 제어

## cgroup으로 제어 가능한 자원
- 각 자원마다 컨트롤러라는 커널 내부 프로그램을 통해 자원 제어
  - 각 자원은 프로세스 그룹 단위로 제어 가능
- 각 컨트롤러는 cgroupfs라는 특별한 파일시스템 사용 
  - 컨트롤러별로 고유한 값이 됨
  - 우분투의 경우 /sys/fs/cgroup 에 각 컨트롤러에 대응하는 cgroup 파일 마운트
  - 저장 장치에 존재하는게 아니라 메모리에만 존재하고, 접근하면 커널의 cgroup 기능 사용 가능 

## 사용 예: CPU 사용 시간 제어
- CPU 컨트롤러를 사용한 CPU 사용 시간 제어
  1. 특정 그룹이 일정 기간동안 사용 가능한 CPU 시간을 제한
  2. 특정 그룹이 사용할 수 있는 CPU 시간 비율을 다른 그룹보다 높/낮게 제어
- 1의 경우 설명

- /sys/fs/cgroup/cpu 하위의 파일을 조작하여 cpu 컨트롤러 사용 가능
  - 기본 디렉토리 하위에 디렉토리 생성 시 새로운 그룹 생성 가능
  - 그룹(디렉토리)가 생성되면 커널이 CPU제어를 위한 다양한 파일 작성
  - 그러나 이건 cgroupv1 내용, 현재 cgroupv2 사용중 `stat -fc %T /sys/fs/cgroup` > cgroup2fs

- cgroupv2
  - `mkdir /sys/fs/cgroup/test` test 그룹 생성
  - `ls /sys/fs/cgroup/test` 기본 제어파일 확인
  - `echo "+cpu" > /sys/fs/cgroup/cgroup.subtree_control` 컨트롤러 활성화
  - `echo "50000 100000" > /sys/fs/cgroup/test/cpu.max` 100ms 동안 50ms 만 활성화 설정 (50%사용)
  - inf-loop.py 실행 후 cgroup test그룹 할당 전후 사용량 비교
  - top -b -n 1 | head

```
nasir-pc# ls /sys/fs/cgroup/test
cgroup.controllers      cpu.max.burst             hugetlb.1GB.max           io.stat              memory.swap.current
cgroup.events           cpu.pressure              hugetlb.1GB.numa_stat     memory.current       memory.swap.events
cgroup.freeze           cpu.stat                  hugetlb.1GB.rsvd.current  memory.events        memory.swap.high
cgroup.kill             cpu.stat.local            hugetlb.1GB.rsvd.max      memory.events.local  memory.swap.max
cgroup.max.depth        cpu.weight                hugetlb.2MB.current       memory.high          memory.swap.peak
cgroup.max.descendants  cpu.weight.nice           hugetlb.2MB.events        memory.low           pids.current
cgroup.pressure         cpuset.cpus               hugetlb.2MB.events.local  memory.max           pids.events
cgroup.procs            cpuset.cpus.effective     hugetlb.2MB.max           memory.min           pids.max
cgroup.stat             cpuset.cpus.partition     hugetlb.2MB.numa_stat     memory.numa_stat     pids.peak
cgroup.subtree_control  cpuset.mems               hugetlb.2MB.rsvd.current  memory.oom.group     rdma.current
cgroup.threads          cpuset.mems.effective     hugetlb.2MB.rsvd.max      memory.peak          rdma.max
cgroup.type             hugetlb.1GB.current       io.latency                memory.pressure
cpu.idle                hugetlb.1GB.events        io.max                    memory.reclaim
cpu.max                 hugetlb.1GB.events.local  io.pressure               memory.stat

./inf-loop.py &
17987

top -b -n 1 | head
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  17987 root      25   5   15828   9088   5760 R  93.8   0.1   0:56.16 inf-loop.py
      1 root      20   0  174000  14668   7884 S   0.0   0.1   0:03.19 systemd
      2 root      20   0    3060   1664   1664 S   0.0   0.0   0:00.01 init-systemd(Ub

echo 17987 > /sys/fs/cgroup/test/cgroup.procs
top -b -n 1 | head
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  17987 root      25   5   15828   9088   5760 R  53.3   0.1   1:04.65 inf-loop.py
      1 root      20   0  174000  14668   7884 S   0.0   0.1   0:03.19 systemd
      2 root      20   0    3060   1664   1664 S   0.0   0.0   0:00.01 init-systemd(Ub
```

## 응용 예
- 실제로는 파일시스템 경유하여 cgroup 제어보다는 간접적으로 사용함
  - systemd 사용: 서비스/사용자별 그룹 생성. 각 그룹명은 system.slice / user.slice 등
  - 도커/쿠버네티스 관리: 쿠버네티스 매니페스트 또는 docker 명령어 인수에 컨테이너 할당 자원 지정
  - libvirt: virt-manager 또는 VM 설정파일을 통한 자원 할당 지정