# Transport Layer

2023.01.13

---

## Introduction

이번 포스팅에서는 transport layer가 제공하는 여러 서비스와 대표적인 transport layer protocol인 UDP, TCP에 대하여 알아보자.

## Transport Layer Services

![Untitled](Transport%20Layer%2016e778a39b6e49ddb6ed99e143dcc6c0/Untitled.png)

Transport Layer는 Application Layer를 대신해서 ***Application Layer*가 다른 *end system*의 프로세스에 보내려는 메세지를 *Network Layer*로 내려주는 역할**을 한다. 자신이 원하는 프로세스에 메세지를 보내야 되기 때문에 **프로세스 간의 *logical communication*을 지원**한다.

Application layer에서 메세지를 받으면 ***sender*는 해당 메세지를 *segment* 단위로 나누어 *Network Layer*에 전달**하며 receiver는 그 segment들을 다시 조합해서 원래 메세지를 Application Layer에 전달한다.

Network Layer가 호스트끼리의 logical communication을 제공한다면, Transport Layer는 호스트 내의 프로세스 간의 logical communication을 지원한다.

## Multiplexing & Demultiplexing

**우리가 보내는 메세지나 데이터는 우리가 원하는 올바른 프로세스가 수신**을 해야한다. 호스트는 IP 주소로 구분이 되지만 호스트에는 여러 프로세스가 있기 때문에 프로세스는 IP 주소만으로 구분하기에는 정보가 부족하다. 그래서 이를 구분해주는 것이 바로 **포트 번호**이다. 이 포트 번호를 붙이고 또 해석하는 작업이 Multiplexing/ Demultiplexing 이다.

![Untitled](Transport%20Layer%2016e778a39b6e49ddb6ed99e143dcc6c0/Untitled%201.png)

- ***Multiplexing***: Transport Layer에서 이 segment를 어디로 보낼지에 대한 헤더를 붙이는 작업
- ***Demultiplexing***: 받은 segment를 올바른 socket으로 보내기 위해 헤더를 해석하여 포트 번호를 확인하는 작업

인터넷은 TCP와 UDP의 두 가지 transport service를 제공하기에 두 service의 multiplexing/demultiplexing 방법도 다르다.

1. ***UDP***: UDP는 ***connectionless service***이기에 sender 쪽에서 헤더에 **받는 쪽의 IP 주소와 포트 번호만 표시**한다. 즉, 보내는 쪽의 IP 주소와 포트 번호가 다르더라도 **같은 *receiver*의 포트 번호를 표시했다면 그 소켓들은 모두 동일한 소켓으로 인식**한다.
2. ***TCP***: TCP는 ***connection-oriented service*** 이기에 ***sender*의 IP 주소와 포트 번호, *receiver*의 IP 주소와 포트 번호를 모두 표시**한다. 즉, 같은 receiver라도 보내는 쪽이 다르면 그 소켓들은 모두 구분된다. 이렇게 connection을 구분하는 이유는 **각각의 *connection*에 대해 *congestion control*이 가능해지고, 다시 보내달라고 요청**할 수도 있기 때문이다. 

## User Datagram Protocol (UDP)

***UDP*는 아무것도 하지 않고 다만 최선을 다해서 전송을 하는 *transport protocol***이다. 그렇기에 중간에 seg-ment가 사라질 수도 있다. UDP는 connectionless이기에 TCP 보다 **더 구현이 간단하고 빠르다는 장점**이 있지만 안정성은 떨어진다.