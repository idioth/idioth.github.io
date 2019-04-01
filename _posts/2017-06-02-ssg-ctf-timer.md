---
layout: post
title: SSG CTF 2017 Timer Write Up
tags: [reversing, writeup]
---

이번에 5월 27일날 열린 SSG CTF에서 문제를 출제하게 되었습니다. 처음 내보는 문제이기도 하고 컨셉을 무엇을 낼까 고민도 많이 했습니다. 아침에 학교갈 준비하면서 머리 말리던 중 눈에 들어온 타이머가 있었는데 그걸 보고 고장난 타이머를 고치는 문제를 내야겠다!!라는 의식의 흐름대로 나온 문제입니다.



고장난 타이머를 고친다.. 무엇인가를 고친다.. = 코드 패치! 해서 아주 단순하게 코드 패치 문제를 만들었는데 생각보다 문제가 난해했는지 많은 분들이 못푸셨더라고요..(사실 solver가 0입니다.. 주륵)



문제에 대해 간단히 설명하자면 32비트 Windows PE 파일이고 리버싱 200점짜리 문제로 만들어졌습니다. 제 기억이 맞다면 2시인가 3시쯤에 문제가 오픈되었는데 5~6시까지 solver가 나타나지 않아서 곧 푸시겠지~라고 생각했으나 8시에는 많은 생각이 들었었죠.. 쥬륵



말재주가 없어서 문제 설명도 난해했고 라이트업도 약간 횡설수설하는 모습을 보여줄수도 있습니다. 처음 내보는 문제라서 좀 더 엉성했던거 같기도 하고.. 아무튼 Reversing200 - Timer 라이트업 시작하겠습니다!!

---

![1](https://user-images.githubusercontent.com/23413308/55320440-12554200-54b2-11e9-8b34-e15108d9b257.jpg)

리버싱은 200, 250, 350점의 문제들이 출제되었는데 이 라이트업은 타이머에 대해서 다룰겁니다!!(짝짝)



![2](https://user-images.githubusercontent.com/23413308/55320473-29942f80-54b2-11e9-902c-ff8029113db9.jpg)

(눈물이 앞을 가리는 0 solved..)

많은 분들이 Fix Me 라고 적어두면 코드 패치라고 생각하실 줄 알았는데 좀 더 어렵게 생각하셔서 못 푸셨던 거라고 생각합니다.



![3](https://user-images.githubusercontent.com/23413308/55320477-2d27b680-54b2-11e9-88a4-18f9423cedb7.jpg)

파일을 실행시키면 Timer의 모드를 입력하라고 나옵니다. 이것 저것 입력을 하면 타이머에 기능이 없다면서 프로그램이 바로 종료되어 버립니다. 고장이 났으니 이해해주며 ida로 열어봅니다.



![4](https://user-images.githubusercontent.com/23413308/55320478-2dc04d00-54b2-11e9-8d8b-715b13e5361e.jpg)

많은 함수들이 뜨지만 걱정할 필요 없습니다. Input mode라는 문자열을 추적하면 쉽게 main을 찾을 수 있으니까요!!



![5](https://user-images.githubusercontent.com/23413308/55320479-2dc04d00-54b2-11e9-8fbc-eb442e75a954.jpg)

Input the mode 문자열을 찾았는데 의문의 Timer fix mode가 있습니다. Input the Timer mode에서 어떠한 모드를 입력하면 fix mode로 진입할 수 있으리라 믿으며 fix mode로 진입해서 어떠한 함수가 호출을 하는지 봅니다.



![6](https://user-images.githubusercontent.com/23413308/55320480-2dc04d00-54b2-11e9-80b8-3e10b0bf1da9.jpg)

sub_129D0로부터 호출이 되는데 나의 친구 우리의 친구 hex-rays로 보기위해 진입해보면



![7](https://user-images.githubusercontent.com/23413308/55320481-2e58e380-54b2-11e9-99e7-33c5f72a6993.jpg)

허걱스 되질 않네;; 어쩔수 없죠 어셈으로 보는 수 밖에요..



![8](https://user-images.githubusercontent.com/23413308/55320482-2e58e380-54b2-11e9-9e75-be25b3d55ead.jpg)

어떠한 함수인지 보았더니 우리가 아까 보았던 main 함수겠군요!! var_8로 mode를 입력하고 0xA와 비교하는데 A가 입력됐을 때 어떠한 동작을 하는지 봅니다.



![9](https://user-images.githubusercontent.com/23413308/55320484-2e58e380-54b2-11e9-983e-9c0a959841e8.jpg)

ㄹㅇ.. 뭔가 특이점이 온거같은데 많은 함수들이 값을 비교하고 점프하고 나중에는 flag로 추정되는 것을 호출해주고 하는데 이 과정에서는 fix mode로 들어가지 않습니다. 괜히 fix mode가 있지는 않겠죠? 이 녀석은 함정입니다. angr 문제라고 생각하시라고 사실 적어본건데 값 입력도 말이 안되고 저 이상한 함수들은 return만 해주는 함수들이여서... 사실 헤메라고 만든 부분인데 이 부분이 좀 더러웠을수도 있겠다고 생각했습니다. 아무튼 이건 아니고 아까 fix mode를 호출한 부분을 봐봅시다.



![10](https://user-images.githubusercontent.com/23413308/55320485-2ef17a00-54b2-11e9-9f0c-c88e5c30486b.jpg)

fixmode는 var_8이 0x41(65)와 비교해서 참일 때 호출이 되는군요! 바이너리를 실행해서 확인해봅시다.



![11](https://user-images.githubusercontent.com/23413308/55320486-2ef17a00-54b2-11e9-9a56-374ea67ae2df.jpg)

fix mode에 진입이 되었습니다. fix mode에서 무엇을 해야하는지 함수를 들여다봅시다.



![12](https://user-images.githubusercontent.com/23413308/55320487-2ef17a00-54b2-11e9-89fb-4138b086aad1.jpg)

전체적으로 훑어보면 sub_11710의 값에 따라 수리가 성공하는 건지 실패하는 건지 뜹니다. sub_11710의 return 값이 2017일 때 수리에 대한 판단을 하는데, if문에 있는 함수들이 모두 or 연산을 합니다. 저 if문 안으로 들어가면 수리 실패라는 메시지 박스가 뜨니까 repairComplete로 들어가려면 저기에 해당되는 모든 함수들의 ret 값이 0이 되어야 합니다.



![13](https://user-images.githubusercontent.com/23413308/55320488-2f8a1080-54b2-11e9-91ee-f84d1da5e3f2.jpg)

0이 되어야하는 함수들의 이름들을 zero%d로 해주고 sub_11710으로 갑니다.



![14](https://user-images.githubusercontent.com/23413308/55320489-2f8a1080-54b2-11e9-98c8-06614ecd946d.jpg)

2017을 return 해주기 위해서는 *v7과 *v8의 xor 연산 값이 17이 나와야 하는군요. 각각이 무슨 값인지 한 번 확인해봅시다.



![15](https://user-images.githubusercontent.com/23413308/55320490-2f8a1080-54b2-11e9-9ebf-089c083481b4.jpg)

v7은 sub_116D0의 return 값입니다.



![16](https://user-images.githubusercontent.com/23413308/55320492-2f8a1080-54b2-11e9-90e3-5e0946ecfc19.jpg)

v8은 0으로 초기화된 후 수행이 되면서 if문에 걸릴 때 마다 if문 안에서 호출한 함수들의 return 값을 더해줍니다. 그런데 main 함수에서 0이 되어야할 함수들이 모두 if문에 들어가있기 때문에 if문에 걸리는 부분들은 모두 수행되지 않고 v8은 0의 값을 가지게 될 것입니다. 0 ^ 17 = 17이므로 v7의 값은 17이 되어야합니다.



![17](https://user-images.githubusercontent.com/23413308/55320493-3022a700-54b2-11e9-96b9-4ea454441c49.jpg)

v7의 값이 되는 함수의 내부인데, *v0 + *v1의 값이 17이면 됩니다. 저는 올리로 볼 때 위에 있는 것을 17(0x11)로 바꿀 것입니다.



이제 0이 되어야하는 함수들을 차근차근 봅시다.

![18](https://user-images.githubusercontent.com/23413308/55320494-3022a700-54b2-11e9-8152-74ba1d385919.jpg)

zero1은 hex-rays가 열리지 않네요. 무엇이 문제인가 어셈을 보아야하는가.. 사실 디컴파일이 실패한 이유는 sp value의 값이 이상하기 때문입니다. zero1의 어셈을 보면



![19](https://user-images.githubusercontent.com/23413308/55320496-3022a700-54b2-11e9-9131-1caa28bad307.jpg)

중간에 이상한 popa가 있는데, 어차피 수행도 안되는거 왜 있지? 저녀석 때문에 스택 포인터에 문제가 생기지 않았을까?하며 확인해보니



![20](https://user-images.githubusercontent.com/23413308/55320497-30bb3d80-54b2-11e9-8eda-35cd9fcefcdb.jpg)

녀석을 기점으로 스택 포인터가 음수의 값을 가집니다. 스택 포인터를 정상적으로 만들어주면 정상적으로 hex-rays가 될 거라고 믿으며 SP Value를 수정해줍니다.



![21](https://user-images.githubusercontent.com/23413308/55320499-30bb3d80-54b2-11e9-908e-96a6b30432c6.jpg)

바뀌어라 얍얍 F5여 제발 돼라 얍얍



![22](https://user-images.githubusercontent.com/23413308/55320500-30bb3d80-54b2-11e9-94f8-f15d5832d0f3.jpg)

굿굿!! 이제 분석을 시작해봅시다. *v4를 리턴하니까 v4를 중점으로 보면 됩니다. v4에 대한 연산은 v2+4와 여러 연산을 하는 부분이 중점이므로 v2와 v2+4의 값을 0으로 만들어 주면 됩니다.



![23](https://user-images.githubusercontent.com/23413308/55320501-30bb3d80-54b2-11e9-9c0c-b3154da501cb.jpg)

zeor2에서 v4는 return 위의 연산에서 자신의 값을 갖고 return에서 계산이 됩니다. return의 연산식에서는 나눗셈, 곱셈, 나머지 연산 이것은 모두 값이 0이면 0을 반환하게 되니 모든 것을 0으로 만들어줍시다.



![24](https://user-images.githubusercontent.com/23413308/55320502-3153d400-54b2-11e9-8bac-3cda6bdb640b.jpg)

zero3.. 잘 들여다보면 v3, v3+4, v3+8(구조체겠죠?)의 값들만 return 값에 영향을 줍니다. 2, 3, 5를 0으로 만들어주면 0이 반환될겁니다.



![25](https://user-images.githubusercontent.com/23413308/55320504-3153d400-54b2-11e9-9dcf-09f36e2c3296.jpg)

v4에 대한 연산을 열심히 하지만 return을 보면 v2와 v2+4만 계산이 됩니다. 얘는 연산이 좀 이상하네요. 함수가 return 할 때는 eax의 값을 반환하니까 eax가 존재한다면 그것을 0으로 바꾸어주면 될 것 같습니다.



![26](https://user-images.githubusercontent.com/23413308/55320505-3153d400-54b2-11e9-9564-c0f51a29e266.jpg)

[edx]와 0ah를 0으로 해줍시당. 이제 마지막 남은 zero5 함수를 봅시다..



![27](https://user-images.githubusercontent.com/23413308/55320506-31ec6a80-54b2-11e9-9f1e-2eb5b5cffb39.jpg)

v1의 제곱 - 11 * v1은 v1이 11이면 깔끔하게 맞아 떨어지겠죠? v1=10(0xA)인 것을 0xB(11)로 수정해주면 됩니다.



수정해야할 함수들의 offset과 수정해야할 값을 계산해서 정리해봅시다.

zero1: 0x11330 / 10, 20 ->0

zero2: 0x11440 / 모든 값 0

zero3: 0x11260 / 2,3,5 -> 0

zero4: 0x11560 / eax -> 0

zero5: 0x11620 / 10->11

v7: 0x116D0 / 912 -> 17, 1203 -> 0



올리로 열어서 해당하는 곳들을 모두 수정해줍시다.

![28](https://user-images.githubusercontent.com/23413308/55320507-31ec6a80-54b2-11e9-8751-6564dd772a26.jpg)

![29](https://user-images.githubusercontent.com/23413308/55320508-31ec6a80-54b2-11e9-9e0a-3910f8123816.jpg)

패치가 완료되어 정상적으로 2017을 리턴하게 되었고 모든 조건을 맞추었습니다. 실행을 하고 타이머에 시간 초를 입력하는 것은 0초를 입력해주시면 빠르게 넘어갈 수 있습니다.



![30](https://user-images.githubusercontent.com/23413308/55320509-31ec6a80-54b2-11e9-8a4e-15548e2c91ee.jpg)

flag : 9ood_3ngin33r_thx!



---

옮기면서 보는데 2년 전에 처음 시작할 때 만든 문제라 그런지 간단한걸 엄청 더럽게 만들어놨었네요 ㅋㅋㅋ