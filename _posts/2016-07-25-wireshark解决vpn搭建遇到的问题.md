<ol>

一直都有自己搭建一个vpn的想法，但是由于种种原因一直没有付诸实践。前几天刷微博看到<a href="http://weibo.com/u/1401880315" target="_blank">@左耳朵耗子</a>发的一条关于如何用一条命令搭建vpn的微博，搭建vpn的念头再次萌生。正好手头上有一个国外的vps，于是立刻去vps上实验了一下。



登录vps，在控制台输入如下的命令，

<blockquote>docker run -d -p 500:500/udp -p 4500:4500/udp -p 1701:1701/tcp -e PSK=xxxx -e USERNAME=yyyy -e PASSWORD=xxxx siomiz/softethervpn</blockquote>

这条命令的作用是: 从docker hub上下载镜像siomiz/softethervpn，并且运行vpn。同时对外启动端口udp端口500， udp端口4500，以及tcp端口1701,  并且设置相应的用户名以及密码。



运行完毕后，我尝试使用windows自带的客户端去连接vpn，但是始终不成功。具体的现象就是客户端一直提示在连接vpn，一段时间后出现如下的错误，



<a href="http://zblacker.com/wp-content/uploads/2016/07/QQ截图20160725215641.png"><img class="alignnone wp-image-148" src="http://zblacker.com/wp-content/uploads/2016/07/QQ截图20160725215641-300x77.png" alt="QQ截图20160725215641" width="300" height="77" /></a>



期初怀疑是windows客户端有问题，于是又使用手机连接，同样不成功。。。由于对vpn建立的整个过程不是很清楚，加上错误提示十分笼统无法直接定位问题，一下子陷入了迷茫。由于最近正好在看&lt;&lt;Wireshark网络分析就这么简单&gt;&gt;,  突然意识到可以通过抓包看看到底为什么报错。

<ul style="list-style-type: circle;">

	<li>启动wireshark</li>

</ul>

打开wireshark，启动抓包，并且在filter一栏上输入

<pre class="EnlighterJSRAW" data-enlighter-language="null">ip.addr == 107.191.60.68</pre>

对抓取的包进行过滤。上面指令的主要作用是告诉wireshark: 把ip层的源地址或者目标地址是107.191.60.68的ip包展示出来。这样做的好处是排除其他数据包对我们的干扰。

<ul style="list-style-type: circle;">

	<li>使用客户端连接vpn</li>

	<li>包分析</li>

</ul>

在wireshark抓到的包中看到了如下的数据，



<a href="http://zblacker.com/wp-content/uploads/2016/07/QQ截图20160725223254.png"><img class="alignnone wp-image-149 size-full" src="http://zblacker.com/wp-content/uploads/2016/07/QQ截图20160725223254.png" alt="QQ截图20160725223254" width="830" height="226" /></a>



&nbsp;



客户端发送了L2TP的请求后，服务器回了一个icmp告诉客户端目标不可达（主机管理员禁止访问）。难道是1701端口启动失败了？在服务器上运行netstat -anop | grep 1701发现1701端口正常启动，那么问题就不应该出现在这。结合icmp给出的提示以及上面windows给出的错误，于是我猜想是不是服务器的防火墙有什么策略？为了验证这个猜想，在服务器上运行如下命令，

<blockquote>iptables -L</blockquote>

可以看到如下的数据，



<a href="http://zblacker.com/wp-content/uploads/2016/07/QQ截图20160725224524.png"><img class="alignnone wp-image-150 size-full" src="http://zblacker.com/wp-content/uploads/2016/07/QQ截图20160725224524.png" alt="QQ截图20160725224524" width="1061" height="796" /></a>



&nbsp;



红线标出的两条规则分别作用于filter表的input和forward链，当这两条规则之前的所有规则都不满足的时候会走到这两条规则，而这两条规则会拒绝大多数的请求，并且返回icp-host-prohibited，这与我们抓包的时候看到的icmp包十分吻合，基本上可以确定之前的问题就是这两条过滤规则导致的。



于是通过下面两条命令把这两条规则去掉了。

<blockquote>iptables -t filter -D INPUT -j REJECT --reject-with icmp-host-prohibited

iptables -t filter -D FORWARD -j REJECT --reject-with icmp-host-prohibited</blockquote>

<ul style="list-style-type: circle;">

	<li>再次使用客户端连接vpn并且抓包</li>

</ul>

<a href="http://zblacker.com/wp-content/uploads/2016/07/QQ截图20160725225553.png"><img class="alignnone wp-image-151 size-full" src="http://zblacker.com/wp-content/uploads/2016/07/QQ截图20160725225553.png" alt="QQ截图20160725225553" width="901" height="97" /></a>



&nbsp;



之前问题没有了，但是发现了新的问题，服务器仍然返回了icmp，只是提示信息变化了。这一次服务器告诉我们端口不可达。。明明端口1701已经启动了，但是为什么会提示这个呢？仔细排查这几个包后发现L2TP的包是通过udp发送出去的，



<a href="http://zblacker.com/wp-content/uploads/2016/07/QQ截图20160725225857.png"><img class="alignnone wp-image-152 size-full" src="http://zblacker.com/wp-content/uploads/2016/07/QQ截图20160725225857.png" alt="QQ截图20160725225857" width="751" height="291" /></a>



&nbsp;



而我们启动的1701是通过tcp启动的。tcp的连接需要三次握手，不能够直接发送数据，所以给客户端返回了这个信息

<ul style="list-style-type: circle;">

	<li>修改启动命令重新启动vpn</li>

</ul>

<blockquote>docker run -d -p 500:500/udp -p 4500:4500/udp -p 1701:1701/udp -e PSK=xxxx -e USERNAME=yyyy -e PASSWORD=xxxx siomiz/softethervpn</blockquote>

<ul style="list-style-type: circle;">

	<li>再次使用客户端连接</li>

</ul>

成功连接。



至此，我的vpn顺利搭建成功~~



<strong>总结，虽然整个搭建过程不复杂，但是在解决问题时涉及到了很多知识点: docker, wireshark抓包分析, 防火墙设置, tcp与udp的区别等等，能通过搭建vpn将这些知识进行串联巩固是一次难得的经验。当我们遇到问题时，多扩展思路善于借助于一些成熟的工具能够快速的帮我们定位甚至解决问题，设想一下如果没有wireshark，面对黑盒的网络我们如何能够这么快的解决这个问题呢。PS: wireshark真是神器，同时也推荐大家阅读下我上面写到的那本书。</strong>



祝大家玩得愉快~~

