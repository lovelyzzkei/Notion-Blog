# Memory Addressing

2023.01.04

---

## Introduction

지금까지는 리눅스가 어떻게 프로세스를 관리하는지에 대하여 전반적으로 알아보았다. 프로세스를 관리하는 가장 중요한 자료구조인 PCB부터 프로세스를 수행하는데 고려해야하는 가장 큰 요소인 interrupt 등 리눅스가 프로세스를 수행하는데 있어 고려해야하는 굵직한 주제들에 대하여 알아봤다.

지금부터 할 얘기는 메모리에 관한 얘기이다. 컴퓨터는 기본적으로 폰-노이만 구조의 A Stored Program Com-puter 구조를 따르기에 메모리를 효율적으로 사용하는 것이 매우 중요하다. 이번 챕터부터 나머지 챕터들에서는 리눅스가 어떻게 메모리를 사용하는지에 대하여 알아보자. 먼저는 **컴퓨터가 메모리를 사용하는데 있어 지원하는 여러가지 하드웨어 유닛들**에 대하여 알아보자. 

**이번 포스팅의 내용들은 모두 하드웨어에서 일어나는 일들을 설명한 것이다.**

## Memory Addressing

CPU에는 크게 인텔과 AMD가 있다. 그 중, 가장 많이 사용되는 인텔 아키텍쳐의 경우 다음의 세 메모리 주소를 사용한다.

- Logical Addresss: Segmentation을 위한 주소, 어셈블리 코드로 주어지는 주소
- Linear Address: Programmer가 다루는 주소. Program Counter가 돌아가는 주소
- Physical Address: 실제 RAM의 주소

중요한 점은 하드웨어 유닛에서 Logical Addr.이 Linear Addr.로, 다시 Linear Addr.이 Physical Addr.로 변환되는 일련의 과정이 자동적으로 이루어지며 우리가 다루는 주소는 모두 Linear Address라는 점이다.

### Segmentation

Logical Address가 Linear Address로 바뀌는 과정이 Segmentation이며 Segmentation Unit이 이를 수행한다. 주로 인텔 아키텍쳐가 사용하는 변환 방식이다. 그림으로 간략하게 나타내면 아래와 같다.

![Untitled](Memory%20Addressing%20e87f6fded9214b37badb554ca7fe4b99/Untitled.png)

Segmentation에 대해서는 OS에서도 다루었으니까 변환 과정에 대한 자세한 설명은 생략하겠다. 그러면 리눅스는 Segmentation을 어떻게 활용하는가?

결론부터 말하자면, **리눅스는 Segmentation을 거의 활용하지 않는다**. 그 이유는 메모리를 관리하는 과정이 Paging보다 더 복잡하며 Segmentation은 인텔 아키텍쳐에서만 주로 사용하기 때문에 **최대한 많은 아키텍쳐를 지원하기 위한 범용적인 운영체제**를 만들기 위해서는 적합하지 않은 기법이기 때문이다. 

몇천 개의 Descriptor 중 리눅스는 수십 개 밖에 활용을 안하는데 이 마저도 Descriptor의 base address가 모두 0이다. 즉, 4G의 address space를 하나의 segment로 인식하며 32비트의 offset으로만 linear address를 얻는다는 얘기이다.

대신 리눅스가 가장 애용하는 메모리 관리 기법은 **Paging**이다.

### Paging

Paging 역시 OS에서 다루었다. 그 큰 철학은 **On-Demand Paging**, 지금 사용하고 있는 부분만을 RAM에 올려 실행하자는 것이다. 그래서 연속적인 linear address의 page를 불연속적인 physical address의 frame에 매핑하는 과정이 Paging 이었다. 그 과정이 아래와 같다.

![Untitled](Memory%20Addressing%20e87f6fded9214b37badb554ca7fe4b99/Untitled%201.png)

위의 그림은 예전 리눅스에서 지원하던 Two-level paging을 도식화 한 것이다. 각 프로세스마다 매핑을 위한 테이블인 Page table이 필요하며 각 프로세스마다 하나의 Page Global Directory(PGD)가 존재한다. 

위의 변환과정을 위하여 필요한 것은 오직 PGD의 base address인데, 이것만 알면 나머지는 linear address의 offset을 통하여 자동적으로 physical address를 얻을 수 있기 때문이다. 그래서 **각 프로세스에는 이 pgd 값을 저장해놓는 공간이 있으며, 이를 CR3 레지스터에 올려놓으면 Paging Unit을 통해 자동적으로 physical address를 얻을 수 있다.**

현재 리눅스가 지원하는 Paging은 5-level paging인데 왠만해서는 4-level 정도만을 사용하고 있다. 그럼에도 이렇게 크게 만들어놓은 것은 후에 CPU와 메모리가 더욱 발전할 것을 생각하여 리눅스가 컴퓨터의 성능에 따라 유연하게 paging을 사용하기 위함이다. 

Linear address에서 physical address로의 변환은 하드웨어가 자동으로 해주지만 커널은 각 프로세스가 어떻게 메모리를 사용하는지 정확하게 파악해야 하기 때문에 실제 offset값과 각 page table의 base address를 알 수 있는 여러 매크로들을 지원하고 있다. 그것들이 위의 그림의 *PAGE_TABLE*_offset() 함수들이다.