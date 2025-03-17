# 리눅스 개요
- 리눅스와 그 일부인 커널이 무엇인지
- 시스템 전체에 리눅스와, 그 외가 어떻게 다른지
- 프로그램과 프로세스처럼 같은 문맥으로 사용되는 용어 의미

## 프로그램과 프로세스

- 프로그램
  - 컴퓨터에서 동작하는 관련된 명령 및 데이터를 하나로 묶은것
    - 컴파일러형 언어(Go)> 소스코드 빌드 후 만들어진 실행 파일
    - 스크립트형 언어(Python) > 소스 코드 그 자체
  - 커널도 프로그램의 일종

- 프로세스
  - 실행되어 동작중인 프로그램
  - 프로세스는 프로그램에 포함됨

## 커널

> A kernel is a computer program at the core of a computer's operating system that always has complete control over everything in the system. [Wikipedia](https://en.wikipedia.org/wiki/Kernel_(operating_system))
> The kernel is an essential part of an operating system that manages and controls access to the resources and devices on a computer. [Codecademy](https://www.codecademy.com/resources/blog/kernel/)

> 커널은 OS에서 소프트웨어와 하드웨어간의 상호작용을 제어하는 프로그램

- 프로세스가 장치에 직접 접근가능하다면 동시성? eventual consistency 이슈 발생 가능
  - [Amazon S3 데이터 일관성 모델](https://docs.aws.amazon.com/ko_kr/AmazonS3/latest/userguide/Welcome.html#ConsistencyModel) 떠올랐네요
  - 서로 다른 프로세스가 같은 장치를 제어하려고하면 순서 이슈 발생 가능
  - 또는 접근 권한에 관한 문제 발생 가능

- 커널은 하드웨어의 도움을 받아, 다시말해 CPU의 모드 기능을 통해 프로세스의 직접 접근을 제한함
  - 일반적인 CPU에는 크게 커널 모드와 사용자 모드 두 모드가 존재
    - 사용자 공간의 프로세스는 장치에 직접 접근 불가
    - 프로세스는 커널을 통해 간접적으로 장치에 접근
  - 사용자 공간의 프로세스와 달리 커널은 장치 제어, 시스템 자원 관리 및 배분 기능 수행 > 중앙 관리
  - 참고
    - https://zu-techlog.tistory.com/122
    - https://makelinux.github.io/kernel/map/

## 시스템 콜
> a system call (commonly abbreviated to syscall) is the programmatic way in which a computer program requests a service from the operating system on which it is executed. [System call](https://en.wikipedia.org/wiki/System_call)
> A system call is a request for service that a program makes of the kernel. [GNU](https://www.gnu.org/software/libc/manual/html_node/System-Calls.html)

> 시스템 콜은 유저모드의 프로세스가 커널 모드의 커널을 호출하기 위한 인터페이스

- 프로세스가 CPU의 특수 명령을 실행하여 시스템 콜 처리
  - 새로운 프로세스나 하드웨어 조작 처럼 커널의 개입이 필요한 경우 시스템 콜 호출
  - CPU에서 예외(exception) 발생
  - CPU모드가 유저모드에서 커널모드로 변경되어 커널처리 동작
  - 커널모드의 시스템콜 처리가 끝나면 사용자 모드로 복귀
- 시스템 콜 이전 커널에서 올바른 요청인지 확인함
  - 처리범위 내의 메모리 등
- 시스템 콜을 통하지 않고 직접 CPU 모드를 변경할 수 없음

### 시스템 콜 확인하기 - strace
[strace - trace system calls and signals](https://man7.org/linux/man-pages/man1/strace.1.html)

- strace 명령어를 통해 프로세스가 어떤 시스템 콜을 호출 하는 지 확인 가능
  - -o 옵션을 통해 출력 저장 가능

```
#!/usr/bin/python3
print("hello world")
```

```
strace -o hello.py.log ./hello.py

cat hello.py.log
# ...
# write(1, "hello world\n", 12)           = 12
# ...

# 총 772번의 호출
cat hello.py.log | wc -l
772

# 호출된 시스콜 종류와 횟수
awk '{ print $1 }' hello.py.log | sort | uniq -c | sort -nr
     58 fstat(3,
     57 openat(AT_FDCWD,
     53 read(3,
     44 lseek(3,
     42 close(3)
     36 mmap(NULL,
     30 fstat(4,
     28 read(4,
     28 lseek(4,
     27 stat("/usr/lib/python3.8",
     22 ioctl(3,
     16 close(4)
     14 ioctl(4,
     13 munmap(0x7fa7471f5000,
     12 getdents64(3,
     ...
      1 write(1,
     ...
```


### 시스템 콜을 처리하는 시간 비율 - sar
[sar - Collect, report, or save system activity information](https://linux.die.net/man/1/sar)
System Activity Report

시스템에 설치된 논리 CPU가 실행하고 있는 명령 비율은 sar 명령어를 통해 확인가능

- sar -P 0 1 1
  - -P 0: CPU 0의 데이터 수집
  - 1: 수집주기 (interval, 1s)
  - 1: 수집횟수 (count, 1time)

```
nasir-pc# sar -P 0 1 1
Linux 5.15.167.4-microsoft-standard-WSL2 (nasir-pc)     03/18/25        _x86_64_        (12 CPU)

03:25:57        CPU     %user     %nice   %system   %iowait    %steal     %idle
03:25:58          0      0.00      0.00      0.00      0.00      0.00    100.00
Average:          0      0.00      0.00      0.00      0.00      0.00    100.00
```

- 첫번째 필드: 시간, 두번째 필드: CPU
- 3번째부터 8번째가 각 사용량, 합쳐서 100%
  - 유저 모드에서의 프로세스의 실행 시간 비율
    - %user, %nice
  - 커널 모드에서의 커널의 시스템콜 처리 시간 비율
    - %system
  - 유휴 상태 비율
    - %idle
  - 그외는 설명 생략

- 유저 모드만 프로세스 무한 발생
  - WSL이라서 그런가? %nice에서 100%
```
taskset -c 0 ./inf-loop.py &
# [1] 2403

nasir-pc# sar -P 0 1 1
Linux 5.15.167.4-microsoft-standard-WSL2 (nasir-pc)     03/18/25        _x86_64_        (12 CPU)

03:34:35        CPU     %user     %nice   %system   %iowait    %steal     %idle
03:34:36          0      0.00    100.00      0.00      0.00      0.00      0.00
Average:          0      0.00    100.00      0.00      0.00      0.00      0.00
```

- 시스템콜 무한 발생
  - 일부 유저모드, 일부 커널모드
```
nasir-pc# taskset -c 0 ./syscall-inf-loop.py   &
[1] 2409
nasir-pc# sar -P 0 1 1
Linux 5.15.167.4-microsoft-standard-WSL2 (nasir-pc)     03/18/25        _x86_64_        (12 CPU)

03:36:34        CPU     %user     %nice   %system   %iowait    %steal     %idle
03:36:35          0      0.00     55.00     45.00      0.00      0.00      0.00
Average:          0      0.00     55.00     45.00      0.00      0.00      0.00
nasir-pc#
```


### 시스템 콜 소요 시간 - strace -T
- strace에 -T 옵션을 사용해 각 시스템콜 처리시간을 측정 가능
```
nasir-pc# cat hello.log | grep write
write(1, "hello world\n", 12)           = 12 <0.000060>
```

- -tt 옵션을 사용해 호출 시각을 표시 가능
```
nasir-pc# cat hello.log.tt | grep write
03:38:21.328646 write(1, "hello world\n", 12) = 12
```

## 라이브러리 
> In computing, a library is a collection of resources that is leveraged during software development to implement a computer program. []Library \(computing\)](https://en.wikipedia.org/wiki/Library_(computing))

> A programming library is a collection of prewritten code that programmers can use to optimize tasks [careerfoundry](https://careerfoundry.com/en/blog/web-development/programming-library-guide/)

> 라이브러리는 프로그래밍 언어에서 자주사용되는 코드등의 모음집

일반적인 프로그래밍 언어 라이브러리처럼 OS 도 라이브러리 존재

### 표준 C라이브러리 - glibc, ldd
- C언어는 ISO가 정한 표준 라이브러리 존재
  - GNU 프로젝트에서 제공하는 glibc 를 사용 (이하 libc)
  - C언어로 작성된 대부분의 프로그램은 libc를 링크함

- 프로그램이 링크하는 라이브러리는 ldd를 통해 확인 가능
  - libc.so.6 > 표준 C 라이브러리
  - ld-linux-x86-64 > 공유 라이브러리를 로드하는 라이브러리
```
nasir-pc# ldd /bin/echo
        linux-vdso.so.1 (0x00007fff6a3dd000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f7726b6e000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f7726d79000)

nasir-pc# ldd /bin/cat
        linux-vdso.so.1 (0x00007ffdc0b55000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f2037326000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f2037532000)

nasir-pc# ldd /usr/bin/python3
        linux-vdso.so.1 (0x00007fff6176d000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fd6e9aa0000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007fd6e9a7d000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007fd6e9a77000)
        libutil.so.1 => /lib/x86_64-linux-gnu/libutil.so.1 (0x00007fd6e9a72000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007fd6e9923000)
        libexpat.so.1 => /lib/x86_64-linux-gnu/libexpat.so.1 (0x00007fd6e98f5000)
        libz.so.1 => /lib/x86_64-linux-gnu/libz.so.1 (0x00007fd6e98d7000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fd6e9ca0000)
```

### 시스템 콜 래퍼 함수
- libc는 시스템 콜 래퍼함수도 제공
  - 시스템콜은 일반 함수와 달리 고급 언어에서 직접 호출 X, 아키텍처 의존적인 어셈블리 코드를 사용하여 호출
  - 아키텍처마다 다르기에 번거롭고 이식성 떨어짐,
  - libc에서 시스템 콜 호출을 위한 시스템 콜 래퍼함수 제공
- 사용자 프로그램에서 시스템콜 대신 시스템콜 래퍼함수를 호출

### 정적 라이브러리와 공유 라이브러리
- 정적/공유 라이브러리는 기능은 동일하지만 프로그램과 결합하는 방식이 다름
  - 프로그램 생성을 위해 소스코드를 컴파일해 오브젝트 파일 생성
  - 오브젝트 파일이 사용하는 라이브러리를 링크해서 실행파일 생성
    - 정적 라이브러리: 라이브러리에 있는 함수를 프로그램에 포함 시킴
    - 공유 라이브러리: 함수 호출 정보를 프로그램에 포함, 프로그램 실행시 라이브러리 로드 후 호출 

- 정적 라이브러리
  - 900K 정도로 상대적으로 큰 용량
  - 공유 라이브러리 링크되어있지 않음
  - libc를 포함하므로 libc.a 를 삭제해도 동작은 가능 (libc.a?)
    - 다른 (공유라이브러리를 사용하는)프로그램이 있을 수 있으니 실제로는 위험
```
nasir-pc# cc -static -o pause pause.c
nasir-pc# ls -l pause
-rwxr-xr-x 1 root root 871832 Mar 18 03:52 pause
nasir-pc# ldd pause
        not a dynamic executable
nasir-pc# ll pause
-rwxr-xr-x 1 root root 852K Mar 18 03:52 pause

```

- 공유 라이브러리
  - 20K 정도로 상대적으로 작은 용량
  - libc등을 동적 링크
  - libc는 실행시 메모리에 로드 되므로 상대적으로 적은용량
  - 프로그램마다 각 사본 대신 libc를 공유해서 사용함
```
nasir-pc# cc -o pause.so pause.c
nasir-pc# ls -l pause.so
-rwxr-xr-x 1 root root 16696 Mar 18 03:54 pause.so
nasir-pc# ldd pause.so
        linux-vdso.so.1 (0x00007ffea3138000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fc700c9c000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fc700ea1000)
nasir-pc# ll pause.so
-rwxr-xr-x 1 root root 17K Mar 18 03:54 pause.so
```
- 일반적으로는 공유라이브러리 사용
  - 시스템에서 차지하는 크기 줄이기
  - 라이브러리 문제시 개별 프로그램을 재빌드하기보다는, 공유 라이브러리 교체로 해결가능
