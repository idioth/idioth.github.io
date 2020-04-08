---
layout: post
title: 만들면서 배우는 OS 커널의 구조와 원리 ch2. 커널을 로드한다
tags: [kernel, os]
---



<b>부트로더</b>

- PC가 처음 부팅되었을 때 로드 되고 실행
- 디스크 상의 커널 프로그램을 램으로 읽어들여 커널이 실행되도록 하는 명령 수행



<b>kernel.bin 복사 루틴</b>

```asm
read:
	mov ax, 0x1000
	mov es, ax
	mov bx, 0
	
	mov ah, 2
	mov al, 1
	mov ch, 0
	mov cl, 2
	mov dh, 0
	mov dl, 0
	int 0x 13
	
	jc read
```

PC 부팅 시 BIOS가 MBR을 읽어 0x7C00에 복사하고, 프로그램을 이곳으로 점프 시킴

read: 이후의 부분을 실행시켜 바로 뒤 섹터에 kernel.bin을 0x10000번지에 복사

int 0x13의 BIOS 콜을 사용(소프트웨어 인터럽트를 걸어 잠시 BIOS에 있는 프로그램 실행)



사용한 BIOS 콜은 "어느 섹터부터 몇 개의 섹터를 읽어라"

- 복사 목적지의 주소 값을 es:bx 형태로 주어야 함

- ah: 서비스 번호 (BIOS 콜의 번호)

- al: 몇 섹터를 읽을 것인지

- ch: 사용할 실린더 값

- cl: 몇 번째 섹터부터 읽을 것인지

- dh: 헤드 값

- dl: 드라이브 번호

  

![pic2](https://user-images.githubusercontent.com/23413308/78516573-fc7d0a00-77f4-11ea-997d-3d052e173802.png)
