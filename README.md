<h1>Server 一个纯异步的Server简单实现</h1>
<h3>基于Nginx/Lighttpd的状态机实现, 再也不用去一坨代码里找真相啦</h3>
<br/>

<p>最新说明:</p>
<p>@2012/10/22 18:42: 代码整理划分模块, 添加plugin回调逻辑, 提供简单的示例plugin demo, 供感兴趣的同学阅读与反馈. </p>
<p>@2012/10/23 15:00: 添加Mysql Plugin, 提供了一个生动的例子, 引入多个Plugin的过程中发现了一些新问题, 因此src/client.cpp, src/server.cpp中的Plugin调用逻辑出现了变动, 在代码中有相关注释, 现在整
个项目更具有可读性与实用性了, 欢迎大家找BUG提建议.</p>
<p>===============================================================================================================</p>
<p>Server自带两个Plugin:<br/>
1 ) slow_query计时plugin, 将会在结果中包含请求时间, 该plugin自身将会每5秒响应一次. <br/>
2 ) fake_mysql伪装mysql query plugin, 任何请求将会阻塞5秒后才返回Mysql查询结果(伪装非常慢的Mysql查询请求).</p>
</p>
<p>===============================================================================================================</p>
<h3>背景:</h3>       
      XX希望用Epoll开发Server, 但业务上有很多阻塞的逻辑, 怕会阻塞Epoll, 所以不知道怎么设计比较好, 也就是知道大概要怎样, 但实际操作上又举步维艰的一种状态.
      
      比如, 老大交给他几个任务:
      1, 需要将Server接受到的请求向其他Server做转发并从其他Server读取应答后发回给Client(像反向代理一样).
      2, 需要将Client请求存入数据库, 或者从数据库中查询一些东西后返回给Client.
      
      XX疑惑的是, 他希望用epoll单线程, 而又不希望业务逻辑阻塞线程. 他大概知道要将业务逻辑放到线程中去阻塞的处理, 但不
      知道怎么融合到epoll单线程中, 有些不知所措了.
<p>===============================================================================================================</p>
<h3>说明:</h3>
<h4>
     我隔天写了一段代码, 只写了一些必要的内容, 让XX借着代码理解一下实现方式. 其实和一般的I/O复用Server开发是基本一样的,但为了解决XX的问题, 引入状态机机制会让整个Server看起来更清晰易懂, 让Server架构和具体的业务逻辑解耦开来, 真正开发时只需要关注自己的业务逻辑, 这是非常有益处的, 同时, 也为XX解开了他最初的疑惑.
</h4>