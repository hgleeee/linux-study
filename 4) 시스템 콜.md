# 시스템 콜 (System Call)

<p align="center"><img src="./images/system_call.png" width="600"></p>
> 리눅스 명령어는 옆에 붙은 숫자에 따라 커맨드(1), 시스템 콜(2), 라이브러리 함수(3)로 구분된다.

- 위 그림의 좌측을 보면 유저 영역 안에 유저가 작성한 코드 my code가 있다. 이 코드에서 printf()를 호출(call) 하는데 이 printf() 코드는 내가 작성한 게 아니라 library function이다. C언어를 배울 때 #include <stdio.h>를 하는 이유를 생각해보면 금방 이해할 것이다. 
- 그럼 이제 printf()가 내가 작성한 코드 my code에 들어오는데 printf()는 출력 즉, I/O를 해 줘야 한다. 멀티 유저 시스템에서 I/O는 오직 커널만 할 수 있기 때문에 I/O를 하는 모든 library function은 무조건 System Call을 사용해야 한다. 커널에게 부탁한다는 뜻이다.
- 시스템 콜을 하게 되면 Wrapper Routine이라는 공간에 가게 되고 이 공간에는 왜 커널로 가게 되는지 알려주는 정보들을 담고 있는 Prepare parameter와 CPU의 모드 비트를 커널로 바꾸는 chmodk가 들어있다.
- chmodk가 실행되면서 프로그램은 런타임 중 트랩에 걸려 커널 영역으로 가게 된다. 커널에서는 Prepare parameter에 담겨있는 내용을 보고 적절한 System call function으로 처리를 해준다.

- 커널 안에 있는 모든 System call function의 이름은 sys_로 시작한다. 리눅스의 명명 규칙이다.

## 1. Wrapper Routine
> 트랩으로 넘어갈 내용들을 준비하고 실질적으로 트랩을 일으키는 공간

<p align="center"><img src="./images/wrapper_routine.png" width="600"></p>

- Wrapper Routine에서 (인텔의 경우) $0x80등 의미 없는 문자들을 이용해 Machine Instruction을 주어 트랩을 발동한다. 
- 그런데 위에서 트랩을 일으키기 전에 Prepare parameter들을 준비하게 되는데 그 중에 가장 중요한 것은 바로 system call number라는 것이다. 
- 이 system call number는 커널이 가지고 있는 system call function의 시작 주소를 담고있는 Array(배열)의 Index 번호로 사용이 된다.

### System call number의 예
- file과 관련된 system call에는 open, close, read, write등이 있다.
- open은 0번, close는 2번, read는 3, write는 4번 등 call number을 이용해 Array의 Index 위치에 접근을 한다.

### 과정
1. 컴파일러(gcc)가 유저가 짠 코드를 보고 라이브러리(printf())를 호출한다.
2. 라이브러리에서 시스템 콜(write)을 호출한다. 위 그림의 write(2)의 2는 시스템 콜을 의미하는 숫자일 뿐 매개변수와 같은 의미는 없다.
3. Wrapper Routine에서 write에 대응하는 system call number가 나오고 트랩을 건다.
4. 커널이 system call number을 가지고 system call function table에 접근해 function의 시작 주소에 접근한다.

## 2. 시스템 콜의 상세 과정
<p align="center"><img src="./images/system_call_process.png" width="600"></p>

1. 유저 프로그램이 시스템 콜을 호출한다.
2. Machine Instruction이 트랩을 발동한다.
3. 하드웨어가 유저 모드에서 커널 모드로 mode bit를 바꾼다.
4. 하드웨어가 sys_call()이라는 커널안의 트랩 핸들러(Trap Handler)로 가게 된다.
5. 이런 핸들러는 커널안의 assembly function을 수행한다.
6. 지금까지 유저 프로그램에서 진행했던 단계를 저장을 한다. (커널 쪽 일이 다 끝나면 시스템 콜을 호출 했던 곳으로 돌아가서 다시 진행을 해야하기 때문에 저장하는 것이다.)
7. 시스템 콜 번호가 커널 안에 sys_call table에 있는 번호에 맞는 번호인지 확인한다.
8. 맞다면 system call function의 주소를 가져온다.
9. 그리고 system call function을 불러 작업한다. (만약 진행 과정 중 디버깅이 필요하다면 디버거를 실행시킨다.)
10. 다시 시스템 콜 호출했던 유저의 영역으로 돌아가고 mode bit를 유저 모드로 전환한다.

## Kernel System Call Function

<p align="center"><img src="./images/kernel_system_call_function.png" width="600"></p>

### 개요
- 스마트폰으로 찍은 사진을 볼 수 있는 갤러리 애플리케이션을 만들었다고 생각해보자. 해당 애플리케이션은 사용자가 자신이 촬영하여 폰에 저장한 사진을 볼 수 있게끔 해준다.
- 애플리케이션을 제작할 때 소스코드에는 분명 스마트폰에 저장된 사진을 읽어오는 기능이 있을 것이다. 
- 이 기능은 library 함수를 사용하여 구현했을 것이고, 실제 동작할 때 library는 I/O를 하기위해 System Call을 호출할 것이다. 커널에게 부탁한다는 매커니즘이 시스템 콜이라는 점을 인지하자.

### 설명
- 커널에서는 유저가 원하는 사진 파일을 시스템 콜을 호출한 유저 영역으로 넘겨줘야 할 것이다. 때로는 커널이 유저 영역으로부터 데이터를 가져와야 하는 경우도 있을 것이다. 
- 즉, 어플리케이션이 제대로 동작하기 위해서는 유저 프로그램과 커널 프로그램이라는 서로 독립된 프로그램 사이에 데이터를 주고 받을 수 있는 수단이 반드시 필요하다.
- 그러한 기능들은 오직 커널만이 가지고 있다. 리눅스는 멀티 유저 시스템이고 시스템의 보안을 위해서 오직 커널만이 모든 메모리에 접근이 가능하다. 
- 좀 더 자세히 살펴보면, 커널이 유저에게 데이터를 보내줄 수는 있어도 유저가 커널로부터 데이터를 읽어올 수는 없고 커널이 유저한테서 데이터를 읽어올 수는 있어도 유저가 커널한테 데이터를 보낼 수는 없다.


## System Call Number
<p align="center"><img src="./images/system_call_number.png" width="600"></p>

### 정의
- System call number는 커널의 system call table의 인덱스 번호로 사용되어 system call function의 주소의 시작값을 불러오는 용도로 사용된다. 
- System call number는 컴파일러와 OS를 제작한 회사에서 정하며 이렇게 정해진 번호는 변경 할 수 없다.

### 시스템 콜 만들기
- 그렇다면 리눅스에 자신만의 시스템 콜(System Call)을 만들 수는 없을까? sys_write()나 sys_read()처럼 특정 기능을 수행하는 시스템 콜을 정의하고 사용할 순 없을까? 물론 직접 만들 수 있다!

<p align="center"><img src="./images/write_system_call_1.png" width="600"></p>

#### 장점
- 우리는 새로운 시스템 콜을 만들 때 우리가 원하는 특정 기능만을 위한 코드를 작성할 수 있다. 즉, 기존에 존재하는 시스템 콜보다 간단하고 성능 또한 좋게 만들 수 있다.
- 예를 들어, 컴퓨터 화면에 특정 알파벳만을 출력하는 기능을 새로 정의할 수 있고 이는 알파벳 뿐만 아니라 숫자, 기호 등을 출력해줄 수 있는 기존의 printf() 함수보다 훨씬 코드도 간결하고 효율적일 것이다.

#### 단점 - 1
- 새로운 시스템 콜을 만들게 되면, 그 시스템 콜만의 새로운 system call number가 필요하게 된다.
- 이렇게 새로 제작할 때마다 system call number를 정의하게 되면 새로 만든 시스템 콜은 그것을 제작한 플랫폼에서만 사용할 수 있다.
- 즉, 다른 플랫폼에서 본인이 만든 시스템 콜(예를 들어 99번)을 호출하는 것은 불가능하다. 다른 플랫폼에는 99번에 해당하는 시스템 콜이 존재하지 않거나 다른 시스템 콜일 수 있기 때문이다. 
- __플랫폼 의존적이라는 치명적인 단점 때문에 보통 시스템 콜을 직접 만들어서 사용하는 일은 거의 없다.__

#### 단점 - 2
- 한번 만든 시스템 콜은 추가만 가능하고 변경은 불가능하기 때문에 나중에 수정을 하는 것도 불가능하다.

#### 단점 - 2 해결법
<p align="center"><img src="./images/write_system_call_2.png" width="600"></p>

- 기존에 있던 시스템 콜인 read나 write에 있는 파일 디스크립터(File Descriptor)을 활용하는 것이 해결법이다.
- 파일 디스크립터는 간단히 설명하자면 운영체제가 만든 파일이나 소켓을 편하게 부르기 위해서 부여한 숫자이다.
- 파일 디스크립터는 보통 적은 숫자만이 활용이 되고 있어 보통은 잘 쓰지 않는 999번 등에 본인의 파일 디스크립터를 지정하고 사용하면 커널 안에 내장된 시스템 콜에 영향을 주지 않고도 사용할 수 있다. 훨씬 안전한 방법이다.










