# 메모리 관리 시스템
- 리눅스에서는 커널의 메모리 관리 시스템 Memory management subsystem 을 통해 설치된 메모리를 관리함
- 각 프로세스 및 커널 자체도 메모리를 사용

## 메모리 관련 정보 수집
free 명령어로 확인가능
- total: 시스템에 설치된 전체 메모리 용량, 15Gi
- free: 명목상 비어있는 메모리
- buff/cache: 버퍼, 캐시, 페이지 캐시(8장)의 메모리, 시스템의 빈 메모리(free)가 줄면 시스템이 해제시킴
- available: 실제로 사용가능한 메모리. free + 해제가능한 커널내부 메모리 영역(ex.page cache)
- used: 시스템이 사용중인 메모리에서 buff/cache 를 제외한 값
  
```
free

              total        used        free      shared  buff/cache   available
Mem:       16196780      894228    14422440        3628      880112    15024372
Swap:       4194304           0     4194304

free -h
              total        used        free      shared  buff/cache   available
Mem:           15Gi       867Mi        13Gi       3.0Mi       859Mi        14Gi
Swap:         4.0Gi          0B       4.0Gi
```

![image](https://github.com/user-attachments/assets/52489469-11f9-48b7-8db9-0ddcbb4a9a87)    
[쿠버네티스가 쉬워지는 컨테이너 이야기 — memory편](https://medium.com/@7424069/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4%EA%B0%80-%EC%89%AC%EC%9B%8C%EC%A7%80%EB%8A%94-%EC%BB%A8%ED%85%8C%EC%9D%B4%EB%84%88-%EC%9D%B4%EC%95%BC%EA%B8%B0-memory%ED%8E%B8-62cafabfd160)
- buff/cache의 일부는 종료되면서 메모리 할당 해제가 가능하다

  
### used
- used 값은 프로세스+커널이 사용하는 메모리를 포함    
- 커널을 제외하고 프로세스 메모리 관점에서 분석

- used 값은 프로세스 메모리 사용량에 따라 증가, 프로세스 종료되면 메모리해제

./memuse.py
- 프로그램 실행시 used 값이 약 80MB 증가
- 구체적인 값은 다른프로그램의 영향도 있으므로 중요 X
- 프로그램 실행 중 메모리를 사용하면 > 시스템 전체 메모리 사용량이 증가한다
- 사용 종료되면 실행 전과 거의 같은 값으로 돌아옴
```
./memuse.py
메모리 사용 전의 전체 시스템 메모리 사용량을 표시합니다.
              total        used        free      shared  buff/cache   available
Mem:       16196780      922220    14392164        3632      882396    14996268
Swap:       4194304           0     4194304
메모리 사용 후의 전체 시스템 메모리 남은 용량을 표시합니다.
              total        used        free      shared  buff/cache   available
Mem:       16196780     1002036    14312348        3632      882396    14916452
Swap:       4194304           0     4194304

free
              total        used        free      shared  buff/cache   available
Mem:       16196780      907600    14406344        3628      882836    15010912
Swap:       4194304           0     4194304
```

### buff/cache
buff/cache 값은 page cache, buffer cache(8장)이 사용하는 메모리 값
- 접근 속도가 느린 저장장치에 있는 파일데이터를,
- 접근 속도가 빠른 메모리에 일시적으로 저장해 빠르게 접근가능하게하는 커널 기능
- 저장 장치의 파일 데이터를 메모리에 캐시함

./buff-cache.sh
- free 명령어 실행 (사전 상태 확인)
- 1GiB 파일 작성 (페이지 캐시 생성)
- free 명령어 실행 (buff/cache 확인)
- 파일 삭제 (페이지 캐시 해제) *수동으로 sleep 5 추가
- free 명령어 실행 (buff/cache 확인)
```
파일 작성 전의 시스템 전체 메모리 사용량을 표시합니다.
              total        used        free      shared  buff/cache   available
Mem:       16196780     2439516    13373188        3640      384076    13476296
Swap:       4194304           0     4194304
1GB 파일을 새로 작성합니다. 커널은 메모리에 1GB 페이지 캐시 영역을 사용합니다.
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 0.737705 s, 1.5 GB/s
페이지 캐시 사용 후의 시스템 전체 메모리 사용량을 표시합니다.
              total        used        free      shared  buff/cache   available
Mem:       16196780     1805092    12931768        3640     1459920    14096988
Swap:       4194304           0     4194304
파일 삭제 후, 즉 페이지 캐시 삭제 후의 시스템 전체 메모리 사용량을 표시합니다.
              total        used        free      shared  buff/cache   available
Mem:       16196780     2348024    13464644        3640      384112    13567776
Swap:       4194304           0     4194304
```

### sar 명령어를 사용하여 메모리 관련 정보 수집
`sar -r` 명령어를 통해 메모리 관련 통계 정보 획득 가능
- 한줄에 필요 정보가 담겨있어 수집 간편

| free   | sar -r |
|--------|------|
| total   | 해당사항없음   |
| free   | kbmemfree   |
| buff/cache   | kbuffers+kbcached   |
| available | 해당 사항 없음 |


```
sar --help  | grep -A 1 "\-r"
       -r [ ALL ]
               Memory utilization statistics [A_MEMORY]

sar -r 1 5
Linux 5.15.167.4-microsoft-standard-WSL2 (nasir-pc)     04/07/25        _x86_64_        (12 CPU)

20:46:15    kbmemfree   kbavail kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty
20:46:16     14562376  14655360   1192924      7.37      4292    311980   2523296     12.37     87516   1125668         4
20:46:17     14562928  14655936   1192308      7.36      4292    311980   2498532     12.25     87532   1122284         4
20:46:18     14563184  14656192   1192056      7.36      4300    311972   2497400     12.25     87520   1121872         4
20:46:19     14563484  14656492   1191752      7.36      4300    311980   2498532     12.25     87524   1123084        44
20:46:20     14564452  14657452   1190808      7.35      4300    311980   2497400     12.25     87520   1121800         0
Average:     14563285  14656286   1191970      7.36      4297    311978   2503032     12.28     87522   1122942        11

sar -h -r 1 5
Linux 5.15.167.4-microsoft-standard-WSL2 (nasir-pc)     04/07/25        _x86_64_        (12 CPU)

20:48:01    kbmemfree   kbavail kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty
20:48:02        13.9G     14.0G      1.2G      7.5%      4.7M    306.2M      2.4G     12.3%     86.1M      1.1G      4.0k
20:48:03        13.9G     14.0G      1.2G      7.5%      4.8M    306.2M      2.4G     12.3%     86.1M      1.1G     40.0k
20:48:04        13.9G     14.0G      1.2G      7.5%      4.8M    306.2M      2.4G     12.2%     86.1M      1.1G     40.0k
20:48:05        13.9G     14.0G      1.2G      7.5%      4.8M    306.2M      2.4G     12.2%     86.1M      1.1G     40.0k
20:48:06        13.9G     14.0G      1.1G      7.4%      4.8M    306.2M      2.4G     12.2%     86.1M      1.1G      0.0k
Average:        13.9G     14.0G      1.2G      7.5%      4.8M    306.2M      2.4G     12.2%     86.1M      1.1G     24.8k
```

## 메모리 재활용 처리
- 시스템 부하가 높아지면 free 메모리 감소
- 재활용한 가능한 메모리 영역을 해제하여 free 값 늘리기
  - 디스크에서 데이터를 읽은 뒤 변경되지 않은 페이지 캐시
  - 동일 데이터가 디스크에 있기해 해제해도 문제X


### 프로세스 삭제와 메모리 강제 해제
- 재활용 메모리를 해제하여도 부족하면, Out Of Memory(OOM) 상태
- 적당한 프로세스를 골라서 강제 종료하는 OOM Killer 작동

dmesg 명령어를 통해 커널 로그를 확인 하면 oom-kill 이벤트 확인 가능
- 동시실행 중인 프로세스를 줄여 메모리 사용량 줄이기 또는 메모리 추가 설치

메모리 용량이 충분한데 동작한다면, 메모리 누수(memory leak) 의심
- 프로세스 메모리 사용량을 정기적으로 모니터링하여 시간에따른 메모리 사용량 증가 프로세스 찾기
- ps aux > RSS 필드 모니터링


![image](https://github.com/user-attachments/assets/80dd5345-04b3-4cc7-98b8-00eca45a80a9)    

[쿠버네티스가 쉬워지는 컨테이너 이야기 — memory편](https://medium.com/@7424069/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4%EA%B0%80-%EC%89%AC%EC%9B%8C%EC%A7%80%EB%8A%94-%EC%BB%A8%ED%85%8C%EC%9D%B4%EB%84%88-%EC%9D%B4%EC%95%BC%EA%B8%B0-memory%ED%8E%B8-62cafabfd160)
- oom_killer는 oom_badness()를 통해 계산된 oom_score에 따라 프로세스 종료
  - oom_score > 커널이 자동 계산한 점수, /proc/<pid>/oom_score 로 확인가능, 높을수록 먼저 죽임
  - oom_score_adj > 사용자가 수동으로 우선순위 조절 가능

[LINUX 메모리 부족과 oom_killer 문제에 대한 이해와 대응](https://m.blog.naver.com/hanajava/223133077977)    
1. 특정 프로세스를 죽임으로써, 최소한 양의 프로세스만 잃을 수 있어야 합니다.
2. 많은 메모리를 회수할 수 있어야 합니다.
3. 프로세스중 Leak 가 발생되지 않는 프로세스는 선택하지 않습니다.
4. 사용자가 특별히 지정한 프로세스를 선택합니다.

​

## 가상 메모리
가상 메모리는 하드웨어와 소프트웨어(커널)을 연동하여 구현
1. 가상 메모리가 없을 때 생기는 문제점
2. 가상 메모리 기능
3. 가상 메모리로 문제점 해결

### 가상 메모리가 없을때 생기는 문제점
1. 메모리 단편화
2. 멀티 프로세스 구현의 어려움
3. 비정상적 메모리 접근

#### 메모리 단편화
프로세스 생성후 메모리 확보/해제 반복 시 메모리 단편화(fragmentation of memory) 발생
- 프로세스 메모리 간 용량을 묶어서 관리 불가
  - 메모리 확보 시마다 몇개의 메모리 영역으로 나뉘어져 있는지 관리 불편
  - 크기가 간격 용량보다 큰 데이터 묶음 사용 불가 (ex. 100바이트X3이지만 300바이트 배열 사용 불가)

![image](https://github.com/user-attachments/assets/ebbb9c91-d78b-4e89-aaf6-8fa3750e3f19)    
[Memory Fragmentation in operating system](https://er.yuvayana.org/memory-fragmentation-in-operating-system/)
- 위 이미지에서 회색영역 10KB, 5KB, 15KB, ... 은 죽은 공간

#### 멀티 프로세스 구현의 어려움
- 프로세스 A를 실행시, 코드영역이 300~400(399), 데이터가 400~500으로 매핑
- 동일 파일의 프로세스 B 실행 시, 300~500영역 프로세스 A 가 사용중
  - 다른 장소에 매핑하여도 명령 및 데이터가 가리키는 메모리 주소가 달라 동작 X
- 동시 실행을 위해 모든 프로그램의 배치 장소가 겹치지 않도록 의식해야함

#### 비정상적인 메모리 접근
- 프로세스가 커널 또는 다른 프로세스의 메모리를 지정하면 접근가능하여 권한 문제 
- 데이터 누출 또는 손상의 위험성

### 가상 메모리 기능
가상 메모리(virtual memory)는 프로세스가 메모리 접근시 `직접 접근` 대신 가상 주소(virtual address)를 사용하여 `간접 접근`
- 실제 주소는 물리 주소(physical address), 접근가능 범위는 주소 공간(address space)
- 2장의 `readelf` 또는 `cat /proc/<pid>/maps` 는 모두 가상 주소
- 프로세스에서 실제 메모리, 물리주소를 직접 지정하는 방법은 X
  
![image](https://github.com/user-attachments/assets/815b6d49-bc62-4974-9724-e6134097c590)    
[가상 메모리](https://ko.wikipedia.org/wiki/%EA%B0%80%EC%83%81_%EB%A9%94%EB%AA%A8%EB%A6%AC)
- 가상 메모리를 사용 시, 프로세스가 활성되면 RAM(물리 메모리), 비활성화된 프로세스는 디스크에 상주한다?

#### 페이지 테이블
- page table: 가상 주소를 물리 주소로 변환하기 위해 커널 메모리 내부에 저장되어있음
  - CPU는 모든 메모리를 페이지 단위로 쪼개서 관리
  - 주소는 페이지 단위로 변환
- page table entry(PTE): 페이지 테이블에서 한 페이지에 대응하는 데이터
  - 가상 주소 및 물리 주소 대응 정보 포함

![image](https://github.com/user-attachments/assets/2ab37b28-b6b5-4013-93f4-61fa8b7d4bea)    
[Difference between page table and inverted page table](https://cs.stackexchange.com/questions/66698/difference-between-page-table-and-inverted-page-table)
- 각 프로세스 (Pi, Pj)의 Page 0는 페이지 테이블에 의해 각 물리 메모리 1,2를 할당받는다


관리 주체
- 페이지 테이블은 커널이 작성 (프로세스 생성 시)
- 프로세스가 가상 주소 접근 시 물리 주소로 변환은 CPU의 작업

존재하지 않는 가상 주소에 접근한다면? > page fault
- 가상 주소 공간 크기는 고정 값
- 페이지 테이블 엔트리의 페이지에 대응하는 물리 메모리 존재여부 관리데이터가 별도 존재
- 물리 메모리가 할당되지 않은 가상 주소에 접근시 페이지 폴트(page fault) 예외(exception) 발생
  - 예외) 실행중인 코드 중간에 별도의 처리 실행
- 페이지 폴트 발생 시, 실행중인 명령이 중단되고 커널 메모리의 페이지 폴트 핸들러 동작
  - 커널의 비정상적인 메모리 접근 감지
  - SIGSEGV 시그널 송신 > 보통은 강제종료
    - Segmentation Violation Signal

![image](https://github.com/user-attachments/assets/a2290e30-f8f2-4be5-8f0a-c62d9497fdf5)    
[[운영체제/OS] 메모리 관리 - 디맨드 페이징과 페이지 부재(Page Fault) Issue](https://studyandwrite.tistory.com/21)
- page fault는 trap이라고도 불린다
1. 프로세스는 자신이 사용하고자 하는 페이지가 page table에 존재하는지 확인한다. 
2. 만약 physical memory에 사용하려는 page가 없으면, Page fault exception이 발생하고 커널모드로 전환된다. 
3. OS는 page table entry의 정보 속에서 Backing storage의 어느 위치에 page가 존재하는지 확인한다. 
4. 필요한 페이지를 physical memory에 load한다. 
5. page table를 다시 업데이트 해준다. 
6. 그리고 이제 원래 하려고 했던 instruction을 수행해준다.
  
`./segv.go`
1. 비정상 주소 접근 전 '비정상 메모리 접근 전' 출력
2. 반드시 접근이 실패하는 nil 주소에 아무 값(코드에선 0) 쓰기
3. 비정상 주소에 접근 후 '비정상 메모리 접근 후' 출력

- C언어나 go언어처럼 메모리 주소를 직접 다루는 언어는 SIGSEGV로 인한 강종 종종발생
- python과 같이 직접 안다루는 언어는 잘 발생 X, but 프로그래밍 언어 처리 또는 C언어 라이브러리 버그등으로 인해 발생 가능
```
go build segv.go

./segv
비정상 메모리 접근 전
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x48cf0b]

goroutine 1 [running]:
main.main()
        /home/nasir/2025-linux-for-beginner/04/segv.go:9 +0x7b
```

### 가상 메모리로 문제 해결하기

#### 메모리 단편화
프로세스의 페이지테이블을 사용해,
- 물리 메모리상의 단편화된 영역이라도
- 프로세스 가상 주소 공간에서는 커다란 하나의 영역으로 다루기 가능

![image](https://github.com/user-attachments/assets/9ab2b1ba-9cb7-4463-b993-f54ab507310b)    
[Introduction to Paging](https://os.phil-opp.com/paging-introduction/#paging)
- 비연속적인 물리 메모리 구간이지만 가상메모리로 할당하여 사용가능


#### 멀티 프로세스 구현의 어려움
- 가상 주소 공간은 프로세스마다 생성됨
- 멀티 프로세스 환경에서 각자의 프로그램의 주소 중복 회피 가능

#### 비정상적 메모리 접근
- 가상 주소 공간이 있다 > 실제 메모리 영역은 모른다
- 특정 프로세스에서 다른 프로세스의 비정상적인 접근 불가

## 프로세스에 새로운 메모리 할당하기
커널이 프로세스에 메모리 할당하는 일반적인 시스템 콜 흐름
1. 프로세스는 시스템 콜 호출로 커널에 필요한 메모리 요청
2. 커널은 시스템의 빈 메모리에서 요청 메모리 영역 확보
3. 커널은 확보한 메모리 영역을 프로세스의 가상 주소 공간에 매핑(페이지테이블)
4. 커널은 가상 주소 공간의 시작위치를 프로세스에게 리턴

그러나 메모리를 바로 사용하기보다는 간격이 있어 메모리 확보 절차는 2단계로 진행
1. 메모리 영역 할당: 가상 주소 공간에 새로운 접근 가능 메모리 영역 매핑
2. 메모리 할당: 확보한 메모리 영역에 물리 메모리 할당

### 메모리 영역 할당: mmap() 시스템 콜
`mmap()` 시스템 콜: 동작 중인 프로세스에 새로운 메모리 영역 할당
- 메모리 영역 크기를 지정하는 인수 존재
- 커널 메모리 관리 시스템이 프로세스의 페이지 테이블 변경
- 요청된 크기만큼의 영역을 페이지 테이블에 추가 매핑
- 매핑된 영역의 시작 주소를 프로세스에게 돌려줌

`./mmap.go`
1. 프로세스의 메모리 매핑 정보 (/proc/<pid>/maps) 출력
2. mmap() 시스템 콜로 1GiB  메모리 요청
3. 메모리 매핑 정보 표시

- 각 줄은 개별 메모리 영역, 첫 번째 필드가 메모리 영역
- 새로운 메모리 확보 전
  - 7f266c37e000-7f266e62f000 가 기존 메모리 영역
- 새로운 메모리 영역 확보 후
  - 7f262c37e000-7f266e62f000 로 확장
  - 7f266c37e000-7f262c37e000 = 1GiB 일것

```
./mmap
*** 새로운 메모리 영역 확보 전 메모리 맵핑 ***
00400000-004aa000 r-xp 00000000 08:20 161586                             /home/nasir/2025-linux-for-beginner/04/mmap
004aa000-00583000 r--p 000aa000 08:20 161586                             /home/nasir/2025-linux-for-beginner/04/mmap
00583000-0059a000 rw-p 00183000 08:20 161586                             /home/nasir/2025-linux-for-beginner/04/mmap
0059a000-005b8000 rw-p 00000000 00:00 0 
c000000000-c004000000 rw-p 00000000 00:00 0 
7f266c37e000-7f266e62f000 rw-p 00000000 00:00 0 
7ffef4cf9000-7ffef4d1b000 rw-p 00000000 00:00 0                          [stack]
7ffef4d9a000-7ffef4d9e000 r--p 00000000 00:00 0                          [vvar]
7ffef4d9e000-7ffef4da0000 r-xp 00000000 00:00 0                          [vdso]

*** 새로운 메모리 영역: 주소 = 0x7f262c37e000, 크기 = 0x40000000 ***

*** 새로운 메모리 영역 확보 후 메모리 매핑 ***
00400000-004aa000 r-xp 00000000 08:20 161586                             /home/nasir/2025-linux-for-beginner/04/mmap
004aa000-00583000 r--p 000aa000 08:20 161586                             /home/nasir/2025-linux-for-beginner/04/mmap
00583000-0059a000 rw-p 00183000 08:20 161586                             /home/nasir/2025-linux-for-beginner/04/mmap
0059a000-005b8000 rw-p 00000000 00:00 0 
c000000000-c004000000 rw-p 00000000 00:00 0 
7f262c37e000-7f266e62f000 rw-p 00000000 00:00 0 
7ffef4cf9000-7ffef4d1b000 rw-p 00000000 00:00 0                          [stack]
7ffef4d9a000-7ffef4d9e000 r--p 00000000 00:00 0                          [vvar]
7ffef4d9e000-7ffef4da0000 r-xp 00000000 00:00 0                          [vdso]
```

### 메모리 할당: Demand paging
Demand paging: 할당된 가상 메모리 페이지에 첫 접근 시 물리 메모리 할당
- mmap() 시스템 콜 호출 시 메모리 영역만 할당(페이지 테이블 엔트리), 첫 접근시 물리 메모리 할당
- Demand paging 구현을 위해 메모리 관리 시스템이 페이지마다 물리 메모리 할당 여부 관리
  1. 프로세스가 페이지 접근
  2. 페이지 폴트 발생
  3. 커널의 페이지 폴트 핸들러 동작 > 대응하는 물리 메모리 할당
- 페이지 폴트 핸들러
  - 페이지 테이블 엔트리 X > 프로그램에 SIGSEGV
  - 페이지 테이블 엔트리 O, 물리 메모리 할당 X > 새로운 메모리 할당

![image](https://github.com/user-attachments/assets/c60dbf35-ae84-4be4-b388-81b5a1f4ba9f)

[Demand Paging In Operating System](https://er.yuvayana.org/demand-paging-in-operating-system/)
- demand paging은 'disk swapping을 사용한 paging' 처럼 시스템 부하를 줄이기 위한 방법
  - 불필요하면 디스크에 내려두고 필요하면 메모리에 로드

`./demand-paging.py`
1. '새로운 메모리 확보 전' 출력 및 입력 대기
2. 100MiB 메모리 영역 확보
3. '새로운 메모리 영역 확보' 출력 및 입력 대기
4. 새로운 메모리 영역을 1페이지씩 접근하며, 10MiB마다 진척도 출력
5. '모든 메모리 영역 접근' 출력 및 입력 대기

#### 시스템 전체 메모리 사용량 변화 확인
- `sar -r` 사용
  - 20:42:31 > 메모리 확보 전
  - 20:42:32 ~ 41 > 메모리 확보 및 접근
  - 20:42:41 ~ > 메모리 할당 해제
- 기간 내 kbmemused 증가 확인
```
❯ sar -h -r 1 > sar.log &
[1] 15984
❯ ./demand-paging.py
20:42:31: 새로운 메모리 영역 확보 전. 엔터 키를 누르면 100메가 새로운 메모리 영역을 확보합니다: 

20:42:32: 새로운 메모리 영역을 확보했습니다. 엔터 키를 누르면 1초당 1MiB씩, 합계 100MiB 새로운 메모리 영역에 접근합니다: 

20:42:32: 10 MiB 진행중
20:42:33: 20 MiB 진행중
20:42:34: 30 MiB 진행중
20:42:35: 40 MiB 진행중
20:42:36: 50 MiB 진행중
20:42:37: 60 MiB 진행중
20:42:38: 70 MiB 진행중
20:42:39: 80 MiB 진행중
20:42:40: 90 MiB 진행중
20:42:41: 새롭게 확보한 메모리 영역에 모두 접근했습니다. 엔터 키를 누르면 종료합니다: 

❯ kill %1
[1]  + 15984 terminated  sar -h -r 1 > sar.log                                                                                     
❯ cat sar.log
Linux 5.15.167.4-microsoft-standard-WSL2 (nasir-pc)     04/08/25        _x86_64_        (12 CPU)

20:42:29    kbmemfree   kbavail kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty
20:42:30        13.6G     13.8G      1.3G      8.7%     12.7M    351.8M      2.4G     12.3%    132.6M      1.3G     60.0k
20:42:31        13.6G     13.8G      1.3G      8.7%     12.7M    351.8M      2.4G     12.3%    132.6M      1.3G     64.0k
20:42:32        13.6G     13.8G      1.3G      8.7%     12.8M    351.8M      2.5G     12.8%    132.6M      1.3G     80.0k
20:42:33        13.6G     13.8G      1.3G      8.7%     12.8M    351.8M      2.5G     12.8%    132.6M      1.3G     80.0k
20:42:34        13.6G     13.8G      1.4G      8.8%     12.8M    351.8M      2.5G     12.8%    132.7M      1.3G     80.0k
20:42:35        13.6G     13.7G      1.4G      8.9%     12.8M    351.8M      2.5G     12.8%    132.7M      1.3G     80.0k
20:42:36        13.6G     13.7G      1.4G      8.9%     12.8M    351.8M      2.5G     12.8%    132.7M      1.3G     84.0k
20:42:37        13.6G     13.7G      1.4G      9.0%     12.8M    351.8M      2.5G     12.8%    132.7M      1.3G     84.0k
20:42:38        13.6G     13.7G      1.4G      9.0%     12.8M    351.8M      2.5G     12.8%    132.7M      1.3G     84.0k
20:42:39        13.6G     13.7G      1.4G      9.1%     12.8M    351.8M      2.5G     12.8%    132.7M      1.4G     84.0k
20:42:40        13.5G     13.7G      1.4G      9.2%     12.8M    351.8M      2.5G     12.8%    132.7M      1.4G     84.0k
20:42:41        13.5G     13.7G      1.4G      9.3%     12.8M    351.8M      2.5G     12.8%    132.7M      1.4G     84.0k
20:42:42        13.5G     13.7G      1.4G      9.3%     12.8M    351.8M      2.5G     12.8%    132.7M      1.4G     84.0k
20:42:43        13.6G     13.8G      1.3G      8.7%     12.8M    351.8M      2.4G     12.3%    132.7M      1.3G     84.0k
20:42:44        13.6G     13.8G      1.3G      8.7%     12.8M    351.8M      2.4G     12.3%    132.7M      1.3G     84.0k
20:42:45        13.6G     13.8G      1.3G      8.7%     12.8M    351.8M      2.4G     12.3%    132.7M      1.3G     68.0k
20:42:46        13.6G     13.8G      1.3G      8.7%     12.8M    351.8M      2.4G     12.3%    132.7M      1.3G     72.0k
```

#### 시스템 전체 페이지 폴트 발생 상황 확인
- `sar -B`
  - 20:46:55 > 메모리 확보 전
  - 20:46:57 ~ 08 > 메모리 확보 및 접근
  - 20:47:08 ~ > 메모리 할당 해제
- 기간 내 초당 페이지 폴트 수인 fault/s 증가 하여야 하나... 다른프로세스들 때문인가 시끄럽군요

```
❯ sar -B 1 > fault.log &
[1] 17028
❯ ./demand-paging.py
20:46:55: 새로운 메모리 영역 확보 전. 엔터 키를 누르면 100메가 새로운 메모리 영역을 확보합니다: 

20:46:57: 새로운 메모리 영역을 확보했습니다. 엔터 키를 누르면 1초당 1MiB씩, 합계 100MiB 새로운 메모리 영역에 접근합니다: 

20:46:59: 10 MiB 진행중
20:47:00: 20 MiB 진행중
20:47:01: 30 MiB 진행중
20:47:02: 40 MiB 진행중
20:47:03: 50 MiB 진행중
20:47:04: 60 MiB 진행중
20:47:05: 70 MiB 진행중
20:47:06: 80 MiB 진행중
20:47:07: 90 MiB 진행중
20:47:08: 새롭게 확보한 메모리 영역에 모두 접근했습니다. 엔터 키를 누르면 종료합니다: 

❯ cat fault.log
Linux 5.15.167.4-microsoft-standard-WSL2 (nasir-pc)     04/08/25        _x86_64_        (12 CPU)

20:46:53     pgpgin/s pgpgout/s   fault/s  majflt/s  pgfree/s pgscank/s pgscand/s pgsteal/s    %vmeff
20:46:54        20.00    168.00   2899.00      0.00   3050.00      0.00      0.00      0.00      0.00
20:46:55      2848.00      0.00   3944.00     40.00   8527.00      0.00      0.00      0.00      0.00
20:46:56       920.00      0.00   3330.00      8.00   1864.00      0.00      0.00      0.00      0.00
20:46:57         0.00    120.00   2180.00      0.00   1218.00      0.00      0.00      0.00      0.00
20:46:58         0.00     72.00   3625.00      0.00   2745.00      0.00      0.00      0.00      0.00
20:46:59         0.00      0.00    107.00      0.00    401.00      0.00      0.00      0.00      0.00
20:47:00         0.00      0.00   6741.00      0.00   6055.00      0.00      0.00      0.00      0.00
20:47:01         0.00      0.00   4391.00      0.00   3212.00      0.00      0.00      0.00      0.00
20:47:02         0.00     28.00   1375.00      0.00    942.00      0.00      0.00      0.00      0.00
20:47:03         0.00      0.00   2402.00      0.00   2081.00      0.00      0.00      0.00      0.00
20:47:04         0.00      0.00     44.00      0.00      6.00      0.00      0.00      0.00      0.00
20:47:05         0.00      0.00   6789.00      0.00   4876.00      0.00      0.00      0.00      0.00
20:47:06         0.00      0.00   1426.00      0.00   1157.00      0.00      0.00      0.00      0.00
20:47:07         0.00     20.00   3609.00      0.00   2296.00      0.00      0.00      0.00      0.00
20:47:08         0.00    128.00    117.00      0.00      6.00      0.00      0.00      0.00      0.00
20:47:09         0.00      0.00   2921.00      0.00   1760.00      0.00      0.00      0.00      0.00
20:47:10         0.00      0.00   4545.00      0.00   3243.00      0.00      0.00      0.00      0.00
20:47:11         0.00      0.00   6016.00      0.00   5623.00      0.00      0.00      0.00      0.00
20:47:12         0.00     24.00   1957.00      0.00    916.00      0.00      0.00      0.00      0.00
20:47:13         4.00     16.00   3096.00      0.00   2734.00      0.00      0.00      0.00      0.00
```

#### demand-paging 프로세스 정보
시스템 전체가 아닌 demand-paging.py 프로세스의 확보한 메모리 영역 용량, 확보한 물리 메모리 용량, 페이지 폴트 총 횟수 등 확인
- ps -o vsz rss maj_flt min_flt
  - 페이지 폴트 횟수는 major, minor 두 중류가 있음 > 차이는 8장 설명
  - 일단 major+minor = 페이지 폴트 총 횟수

`./capture.sh`
- 1초마다 demand-paging.py의 메모리 관련 정보 출력
  - 1번 필드: 확보한 메모리 영역 크기 (vsz)
  - 2번 필드: 확보한 물리 메모리 크기 (rss)
  - 3번 필드: 메이저 폴트 횟수 (maj_flt)
  - 4번 필드: 마이너 폴트 횟수 (min_flt)
- timeline
  - 01:53:22: 새로운 메모리 영역 확보 전
  - 01:53:53: 새로운 메모리 영역을 확보
  - 01:54:02: 새롭게 확보한 메모리 영역에 모두 접근
- 볼거
  - 01:53:53 부터 vsz 약 100MiB 증가
  - 01:53:53 ~ 02 간 rss 및 min_fault 증가 > vsz rss 같아질때 까지
```
Tue Apr  8 01:53:44 KST 2025:  16128  9252      0   1107
Tue Apr  8 01:53:45 KST 2025:  16128  9252      0   1107
Tue Apr  8 01:53:46 KST 2025:  16128  9252      0   1107
Tue Apr  8 01:53:47 KST 2025:  16128  9252      0   1107
Tue Apr  8 01:53:48 KST 2025:  16128  9252      0   1107
Tue Apr  8 01:53:49 KST 2025:  16128  9252      0   1107
Tue Apr  8 01:53:50 KST 2025:  16128  9252      0   1107
Tue Apr  8 01:53:51 KST 2025:  16128  9252      0   1107
Tue Apr  8 01:53:52 KST 2025:  16128  9252      0   1107
Tue Apr  8 01:53:53 KST 2025: 118528  9252      0   1107
Tue Apr  8 01:53:54 KST 2025: 118528 21396      0   1559
Tue Apr  8 01:53:55 KST 2025: 118528 31636      0   1564
Tue Apr  8 01:53:56 KST 2025: 118528 41876      0   1569
Tue Apr  8 01:53:57 KST 2025: 118528 52116      0   1574
Tue Apr  8 01:53:58 KST 2025: 118528 62356      0   1579
Tue Apr  8 01:53:59 KST 2025: 118528 72744      0   1584
Tue Apr  8 01:54:00 KST 2025: 118528 82984      0   1589
Tue Apr  8 01:54:01 KST 2025: 118528 93224      0   1594
Tue Apr  8 01:54:02 KST 2025: 118528 103464     0   1599
Tue Apr  8 01:54:03 KST 2025: 118528 111840     0   1668
```

## 페이지 테이블 계층화
페이지 테이블은 평탄한(1차원)구조가 아닌 계층화 구조를 사용해 메모리 소비량 감소
- x86_64 시스템의 경우 가상주소공간은 128TiB, 1페이지는 4KiB, 페이지테이블엔트리는 8바이트 크기
- 1 프로세스당 페이지 테이블 최대 256GiB(=8byte X 128TiB/4KiB)
- x84 아키텍쳐의 경우 4단 구조 페이지 테이블 사용
  - 평탄한 구조 대비 필요한 하위 페이지 테이블 만 생성하여 필요 메모리 감소
- 페이지 테이블에 할당된 메모리는 `sar -r ALL` 중 `kbpgtbl` 필드에서 확인
  
![image](https://github.com/user-attachments/assets/2430ccaa-90ca-4d92-9a9a-6f6a71f4295e)    
[How Do Multi-Level Page Tables Save Memory Space?](https://www.baeldung.com/cs/multi-level-page-tables)
- 가상메모리는 메모리+디스크로 작동, 필요한거만 메모리에 올려서 쓰고 아닌건 디스크로

![image](https://github.com/user-attachments/assets/77114212-53fd-4ae2-a8fa-4663fcddb7dd)    
[x86 Page Tables](https://cs4118.github.io/www/2023-1/lect/18-x86-paging.html)
- 레벨1에서 레벨4로 진화하면서, 하나씩 구성요소가 늘어남
  - lvl1) PTE, Page Table Entry
  - lvl2) + PGD, Page table Global Directory
  - lvl3) + PGDP, Page table Global Directory Pointer
  - lvl4) PGD, PUD(Page Upper Directory), PMD (Page Middle Directory), PTE 

### Huge Page
리눅스에선 페이지당 크기가 큰 페이지를 이용하여 프로세스의 페이지 테이블에 필요한 메모리 용량 줄이기 가능
- 프로세스 메모리 증가 > 페이지테이블이 사용하는 물리 메모리 사용량 증가
  - 1페이지가 100byte, 400byte가 한 묶음인 2단 구조의 페이지 테이블 > 20개의 페이지 테이블 엔트리
  - 1페이지가 400byte인 Huge Page를 사용하면 1단 구조의 페이지 테이블 > 4개의 페이지 테이블 엔트리
- 페이지 테이블 엔트리가 줄어 페이지 테이블의 메모리 사용량 감소
- fork() 함수시 페이지 테이블 복사비용도 줄어 fork() 함수처리 빨라짐
- mmap()함수의 `MAP_HUGETLB` 플래그 지정을 통해 사용가능
  - 데이터베이스, 가상머신 등 가상메모리를 대량으로 사용하는 프로세스면 사용 설정 고민

![image](https://github.com/user-attachments/assets/f360f515-d4fe-4563-a80e-65221ec5e7e5)
[Hugepage를 활용한 성능 향상: 자원 활용을 위한 효과적인 전략](https://s-space.snu.ac.kr/bitstream/10371/196495/1/000000177997.pdf)
- PDF page 15, HugePage의 구조
- 2번의 200KB malloc 과 1번의 800KB malloc 한다고했을때
  - 리눅스 기본 4KB 페이지는 총 50x2 + 200 의 300개의 page 호출필요
  - 2MB의 page를 사용하면 1200KB 메모리 할당을 한번의 호출로 처리가능
  - 대신 2MB에서 1200KB을 제외한 나머지 공간은 단점 

### Transparent Huge Page(THP)
Transparent Huge Page: 가상 주소 공간 내부의 연속된 4KiB 페이지가 조건을 만족하면 Huge Page로 변경
- 일일히 Huge Page 요청은 번거롭기 때문에 등장
- 단점
  - 여러 페이지를 합쳐서 하나의 Huge Page 생성에 들어가는 처리 비용
  - 구성 조건이 해제되면 커다란 페이지를 다시 4KiB로 분할처리 비용
- /sys/kernel/mm/transparent_hugepage/enabled 으로 설정 확인
  - always: 시스템 상의 프로세스의 모든 메모리 대상 유효화
  - madvice: madvise() 시스템콜에서 MADV_HUGEPAGE 플래그 설정 시
  - never: 무효화

```
cat /sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never
```
