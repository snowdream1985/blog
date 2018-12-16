+++
date = "2017-02-26T12:56:34+08:00"
draft = false
title = "Android进程级别和oom_adj对应关系"
+++

oom_adj 数值越小证明优先级越高, 被干掉的时间越晚. 如果大于 8 一般就是属于 backgroud 随时可能被干掉. -100为system进程的值, 表示绝不会被干掉.

### [oom_adj=0]
**前台进程 (Active Process)**

前台进程包括: 

    1.活动 正在前台接收用户输入 
    2.活动、服务与广播接收器正在执行一个onReceive事件的处理函数
    3.服务正在运行 onStart、onCreate或onDestroy事件处理函数.
 
**已启动服务的进程(Started Service Process)**

    这类进程包含一个已启动的服务. 服务并不直接与用户输入交互,因此服务的优先级低于可见活动的优先级,但是,已启动服务的进程任被认为是前台进程,只有在活动以及可见活动需要资源时,已启动服务的进程才会被杀死.

### [oom_adj=1]
**可见进程 (Visible Process)**

    活动是可见的,但并不在前台,或者不响应用户的输入.例如,活动被非全屏或者透明的活动所遮挡.
 
### [oom_adj=2]
**后台进程 (Backgroud Process)**

    这类进程不包含任何可见的活动与启动的服务.通常大量后台进程存在时,系统会采用（last-seen-first-kill）后见先杀的方式,释放资源为前台进程使用.
 
### [oom_adj=4]
**主界面 (home process)**
 
### [oom_adj=7]
**隐藏进程 (hidden process)**
 
### [oom_adj=14]
**内容提供者 (content provider)**
 
### [oom_adj=15]
**空进程 (Empty process)**