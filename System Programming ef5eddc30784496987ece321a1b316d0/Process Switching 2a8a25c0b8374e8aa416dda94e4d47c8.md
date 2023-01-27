# Process Switching

2023.01.15

---

## Introduction

프로세스에 이어서 커널이 process switching을 어떻게 수행하는지 내부적인 매커니즘과 커널에서 thread가 어떻게 구현되어 있는지 알아보자.

## Process Switching

앞에서 프로세스에 대하여 알아봤다. 근데 컴퓨터에서 프로세스가 하나만 돌아가는 것이 아니기에 다른 프로세스도 돌려야 하고, 그러면 ***Process Switching***을 해야한다. 커널에서 process switching이 어떻게 구현 되어있을까?

큰 틀에서 보면 **현재 CPU에서 실행중인 프로세스(*prev*)를 멈추고, 이전에 중단되었던 프로세스(*next*)를 다시 실행**시킨다. 다시 실행시키는 프로세스를 고르는 것은 ***process scheduling*** 이라고 해서 후에 살펴볼 것이다. Process Switching은 크게 4단계로 구성되는데

1. *prev*의 *User mode* 레지스터 값들을 *prev*의 커널 스택에 저장한다.
2. *prev*의 *Address space*에서 *next*의 ***Address space***로 전환한다.
3. *prev*의 커널 스택에서 *next*의 **커널 스택**으로 전환한다.
4. *prev*의 *Hardware Context*에서 *next*의 ***Hardware Context***로 전환한다.

여기서 ***Hardware Context*는 현재 프로세스가 사용하고 있던 레지스터의 값들**을 의미한다. 여기에는 프로세스가 실행되는데 필요한 모든 정보들이 담겨 있으며, 모든 프로세스가 **같은 레지스터를 사용**하기에 이 값들을 저장하고 다음 프로세스의 레지스터 값들을 불러와주어야 한다. 이 정보들 중 일부는 PCB에 `thread_struct` 라는 자료구조에, 나머지는 커널 스택에 저장된다.

![Untitled](Process%20Switching%202a8a25c0b8374e8aa416dda94e4d47c8/Untitled.png)

`schedule()` 함수가 실행될 때마다 Process Switching이 발생하며 위의 과정들은 `switch_to` 매크로를 통해 수행된다. 그러면 `switch_to` 매크로의 수행 과정에 대해 자세히 살펴보자.

## Switch_to

![Untitled](Process%20Switching%202a8a25c0b8374e8aa416dda94e4d47c8/Untitled%201.png)

![Untitled](Process%20Switching%202a8a25c0b8374e8aa416dda94e4d47c8/Untitled%202.png)

왼쪽은 리눅스 2.6.11 버전의 switch_to 코드이고 오른쪽은 리눅스 6.0 버전의 switch_to 코드이다. switch_to 코드는 어셈블리로 작성이 되어 있고, 버전을 거듭하면서 리눅스의 코드도 많이 복잡해졌다. 하지만 전체 맥락은 바뀌지 않기 때문에 왼쪽 코드를 기준으로 설명을 하겠다.

왼쪽 코드도 어려워 보이지만 찬찬히 읽어보면 다음의 일을 하는 코드라고 볼 수 있다.

![Untitled](Process%20Switching%202a8a25c0b8374e8aa416dda94e4d47c8/Untitled%203.png)

이를 그림으로 표현하면 다음과 같다.

![Untitled](Process%20Switching%202a8a25c0b8374e8aa416dda94e4d47c8/Untitled%204.png)

1. **기존의 *ebp*와 *flag*를 *prev*의 커널 스택에 저장**
2. ***prev*의 *esp*를 *prev*의 *thread*에 저장**
3. ***next*의 *thread*에 있는 *esp*를 레지스터로 가져옴**
4. L1 라벨 주소를 ***prev*의 *thread의* *eip*에 저장**
5. ***next*의 *thread*의 *eip*(L1 라벨 주소)를 스택의 top에 올림**
6. `__switch_to`로 점프하여 나머지 레지스터들 값도 전환
7. `__switch_to`에서 eip 값도 바꾸기 때문에 ***return*과 동시에 *PC*가 바뀌고**, return address를 항상 스택의 top에서 찾기 때문에 다음 eip는 이전에 불러온 L1
8. 스택에서 ***ebp*와 *flag*를 꺼내면서 *Hardware Context Switching* 종료**

이후 다시 User mode로 돌아가며 그 return address는 커널 스택의 맨 위에 존재하게 된다.