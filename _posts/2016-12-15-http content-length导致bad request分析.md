这两天在做项目的时候遇到了一个奇怪的问题，一个请求每次在服务器都返回bad request。bad request的具体含义如下：

> The request could not be understood by the server due to malformed syntax. The client SHOULD NOT repeat the request without modifications.



也就是说这个请求是非法的，服务器在进行协议解析的时候抛出了错误。这个请求主要负责发送一大段二进制流到服务器，发送使用的是httpclient. 不过奇怪的是如果把这个请求单独提出来执行，不会出现这个问题。



由于现有的错误信息无法给定位问题带来实质性的进展，我们需要更多的信息。由于是协议解析问题，抓包无疑是一个很好的方法。在服务器上运行tcpdump -w xxx后， 将抓到的包用wireshark打开，然后定位到出问题的那个请求，由于出问题的请求发送的二进制数据量较大，所以需要借助wireshark的tcp流追踪功能将对应连接发送的所有请求筛选出来。



通过筛选之后发现在出问题的请求之前，该连接还处理另外一个请求。



![http://ww2.sinaimg.cn/large/87f5e2f6gw1faqqpupgiuj21kw06zgra.jpg](http://ww2.sinaimg.cn/large/87f5e2f6gw1faqqpupgiuj21kw06zgra.jpg)



如上图所示，红色的部分代表该连接建连的过程，蓝色的为该连接处理的第一个请求，绿色的为第二个请求的一部分。我们可以看到这些请求的seq值是连续的，这也证明了这两个请求由同一连接处理。



可能有人会好奇为什么一个请求可以处理多个http请求呢？正常的情况下http连接都是短连接，在一次交互后都会进行关闭连接操作。但是也有例外，如果不想关闭该连接，则可以在header中设置Connection为keep-alive，在这种情况下tcp连接不会立刻断开，客户端可以使用该连接完成多次的请求，这样做的好处是可以减少多次反复建连带来的代价。而httpclient的poolinghttpclientconnectionmanager正好利用了这个特性，针对不同的host分别缓存一批连接，这种http连接池的概念和线程池类似，避免了反复创建资源带来的性能损耗。



回到刚才抓取到的数据上，仔细观察出问题的返回包，



![http://ww3.sinaimg.cn/large/87f5e2f6gw1faqrelj6a9j21kw028tat.jpg](http://ww3.sinaimg.cn/large/87f5e2f6gw1faqrelj6a9j21kw028tat.jpg)

![http://ww4.sinaimg.cn/large/87f5e2f6gw1faqremfsi0j21kw0bxwhg.jpg](http://ww4.sinaimg.cn/large/87f5e2f6gw1faqremfsi0j21kw0bxwhg.jpg)



可以看到一些奇怪的现象,



* 该返回值对应的请求为5993帧，而该帧上显示的请求path与第一个请求一致

* 该返回值之前还有另外一个返回值在第6帧，而这个返回值恰好也是第一个请求的返回值



上面的两个怪现象似乎表明这个400返回值不是针对第二个请求的，而是针对第一个请求的，这就比较奇怪了。理论上第一个请求结束后，服务器应该处理第二个请求才是正常的现象，于是猜测是不是什么原因导致了服务器虽然对第一个请求做出了响应，但是服务器仍然觉得第一个请求没有结束，从而导致了服务器在收到了第二个请求的数据后，错把第二个的请求当作第一个，从而协议解析出了问题。



wireshark在进行包图形化展示的时候，http请求都会以如下的方式呈现，

> GET(post) ／path1/path2/...



而第一个包在最开始的第4帧展示的是tcp包，这也就表明了实际上该请求并没有发送完成，直到接收到第5993个包的时候才认为该请求发送完成。观察下wireshark聚合后的请求，



![http://ww1.sinaimg.cn/large/87f5e2f6gw1faqshr2v4rj21kw0atado.jpg](http://ww1.sinaimg.cn/large/87f5e2f6gw1faqshr2v4rj21kw0atado.jpg)



该请求设置了content-length，长度为12763136，而聚合后的请求总长度12763490，长度大致与content-length一致。似乎就是这个参数导致了问题的出现，那么我们来看一下这个参数的具体定义，

>    The Content-Length entity-header field indicates the size of the

   entity-body, in decimal number of OCTETs, sent to the recipient or,

   in the case of the HEAD method, the size of the entity-body that

   would have been sent had the request been a GET.



在请求中该参数表明发送的body的长度，因为带了这个参数，所以服务器认为收到第一个请求后还会有后续的请求，从而导致了误把第二个请求当作了第一个请求。这里之所以防止了这个header是想表明第二个请求将要发送的数据，而服务器在响应这个请求的时候必须知道长度。



为了验证猜想是不是正确的，将header去掉，放置在parameter中传过去，第二个请求可以正常响应，在wireshark中的显示也正常了。



![http://ww1.sinaimg.cn/large/87f5e2f6jw1faqtlbdnibj21kw03edib.jpg](http://ww1.sinaimg.cn/large/87f5e2f6jw1faqtlbdnibj21kw03edib.jpg)



大家如果有兴趣的话可以通过下面的代码来复现下问题，



server

```

package hello;



import org.springframework.boot.*;

import org.springframework.boot.autoconfigure.*;

import org.springframework.stereotype.*;

import org.springframework.web.bind.annotation.*;



@Controller

@EnableAutoConfiguration

public class SampleController {



    @RequestMapping("/")

    @ResponseBody

    String home() {

        return "Hello World!\n";

    }



    public static void main(String[] args) throws Exception {

        SpringApplication.run(SampleController.class, args);

    }

}

```



client

```

package hello;



import org.apache.http.HttpResponse;

import org.apache.http.client.HttpClient;

import org.apache.http.client.methods.HttpGet;

import org.apache.http.client.methods.HttpPost;

import org.apache.http.entity.ByteArrayEntity;

import org.apache.http.impl.client.HttpClients;

import org.apache.http.impl.conn.PoolingHttpClientConnectionManager;



import java.io.BufferedReader;

import java.io.IOException;

import java.io.InputStreamReader;



/**

 * Created by zblacker on 16/12/15.

 */

public class Client {

    public static void main(String[] args) {

        PoolingHttpClientConnectionManager m = new PoolingHttpClientConnectionManager();

        HttpClient client = HttpClients.custom().setConnectionManager(m).build();



        HttpGet get = new HttpGet("http://127.0.0.1:8080");

        // 下面的代码会导致第二个请求出现错误

        get.addHeader("Content-Length", String.valueOf(6000));



        try {

            HttpResponse r = client.execute(get);

            BufferedReader br = new BufferedReader(new InputStreamReader(r.getEntity().getContent()));

            String line;

            while ((line = br.readLine()) != null) {

                System.out.printf(line);

            }

        } catch (IOException e) {

            e.printStackTrace();

        }



        HttpPost post = new HttpPost("http://127.0.0.1:8080");

        post.setEntity(new ByteArrayEntity(new byte[6000]));

        try {

            HttpResponse r = client.execute(post);

            BufferedReader br = new BufferedReader(new InputStreamReader(r.getEntity().getContent()));

            String line;

            while ((line = br.readLine()) != null) {

                System.out.printf(line);

            }

        } catch (IOException e) {

            e.printStackTrace();

        }

    }

}



```
