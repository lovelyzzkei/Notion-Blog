# Process

2023.01.13

---

## Introduction

OS에서 얘기했던 프로세스가 실제로 리눅스 커널에서 어떻게 구현이 되어 있는지 알아보자.

## Review

간단하게 OS 시간 때 공부했던 프로세스의 내용들을 복습하고 넘어가자.

### Dual-Mode Operation

![Untitled](Process%206ca07a341ce24a5da733bc698bc154fc/Untitled.png)

OS는 일반 사용자들이 사용할 수 있는 User Mode와 OS가 돌아가는 Kernel Mode로 나뉜다. 주로 Hardware Interrupt, Software Interrupt(exception), System call 등에 의해 mode switching이 발생한다.

### Process

<그림 추가>

**프로세스는 프로그램의 이미지가 메모리에 올라온 동적인 상태**를 말한다. 프로세스는 크게 **이미지와 *process context*로 구성**되며 이미지에는 code, data 등 프로세스를 움직이는 물리적인 component들이, process context에는 프로세스가 사용하는 ***register*의 상태인 *process context***와 프로세스에 대응되는 **커널의 상태인 *Kernel context***가 있다. 

### Thread

<그림 추가>

하나의 프로세스에 대한 여러 개의 복사본이 생길 경우 이들을 모두 복사하는 것은 메모리 낭비이기에 kernel stack 등 공유가 불가능한 각 프로세스의 고유 정보들을 제외하고 나머지 code, data 등은 공유를 하여 메모리 사용의 효율성을 높이는 개념이 스레드이다. 하나의 프로세스가 여러 스레드를 생성할 경우 각 스레드가 따로 따로 스케쥴링될 수 있기에 멀티 프로세서의 이점을 충분히 사용할 수 있다.

## Linux Process Descriptor (PCB)

이제 프로세스가 어떻게 리눅스에 구현되어 있는지 알아보자. 먼저 위에서 얘기한 process context, 이미지 등 **프로세스의 정보를 담을 자료구조**가 필요하다. 이것이 바로 ***Linux Process Descriptor***이다. `task_struct`라는 이름의 자료구조 안에 프로세스와 관련된 다양한 정보들이 담겨 있으며 **커널이 이를 관리**한다.

![Untitled](Process%206ca07a341ce24a5da733bc698bc154fc/Untitled%201.png)

각 프로세스는 자신만의 PCB를 가지며 ***PID***를 통해 구분된다.

## User Stack & Kernel Stack

프로그램을 실행시키면 각종 데이터들이 스택에 쌓이고 없어진다. 커널도 마찬가지이다. 커널도 하나의 코드이기에 Kernel mode로 가서 커널 코드를 실행하면 **그 정보들을 담기 위한 스택이 필요**하다. **그래서 *User mode*의 스택과 *Kernel mode*의 스택이 모두 필요**하다.

Mode Switching이 발생하면, CPU의 레지스터는 한 세트 밖에 없기 때문에 **커널도 이 레지스터들을 써서** 자신의 코드를 수행해야 한다. **즉, *User mode*에서의 레지스터 정보들을 모두 *Kernel Stack*에 저장해두고 커널 코드를 실행한다.** 그리고 끝나면 스택에서 다시 꺼내면서 원상복구를 한다.

근데 프로세스마다 실행하는 커널 코드가 다르기에 커널 스택에 저장되는 정보들도 다 다를 것이다. 따라서 **프로세스마다 커널 스택이 하나씩 필요**하다.

### Implementation

그러면 프로세스마다 커널 스택과 PCB가 필요한데 커널에게 주어진 공간은 1G 뿐이다. 그리고 동시에 실행되는 프로세스는 굉장히 많다. 그렇다면 리눅스는 이를 어떻게 구현했을까?

![Untitled](Process%206ca07a341ce24a5da733bc698bc154fc/Untitled%202.png)

각 프로세스에게 8KB에 해당하는 커널 메모리가 제공된다. 이 메모리 공간에는

1. `thread_info` 자료구조와
2. 커널 스택이 저장된다.

`esp` 레지스터의 경우 현재 커널 스택의 top을 가리키고 있고, 페이지의 단위가 8KB이기 때문에 하위 13비트($2^{13}$= 8KB)를 0으로 만들면 페이지의 바닥인 `thread_info`의 주소를 알 수 있다. 

그리고 `thread_info` 안에 해당 PCB의 포인터가 존재하여 PCB에도 접근이 가능하다.

**즉, `esp`를 알면 PCB도 알 수 있다!**

## Process list

커널은 프로세스들을 스케줄링하고 관리하려면 모든 프로세스들에 대한 정보를 알아야하며 쉽게 관리할 수 있어야 한다. 그러면 커널은 존재하는 모든 프로세스를 어떻게 관리할까?

![Untitled](Process%206ca07a341ce24a5da733bc698bc154fc/Untitled%203.png)

먼저 모든 프로세스는 이중 연결 리스트 구조로 연결되어 있다. `task_struct` 내의 `list_head` 자료구조로 선언되어 있는 `tasks` 안에 `next`, `prev` 포인터가 해당 프로세스의 다음, 이전 프로세스를 가리킨다. 또한, `task_struct` 내에 pid hash table과 관련된 자료구조들(e.g.. `pid_links`)도 존재하여 특정 PID의 PCB를 더 빠르게 찾을 수 있게 해준다.

## Wait Queues

프로세스가 돌면서 여러 이벤트가 발생하는데 그 중에서 **특정 이벤트를 기다려야 할 수도 있다.** 예를 들어, 프로세스가 I/O 관련 시스템 콜을 하여 커널 모드로 들어간 뒤,  ***I/O operation*의 수행을 기다리는 *block* 상태**가 될 수 있다.

이 경우 프로세스가 해당 I/O operation을 기다리는 장소가 필요하다. 이렇게 프로세스가 이벤트의 발생을 기다리는 큐, **특별히 커널 관점에서는 자고 있는 프로세스가 있는 곳이 *wait queue*** 이다. I/O controller 마다 wait queue가 존재하며, 특정 이벤트가 발생하면 I/O controller는 커널에 이벤트가 발생했다는 ***interrupt*를 건다.** 

**이 *interrupt*를 커널이 받아 *wait queue*에서 자고 있는 프로세스를 깨워 다시 *TASK_RUNNING* 상태로 바꾼다.**