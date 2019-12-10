#  DSR协议分析
  
  
  
  
  
  
- [DSR协议分析](#dsr协议分析 )
  - [名词解释](#名词解释 )
  - [工作环境](#工作环境 )
    - [传统路由与无线路由](#传统路由与无线路由 )
  - [功能](#功能 )
    - [路由发现 > 3.1 3.3](#路由发现-31-33 )
    - [路由维护 > 3.2 3.4](#路由维护-32-34 )
  - [实现](#实现 )
    - [数据结构](#数据结构 )
  - [参考](#参考 )
  
  
  
  
TODO: 将本md文件只作为在MPE的源文件, 然后通过MPE导出一个符合markdown语法的README到
  
协议主要通过两个机制来实现路由发现(Route Discovery)和路由维护(Route Maintance)
> Route Discovery is the mechanism by which a node S wishing to send a packet to a destination node D obtains a source route to D. Route Discovery is used only when S attempts to send a packet to D and does not already know a route to D.
> Route Maintenance is the mechanism by which node S is able to detect, while using a source route to D, if the network topology has changed such that it can no longer use its route to D because a link along the route no longer works. When Route Maintenance indicates a source route is broken, S can attempt to use any other route it happens to know to D, or it can invoke Route Discovery again to find a new route for subsequent packets to D. Route Maintenance for this route is used only when S is actually sending packets to D.
  
路由发现, S想要想D发送数据包但是却没有D的路由时使用
路由维护, S已知D的路由, 但网络的拓扑结构改变了, 链接已经失效; 当路由维护功能发现链路不可用时, S会选择其他路径或者重新激活路由发现
在DSR中, 上述两个功能都是被动调用的, **DSR不会周期性的发送数据包**, 在变化的移动网中不会加大网络的负荷.
路由发现和路由维护都允许单向链接和非对称路由. 在无线网中, 两个节点间的关系不一定在每个方向上都是平等的, 因为每个节点的传输性能不一样.
DSR为每个报文选择下一跳的路由, 同时提供下一跳的IP. DSR使用ARP去翻译IP到下一跳的MAC地址.
  
##  名词解释
  
| terminology           | meaning  |
| --------------------- | -------- |
| routing advertisement | 路由通告 |
  
##  工作环境
  
dsr应用在Adhoc无线网络环境下. 最大支持200跳. DSR被设计用于高速变化的网络环境下.
假设网络环境中所有设备都主动合作, 遵循DSR协议. 假设网络半径较小在5到10跳之间.
网络下两个节点间的关系不一定在每个方向上都是平等的, 因为每个节点的传输性能不一样, 在一对节点的通信中, 因为传输能力的原因, 可能有的节点间只有一方能发, 能收到信息不代表也能给对方发送信息.
  
###  传统路由与无线路由
  
AdHoc指无线路由, 多跳的, 无中心, 自组织的无线网络. 网络中没有固定的基础设施, 每个节点都是移动的, 并且动态地保持与其他节点的联系, 每个节点同时也是路由器.
  
##  功能
  
  
###  路由发现 > 3.1 3.3
  
发起者(initiator)想要发送一个报文到目标(target). 一般情况下发送者会搜索其路由表看是否有匹配的, 如果没有则会启动路由发现功能.
假设A是路由发现的发起者, A想要一条到E的路由, 路由发现一般过程如下:
1. A发送一条路由请求(Route Request)广播, 所有在A的广播范围内的设备都会收到
2. 路由请求报文中包含发起者和目标的信息; 同时包含一个唯一的请求标识, 标识由发起者决定; 同时包含所有发送这份报文的节点的记录, 从A节点开始, "A", "AB"
3. 中间节点收到路由请求报文后, 将其地址添加到路由记录列表中, 同时将此路由请求广播; 如果收到路由请求的节点在一定时间内(时间范围TODO:)收到一份相同的路由请求报文(id相同), 或者当前节点已经在路由记录列表中了, 则丢弃该报文
4. 如果收到路由请求的节点是目标节点, 则其返回一个路由应答(Route Reply), 路由应答中包含了路由发现过程中的节点列表记录; 为了目标节点为了发送应答报文给发起者, 首先会在自己的路由表中寻找发起者的路由, 如果找到, 则由此路由发送应答报文;
5. **如果目标节点未找到到发起者的路由**, 则目标节点发起一个到前发起者的路由请求(E->A, 因为无线网络下链路不是双向的); 为了防止路由请求递归发生, 目标节点的路由请求中会包含之前的路由请求报文(A->E), 同时也允许携带其他类型的数据报文; 为了避免过多的路由请求, 目标节点也可以简单地用*反向路由*将路由应答报文按*源路由(Source Route)*返回, 这需要底层协议的支持, 双向链路测试发生在发起者使用这条路由之前
6. 当发起者收到路由应答报文后, 将其记录到路由表
  
一点点细节:
1. 发送路由请求报文的节点会在本地保存一份备份(Send Buffer); 备份标记了保存的时间, 如果一定时间内(SendBufferTimeout)收到重复的报文则直接丢弃, 发送缓存应由FIFO队列实现
2. 节点应该不定时会发起一个新的路由发现到保存在*缓存*中的目标节点; 为了防止过量的路由请求, 节点应该实现一个算法限制发送路由请求的频率, 每次倍增路由请求的超时时间; 最大请求限制与ARP请求对单节点的限制相同
  
###  路由维护 > 3.2 3.4
  
当沿着路由发现生成的*源路由*发送报文时, 每个节点有有责任确认链路下一跳的可达性.
1. 默认情况下, 在无线网络环境下, (passive acknowledgement), 节点收到临近节点的确认报文后, 在一段时间内会不要求临近节点再次发送确认报文;
2. 如果确认请求发送一定量后没有收到确认报文, 则认为该链路断了, 应该将与其相关的路由删除
> FIXME: 翻译 in wireless networks, acknowledgements are often provided at no cost
  
###  小结
  
参与者: 发起者, 中间节点, 目标节点
报文: 路由请求, 路由响应, 确认
缓存: 路由表, 发送缓存
  
##  实现
  
  
###  项目结构
  
| 文件       | 内容               |
| ---------- | ------------------ |
| io         | 数据包的发送和接受 |
| dev        |
| module     |
| pkt        | 数据包, 报文       |
| srt        | 源路由             |
| rreq       | route request      |
| rerr       | 路由错误           |
| rrep       | 路由应答           |
| ack        | 确认, 确认请求     |
| opt        | 选项               |
| link-cache | 链路缓存           |
| send-buf   | 发送缓存           |
| neigh      | 相邻节点           |
| rtc-simple | rtc路由缓存        |
| endian     | 字节序             |
  
  
  
  
###  数据结构
  
  
  
##  参考
  
任务要求: 做一个ppt
基本原理, 代码分析, 16到17周讲解, 5到10分钟, 12.15号之前发到zhqqin33@qq.com
[DSR RFC4728](./rfc4728.pdf )
[dsr协议分析](https://blog.csdn.net/norbert_jxl/article/details/9241421 )
[需要下载](https://download.csdn.net/download/gtnets/2334998 )
[dsr与aodv比较](https://blog.csdn.net/hbhzwj/article/details/5269073 )
[adhoc的dsr应用报告?](http://www.wanfangdata.com.cn/details/detail.do?_type=perio&id=wlwjs201407019 )
[adhoc网络需要解决的问题, 百度百科](https://baike.baidu.com/item/AdHoc%E7%BD%91%E7%BB%9C/6106787?fr=aladdin )
  
  
  