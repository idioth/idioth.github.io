---
layout: post
title: 만들면서 배우는 OS 커널의 구조와 원리 ch4. 인터럽트와 예외
tags: [kernel, os]
---

![pic](https://user-images.githubusercontent.com/23413308/78744504-b06cc980-799c-11ea-8c5c-fd5ebbcad8fc.png)

인터럽트 : CPU가 프로그램을 실행하고 있을 때, 입출력 하드웨어 장치 혹은 예외 상황이 발생하여 처리가 필요할 경우 CPU에 알려 처리할 수 있도록 하는 것



### IDT

![pic](https://user-images.githubusercontent.com/23413308/78744899-b31bee80-799d-11ea-8400-5f527db06768.png)

2번째 16비트 한 워드 : 핸들러가 위치한 코드 세그먼트의 셀렉터 값, 커널 모드 코드 세그먼트 셀렉터 값을 기입하면 됨

P 비트 : 커널 모드의 코드 세그먼트가 RAM 상에 존재하는가, 항상 1

DPL : 핸들러가 실행될 권한 레벨. 항상 커널 모드에서 동작하므로 0~3 중 0 레벨로 지정(2진수 00)

0과 1로 정해진 비트 값 : IDT에 위치한 인터럽트 디스크립터라는 것을 CPU에 알려주는 값

D 비트 : 현재 지정한 코드 세그먼트가 16비트(0)인지 32비트(1)인지. 32비트이므로 1 기입



```assembly
cld
mov ax, SysDataSelctor
mov es, ax
xor eax, eax
xor ecx, ecx
mov ax, 256
mov edi,0

loop_idt:
	leas esi, [idt_ignore]
	mov cx, 8
	rep movsb
	dec ax
	jnz loop_idt
```

RAM상에 IDT 만들어주는 부분



```assembly
idt_ignore:
	dw isr_ignore
	dw SysCodeSelctor
	db 0
	db 0x8E
	dw 0x0001
```

핸들러 isr_ignore, 코드 세그먼트 셀렉터 SysCodeSelctor, 맨 아랫줄 0x0001

물리 주소 = SysCodeSelector : 0x10000+isr_ignore

0x10000 : 프로그램이 위치한 RAM상의 주소



```assembly
isr_ignore:
	push gs
	push fs
	push es
	push ds
	pushad
	pushfd
	
	mov ax, VideoSelector
	mov es, ax
	mov edi, (80*7*2)
	lea esi, [msg_isr_ignore]
	call printf
	
	popfd
	popad
	pop ds
	pop es
	pop fs
	pop gs
	
	iret
```

CPU의 모든 레지스터 값과 FLAG를 스택에 보존 => 인터럽트 발생 전 실행되던 루틴이 어떤 레지스터를 사용하는지 모르기 때문



![pic](https://user-images.githubusercontent.com/23413308/78748429-fe86ca80-79a6-11ea-8cbe-9978f90f5bc8.png)

<b>인터럽트 호출 과정</b>

1. 0x20번의 인터럽트 발생
2. CPU는 IDTR 참조
3. IDTR은 IDT의 Base Address를 가지므로 IDT의 첫 번째 번지를 가리킴
4. 인터럽트 번호가 0x20이므로 IDT의 첫 번째 번지에서 0x20번째의 디스크립터를 찾아냄
5. 세그먼트 셀렉터 값을 가지고 GDT에서 해당하는 디스크립터 찾아냄
6. RAM 상의 해당 세그먼트의 Base Address를 찾아냄
7. IDT의 0x20번째 디스크립터에 포함된 오프셋 값을 갖고 세그먼트 범위 안 실제 위치를 찾아냄
8. 핸들러가 모두 실행되어 iret 명령이 내려지면 처음 인터럽트가 걸린 프로그램의 다음 명령 실행



### PIC

- PC의 모든 외부로부터 하드웨어 인터럽트는 8259A라는 칩을 통해 입력 받음

- 8259A를 PIC라 부름

- 한 개의 PIC는 8개의 IRQ 핀을 가짐
- PIC 두 개가 마스터, 슬레이브로 연결. 마스터의 INT 핀이 CPU의 INT 핀으로 연결
- 슬레이브의 INT 핀은 마스터의 3번째 IRQ 핀인 2번 핀에 연결
- 두 PIC 모두 /INTA 핀이 CPU의 /INTA 핀에 연결

![pic](https://user-images.githubusercontent.com/23413308/78848833-d900cc00-7a4d-11ea-94a2-03544978ca95.png)

| IRQ  | 연결된 하드웨어 장치의 인터럽트 |
| ---- | ------------------------------- |
| 0    | 타이머                          |
| 1    | 키보드                          |
| 2    | 슬레이브 PIC                    |
| 3    | COM2                            |
| 4    | COM1                            |
| 5    | 프린터 포트 2                   |
| 6    | 플로피디스크 컨트롤러           |
| 7    | 프린터 포트 1                   |
| 8    | 리얼 타임 클록                  |
| 9    | X                               |
| 10   | X                               |
| 11   | X                               |
| 12   | PS/2 마우스                     |
| 13   | Coprocessor                     |
| 14   | 하드디스크 1                    |
| 15   | 하드디스크 2                    |

마스터 PIC에 연결된 장치 중 인터럽트 발생 시

1. 자신의 INT 핀에 신호를 실어 CPU의 INT 핀에 전달
2. CPU가 신호를 받고 EFLAG의 IE 비트를 1로 세트, 인터럽트를 받을 수 있는 상황이면 /INTA를 통해 마스터 PIC에 인터럽트를 받았다고 전달
3. 마스터 PIC가 신호를 받으면 인터럽트 발생한 IRQ 숫자를 데이터 버스를 통해 CPU로 전달
4. 보호 모드로 실행 중이라면 IDT에서 해당 번호에 맞는 디스크립터를 찾아 핸들러 실행



슬레이브 PIC에 연결된 장치 중 인터럽트 발생 시

1. 자신의 INT 핀에 신호를 실어 마스터 PIC의 IRQ 2번에 인터럽트 신호 전달
2. 마스터 PIC는 자신의 IRQ 핀에서 인터럽트가 발생 했으므로 CPU에게 전달
3. CPU가 /INTA 신호를 주면 데이터 버스에 숫자를 실어 인터럽트가 발생한 IRQ 번호 전달(숫자 8~15사이)



### ICW

마스터 PIC와 슬레이브 PIC가 제대로 동작하기 위해서는 초기화해줘야 함

자신이 마스터인지 슬레이브인지, 어떤 모드로 동작하는지 등등

ICW1, ICW2, ICW3, ICW4로 구성되어 있으며 프로그램은 순서대로 실행



#### ICW1

![icw1](https://user-images.githubusercontent.com/23413308/78849331-2cbfe500-7a4f-11ea-87b0-b0a04ae4badd.png)

PIC를 초기화 하는 명령어

LTIM : 인터럽트 발생 시 인터럽트 신호의 엣지에서 인터럽트를 인정할 것인지 High level로 신호가 모두 올라간 상태에서 인정할 것인지. 0 : 엣지 트리거링, 1 : 레벨 트리거링

SNGL : 마스터/슬레이브 형식(0), 마스터 하나만 사용(1)

IC4 : ICW4 명령어가 필요하지 않음(0), 필요함(1)



#### ICW2

![icw2](https://user-images.githubusercontent.com/23413308/78849473-8d4f2200-7a4f-11ea-88a4-a4d5b18a76fe.png)

PIC가 인터럽트를 받았을 때 IRQ 번호에 얼마를 더해 CPU에 전달할지 지정

0~2비트가 0인 이유 : 숫자를 8 단위로 기재

es) 2진수로 00010000(0x10)을 넣으면 인터럽트 0번 발생 시 0x10을 보냄 -> IRQ 16번

CPU에 설정된 exception 번호와 메인보드의 인터럽트 회로 구현에서 충돌이 일어나므로 번호를 바꾸어 줌



#### ICW3

![icw3](https://user-images.githubusercontent.com/23413308/78849756-575e6d80-7a50-11ea-9830-719760eb61f7.png)

PIC의 마스터, 슬레이브로서의 연결 방법

S0~S7 : 마스터 PIC의 각 IRQ 선

비트 0을 넣음 : IRQ 선이 하드웨어 장치에 연결

1을 넣음 : IRQ 선이 슬레이브 PIC에 연결

ID0~ID2의 3비트 : 마스터 PIC의 몇 번째 IRQ 핀에 연결되어 있는지 숫자로 표현



#### ICW4

![icw4](https://user-images.githubusercontent.com/23413308/78849835-8b399300-7a50-11ea-88f2-e18a14ab5f60.png)

추가 명령어, ICW1의 IC4가 세트되어 있으면 추가해야 함

SFNM, BUF, M/S : 우리가 사용하는 PC에서는 구현되지 않아도 무방하므로 항상 0

AEOI : PIC의 reset의 자동 여부, 자동 수행 시 CPU에게 알린 후 바로 리셋. 수동 시 인터럽트 처리 후 PIC에 명령을 주는 형식으로 초기화

UPM : 0(MCS-80/85 모드), 1(8086 모드)



```assembly
; ICW1
mov al, 0x11		; PIC 초기화
out 0x20, al		; Master PIC
dw 0x00eb, 0x00eb	; jmp $+2, jmp $+2
out 0xA0, al		; Slave PIC
dw 0x00eb, 0x00eb

; ICW2
mov al, 0x20		; Mster PIC 인터럽트 시작점
out 0x21, al
dw 0x00eb, 0x00eb
mov al, 0x28		; Slave PIC 인터럽트 시작점
out 0xA1, al
dw 0x00eb, 0x00eb

; ICW3
mov al, 0x04		; Master PIC IRQ 2번에
out 0x21, al		; Slave PIC 연결되어 있음
dw 0x00eb, 0x00eb
mov al, 0x02		; Slave PIC가 Master PIC의
out 0xA1, al		; IRQ 2번에 연결되어 있음
dw 0x00eb, 0x00eb

; ICW4
mov al, 0x01		; 8086 Mode
out 0x21, al
dw 0x00eb, 0x00eb
out 0xA1, al
dw 0x00eb, 0x00eb

; 모든 인터럽트 막음
mov al 0xFF
out 0xA1, al
dw 0x00eb, 0x00eb
mov al, 0xFB
out 0x21, al
```



### 예외(Exception)

프로그램이나 시스템에 문제를 일으킬 만한 명령을 실행함으로써 CPU가 문제 해결을 위해 발생시키는 인터럽트

```
인터럽트 0 - 0 나누기 예외(#DE)
인터럽트 1 - 디버그 예외(#DB)
인터럽트 2 - NMI 인터럽트
인터럽트 3 - 브레이크 포인트 예외(#BP)
인터럽트 4 - 오버플로 예외(#OF)
인터럽트 5 - BOUND 범위 초과 예외(#BR)
인터럽트 6 - 무효 COPDOE 예외(#UD)
인터럽트 7 - 디바이스 사용 불가 예외(#NM)
인터럽트 8 - 더블 폴트 예외(#DF)
인터럽트 9 - Coprocessor 세그먼트 오버런 에외
인터럽트 10 - 무효 TSS 예외(#TS)
인터럽트 11 - 세그먼트 부재 예외(#NP)
인터럽트 12 - 스택 폴트 예외(#SS)
인터럽트 13 - 일반 보호 예외(#GP)
인터럽트 14 - 페이지 폴트 예외(#PF)
인터럽트 16 - x87 FPU 부동 소수점 에러(#MF)
인터럽트 17 - 얼라인먼트 체크 예외(#AC)
인터럽트 18 - 머신 체크 예외(#MC)
인터럽트 19 - SIMD 부동 소수점 예외(#XF)
인터럽트 20~31 - 인텔사에서 예약
인터럽트 32~355 - 유저 정의 인터럽트
```