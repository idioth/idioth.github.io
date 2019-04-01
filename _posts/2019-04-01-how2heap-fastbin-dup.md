---
layout: post
title: how2heap - fastbin_dup.c 번역
tags: [system, heap]
---

[원문]



```c
#include <stdio.h>
#include <stdlib.h>

int main()
{
	printf("이 파일은 fastbins에서 간단한 double-free 공격을 설명한다.\n");

	printf("3개의 버퍼를 할당한다.\n");
	int *a = malloc(8);
	int *b = malloc(8);
	int *c = malloc(8);

	printf("1st malloc(8): %p\n", a);
	printf("2nd malloc(8): %p\n", b);
	printf("3rd malloc(8): %p\n", c);

	printf("첫 번째 것을 free한다.\n");
	free(a);

	printf("%p를 다시 free한다면, %p는 free list의 꼭대기에 있기 때문에 crash가 날 것이다.\n", a, a);
	// free(a);

	printf("그래서 대신 우리는 %p를 free할 것이다.\n", b);
	free(b);

	printf("이제 %p가 free list의 head에 있지 않기 때문에 %p를 다시 프리할 수 있다.\n", a, a); // 원문에서는 a 하나입니다.
	free(a);

	printf("현재 free list는 [ %p, %p, %p ] 이다. 우리가 malloc을 3번 하면 %p를 2번 얻을 수 있다.\n", a, b, a, a);
	printf("1st malloc(8): %p\n", malloc(8));
	printf("2nd malloc(8): %p\n", malloc(8));
	printf("3rd malloc(8): %p\n", malloc(8));
}
```



[원문]: <https://github.com/shellphish/how2heap/blob/master/fastbin_dup.c>