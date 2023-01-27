# Memory Management - Part 1

2023.01.05

---

## Introduction

앞에서는 하드웨어가 어떻게 linear address를 physical address로 변환해주는지 알아보았다. 하지만 커널도 현재 메모리 상황에 대해 빠삭하게 알고 있어야한다. 그래야 새로운 프로세스가 만들어졌을 때 PCB를 메모리 어디에 할당할지 등 빠르게 작업을 수행할 것이기 때문이다. 

이번 포스팅에서는 커널이 어떠한 자료구조와 정책을 가지고 메모리를 관리하는지 알아보자. 개인적으로 이 부분이 메모리에 있어 가장 중요한 부분이라 생각하며 처음 공부할 때는 이런 것까지 관리를 하나 싶을 정도로 커널이 굉장히 자세하게 메모리를 관리하고 있다는 느낌을 받았다.

## Linear Address Space

이전에 OS 시간에 배웠던 linear address space에 대하여 다시 한 번 가볍게 짚고 넘어가자. 

![Untitled](Memory%20Management%20-%20Part%201%203998f819ca6b40c5a4df192902ad9d03/Untitled.png)

Process를 공부하면서 각 process의 linear address space는 user address space와 kernel address space로 나뉜다고 하였다. 근데 생각해보면 kernel address space는 모든 프로세스가 동일할 것이므로 하나의 master를 두어 이를 공유하는 것이 더 효율적이다. 후에 살펴보겠지만 이는 `swapper_pg_dir` 이라는 것을 사용하여 구현한다.

기본적으로 3:1로 나누는 것이 가장 효율적이라고 알려져 있지만 커널이 동작하는 상황에 따라 2:2, 1:3으로도 나누는 것이 가능하다.

## Physical Memory Mapping in Linux

![Untitled](Memory%20Management%20-%20Part%201%203998f819ca6b40c5a4df192902ad9d03/Untitled%201.png)

그러면 linear address space 위의 것들은 physical memory 올라와야 컴퓨터에서 실행이 될 것이다. 이것이 계속해서 얘기하는 폰-노이만 구조와 On-Demand Paging의 기본 개념이다. 그리고 기본적으로 Paging을 통해서 매핑이 된다고 이야기하였다. 

### Direct Mapping

**Kernel의 Code와 Data는 모든 프로세스에서 동일하고 swap out되서는 안되기 때문에 이들은 Physical memory에 pinned 되어 실행이 된다.** 그러면 나머지 Kernel Address Space도 paging을 통해서 매핑이 될까? 

일단은 맞다. 잠시 뒤에 설명하겠지만 각 프로세스마다 Page Global Directory가 있으니 여기의 상위 1/4는 Kernel address를 매핑하는데 사용된다. 

하지만, 기본적으로 더 쉬운 방법으로 매핑이 된다. 바로 위의 그림과 같이 그냥 주소값에서 `PAGE_OFFSET`만큼 빼주는 것만으로 Physical Memory에 매핑을 해주는데 그 이유는 다음과 같다.

- Page table과 같이 추가적인 매핑 작업 없이 실행할 수 있기 때문에 Kernel의 코드들을 더 빠르게 실행할 수 있다.
- DMA transfer에 적합하다. DMA의 I/O 장치들은 메모리가 한정적이어서 paging을 쓰지 못하고 메모리의 일정 부분을 연속적으로 사용하여 operation을 수행하기 때문에 paging을 쓰면 복잡해져서 direct map- ping으로 처리하는 것이 더 간단하다.

이때, Kernel address가 Physical Memory에 Direct Mapping되는 부분을 lowmem, Direct Mapping 하고 남은 부분을 highmem이라고 부른다.

### Vmalloc & Persistent mapping

근데 그림을 자세히 보면 1G 전체가 아니라 896M만 Direct mapping을 한다. 나머지 128M는 무엇을 위해서 사용되는걸까? 아래의 그림은 Kernel address space가 physical memory에 어떻게 매핑이 되는지 조금 더 구체적으로 나타낸 그림이다.

![Untitled](Memory%20Management%20-%20Part%201%203998f819ca6b40c5a4df192902ad9d03/Untitled%202.png)

커널이 실행되면서 Physical memory의 frame들이 allocate/release를 반복하게 되는데 이 과정에서 external fragmentation이 발생하게 된다. 중간중간에 쪼가리 frame들이 남는다는 것이다. 그러면 발생할 수 있는 문제점이, 커널이 여러 개의 연속적인 frame을 요청했을 때 메모리 전체의 비어있는 frame을 모으면 할당을 해줄 수 있지만 여러 군데에 분산되어 있으니 연속적으로 할당을 할 수 없는 상황에 놓이게 된다.

***Vmalloc area*는 이와 같이 불연속적인 physical memory의 frame을 연속적인 Kernel address space에 매핑시켜주는 공간**이다. 커널을 100% 만족시킬 수는 없지만 나름 최선을 다하여 연속적으로 frame을 할당해주는 방법인 것이다. 커널에서 `vmalloc()` 함수를 통하여 해당 부분을 사용할 수 있다.

하지만 단점은 보다시피 direct mapping이 아닌 mapping이기 때문에 kernel의 page table을 업데이트 해주어야한다. 

두번째로는 kernel이 Highmem에 매핑하기 위해 사용되는 persistent mapping을 위하여 사용된다. 하지만 여기는 시간이 많이 걸려서 잘 사용하지는 않는다.

## Constructing Page Tables

각 프로세스마다 4G의 virtual address 공간을 쓰고, 이에 대한 page table을 구축해야 한다. 각 프로세스마다 하나의 PGD가 있고, 32-bit라 했을때 전체 1024개의 엔트리 중에서 상위 256개의 엔트리는 Kernel Page Table을 위한 엔트리인데, 이 엔트리가 앞에서 얘기했던 `swapper_pg_dir`이다. 이 swapper_pg_dir은 위의 그림에서와 같이 커널의 코드와 함께 pinning 되어 있으며 새로운 프로세스가 생성될 때마다 page table에 swapper_pg_ dir의 엔트리를 복사하여 넣기만 하면 된다. 

Kernel page table이 초기화되는 과정은 다음과 같다.

1. 처음에는 8MB만 가지고 커널의 코어 자료구조들을 초기화한다.
2. 일정시간 이후부터는 전체 메모리에 대한 Kernel page table을 세팅한다. `paging_init()`을 통하여 low memory 만큼 direct mapping을 하고 `swapper_pg_dir`로 reinitialize를 한다.

User page table의 경우에는 각 프로세스에 따라 사용되는 양상이 다르다. 이를 종합적으로 나타낸 것이 아래의 그림이다.

![Untitled](Memory%20Management%20-%20Part%201%203998f819ca6b40c5a4df192902ad9d03/Untitled%203.png)

Kernel address space는 기본적으로 direct mapping 되지만 vmalloc을 위하여 kernel page table이 존재한다. 또, User address space는 일반적인 paging을 통하여서 physical frame에 매핑된다. 

**즉, 각 frame의 주인은 User와 Kernel 모두 가능하다.** 위의 그림에서 frame을 보라색으로 칠한 이유가 **User**와 **Kernel**이 모두 사용가능하다는 것을 나타내기 위함이다.

여기까지가 Linux가 Virtual memory를 어떻게 Physical Memory에 매핑하는지에 대한 것들이다. 다음 파트에서는 Linux가 Physical Memory를 어떤 자료구조를 가지고 관리하는지, 그리고 어떻게 Physical Memory를 할당하는지에 대한 **Kernel Memory Allocation(KMA)**에 대하여 알아보겠다.