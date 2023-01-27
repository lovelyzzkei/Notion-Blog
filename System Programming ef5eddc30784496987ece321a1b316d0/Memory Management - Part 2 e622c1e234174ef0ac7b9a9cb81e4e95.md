# Memory Management - Part 2

2023.01.07

---

## Introduction

저번 포스팅에서는 우리의 Program Counter가 도는 Virtual memory가 어떻게 Physical memory에 매핑이 되는지에 대하여 알아봤다. 이번에는 커널이 어떠한 자료구조를 가지고 Physical memory를 관리하며 어떤 방법으로 Physical memory의 frame들을 할당해주는지 알아보자.

## Physical Memory Management

먼저, 커널이 Physical memory를 관리하는 자료구조에 대하여 알아보자. 이를 관리하는 이유는 각 frame이 비어있는지, 그렇지 않다면 어떤 프로세스가 이를 사용하는지를 파악하고, 빈 frame들을 파악하여 커널이 frame을 요청하면 빠르게(low latency) 할당해주기 위함이다.

### Page frame

![Untitled](Memory%20Management%20-%20Part%202%20e622c1e234174ef0ac7b9a9cb81e4e95/Untitled.png)

커널은 physical memory의 frame을 `page`라는 자료구조를 두어 여기에 각 frame 자체에 대한 정보를 저장한다(이름이 virtual memory의 page와 헷갈릴 수 있음에 유의하자). 이 frame이 비어있는지, 그렇지 않다면 누가 이 frame을 소유하고 있는지에 대한 정보를 저장하고 이를 `mem_map` 배열을 이용하여 관리한다. 

![Untitled](Memory%20Management%20-%20Part%202%20e622c1e234174ef0ac7b9a9cb81e4e95/Untitled%201.png)

![Untitled](Memory%20Management%20-%20Part%202%20e622c1e234174ef0ac7b9a9cb81e4e95/Untitled%202.png)

![Untitled](Memory%20Management%20-%20Part%202%20e622c1e234174ef0ac7b9a9cb81e4e95/Untitled%203.png)

또, `__pa(x)`, `virt_to_pfn(addr)` 등의 매크로도 지원하고 있어 해당 virtual address에 매핑된 physical address와 그 frame 등의 정보들도 얻을 수 있다.

### Node

컴퓨터의 메모리 설계에는 두가지 방법이 있다.

- ***UMA (Uniform Memory Access)***: 프로세스가 메모리에 접근하는 시간이 CPU에 상관없이 동일하다.
- ***NUMA (Non-Unifom Memory Access)***: 프로세스가 메모리에 접근하는 시간이 CPU마다 다르다.

Linux는 각 메모리를 node라는 모듈로 나누어서 UMA와 NUMA를 모두 지원한다. Linux에서는 `pglist_data`라는 자료구조를 두어 Node를 관리한다.

![Untitled](Memory%20Management%20-%20Part%202%20e622c1e234174ef0ac7b9a9cb81e4e95/Untitled%204.png)

위의 코드는 `<include/linux/mmzone.h`>에 있는 `pglist_data`의 코드 중 일부이다. 여기 코드를 보면 `zone`이라는 자료구조가 눈에 들어오는데 Linux는 전체 메모리를 크게 여러 개의 zone으로 나누어 관리한다. 즉, 큰 범위에서는 zone, 작은 범위에서는 frame으로 나누어 관리를 하는 것인데 그러면 node와 frame 사이를 잇는 zone에 대하여 알아보자.

### Zone

Zone은 메모리 frame들을 usage에 따라 그룹지어서 관리하는 자료구조이다. 메모리는 정말 다양한 방면에서 사용되는데, 앞에서 direct mapping 때 살짝 언급했듯이 DMA를 위해서도 사용되고, 일반적인 우리의 프로그램들을 실행하기 위해서도 사용되고, 또 Highmem을 위해서도 사용된다.

즉, 각 usage에 따라 frame들을 따로따로 관리하기 위하여 등장한 개념이 zone이다.

![Untitled](Memory%20Management%20-%20Part%202%20e622c1e234174ef0ac7b9a9cb81e4e95/Untitled%205.png)

Linux 코드에 보면 위와 같이 여러 종류의 zone을 정의해두었다. 위의 zone 이외에도 몇 개의 zone이 더 존재하는데 위의 zone들이 가장 일반적으로 사용되는 zone이다.

- `ZONE_DMA`: 적은 메모리를 가지는 DMA 기기들이 사용하는 메모리 frame의 집합
- `ZONE_NORMAL`: Kernel address space가 direct mapping 되는 메모리 frame의 집합
- `ZONE_HIGHMEM`: Highmem에서 동적으로 매핑되는 frame들의 집합.

Node, Zone, 그리고 Frame을 합쳐서 도식화한 그림이 아래와 같다.

![Untitled](Memory%20Management%20-%20Part%202%20e622c1e234174ef0ac7b9a9cb81e4e95/Untitled%206.png)

Node는 각 zone을 관리하며 각 zone에는 해당 zone에 포함되는 frame들을 관리하는 `mem_map` 배열과 빈 frame들을 관리하는 `free_area[x]`가 존재한다. 이 `free_area`는 후에 얘기할 memory allocation 기법의 buddy system을 위해 사용된다.

## Kernel Memory Allocation (KMA)

커널이 어떤 자료구조로 physical memory를 관리하는지 알아봤으니까, 마지막으로 **physical memory의 frame들을 커널이 어떻게 할당하는지에 대한 *Kernel Memory Allocation* 기법**들에 대하여 알아보자.

### Contiguous Page Frame Allocator

먼저는 **연속적으로** page frame들을 할당해주는 *contiguous page frame allocator* 이다. ‘연속적’ 이라는 강한 조건을 가진 allocator인데 왜 연속적으로 physical frame을 할당받는 것일까? 여기에는 크게 두 가지 이유가 존재하는데

- 많은 기기들이 paging을 지원하지 않아 **virtual memory를 사용할 수 없어서** **물리적으로 연속적인 *page frame*을 할당받아야 한다.**
- ***locality***를 통해 TLB overhead를 최소화하고 TLB reach를 늘릴 수 있다.

하지만 연속적인 frame을 할당받고 놓다보면 ***fragmentation(단편화)***가 발생할 수 밖에 없는데, 이를 최소화하기 위해 ***buddy system algorithm***을 사용한다. 처음에 zone allocator가 어느 zone에 할당할지를 결정하고 각 zone마다 buddy system을 따로 구현하여 요구하는 만큼의 physical frame을 할당해준다.

Buddy system에 대해서는 OS에서 알아봤으니 커널이 어떠한 자료구조를 가지고 buddy system을 운영하는지 알아보자.

![Untitled](Memory%20Management%20-%20Part%202%20e622c1e234174ef0ac7b9a9cb81e4e95/Untitled%207.png)

zone을 설명하면서 언급했듯이 buddy system은 free_area를 사용하여 수행된다. `free_area[x]`에는 `mem_`

`map`의 frame들 중에서 연속적으로 $2^x$개의 비어있는 frame들의 시작 포인터를 연결 리스트 형태로 저장한다.  커널은 `alloc_pages()`를 통하여 해당 크기 만큼의 frame을 반환해주고 `free_pages()`를 통하여 놓는다.

### Noncontiguous Page Frame Allocator

만약 physical하게 연속적인 frame이 없다면 불연속적인 page frame들을 모아다가 할당해준다. 이를 통하여 ***external fragmentation***을 피할 수 있지만 page table을 업데이트 해주어야 하는 overhead가 있기 때문에 필요할 때만 사용한다. 앞에서 봤던 ***vmalloc area***에서 `vmalloc()`으로 할당을 받으며 반복적으로 `alloc_page()`를 수행하고 page_table을 업데이트하는 방식으로 수행된다. 놓을때는 `vfree()`를 사용한다.

주로 다음의 목적들을 수행할 때 vmalloc을 사용한다.

- Module을 위한 공간
- I/O driver를 위한 버퍼
- Highmem frame을 사용

### Memory Object Allocator

커널에서 Memory Object라 함은 PCB, 소켓과 같이 커널 내에서 사용되는 자료구조를 의미한다. 이들 중에서는 **빈번하게 반복적으로 frame을 할당받고 release 하는 자료구조들도 존재하는데 이들에 대한 효율적인 관리**가 필요하다. 내가 방금 전에 release 하였는데 커널이 또다시 frame을 요청한다면 굳이 놓지 않는 것이 더 효율적일 것이기 때문이다. 이를 다뤄주는 것이 ***slab allocator***이다.

Slab allocator에는 두 가지 큰 특징이 있는데

- 각 memory object type 마다 ***memory cache*를 유지한다**
- 다 사용된 object 들은 release 하지 않고 **빈 object를 cache에 두어 후에 object를 다시 할당 받을 때 재사용**한다

![Untitled](Memory%20Management%20-%20Part%202%20e622c1e234174ef0ac7b9a9cb81e4e95/Untitled%208.png)

각 slab은 연속적인 frame으로 구성되며 이 안에 allocate된 object들도 있고, release된 object들도 존재한다. 커널이 자주 사용하는 object에 대해서는 `kmem_cache_alloc()`/`kmem_cache_free()`로 slab을 allocate/ release 하며 이외의 목적으로 배열과 같이 공간을 확보하기 위해서는 `kmalloc()`/`kfree()`를 사용한다.

Slab 자체를 추가적으로 allocate 하기 위해서는 다음 두 조건이 만족되어야 하는데

- 새로운 objcet에 대한 요청이 들어오고
- Slab에 빈 object가 없는 경우에만

새 slab을 할당하여 **최대한 slab allocate를 미루고** 메모리를 다른 목적으로 효율적으로 사용한다.

반대로 slab을 release 하기 위해서는

- Buddy system이 새로운 커널의 요청을 수행할 수 없고
- Slab이 모두 비어있는 경우에만

기존의 slab을 release 하여 역시 **최대한 slab release를 미룬다.**