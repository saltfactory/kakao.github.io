---
layout: post
title: 'CLOSE_WAIT & TIME_WAIT 최종 분석'
author: kaon.park
date: 2016-04-21 13:04
tags: [tcp,close-wait,time-wait,network,socket]
image: /files/covers/plug-socket.jpg
---
> 트래픽이 많은 웹 서비스를 운영하다보면 CPU는 여유가 있지만 웹서버가 응답을 제대로 처리하지 못하고 먹통이 되는 경우를 종종 보게 됩니다. 여러가지 이유가 있겠지만, 이 글에서는 가장 대표적인 경우인 `CLOSE_WAIT` 상태를 재현하고 원인과 문제점 그리고 해결책을 알아봅니다.
> 나아가 `TIME_WAIT`의 동작 과정을 직접 만든 예제와 리눅스 커널 소스를 통해 확인하고, 인터넷에 퍼진 낡은 그래서 더이상 유효하지 않은 정보들을 바로 잡습니다.

## Part I. `CLOSE_WAIT`

### `CLOSE_WAIT`로 인한 서버 행업 현상

서버 부하 테스트 과정 중 일정 시간이 경과하면 점점 더 느려지면서 행업 상태에 빠지는 경우가 생겼습니다. 부하가 높으면 느려지는건 당연한 일이지만, 더 골치아픈 문제는 테스트가 끝나도 행업 상태에서 복구되지 않았다는 점입니다. 이는 담당자가 매 번 상태를 확인하고 복구해야 함을 뜻하며 서비스에는 도입할 수 없을 정도로 치명적입니다. 분명히 특정한 원인이 있을 것이며 그에 따른 적절한 해결책이 존재할 것입니다.

먼저 행업 직전, 8080으로 서비스 중인 포트 상황은 아래와 같습니다:

```console
$ lsof -i:8080
COMMAND   PID   USER   FD   TYPE    DEVICE SIZE/OFF NODE NAME
java    27836 someone  1712r  IPv4 844147418      0t0  TCP 10.xx.xx.26:somehost->10.yy.yy.149:60494 (CLOSE_WAIT)
java    27836 someone 1720r  IPv4 844143466      0t0  TCP 10.xx.xx.26:somehost->10.yy.yy.149:60133 (CLOSE_WAIT)
java    27836 someone 1749u  IPv4 844151483      0t0  TCP 10.xx.xx.26:somehost->10.yy.yy.152:50739 (CLOSE_WAIT)
java    27836 someone 1754w  IPv4 844155954      0t0  TCP 10.xx.xx.26:somehost->10.yy.yy.149:61397 (ESTABLISHED)
java    27836 someone 1755u  IPv4 844151485      0t0  TCP 10.xx.xx.26:somehost->10.yy.yy.150:36813 (CLOSE_WAIT)
java    27836 someone 1764u  IPv4 844145322      0t0  TCP 10.xx.xx.26:somehost->10.yy.yy.149:60331 (CLOSE_WAIT)
java    27836 someone 1772u  IPv4 844146275      0t0  TCP 10.xx.xx.26:somehost->10.yy.yy.152:50196 (CLOSE_WAIT)
...
```

`ESTABLISHED` 한 개를 제외한 나머지 모두가 `CLOSE_WAIT` 상태입니다. 이 상태에서 꾸준히 증가하다 해당 서버의 경우 4,023개 째에서 행업되었습니다.

```console
$ netstat -tonp
tcp        1      0 10.xx.xx.26:8080           10.yy.yy.150:16011          CLOSE_WAIT  27836/java          keepalive (7196.98/0/0)
tcp        1      0 10.xx.xx.26:8080           10.yy.yy.149:40471          CLOSE_WAIT  27836/java          keepalive (7197.80/0/0)
tcp        1      0 10.xx.xx.26:8080           10.yy.yy.150:16287          CLOSE_WAIT  27836/java          keepalive (7197.62/0/0)
tcp        1      0 10.xx.xx.26:8080           10.yy.yy.149:40484          CLOSE_WAIT  27836/java          keepalive (7197.83/0/0)
tcp        1      0 10.xx.xx.26:8080           10.yy.yy.151:30980          CLOSE_WAIT  27836/java          keepalive (7197.09/0/0)
tcp        1      0 10.xx.xx.26:8080           10.yy.yy.152:30272          CLOSE_WAIT  27836/java          keepalive (7197.76/0/0)
tcp        1      0 10.xx.xx.26:8080           10.yy.yy.152:30147          CLOSE_WAIT  27836/java          keepalive (7197.46/0/0)
...
```

`netstat` 상태도 마찬가지인데 `-o` 옵션으로 networking timer를 살펴 봐도 거의 2시간이 설정되어 있어 의미가 없고, 이 상태에서 종료되지 않고 점점 증가합니다.

### TCP 상태(복습)

먼저 네트워크 서적의 바이블격인 [TCP/IP Illustrated](http://www.yes24.com/24/goods/8234905) 에 등장하는 TCP 커넥션 다이어그램은 아래와 같습니다.

![TCP 상태 전이 다이어그램](https://farm1.staticflickr.com/440/18338404268_f693b065d4_o.png)

이 중 `ESTABLISHED` 이후 종료 과정에서 어플리케이션의 `close()` 호출 부분을 추가로 표시했습니다. **Active Close** 쪽이 먼저 `close()`를 수행하고 `FIN`을 보내면 **Passive Close** 쪽은 `ACK`을 보낸 후 어플리케이션의 `close()`를 수행합니다. 보다 상세한 과정은 다음과 같습니다:

1. 먼저 `close()`를 실행한 클라이언트가 `FIN`을 보내고 `FIN_WAIT1` 상태로 대기한다.
2. 서버는 `CLOSE_WAIT`으로 바꾸고 응답 `ACK`를 전달한다. 그와 동시에 해당 포트에 연결되어 있는 어플리케이션에게 `close()`를 요청한다.
3. `ACK`를 받은 클라이언트는 상태를 `FIN_WAIT2`로 변경한다.
4. `close()` 요청을 받은 서버 어플리케이션은 종료 프로세스를 진행하고 `FIN`을 클라이언트에 보내 `LAST_ACK` 상태로 바꾼다.
5. `FIN`을 받은 클라이언트는 `ACK`를 서버에 다시 전송하고 `TIME_WAIT`으로 상태를 바꾼다. `TIME_WAIT`에서 일정 시간이 지나면 `CLOSED`된다. `ACK`를 받은 서버도 포트를 `CLOSED`로 닫는다.

한 가지 주의할 점은 클라이언트와 서버 대신 Active Close와 Passive Close라는 표현을 사용한 것인데, 반드시 서버만 `CLOSE_WAIT` 상태를 갖는 것은 아니기 때문입니다. 서버가 먼저 종료하겠다고 `FIN`을 보낼 수 있고, 이런 경우 서버가 `FIN_WAIT1` 상태가 됩니다. 따라서, 클라이언트와 서버가 아닌 **Active Close**(또는 Initiator, 기존 클라이언트)와 **Passive Close**(또는 Receiver, 기존 서버)정도로 표현하는 것이 정확합니다.

### `CLOSE_WAIT` 재현

`CLOSE_WAIT`을 아래와 같이 Java 코드로 재현 했습니다.[^1] 아래 예제에서도 **서버가 먼저 `FIN`을 보내 클라이언트가 `CLOSE_WAIT` 상태에 빠지는 것**을 보여줍니다.

* `Server.java` (sends some data and exits)
```java
package com.company;

import java.io.IOException;
import java.io.OutputStream;
import java.net.ServerSocket;
import java.net.Socket;

public class Server {

    public static void main(String[] args) {
        try {
            Socket socket = new ServerSocket(12345).accept();
            OutputStream out = socket.getOutputStream();

            out.write("Hello World\n".getBytes());
            out.flush();
            out.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

* `Client.java` (will sleep in `CLOSE_WAIT`)
```java
package com.company;

import java.io.InputStream;
import java.net.InetAddress;
import java.net.Socket;

public class Client {
    public static void main(String[] args) throws Exception {
        Socket socket = new Socket(InetAddress.getByName("localhost"), 12345);
        InputStream in = socket.getInputStream();
        int c;
        while ((c = in.read()) != -1) {
            System.out.write(c);
        }
        System.out.flush();

        // should now be in CLOSE_WAIT
        Thread.sleep(Integer.MAX_VALUE);
    }
}
```

각각 양쪽의 서버/클라이언트를 실행하면 클라이언트가 `CLOSE_WAIT` 상태에 빠지고, 행업되어 있는 동안에는 이 상태가 사라지지 않습니다.

아래는 위의 예제를 맥에서 실행하고, `netstat`를 실행한 모습이다. 참고로 맥(BSD 계열)과 리눅스의 `netstat` 옵션은 많이 다릅니다.

```console
$ netstat -p tcp -a -n | grep CLOSE_WAIT | grep 12345
```

![CLOSE_WAIT 상태에 빠진 클라이언트](https://farm9.staticflickr.com/8822/17428044644_962ce32d71_b.jpg)

10분이 지나도 `CLOSE_WAIT` 상태가 계속 변하지 않음을 확인할 수 있습니다.

Passive Close 측이 `CLOSE_WAIT` 상태에 빠지면 Active Close 측은 `FIN`을 못 받는 상태이기 때문에 `FIN_WAIT2`에서 마찬가지로 대기하게 됩니다. 그러나 `CLOSE_WAIT`와 달리 `FIN_WAIT2`는  일정 시간이 경과하면 `TIME_WAIT` 상태가 됩니다.

```console
$ netstat -ton
tcp        0      0 10.41.249.26:8080           10.51.31.149:40484          FIN_WAIT2  timewait (28.29/0/0)
```

뒤에서 설명하겠지만, `FIN_WAIT2`는 `net.ipv4.tcp_fin_timeout`을 설정해서 변경할 수 있으며 CentOS의 경우에 `60초`로 설정되어 있습니다. `TIME_WAIT`는 2*MSL(Maximum Segment Lifetime)으로 `60초`로 고정되어 있으며 변경할 수 없습니다.

### `CLOSE_WAIT` 종료

커널 옵션으로 타임아웃 조절이 가능한 `FIN_WAIT`이나 재사용이 가능한 `TIME_WAIT`과 달리, `CLOSE_WAIT`는 **포트를 잡고 있는 프로세스의 종료 또는 네트워크 재시작** 외에는 제거할 방법이 없습니다. 즉, **로컬 어플리케이션이 정상적으로 `close()`를 요청하는 것이 가장 좋은 방법**입니다.

> You can’t (and shouldn’t). CLOSE_WAIT is a state defined by TCP for connections being closed waiting for the counterpart to acknowledge this.[^2]
> No, there is no timeout for CLOSE_WAIT. I think that’s what the off means in your output. To get out of CLOSE_WAIT, the application has to close the socket explicitly (or exit).[^3]
> Since there is no CLOSE_WAIT timeout, a connection can stay in this state forever (or at least until the program does eventually close the connection or the process exists or is killed).
> If you cannot fix the application or have it fixed, the solution is to kill the process holding the connection open.[^4]

저마다 강조하는 바를 살펴봐도 `CLOSE_WAIT`는 커널 옵션이나 설정으로 타임아웃을 줄 수 없으며 로컬 어플리케이션의 문제이기 때문에 정상적인 문제 해결이 필요하다는 지적을 하고 있습니다.

### `CLOSE_WAIT` 원인

그렇다면 해당 서버의 `CLOSE_WAIT` 상태 원인은 무엇일까요? 문제가 된 것은 자바로 만든 웹 서비스이므로 행업 상태인 해당 JVM의 Thread Dumps를 분석했습니다.(참고: [Java Thread Dump Analyzer](https://github.com/kakao/java_thread_dump_analyzer)를 사용하면 좀 더 쉽게 쓰레드 덤프를 분석할 수 있습니다.)

```console
$ jstack 27836
"http-apr-8080-exec-24" daemon prio=10 tid=0x00007f9240054800 nid=0x6cc1 waiting on condition [0x00007f9271964000]
   java.lang.Thread.State: WAITING (parking)
    at sun.misc.Unsafe.park(Native Method)
    - parking to wait for  <0x0000000797283fc8> (a java.util.concurrent.FutureTask)
    at java.util.concurrent.locks.LockSupport.park(LockSupport.java:186)
    at java.util.concurrent.FutureTask.awaitDone(FutureTask.java:425)
    at java.util.concurrent.FutureTask.get(FutureTask.java:187)
    at java.util.concurrent.AbstractExecutorService.invokeAll(AbstractExecutorService.java:243)
    at net.daum.crystal.core.execute.SearchManager.search(SearchManager.java:104)
    at net.daum.crystal.core.execute.SearchManager.search(SearchManager.java:163)
...
```

그 결과 대부분의 쓰레드가 검색 결과를 받아오지 못하고 대기중(WAITING)인 것을 확인했습니다. 그런데 재밌게도 이 검색 결과는 로컬에서 받아오는 것입니다. 즉, 로컬은 행업 상태여서 검색 결과를 보내주지 못하는 상황이고, 쓰레드는 검색 결과를 받기 위해 대기하는 상황입니다.

원래 일정 시간이 경과하면 타임아웃으로 끊어야 하는데, 행업 상태에 빠지다보니 이 조차도 처리되지 않고 서로가 서로를 기다리는 상황인 거죠.

즉, 보낼 수도 없고 받을 수도 없는 일종의 **교착 상태(deadlock)**가 원인으로 지목됐습니다. 아울러 이 상태는 성능 테스트가 끝나도 정상으로 복구되지 않았습니다.

이 부분을 좀 더 구체적으로 설명하면,

* 파일서버가 필요해 별도 웹 서버를 구성하지 않고 톰캣의 `/ROOT`를 이용
* Static HTML 파일을 올려두고 톰캣에서 구동중인 웹앱의 `HttpClient`에서 로컬 자기 자신(동일한 톰캣)을 호출해 HTML을 받아가도록 구성
* 그러나, 부하를 높이니 점점 느려지다 HTML 조차 내려주지 못하는 행업 상태 발생
* 모든 소켓이 `CLOSE_WAIT` 상태에 빠짐

### `CLOSE_WAIT` 해결

요청과 응답을 받는 과정에서 recursive 한 호출이 교착 상태의 원인이었으며 별도 서버를 구성하여 상호 의존성 없이 호출 가능하도록 구성했다. 톰캣을 통한 동일한 WAS가 아닌, 별도의 nginx를 구성하고 다른 프로세스에서 HTML 파일을 내려주도록 처리해 문제를 해결했습니다.

테스트 결과, 더 이상 문제가 발생하지 않음을 확인했습니다.

## Part II. `TIME_WAIT`

`TIME_WAIT` 상태가 늘어나면 서버의 소켓이 고갈되어 커넥션 타임아웃이 발생한다는 얘기를 종종 듣습니다. 이 말이 올바른 얘기인지, `TIME_WAIT`은 어떠한 경우에 발생하고 어떤 특징이 있는지 살펴보겠습니다.

### `TIME_WAIT`는 왜 어려운가?

`TIME_WAIT` 이란 TCP 상태의 가장 마지막 단계이며, 앞에서 살펴보았습니다. Active Close 즉, 먼저 `close()`를 요청한 곳에서 최종적으로 남게 되며, 2*MSL(Maximum Segment Lifetime)동안 유지됩니다.

이 단순한 과정이 매번 어렵게 느껴지는 이유는 대부분은 고급언어로 소켓을 랩핑해서 사용하기 때문에(앞에서 예제로 제시한 Java 코드[^1]도 `accept()` 이전 모든 과정이 라이브러리로 랩핑되어 있음) 소켓에 문제가 생기지 않는 한 로우 레벨로 내려가 확인할 일이 흔치 않고, 또한 확인하는 방법을 아는 이도 드물기 때문일 겁니다.

구현 및 재현에 나름의 고급 기술이 필요하다보니 거의 대부분은 실제로 검증 과정을 거치지 못하고 문서로만 익히게 되는데, 그러다 보니 잘못된 정보, 오래된 정보로 더 이상 유효하지 않은 내용들이 무분별하게 전제됩니다.

제대로 된 검증 과정도 거치지 않은 채, 또는 검증할 능력이 부족한 상태에서 계속 인용되면서 잘못된 정보가 지속적으로 확대 재생산됩니다. 심지어 스택오버플로우에도 절반 이상은 잘못된 정보입니다. 그나마 해외에는 제대로 된 문서가 일부 있지만 우리말로 된 문서 중에는 100% 정확한 문서가 전혀 없다고 봐도 틀리지 않습니다.

잘못된 정보를 접한 이들은 서버 동작과 일치하지 않으니 계속 이해를 못하게 되고 점점 더 어렵게 느껴집니다. 그야말로 "진퇴양난"이죠.

현재 시점에서 인터넷에 있는 가장 정확한 문서는 Vincent Bernat 가 작성한 [Coping with the TCP TIME-WAIT state on busy Linux servers](http://vincent.bernat.im/en/blog/2014-tcp-time-wait-state-linux.html) 입니다. 이외 대부분의 문서는 잘못된 내용을 담고 있는 경우가 대부분이므로 주의가 필요합니다.

참고로 이 문서는 리눅스 커널 4.1 커널 소스를 직접 파악하여 리눅스의 TCP 동작을 정리한 내용입니다. TCP 는 몇년새 큰 변화가 없기 때문에 3.x 이상은 대부분 동일하지만, BSD나 윈도우의 동작 방식과는 다를 수 있으므로 유의하시기 바랍니다.

### `TIME_WAIT`는 왜 필요할까?

`TIME_WAIT` 상태가 왜 필요하고, 왜 그렇게 길게 설정되어 있는지 이유를 살펴보도록 하겠습니다. 만일 `TIME_WAIT`이 짧다면 아래와 같은 두 가지 문제[^12]가 발생합니다.

첫 번째는 **지연 패킷이 발생할 경우**입니다.

![지연 패킷이 발생할 경우](http://d1g3mdmxf8zbo9.cloudfront.net/images/tcp/duplicate-segment.png)

이미 다른 연결로 진행되었다면 지연 패킷이 뒤늦게 도달해 문제가 발생합니다. 매우 드문 경우이긴 하나 때마침 `SEQ`까지 동일하다면 잘못된 데이타를 처리하게 되고 데이타 무결성 문제가 발생합니다.

두 번째는 **원격 종단의 연결이 닫혔는지 확인해야 할 경우**입니다.

![원격 종단의 연결이 닫혔는지 확인해야 할 경우](http://d1g3mdmxf8zbo9.cloudfront.net/images/tcp/last-ack.png)

마지막 `ACK` 유실시 상대방은 `LAST_ACK` 상태에 빠지게 되고 새로운 `SYN` 패킷 전달시 `RST`를 리턴합니다. 새로운 연결은 오류를 내며 실패합니다. 이미 연결을 시도한 상태이기 때문에 상대방에게 접속 오류 메시지가 출력될 것입니다.

따라서 반드시 `TIME_WAIT`이 일정 시간 남아 있어서 패킷의 오동작을 막아야 합니다.

[RFC 793](https://tools.ietf.org/html/rfc783) 에는 `TIME_WAIT`을 **2 MSL**로 규정했으며 CentOS 6에서는 **60초** 동안 유지됩니다. 아울러 이 값은 조정할 수 없습니다.

> **틀린 정보**: `net.ipv4.tcp_fin_timeout` 을 설정하면 `TIME_WAIT` 타임아웃을 변경할 수 있다.

`TIME_WAIT`의 타임아웃 정보는 커널 헤더 `include/net/tcp.h` 에 하드 코딩 되어 있으며 변경이 불가능합니다.

* `include/net/tcp.h`
```c
#define TCP_TIMEWAIT_LEN (60*HZ) /* how long to wait to destroy TIME-WAIT
                                  * state, about 60 seconds     */
```

### 예제 TCP 서버

이번에는 소켓의 로우 레벨까지 확인하기 위해 예제 서버 프로그램을 C 로 구현해보겠습니다. 리눅스를 포함한 모든 유닉스 기반 OS 의 API 가 C 로 구현되어 있고 특히 네트워크 프로그램에서 커널의 동작과 C 의 어플리케이션 API 는 정확히 1:1 로 대응됩니다. 따라서 구체적으로 네트워크가 어떻게 동작하는 지를 C 로 직접 구현해 하나씩 확인해보겠습니다.

먼저, 서버 프로그램은 예전에 커넥션 테스트 용도로 개발해 [깃헙에 공개한 CONTEST 서버](https://github.com/likejazz/contest-server)를 기반으로 일부 코드를 추가해서 구현했습니다.

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <time.h>
#include <pthread.h>
#include <net/if.h>
#include <sys/ioctl.h>

#define BACKLOG       128     // backlog for listen()
#define DEFAULT_PORT  5000    // default listening port
#define BUF_SIZE      1024    // message chunk size

time_t ticks;

typedef struct {
  int client_sock;
} thread_param_t;
thread_param_t *params;       // params structure for pthread

pthread_t thread_id;          // thread id

void error(char *msg) {
  perror(msg);
  exit(EXIT_FAILURE);
}

/**
* thread handler after accept()
*/
void *handle_client(void *params) {
  char msg[BUF_SIZE];
  thread_param_t *p = (thread_param_t *) params;  // thread params

  // clear message buffer
  memset(msg, 0, sizeof(msg));

  ticks = time(NULL);
  snprintf(msg, sizeof(msg), "%.24s\r\n", ctime(&ticks));
  write(p->client_sock, msg, strlen(msg));
  printf("Sent message to Client #%d\n", p->client_sock);

  // wait
  usleep(10);

  // clear message buffer
  memset(msg, 0, sizeof(msg));

  // active close, It will remains in `TIME_WAIT` state.
  if (close(p->client_sock) < 0)
    error("Error: close()");
  free(p);

  return NULL;
}

int main(int argc, char *argv[]) {
  int listenfd = 0, connfd = 0;
  struct sockaddr_in serv_addr;
  struct ifreq ifr;

  // build server's internet addr and initialize
  memset(&serv_addr, '0', sizeof(serv_addr));

  serv_addr.sin_family = AF_INET;                // an internet addr
  serv_addr.sin_addr.s_addr = htonl(INADDR_ANY); // accept any interfaces
  serv_addr.sin_port = htons(DEFAULT_PORT);      // the port we will listen on

  // create the socket, bind, listen
  if ((listenfd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
    error("Error: socket()");
  if (bind(listenfd, (struct sockaddr *) &serv_addr, sizeof(serv_addr)) < 0)
    error("Error: bind() Not enough privilleges(<1024) or already in use");
  if (listen(listenfd, BACKLOG) < 0)
    error("Error: listen()");

  ifr.ifr_addr.sa_family = AF_INET;               // an internet addr
  strncpy(ifr.ifr_name, "eth0", IFNAMSIZ-1);      // IP address attached to "eth0"
  ioctl(listenfd, SIOCGIFADDR, &ifr);             // get ifnet address

  // print <IP>:<Port>
  printf("Listening on %s:%d\n",
         inet_ntoa(((struct sockaddr_in *)&ifr.ifr_addr)->sin_addr),
         DEFAULT_PORT);

  while (1) {
    connfd = accept(listenfd, (struct sockaddr *) NULL, NULL);

    // memory allocation for `thread_param_t` struct
    params = malloc(sizeof(thread_param_t));
    params->client_sock = connfd;

    // 1 client, 1 thread
    pthread_create(&thread_id, NULL, handle_client, (void *) params);
    pthread_detach(thread_id);
  }
}
```

Java에 비해 코드는 훨씬 더 길지만 동작 방식에는 큰 차이가 없습니다. **의도적으로** 서버가 먼저 `close()`를 시도하는 점도 동일합니다. 즉, 서버측이 Active Close가 되고 `TIME_WAIT` 상태에 빠지게되는거죠.

### TCP 서버 구현 방식

서버의 동작은 `socket()` - `bind()` - `listen()` - `accept()` 과정을 거쳐 클라이언트와 연결되며 위 코드에는 그 과정이 상세히 잘 나와 있습니다.

`accept()` 이후 서버는 `listen()` 중인 소켓과 별도로 `accept()` 소켓을 추가로 생성해 클라이언트를 할당합니다. 서버가 클라이언트를 관리하는 방식은 크게 4가지로 구분할 수 있습니다.[^13]

1. 요청 당 프로세스 할당, `fork()`로 자식 프로세스를 만들어 클라이언트를 담당합니다. 전통적인 **blocking I/O** 방식입니다.
2. 요청 당 쓰레드 할당. 위 예제에서 사용한 방식으로 `pthread_create()`를 통해 쓰레드를 생성, 클라이언트를 담당합니다. 마찬가지로 **blocking I/O** 방식입니다.
3. 쓰레드풀을 구성해 각각의 쓰레드가 여러 커넥션을 **asynchronous I/O** 방식으로 담당합니다.
4. 쓰레드풀을 구성해 각각의 쓰레드가 여러 커넥션을 `select()`, `poll()`, **nonblocking I/O** 같은 이벤트 기반 방식으로 담당합니다.

이 중 대용량 처리에 3, 4번이 우세하며 특히 4번이 대세입니다. 하지만, 이 글에서 모두 언급하기엔 지나치게 방대하므로 추후 별도로 정리해보기로 하고, 자세한 사항은 [C10K Problem](http://www.kegel.com/c10k.html)을 참고하시기 바랍니다.

이 글에서는 가장 간편한 방식인 2번, 클라이언트를 각각의 쓰레드에 할당하고 `close()` 될 때 쓰레드도 함께 종료되는 방식으로 구현했습니다.

![htop을 통해 확인한 서버 쓰레드](https://farm1.staticflickr.com/267/17905712974_17c2eaff1b_b.jpg)

`htop`을 이용해 쓰레드 옵션을 켜고(`H` 키), 트리 모드(`t` 키)에서 요청이 있을 때마다 쓰레드가 하나씩 생성되는 모습을 직접 확인한 화면입니다.

현재 2개의 요청을 처리 중이며, 물론 요청이 끝나면 쓰레드도 함께 종료됩니다. 만약 프로세스나 쓰레드를 할당하지 않았다면 요청이 끝날 때까지 다른 요청은 받지 못하는 말 그대로 *blocking* 상태가 유지됩니다.

### TCP 연결 종료 과정

샘플 서버를 구동하고 클라이언트에서 `FIN/ACK`이 잘 전달되는지 확인합니다. 아울러 서버의 Active Close 후 서버측에 남게되는 `TIME_WAIT` 상태를 직접 확인합니다.

[TCP/IP Illustrated](http://www.yes24.com/24/goods/8234905)의 TCP 연결 종료 다이어그램은 아래와 같습니다:

![TCP 연결 종료 다이어그램](https://farm9.staticflickr.com/8896/18339898259_71c350b396_b.jpg)

연결 종료의 4-way handshake 과정은 다음과 같습니다:

* `FIN+ACK`, seq = K, ack = L
* `ACK`, seq = L, ack = K + 1
* `FIN+ACK`, seq = L, ack = K + 1
* `ACK`, seq = K, ack = L + 1

서버 접속 후 `tcpdump` 로 패킷 상태를 덤프한 결과를 보면:

```console
$ tcpdump -nn -i eth0 host 10.xx.xx.214
13:32:31.626549 IP 10.yy.yy.237.5000 > 10.xx.xx.214.30202: Flags [F.], seq 27, ack 1, win 16, options [nop,nop,TS val 2599042792 ecr 2329783108], length 0
13:32:31.626877 IP 10.xx.xx.214.30202 > 10.yy.yy.237.5000: Flags [.], ack 27, win 6, options [nop,nop,TS val 2329783108 ecr 2599042792], length 0
13:32:31.626964 IP 10.xx.xx.214.30202 > 10.yy.yy.237.5000: Flags [F.], seq 1, ack 28, win 6, options [nop,nop,TS val 2329783108 ecr 2599042792], length 0
13:32:31.626975 IP 10.yyy.yy.237.5000 > 10.xx.xx.214.30202: Flags [.], ack 2, win 16, options [nop,nop,TS val 2599042792 ecr 2329783108], length 0
```

포트 5000번이 서버입니다. 서버가 먼저 `FIN`을 보내며 Active Close를 시도했습니다. `tcpdump` 의 결과 Flags, 패킷 타입 플래그는 다음과 같습니다.

* `[S]` - SYN (Start Connection)
* `[.]` - No Flag Set
* `[P]` - PSH (Push Data)
* `[F]` - FIN (Finish Connection)
* `[R]` - RST (Reset Connection)

`[.]`은 `ACK`를 뜻하며 `[F.]` 은 FIN+ACK 을 가리키는 싱글 패킷입니다. 이에 따라 실제 덤프 결과를 분석해보면,

* `FIN+ACK`, seq = 27, ack = 1
* `ACK`, seq = 1, ack = 27
* `FIN+ACK`, seq = 1, ack = 28
* `ACK`, seq = 28, ack = 2

verbose 모드가 아니었기 때문에 `ACK`의 seq는 보이지 않습니다. 추가로 `-vv` 옵션을 부여하면 verbose 모드가 되며 확인할 수 있습니다.

그런데 CentOS 6에서는 결과 중 두 번째 `ACK`의 ack 값과 마지막 `ACK`의 seq 값이 TCP/IP Illustrated 에서 명시된 숫자와 하나씩 다릅니다. OS 별로 커널 버전 별로 조금씩 다르게 동작하기도 하는데 이 부분은 추후에 보다 정확한 확인이 필요할 것 같습니다.

이렇게 종료 handshake 과정이 정상적으로 끝나면 Active Close를 먼저 요청한 서버 쪽에 `TIME_WAIT`이 남게 됩니다.

![TIME_WAIT 상태의 서버](https://farm9.staticflickr.com/8858/18532247925_9d15cf4ff6_b.jpg)

처리한 쓰레드도 이미 종료된 상태로 할당된 프로세스도 보이지 않습니다. 이미 커널로 소유권이 넘어 갔으며 프로세스를 종료해도 `TIME_WAIT` 상태는 사라지지 않습니다. 오히려 소켓이 해당 포트를 점유하고 있는 상태로 60초 동안 재시작을 할 수 없게 됩니다.

재시작을 시도하면 `bind()` 단계에서 오류가 발생하며 시간이 지나 `TIME_WAIT`이 모두 사라진 후에야 가능합니다.

```console
$ ./server
ERROR: Address already in use
```

### 소켓 옵션

만일 즉시 재시작이 필요하다면 `setsockopt()`의 `SO_REUSEADDR` 옵션을 적용하면 `bind()` 단계에서 커널이 가져간 소유권을 다시 돌려받으며 즉시 재시작 가능합니다.

```c
setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes));
```

> **틀린 정보**: `SO_REUSEADDR` 와 `tcp_tw_reuse` 는 동일하게 적용되는 옵션이다.

* `net/core/sock.c`
```c
        case SO_REUSEADDR:
                sk->sk_reuse = (valbool ? SK_CAN_REUSE : SK_NO_REUSE);
                break;
```

커널 코드에서 `SO_REUSEADDR`은 `socket` 구조체 `sk_reuse`를 `1`로 설정하므로 `tcp_tw_reuse`를 설정하는 것과 동일한 역할을 합니다. 그러나 여기에는 결정적인 차이가 있는데, 전자는 커널(서버)의 역할이고 후자는 glibc(클라이언트)의 역할로 서로 다르다는 점이죠.

* `SO_REUSEADDR`: **서버 소켓**에 적용되는 옵션
* `tcp_tw_reuse`: **클라이언트 소켓**에 적용되는 옵션

클라이언트 소켓에 `SO_REUSEADDR`을 부여한다고 `TIME_WAIT`을 재사용할 수 있는게 아닙니다. `tcp_tw_reuse` 커널 설정이 필요합니다.

반대로 서버 소켓에서 `bind()`시 `tcp_tw_reuse` 설정을 했다고 바인딩 되지 않습니다. `SO_REUSEADDR` 옵션을 부여해야 합니다.

이후에 다시 자세히 설명하도록 하겠습니다.

### 쌍방

양쪽 모두 `TIME_WAIT`이 남는 재미있는(?) 현상도 발생할 수 있습니다. 양쪽 모두 동일 시점에 Active Close를 시도할 경우입니다.

```console
$ tcpdump -nn -i eth0 host 10.xx.xx.141
14:37:45.682624 IP 10.xx.xx.141.1063 > 10.yy.yy.140.5000: Flags [S], seq 50755794, win 14600, options [mss 1460,sackOK,TS val 382514872 ecr 0,nop,wscale 10], length 0
...
14:37:45.683122 IP 10.xx.xx.141.1063 > 10.yy.yy.140.5000: Flags [F.], seq 1, ack 27, win 15, options [nop,nop,TS val 382514873 ecr 382506868], length 0
14:37:45.683125 IP 10.yy.yy.140.5000 > 10.xx.xx.141.1063: Flags [F.], seq 27, ack 1, win 15, options [nop,nop,TS val 382506868 ecr 382514873], length 0
14:37:45.683133 IP 10.xx.xx.140.5000 > 10.xx.xx.141.1063: Flags [.], ack 2, win 15, options [nop,nop,TS val 382506868 ecr 382514873], length 0
14:37:45.683160 IP 10.xx.xx.141.1063 > 10.yy.yy.140.5000: Flags [.], ack 28, win 15, options [nop,nop,TS val 382514873 ecr 382506868], length 0
```

여기엔 특이한 경우가 두 번이나 발생했는데, 먼저 최초 `SYN` 이후 패킷 전달이 잘 끝나고 어플리케이션 종료 시점에 양쪽 모두 거의 동일한 시간에 `FIN+ACK`을 전달하며 Active Close를 시도했습니다.

보통 이런 경우 한쪽에서만 `ACK`를 보내고 송신한 쪽이 `TIME_WAIT` 상태로 끝나게 되는데 공교롭게도 `ACK` 또한 양쪽 모두 동시에 전달을 시도했습니다.

매우 특이한 경우지만 드물게 발생할 수 있는 경우입니다. 그리고 이런 경우 양쪽 모두 `TIME_WAIT` 상태에 빠지게 됩니다.

* Server
```console
$ ss -tanop | grep 1063
TIME-WAIT  0  0  10.yy.yy.140:5000  10.xx.xx.141:1063  timer:(timewait,45sec,0)
```

* Client
```console
$ ss -taonp | grep 1063
TIME-WAIT  0  0  10.xx.xx.141:1063  10.yy.yy.140:5000  timer:(timewait,42sec,0)
```

### 자원 할당

그렇다면, 다수의 `TIME_WAIT`은 과연 시스템 성능 저하를 가져올까요?

> Each socket in TIME_WAIT consumes some memory in the kernel, usually somewhat less than an ESTABLISHED socket yet still significant. A sufficiently large number could exhaust kernel memory, or at least degrade performance because that memory could be used for other purposes.[^14]
> Because application protocols do not take TIME-WAIT TCB distribution into account, heavily loaded servers can have thousands of connections in TIME-WAIT that consume memory and can slow active connections. In BSD-based TCP implementations, TCBs are kept in mbufs, the memory allocation unit of the networking subsystem[9]. There are a finite number of mbufs available in the system, and mbufs consumed by TCBs cannot be used for other purposes such as moving data. Some systems on high speed networks can run out of mbufs due to TIME-WAIT buildup under high connection load. A SPARCStation 20/71 under SunOS 4.1.3 on a 640 Mb/s Myrinet[10] cannot support more than 60 connections/sec because of this limit.[^15]

스택오버플로우의 답변[^14]이나 그렇다는 논문[^15]이 있는데 특히 메모리 점유 문제를 심각하게 얘기합니다. 여기서 주의해야 할 점은 `TIME_WAIT`으로 인한 성능 저하 논문은 1997년에 출판됐다는 점이고, 지금은 2015년이라는 점입니다. 벌써 18년전 이야기죠. 그 당시 서버의 메모리는 512MB였고 지금 테스트를 진행하는 서버의 메모리는 64G입니다. 100배가 넘죠. 그런데 아직도 `TIME_WAIT` 때문에 메모리 점유가 심각하다고 말한다면 뭔가 이상하지 않나요?

`struct tcp_timewait_sock`는 고작 168바이트에 불과한데요.[^12]

```c
struct tcp_timewait_sock {
    struct inet_timewait_sock tw_sk;
    u32    tw_rcv_nxt;
    u32    tw_snd_nxt;
    u32    tw_rcv_wnd;
    u32    tw_ts_offset;
    u32    tw_ts_recent;
    long   tw_ts_recent_stamp;
};
```

만약 4만 개의 `TIME_WAIT` 상태 인바운드 커넥션이 있다면 10MB 안쪽의 메모리를 차지하겠죠. 아웃바운드 커넥션도 고작 2.5MB가 더 필요할 뿐 입니다.[^12] 요즘 서버 사양에서 `TIME_WAIT` 상태가 차지하는 메모리 용량은 아주 미미합니다.

현재 서버의 소켓 상태는 아래 명령어로 확인할 수 있습니다.

```console
$ cat /proc/net/sockstat
sockets: used 76
TCP: inuse 17 orphan 0 tw 0 alloc 17 mem 1
...
```

그렇다면 `TIME_WAIT` 상태가 증가하면 어떤 일이 발생할까요? 실제 부하 테스트를 통해 이를 확인해보기로 했습니다.

![대량의 TIME_WAIT 상태 만들기](https://farm9.staticflickr.com/8891/18532873405_6cf6227ca9_b.jpg)

여러 대의 클라이언트를 동원해 연속된 요청으로 9만개 가까운 `TIME_WAIT` 상태를 만들어 냈습니다.

```console
$ ss -ant | awk '{print $1}' | grep -v '[a-z]' | sort | uniq -c
      1 ESTAB
     15 LISTEN
  89262 TIME-WAIT
```

> **틀린 정보**: 서버의 소켓 수는 할당 가능한 로컬 포트 만큼인 최대 65,535개이고 `net.ipv4.ip_local_port_range` 설정으로 변경할 수 있다.

서버는 로컬 포트를 사용하지 않습니다.

만일 **많은 사람들이 오해**하고 있는 것 처럼, 서버가 로컬 포트를 사용하고 로컬 포트는 단 하나의 소켓에만 바인딩된다고 가정하면,

1. 지금 처럼 9만개 가까운 `TIME_WAIT`을 만들어 낼 수 없습니다. 로컬 포트는 (2^16)-1 = 65,535개가 최대겠죠.
2. OS 몰래 비밀 포트를 여는 백도어가 존재하는게 아니라면 `tcpdump`에 보이지 않을 리가 없죠. 최초 바인딩된 포트만 사용해 패킷을 주고 받는걸 확인할 수 있습니다.

서버가 할당하는 것은 포트가 아닌 **소켓**이며 서버의 포트는 최초 `bind()`시 하나만 사용합니다. 로컬 포트를 할당하는 것은 클라이언트이며, 클라이언트가 connect()시 로컬 포트를 임의로(ephemeral port) 바인딩하면서 서버의 소켓과 연결됩니다.

소켓은 `<protocol>`, `<src addr>`, `<src port>`, `<dest addr>`, `<dest port>` 이 5개 값이 유니크하게 구성됩니다. 따라서 서버 포트가 추가되거나 클라이언트의 IP가 추가될 경우 그 만큼의 새로운 쌍을 생성할 수 있어 `TIME_WAIT`가 많이 남아 있어도 별 문제가 없습니다.

소켓의 수는 설정된 리눅스 파일 디스크립터만큼 생성할 수 있습니다. 아래 설정으로 서버가 이론적으로는 26만개가 넘는 클라이언트도 받을 수 있는 거죠.

```console
$ sysctl fs.file-max
fs.file-max = 262144
```

서버가 또 다른 서버에 클라이언트로 접속하지만 않는다면 자신의 로컬 포트는 사용할 일이 없으며, 리눅스의 로컬 포트 범위는 3만개 정도로 설정되어 있습니다. 마찬가지로 이론적으로는 3만개의 서버에 동시 접속이 가능한 거죠.

```console
$ sysctl net.ipv4.ip_local_port_range
net.ipv4.ip_local_port_range = 32768  61000
```

만일 서버 투 서버로 1:1 대용량 접속이 발생할 경우 한 대의 클라이언트에서 가능한 최대 요청 수는 500 RPS(Requests Per Second) 정도입니다. 500 * 60 (TIME_WAIT 시간) = 3만개 이기 때문이죠. 이 수치를 넘어서지 않는다면 아무런 커널 설정도 변경할 필요가 없으며, 부하 테스트 등 특수한 용도여야 이 수치를 넘어설 수 있을 겁니다.

### 포트 재사용

그러나 이 수치를 넘어선다면 클라이언트의 로컬 포트가 고갈 될 것이며 `TIME_WAIT` 상태를 재사용 해야 합니다. 아래 3가지 경우로 분류할 수 있습니다:

1. 서버에 `TIME_WAIT` 상태가 남아 있으며, 클라이언트의 로컬 포트가 고갈된 경우
2. 클라이언트에 `TIME_WAIT` 상태가 남아 있으며, 클라이언트의 로컬 포트가 고갈된 경우
3. 클라이언트에 `TIME_WAIT` 상태가 남아 있으며, 클라이언트의 로컬 포트가 고갈되고, 서버의 다른 포트에 접속할 경우

1번의 경우 클라이언트 입장에서는 서버에 남아 있는 `TIME_WAIT` 상태를 알 수 없습니다. 따라서 클라이언트는 계속해서 임의의 포트(ephemeral port)에 `SYN` 패킷을 내보냅니다. 임의의 포트는 순차 증가하는 형태이므로 FIFO 기준을 따르게 됩니다. 서버에는 `TIME_WAIT` 상태로 남아 있지만 동일 소켓이 `SYN`을 수신하면 재사용하게 됩니다. 양쪽 모두 별도 설정은 필요 없습니다다.

* `net/ipv4/tcp_minisocks.c`
```c
/*
 *      RFC 1122:
 *      "When a connection is [...] on TIME-WAIT state [...]
 *      [a TCP] MAY accept a new SYN from the remote TCP to
 *      reopen the connection directly.
 */
```

2번은 오류가 발생하며 더 이상 접속할 수 없게 됩니다.

```console
$ telnet 10.xx.xx.88 5000
Trying 10.xx.xx.88...
telnet: connect to address 10.xx.xx.88: Cannot assign requested address
telnet: Unable to connect to remote host: Cannot assign requested address
```

소켓은 `<protocol>`, `<src addr>`, `<src port>`, `<dest addr>`, `<dest port>` 5개 값으로 구성되며 로컬 포트가 고갈되면 더 이상 유니크한 값을 만들어 낼 수 없게 됩니다. 클라이언트의 `net.ipv4.tcp_tw_reuse` 옵션을 설정하여 기존 클라이언트에 `TIME_WAIT` 상태로 남아 있던 소켓을 재사용해야 합니다.

```console
$ echo 1 > /proc/sys/net/ipv4/tcp_tw_reuse
$ sysctl net.ipv4.tcp_tw_reuse
net.ipv4.tcp_tw_reuse = 1
```

3번은 문제가 없습니다. 앞서 언급했듯 소켓은 5개 유니크 값이며 맨 마지막이 `<dest port>` 입니다. 이 말은 포트가 다를 경우 다시 그만큼의 새로운 소켓 쌍을 만들어낼 수 있다는 의미죠. 재사용이 필요 없습니다.

```console
$ ss -tonpa | grep 1033
TIME-WAIT  0      0              10.xx.xx.141:1033          10.yy.yy.140:5000   timer:(timewait,52sec,0)
ESTAB      0      0              10.xx.xx.141:1033          10.yy.yy.140:22     users:(("telnet",29636,3))
```

위 상태는 실제 클라이언트 환경에서 동일한 로컬 포트에 하나는 `TIME_WAIT`, 하나는 `ESTABLISHED` 상태인 모습입니다. 상대방 포트는 다르기 때문에 이렇게 동일한 로컬 포트를 함께 쓰는 것도 가능합니다. 같은 원리로 서버도 하나의 포트에 여러 개의 소켓이 할당됩니다.

### 바인드(bind)

그렇다면 리눅스 커널에서 로컬 포트가 어떤 알고리즘으로 바인드(`bind()`) 되는지 좀 더 자세히 살펴보겠습니다. 클라이언트가 패킷을 전송할때 아직 할당된 로컬 포트가 없다면 아래와 같이 오토 바인드를 진행합니다.

* `net/ipv4/af_inet.c`
```c
int inet_sendmsg(struct socket *sock, struct msghdr *msg, size_t size)
{
...
        /* We may need to bind the socket. */
        if (!inet_sk(sk)->inet_num && !sk->sk_prot->no_autobind &&
            inet_autobind(sk))
                return -EAGAIN;

        return sk->sk_prot->sendmsg(sk, msg, size);
}
```

`inet_autobind()`는 가능한 포트를 소켓 구조체 멤버 함수인 `get_port()`를 통해 찾는군요.

* `net/ipv4/af_inet.c`
```c
static int inet_autobind(struct sock *sk)
{
        struct inet_sock *inet;
        /* We may need to bind the socket. */
        lock_sock(sk);
        inet = inet_sk(sk);
        if (!inet->inet_num) {
                if (sk->sk_prot->get_port(sk, 0)) {
                        release_sock(sk);
                        return -EAGAIN;
                }
                inet->inet_sport = htons(inet->inet_num);
        }
        release_sock(sk);
        return 0;
}
```

`get_port()`는 `inet_csk_get_port()`로 선언되어 있는데, 이 함수에는 두 번째 인자가 `0`일때 선택 가능한 로컬 포트를 자동으로 찾아내는 알고리즘이 구현되어 있습니다.

* `net/ipv4/inet_connection_sock.c`
```c
/* Obtain a reference to a local port for the given sock,
 * if snum is zero it means select any available local port.
 */
int inet_csk_get_port(struct sock *sk, unsigned short snum)
{
        struct inet_hashinfo *hashinfo = sk->sk_prot->h.hashinfo;
...
        local_bh_disable();
        if (!snum) {
                int remaining, rover, low, high;

again:
                inet_get_local_port_range(net, &low, &high);
                remaining = (high - low) + 1;
                smallest_rover = rover = prandom_u32() % remaining + low;

                smallest_size = -1;
                do {
                        if (inet_is_local_reserved_port(net, rover))
                                goto next_nolock;
                        head = &hashinfo->bhash[inet_bhashfn(net, rover,
                                        hashinfo->bhash_size)];
...
                next_nolock:
                        if (++rover > high)
                                rover = low;
                } while (--remaining > 0);
```

이 코드를 보면 먼저 `inet_get_local_port_range()` 함수에서 `net.ipv4.ip_local_port_range`로 선언된 가능한 포트 범위를 읽어들이고 이 중 랜덤으로 첫번째 값을 고릅니다. 그리고 해시 테이블의 정보와 비교하여 사용 가능한 상태인지 +1 씩 증가하면서 확인합니다. 마지막 값에 도달한 다음에는 다시 가장 낮은 값으로 돌아가 반복합니다. 이 과정을 통해 사용 가능한 포트를 찾아냅니다.

* `net/ipv4/inet_connection_sock.c`
```c
if (net_eq(ib_net(tb), net) && tb->port == rover) {
        if (((tb->fastreuse > 0 &&
              sk->sk_reuse &&
              sk->sk_state != TCP_LISTEN) ||
             (tb->fastreuseport > 0 &&
              sk->sk_reuseport &&
              uid_eq(tb->fastuid, uid))) &&
            (tb->num_owners < smallest_size || smallest_size == -1)) {
                smallest_size = tb->num_owners;
                smallest_rover = rover;
                if (atomic_read(&hashinfo->bsockets) > (high - low) + 1 &&
                    !inet_csk(sk)->icsk_af_ops->bind_conflict(sk, tb, false)) {
                        snum = smallest_rover;
                        goto tb_found;
                }
        }
...
```

그러나 `sk->sk_reuse` 즉, `net.ipv4.tcp_tw_reuse`가 선언되어 있고 타임스탬프가 더 크고, `TIME_WAIT`인 경우 혹시 현재 상태가 `TCP_LISTEN` 중일때가 아니라면 바로 재사용합니다. 따라서 반복하지 않고 바로 다음 포트를 사용하게 됩니다.

물론 반복한다고 해도 더 이상 성능저하가 발생하진 않습니다. 2008년 크리스마스 이브(...)에 Evgeniy Polyakov가 빈 포트를 찾기 위해 전체 바인드 해시 테이블을 뒤지는 문제(traverse the whole bind hash table to find out empty bucket)에 대한 패치[^16]를 제출했고 2009년 1월에 받아들여졌기 때문이죠.[^17]

그 전에는 바인딩 포트 수가 많아지면 오토 바인드가 점점 더 느려지는 성능 문제가 있었습니다. 특히 로컬 포트가 모두 고갈될 경우 성능 문제가 발생할 수 있었죠.

![오래된 커널에서 오토 바인드가 느려지는 현상](http://marc.info/?l=linux-netdev&m=123028028631172&q=p3)

우연찮게도 이 문제는 다른 테스트 도중 오래된 서버(RHEL 4)에서 직접 재현할 수 있었습니다.

위 1번의 경우였는데,

> 1. 서버에 `TIME_WAIT` 상태가 남아 있으며, 클라이언트의 로컬 포트가 고갈된 경우

원래는 서버/클라이언트 양쪽 모두 아무런 설정 없이 문제가 없어야 되는 상황입니다. 그런데 부하 테스트 중 `Cannot assign requested address` 오류가 발생했습니다.

```console
$ telnet 10.41.118.88 80
Trying 10.41.118.88...
telnet: connect to address 10.41.118.88: Cannot assign requested address
telnet: Unable to connect to remote host: Cannot assign requested address
```

포트 고갈로 발생하던 2번과 동일한 오류이며 시스템 전체가 과부하 상태에 빠졌습니다. 부하가 매우 높을 때만 간헐적으로 발생하는 걸로 봐서 위에서 언급한 *빈 포트를 찾기 위해 전체 바인드 해시 테이블을 뒤지는 문제* 비용이 높기 때문으로 보입니다. 이미 로컬 포트를 다 사용한 상태에서 매 번 스캔에 리소스를 허비하는 것 입니다.

이 경우 해결책은 2번과 마찬가지로 클라이언트에서 `net.ipv4.tcp_tw_reuse`를 명시적으로 선언하면 간단히 해결됩니다. 커널이 비어 있는 포트를 매 번 스캔할 필요 없이 항상 다음 포트를 빠르게 재사용합니다.

```console
$ echo 1 > /proc/sys/net/ipv4/tcp_tw_reuse
$ sysctl net.ipv4.tcp_tw_reuse
net.ipv4.tcp_tw_reuse = 1
```

### 타임스탬프

앞서 `TIME_WAIT`에 대해 설명할 때, 일정 시간 남아 있어서 패킷의 오동작을 막아야 한다고 언급했는데 어떻게 재사용이 가능한지 궁금할 것이다.

비밀은 `net.ipv4.tcp_timestamps`에 있습니다.[^18] [RFC 1323](https://tools.ietf.org/html/rfc1323) 에서 고성능 익스텐션으로 제안된 옵션 중 하나이며, 여기에는 두 개의 4 바이트 타임스탬프 필드가 있습니다. `net.ipv4.tcp_tw_reuse`를 활성화하면, 새로운 타임스탬프가 기존 커넥션의 가장 최근 타임스탬프보다도 큰 경우 `TIME_WAIT` 상태인 커넥션을 재사용하게 됩니다. `tcp_tw_reuse`가 비활성화 상태라면 매 번 비어 있는 포트를 스캔하게 되지만, 활성화 상태라면 바로 다음 포트를 사용 또는 재사용 하는 거죠.

그렇다면 처음에 `TIME_WAIT`이 짧을 때 두 가지 문제가 발생할 수 있다고 언급했는데 이 문제에 대해서는 과연 안전할까요?

첫 번째 문제는 쉽습니다. 동일한 `SEQ`라도 이미 지난 타임스탬프이므로 확인 후 그냥 버리면 됩니다.

두 번째 문제도 타임스탬프로 해결됩니다. 서버가 `ACK`을 받지 못한 상태에서 새로운 커넥션이 `SYN`을 보내면 타임스탬프를 비교해 무시합니다. 그러는 사이 서버의 `FIN`이 재전송됩니다. 그러면 `SYN_SENT` 상태에 있던 클라이언트는 `RST`를 보냅니다. 이제 서버가 `LAST_ACK` 상태를 빠져나옵니다. 그러는 사이 `ACK`을 받지 못한 클라이언트는 1초 후 다시 `SYN`을 전송합니다. 이제 서버도 `SYN+ACK`을 보냅니다. 이제 둘은 정상적으로 `ESTABLISHED` 됩니다.[^12]

타임스탬프가 없었을 때는 오류를 내며 새로운 연결이 종료됐지만 이제 정상적으로 연결됐습니다. 단지 약간의 딜레이만 발생했을 뿐이죠.

![last-ack-reuse](http://d1g3mdmxf8zbo9.cloudfront.net/images/tcp/last-ack-reuse.png)

재사용을 위해서는 `net.ipv4.tcp_timestamps` 타임스탬프 옵션이 서버/클라이언트 양쪽 모두 반드시 켜져 있어야 합니다. 리눅스 커널의 기본 값이며 굳이 끌 필요가 전혀 없는 옵션입니다. 어느 한 쪽이라도 꺼져 있으면 더 이상 타임스탬프가 부여되지 않습니다. 옛날에는 CPU 자원을 절약하기 위해 끄기도 했지만 이제는 그런 시대가 아니죠.

아래는 클라이언트가 타임스탬프를 부여했으나 일부러 `tcp_timestamps`를 꺼둔 서버에서 패킷이 어떻게 오고 가는지 재현한 모습입니다. 서버가 `nop`로 응답했고, 그 다음 패킷부터는 타임스탬프가 부여되지 않음을 확인할 수 있습니다.

```console
14:02:48.704077 IP 10.xx.xx.141.1027 > 10.yy.yy.140.5000: Flags [S], seq 2561683863, win 14600, options [mss 1460,sackOK,TS val 380417893 ecr 0,nop,wscale 10], length 0
14:02:48.704120 IP 10.yy.yy.140.5000 > 10.xx.xx.141.1027: Flags [S.], seq 1524418616, ack 2561683864, win 14600, options [mss 1460,nop,nop,sackOK,nop,wscale 10], length 0
14:02:48.704233 IP 10.xx.xx.141.1027 > 10.yy.yy.140.5000: Flags [.], ack 1, win 15, length 0
14:02:48.704505 IP 10.yy.yy.140.5000 > 10.xx.xx.141.1027: Flags [P.], seq 1:27, ack 1, win 15, length 26
```

### 재활용

`TIME_WAIT`을 가장 효율적으로 재활용 하는 방법은 지금 소개하는 `net.ipv4.tcp_tw_recycle` 옵션이지만, NAT 환경에서 문제가 있습니다.[^12] 서버가 로드 밸런서 뒤에 위치하는 서비스 환경에선 장비간 타임스탬프가 일치하지 않아 역전 현상이 발생하면 [패킷 드롭이 발생](http://tagnee.tistory.com/22)할 수 있으므로 **사용하면 안됩니다**. 패킷은 마이크로세컨드 단위로 매우 빠르게 동작하고 장비간 시간을 마이크로 단위로 정확히 맞추기는 사실상 불가능에 가깝기 때문에 사용하기 힘듭니다.

그러나, 성능은 월등합니다.

* `net/ipv4/tcp_minisocks.c`
```c
                if (recycle_ok) {
                        tw->tw_timeout = rto;
                } else {
                        tw->tw_timeout = TCP_TIMEWAIT_LEN;
                        if (state == TCP_TIME_WAIT)
                                timeo = TCP_TIMEWAIT_LEN;
                }
```

`tcp_tw_recycle` 이 활성화 되어 있으면 `TIME_WAIT` 상태를 `TCP_TIMEWAIT_LEN` - 60초가 아닌 `rto` - retransmission timeout 값으로 적용합니다.

```c
...
const int rto = (icsk->icsk_rto << 2) - (icsk->icsk_rto >> 1);
...
newicsk->icsk_rto = TCP_TIMEOUT_INIT;
...
#define TCP_TIMEOUT_INIT ((unsigned)(1*HZ)) /* RFC6298 2.1 initial RTO value  */
```

`rto` 값이 얼마로 설정되었는지 쭈욱 따라가서 쉬프트 연산자를 계산해보면 `1*(2^2) - 1/(2^1)` 이므로 `3.5초`가 됩니다. `tcp_tw_recycle` 를 활성화 하는 것만으로 기존 60초에서 획기적으로 줄어든 셈입니다.

그러나, 앞서 언급했듯 장비간 타임스탬프를 마이크로 세컨드 단위로 일치시키긴 힘드므로 NAT 환경등에선 사용할 수 없고 반드시 서버/클라이언트가 1:1로 직접 연결된 경우에만 사용해야 합니다.

### 타임아웃

> **틀린 정보**: `net.ipv4.tcp_fin_timeout`을 설정하면 `TIME_WAIT` 타임아웃을 변경할 수 있다.

커널 헤더에 `TIME_WAIT`은 `TCP_TIMEWAIT_LEN` 이라는 상수로 `60초` 하드 코딩 되어 있으며 변경할 수 없습니다.

* `include/net/tcp.h`
```c
#define TCP_TIMEWAIT_LEN (60*HZ) /* how long to wait to destroy TIME-WAIT
                                  * state, about 60 seconds     */
#define TCP_FIN_TIMEOUT TCP_TIMEWAIT_LEN
                                 /* BSD style FIN_WAIT2 deadlock breaker.
                                  * It used to be 3min, new value is 60sec,
                                  * to combine FIN-WAIT-2 timeout with
                                  * TIME-WAIT timer.
                                  */
```

커널 코드 주석에서 보듯, 예전에는 `FIN_WAIT2`와 `TIME_WAIT`을 합쳐 3분으로 관리했으나 옛날(*deprecated*) 이야기이며, 지금은 각각 `60초`로 관리합니다. 이 중 `sysctl`로 변경 가능한 값은 `TCP_FIN_TIMEOUT` 뿐이죠. `TCP_FIN_TIMEOUT`은 `FIN_WAIT2`의 대기 시간이며 `TIME_WAIT`과는 무관합니다.

`FIN_WAIT2`는 Active Close를 했는데 Passive Close 쪽에서 `close()`를 처리하지 못하고 `CLOSE_WAIT` 상태에 빠졌을 때를 말합니다. 즉, 상대방에 문제가 있는 상태로, 일정 시간 기다려 주는게 좋습니다.

`TIME_WAIT`의 대기 시간이 `tcp_fin_timeout` 설정에 영향을 받는다는 내용은 옛날 이야기이며, 지금은 오히려 그 반대인 `TCP_TIMEWAIT_LEN` 상수값이 `FIN_WAIT2`의 대기시간에 영향을 끼칩니다. 아래 커널 코드에서 확인할 수 있습니다.

* `include/net/tcp.h`
```c
static inline int tcp_fin_time(const struct sock *sk)
{
        int fin_timeout = tcp_sk(sk)->linger2 ? : sysctl_tcp_fin_timeout;
        const int rto = inet_csk(sk)->icsk_rto;

        if (fin_timeout < (rto << 2) - (rto >> 1))
                fin_timeout = (rto << 2) - (rto >> 1);

        return fin_timeout;
}
```

* `net/ipv4/tcp.c`
```c
        if (sk->sk_state == TCP_FIN_WAIT2) {
                struct tcp_sock *tp = tcp_sk(sk);
                if (tp->linger2 < 0) {
                        tcp_set_state(sk, TCP_CLOSE);
                        tcp_send_active_reset(sk, GFP_ATOMIC);
                        NET_INC_STATS_BH(sock_net(sk),
                                        LINUX_MIB_TCPABORTONLINGER);
                } else {
                        const int tmo = tcp_fin_time(sk);

                        if (tmo > TCP_TIMEWAIT_LEN) {
                                inet_csk_reset_keepalive_timer(sk,
                                                tmo - TCP_TIMEWAIT_LEN);
                        } else {
                                tcp_time_wait(sk, TCP_FIN_WAIT2, tmo);
                                goto out;
                        }
                }
        }
```

* `net/ipv4/tcp_timer.c`
```
        if (sk->sk_state == TCP_FIN_WAIT2 && sock_flag(sk, SOCK_DEAD)) {
                if (tp->linger2 >= 0) {
                        const int tmo = tcp_fin_time(sk) - TCP_TIMEWAIT_LEN;

                        if (tmo > 0) {
                                tcp_time_wait(sk, TCP_FIN_WAIT2, tmo);
                                goto out;
                        }
                }
                tcp_send_active_reset(sk, GFP_ATOMIC);
                goto death;
        }
```

먼저 `tcp.h` 코드를 보면 `tcp_fin_timeout` 설정은 링거 옵션에 영향을 받습니다. 링거에 대해선 뒤에서 설명하겠습니다.

그리고 `tcp.c` 코드에는 `FIN_WAIT2` 상태일 때 대기 시간 설정 로직이 있습니다. `tcp_fin_time()`을 호출한 `tcp_fin_timeout` 값이 만약 `TCP_TIMEWAIT_LEN` 상수보다 크면 그 시간을 뺀 만큼을 기다리게 되는데, 곧바로 `timewait` 타이머가 동작하는게 아니라 먼저 `keepalive` 상태는 그대로 두고 `TCP_TIMEWAIT_LEN` 값인 60초를 뺀만큼 타이머만 설정합니다. 이 시간이 종료된 다음 `tcp_timer.c` 에선 `timewait` 상태로 변환하고 다시 타이머를 구동하는데 마찬가지로 `TCP_TIMEWAIT_LEN` 60초 뺀 만큼 구동합니다. 따라서 `keepalive / timewait` 두 번의 `FIN_WAIT2` 타이머가 동작합니다.

즉, 리눅스 커널의 `TCP_TIMEWAIT_LEN`의 상수값은 `FIN_WAIT2`의 대기 시간에 영향을 끼칩니다. `TIME_WAIT`은 항상 60초 이므로 만약 `tcp_fin_timeout`을 60 이상으로 설정한다면 위에서 언급한대로 keepalive 타이머가 먼저 동작합니다. 만약 60보다 작은 값을 설정하면 그냥 그 시간 만큼 timewait 타이머만 동작한다. 기본값은 60이므로 원래는 60초만큼 `FIN_WAIT2` / timewait 상태로 기다리게 됩니다.

그렇다면 `tcp_fin_timeout`을 90으로 설정하고 실제로 그렇게 동작하는지 확인해 보겠습니다. 90인 경우 60초를 뺀 `FIN_WAIT2` / keepalive 로 30초, `FIN_WAIT2` / timewait 으로 30초를 기다리겠죠. 서버는 정확히 45초 후에 ACK을 보내도록 설정했습니다. 그렇게 하면 45초 이후 `TIME_WAIT` 상태가 되어 다시 60초 간 유지되겠죠. 실제로 그렇게 동작하는지 확인해 보았습니다:

```console
$ while true; do ss -tanop | grep 5000; sleep 1; done
FIN-WAIT-2 0      0              10.xx.xx.141:50094         10.yy.yy.140:5000   timer:(keepalive,29sec,0)
FIN-WAIT-2 0      0              10.xx.xx.141:50094         10.yy.yy.140:5000   timer:(keepalive,28sec,0)
...
FIN-WAIT-2 0      0              10.xx.xx.141:50094         10.yy.yy.140:5000   timer:(keepalive,2.067ms,0)
FIN-WAIT-2 0      0              10.xx.xx.141:50094         10.yy.yy.140:5000   timer:(keepalive,1.020ms,0)
FIN-WAIT-2 0      0              10.xx.xx.141:50094         10.yy.yy.140:5000   timer:(timewait,29sec,0)
FIN-WAIT-2 0      0              10.xx.xx.141:50094         10.yy.yy.140:5000   timer:(timewait,28sec,0)
...
FIN-WAIT-2 0      0              10.xx.xx.141:50094         10.yy.yy.140:5000   timer:(timewait,16sec,0)
FIN-WAIT-2 0      0              10.xx.xx.141:50094         10.yy.yy.140:5000   timer:(timewait,15sec,0)
TIME-WAIT  0      0              10.xx.xx.141:50094         10.yy.yy.140:5000   timer:(timewait,59sec,0)
TIME-WAIT  0      0              10.xx.xx.141:50094         10.yy.yy.140:5000   timer:(timewait,58sec,0)
...
TIME-WAIT  0      0              10.xx.xx.141:50097         10.yy.yy.140:5000   timer:(timewait,1.188ms,0)
TIME-WAIT  0      0              10.xx.xx.141:50097         10.yy.yy.140:5000   timer:(timewait,141ms,0)
```

정확하게 예상한대로 동작함을 확인할 수 있습니다.

`FIN_WAIT2`에서 `ACK`을 받지 못하면 소켓은 곧바로 종료됩니다. 따라서 리눅스 커널은 `FIN_WAIT2`를 `TIME_WAIT`처럼 활용하고 있습니다. `TIME_WAIT`은 패킷의 오동작을 막아주는 역할을 하는데 `FIN_WAIT2` / timewait 상태는 `tcp_tw_recycle` 처리와 일부 잘못된 패킷 처리 로직이 상단에 추가된 형태로 사실상 `TIME_WAIT`과 동일한 역할을 하게 됩니다.

* `net/ipv4/tcp_minisocks.c`
```c
enum tcp_tw_status
tcp_timewait_state_process(struct inet_timewait_sock *tw, struct sk_buff *skb,
                           const struct tcphdr *th)
{
...
        if (tw->tw_substate == TCP_FIN_WAIT2) {
                /* Just repeat all the checks of tcp_rcv_state_process() */

                /* Out of window, send ACK */
                if (paws_reject ||
                    !tcp_in_window(TCP_SKB_CB(skb)->seq, TCP_SKB_CB(skb)->end_seq,
                                   tcptw->tw_rcv_nxt,
                                   tcptw->tw_rcv_nxt + tcptw->tw_rcv_wnd))
                        return tcp_timewait_check_oow_rate_limit(
                                tw, skb, LINUX_MIB_TCPACKSKIPPEDFINWAIT2);

                if (th->rst)
                        goto kill;
...
// `FIN_WAIT2`인 경우 `TIME_WAIT`의 처리 함수에
// 상단에 몇 가지 패킷 처리가 추가 되어 동일하게 동작한다.
...
        }

        /*
         *      Now real TIME-WAIT state.
         *
// 실제 `TIME_WAIT` 상태에 대한 처리
```

그래서 `net.ipv4.tcp_fin_timeout` 값은 90 정도로, `FIN_WAIT2`의 keepalive / timewait 이 모두 일정시간 동작할 수 있는 값으로 설정하는 것을 추천합니다.

### 재시도

명칭이 비슷하여 혼동할 수 있으나 `FIN_WAIT1`은 `FIN_WAIT2`와 다른 상태입니다. `FIN`을 보내주길 하염없이 기다리기만 하는 `FIN_WAIT2`와 달리 `FIN_WAIT1`은 우리가 보낸 최초 `FIN`에 대해 아직 `ACK` 응답이 도달하지 않은 상태로, 일반적으로 상대방 OS에 문제가 있는 경우로 간주할 수 있습니다. 왜냐하면 최초 `ACK` 응답은 리눅스던 윈도우던 OS가 바로 보내야 하는 패킷이기 때문이죠.

또한 기다리기만 하는게 아니라 지속적으로 `FIN`을 재시도합니다. 첫 패킷에는 2초, 그 다음 부터는 5초, 10초. 대기시간을 늘려가며, 따로 커널 설정을 변경하지 않은 서버에서 최대 8번까지 재시도 하는 것을 확인할 수 있었습니다.

![FIN_WAIT 상태의 FIN 재시도](https://farm1.staticflickr.com/275/18556772069_4671bf2e8e_b.jpg)

마지막 8번째가 55초 대기하는 것을 포함, 총 2분 정도 대기했으며 마지막까지 `ACK`을 수신하지 못할 경우 결국 소켓은 종료됩니다. 대기 시간이 길고, 재시도 횟수와 대기 시간을 조정할 수 있는 커널 설정까진 미처 확인하지 못했지만, 상대방 OS에 문제가 있는 상태인 만큼 새로운 연결 `SYN`에도 응답하지 않을 것이고, 해당 IP와는 더 이상 연결이 불가능해지겠죠.

### 링거

소켓에는 데이타가 아직 남아 있을때 종료 방식을 결정하는 링거 옵션이 있으며 아래 3가지 경우로 나뉩니다.

```c
struct  linger {
  int l_onoff;    /* option on/off */
  int l_linger;   /* linger time */
};
```

1. `l_onoff = 1`, `l_linger = 0` : 즉시 종료하고 버퍼에 남아 있는 데이타는 버립니다. 비정상 종료로 `RST`를 보내고 즉시 연결을 끊습니다. `RST`의 동작에 관해서는 아래에 다시 설명하겠습니다.
2. `l_onoff = 1`, `l_linger = non-zero` : 명시된 시간(초)동안 정상적으로 진행하며 그 이후에는 비정상 종료 처리합니다. 만일 버퍼에 전송되지 못한 메시지가 남아 있다면 명시된 시간 동안은 어플리케이션이 `close()`를 진행하지 못하고 최종적으로 `RST`를 보내기 위해 `sleep` 상태에서 대기하기 때문에 어플리케이션 지연 현상이 발생하므로 유의해야 합니다. non-blocking 으로 동작하는 것도 가능은 합니다.
3. `l_onoff = 0` : 정상적인 *4-way handshake* 종료 과정을 진행하며 소켓의 기본값입니.

`RST` 는 비정상 종료시 보내는 패킷입니다. 수신한 상대방은 `Connection reset by peer` 오류가 나게 되죠.

양쪽 모두 바로 연결이 끊어지며, 양쪽 모두 `TIME_WAIT` 상태가 남지 않는다는 점에서 가장 빠르고 깔끔해 유용해보이지만 문제는 비정상 종료라는 점입니다. `RST`는 더 이상 연락하지 말자는 일방적인 이별 통보로, 또다른 side effects를 야기할 수 있습니다. 또한 양쪽 모두에 `TIME_WAIT`을 남기지 않기 때문에 패킷의 오동작을 막아줄 장치가 없습니다.

어떠한 `TIME_WAIT`도 남아 있지 않아야 할 특수한 목적이 아니라면, 일반적으로는 링거 옵션을 사용하지 않아야 하고 `RST` 비정상 종료 패킷을 보내는 일이 없어야 합니다.

### 정리

TCP/IP Illustrated를 쓴 리차드 스티븐스의 또 다른 책 Unix Network Programming에는 이런 구절[^12]이 있습니다.

> The `TIME_WAIT` state is our friend and is there to help us (i.e., to let old duplicate segments expire in the network). Instead of trying to avoid the state, we should understand it.

`TIME_WAIT`은 우리를 도와주는 친구다. 네트워크에서 오래된 중복 세그먼트를 날려주는 훌륭한 역할을 한다. 자꾸 없애려고 노력하지 말고 이해해야 한다.

`TIME_WAIT`은 패킷의 오동작을 막아주는 우리의 친구같은 존재입니다.

수 많은 잘못된 정보들 사이에서 아래와 같은 올바른 정보를 반드시 기억해두길 바랍니다:

* `TIME_WAIT`의 타임아웃은 `60초`로 하드 코딩되어 있다. 설정할 수 없다.
* 다수의 `TIME_WAIT`이 서버 성능을 저하시킨다는 논문[^15]은 1997년에 출판됐다. 지금은 2015년이다.
* 클라이언트가 서버 투 서버로 한 서버에 요청이 많을 경우 `tcp_tw_reuse` 옵션을 설정해 `TIME_WAIT`을 재사용하도록 한다. 서버는 해당 사항이 없다.
* 오래된 서버인 경우 클라이언트가 서버 투 서버 통신을 많이 한다면 빈 포트 스캔으로 성능 저하가 발생하므로 마찬가지로 `tcp_tw_reuse` 옵션을 설정한다.
* `tcp_tw_reuse`와 `SO_REUSEADDR`는 서로 다른 소켓에 적용되는 옵션이다.
* 서버/클라이언트 모두 `tcp_timestamps`가 기본값인 켜져 있어야 하며, 끄면 안되고 끌 필요도 없다.
* `net.ipv4.tcp_fin_timeout`은 `90` 정도로 설정한다.
* `FIN_WAIT1`은 상대방 OS에 문제가 있는 경우다.
* `FIN_WAIT2`는 상대방 어플리케이션에 문제가 있는 경우다.
* `FIN_WAIT2`는 `TIME_WAIT`의 역할을 대행한다.
* 특수한 경우가 아니면 링거 옵션은 사용하지 않는다.
* 서버가 클라이언트를 `accept()` 할때 할당하는 것은 소켓이다. 포트가 아니다.
* 소켓의 최대 갯수는 65,535개가 아니다. 소켓은 `<protocol>`, `<src addr>`, `<src port>`, `<dest addr>`, `<dest port>` 5개의 값으로 유니크하게 구성되며, 서버 포트 또는 클라이언트의 IP가 추가될 경우 그 만큼의 새로운 쌍을 생성할 수 있다.

### TL;DR

* **서버**는 아무것도 할 필요가 없다.
* **클라이언트**[^19]는 `net.ipv4.tcp_tw_reuse`를 1로 설정한다.
* **서버**와 **클라이언트**가 NAT 없이 1:1 로 직접 연결되어 있다면 압도적인 성능을 보이는 `net.ipv4.tcp_tw_recycle`을 적극 활용한다.

----
[^1]: http://www.codeitive.com/0xJeqqgPPW/reproduce-tcp-closewait-state-with-java-clientserver.html
[^2]: http://stackoverflow.com/questions/15912370/how-do-i-remove-a-close-wait-socket-connection#comment22663601_15912370
[^3]: http://unix.stackexchange.com/a/10132
[^4]: http://benohead.com/tcp-about-fin_wait_2-time_wait-and-close_wait/
[^12]: http://vincent.bernat.im/en/blog/2014-tcp-time-wait-state-linux.html
[^13]: http://www.quora.com/What-is-the-ideal-design-for-server-process-in-Linux-that-handles-concurrent-socket-I-O
[^14]: http://stackoverflow.com/a/1854196
[^15]: http://www.isi.edu/touch/pubs/infocomm99/infocomm99-web/
[^16]: https://github.com/torvalds/linux/commit/a9d8f9110d7e953c2f2b521087a4179677843c2a#diff-3973f2a099d75b1ec9f7fe686cd0796a
[^17]: http://marc.info/?l=linux-netdev&m=123174371107238&w=2
[^18]: https://www.facebook.com/likejazz/posts/10153290132235837?comment_id=10153292111270837
[^19]: 여기서 말하는 클라이언트란 일반적인 클라이언트가 아니라, 서버 투 서버로 대용량으로 접속하는 클라이언트를 말한다.

----
> 이 글은 [kaon.park](https://github.com/likejazz)이 [블로그 likejazz.com](http://likejazz.com)에 포스팅한 글 [`CLOSE_WAIT` 문제 해결](http://docs.likejazz.com/close-wait)과 [`TIME_WAIT` 상태란 무엇인가?](http://docs.likejazz.com/time-wait/)를 원저자의 동의를 얻어 엮은 글입니다.
>
> 글의 내용도 유용하지만, 문제의 원인을 찾고 해결책을 찾기 위해 리눅스 커널 소스까지도 뒤질 수 있는 패기! 그것이 더 좋은 개발자가 되는 지름길이 아닐까요? 여러분도 도전해보세요! 책으로 배우는 지식과는 비교할 수 없이 값진 경험을 얻게 될 것입니다.
>
> 편집자가 사족을 달자면, [TCP/IP Illustrated](http://www.yes24.com/24/goods/8234905) 보세요. 두 번 보세요~ [스티븐스](https://en.wikipedia.org/wiki/W._Richard_Stevens)는 사랑입니다.

* 커버 이미지 출처: [Plug Socket](https://flic.kr/p/7ymr9j) &copy; [Will Jackson](https://www.flickr.com/photos/willmx/)

----
> ## 채용 공고
> 카카오에서 통합 검색을 담당할 개발자를 채용 중입니다.
> 검색 서비스의 최상단에서 사용자의 검색 요청을 처리하는 시스템으로, 이 글의 내용처럼 `TIME_WAIT`을 포함한 다양한 네트워크 프로그래밍 및 문제 해결에 관심 있는 분들의 많은 지원 바랍니다.
> [지원하기](http://kakaocorp.com/recruit/progressRecruit?uid=9601)