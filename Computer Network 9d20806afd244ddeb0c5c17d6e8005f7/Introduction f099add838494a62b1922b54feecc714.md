# Introduction

2023.01.08

---

## What’s the Internet?

![Untitled](Introduction%20f099add838494a62b1922b54feecc714/Untitled.png)

일상 생활에 빗대어 위의 그림을 해석해보자.

우리는 주로 컴퓨터나 스마트폰을 사용해서 정보를 검색하고 여러 서비스들을 사용합니다. 이때 주로 와이파이나 데이터를 이용하여 서비스들을 이용하죠. 정보를 요청하는 것입니다. 이 와이파이나 데이터로 공유기 또는 기지국에 연결되는 것을 *mobile network*, *home network* 라고 할 수 있습니다. 얘들이 받아서 서버로 요청을 전달하는 것이죠.

이 요청도 크기가 크다면 **패킷**이라는 단위로 나뉘어서 보내집니다. 하지만 서버로 한번에 갈 수는 없습니다. 멀리 있을 수 있기 때문에 여러 네트워크를 거쳐서 가야하며 그 **네트워크를 이어주는 것이 바로 라우터(*router*)**입니다. 

라우터는 아무 형태의 요청을 받지 않습니다. 라우터가 알아들을 수 있는 형태의 요청만을 보고 해석하여 다음 라우터로 보내주죠. 이 형식이 **프로토콜(*protocol*)**입니다.

라우터의 가장 큰 역할은 패킷을 어느 라우터로 보낼 것인지 결정하는 일입니다. 각 패킷에는 IP 주소가 있어서 **어느 라우터로 보낼지 결정하고 전달**할 수 있죠. 이것을 ***Packet switch***라고 합니다.

**Internet은 이 네트워크들의 네트워크**이다. (Interconnected ISP) 여기서 ISP는 *Internet Service Provider*의 약자로 우리나라의 KT, SKT와 같이 개인이나 기업체에게 인터넷 접속 서비스, 웹사이트 구축 및 웹호스팅 서비스 등을 제공하는 회사를 말한다.

## Network Edge

![Untitled](Introduction%20f099add838494a62b1922b54feecc714/Untitled%201.png)

- Host (End System): 우리가 사용하는 기기들
- Edge router: Network router로 들어가기 위한 첫 라우터. 이 라우터에 연결하는 네트워크가 access network
- Physical media: Copper, fiber와 같이 실제로 비트가 전송되는 매개체. Copper, fiber와 같이 유선의 형태와 radio, wi-fi 같이 무선의 형태가 존재

호스트는 데이터를 ***chunk(packet)***로 잘라서 보낸다. link의 transmission rate이 R이고, L비트의 데이터라면 패킷을 전송하는데 드는 시간은 $L/R$ 이다.

## Network Core

라우터들이 상호 연결되어 있는 덩어리(mesh)

### Packet Switching

![Untitled](Introduction%20f099add838494a62b1922b54feecc714/Untitled%202.png)

라우터의 가장 큰 기능은 **패킷을 보고 어디로 보낼지 결정하는 *routing***과 **해당 라우터로 전송하는 *forwarding***이다. 이에 대해서는 후에 4장에서 더 자세히 살펴볼 것이다. 이 두 기능을 통하여 패킷을 전달하는 것이 packet switching이다. 

라우터는 기본적으로 패킷을 ***Store and forward***한다. 라우터에 들어오는 패킷이 여러개이기 때문에 버퍼에 저장했다가 차례로 내보내는 형식을 취하는데 만약에 패킷이 도착하는 속도(R)가 패킷을 전송하는 속도(R’)보다 크다면

1. 패킷이 큐에 쌓이게 되어 ***delay***가 발생하고
2. 큐가 다 차버리면 패킷이 사라지는 ***loss***가 발생하게 된다.

### Circuit Switching

Packet Switching의 경우 패킷이 중간에 사라질 수 있기 때문에 이를 방지하기 위하여 데이터를 보내기 이전에 먼저 **필요한 네트워크를 할당받아 성능을 보장하는 방법**이다. 여기에는 대역폭을 나눠서 할당하는 FDM과 네트워크를 사용하는 시간을 나눠서 할당하는 TDM이 있다.

![Untitled](Introduction%20f099add838494a62b1922b54feecc714/Untitled%203.png)

이렇게 할당 받은 circuit은 공유를 하지 않기 때문에 성능은 보장할 수 있지만 사용하지 않을 경우 해당 네트워크를 낭비하는 것이기 때문에 효율성의 부분에서는 Packet Switching 보다 떨어진다.

예를 들어, link가 1Mb/s 이고, 각 사용자가 100kb/s를  사용하며, 전체 시간의 10%만 사용한다고 했을 때

- Circuit: 모든 사용자에게 동일한 성능을 보장해야하기 때문에 1Mb / 100Kb = 10명까지 사용가능하다.
- Packet: 하지만 이는 10명만 동시에 사용하지 않으면 더 많은 사용자가 link를 사용할 수 있다는 것과 같다. 만약, 35명이 해당 네트워크를 사용할 경우, $\Sigma{35\choose n}p^n(1-p)^{35-n}\leq0.0004$로 무리 없이 35명이 사용가능하다.

그렇다면 packet switching이 무조건 좋은 것이냐고 하면 그건 또 아니다. 네트워크를 더 많은 사람들이 사용할 수 있어 편리하지만, 한번에 많은 사람들이 사용하게 될 경우 패킷의 지연 및 손실이 발생하게 되고, 이를 메꾸기 위하여 패킷을 다시 전송하며 **한꺼번에 많은 패킷이 몰리는 *excessive congestion* 현상**이 발생할 수 있다.

## Delay, Loss, Throughput

### Delay & Loss

![Untitled](Introduction%20f099add838494a62b1922b54feecc714/Untitled%204.png)

패킷의 전송과정에서 발생하는 delay는 위의 네 가지 과정에서 발생하는 delay의 합으로 생각할 수 있다.

$$
d_{nodal}=d_{proc}+d_{queue}+d_{trans}+d_{prop}
$$

- $d_{proc}$: 라우터 내에서 다음 링크를 결정하는 시간. 매우 짧다.
- $d_{prop}$: 실제 링크를 타고 전달되는 시간. 역시 매우 짧다.
- $d_{trans}$: $d_{prop}$과 달리 라우터에서 모든 패킷을 링크로 내보내는 시간이다. link의 대역폭을 $R$, 패킷의 길이를 $L$이라고 하면 그 시간은 $L/R$이 된다.
- $d_{queue}$: 패킷이 큐에서 대기하는 시간이다. 라우터에 초당 패킷이 얼마나 들어오느냐에 따라 달라진다.

위의 delay들 중에서 $d_{trans}$와 $d_{queue}$가 delay에 가장 영향을 많이 미친다.

### Throughput

![Untitled](Introduction%20f099add838494a62b1922b54feecc714/Untitled%205.png)

Throughput은 초당 얼마만큼의 bit를 sender와 receiver 사이에 전달할 수 있는지를 수치로 나타낸 것이다. 위의 그림에서 $R_s$가 $R_c$보다 크면 **병목현상**이 발생하여 패킷의 지연과 손실이 발생하게 된다. 

## Protocol layers

네트워크를 사용하는 기기들도 많고 지원해야 하는 기능도 많아서 되게 복잡하다. 그래서 이를 **구조화**한 것이 ***layering***이다. **각각의 layer는 하나의 서비스만을 수행하며, 각 서비스는 인접 layer에 의존한다.** 이렇게 layering을 하게 되면

- 복잡한 시스템의 구조를 명확히 하여 **관계를 쉽게 파악**할 수 있고
- **유지관리가 쉬워진다.**

대표적인 layering이 ***Internet protocol stack***이다.

![Untitled](Introduction%20f099add838494a62b1922b54feecc714/Untitled%206.png)

- application: 네트워크를 사용하는 application에 서비스 제공(FTP, HTTP)
- transport: process간 데이터 전달(TCP, UDP)
- network: 데이터들을 라우터로 전송(IP)
- link: 데이터를 실제로 전달해주는 기능(Ethernet)
- physical: 실제 wire

보내려는 데이터가 있으면 각 layer에서 해당 layer끼리 알아들을 수 있는 **header를 데이터에 붙여서 보내고(*encapsulation*)** 받는 쪽의 layer에서 해당 header를 해석하여 데이터를 읽는다.

다음 포스팅부터 이 protocol의 stack을 위에서부터 하나씩 알아가보자.