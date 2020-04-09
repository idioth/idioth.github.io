---
loayout: post
title: 만들면서 배우는 OS 커널의 구조와 원리 ch5. Task Switching
tags: [kernel, os]
---

### TSS(Task State Segment)

- 선점형 방식 : 프로그램이 어떤 상황이던 일단 정지시키고 다른 프로그램이 이전에 실행했던 곳부터 다시 실행되도록 하는 방식
- CPU에서 수행되던 프로그램의 레지스터 값들을 모두 램에 저장 한 후, 다른 프로그램의 레지스터 값을 CPU에 불러와서 다시 실행되도록 해야 함
- 태스크 스위칭을 구현하기 위해서 RAM 상에 레지스터 값들을 보존해야 할 영역을 만들어 놓아야 함

![tss](https://user-images.githubusercontent.com/23413308/78859047-22aae000-7a69-11ea-8dd3-512e5d02f94b.png)

GDTR, IDTR, CR0~CR2 등 모든 태스크가 공통으로 사용하는 레지스터를 제외한 모든 레지스터 포함

```assembly
; 그림은 RAM에서의 주소가 아래에서부터 위로 증가하고 소스는 웨이서부터 아래로 증가
tss1:
	dw 0, 0			; 이전 태스크로의 back link
	dd 0			; ESP0
	dw 0, 0			; SS0, 사용 안함
	dd 0			; ESP1
	dw 0, 0			; SS1, 사용 안함
	dd 0			; ESP2
	dw 0, 0			; SS2, 사용 안함
	dd 0, 0, 0		; CR3, EIP, EFLAGS
	dd 0, 0, 0, 0	; EAX, ECX, EDX, EBX
	dd 0, 0, 0, 0	; ESP, EBP, ESI, EDI
	dw 0, 0			; ES, 사용 안함
	dw 0, 0			; CS, 사용 안함
	dw 0, 0			; SS, 사용 안함
	dw 0, 0			; DS, 사용 안함
	dw 0, 0			; FS, 사용 안함
	dw 0, 0			; GS, 사용 안함
	dw 0, 0			; LDT, 사용 안함
	dw 0, 0			; 디버그용 T 비트, I/O 허가 비트맵
```

이전 태스크로의 링크

- 이전에 실행되던 프로그램의 TSS 영역의 세그먼트 셀렉터 값
- TSS 영역은 GDT에 있는 TSS 세그먼트 디스크립터와 한 쌍을 이룸
- 이 디스크립터를 셀렉터로 지정하여 JMP, CALL 명령으로 태스크 스위칭을 함
- CALL 명령으로 스위칭을 할 시 TSS 영역에 이전 태스크의 TSS 디스크립터의 셀렉터 값을 저장해둠
- IRET 명령으로 프로그램을 마치고 CPU는 이전 태스크로의 back link를 통해 이전 태스크로 스위칭



ESP0, SS0

- 유저 모드(레벨 3) 태스크가 실행 중 커널 모드(레벨 0)로 스위칭을 하면 스택 값이 바뀌어야 함
- 따라서 TSS 영역에 시스템 레벨별로 스택이 따로 존재



CR3 : 페이징 구현과 관련

디버그용 T 비트 : 디버깅하고 있는 태스크가 스위칭 되기 전에 디버깅 중이였다는 표시

I/O 허가 비트맵 : 유저 레벨 태스크는 주변 장치를 맘대로 사용 불가. 사용 가능한 I/O 장치 표시



![tssdesc](https://user-images.githubusercontent.com/23413308/78859790-67377b00-7a6b-11ea-8fe7-4d105f185a56.png)

```assembly
descriptor:
	dw 104
	dw 0
	db 0
	db 0x89
	db 0
	db 0
```

TSS 디스크립터에서는 Limit 값이 항상 0x67 이상의 값을 가져야 함

-> 그렇지 않으면 무효 TSS 예외(#TS)가 발생

Type 필드 B 비트 : 태스크가 실행 중이거나 실행을 기다리고 있는 중. 처음에는 0으로 클리어

태스크 스위칭은 항상 커널 레벨에서 이루어지는 것이 일반적이므로 DPL은 00

Base Address : TSS 영역의 시작 주소



태스크 스위칭의 흐름

```assembly
mov ax, TSS1Selector
ltr ax					; CPU에 있는 TR 레지스터에 TSS 디스크립터 셀렉터 값을 넣음

lea eax, [process2]
mov [tss2_eip], eax
mov [tss2_esp], esp

jmp TSS2Selector:0			; 다시 태스크 스위칭이 되면 이곳으로 돌아옴, 태스크 스위칭 되는 곳

mov edi, 80*2*9
lea esi, [msg_process1]
call printf
jmp $

process2:
	mov edi, 80*2*7			; 실행된다면 태스크 스위칭 된 상태
	lea esi, [msg_process2]
	call printf
	jmp TSS1Selector:0

msg_process1 db "This is System process 1", 0
msg_process2 db "This is System Process 2", 0

tss2_eip:
	dd 0, 0					; EIP, EFLAGS
	dd 0, 0, 0, 0
	
tss2_esp:
	dd 0, 0, 0, 0
	dw SysDataSelector, 0
	dw SysCodeSelector, 0
	dw SysDataSelector, 0
	dw SysDataSelector, 0
	dw SysDataSelector, 0
	dw SysDataSelector, 0
	dw 0, 0
	dw 0, 0
```

1. LTR 명령으로 TSS 영역 지정 후 jmp TSS2Selector:0
2. CPU 내부 TR 레지스터를 참조하고 GDTR 레지스터를 참조하여 TSS1Selector 디스크립터 찾음
3. TSS1Selector 디스크립터의 Base Address를 통해 tss1 영역을 찾음
4. tss1에 모든 레지스터 값 저장
5. GDT에서 TSS2Selector 디스크립터를 찾음(이때 TR 레지스터 : TSS2Selector)
6. TSS2Selector 디스크립터의 Base Address를 통해 tss2 영역을 찾음
7. 모든 레지스터 값 CPU에 복원
8. 복원되었을 때 EIP 레지스터에 있는 주소부터 실행(process2)



### CALL 명령에 의한 Task Switching

TSS 디스크립터에 대하여 셀렉터를 사용한 CALL 명령에서 태스크 스위칭이 일어나고 IRET 명령을 통해서도 태스크 스위칭이 일어남



![eflag](https://user-images.githubusercontent.com/23413308/78860698-cbf3d500-7a6d-11ea-83ff-0d615991c26a.png)

NT 비트 : 인터럽트 핸들러의 IRET인지 태스크 스위칭되어 이전의 태스크로 돌아가는 IRET인지 구별할 때 사용



ex) 실행 중인 태스크 X, 스위칭할 태스크 Y

1. LTR명령은 해당 TSS 디스크립터의 B 비트를 1로 세트 -> CPU는 현재 태스크의 B 비트는 항상 1이라고 인식
2. CALL 명령에 의한 태스크 스위칭이 일어날 때 태스크 Y의 NT 비트, B 비트 1로 세트
3. 이전 태스크로의 back link에  태스크X의 TSS 디스크립터 셀렉터 저장
4. 태스크 X의 B 비트는 1인 상태로 남음
5. IRET을 사용하면 이전 태스크(태스크 X)로 스위칭
6. IRET 명령이 실행되기 위해서는 태스크 Y의 NT 비트가 반드시 1 이어야 함(인터럽트와 구분 위함)
7. 태스크 X의 B 비트도 1이어야 함
8. 태스크 스위칭 될 때 태스크 Y의 NT 비트와 B 비트는 0 으로 Clear