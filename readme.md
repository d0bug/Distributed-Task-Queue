
##Distributed Task Queue 分布式任务队列框架

  `Distributed Task Queue` 是一个简单的分布式框架,核心原理是**由任务服务器分配代码到各个任务执行的主机**,换句话说也就是**远程代码执行**,只需要在跑任务的机器上布置任务执行模块,无需做更多的操作即可实现分布式任务执行
  
##How to using Distributed Task Queue 

  `task_server` 是任务派遣服务器,包含有任务调度算法,使用HTTP 协议通信,支持的接口如下:<br/>
  
  slave machine 登陆接口(**slave_machine_login_password** 登陆密码,**slave_machine_ip** 主机IP ,**slave_machine_name** 主机名)<br/>
  返回值:slave_machine_id 服务器分配主机唯一ID <br/>
    http://127.0.0.1/login?slave_machine_login_password=&slave_machine_ip=&slave_machine_name=
  
  slave machine 退出接口(**slave_machine_id** 服务器分配主机唯一ID )<br/>
  返回值:success 或者error <br/>
  `TIPS : 退出之后任务系统会重新调度任务列表`<br/>
    http://127.0.0.1/logout?slave_machine_id=
  
  slave machine 任务领取接口(**slave_machine_id** 服务器分配主机唯一ID )<br/>
  返回值:任务详细信息<br/>
    http://127.0.0.1/dispatch?slave_machine_id=
  
  slave machine 任务报告接口(**slave_machine_id** 服务器分配主机唯一ID ,**slave_machine_execute_task_id** 任务ID ,**slave_machine_report** 详细报告信息)<br/>
  返回值:success 或者error <br/>
    http://127.0.0.1/report?slave_machine_id=&slave_machine_execute_task_id=&slave_machine_report=
    
  task dispatch server 管理接口(**task_dispatch_manager_password** 管理密码,**manager_operate_type** 管理命令,**manager_operate_argument** 管理命令参数列表)<br/>
  返回值:详细信息<br/>
  `TIPS : 限制在本地主机IP 管理`<br/>
    http://127.0.0.1/manager?task_dispatch_manager_password=&manager_operate_type=recovery&manager_operate_argument=
    
    manager_operate_type 支持命令:
    recovery                恢复task dispatch 服务器环境
    hot_backup              热备份task dispatch 数据(直接保存当前任务队列信息和主机列表)
    cold_backup             冷备份task dispatch 数据(等待所有主机执行完成任务再备份任务队列信息和主机列表)
    queue                   查看当前任务队列信息
    slave_machine_list      查看slave machine 列表
    
  task dispatch server 添加单任务接口(**task_dispatch_manager_password** 管理密码,**task_type** 任务类型,**task_eval_code** 任务代码)<br/>
  返回值:success 或者error<br/>
  `TIPS : 限制在本地主机IP 管理`<br/>
    http://127.0.0.1/add_task?task_dispatch_manager_password=&task_type=&task_eval_code=
  
  关于服务器相关的接口使用例子保存在`task_server.py test_case()` ,客户端相关的接口使用例子保存在`task_client.py task_slave` 类
  
  <br/>
  
  `task_client` 是执行任务的模块,只需要运行这个Python 即可,`task_client.py` 依赖`requests` ,在布置的时候记得需要安装它<br/>

  ![using_example](https://raw.githubusercontent.com/lcatro/Distributed-Task-Queue/master/readme_pic/using_example.png)

##Distributed Task Queue 的其他细节

####远程代码执行与任务执行

  执行任务的主机尽量简单布置,最好能让一个模块就能够做很多的事情,所以把代码当作是一个任务来执行是不错的解决方案,首先可以解决掉复杂的任务语句处理(就像Windows 的控制台解析批处理文件一样,会使得执行任务的模块变得臃肿而且拓展性不高),除去Python 可以执行分配下来的任务,几乎支持所有可以使用`eval()` 执行的语言(无论是Python 还是JavaScript 等),不受执行的平台所限制(即便是作为本地进程来运行的Python ,还是浏览器的前端JavaScript 都可以领取任务执行)<br/>

  派遣到主机分为两种任务:`single_task` (单任务)和`multiple_task` (多任务),`multiple_task` 相当于提供一组`single_task` 单任务列表,让执行任务的主机按照顺序来执行每个任务

---

####关于任务的负载均衡

  关于任务的负载均衡使用简单的遍历算法,让任务数较多的主机把自己的任务分配到任务比较少的主机,尽可能让每个执行任务的主机都能够平衡执行任务的压力,关于分配的算法写在`task_dispatch.dispatch()` 中

---

####任务服务器的维护备份

  任务服务器的备份方式分别为:**热备份**和**冷备份**,相关逻辑在`task_server.py task_dispatch.hot_backup_task_dispatch() 和task_dispatch.cold_backup_task_dispatch()` ,热备份直接把任务队列和当前登陆到任务服务器的主机列表保存到`database/task_dispatch.db` 中;冷备份需要等待所有执行任务的主机执行完所有任务之后进入停机状态,然后保存数据,使用冷备份的方式需要用`task_dispatch.recovery_task_dispatch()` 使dispatch 服务器重新启动.简单地说,热备份适用于定时备份当前的数据,冷备份适用于服务器进入停机维护状态
