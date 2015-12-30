title:  "TCP/IP,Java Socket简单分析"
date: 2015-12-30
categories:
- network
tags:
- network
---

> 网络是互联网的基础设施

## TCP/IP协议状态图
在用`netstat -anp`查询出来的结果中，有state一列，是TCP/IP状态，这些状态是如何流转的？特别是三次握手建立连接，四次握手断开连接这些基本的状态迁移是怎么进行的，通过下图能有一个比较清晰的了解。
![TCP/IP状态迁移图](/images/2015/12/tcp-ip-states.png)
三次握手比较好理解，断开连接比较复杂。并且在分析长连接过程中，我们往往要考虑的就是怎么样能够找出实际已经断开的连接，这样就能主动去复用或者回收掉这个长连接。
断开连接分为：
- 客户端主动断开。1）客户端发送FIN；2）服务端收到FIN，并发送ACK，进入到CLOSE_WAIT；3）服务端结束发送数据，并发送FIN，进入到LAST_ACK；4）服务端收到ACK，连接回收，如果没收到ACK估计也会回收
- 服务端主动断开。1）服务端结束，发送FIN，进入到FIN_WAIT_1；2）在FIN_WAIT_1状态下，根据接收到的FIN，ACK的不同顺序，服务端会进入FIN_WAIT_2或者CLOSING，但是最终；3）服务端会进入到TIME_WAIT，在这个状态下，一般会等待2小时，原因是/etc/sysctl.conf里面的keepalive参数，最终会被内核回收掉这个连接

## socket.close()过程中的状态流转
### 前提
- 以下讨论的都是Java Socket，Berkeley Socket是否是同样的行为没有实证过，姑且认为同Java Socket一样表现，毕竟Java Socket最终是需要OS上的Socket实现的；
- 结构清晰，逻辑完整，且网络良好的情况下，是很难看到除了ESTABLISHED以外的状态的，因为socket.close()的时候会flush，关闭io流并关闭Socket描述符，这一步就是发送FIN/SYN/ACK等报文，然后内核回收Socket资源了；
- 以下出现的一些中间state是人为在一端进行sleep，比如客户端首先socket.close()，但是服务端不马上进行socket.close()，而是等待一下，这样FIN SYN ACK就不会马上发送，所以就能看到这些中间状态了；
- 服务端口12345，由于是在本地测试，所以两条状态中上面的是服务端Socket，下面的是客户端Socket

### netstat观察到的情况
- 所有的起始状态都是ESTABLISHED，上面一条状态是服务端的Socket，下面一条是客户端的Socket

Proto      |Recv-Q |Send-Q  |Local Address          |Foreign Address        |(state)
-----------|-------|--------|-----------------------|-----------------------|-----------
tcp4       |0      |0       |192.168.31.102.12345   |192.168.31.102.53615   |ESTABLISHED
tcp4       |0      |0       |192.168.31.102.53615   |192.168.31.102.12345   |ESTABLISHED

- 客户端主动关闭。服务端Socket：CLOSE_WAIT->LAST_ACK->内核回收Socket成功，客户端Socket：FIN_WAIT_2->TIME_WAIT->内核回收Socket成功

Proto      |Recv-Q |Send-Q  |Local Address          |Foreign Address        |(state)
-----------|-------|--------|-----------------------|-----------------------|-----------
tcp4       |0      |0       |192.168.31.102.12345   |192.168.31.102.53615   |CLOSE_WAIT
tcp4       |0      |0       |192.168.31.102.53615   |192.168.31.102.12345   |FIN_WAIT_2

- 服务端主动关闭。可以看到其实就是反过来而已

Proto      |Recv-Q |Send-Q  |Local Address          |Foreign Address        |(state)
-----------|-------|--------|-----------------------|-----------------------|-----------
tcp4       |0      |0       |192.168.31.102.12345   |192.168.31.102.53650   |FIN_WAIT_2
tcp4       |0      |0       |192.168.31.102.53650   |192.168.31.102.12345   |CLOSE_WAIT

- 暂时性的断网。重新连接后客户端依旧可以和服务端通信，并且维持原来的ESTABLISHED状态

- 稍长时间的断网。客户端在read上抛出异常，Socket结束，但是无法通知到服务端，服务端维持ESTABLISHED状态。注意客户端之所以会主动抛出异常，而服务端不抛出异常，是因为一般而言客户端是主动发起请求放，发起请求以后会主动去read服务端的响应，时间一长就会抛出Operation Time Out。但其实如果调整TCP参数，完全可以将read的TIMEOUT给调整为无限大。如果这样则客户端也就一直是ESTABLISHED状态了。这时内核无法回收Socket。

|Proto      |Recv-Q |Send-Q  |Local Address          |Foreign Address        |(state)|
|-----------|-------|--------|-----------------------|-----------------------|-----------|
|tcp4       |0      |0       |192.168.31.102.12345   |192.168.31.102.53853   |ESTABLISHED|

### 需要解决的问题
从上面可以看出，在代码结构清晰，逻辑完整的前提的下，需要解决问题是断网重连的问题。这个也是很多互联网公司会遇到的棘手问题之一。举个例子，Uber，嘀嘀打车之类的App都是与服务器长连接的，但是司机在行车过程中会进入隧道或者信号不好区域，这个时候就是原来的长连接由于网络的原因虽然没有销毁但是已经不可用了。
解决方案
- 依赖[TCP/IP的keepalive机制](http://www.tldp.org/HOWTO/html_single/TCP-Keepalive-HOWTO/)，不喜英文的同学[看这里](http://www.felix021.com/blog/read.php?2076)。一般都需要修改内核的参数，因为keepalive默认是2小时才会认为不可用，被内核回收。至于在keepalive超时以后的状态流转就不清楚了，相关的RFC没有查到。我可以简单猜测一下，由于keepalive是内核TCP/IP协议栈实现的，其在发现对方dead之后，直接将Socket状态置为不可用，这样App层的Java Socket肯定会抛出SocketException，这样内核也回收了Socket，App也收到通知并可以进行自己的逻辑处理。贴一段jdk7中关于SocketOptions.SO_KEEPALIVE的描述
``` java
/**
 * When the keepalive option is set for a TCP socket and no data
 * has been exchanged across the socket in either direction for
 * 2 hours (NOTE: the actual value is implementation dependent),
 * TCP automatically sends a keepalive probe to the peer. This probe is a
 * TCP segment to which the peer must respond.
 * One of three responses is expected:
 * 1. The peer responds with the expected ACK. The application is not
 *    notified (since everything is OK). TCP will send another probe
 *    following another 2 hours of inactivity.
 * 2. The peer responds with an RST, which tells the local TCP that
 *    the peer host has crashed and rebooted. The socket is closed.
 * 3. There is no response from the peer. The socket is closed.
 *
 * The purpose of this option is to detect if the peer host crashes.
 *
 * Valid only for TCP socket: SocketImpl
 *
 * @see Socket#setKeepAlive
 * @see Socket#getKeepAlive
 */
public final static int SO_KEEPALIVE = 0x0008;
```
- 应用层的heartbeat实现。初步的一个想法是在应用层有一个socketManager管理所有连接上的socket，定时发送ping/pong，可以有几个分级的失败队列之类的，最终认为失败的就`socket.close()`。一般需要给心跳一个单独的port，而在检测链路的时候业务port才是主socket通信端口，所以需要`Pair<mainSocket, hbSocket>`这样的结构，但是这样会使得打开文件数立马增加一倍，所以我个人还是比较倾向于同TCP本身提供的keepalive机制，只是相应的参数可以调整为适应业务的，比如10分钟即可。

## 代码
上文一直强调“结构清晰，逻辑完整”的socket代码。其中最重要的一点就是`socket.close()`写在finally中，下面看一下代码结构
- 服务端
```java
public static void main(String[] args) {
    Server server = new Server();
    server.init();
}

public void init() {
    try {
        ServerSocket serverSocket = new ServerSocket(PORT);
        while (true) {
            Socket client = serverSocket.accept();
            // 处理这次连接
            new HandlerThread(client);
        }
    } catch (Exception e) {
        System.out.println("服务器异常: " + e.getMessage());
    }
}

private class HandlerThread implements Runnable {
    private Socket socket;
    public HandlerThread(Socket client) {
        socket = client;
        new Thread(this).start();
    }

    public void run() {
        try {
            DataInputStream input = new DataInputStream(socket.getInputStream());
            DataOutputStream out = new DataOutputStream(socket.getOutputStream());

            /* 业务逻辑 */
            BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(System.in));
            while (true) {
                String receiveData = input.readUTF();//这里要注意和客户端输出流的写方法对应,否则会抛 EOFException
                // 处理客户端数据
                System.out.println("客户端发过来的内容:" + receiveData);
                if ("close".equalsIgnoreCase(receiveData)) {
                    System.out.println("服务端将关闭连接");
                    out.writeUTF("ok");
                    Thread.sleep(10000);
                    break;
                }
                // 读取输入,然后发送给客户端
                System.out.println("请输入:\t");
                String sendData = bufferedReader.readLine();
                // 发送键盘输入的一行
                out.writeUTF(sendData);
            }

        } catch (Exception e) {
            System.out.println("服务器 run 异常: " + e.getMessage());
        } finally {
            System.out.println("finnally, server close ");
            if (socket != null) {
                try {
                    socket.close();
                } catch (Exception e) {
                    socket = null;
                    System.out.println("服务端 finally 异常:" + e.getMessage());
                }
            }
        }
    }
}
```
- 客户端
```java
public static void main(String[] args) {
    System.out.println("客户端启动...");
    System.out.println("当接收到服务器端字符为 \"ok\" 的时候, 客户端将终止\n");
    Socket socket = null;
    try {
        //创建一个流套接字并将其连接到指定主机上的指定端口号
        socket = new Socket(IP_ADDR, PORT);
        //读取服务器端数据流
        DataInputStream input = new DataInputStream(socket.getInputStream());
        //向服务器端发送数据流
        DataOutputStream out = new DataOutputStream(socket.getOutputStream());

        /* 业务逻辑 */
        BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(System.in));
        while (true) {
            System.out.println("请输入:\t");
            String sendData = bufferedReader.readLine();
            out.writeUTF(sendData);
            String receiveData = input.readUTF();
            System.out.println("服务器端返回过来的是: " + receiveData);
            // 如接收到 "OK" 则断开连接
            if ("OK".equalsIgnoreCase(receiveData)) {
                System.out.println("客户端将关闭连接");
                Thread.sleep(10000);
                break;
            }
        }
    } catch (Exception e) {
        System.out.println("客户端异常:" + e.getMessage());
    } finally {
        System.out.println("finnally, client close ");
        if (socket != null) {
            try {
                socket.close();
            } catch (IOException e) {
                socket = null;
                System.out.println("客户端 finally 异常:" + e.getMessage());
            }
        }
    }

}
```