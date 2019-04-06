---
layout: post
title: c++ cin 입력 오류 처리 방법
tags: [c++, programming]
---



친구 학교 c++ 과제 중 vector와 try catch를 통해 4 글자 숫자 야구 프로그램을 만드는 과제를 도와주게 되었습니다.

exception은 숫자들이 동일할 때, 숫자가 범위를 벗어날 때, 그리고 input이 숫자가 아닌 문자일 때 세 가지 였는데

input이 문자일 때로 exception이 넘어가면 계속 무한 루프에 빠져서 도대체 뭐가 문제인지 하던 중..

잘 보니 vector의 type은 int로 되어있었고, throw로 넘어가는게 아니라 cin에서 에러가 일어나는거 였더군요..

int로 선언된 곳에 문자열이 들어가니 cin fail이 되면서 일어나는 오류였습니다.

while(ture)가 들어간 반복문이여서 무한 루프에 들어간거였죠.



해결 방법

1. cin 입력이 정상적으로 처리되었는지 판단, cin.fail() 혹은 !cin (cin이 실패했느냐)
2. cin.clear() 상태 초기화
3. 입력버퍼 초기화, cin.ignore 혹은 fseek(stdin, 0, SEEK_END) 사용



```c++
if(cin.fail())
    throw 'a';

catch(char e){
    cerr << "Error: Not a number" << endl;
    cin.clear();
    fseek(stdin, 0, SEEK_END);
}

```

표준 stream 관련 함수들은 주어진 format과 일치 않는 데이터를 에러로 처리하고 data는 그대로 보존되기 때문에 나타나는 오류라고 합니다.