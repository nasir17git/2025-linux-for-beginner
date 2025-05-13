# 메모리 계층


https://cstaleem.com/wp-content/uploads/2020/08/Memory-Hierarchy-in-COA.png


Memory Hierarchy Design


https://binaryterms.com/wp-content/uploads/2023/02/Memory-Hierarchy-.jpg
https://binaryterms.com/memory-hierarchy-in-computer-architecture.html
기억장치는 계층별로 용량/가격/성능이 다르다.
이 챕터에서는 구체적으로 각 장치별 크기 및 성능 차이, 
이 차이점을 고려한 하드웨어 & 리눅스의 작동을 확인한다


## 캐시 메모리

https://cs61.seas.harvard.edu/site/img/storage-hierarchy.png
https://cs61.seas.harvard.edu/site/2021/Storage1/
- 캐시 메모리(Cache Memory): CPU와 메모리 간 속도 차이를 줄이기 위해 자주 사용하는 데이터를 임시로 저장하는 고속 메모리
- 일반적인 CPU의 동작
  1. 명령을 읽고 내용에 따라 메모리에서 레지스터로 데이터 읽기
  2. 레지스터의 데이터를 갖고 계산
  3. 계산 결과를 다시 메모리 저장
- 레지스터의 계산은 회당 1나노초 미만, 메모리 접근은 회당 수십 나노초 이상, 따라서 1,3의 병목 발생
- 이를 위해 캐시 메모리 사용
  - CPU 내부에 존재하는 고속 기억장치
  - 메모리 > 레지스터 데이터 읽을때, 캐시라인(cache-line) 단위로 데이터 읽어서 레지스터로 전달
  - 하드웨어 단에서 작동하므로 커널 개입 X

https://people.csail.mit.edu/devadas/6.004/Lectures/lect18/img018.GIF
Dirty Bits for Write-Back Caches

- 캐시 메모리의 동작 방식
  - CPU가 메모리에 접근 시, 캐시 메모리에 데이터 복사
  - 이후 데이터 접근 시 메모리가 아닌 캐시메모리에 접근하여 빠른 처리 가능
  - 데이터가 변경 시, 메모리에 직접 반영 대신 캐시메모리에 반영하고 데이터 변경 표시
    - 이때 변경 표시가 붙은 캐시라인을 더티(dirty)하다고 표기
  - 더티 표시가 있는 캐시라인의 데이터를 메모리에 쓰면 더티 표시 제거
- 메모리에 데이터 쓰는 방식
  - 라이트 쓰루(write-through, 바로 쓰기)
    - 데이터를 캐시 메모리와 메모리에 바로 기록
    - 구현 간단
  - 라이트 백(write-back, 나중에 쓰기)
    - 캐시 메모리에만 기록, 메모리에 기록은 나중에 한번에 쓰기
    - 바로 메모리 접근X  처리 속도 빠름
- 캐시 메모리가 가득차고, 새로운 데이터를 읽어와야한다면 기존 캐시라인 중 하나를 대체
  - 클린 (clean): 더티한 캐시라인이면 메모리에 저장하고 버림
  - 스래싱 (thrashing): 캐시 메모리 full, 계속 새로운 데이터 접근하면 캐시라인 데이터의 빈번한 대체 > 성능저하

### 참조 지역성
- 참조 지역성(Locality of Reference):  프로그램이 메모리를 접근할 때 시간적·공간적으로 가까운 위치를 반복해서 참조하는 경향
  - 만약 CPU가 사용하는 데이터가 전부 캐시 메모리에 존재한다면, 캐시 메모리 접근만으로 처리 가능
  - write-back 방식이라면, 캐시 메모리 기록만으로 처리 완료
  - 많은 프로그램에서 참조지역성이라는 다음의 특징 존재
    - 시간 지역성(Temporal Locality)
      - 특정 시점에 접근하는 메모리는 가까운 미래에 다시 접근할 확률 높음
      - 반복 처리 내부에 존재하는 코드 등
    - 공간 지역성(Spatial Locality)
      - 특정 시점에 접근하는 메모리는 그 근처에 있는 메모리도 자주 접근할 확률 높음
      - 배열 요소 전체를 순서대로 조사하는 배열 데이터
  - 이러한 지역성을 활용하면 이상적인 고속 처리 기대 가능


### 계층형 캐시 메모리
- 계층형 캐시 메모리(Hierarchical Cache): 속도와 크기가 다른 여러 단계의 캐시(L1, L2, L3 등)를 계층적으로 배치해 CPU와 메인 메모리 간 성능 차이를 효율적으로 완화하는 구조
- 최근의 CPU는 캐시 메모리를 L1,L2,L3로 계층화된 구조 형태로 관리
  - L1에 가까울수록 빠르고 용량이 적음, 멀수록 느려지고 용량 많음
  - `/sys/devices/system/cpu/cpu0/cache/index0` 에서 확인 가능

```
ll /sys/devices/system/cpu/cpu0/cache
total 0
drwxr-xr-x 2 root root    0 May 13 15:21 index0
drwxr-xr-x 2 root root    0 May 13 15:22 index1
drwxr-xr-x 2 root root    0 May 13 15:22 index2
drwxr-xr-x 2 root root    0 May 13 15:22 index3

sudo find /sys/devices/system/cpu/cpu0/cache/index0 -type f -exec sh -c 'echo "--- {} ---"; cat {}' \;

--- /sys/devices/system/cpu/cpu0/cache/index0/uevent ---
--- /sys/devices/system/cpu/cpu0/cache/index0/physical_line_partition ---
1
--- /sys/devices/system/cpu/cpu0/cache/index0/number_of_sets ---
64
--- /sys/devices/system/cpu/cpu0/cache/index0/ways_of_associativity ---
8
--- /sys/devices/system/cpu/cpu0/cache/index0/id ---
0
--- /sys/devices/system/cpu/cpu0/cache/index0/shared_cpu_list ---
0-1
--- /sys/devices/system/cpu/cpu0/cache/index0/type ---
Data
--- /sys/devices/system/cpu/cpu0/cache/index0/size ---
32K
--- /sys/devices/system/cpu/cpu0/cache/index0/level ---
1
--- /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size ---
64
--- /sys/devices/system/cpu/cpu0/cache/index0/shared_cpu_map ---
003
```

### 캐시 메모리 접근 속도 측정

- cache.go
  1. 2^2(=4)K바이트부터 2^2.25K바이트, 2^2.5K바이트, ...식으로 최종적으로 64M바이트가 될 때까지 다음 처리를 반복
    - 숫자에 해당하는 크기의 버퍼를 확보
    - 버퍼의 모든 캐시라인에 순차적으로 접근. 마지막 캐시라인까지 접근했으면 다시 첫 캐시라인으로 돌아가고, 최종적으로 소스 코드에 지정한 NACCESS번 메모리에 접근함
    - 접근 1회당 걸린 시간을 기록
  2. 1에서 나온 결과를 바탕으로 그래프를 작성해서 cache.jpg 파일로 저장
    - plot-cache.py 파일을 통해 그래프 생성

- 각 캐시 크기를 경계를 접근 시간은 계단식 변화,
- 버퍼 크기가 각 L1,2,3 캐시 메모리 용량 도달 및 전후로 접근 속도 변화
- 버퍼 접근시간 외에 명령문 포함된 시간이라 버퍼 크기 작을때는 영향 받아 속도 떨어짐 참고


## Simultaneous Multi Threading(SMT)
- Simultaneous Multi-Threading(SMT): 하나의 CPU 코어가 여러 스레드를 동시에 실행하여 자원을 병렬로 활용하는 기술
- 데이터 전송, CPU 연산 유닛의 대기로 인한 낭비를 줄이기 위해 사용
  - 명령어 실행시 데이터 전송을 기다리며 CPU 계산 자원은 대기
  - 정수 연산/부동 소수점 연산 유닛등에서 다른 유닛은 대기
- CPU 코어 내부의 레지스터와 같은 일부 자원을 각 스레드 생성
  - CPU의 스레드와는 별개의 단어
- 리눅스 커널은 이러한 각 스레드는 논리 CPU로 인식
  - 하나의 CPU를 t0,t1 2개의 스레드로 분리 후 각 p0,p1 프로세스 동작
  - t0에서 p0가 동작 중 남는 자원이 있으면 t1의 p1이 사용 가능
    - t0-p0가 정수 연산 수행 > t1-p1이 부동소수점 연산 가능

## 페이지 캐시
- CPU > 메모리 접근 속도 느림: 캐시 메모리
- CPU > 저장장치 접근 속도 느림: 페이지 캐시

https://user-images.githubusercontent.com/62331555/81669624-70a86e80-9481-11ea-8183-0e578c3b57f9.png
page cache - horoyoiiv/linux GitHub Wiki

- 캐시메모리: 데이터를 캐시 메모리에 캐시
- 페이지캐시: 파일 데이터를 메모리에 캐시
  - 더티 페이지(dirty page), 라이트백(write back) 개념 존재

- 프로세스가 파일을 읽을 때
  - 커널은 프로세스 메모리에 파일 데이터 직접 복사 X
  - 커널 메모리의 페이지 캐시에 파일 데이터 복사, 해당 데이터를 프로세스 메모리에 복사
  - 프로세스가 파일 데이터 읽으면 커널 메모리의 페이지 캐시에서 데이터 제공
- 프로세스가 파일을 변경할 때
  - 커널은 페이지 캐시에만 기록하고, 더티 표시(=더티 페이지)
  - 특정 타이밍에 저장 장치에 반영됨(=라이트백)

### 페이지 캐시 효과
- 크기가 1GiB인 testfile 읽고 쓰는데 소요시간으로 페이지 캐시 효과 측정
- 파일 쓰기
  - dd 명령어에서 oflag=sync 옵션 사용시 > 1.07초
  - dd 명령어에서 oflag=sync 옵션 미사용시 > 0.78초
  - 1G 파일 쓰기가 페이지 캐시에 올라갈 수 있어 빠르게 처리됨
    - 그럼 페이지 캐시크기가 1G보다 작다면 스래싱생겨서 오히려 느려지나?
- 파일 읽기
  - /proc/sys/vm/drop_caches에 3을 써서 캐시 비우기 약 2GB > 200MB (buff/cache)
  - testfile 읽기 첫시도 약 0.4초, 2트 0.2초
  - 첫번째는 저장장치에서, 두번쨰는 캐시메모리에서 읽어서 빨라짐


```
# 파일 쓰기
dd if=/dev/zero of=testfile oflag=sync bs=1G count=1
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 1.07609 s, 998 MB/s

dd if=/dev/zero of=testfile bs=1G count=1
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 0.787553 s, 1.4 GB/s
```

```
# 캐시 비우기
❯ free
              total        used        free      shared  buff/cache   available
Mem:       16196784     2751324    10794384        6300     2651076    13101664
Swap:       4194304           0     4194304
❯ echo 3 | sudo tee /proc/sys/vm/drop_caches
3
❯ free
              total        used        free      shared  buff/cache   available
Mem:       16196784     2736736    12653684        6312      806364    13168240
Swap:       4194304           0     4194304
```

```
# 파일 읽기
❯ dd if=testfile of=/dev/null bs=1G count=1
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 0.417268 s, 2.6 GB/s
❯ dd if=testfile of=/dev/null bs=1G count=1
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 0.212594 s, 5.1 GB/s


```
## 버퍼 캐시
- 버퍼 캐시(Buffer Cache): 파일 데이터 이외의 것을 캐시
  - 파일 시스템을 사용하지 않고 디바이스 파일로 저장 장치 직접 접근 시
  - 파일 크기, 권한 등 메타데이터 접근 시
- 버퍼 캐시의 데이터가 실제 디스크에 반영되지않은 더티 상태 존재
- 장치의 버퍼 캐시와 fs의 페이지 캐시는 서로 다르므로 동기화 X

- dd if=<fs에 대응하는 디바이스파일> of=<백업 파일>
  - fs의 더티 페이지 내용은 백업 파일에 반영 X
  - 마운트된 파일시스템의 디바이스 파일엔 접근하지 않는게 좋음

## 쓰기 타이밍
- 더티 페이지는 백그라운드 커널의 라이트백 처리에 따라 저장
- 저장 타이밍
  - 주기적, 기본값은 5초에 1회
  - 더티 페이지가 늘어났을때
- vm.dirty_writeback_centisecs 으로 라이트백 주기 설정 가능(센티초, 1/100초)
  - 기본 값 500 > 5초
  - 값이 0 > 주기적 라이트백 하지 않음
- vm.dirty_background_ratio 로 라이트백 기준 설정 가능(%)
  - 전체 물리 메모리 중 더티 페이지가 차지하는 비율
  - 기본 값 10 > 메모리의 10%
- vm.dirty_background_bytes 로 라이트백 기준 바이트 설정 가능
  - 기본 값 0 > 미설정
- vm.dirty_ratio 로 라이트백 기준 설정 가능(%)
  - 가본값 20 
- 더티 페이지의 라이트백이 자주 발생하면 시스템 멈춤, OOM 발생 가능. 적절한 커널 튜닝 필요

## 직접 입출력
- 일부 상황에서는 페이지/버퍼 캐시가 유용하지 않음
  - 한번 읽고 쓴 뒤 두번 다시 사용하지 않는 데이터
    - 특정 fs 내용을 이동식 저장 장치에 백업
    - 백업 이후 제거되므로 페이지/버퍼 캐시 필요X, 다른 캐시 해제될 가능성
  - 프로세스가 자체적으로 페이지 캐시에 해당하는 기능 구현
- 직접 입출력(Direct I/O)를 사용해 페이지 캐시 없이 처리 가능
  - open() 함수의 O_DIRECT 플래그 사용
  - dd명령어의 iflag 또는 oflag=direct 옵션 사용
  - 장치 입출력 전후간 cache(buff/cache) 차이 상대적 근소함 
```
❯ free
              total        used        free      shared  buff/cache   available
Mem:       16196784     3755960    10397440        6416     2043384    12149364
Swap:       4194304           0     4194304
❯ dd if=/dev/zero of=testfile bs=1G count=1 oflag=direct,sync
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 0.237642 s, 4.5 GB/s
❯ free
              total        used        free      shared  buff/cache   available
Mem:       16196784     3747268    10404812        6420     2044704    12158032
Swap:       4194304     
```

## 스왑
- 스왑(swap): 저장 장치 일부를 임시로 메모리처럼 사용
  - 물리 메모리 고갈 시, 사용 중인 메모리 일부를 저장 장치로 옮기고(스왑 영역) 메모리에 빈 공간 생성
  - 물리 메모리가 고갈된 상태, 페이지 폴트 발생 시
   - 페이지 아웃,스왑 아웃(page-out,swap-out): 커널이 한동안 사용되지 않았을거라고 생각하는 메모리를 스왑 영역으로 옮김
  - 물리 메모리 빈 공간이 생기고 프로세스가 접근 시
   - 페이지 인, 스왑 인(page-in,swap-in): 커널이 스왑 영역에서 메모리로 옮김
- 메이저 폴트, 마이너 폴트(major, minor fault)
  - 메이저 폴트: 페이지 인으로 인한 저장 장치의 접근
  - 마이너 폴트: 그 외
- 스레싱(thrashing): 메모리 접근마다 페이지 인,아웃이 반복되는 상태
  - 페이지 캐시의 스레싱과는 다름
  - 가상 메모리처리에 과부하가 생겨 CPU,OS가 페이징에 처리 능력 소비 중
  - ex) PC 사용 중 별다른 처리 안해도 저장 장치 램프가 작동중일때

## 통계 정보
- sar -r 명령어에서 살펴보지 않았던 영역들

| 필드명       | 의미                                                                 |
|--------------|----------------------------------------------------------------------|
| `kbmemfree`  | 비어 있는 메모리 용량(KiB 단위). 페이지 캐시나 버퍼 캐시, 스왑 영역은 포함 안됨 |
| `kbavail`    | 사실상 비어 있는 메모리 용량(KiB 단위). `kbmemfree`에 `kbbuffers`와 `kbcached`를 더한 값. 스왑 영역은 포함 안됨 |
| `kbbuffers`  | 버퍼 캐시 용량(KiB 단위)                                              |
| `kbcached`   | 페이지 캐시 용량(KiB 단위)                                            |
| `kbdirty`    | 더티한 상태의 페이지 캐시와 버퍼 캐시 용량(KiB 단위)       

```
# sar -r 1
Linux 5.15.167.4-microsoft-standard-WSL2 (nasir-pc)     05/13/25        _x86_64_        (12 CPU)

20:25:13    kbmemfree   kbavail kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty
20:25:14     11350996  13151780   2645292     16.33    150872   1878800   6402416     31.40    408028   3898936      1840
20:25:15     11351552  13152468   2644616     16.33    150984   1878804   6326076     31.02    408136   3895652      4312
20:25:16     11733840  13522392   2274700     14.04    151104   1866308   5647432     27.70    408256   3515372      2036
^C
Average:     11478796  13275547   2521536     15.57    150987   1874637   6125308     30.04    408140   3769987      2729
```

- sar -B: 페이지 인, 아웃 관련 정보 확인
  - 스왑 외에도, 페이지/버퍼 캐시가 디스크와 데이터 주고받는것도 페이지 인/아웃


| 필드명       | 의미                                                                 |
|--------------|----------------------------------------------------------------------|
| `pgpgin/s`   | 초당 페이지 인 데이터량(KiB 단위). 페이지 캐시, 버퍼 캐시, 스왑 전부를 포함 |
| `pgpgout/s`  | 초당 페이지 아웃 데이터량(KiB 단위). 페이지 캐시, 버퍼 캐시, 스왑 전부를 포함 |
| `fault/s`    | 페이지 폴트 수                                                       |
| `majflt/s`   | 페이지 폴트 중에서 페이지 인이 일어난 수(메이저 폴트)      

```
# sar -B 1
Linux 5.15.167.4-microsoft-standard-WSL2 (nasir-pc)     05/13/25        _x86_64_        (12 CPU)

20:25:27     pgpgin/s pgpgout/s   fault/s  majflt/s  pgfree/s pgscank/s pgscand/s pgsteal/s    %vmeff
20:25:28        64.00   1676.00  43146.00      0.00  50881.00      0.00      0.00      0.00      0.00
20:25:29         0.00  14712.00   7561.00      0.00    979.00      0.00      0.00      0.00      0.00
20:25:30         0.00    468.00   4821.00      0.00   3534.00      0.00      0.00      0.00      0.00
20:25:31       128.00   1520.00   4830.00      1.00   2714.00      0.00      0.00      0.00      0.00
^C
Average:        48.00   4594.00  15089.50      0.25  14527.00      0.00      0.00      0.00      0.00
```
- swapon --show: 스왑 영역 확인
- sar -W: 스왑 활동 통계
  - pswin/s: 초당 스왑 인 데이터량(KiB 단위), 페이지 인 횟수
  - pswout/s: 초당 스왑 아웃 데이터량(KiB 단위), 페이지 아웃 횟수

```
# sar -W 1
Linux 5.15.167.4-microsoft-standard-WSL2 (nasir-pc)     05/13/25        _x86_64_        (12 CPU)

20:25:58     pswpin/s pswpout/s
20:25:59         0.00      0.00
20:26:00         0.00      0.00
20:26:01         0.00      0.00
20:26:02         0.00      0.00
^C
Average:         0.00      0.00
```

- sar -S : 스왑 영역 이용 상황
  - kbswpused 필드의 스왑영역 사용량 추세 확인, 높아지면 위험
```
sar -S 1
Linux 5.15.167.4-microsoft-standard-WSL2 (nasir-pc)     05/13/25        _x86_64_        (12 CPU)

20:26:15    kbswpfree kbswpused  %swpused  kbswpcad   %swpcad
20:26:16      4194304         0      0.00         0      0.00
20:26:17      4194304         0      0.00         0      0.00
20:26:18      4194304         0      0.00         0      0.00
20:26:19      4194304         0      0.00         0      0.00
^C
Average:      4194304         0      0.00         0      0.00

```