# 프로세스 관리(응용편)
4장에 설명한 가상메모리 지식이 없으면 이해가 어려운 프로세스 관리의 다른 기능 설명


##  빠른 프로세스 작성 처리
가상 메모리 기능을 응용해서 프로세스 작성을 빠르게함
- fork() 함수
  - 현재 프로세스를 복제해서 새로운 자식 프로세스 생성
- execve() 함수
  - 현재 프로세스를 새로운 프로그램으로 덮어쓰기 해서 실행

### fork() 함수 고속화: 카피 온 라이트
- Copy On Write(CoW): fork()시 함수 호출이 아닌, 처음 쓰기시 데이터 복사

- 초기엔 부모, 자식 간 동일만 물리페이지 공유
  - fork() 함수 호출 시 부모 프로세스의 메모리를 자식 프로세스에 모두 복사 X, 대신 페이지 테이블만 복사(쓰기권한 X)     
- 데이터 갱신 시 페이지 공유 해제 후 전용 페이지 생성
  - 자식 프로세스가 페이지 데이터 갱신시
  - 쓰기 권한이 없으므로 CPU page fault
  - CPU 커널모드 진입, page fault handler
  - 페이지폴트핸들러는 접속한 페이지를 별도 물리 메모리에 복사
  - 자식 프로세스가 변경하려는 페이지테이블엔트리 부모/자식 각각 ㅂㄴ경

./cow.py
1. 100MiB 메모리 영역 확보 후 모든페이지에 데이터 쓰기
2. 시스템 전체의 물리 메모리, 프로세스의 물리메모리(RSS), 메이저 폴트, 마이너 폴트 출력
3. fork() 함수 호출
4. 자식 프로세스 종료 대기  
    a. 자식 프로세스 대상으로 2.의 정보 출력    
    b. 1.에서 확보한 모든 페이지 접근   
    c. 자식 프로세스 대상으로 2.의 정보 출력    

- 확인사항
  - fork() 함수 실행 후, 쓰기 까지 메모리 영역은 부모/자식간 공유 중?
    - 약 1MiB(1562856-1562128=728KB) 만 증가, 페이지 테이블 복사에 해당
  - 메모리 영역에 쓰기 발생시 시스템 메모리 사용 100MiB 증가 및 페이지 폴트 발생?
    - 약 100MiB(1665168-1562856=102312) 증가 및 페이지폴트 증가 확인
  - 자식프로세스의 RSS 값은 메모리 접근 전 후로 크게 차이가 없음 (약 110MiB)
    - RSS 값은 다른 프로세스와의 공유 여부 모름
    - 공유 페이지에 쓰기(CoW) 이전에는 공유상태지만 각각 보고
    - 따라서 전체 RSS의 총량은 전체 시스템 물리메모리보다 커질 수 있다
```
./cow.py
*** 자식 프로세스 생성 전 ***
free 명령어 실행 결과:
              total        used        free      shared  buff/cache   available
Mem:       16196784     1562128    13754660        3628      879996    14361016
Swap:       4194304           0     4194304
부모 프로세스의 메모리 관련 정보
  RSS  MAJFL  MINFL
112292     0   2106

*** 자식 프로세스 생성 직후 ***
free 명령어 실행 결과:
              total        used        free      shared  buff/cache   available
Mem:       16196784     1562856    13753932        3628      879996    14360288
Swap:       4194304           0     4194304
자식 프로세스의 메모리 관련 정보
  RSS  MAJFL  MINFL
110216     0    657

*** 자식 프로세스의 메모리 접근 후 ***
free 명령어 실행 결과:
              total        used        free      shared  buff/cache   available
Mem:       16196784     1665168    13651620        3628      879996    14257976
Swap:       4194304           0     4194304
자식 프로세스의 메모리 관련 정보
  RSS  MAJFL  MINFL
110288     0  26718
```


![image](https://github.com/user-attachments/assets/80dadd9b-ff8a-4fa5-ac2c-39ff347edc49)

![copy-on-write](https://wizardzines.com/images/uploads/copy-on-write.png)    
[copy-on-write](https://wizardzines.com/comics/copy-on-write/)
- 리눅스에서 프로세스 시작으로 많이 쓰이는 시스템콜은 fork()와 clone()
- 간단히 보기 좋음 > comics! 탭 

### execve() 함수 고속화: Demand paging
- Demand paging: 처음에는 PTE만 생성, 실제 호출하여 사용시 물리 메모리 할당 
  - execve() 호출 시 PTE만 생성되고 실제 메모리 할당 X
  - 프로그램이 엔트리포인트에서 실행을 시작하면 연결 물리 메모리 없어서 page fault
  - page fault 처리로 물리 메모리 할당
  - 이하 프로세스 새로 생성시 반복

![image](https://github.com/user-attachments/assets/a2290e30-f8f2-4be5-8f0a-c62d9497fdf5)    
[[운영체제/OS] 메모리 관리 - 디맨드 페이징과 페이지 부재(Page Fault) Issue](https://studyandwrite.tistory.com/21)

## 프로세스 통신
프로세스 통신 (IPC, Inter-Process Communication)
- 서로 다른 프로세스 간 데이터 공유 및 동기화를 위해 OS가 제공하는 기능


![ICP](https://images.tpointtech.com/blog/images/what-is-inter-process-communication2.png)    
[What is Inter Process Communication?](https://www.tpointtech.com/what-is-inter-process-communication)
- 교재의 공유 메모리, 시그널, 파이프, 소켓, 외에도 다른 IPC 종류가 많다
- 일반적으로 아래 7개를 많이 언급하는 듯함
  1. Pipes
  2. Shared Memory
  3. Message Queue
  4. Direct Communication
  5. Indirect communication
  6. Message Passing
  7. FIFO
- 교재의 시그널, 소켓 등은 별도 방법으로 소개됨
- 누가 시작했는지 몰라도 이미지에 sharred memory로 오타냈더니sharred memory 쓰는 유사이미지 많이보임

### 공유 메모리
./non-shared-memory.py
1. 정수 데이터 1000 생성 및 데이터 출력
2. 자식 프로세스 생성
3. 부모는 자식의 종료 대기, 자식은 1. 의 데이터 2배로 만들고 종료
4. 부모는 데이터 값 출력

- 실제 작동 X
  - fork() 이후 부모 자식은 데이터 공유 X, 한쪽 갱신이 다른쪽 영향 X
  - fork() 호출 직후에는 공유지만 쓰기작업을 하면 별도의 물리 메모리 할당
- 공유메모리(shared memory) 방식을 사용하면 여러 프로세스에 동일 메모리 매핑 가능
```
❯ ./non-shared-memory.py
자식 프로세스 생성전 데이터 값: 1000
자식 프로세스 종료후 데이터 값: 1000
```

./shared-memory.py
- mmap() 시스템콜을 사용하여 공유 메모리 설정
  1. 정수 데이터 1000 생성 및 데이터 출력
  2. 공유 메모리 영역 작성 및 1.의 데이터값을 영역 첫 부분에 저장
  2. 자식 프로세스 생성
  3. 부모는 자식의 종료 대기, 자식은 1. 의 데이터 2배로 만들고 공유메모리 저장 및 종료
  4. 부모는 데이터 값 출력

```
❯ ./shared-memory.py
자식 프로세스 생성 전 데이터 값: 1000
자식 프로세스 종료 후 데이터 값: 2000
```

![image](https://github.com/user-attachments/assets/d39da335-376a-49c7-bcc9-d16091ad5c70)    
[IPC와 Shared Memory](https://dokhakdubini.tistory.com/490), [OS23 - Interprocess Communication | Shared Memory | Message Passing | Signals](https://youtu.be/DpuiwZN-7oc?si=Npl3xh8wejhzRlZ9)    
- shared memory랑 message passing/queue 비교한 자료 많음
- shared memory
  - 장점 처음호출할때만 커널도움 받고 이후엔 안쓰니? 빠름
  - 단점 유저프로세스에서 동기화/접근 제어방법 고려필요
- message passing
  - 커널에 생성되는 공유 메모리 > send, receive로 데이터 주고받기
  - 장점 explicit sharing, 에러에 덜취약(OS,커널이 관리하니까)
  - 단점 느림

### 시그널
커스텀하게 사용할 수 있는 시그널도 존재, 이를 사용하여 프로세스간 진행정도 확인 가능
- 기존) SIGINT, SIGTERM, SIGKILL과 같이 용도가 정해진 시그널
- 이외) SIGUSR1, SIGUSR2 처럼 자유롭게 용도할당이 가능한 시그널도 POSIX에 정의됨
- 이를 통해 통신이 가능하나, 기능이 다양하지 않아 단순작업에만 활용 가능
  - 원시적인 구조여서 시그널 도착여부정도만 확인 가능
  - 데이터 등은 별도 방식을 통해 주고받아야함
- dd 명령어는 SIGUSR1 시그널 시 진척 상황 표시 하도록 되어있음

```
❯ dd if=/dev/zero of=test bs=1 count=1G &
[1] 8034
❯ DDPID=$!
❯ kill -SIGUSR1 $DDPID
9326096+0 records in
9326096+0 records out
9326096 bytes (9.3 MB, 8.9 MiB) copied, 16.0407 s, 581 kB/s%                                                                                                                                  
❯ kill -SIGUSR1 $DDPID
15875325+0 records in
15875325+0 records out
15875325 bytes (16 MB, 15 MiB) copied, 27.2884 s, 582 kB/s
❯ kill -SIGUSR1 $DDPID
26193175+0 records in
26193175+0 records out
26193175 bytes (26 MB, 25 MiB) copied, 45.0779 s, 581 kB/s
❯ kill %1
[1]  + 8034 terminated  dd if=/dev/zero of=test bs=1 count=1G  
```

![signal](https://wizardzines.com/images/uploads/signals.png)    



### 파이프
pipe(|): 프로그램끼리의 처리 결과 연계
- free | awk '(NR==2){print $2}'
  - free 명령어 출력 중 두번째 줄(NR,Number of Records)의 두번째 필드($2) 출력
- bash 쉘의 | 는 단방향 전달만 되지만 교재에는 양방향 통신도 된다해서 pipe()시스템 콜은 다른가했더니..
  - GPT는 pipe 두번써서 양방향 통신하라고하네요
```
❯ free
              total        used        free      shared  buff/cache   available
Mem:       16196784     3019900      160408        3616    13016476    12841076
Swap:       4194304         524     4193780
❯ free | awk '(NR==2){print $2}'
16196784
```

![854](https://github.com/user-attachments/assets/d453dac1-f1f1-4ede-9e59-1a8ff4deefbd)      
[OS24 - Pipes | Interprocess Communication](https://youtu.be/s-cBPllAYD8?si=CWmAO6vmCHD3SB8S&t=501)      
ㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋ

### 소켓
소켓(socket): 프로세스간 데이터를 주고받기 위한 양방향 통신 채널
- 크게 2종류의 소켓 존재
  - 유닉스 도메인 소켓(UNIX domain socket)
    - 같은 기기의 프로세스간 통신
  - TCP,UDP 소켓
    - 인터넷 프로토콜 스위트(Internet protocol suite) 또는 TCP/IP에 따라 다수 프로세스간 통신
    - UNIX Socket에 비해 속도가 느리지만 다른 기기의 프로세스간 통신에 사용 가능
- 친숙한 소켓: mysqld, docker daemon

![unix domain sockets](https://wizardzines.com/images/uploads/unix-domain-sockets.png)    
- 

## 배타적 제어
배타적 제어(exclusive control): 특정 자원에 한번에 하나의 처리만 접근 가능하도록 관리
- Database의 스토리지 같이 시스템 자원 중 동시 접근하면 안되는것들이 많음
- 배타적 제어를 통해 방지하지만 직관적이지않아 어려우므로 File lock으로 설명

./inc.sh
- 특정 파일을 읽어서 숫자1을 더하고 종료
  - 순차적으로 실행하면 정상 작동
  - 병렬적으로 실행하면 기대하지 않은 값 출력
    - 서로다른 프로그램이 파일을 읽고 업데이트하기 때문
```
❯ echo 0 > count
❯ for ((i=0;i<1000;i++)) ; do ./inc.sh ; done
❯ cat count
1000

❯ echo 0 > count
❯ for ((i=0;i<1000;i++)) ; do ./inc.sh & done; for ((i=0;i<1000;i++)); do wait; done
...생략...
❯ cat count
8
```

상호 배제(mutual exclusion): 여러 프로세스나 스레드가 공유 자원에 동시에 접근하지 못하도록 서로 배타적으로 접근하게 하는 것
- 한번에 하나의 프로그램에서만 파일값 처리를 해야함
- 2가지 용어 정리
  1. 크리티컬 섹션(critical section), 임계 구역: 동시에 실행되면 안되는 처리 흐름
    - inc.sh의 경우 count 값을 읽고 1을 더해 다시 쓰는 처리
  2. 아토믹(atomic)처리: 시스템 외부에서 하나의 처리로 다루어야하는 처리 흐름
    - inc.sh의 경우 크리티컬 섹션이 아토믹하다면, 프로그램 A 동작 중 프로그램 B는 끼여들기 불가

./inc-wrong-lock.sh
- 처리 시작 전 lock 파일의 유무를 확인, 없을때만 생성해서 들어가고 종료 후 lock파일 삭제
- 실패, 아토믹 하지 않기 때문
  - touch 명령어는 원자적이지 않음
  - test > touch 명령어 간의 시간차 동안 다수 프로그램 실행됨
- 아토믹한 처리하고 싶으면 지원하는 기능을 써라
```
❯ echo 0 > count
❯ for ((i=0;i<1000;i++)) ; do ./inc-wrong-lock.sh & done; for ((i=0;i<1000;i++)); do wait; done
...생략...
❯ cat count
cat count
5
```

File lock
- flock() / fcntl() 시스템 콜 사용하여 가능
  1. 파일이 lock 상태인지 확인
  2. lock 상태라면 시스템 콜 실패
  3. unlock상태면 lock 상태로 변경 후 시스템 콜 성공
- 명령어로는 flock로 가능
  - inc-lock.sh

```
echo 0 > count
for ((i=0;i<1000;i++)) ; do ./inc-lock.sh & done; for ((i=0;i<1000;i++)); do wait; done
cat count
1000
```

## 돌고 도는 배타적 제어
- File lock은 C언어 같은 고급언어가 아닌, 기계어 계층에서 구현됨
 - 아래의 예시와 유사하나, 아래는 동작 X
 - 주석코멘트 부분이 아토믹하지않아, load r0 mem 동시 실행시 동시 실행 가능
- CPU 아키텍처에선 아토믹한 해당 부분 처리를 위한 명령어 존재
  - compare and exchange, compare and swap
- 고급 언어 수준의 배타적 제어 실행시, CPU단의 처리 보다 시간 및 메모리 사용량 증가
  - 피터슨 알고리즘(Peterson's algorithm) 참조 
```
# 가상의 어셈블리 언어를 사용한 lock 구현
start:
  # mem 주소의 메모리를 읽어서 r0 레지스터에 저장, mem 내용이 1> lock / mem 내용이 0>unlock
  load r0 mem
  # r0 == 0?
  test r0
  # r0 = 0이면(unlock) enter label로 점프
  jmpz enter
  # r0 != 0 이면(lock) start label로 복귀
  jmp start
enter:
  # mem에 1 쓰기, 이때 lock
  store mem 1 
...
<크리티컬 섹션>
...
  # mem에 0 쓰고 unlock
  store mem 0
```

## 멀티 프로세스와 멀티 스레드
- CPU 멀티 코어화 > 프로그램 병렬동작 중요성 UP
- 병렬 동작의 2가지 방식
  1. 다른 작업을 하는 프로그램 동시 동작
  2. 특정 목정의 하나 프로그램을 여러 흐름으로 분할하여 실행
- 2. 의 경우와 같은 분할 실행 방법 확인, 크게 두가지
  - 멀티 프로세스(multi process)
    - fork(),execve() 같은 함수 사용, 필요한 만큼의 프로세스 생성 후 프로세스간 통신
  - 멀티 스레드(multi thread)
    - 프로세스 내부에 여러개의 흐름 생성
    - 단일 스레드 > 싱글스레드 프로그램, 다수 스레드 > 멀티스레드 프로그램

- 스레드 기능 제공을 위한 다양한 방법
  - POSIX > POSIX 스레드라는 스레드 조작용 API
  - linux > libc 등으로 POSIX 스레드 제어

- 멀티스레드 장점
  - 페이지 테이블 복사 필요 X > 생성 시간 짧음
  - 다양한 자원을 프로세스내부의 모든 스레드 공유 > 메모리와같은 자원소비량 down
  - 모든 스레드가 메모리 공유 > 협조하여 동작 용이
- 멀티스레드 단점
  - 하나의 스레드의 정애가 모든 스레드의 영향
    - 하나의 스레드가 비정상적주소 참조로 인해 이상 종료 > 프로세스 전체 이상 종료
  - 각 스레드가 호출하는 처리가 멀티스레드프로그램에서 불러도 문제없는지(thread safe) 확인
    - 내부적으로 전역 변수를 배타적 제어없이 접근하는 처리 > thread safe하지 않음
- 이러한 문제를 직접 다루기는 어렵기 때문에 지원기능을 사용
  - ex. Go에는 goroutine 언어 내장기능으로 스레드 처리

#### 커널스레드 & 사용자 스레드
- 스레드 구현방식 2가지
  - kernel thread > 커널이 직접 관리하는 스레드로, 스케줄링과 실행을 커널이 담당
    - 프로세스 스케줄시 스케줄링 대상은 프로세스가 아닌 커널 스레드
      - 프로세스 생성 시 커널은 하나의 커널 스레드 작성
      - 이 프로세스에서 clone() 시스템 콜 호출하면 신규 스레드에 대응하는 또 다른 커널 스레드 생성
      - 프로세스 내부의 스레드는 서로다른 논리 CPU에서 동작
        - fork() 호출시나 스레드 작성시나 모두 clone() 시스템콜 사용
      - clone() 시스템콜은 기존 커널 스레드와, 커널스레드 간의 공유자원 설정
        - 프로세스 생성(fork() 호출)은 가상 주소 공간을 공유하지 않음
        - 스레드 생성은 가상 주소 공간을 공유
      - ps -eLF로 커널스레드의 목록 조회가능
        - LWP(Lightweight Process)
          - 커널이 스케줄링한 사용자 스레드의 대표 실행 단위
          - 프로세스 생성시 만들어진 LWP ID는 PID와 동일
        - PID 377 과 같은 cron은 싱글스레드
        - PID 352와 같은 ModemManager는 3개의 커널스레드(352,365,374)
  - user thread > 사용자 공간에서 구현된 스레드로, 커널은 하나의 프로세스로만 인식
    - clone() 시스템 콜 사용X, 사용자 공간에서 스레드 라이브러리로 구현한것이 사용자 스레드
    - 실행할 명령에 대한 정보는 스레드 라이브러리 안에 저장되어있음
    - 스레드가 I/O호출등으로 인해 대기 상태 > 스레드 라이브러리가 다른 스레드로의 실행 전환
    - 프로세스내 다수 사용자 스레드가 있어도, 커널에서는 하나의 커널 스레드로 보임 > 동일한 논리 CPU에서 실행
```
ps -eLF
UID          PID    PPID     LWP  C NLWP    SZ   RSS PSR STIME TTY          TIME CMD
root           1       0       1  0    1 42569 13268   3 19:50 ?        00:00:00 /sbin/init
root           2       1       2  0    2   696  1932   0 19:50 ?        00:00:00 /init
root           2       1       9  0    2   696  1932   9 19:50 ?        00:00:00 /init
...
root         352       1     352  0    3 78772 11648   9 19:50 ?        00:00:00 /usr/sbin/ModemManager
root         352       1     365  0    3 78772 11648   0 19:50 ?        00:00:00 /usr/sbin/ModemManager
root         352       1     374  0    3 78772 11648   8 19:50 ?        00:00:00 /usr/sbin/ModemManager
root         377       1     377  0    1  2137  2912   6 19:50 ?        00:00:00 /usr/sbin/cron -f
root         379       1     379  0   17 389669 36300  1 19:50 ?        00:00:00 /usr/sbin/libvirtd
root         379       1     422  0   17 389669 36300  5 19:50 ?        00:00:00 /usr/sbin/libvirtd
root         379       1     423  0   17 389669 36300 11 19:50 ?        00:00:00 /usr/sbin/libvirtd
...
```




- 커널 스레드와 사용자 스레드 차이점
  - 물리적 배치상태 관점
    - 스레드 정보를...
      - 커널스레드면 커널이 관리
      - 사용자 스레드면 프로세스가 관리
  - 프로세스 스케줄링 관점
    - 커널 스레드는 모든 스레드를 동등하게 취급, 스레드별 CPU 할당
    - 사용자 스레드는 프로세스의 스레드 구분 불가, 프로세스 별로 CPU 할당
      - 멀티스레드 프로세스의 경우 각 스레드 배분우선순위는 스레드 라이브러리가 담당
  
  - 커널스레드는..
    - 논리 CPU가 여러개일때 동시 실행이 가능하다는 장점
    - 생성 비용, 스레드 실행 전환 비용은 높음
  - goroutine은 사용자스레드로 구현됨

- 프로세스가 아닌 커널이 직접 스레드를 만들기도함
  - ps aux 결과 > [kthreadd], [rcu_gp] 와 같이 COMMAND 필드 문자열이 []로 감싸진경우
  - - WSL2에서는 안보이는듯..
- 커널 프로세스의 트리 구조 > kthreadd가 루트로 동작
  - 리눅스 커널 실행시 초기단계(init?)에 PID=2인 kthreadd 기동
  - 이후 kthreadd가 자식 커널 스레드 기동
  - kthreadd <> 각종 커널스레드 관계는 init과 각종 프로세스와 비슷

```
❯ ps aux | grep '\['
nasir       5254  0.0  0.0   8172   728 pts/6    S+   20:07   0:00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn --exclude-dir=.idea --exclude-dir=.tox --exclude-dir=.venv --exclude-dir=venv \[
```


<p align="center">
  <img src="https://i.sstatic.net/B0aVa.png" alt="avg-tat-24" width="45%" />
  <img src="https://i.sstatic.net/kGZ7H.png" alt="throughput-24" width="45%" />
</p>
[What are the relations between processes, kernel threads, lightweight processes and user threads in Unix](https://unix.stackexchange.com/questions/472324/what-are-the-relations-between-processes-kernel-threads-lightweight-processes)

- 왜 n to n 자료만 많이 보이는가
