## 2020年4月26日 星期日

为了解决`ns_rpc`可能的卡点和超时计算问题，用[tokio::select](https://docs.rs/tokio/0.2.19/tokio/macro.select.html)替换了[futures::select](https://docs.rs/futures/0.3.4/futures/macro.select.html)。然后发现在`select`宏中使用`await`时会造成卡点，使得`select`调度不公平。为了解决这个问题，又是硬着头皮把`futures`的各种解决方法试了一大圈，总算是搞定了事情，但是编程效率太低了。往好处想，也算是把`futures`和`tokio`又学习了一遍。

通过`tokio`的这次升级，得出的经验是，以后的包依赖版本号还是用三位比较好。因为`caret`以来模式是大于等于这个版本，与两位版本号区别是不用后向兼容。

`tokio`的[time crate](https://docs.rs/tokio/0.2.19/tokio/time/index.html)中很反人类的实现了几个超时函数，[timeout](https://docs.rs/tokio/0.2.19/tokio/time/fn.timeout.html)不说了，还算是正常。[Delay](https://docs.rs/tokio/0.2.19/tokio/time/struct.Delay.html), [Interval](https://docs.rs/tokio/0.2.19/tokio/time/struct.Interval.html)把我折腾的够呛。`Delay::reset`实际上与`delay_until`的参数一样，而不是我想象的`duration`不用再加上。`interval`则是立刻触发而不是等下一个时间点，而且interval是固定触发的，而不是每次调用时启动计时。

在Review ns_web 代码时，看到entry初始化client时引入了一个downstream的死锁点。而query函数中也会检查client是否初始化。按照规范，不能够在entry中调用耗时的函数，特别是会导致连环死锁的激活函数。解决这个问题的一个思路是采用一种可以类似[lazy_static](https://docs.rs/lazy_static/1.4.0/lazy_static/)的函数，按需进行一次性的初始化，并且最好是允许在读操作函数中执行。翻查一下crate，发现[once_cell](https://docs.rs/once_cell/1.3.1/once_cell/)能够很好解决这个问题。于是简单用once_cell更换lazy_static，果然非常平滑，替换完毕所有测试及例子运行正常。后续需要把inner的Option用OnceCell替换。

使用`submodule`命令把web和writings两个文案库合并到一处，以后可以使用typora进行统一文档书写了。

## 2020年4月25日 星期六

今天tokio升级到了v0.2.19，对ns来说，可以不再使用性能弱的`tokio::io::split`，而是使用`tokio::net::TcpStream::into_split`，一方面是性能好，另一方面是可以通过 deref 获取TcpStream进行更为细致的操作，比如关闭连接。

这次改动需要最新的tokio和最新的rust。tokio可以通过Cargo.toml进行限定，可是rust呢？好像还没有办法进行限定，只能粗略限定edition和channel。没办法，干脆都不做限定吧。

## 2020年4月24日 星期五

前几天在知乎上回答一个关于编程的[问题](https://www.zhihu.com/question/381052177/answer/1094025713)，没想到很快得到了好几个回复，都是劈头盖脸的一通话，而且很明显曲解了我回答的本意。哭笑不得的同时，我在反思：要么是我的文字表达能力太差，引起了别人的误解；要么是别人的文字理解能力太差，误解了我的本意。但是总的结论是：程序员群体作为理工男可能普遍缺乏较好的文学训练，文字功底太差。所以，告诫自己：以后要多写些文字，不然随着年龄的增长，越来越“忘言”了。

昨天总算是给[neunit_system](https://github.com/neunit/ns)打上了0.3.0的标签，框架的基本开发工作也算是告一个小段落。后续的工作要多多编写实际应用的代码程序，在应用开发中总结框架的不足。框架中还留下不少扫尾工作和半吊子工程：

- [x] 文档清理，更新同步文档内容与代码修改。
- [ ] 加入一个自动的文档生成器，把程序代码生成文档和网页。
- [x] 订阅者可以自行释放sink或者通过返回错误结果实现unsubscribe
- [ ] 实现控制台功能，能够在控制台中对系统进行控制。
- [ ] 实现代码的热加在机制
- [ ] 动态加载机制。由于tokio运行时的限制，当前只能采用静态加载机制。硬要实现的话，blocking feature无法使用。
- [ ] 文件系统模块
- [ ] proxy/stub模块
- [ ] 数据库连接模块

选择一个应用进行开发。基于当前项目，可以从Web服务框架开始。包含如下内容：

- [x] Web服务；
- [ ] 文件沙盒服务；
- [ ] 数据库服务；
- [ ] REST接口；
- [ ] GraphQL接口；




早上刚刚更新了rust v1.43，看release notes没有让人眼前一亮的改动。不过 make clippy 时看到了些许的不同：use mime; 这种根引用已经会被警告，标志为不必要。

实现心头一直挂着的订阅机制，居然花了一整天时间。提交完代码感觉眼睛要瞎了。总算是解脱了。ns_iot版本号也顺利升到 v0.3.1 

