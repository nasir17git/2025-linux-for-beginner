# 프로세스 관리 (기초편)
리눅스가 프로세스를 관리하는 프로세스 관리 시스템에 대한 설명

- ps aux
- ps -aux --no-header | wc -l
- 어떤 프로세스가 있고, 어떻게 관리되고 있는지
```
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0 171196 14228 ?        Ss   21:03   0:00 /sbin/init
root           2  0.0  0.0   2784  1928 ?        Sl   21:03   0:00 /init
root           7  0.0  0.0   2776   132 ?        Sl   21:03   0:00 plan9 --control-socket 7 --log-level 4 --server-fd 8 --pipe-fd 1
root          52  0.0  0.0  52152 15156 ?        S<s  21:03   0:00 /lib/systemd/systemd-journald
...

ps -aux --no-header | wc -l
# 71
```



## 프로세스 생성
- 새로운 프로세스의 생성 목적
  - 동일한 프로그램 처리를 다수 프로세스의 나눠서 처리
    - ex) 웹서버에서 다수의 요청 받기
  - 다른 프로그램을 생성하기
    - ex) bash에서 각종 프로그램을 새로 생성

- 실제 생성을 위해, fork(), execve() 함수 사용
  - 내부적으로 clone(), execve() 시스템 콜 호출
  - 동일 프로그램을 다수 프로세스 처리 > fork() 사용
  - 다른 프로그램 생성 시 fork(), execve() 사용


### 같은 프로세스를 두 개로 분열시키는 fork() 함수
fork() 함수 호출 시 해당 프로세스 사본 생성 후 복귀     
원본을 부모 프로세스(parent process), 생성된 프로세스를 자식 프로세스 (child process)     


순서
1. 부모 프로세스가 fork() 함수 호출
2. 자식 프로세스 메모리 확보 후 부모 프로세스 메모리 복사
3. 부모,자식 모두 fork()복귀
4. fork() 함수 반환값이 달라 분기 처리 가능

https://www.google.com/url?sa=i&url=https%3A%2F%2Fpythontic.com%2Fmodules%2Fos%2Ffork&psig=AOvVaw0JcKD04-vJ80PGNbKUM_Dq&ust=1742905117675000&source=images&cd=vfe&opi=89978449&ved=0CBQQjRxqFwoTCKjAwdnZoowDFQAAAAAdAAAAABAh

- 실제 메모리 복사 시 Copy-on-Write 기능(7장)으로 인해 적은 비용으로 가능
- 동일 프로그램 다수 프로세스 처리시 생기는 오버헤드는 많지 않다


- fork.py 호출 시 3831 부모프로세스가 호출되고, 분기해서 새로운 자식인 3832 생성
```
./fork.py
# 부모 프로세스: pid=3831, 자식 프로세스의 pid=3832
# 자식 프로세스: pid=3832, 부모 프로세스의 pid=3831
```


### 다른 프로그램을 기동하는 execve() 함수
fork() 함수로 프로세스 사본을 만들었으면, 자식 프로세스에서 execve()함수 호출 > 새로운 프로그램으로 변경  

1. execve() 함수 호출
2. execve() 함수 인수로 지정한 실행 파일에서 프로그램 읽은 뒤, 메모리에 배치(메모리맵) 에 필요 정보 가져옴
3. 현재 프로세스의 메모리를 새로운 프로세스 데이터로 덮어씀
4. 프로세스를 새로운 프로세스의 최초 실행명령(Entrypoint) 부터 실행

> 다시말해, fork()함수는 프로세스가 늘어나지만, 다른 프로그램 생성시 프로세스 치환형태로 작동

- fork-and-exec.py 호출시 4216 부모에서 4217 자식은 동일
- execve()로 인해 자식인 4217은 별도 프로그램으로 echo 실행
```
fork-and-exec.py
부모 프로세스: pid=4216, 자식 프로세스 pid=4217
자식 프로세스: pid=4217, 부모 프로세스 pid=4216
pid=4217에서 안녕   
```

execve() 함수의 동작을 위해서는 프로그램코드,데이터 외에 다음 데이터 필요
- 코드 영역의 파일 오프셋, 크기 및 메모리 맵 시작 주소
- 데이터 영역의 파일 오프셋, 크기 및 메모리 맵 시작 주소
- 최초로 실행할 명령의 메모리 주소(엔트리포인트)

리눅스 실행 파일은 보통 ELF(Executable and Linking Format) 포맷 사용
각종 정보는 readelf 명령어로 확인

- readelf -h pause > Entry point address:               0x401050 < 엔트리 포인트
  -  -h --file-header       Display the ELF file header
- readelf -S pause > 코드 및 데이터의 파일 오프셋, 크기, 메모리맵 시작 주소 확인
  -  -S --section-headers   Display the sections' header
  - 실행 파일은 각각 섹션이라고 부르는 여러 영역으로 구성됨
  - 섹션 정보는 두줄이 한묶음 / 숫자는 16진수
  - 섹션 주요 정보
    - 섹션명 (Name)
    - 메모리맵 시작 주소 (Address)
    - 파일 오프셋 (Offset)
    - 크기 (Size)
```
readelf -h pause
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x401050
  Start of program headers:          64 (bytes into file)
  Start of section headers:          14648 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         31
  Section header string table index: 30

readelf -S pause
There are 31 section headers, starting at offset 0x3938:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000400318  00000318
       000000000000001c  0000000000000000   A       0     0     1
  [ 2] .note.gnu.propert NOTE             0000000000400338  00000338
       0000000000000020  0000000000000000   A       0     0     8
  [ 3] .note.gnu.build-i NOTE             0000000000400358  00000358
       0000000000000024  0000000000000000   A       0     0     4
  [ 4] .note.ABI-tag     NOTE             000000000040037c  0000037c
       0000000000000020  0000000000000000   A       0     0     4
  [ 5] .gnu.hash         GNU_HASH         00000000004003a0  000003a0
       000000000000001c  0000000000000000   A       6     0     8
  [ 6] .dynsym           DYNSYM           00000000004003c0  000003c0
       0000000000000060  0000000000000018   A       7     1     8
  [ 7] .dynstr           STRTAB           0000000000400420  00000420
       000000000000003e  0000000000000000   A       0     0     1
  [ 8] .gnu.version      VERSYM           000000000040045e  0000045e
       0000000000000008  0000000000000002   A       6     0     2
  [ 9] .gnu.version_r    VERNEED          0000000000400468  00000468
       0000000000000020  0000000000000000   A       7     1     8
  [10] .rela.dyn         RELA             0000000000400488  00000488
       0000000000000030  0000000000000018   A       6     0     8
  [11] .rela.plt         RELA             00000000004004b8  000004b8
       0000000000000018  0000000000000018  AI       6    24     8
  [12] .init             PROGBITS         0000000000401000  00001000
       000000000000001b  0000000000000000  AX       0     0     4
  [13] .plt              PROGBITS         0000000000401020  00001020
       0000000000000020  0000000000000010  AX       0     0     16
  [14] .plt.sec          PROGBITS         0000000000401040  00001040
       0000000000000010  0000000000000010  AX       0     0     16
  [15] .text             PROGBITS         0000000000401050  00001050
       0000000000000175  0000000000000000  AX       0     0     16
  [16] .fini             PROGBITS         00000000004011c8  000011c8
       000000000000000d  0000000000000000  AX       0     0     4
  [17] .rodata           PROGBITS         0000000000402000  00002000
       0000000000000004  0000000000000004  AM       0     0     4
  [18] .eh_frame_hdr     PROGBITS         0000000000402004  00002004
       0000000000000044  0000000000000000   A       0     0     4
  [19] .eh_frame         PROGBITS         0000000000402048  00002048
       0000000000000100  0000000000000000   A       0     0     8
  [20] .init_array       INIT_ARRAY       0000000000403e10  00002e10
       0000000000000008  0000000000000008  WA       0     0     8
  [21] .fini_array       FINI_ARRAY       0000000000403e18  00002e18
       0000000000000008  0000000000000008  WA       0     0     8
  [22] .dynamic          DYNAMIC          0000000000403e20  00002e20
       00000000000001d0  0000000000000010  WA       7     0     8
  [23] .got              PROGBITS         0000000000403ff0  00002ff0
       0000000000000010  0000000000000008  WA       0     0     8
  [24] .got.plt          PROGBITS         0000000000404000  00003000
       0000000000000020  0000000000000008  WA       0     0     8
  [25] .data             PROGBITS         0000000000404020  00003020
       0000000000000010  0000000000000000  WA       0     0     8
  [26] .bss              NOBITS           0000000000404030  00003030
       0000000000000008  0000000000000000  WA       0     0     1
  [27] .comment          PROGBITS         0000000000000000  00003030
       000000000000002b  0000000000000001  MS       0     0     1
  [28] .symtab           SYMTAB           0000000000000000  00003060
       00000000000005e8  0000000000000018          29    45     8
  [29] .strtab           STRTAB           0000000000000000  00003648
       00000000000001ca  0000000000000000           0     0     1
  [30] .shstrtab         STRTAB           0000000000000000  00003812
       000000000000011f  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)

```

- 프로그램에서 작성한 프로세스의 메모리맵은 /proc/<pid>/maps 파일에서 확인가능
- 시작 메모리맵의 값에 따라 필요한 정보 유형을 확인 가능
  - 00400*는 코드파일 오프셋 > 코드 영역
  - 00610은 없는데..?
```
❯ ./pause &
[1] 5583
❯ cat /proc/5583/maps
00400000-00401000 r--p 00000000 08:20 112354                             /home/nasir/2025-linux-for-beginner/02/pause
00401000-00402000 r-xp 00001000 08:20 112354                             /home/nasir/2025-linux-for-beginner/02/pause
00402000-00403000 r--p 00002000 08:20 112354                             /home/nasir/2025-linux-for-beginner/02/pause
00403000-00404000 r--p 00002000 08:20 112354                             /home/nasir/2025-linux-for-beginner/02/pause
00404000-00405000 rw-p 00003000 08:20 112354                             /home/nasir/2025-linux-for-beginner/02/pause
7f20d2730000-7f20d2752000 r--p 00000000 08:20 43427                      /usr/lib/x86_64-linux-gnu/libc-2.31.so
7f20d2752000-7f20d28ca000 r-xp 00022000 08:20 43427                      /usr/lib/x86_64-linux-gnu/libc-2.31.so
7f20d28ca000-7f20d2918000 r--p 0019a000 08:20 43427                      /usr/lib/x86_64-linux-gnu/libc-2.31.so
7f20d2918000-7f20d291c000 r--p 001e7000 08:20 43427                      /usr/lib/x86_64-linux-gnu/libc-2.31.so
7f20d291c000-7f20d291e000 rw-p 001eb000 08:20 43427                      /usr/lib/x86_64-linux-gnu/libc-2.31.so
7f20d291e000-7f20d2924000 rw-p 00000000 00:00 0 
7f20d2930000-7f20d2931000 r--p 00000000 08:20 43407                      /usr/lib/x86_64-linux-gnu/ld-2.31.so
7f20d2931000-7f20d2954000 r-xp 00001000 08:20 43407                      /usr/lib/x86_64-linux-gnu/ld-2.31.so
7f20d2954000-7f20d295c000 r--p 00024000 08:20 43407                      /usr/lib/x86_64-linux-gnu/ld-2.31.so
7f20d295d000-7f20d295e000 r--p 0002c000 08:20 43407                      /usr/lib/x86_64-linux-gnu/ld-2.31.so
7f20d295e000-7f20d295f000 rw-p 0002d000 08:20 43407                      /usr/lib/x86_64-linux-gnu/ld-2.31.so
7f20d295f000-7f20d2960000 rw-p 00000000 00:00 0 
7ffe7e756000-7ffe7e778000 rw-p 00000000 00:00 0                          [stack]
7ffe7e7e8000-7ffe7e7ec000 r--p 00000000 00:00 0                          [vvar]
7ffe7e7ec000-7ffe7e7ee000 r-xp 00000000 00:00 0                          [vdso]
```

### ASLR로 보안 강화
리눅스 커널의 Address Space Layout Randomization(ASLR) 보안기능
- 프로그램 실행마다 각 섹션을 다른 주소에 매핑
- 따라서 공격 대상 코드/데이터가 고정 특정 주소에 존재하지 않게 됨
- 필요 조건
  - 커널의 ASLR 유효
  - 프로그램의 ASLR 대응, PIE(Position Independent Execuable)

gcc의 기본값은 모든 프로그램을 PIE로 빌드, -no-pie 옵션시 PIE 빌드 X  
- 프로그램의 PIE 지원 여부는 file 명령어로 확인

```
file pause
# PIE가 아닐 경우 > LSB executable
pause: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=8cf45be52e3dc7788b28be05b46989a4796750fd, for GNU/Linux 3.2.0, not stripped

# PIE 인 경우 > LSB shared object
cc -o yespie-pause pause.c 
file yespie-pause
yespie-pause: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=de5546a21c780cd40dc75c24eeb1c7fa535c4112, for GNU/Linux 3.2.0, not stripped
```

- 각 2번 실행결과로확인
  - No PIE > 같은 메모리맵
  - Yes PIE > 다른 메모리맵
```
# No PIE
❯ ./pause &
[1] 7425
❯ ./pause &
[2] 7431
❯ cat /proc/7425/maps
00400000-00401000 r--p 00000000 08:20 112354                             /home/nasir/2025-linux-for-beginner/02/pause
00401000-00402000 r-xp 00001000 08:20 112354                             /home/nasir/2025-linux-for-beginner/02/pause
00402000-00403000 r--p 00002000 08:20 112354                             /home/nasir/2025-linux-for-beginner/02/pause
00403000-00404000 r--p 00002000 08:20 112354                             /home/nasir/2025-linux-for-beginner/02/pause
00404000-00405000 rw-p 00003000 08:20 112354                             /home/nasir/2025-linux-for-beginner/02/pause
7fe3b97a0000-7fe3b97c2000 r--p 00000000 08:20 43427                      /usr/lib/x86_64-linux-gnu/libc-2.31.so
...
❯ cat /proc/7431/maps
00400000-00401000 r--p 00000000 08:20 112354                             /home/nasir/2025-linux-for-beginner/02/pause
00401000-00402000 r-xp 00001000 08:20 112354                             /home/nasir/2025-linux-for-beginner/02/pause
00402000-00403000 r--p 00002000 08:20 112354                             /home/nasir/2025-linux-for-beginner/02/pause
00403000-00404000 r--p 00002000 08:20 112354                             /home/nasir/2025-linux-for-beginner/02/pause
00404000-00405000 rw-p 00003000 08:20 112354                             /home/nasir/2025-linux-for-beginner/02/pause
7fb8f2dd4000-7fb8f2df6000 r--p 00000000 08:20 43427                      /usr/lib/x86_64-linux-gnu/libc-2.31.so
...

# Yes PIE
❯ ./yespie-pause &
[3] 7676
❯ ./yespie-pause &
[4] 7679
❯ cat /proc/7676/maps
55bf73895000-55bf73896000 r--p 00000000 08:20 10319                      /home/nasir/2025-linux-for-beginner/02/yespie-pause
55bf73896000-55bf73897000 r-xp 00001000 08:20 10319                      /home/nasir/2025-linux-for-beginner/02/yespie-pause
55bf73897000-55bf73898000 r--p 00002000 08:20 10319                      /home/nasir/2025-linux-for-beginner/02/yespie-pause
55bf73898000-55bf73899000 r--p 00002000 08:20 10319                      /home/nasir/2025-linux-for-beginner/02/yespie-pause
55bf73899000-55bf7389a000 rw-p 00003000 08:20 10319                      /home/nasir/2025-linux-for-beginner/02/yespie-pause
7fdd7c224000-7fdd7c246000 r--p 00000000 08:20 43427                      /usr/lib/x86_64-linux-gnu/libc-2.31.so
...

❯ cat /proc/7679/maps
561f7ebc4000-561f7ebc5000 r--p 00000000 08:20 10319                      /home/nasir/2025-linux-for-beginner/02/yespie-pause
561f7ebc5000-561f7ebc6000 r-xp 00001000 08:20 10319                      /home/nasir/2025-linux-for-beginner/02/yespie-pause
561f7ebc6000-561f7ebc7000 r--p 00002000 08:20 10319                      /home/nasir/2025-linux-for-beginner/02/yespie-pause
561f7ebc7000-561f7ebc8000 r--p 00002000 08:20 10319                      /home/nasir/2025-linux-for-beginner/02/yespie-pause
561f7ebc8000-561f7ebc9000 rw-p 00003000 08:20 10319                      /home/nasir/2025-linux-for-beginner/02/yespie-pause
7fbc1cdb8000-7fbc1cdda000 r--p 00000000 08:20 43427                      /usr/lib/x86_64-linux-gnu/libc-2.31.so
...
```

## 프로세스의 부모 자식 관계
프로세스 생성시 부모프로세스가 자식프로세스 생성 > 어디까지?
- 시스템 부팅시 초기화 순서
1. 컴퓨터 전원 시작
2. BIOS, UEFI 펌웨어 기동 및 하드웨어 초기화
3. 펌웨어가 GRUB 같은 부트로더 기동
4. 부트로더가 OS 커널 기동(ex. linux kernel)
5. 리눅스 커널이 init 프로세스 기동
6. init 프로세스가 자식..자식..자식... 프로세스 트리 구성

```
pstree -p
systemd(1)─┬─ModemManager(345)─┬─{ModemManager}(357)
           │                   └─{ModemManager}(362)
           ├─Relay(836)(835)─┬─wslconnect(1074)───zsh(1075)
           │                 ├─zsh(1135)
           │                 ├─zsh(1136)
           │                 └─zsh(1139)───gitstatusd-linu(1144)─┬─{gitstatusd-linu}(1145)
           │                                                     ├─{gitstatusd-linu}(1146)
...
           ├─init-systemd(Ub(2)─┬─SessionLeader(1249)───Relay(1251)(1250)─┬─sh(1251)───sh(1252)───sh(1257)───node(1261)─┬─node(131
           │                    │                                         │                                             ├─node(135
           │                    │                                         │                                             ├─node(136
           │                    │                                         │                                             ├─{node}(1
...
```





### fork() 함수와 execve() 함수 이외의 프로세스 생성 방법
프로세스에서 fork() 및 execve() 함수 순서대로 호출은 번거로움
유닉스 계열 OS의 C언어 인터페이스 규격인 POSIX에 정의된 posix_spawn() 함수 사용시 간단히 처리 가능

```
spawn.py
ss
```




## 프로세스 상태
프로세스 상태 (Process state)
- 많은 프로세스들이 항상 CPU 먹고있는건 아님
- 기동 시각/CPU 사용 시간은 ps aux 의 START 및 TIME 필드에서 확인 가능
- 프로세스 상태는 STAT 필드에서 확인 가능
  - S > sleep 
  - R > runnable / running? > 3장 타임슬라이스/컨텍스트 스위치절에서 확인
  - Z > zombie > 이후 소멸
- 모든 프로세스가 슬립상태라면, idle process 동작 > 이거그거죠 노트북 덮어놓는거
```
ps aux --sort=-time

USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
nasir       1352  2.1  1.6 76974256 260392 pts/3 Sl+  19:54   0:38 /home/nasir/.cursor-server/bin/906121b8c0bdf041c14a15dac228e66ab
nasir       1261  0.1  0.5 11802420 94904 pts/3  Sl+  19:54   0:03 /home/nasir/.cursor-server/bin/906121b8c0bdf041c14a15dac228e66ab
nasir       1387  0.2  0.4 1006712 70144 pts/3   Sl+  19:54   0:03 /home/nasir/.cursor-server/bin/906121b8c0bdf041c14a15dac228e66ab
nasir       1376  0.1  0.0  18704 10596 pts/6    Ss   19:54   0:02 /usr/bin/zsh -i
root         275  0.0  0.0 446188 11356 ?        Ssl  19:54   0:01 snapfuse /var/lib/snapd/snaps/snapd_23771.snap /snap/snapd/23771
nasir       1304  0.0  0.3 990260 48992 pts/5    Ssl+ 19:54   0:01 /home/nasir/.cursor-server/bin/906121b8c0bdf041c14a15dac228e66ab
nasir       1364  0.1  0.4 11792300 67568 pts/3  Sl+  19:54   0:01 /home/nasir/.cursor-server/bin/906121b8c0bdf041c14a15dac228e66ab
root           1  0.0  0.0 170260 12112 ?        Ss   19:54   0:00 /sbin/init
...
nasir       8582  0.0  0.0  10784  3300 pts/6    R+   20:25   0:00 ps aux --sort=-time
```

https://www.google.com/url?sa=i&url=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FProcess_state&psig=AOvVaw3sbEp_Eq3pK7jb7hpdDKH6&ust=1742905998558000&source=images&cd=vfe&opi=89978449&ved=0CBQQjRxqFwoTCJih4PjcoowDFQAAAAAdAAAAABA_

## 프로세스 종료
exit_group() 시스템 콜 호출
- 프로그래밍 언어에서 exit()함수 호출 시 해당 시스템콜 호출 
- or 직접 안불러도 libc에서 호출함
- 커널이 메모리같은 프로세스에 할당된 자원을 회수

프로세스 종료 후, 부모 프로세스는 wait(), waitpid() 시스템콜 호출하여 다음 정보확인
- 프로세스 반환값, exit()인수에 저장된 값
  - 예를 들어 프로세스 반환값에 따라 프로세스의 정상, 비정상 종료 여부를 판정하여 에러로그 출력하는 후속 처리 가능
- 시그널에 따른 종료 여부
- 종료때 까지의 CPU 시간 사용 정보

- bash 내장된 wait 명령어를 사용하면 백그라운드 프로세스에 wait() 시스템콜을 호출해 종료상태 확인가능

```
./wait-ret.sh
false 명령어가 종료되었습니다: 1
```

## 좀비 프로세스와 고아 프로세스
부모는 wait()계열 시스템콜로 자식 부르기 가능   
= 자식은 종료되도 부모가 부를때까지 어떠한 형태로 남아있음    
= 죽었어도 부모가 종료를 확인하지 않았으면 좀비라고 부름    

일반적으로 부모는 자식돌보면서 종료상태를 제때 회수해 커널로 돌려줘야함   
좀비 많으면 엄마 버그임   


wait() 계열 시스템 콜을 실행 전 부모 프로세스가 종료되면 자식은 고아프로세스가 됨   

커널은 init(pid 1)을 새로운 부모로 지정
- 좀비의 엄마가 종료되면 init에 좀비 부착
- init은 정기적으로 wait() 콜로 시스템 자원 회수

## 시그널
특정 프로세스가 다른 프로세스에 신호를 보내, 외부에서 실행 순서를 강제로 변경
- 기본적으로 프로세스는 실행 순서에 따라 실행
- 조건분기또한 미리 정해진 조건문에 따라 이동

대표적으로 SIGINT 시그널
- bash 등에서 Ctrl+C > 곧바로 종료 (기본값)
- kill 명령어 kill -INT <pid> > 해당 프로세스에 SIGINT 보내기
- 그외의 시그널
  - SIGCHLD: 자식 프로세스 종료 시 부모프로세스에 보냄, 보통 시그널핸들러 내부에서 wait()계열 시스템콜 호출
  - SIGSTOP: 프로세스 실행 일시정지, bash등의 Ctrl+Z
  - SIGCONT: SIGSTOP등으로 정지된 프로세스 재개, fg

프로세스는 각 시그널에 시그널 핸들러 등록
- 프로세스 실행 중 해당 시그널 수신 시
  - 실행 처리 중단 
  - 시그널 핸들러의 처리 수행
  - 원래 작업 복귀

- 시그널 핸들러를 사용하여 Ctrl+C(SIGINT) 동작하지 않는 프로그램 생성 가능
  - SIGSTOP(Ctrl+Z)로 백그라운드로 보내고 kill 
```
./intignore.py
^C^C^C^C^C^C^C
```

## 셸 작업 관리 구현
- 작업
  - bash 등의 셸에서 백그라운드 프로세스 제어하는 동작 구조
    - 명령어 뒤의 & > 백그라운드에서 실행
    - jobs > 백그라운드 프로세스 목록
    - fg x > 포그라운드로 꺼내오기 
- 세션과 프로세스 그룹의 개념을 사용

### 세션
세션은 사용자가 시스템에 로그인 했을때 로그인 세션에 대응 하는 개념
- gterm 같은 단말 에뮬레이터 사용
- ssh 등을 통해 접속

모든 세션에는 해당 세션을 제어하는 단말(Terminal)이 존재
- 세션 내부 프로세스 조작을 위해 단말을 이용하여 프로세스 지시 & 출력 
- pty/<n>의 가상 단말이 할당됨
- 각 세션은 세션ID or SID가 할당
  - 세션 리더의 프로세스가 존재하고, 보통은 쉘임. 
  - 세션 리더 PID = SID
  - 관련 정보는 ps ajx 같은 명령어로 확인 가능
    - a (all) > 현재 터미널 외의 모든 사용자의 프로세스 포함
    - j (jobs format) > 작업(job) 관련 정보, PGID 및 SID
    - x (without controlling terminal) > 터미널에 연결되지않은, 백그라운드/데몬 포함


ps ajx
- SID 1376인 세션리더 zsh 존재
- 그안에 PID 1484 1485등이 속함
- ps ajx는 IDE 내부의 터미널에서 실행 > 11668
```
ps ajx
   PPID     PID    PGID     SID TTY        TPGID STAT   UID   TIME COMMAND
      0       1       1       1 ?             -1 Ss       0   0:00 /sbin/init
      1       2       0       0 ?             -1 Sl       0   0:00 /init
...
   1250    1484    1483    1376 pts/6      11668 S     1000   0:00 /usr/bin/zsh -i
   1250    1485    1483    1376 pts/6      11668 S     1000   0:00 /usr/bin/zsh -i
   1250    1488    1487    1376 pts/6      11668 S     1000   0:00 /usr/bin/zsh -i
   1488    1493    1487    1376 pts/6      11668 Sl    1000   0:00 /home/nasir/.cache/gitstatus/gitstatusd-linux-x86_64 -G v1.5.4 -
   1376   11668   11668    1376 pts/6      11668 R+    1000   0:00 ps ajx
...
```

- 세션 할당된 단말이 행업등으로 인해 연결이 끊기면 SIGHUP 시그널
  - 단말 창을 닫는 경우도 동일
- 이때 모든 작업이 종료 됨, 이를 방지하기 위해
  - nohup 명령어 > SIGHUP을 무시하고 프로세스 기동
  - disown 명령어 > 실행중인 명령어를 bash에서 제외

### 프로세스 그룹

프로세스 그룹은 여러 프로세스를 하나로 묶어 한번에 관리
- 세션 내부에는 여러 프로세스그룹 존재
- 셸이 만든 작업=프로세스 그룹에 해당하다고 이해 해도 무방

bash로 로그인
- go build <소스코드> &
- ps aux | less
- 위의 경우 각각 2개의 프로세스 그룹(작업) 생성

세션 내부의 프로세스 그룹은 두 종류로 구분 가능
- 포그라운드 프로세스 그룹: 셸의 포그라운드 작업, 세션당 하나만 존재하고 세션 단말에서 직접 접근 가능
- 백그라운드 프로세스 그룹: 셸의 백그라운드 작업, SIGSTOP처럼 실행 일시 중단 및 fg등으로 꺼내기전까진 백그라운드

프로세스 그룹에는 고유 ID인 PGID 할당
```
ps ajx | less
```

## 데몬
데몬은 일반 프로그램과 달리, 상주하며 항상 실행되는 프로세스
- 단말의 입출력 필요없기 때문에 할당되지 않음
- 로그인 세션을 종료해도 영향이 없도록 별도의 세션 보유
- 데몬 종료여부 고민할 필요없이 init이 부모 프로세스

ps ajx 명령어 등으로 세션 정보 확인시, TTY(연결 터미널정보) ? 로 확인 가능
