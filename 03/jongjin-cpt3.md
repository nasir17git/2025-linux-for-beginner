# 프로세스 스케줄러
> 프로세스 스케줄러: 프로세스에 CPU 자원 할당을 담당하는 리눅스 커널
- 시스템에 존재하는 대부분의 프로세스는 슬립 상태
- 여러개의 실행가능한 프로세스가 존재 시, 커널은 어떻게 각 프로세스를 CPU에서 실행할것인가?

일반적인 스케줄러의 설명
- 하나의 논리 CPU는 동시에 하나의 프로세스만 처리
- 실행가능한 프로세스가 타임슬라이스(time slice) 단위로 CPU 사용


## 기본지식: 경과 시간과 사용 시간
- 경과 시간(elapsed time)
  - process 시작부터 종료까지의 경과시간
- 사용 시간(CPU time)
  - 프로세스가 실제로 CPU를 사용한 시간

time 명령어를 통해 확인해보기
- 출력결과의 total(real)/user/sys 확인 가능
  - total(real) > 경과 시간
  - user & sys > 사용 시간
    - user > 프로세스가 사용자 공간에서 동작한 시간
    - sys > 프로세스의 시스템콜 등에 소요된 시간
``` 
time ./load.py
./load.py  1.51s user 0.01s system 99% cpu 1.526 total

time sleep 3
sleep 3  0.00s user 0.00s system 0% cpu 3.002 total
```

## 논리 CPU를 하나만 사용하는 경우
동시 실행을 1배, 2배, 3배로 실행했을 경우 개별 프로세스의 시간은 차이가 적지만 전체 실행 시간은 증가
- 하나의 논리 CPU는 한번에 프로세스 하나만 처리 가능
- 스케줄러가 각 프로세스에 순서대로 CPU 자원을 할당
> 전체 실행시간은 프로세스 개수에 비례

multiload.sh [-m] <프로세스 개수>
- 부하처리 프로세스를 <프로세스 개수> 만큼 동작시켜 대기
- 각 프로세스 실행에 걸린 시간 출력
- -m: 각프로세스를 여러 CPU에서 동작
```
./multiload.sh 1

real    0m1.499s
user    0m1.487s
sys     0m0.000s

./multiload.sh 3

real    0m4.428s
user    0m1.475s
sys     0m0.000s

real    0m4.443s
user    0m1.492s
sys     0m0.000s

real    0m4.454s
user    0m1.475s
sys     0m0.010s
```

## 논리 CPU를 여러 개 사용하는 경우
부하분산 처리의 동작 이론은 매우 복잡하므로 스킵
- 논리 CPU가 2개이고, 부하처리할 프로세스가 2개면 각각 CPU자원을 처리


```
./multiload.sh -m 3

real    0m1.664s
user    0m1.651s
sys     0m0.010s

real    0m1.755s
user    0m1.741s
sys     0m0.010s

real    0m1.803s
user    0m1.801s
sys     0m0.000s
```

## real보다 user+sys가 커지는 경우
직감적으로 real >= user + sys 여야 할것 같지만 실제로 user+sys가 real보다 큰 경우도 존재
- 차이가 작은 경우) 각 시간 측정 방법이 다르고, 측정 정밀도가 높지 않기 때문
- 차이가 큰 경우) 프로세스가 자식을 생성하고, 각 다른 CPU에서 동작한다면 가능

time ./multiload.sh -m 2
- user 값(3.16s)가 total 1.60보다 크게 조회됨

```
time ./multiload.sh -m 2

real    0m1.596s
user    0m1.576s
sys     0m0.000s

real    0m1.606s
user    0m1.587s
sys     0m0.000s
./multiload.sh -m 2  3.16s user 0.00s system 196% cpu 1.609 total
```

## 타임 슬라이스
스케줄러는 실행가능한 프로세스에 타임 슬라이스 단위로 CPU 할당
- 앞의 예시로 하나의 CPU에는 하나의 프로세스 동작 확인
- 구체적으로 어떻게 CPU 자원을 배분하는지?

sched.py <프로세스 개수>
- CPU시간을 사용하는 프로세스를 작동시켜 통계 정보 수집
  - 논리 CPU에서 특정시점에 어떤 프로세스가 진행되는지
  - 진척도는 어떠한지
- schd-<프로세스 개수>.jpg 생성

1개의 CPU에서 여러 처리를 동시 실행시,  
각 처리는 밀리초 단위의 타임 슬라이스로 쪼개서 CPU를 교대로 사용함
```
for i in 1 2 3; do ./sched.py $i; done
```


### 타임 슬라이스 구조
- 동시실행 2 대비 동시실행3인 경우 각 프로세스의 타임 슬라이스가 짧음
- 리눅스 스케줄러는 sysctl의 kernel.sched_latency_ns 파라미터의 간격에 따라 CPU 시간을 가져옴

```
sysctl kernel.sched_latency_ns
sysctl: cannot stat /proc/sys/kernel/sched_latency_ns: No such file or directory

mount -t debugfs none /sys/kernel/debug
cat /sys/kernel/debug/sched/debug | grep latency

  .sysctl_sched_latency                    : 24.000000
```

리눅스 커널 버전 2.6.23 이전은 타임슬라이스가 고정 값 (10ms)
- 프로세스가 늘어날수록 각 프로세스에 CPU 시간 돌아오기까지 대기시간 문제 발생
- 요즘 스케줄러는 프로세스 개수에 따라 동적으로 변경
  - 커널 버전 확인 uname -r
  - 5.15.167.4-microsoft-standard-WSL2

목표 레이턴시 또는 타임 슬라이스 값 계산시 영향 끼치는 요소들
- 시스템에 설치된 논리 CPU 개수
- 일정한 값을 넘는 논리 CPU에서, 실행 및 대기 중인 프로세스 개수
- 프로세스 우선도를 뜻하는 nice 값 

nice
- 프로세스 실행 우선도를 -20부터 19까지 설정 (기본 0)
  - -20이 최우선, 19가 최하위
  - 우선도를 낮추는건 모두가능/ 높이는건 root만 가능
- nice 명령어, renice 명령어, nice()시스템 콜, setpriority()시스템 콜로 변경 가능
- nice 값이 낮은(=우선도가 높은) 프로세스에 타임슬라이스 더 많이 부여

./sched-nice.py <nice 값>
- 논리 CPU0에서 100밀리초 동안 CPU 자원을 소비하는 부하 처리를 2개 실행하고 프로세스가 종료할 때까지 기다립니다.
  - 부하처리 0, 1의 nice값은 각각 0(기본값), <nice값>이 됩니다.
  - 'sched-2.jpg' 파일에 실행 결과 그래프를 저장합니다.
  - 그래프 x축은 프로세스의 경과 시간[밀리초], y축은 진척도[%]
- 부하처리 0이 1보다 타임슬라이스를 많이 가져감 확인
```
./sched-nice.py 5
```

스케줄러의 구현사항은 POSIX 표준이 아니므로 커널변경에 따라 변경 가능
- kernel.sched_latency_ns 의 기본값이 수차례 변경되었음
- ls -alh /proc/sys/kernel | grep sched
  - sched_cfs_bandwidth_slice_us? 
  - CFS(Completely Fair Scheduler)


## 컨텍스트 스위치
컨텍스트 스위치
- 논리 CPU에서 동작하는 프로세스가 전환되는것
- 타임 슬라이스가 끝나면 발생
  - 프로세스에서 foo() 이후 bar() 실행된다는 연결 보장 X
  - foo() 호출 이후 bar() 호출 간 다른 프로세스 동작 가능성 O
- 따라서 특정 프로세스의 처리속도가 느릴때,
  - 처리 자체에 문제가 있다 X
  - 처리 간 컨텍스트 스위치로 인해 다른 프로세스 동작가능성 있다 O

## 처리 성능
정해진 성능요건 측정에 관한 지표
- 턴어라운드 타임(turnaround time)
  - 처리(반환) 시간
  - 시스템에 처리 요청 부터 처리 종료까지 걸린 시간
- 스루풋(throughput)
  - 처리량
  - 단위 시간당 처리를 끝낸 개수

multiload.sh 프로그램 대상으로 다음의 성능지표 수집
- cpuperf.sh 및 plot-perf.py 사용
  - 평균 턴어라운드 타임: 모든 부하처리의 real 값 평균
  - 스루풋: 프로세스 개수를 multiload.sh 프로그램의 real값으로 나눈 몫

cpuperf.sh [-m] <최대 프로세스 개수>
- cpuperf.data 파일에 성능 정보를 저장
  - 엔트리 개수는 <최대 프로세스 개수>
  - 출력 형식은 '<프로세스 개수> <평균 턴어라운드 타임[초]> <스루풋[프로세스/초]>'
- 성능 정보를 바탕으로 평균 턴어라운드 타임 그래프를 만들어서 'avg-tat.jpg'에 저장
- 같은 방법으로 스루풋 그래프를 만들어서 'throughput.jpg'에 저장
- -m 옵션은 multiload.sh 프로그램에 그대로 전달

plot-perf.py <최대 프로세스 개수>
- cpuperf 프로그램 실행 결과를 저장한 'perf.data' 파일을 가지고 성능 정보를 그래프로 작성합니다.
- 'avg-tat.jpg' 파일에 평균 턴어라운드 타임 그래프를 저장합니다.
- 'throughput.jpg' 파일에 스루풋 그래프를 저장합니다.


1논리 CPU, 8 최대 프로세스
- ./cpuperf.sh 8
  - 논리 CPU < 프로세스 개수 시 평균 처리시간 증가, 스루풋 차이없음
  - 프로세스 개수 늘리면 컨텍스트 스위치로 인해 평균 처리시간 지연, 스루풋 떨어짐

처리시간(응답성능)이 중요한 시스템이면 각 기기의 CPU 사용률을 적게 유지하는것이 중요
- 사용자요청을 받아 파일 생성 후 돌려주는 프로그램 가정
  - 논리 CPU 부하가 높다면, 평균 처리시간이 점점 길어짐
  - 웹앱 응답시간 지연과 연결되므로 사용자 경험이 나빠짐
  
모든 논리 CPU를 사용해 성능 데이터 수집
- grep -c processor /proc/cpuinfo
  - 12 
- SMT(Simultaneous MultiThreading) 비활성화 필요
  - WSL2에서는 직접 SMT 제어가 불가능..
  - echo off > /sys/devices/system/cpu/smt/control
    - echo: write error: device or resource busy

./cpuperf.sh -m 24
- 평균턴어라운드타임: 논리CPU개수(12)까지는 천천히/그뒤 급격히
- 스루풋: 논리CPU개수(12)까지는 상승/ 이후 유지
- 결론
  - 논리 CPU 개수가 많아도 충분한 수의 프로세스가 있어야 스루풋 향상
  - 프로세스를 늘린다고해서 스루풋 개선 X

## 프로그램 병렬 실행의 중요성 
- 과거 CPU는 논리 CPU당 성능(single thread performance)에 집중
- 최근 CPU는 코어 갯수 증가 등 전체 CPU 성능 개선에 집중
- 커널 또한 코어 개수가 늘어난 경우의 확장성(scalability) 향상