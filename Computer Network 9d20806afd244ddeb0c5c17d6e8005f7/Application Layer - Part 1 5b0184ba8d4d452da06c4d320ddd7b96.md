# Application Layer - Part 1

2023.01.10

---

## Introduction

Internet Protocol Stack의 5가지 레이어 중 가장 위에 있는 Application Layer 부터 알아보자.

## Principles of Network Applications

컴퓨터 또는 스마트폰으로 사용하는 프로그램이나 앱은 거의 대부분 네트워크를 사용한다. 사람들과 메세지를 주고 받기도 하고, 동영상을 보기도 한다. 그러면 이렇게 네트워크를 사용하는 프로그램들이 어떻게 동작할까? 

프로그래머 입장에서 프로그램을 만들 때 어느 포트로 나가는 등 네트워크의 모든 것을 신경쓰기에는 벅차다. 프로그램을 오류 없이 만들기도 바쁜데 네트워크를 일일히 구현하고 따지고 있을 수 없기 때문이다. Application Layer는 이러한 부담을 덜어준다. ***Network core device*는 신경쓰지 않고 제공되는 서비스만 잘 이용하면 된다.**

만들 수 있는 application의 구조에는 크게 두가지가 있다.

### Client-Server

### Peer-To-Peer (P2P)

![Untitled](Application%20Layer%20-%20Part%201%205b0184ba8d4d452da06c4d320ddd7b96/Untitled.png)

서버는 계속 켜져있고 클라이언트가 서버에 요청을 보내면 해당 요청에 대응되는 응답을 보내는 구조이다. 웹 브라우징, 유튜브 등이 이에 속한다.

![Untitled](Application%20Layer%20-%20Part%201%205b0184ba8d4d452da06c4d320ddd7b96/Untitled%201.png)

계속 켜져있는 서버 없이 클라이언트(peer)끼리 직접적으로 통신을 하며 하나의 peer가 다른 peer로부터 서비스를 받기도 하고 제공하기도 한다. 즉, 각각의 peer가 서버가 될 수 있기에 확장성이 크다. 토렌트가 이에 속한다.

![Untitled](Application%20Layer%20-%20Part%201%205b0184ba8d4d452da06c4d320ddd7b96/Untitled%202.png)

두 구조에서 공통된 점은 호스트가 다른 호스트와 통신을 한다는 것이다. 즉, 하**나의 호스트에서 수행되고 있는 프로세스가 다른 호스트의 프로세스로 메세지를 날리는 것과 동일**하다. 하지만 application layer는 우리의 프로그램이기에 다른 호스트의 application layer과 직접 통신이 불가능하다. 결국에 메세지는 physical link를 통하여 전송되기 때문이다. 

이 메세지를 physical layer까지 내려보내야 하는데 중간에 다뤄야 할 이슈들이 몇가지 존재한다. 그래서 메세지를 바로 physical link로 보내지 않고 먼저 ***transport layer*라는 곳으로 내려보내는데 이를 도와주는 것이 바로 소켓(*socket*)**이다.

![Untitled](Application%20Layer%20-%20Part%201%205b0184ba8d4d452da06c4d320ddd7b96/Untitled%203.png)

그러면 Application은 어떤 transport service가 필요할까?

- Data Integrity: 데이터를 보내는 중에 없어져서는 안되는 app이 있고, 조금은 없어져도 괜찮은 app이 있다. 하지만 우리의 프로그램은 왠만해서 데이터가 없어지지 않는다고 생각하고 데이터를 보낸다.
- Timing: 데이터가 빨리 도착해야하는 app이 있고, 그렇지 않은 app이 있다.
- Throughput: 최소한 이만큼은 처리가 되면 좋겠는 app이 있고, 상관이 없는 app이 있다.

Application은 위의 service들이 보장된다고 생각하고 데이터를 보내기 때문에 transport layer에서 이들을 보장해주어야 한다. Transport layer service에는 크게 ***reliable, connection-oriented*한 *TCP***와 ***unreliable*한 *UDP***가 있다.

## Web & HTTP

HTTP는 **웹의 *application layer protocol***이며 client-server model이다.

![Untitled](Application%20Layer%20-%20Part%201%205b0184ba8d4d452da06c4d320ddd7b96/Untitled%204.png)

다음은 클라이언트가 웹 페이지에 접속하려고 할때 일어나는 일들이다.

1. 클라이언트가 port 80으로 서버에 TCP connection을 요청
2. 서버는 이를 승낙
3. HTTP 메세지가 클라이언트와 서버 사이에 오감
4. 서버는 클라이언트가 요청한 사이트를 보여주며 이후 TCP connection이 종료

이렇게 연결이 종료가 되면 서버는 이전 클라이언트에 대한 정보를 잊어버린다. 즉, ***HTTP*는 *stateless***이다.

HTTP에도 두가지 방식이 존재한다.

1. ***Non-persistent HTTP***: **한번의 TCP 연결로 최대 하나의 *object*만 전송**한다. 여러 object를 보내려면 여러 connection이 필요하기 때문에 오버헤드가 큰 방식이다. 하나의 object를 보내는데 TCP로 연결하는 시간에 파일 전송 시간을 더한 시간이 걸린다.
2. ***Persistent HTTP***: ***Response*를 보낸 후에도 *connection*을 유지**하는 HTTP 방식이다. 하나의 connection에 대해서 여러 object를 전송할 수 있기에 모든 object를 전송하는데 non-persistent HTTP보다 더 적은 시간이 소요된다.

기본적으로 HTTP는 stateless 하지만 보통 웹사이트에 접속하면 나를 알고 있다는 듯이 이전에 봤던 상품들을 보여주거나 이전에 봤던 곳 이후부터 보여준다든가 좀 더 유저 친화적인 웹사이트를 볼 수 있다. 이런 방법들은 대부분 **쿠키(*cookie*)**를 이용한 방법이다. **쿠키가 일시적으로 유저의 *state*를 저장해준다.**

![Untitled](Application%20Layer%20-%20Part%201%205b0184ba8d4d452da06c4d320ddd7b96/Untitled%205.png)

구체적인 동작 과정은 위 그림과 같다.

1. 유저가 웹사이트에 접속하려는 HTTP request를 서버에 보낸다.
2. 서버는 해당 유저가 처음 방문하는 유저임을 알고 해당 유저의 ID를 만들어 DB에 저장한다.
3. 위의 **ID를 *Response* 메세지에 담아서 보낸다.**
4. 그러면 **다음부터 요청을 보낼 때는 이 ID를 헤더에 같이 담아서** 서버에 보낸다.
5. 서버는 이 쿠키 정보를 가지고 관련된 정보를 DB에서 찾아다가 유저에게 보여준다. 

### Web Cache

기본적으로 클라이언트가 HTTP request를 하면 서버까지 가서 그 object를 가져온다. 근데 이게 시간이 많이 걸리고, 동시에 여러 클라이언트가 하나의 웹서버에 접속하게 되면 HTTP request가 몰려서 ***core*에 *congestion*이 발생**하게 된다. 그래서 **클라이언트와 서버 사이에 이 *object*를 *caching*하는 *proxy server*를 두어 응답 시간도 빨라지고 트래픽도 감소**시킨다.

![Untitled](Application%20Layer%20-%20Part%201%205b0184ba8d4d452da06c4d320ddd7b96/Untitled%206.png)

근데 ***proxy server*에 *caching* 되어있는 *object*가 최신것이 아니면 클라이언트에게 보내서는 안된다.** 이를 막기 위해서 클라이언트는 서버에게 ***Conditional GET***으로 요청을 보낸다. 헤더에 `If-modified-since`를 넣어서 이날 이후에 수정이 되었는지를 물어보고, **수정이 안되었다면 *object* 없이 메세지만** 보내고 **수정이 되었다면 *object*를 넣어 같이 보낸다.**

## Electronic mail

mail이 전송되는 원리와 그 protocol에 대해 알아보자.

![Untitled](Application%20Layer%20-%20Part%201%205b0184ba8d4d452da06c4d320ddd7b96/Untitled%207.png)

***SMTP*는 메일 서버간의 메일을 주고 받는 *protocol***이다. 메일이 전송되는 과정을 자세하게 표현하면 다음과 같다.

1. 네이버나 지메일에서 메일을 작성하고 전송을 누르면 SMTP를 통해 sender의 메일 서버로 보내진다.
2. sender의 메일 서버에서 SMTP를 통해 receiver의 메일 서버로 보내진다.
3. receiver는 mail access protocol을 이용하여 자신의 메일 서버에서 메일을 받아온다.

기본적으로 SMTP는 TCP를 사용하기에 ***handshaking, msg transfer, closure***의 세 과정을 거친다. 메일은 header와 body로 구성되며 헤더에는 어디로 보내고, 누가 보내는지, body에는 실제 메세지가 담겨서 SMTP를 통해 전달된다.