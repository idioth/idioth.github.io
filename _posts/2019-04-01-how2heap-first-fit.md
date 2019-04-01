---
layout: post
title: how2heap - first_fit.c 번역
tags: [system, heap]
---



기존 tistory 블로그에 있었던 게시글들을 다 가져오는 중입니다.. ㅜㅜ

---

[원문]

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main()
{
	printf("이 파일은 공격 방법에 대해서 설명하지 않지만 glibc의 메모리 할당이 본질적으로 어떻게 이루어지는지 보여준다.\n");
	printf("glibc는 free된 chunk를 선택할 때 first-fit 알고리즘을 사용한다.\n");
	printf("chunk가 free 되었고 크기가 충분하다면, malloc은 이 chunk 선택할 것이다.\n");
	printf("이것은 use-after-free 상황에서 exploit이 가능하게 해준다.\n");

	printf("2개의 버퍼를 할당한다. 그들은 충분히 커야하고, fastbin이 되어선 안된다.\n");
	char* a = malloc(512);
	char* b = malloc(256);
	char* c;

	printf("1st malloc(512): %p\n", a);
	printf("2nd malloc(256): %p\n", b);
	printf("우리는 이곳에 계속 malloc할 수 있다.\n");
	printf("우리가 나중에 읽을 수 있도록 \"This is A!\"라는 문자열을 넣어주자.\n");
	strcpy(a, "this is A!");
	printf("처음 할당된 %p는 %s을 가리킨다.\n", a, a);

	printf("첫 번째를 free한다...\n");
	free(a);

	printf("어떤 것도 다시 free할 필요가 없다. 512보다 작게 할당하기만 하면, %p에 할당될 것이다.\n", a);

	printf("500 bytes를 할당해보자.\n");
	c = malloc(500);
	printf("3rd malloc(500): %p\n", c);
	printf("그리고 이곳에 \"This is C!\"라는 다른 문자열을 넣어주자.\n");
	strcpy(c, "this is C!");
	printf("세 번째 할당된 %p는 %s를 가리킨다.\n", c, c);
	printf("첫 번째 할당된 %p는 %s를 가리킨다.\n", a, a);
	printf("우리가 첫 번째 할당된 것을 다시 사용하면, 세 번째 할당된 것의 데이터를 가진다.");
}
```



[원문]: <https://github.com/shellphish/how2heap/blob/master/first_fit.c>

