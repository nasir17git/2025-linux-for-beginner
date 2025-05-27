# 가상화 기능
가상머신(Virtual Machine)의 구현 과정 및 동작 방식 확인

## 가상화 기능이란 무엇인가
- 가상화 기능(virtualization)
  - 하드웨어 자원을 추상화하여 하나의 컴퓨터에서 여러 개의 가상 컴퓨터를 실행할 수 있도록 하는 기술
  - PC 또는 서버등 물리 기기에서 VM 을 동작하는 소프트웨어 기능 및 동작을 돕는 하드웨어 기능의 조합

- 사용 예시
  - 하드웨어 최대 활용: 1대의 물리 기기에서 여러대의 VM 활용. ex) IaaS
  - 서버 통합: 수십대의 물리기기 시스템을 VM으로 대체해 적은 물리 기기에 집약
  - 레거시 시스템 수명연장: 하드웨어 지원이 끝난 레거시 시스템을 VM으로 대체해 수명 연장
  - 특정 OS를 다른 OS에서 실행: 윈도우에서 리눅스 또는 그 반대
  - 개발,테스트 환경: 운영 환경과 유사/동일한 환경 구축

## 가상화 기능의 종류
- 가상화 기능은 크게 두 가지 종류로 나뉨
  - 하드웨어 기반 가상화
  - 소프트웨어 기반 가상화



## 가상화 소프트웨어
- 물리기기에 설치된 가상화 SW가 VM의 생성, 관리, 삭제를 담당
- 물리 기기의 하드웨어 자원을 가상머신에 분배, 물리 기기의 CPU를 물리 CPU(Physical CPU,PCPU)와 가상 CPU(Virtual CPU,VCPU)로 분리
- 커널과 그 위의 프로세스와 유사한 가상화 SW와 그 위의 VM 

## 이 장에서 사용하는 가상화 소프트웨어
- 3종류의 가상화 SW 활용
  1. KVM (Kernel-based Virtual Machine): 리눅스 커널이 제공하는 가상화 기능
  2. QEMU: CPU 및 하드웨어 에뮬레이터
  3. virt-manager: VM 생성, 관리, 삭제 지원. 생성 이후 실행은 QEMU 담당
- 물리기기의 OS를 호스트OS, VM의 OS를 게스트OS라고 지칭
- 가상화 SW는 커널입장에선 프로세스 중 하나
- 생성 및 삭제 흐름
  1. virt-manager가 VM 기반 형태 생성(CPU 개수, 메모리 용량, 기타 하드웨어)
  2. virt-manager가 생성한 파일을 토대로 QEMU가 가상머신 생성
  3. QEMU+KVM 연계하여 필요한 만큼 가상 머신 실행 (전원 켜기, 끄기, 재시작 포함)
  4. virt-manager가 가상머신 삭제
- virt-manager의 가상 입출력장치로 VM제어 (가상 디스플레이, 입력장치-키마, DVD드라이브 등)


## 가상화를 지원하는 CPU 기능
- 기존의 사용자모드/커널모드
  - 사용자모드: CPU가 프로세스 실행시 일반적인 동작구역
  - 커널모드: 시스템콜 또는 인터럽트 발생시 커널이 사용하는 모드

- 가상화 CPU에서 VMX-root/nonroot 모드로 확장
  - VMX-nonroot: VM 처리,
  - VMX-root: 물리기기처리, VM 작동시 하드웨어 접근 또는 인터럽트 발생시 전환 

- CPU 가상화 지원 기능
  - 인텔은 VT-x, AMD는 SVM. 기능은 유사하지만 CPU레벨의 명령어셋이 다름. > KVM이 추상화


### QEMU + KVM 조합

## 가상 머신은 호스트 OS에서 어떻게 보이는가?
- 가상 머신 생성
  - `virt-install --name ubuntu2004 --vcpus 1 --cpuset=0 --memory 8192 --os-variant ubuntu20.04 --graphics none --extra-args 'console=ttyS0 --- console=ttyS0' --location http://us.archive.ubuntu.com/ubuntu/dists/focal/main/installer-amd64/`
- 이후 virsh CUI  명령어를 통해 가상머신 조작
  - virsh dumpxml ubuntu2004 > 가상 머신의 name,memory,cpu,disk,network 등 설정 정보 확인


### 호스트 OS에서 본 게스트 OS
- virsh
  - `virsh list` : 가상머신 목록 확인
  - `virsh start ubuntu2004` : 가상머신 시작
  - `virsh destroy ubuntu2004` : 가상머신 종료
  - `virsh undefine ubuntu2004` : 가상머신 삭제

- ps ax | grep qemu 
  - qemu-system-x86_64 프로세스 확인 > 동작중인 VM의 실체
  - virsh dumpxml ubuntu2004 로 조회한 구성정보가 인수로 변환되어 들어감
```
ps ax | grep qemu
 146768 ?        Sl     0:43 /usr/bin/qemu-system-x86_64 -name guest=ubuntu2004,debug-threads=on -S -object secret,id=masterKey0,format=raw,file=/home/nasir/.config/libvirt/qemu/lib/domain-2-ubuntu2004/master-key.aes -machine pc-q35-4.2,accel=tcg,usb=off,dump-guest-core=off -cpu qemu64 -m 8192 -overcommit mem-lock=off -smp 1,sockets=1,cores=1,threads=1 -uuid 954c1f7e-50a1-4880-84db-4f1fa541ccf9 -display none -no-user-config -nodefaults -chardev socket,id=charmonitor,fd=27,server,nowait -mon chardev=charmonitor,id=monitor,mode=control -rtc base=utc,driftfix=slew -global kvm-pit.lost_tick_policy=delay -no-hpet -no-shutdown -global ICH9-LPC.disable_s3=1 -global ICH9-LPC.disable_s4=1 -boot strict=on -device pcie-root-port,port=0x8,chassis=1,id=pci.1,bus=pcie.0,multifunction=on,addr=0x1 -device pcie-root-port,port=0x9,chassis=2,id=pci.2,bus=pcie.0,addr=0x1.0x1 -device pcie-root-port,port=0xa,chassis=3,id=pci.3,bus=pcie.0,addr=0x1.0x2 -device pcie-root-port,port=0xb,chassis=4,id=pci.4,bus=pcie.0,addr=0x1.0x3 -device pcie-root-port,port=0xc,chassis=5,id=pci.5,bus=pcie.0,addr=0x1.0x4 -device pcie-root-port,port=0xd,chassis=6,id=pci.6,bus=pcie.0,addr=0x1.0x5 -device ich9-usb-ehci1,id=usb,bus=pcie.0,addr=0x1d.0x7 -device ich9-usb-uhci1,masterbus=usb.0,firstport=0,bus=pcie.0,multifunction=on,addr=0x1d -device ich9-usb-uhci2,masterbus=usb.0,firstport=2,bus=pcie.0,addr=0x1d.0x1 -device ich9-usb-uhci3,masterbus=usb.0,firstport=4,bus=pcie.0,addr=0x1d.0x2 -device virtio-serial-pci,id=virtio-serial0,bus=pci.2,addr=0x0 -blockdev {"driver":"file","filename":"/home/nasir/.local/share/libvirt/images/ubuntu2004.qcow2","node-name":"libvirt-1-storage","auto-read-only":true,"discard":"unmap"} -blockdev {"node-name":"libvirt-1-format","read-only":false,"driver":"qcow2","file":"libvirt-1-storage","backing":null} -device virtio-blk-pci,scsi=off,bus=pci.3,addr=0x0,drive=libvirt-1-format,id=virtio-disk0,bootindex=1 -netdev user,id=hostnet0 -device virtio-net-pci,netdev=hostnet0,id=net0,mac=52:54:00:32:12:79,bus=pci.1,addr=0x0 -chardev pty,id=charserial0 -device isa-serial,chardev=charserial0,id=serial0 -chardev socket,id=charchannel0,fd=29,server,nowait -device virtserialport,bus=virtio-serial0.0,nr=1,chardev=charchannel0,id=channel0,name=org.qemu.guest_agent.0 -device virtio-balloon-pci,id=balloon0,bus=pci.4,addr=0x0 -object rng-random,id=objrng0,filename=/dev/urandom -device virtio-rng-pci,rng=objrng0,id=rng0,bus=pci.5,addr=0x0 -sandbox on,obsolete=deny,elevateprivileges=deny,spawn=deny,resourcecontrol=deny -msg timestamp=on
 146885 pts/0    Sl+    0:00 virsh --connect qemu:///session console ubuntu2004
 ```


### 여러 머신을 실행하는 경우
- virt-manager를 사용하여 복제 가능
  - virt-clone -o ubuntu2004 -n ubuntu2004-clone --auto-clone

- ps ax | grep qemu-system-x86_64 
  - 기존외에 복제된 프로세스 확인

## 가상화 환경과 프로세스 스케줄링
- 물리기기와 동일하게 두 프로세스가 교대로 동작
- 가상머신의 각 VCPU는, 대응하는 호스트OS 프로세스의 커널스레드가 됨

### 물리 기기에서 프로세스가 동작하는 경우
- `virt-install --name ubuntu2004 --vcpus 1 ...` 로 시작하여 VM에는 vcpu 1개만 할당되어 있음
- VCPU0 스레드외의 처리가 동작한다면 > 호스트OS의 커널스레드가 동작
- sched-virt.py 및 inf-loop.py 실행
  - 전체 소요시간 증가 및 inf-loop.py 실행시 진척없음 확인


### 통계 정보
- 호스트 OS에서 통계정보 조회시, VM(게스트OS)의 전체 통계정보가 조회되며, 구체적인 사용내역은 호스트에선 알 수 없고 VM 내부에서 확인 필요
  - sar 명령어에서 %steal 컬럼: 호스트 머신의 CPU를 다른 VM이 사용 중이라, 이 VM이 CPU를 사용하지 못하고 기다린 시간의 비율

- 호스트에서는 별도 작업X, VM 내부에서는 inf-loop.py 실행
  - 호스트에선 
    - sar) %user 100% 확인 > PCPU0 전부 사용중
    - top) qemu-system-x86_64 프로세스가 CPU 100% 사용 확인
  - 게스트에선
    - sar) %steal 0.99% 확인 > VM의 VCPU가 아닌 호스트의 PCPU 사용중이라 VM이 대기한 시간비율
    - top) inf-loop.py가 CPU 100% 사용 확인

- 호스트와 게스트 모두 inf-loop.py 실행
  - 호스트에서
    - sar) %user 100% 확인 > PCPU0 전부 사용중
    - top) qemu-system-x86_64 및 inf-loop.py 프로세스가 각각 %CPU 50% 사용
  - 게스트에서
    - sar) %user 및 %steal 각 50% 확인 > 호스트의 inf-loop.py로인해 %steal 증가, 하지만 VM은 알 수 없음
    - top) inf-loop.py가 CPU 80% 사용 확인 > VM 내부에서는 VM의 inf-loop.py가 CPU 점유하고있는것처럼 보임

## 가상 머신과 메모리 관리
- 물리기기의 메모리는 커널 메모리와 프로세스 메모리가 공존, VM 메모리도 그 중 일부 (qemu-system-x86_64 프로세스의 메모리)
- qemu-system-x86_64 프로세스의 메모리는 VM 관리용 메모리와 VM에 할당된 메모리로 나뉨
  - VM 관리용 메모리: 하드웨어 에뮬레이션용 코드 및 데이터에 사용
  - VM에 할당된 메모리: VM 내부의 커널 또는 프로세스 메모리

### 가상 머신이 사용하는 메모리
- VM 실행시 변화하는 메모리 사용량
  1. 커널메모리에 VM 관리용 메모리 추가
  2. qemu-system-x86_64 프로세스 메모리에 VM 관리용 메모리 추가
  3. qemu-system-x86_64 프로세스 메모리에 VM에 할당된 메모리 추가
  4. 페이지 캐시에 VM 기동시 읽은 디스크 이미지 영역 추가

- 구체적인 메모리 사용량 확인하기
  0. 가상 머신 정지 및 페이지 캐시 삭제(/proc/sys/vm/drop_caches에 3)
  1. 호스트 OS의 free 명령어로 호스트 OS의 메모리 사용량 확인
  2. 가상머신 실행
  3. 호스트 OS의 free 명령어로 게스트 OS의 메모리 사용량 확인
  4. 호스트 OS에서 ps 명령어로 qemu-system-x86_64 프로세스의 메모리 사용량 확인
  5. 게스트 OS에서 free 명령어로 게스트 OS의 메모리 사용량 확인


## 가상 머신과 저장 장치
- 물리기기 저장장치의 파일이, VM의 저장장치로 활용
- libvirt 설정파일을 통해 확인 가능
- virsh dumpxml ubuntu2004 | grep -A 5 disk
  - 호스트의 /home/nasir/.local/share/libvirt/images/ubuntu2004-1.qcow2 파일이 게스트의 저장장치로 활용 중

```
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/home/nasir/.local/share/libvirt/images/ubuntu2004-1.qcow2' index='1'/>
      <backingStore/>
      <target dev='vda' bus='virtio'/>
      <alias name='virtio-disk0'/>
      <address type='pci' domain='0x0000' bus='0x03' slot='0x00' function='0x0'/>
    </disk>
    <controller type='usb' index='0' model='ich9-ehci1'>
      <alias name='usb'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x1d' function='0x7'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci1'>
```

### 가상 머신과 저장소 입출력
- 물리기기의 저장소 입출력(storage I/O)는 물리기기의 유저스페이스의 프로세스 > 커널의 파일시스템 > 커널의 디바이스 드라이버 > 저장장치 순으로 작동후 역순으로 프로세스까지 복귀
- VM의 저장소 입출력은 VM내부의 유저스페이스의 프로세스 > 커널의 파일시스템 > 커널의 디바이스 드라이버 이후, 물리기기 커널의 KVM > 호스트OS의 qemu-system-x86_64 프로세스 > 물리커널의 파일시스템 > 물리커널의 디바이스 드라이버 > 저장장치를 거쳐 역순으로 프로세스까지 복귀
- 따라서 물리기기 직접사용에 비해 저장소 입출력 성능이 대폭 나빠짐

- 이를 완화하기 위해 반가상화(para virtualization) 기능 활용

### 저장 장치 쓰기와 페이지 캐시
- VM의 저장장치 파일에 데이터 기록시, 물리기기의 qemu-system-x86_64 프로세스의 작업은 동기? 페이지 캐싱 및 직접 입출력은 가능?
- libvirt 설정에 따라 동작
  - <driver> 태그의 캐시속성으로 지정됨
  - writeback 기본 입출력 캐시속성 사용 > 쓰기는 비동기적, 페이지캐싱 사용 > VM의 작업이 호스트반영간 딜레이 존재 가능
  - writethrough 설정시 쓰기 작업 동기화 가능

### 반가상화 장치와 virtio-blk
- 반가상화(para virtualization)
  - VM에서 하드웨어를 완전히 에뮬레이션 하는 대신, 가상화 SW및 VM이 별도 인터페이스로 접속해 성능 개선
  - 이러한 장치를 반가상화 장치, 디바이스 드라이버를 반가상화 드라이버라고 지칭

#### 호스트 OS와 게스트 OS의 저장소 입출력 성능 역전 현상
- 직접 입출력을 사용하여 파일 쓰기 및 읽기 작업 수행시
  - 호스트 OS의 쓰기 성능이 게스트 OS보다 빠름
  - 호스트 OS의 읽기 성능이 게스트 OS보다 느릴 수 있음
- 쓰기 작업시 저장장치까지의 경로가 호스트OS가 더 짧기에 호스트 OS의 쓰기 성능이 빠름
- 읽기 작업시 호스트OS는 저장장치까지가서 읽어야하지만, 게스트OS는 (호스트 OS의 페이지캐시에 남아있다면) 호스트 OS의 캐시메모리에서 읽을 수 있어서 빠를 수 있음

### virtio-blk 구조
- virtio-blk는 호스트OS와 게스트OS가 공유하는 캐시를 사용해 가상화IO장치(virtio)에 접근하여 입출력 속도 향상
  1. 게스트 OS의 virtio-blk 드라이버 큐에 명령어 삽입
  2. virtio-blk 드라이버는 호스트OS에 제어 넘김
  3. 호스트OS의 가상화 SW가 큐에서 명령어 꺼내서 처리
  4. 가상화 SW가 VM에 제어권 넘김
  5. virtio-blk 장치는 명령어 실행 결과를 수신
- 처리 1에 다수의 명령어를 전달할 수 있어 완전가상화보다 빠르게 동작 가능

- 완전 가상화 장치에서 저장장치에 쓰기 작업시
  1. 어떤 크기로 써야하는지 장치에 지시
  2. 어느 위치에 써야할지 장치에 지시
  3. 1,2 작업을 수행하도록 지시
- 이때 CPU는 VMX-nonroot, VMX-root 모드를 왔다갔다 하면서 작업 수행
- virtio-blk 장치에서는 다수의 명령어를 한꺼번에 큐에 넣고 처리할 수 있어 장치 접근 및 모드전환이 한번만 수행되어 시간 절감

#### PCI 패스스루
- VM에서 물리장치를 직접 호출하여 성능 개선
  - VM이 가상화 SW를 통해 물리장치에 접근하는 전가상화
  - VM이 virtio를 통해 물리장치에 접근하는 반가상화 대비 빠름