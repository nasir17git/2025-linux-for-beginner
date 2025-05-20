# 블록 계층

<table>
  <tr>
    <td align="center">
      <img src="https://i.sstatic.net/jAtlb.png" width="400"><br>
      <a href="https://unix.stackexchange.com/questions/258410/which-components-of-linux-io-subsystem-are-device-independent-and-device-depende" target="_blank">
        Understanding the Linux Kernel, Bovet
      </a>
    </td>
    <td align="center">
      <img src="https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEi-wIHmLWiYgWrdttkbIlcD5znNfxk9w2OkFqzxS2uhF3QTGltKdzvnXuuNKZqLsXrAw6TKCj_JhjH3BqHHxuY5m-Lo2UDXsadjuwzSHcvMpYPr4HnncvpEJk5Q_cZsa-xcuxyLtsnKu4U/s1600/block+hierarchy+(3).png" width="400"><br>
      <a href="https://ari-ava.blogspot.com/2014/06/opw-linux-block-io-layer-part-1-base.html" target="_blank">
        Linux Kernel Development, Love, Robert
      </a>
    </td>
  </tr>
</table>

블록 계층(block layer): 사용자 요청을 파일시스템 → 가상 파일시스템(VFS) → 페이지 캐시 → 블록 I/O 계층 → 디바이스 드라이버 → 실제 디스크 순으로 전달하며, I/O 스케줄링과 캐싱을 통해 성능을 최적화하는 계층 구조

- 파일 시스템과 장치 드라이버 사이에서, 저장장치(블록장치)의 성능 향상을 위한 커널 기능

## 하드 디스크의 특징

![image](https://github.com/user-attachments/assets/2823027f-e48b-4c93-b3f5-fbd2e7606781)    


- 플래터(platter): 자기 정보로 표현된 데이터를 저장하는 자기 디스크
- 섹터(sector): 디스크에서 데이터를 저장하고 읽는 최소 단위
  - 바이트 단위가 아닌, 512B/4KiB 단위로 저장
- 자기 헤드(magnetic head): 섹터에 저장된 데이터 읽고 쓰기
- 스윙 암(swing arm/actuator arm): 자기 헤드를 플래터 위에 위치시키는 장치

- 하드 디스크의 데이터 전송 순서
  1. 디바이스 드라이버가 데이터 RW에 필요한 정보 전달
    - 섹터번호, 섹터 개수, 접근종류(R/W) 등
  2. 스윙 암과 플래터를 움직여 원하는 섹터위에 자기 헤드 위치 이동
  3. 데이터 읽고 쓰기
- 1,3은 전기 신호 처리라서 빠름, 2는 기계적 처리라서 상대적 느림. 대부분은 2의 소요시간
- 연속 섹터에 있는 데이터면 한번의 접근 요청으로 처리 가능
  - 스윙암을 통해 자기헤드 위치를 맞추면, 플래터 회전으로 연속 작업 가능
- 따라서 각종 fs은 각 파일 데이터를 연속적 영역에 배치


## 블록 계층의 기본 기능

- 연속된 데이터 처리는 빠르다는 하드디스크 특징 살림
- 
![image](https://github.com/user-attachments/assets/1bc68854-1c61-453a-8081-83a8512c9237)    
[Linux I/O scheduler for solid-state drives](https://www.cnblogs.com/coryxie/p/3983930.html)    

- 입출력 스케줄러(I/O scheduler): 디바이스 드라이버에서 전달받은 입출력 요청을 수집하고, 이를 최적화된 순서로 전달
  - 합치기(merge): 연속된 섹터의 요청 합치기
  - 정렬(sort): 비연속 섹터의 I/O요청 순서를 섹터 번호 순 정렬

![image](https://github.com/user-attachments/assets/2fb71ccd-46e8-4a95-81b8-f6d2c061fb71)    
[Storage subsystem performance: analysis and recipes](https://gudok.xyz/sspar/#_random_access)

- 미리 읽기(readahead): 데이터 전송 시 필요한 데이터를 미리 읽어오기
  - 메모리와 유사한 공간적 지역성 활용
  - 블록 장치의 특정영역을 읽을때 후속 영역을 미리 페이지 캐시에 저장
  - 순차 읽기 시 효과적


## 블록 장치의 성능 지표와 측정 방법
- 블록 장치 성능의 3가지 종류
  - 스루풋(throughput): 단위 시간당 데이터 전송량
  - 대기시간(latency): 입출력 1회당 소요 시간
  - IOPS(Input/Output Operations Per Second): 초당 입출력 횟수

  

### 하나의 프로세스만 입출력을 호출하는 경우
- 스루풋(throughput): 단위 시간당 데이터 전송량
  - 크기가 큰 데이터파일 복사 등 대량 데이터 처리 시 중요
- 대기시간(latency): 입출력 1회당 소요 시간
  - 저장 장치의 응답 성능, 대용량 데이터 전송보다는 자잘한 입출력이 많은 경우
  - 시스템 응답속도와 연관
- IOPS(Input/Output Operations Per Second): 초당 입출력 횟수
  - 레이턴시와 유사, 단위 시간당 처리 횟수

### 여러 프로세스가 병렬로 입출력을 호출하는 경우 
- 특정 블록 장치에서, 2개의 프로세스가 병렬로 입출력 호출하는 경우
  - 스루풋은 2배가 됨 (동일시간 데이터 전송량이 2배)
    - 장치나 버스(데이터 전송경로)의 한계까지 데이터 전송량이 증가
  - 대기시간은 절반이 됨 (동일 데이터 전송량에 대해 절반 시간)
    - 순차처리가 아닌 병렬 처리로 동일한 시간에 종료
  - IOPS는 2배보다는 적게 증가 (작업 전환 시간)
    - 프로세스가 I/O 처리를 마치고 CPU 사용동안 남는 시간을 다른 프로세스가 사용 가능

### 성능 측정 도구 fio
`fio - flexible I/O tester`

- 파일 시스템 성능 측정 도구이나 장치 성능 측정 도구로도 사용 가능
  - 입출력 패턴, 병렬도, 입출력방식(I/O Engine) 등 다양한 옵션 제공
  - 스루풋, 레이턴시, IOPS 등 성능 지표 측정 가능

- 설정 가능 옵션
  - --name: 각각의 성능 측정 작업명
  - --filename: 입출력 대상 파일명
  - --filesize: 입출력 대상 파일 크기
  - --size: 입출력의 합계 크기
  - --bs(=blocksize): 입출력 크기. 합계 입출력 횟수는 --size로 지정한 값을 --bs로 지정한 값으로 나눈 값이 됩니다.
  - --readwrite: 입출력 종류를 선택합니다. read(순차 읽기), write(순차 쓰기), randread(무작위 읽기), randwrite(무작위 쓰기) 등.
  - --sync=1: 쓰기 작업을 동기화합니다.
  - --numjobs: 입출력 병렬도. 기본값은 1으로 병렬 실행하지 않습니다.
  - --group_reporting: 병렬도가 2 이상일 때 성능 측정 결과를 각각 출력(기본값)하는 대신에 모든 처리를 합쳐서 출력합니다.
  - --output-format: 출력 형식

`fio --name test --readwrite=randread --filename testdata --filesize=1G --size=4M --bs=4k --output-format=json`
- 무작위 읽기 작업을 수행하고, 결과를 JSON 형식으로 출력
  - 1GiB 파일을 무작위로 4MiB 크기씩 4k 블록 단위로 읽음
- 결과 확인
  - bw_bytes: 바이트 단위 스루풋(초당 전송량)
  - iops: 초당 입출력 횟수
  - lat_ns: 레이턴시(최소, 최대, 평균, 표준편차)


```
fio --name test --readwrite=randread --filename testdata --filesize=1G --size=4M --bs=4k --output-format=json > fio-output.json

jq '.jobs[].read | {bw_bytes, iops, lat_ns}' fio-output.json

{
  "bw_bytes": 10230009,
  "iops": 2497.560976,
  "lat_ns": {
    "min": 106591,
    "max": 6256151,
    "mean": 397779.107422,
    "stddev": 515205.407586
  }
}

jq '.jobs[].write | {bw_bytes, iops, lat_ns}' fio-output.json

{
  "bw_bytes": 0,
  "iops": 0,
  "lat_ns": {
    "min": 0,
    "max": 0,
    "mean": 0,
    "stddev": 0
  }
}
```



## 블록 계층이 하드 디스크 성능에 주는 영향
- 블록 계층의 I/O 스케줄링과 미리 읽기 기능을 각각 유효/무효화 하여 fio로 성능 측정
  - I/O 스케줄링 무효화: /sys/block/<장치명>/queue/scheduler에 none 입력
  - 미리 읽기 무효화: /sys/block/<장치명>/queue/read_ahead_kb에 0 입력
- 두가지 패턴의 성능 측정
  - 패턴 A: I/O 스케줄러 효과, 크기가 작은 여러 데이터 무작위 쓰기 (latency,IOPS)
    - 무효화 > none 설정
    - 유효화 > mq-deadline 설정
  - 패턴 B: 미리 읽기 효과, 크기가 큰 하나의 데이터 순차 읽기 (throughput)
    - 무효화 > read_ahead_kb 0 설정
    - 유효화 > read_ahead_kb 128 설정 (기본값)

./measure.sh <disktype>.conf
- hdd, ssd 등 저장 장치 설정값 전달하여 성능 측정 후 이미지 생성

### 패턴 A 측정 결과
![990](https://github.com/user-attachments/assets/3ec16f4f-198f-4e34-8c37-aae9dce8c365)


- I/O 스케줄러로 인해 디스크 입출력 요청이 효율적인 순서로 변경됨
  - IOPS 증가
  - 레이턴시 감소


### 패턴 B 측정 결과
![993](https://github.com/user-attachments/assets/832327b6-34b6-4e80-a305-d00436f4bfb8)


- 미리 읽기 효과로 인해 단위 시간당 데이터 전송량 증가
  - 스루풋 증가


### 기술 혁신에 따른 블록 계층의 변화
- 기계적 동작이 필요한 hdd대신, 전기 신호로 처리하는 ssd의 등장
- 무작위 접근 성능에서 큰 개선
- hdd와 동일 인터페이스 사용하는 SATA,SAS SSD / 별도 인터페이스 사용하는 NVMe SSD
- 성능 상승 및 가격 상승

- 높은 IOPS를 위해 많은 논리 CPU에서 병렬작업 필요
  - 기존 I/O 스케줄러는 다수 논리 CPU 호출해도 하나의 논리CPU에서 처리하여 확장성 X
  - 멀티 큐(multi-queue) 방식을 통해 다수 CPU에서 동작
  - mq-deadline 스케줄러 > multi-queue 방식 사용
- I/O 스케줄러 사용시 재정렬 과정에서 오히려 성능 저하
  - 하드웨어 성능향상으로 인한 바로 처리 > I/O 스케줄러 대기 후 재정렬
  - 우분투 20.04는 ssd 사용 시  I/O 스케줄러 미사용 기본

## 블록 계층이 NVMe SSD 성능에 미치는 영향

### 패턴 A 측정 결과

![991](https://github.com/user-attachments/assets/a2851f8a-f9eb-4764-9d82-0df8fabdad0e)
![992](https://github.com/user-attachments/assets/9a2d0f62-0a48-424c-8664-e04e6c52d5f6)


- I/O 스케줄러로 인해 일부 구간 성능 손실 발생
  - IOPS 관점
    - 무효 및 병렬도 낮은 구간의 IOPS 높음
  - 레이턴시 관점
    - 무효 및 병렬도 낮은 구간의 레이턴시 감소 
    - 유효 및 병렬도 높은 구간의 레이턴시 감소 
- 기본적으로 HDD 대비 IOPS등 100배 이상 상승

### 패턴 B 측정 결과
![994](https://github.com/user-attachments/assets/9b242004-1114-4112-940b-6e58d92f44d7)


- 미리 읽기 효과로 인해 단위 시간당 데이터 전송량 증가
  - 스루풋 증가
  - I/O 활성화 시 성능 감소
- 기본적으로 HDD 대비 스루풋 상승 (GiB 단위)

#### 실제 성능 측정


![image](https://github.com/user-attachments/assets/6bb3ed14-60d3-4e16-9f22-7b32b91a84bf)
[Identifying Performance Bottlenecks: Tips and Techniques](https://blog.nashtechglobal.com/identifying-performance-bottlenecks-tips-and-techniques/)

- Performance bottleneck, applicatiom perfomance management(APM)/analysis, etc...

- 실제 성능 측정시에는 다양한 부분의 영향을 받음
  - 네트워크 성능 (대역폭)
  - 어플리케이션 성능 (CPU, 메모리 등)
  - 저장 장치 성능 (스토리지)
- 각 처리에 소요되는 시간 측정 필요


![image](https://github.com/user-attachments/assets/67f2dfd5-d537-4f78-80e5-e14b07ec8552)  
![image](https://github.com/user-attachments/assets/4b4d2998-76f3-461a-bf3d-8329dd7e7375)    

- apm, (distributed) tracing, observability, etc...

### 저장장치 성능 측정

![image](https://github.com/user-attachments/assets/8c05994f-7a9f-41dd-b5b8-5497fabe2648)
[Increase IOPS and throughput with sharding](https://planetscale.com/blog/increase-iops-and-throughput-with-sharding)      
- 결국엔 자기네 솔루션 쓰라는거긴 한데
- 워크로드에 따라 동일한 스토리지라도 기록된 성능만큼 안나올 수 있다

