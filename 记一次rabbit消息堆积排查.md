# 记一次rabbit消息堆积排查
## 现象
rabbit消息堆积100W+, 5台机器消费总速率2/s。

## 排查思路
1. 登陆显示打印jstack,并根据消费端代码`OutCaseUpdateListener.locateCase`定位到对应线程快照如下。
2. 线程状态RUNNABLE，说明未发生block或死锁。
3. 重复步骤1，发现每次快照都是一样。详细分析是等待mysql结果（MysqlIO）,怀疑是数据库slow-sql。
4. 检查数据库slow-sql-log，发现确实是慢查询，并且是由于隐式转换导致索引失效。

```
"SimpleAsyncTaskExecutor-1" #470 prio=5 os_prio=0 tid=0x00007fe90c8a7800 nid=0xb9a runnable [0x00007fe90acf0000]
   java.lang.Thread.State: RUNNABLE
        at java.net.SocketInputStream.socketRead0(Native Method)
        at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
        at java.net.SocketInputStream.read(SocketInputStream.java:171)
        at java.net.SocketInputStream.read(SocketInputStream.java:141)
        at com.mysql.jdbc.util.ReadAheadInputStream.fill(ReadAheadInputStream.java:101)
        at com.mysql.jdbc.util.ReadAheadInputStream.readFromUnderlyingStreamIfNecessary(ReadAheadInputStream.java:144)
        at com.mysql.jdbc.util.ReadAheadInputStream.read(ReadAheadInputStream.java:174)
        - locked <0x00000006f35c01d0> (a com.mysql.jdbc.util.ReadAheadInputStream)
        at com.mysql.jdbc.MysqlIO.readFully(MysqlIO.java:3008)
        at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:3469)
        at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:3459)
        at com.mysql.jdbc.MysqlIO.checkErrorPacket(MysqlIO.java:3900)
        at com.mysql.jdbc.MysqlIO.sendCommand(MysqlIO.java:2527)
        at com.mysql.jdbc.MysqlIO.sqlQueryDirect(MysqlIO.java:2680)
        at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2501)
        - locked <0x00000006f35b8fb8> (a com.mysql.jdbc.JDBC4Connection)
        at com.mysql.jdbc.PreparedStatement.executeInternal(PreparedStatement.java:1858)
        - locked <0x00000006f35b8fb8> (a com.mysql.jdbc.JDBC4Connection)
        at com.mysql.jdbc.PreparedStatement.execute(PreparedStatement.java:1197)
        - locked <0x00000006f35b8fb8> (a com.mysql.jdbc.JDBC4Connection)
        at com.zaxxer.hikari.pool.ProxyPreparedStatement.execute(ProxyPreparedStatement.java:44)
        at com.zaxxer.hikari.pool.HikariProxyPreparedStatement.execute(HikariProxyPreparedStatement.java)
        .....

        at com.xx.xx.xx.xx.task.OutCaseUpdateListener.locateCase(OutCaseUpdateListener.java:56)
        at com.xx.xx.xx.xx.task.OutCaseUpdateListener.onOverdue(OutCaseUpdateListener.java:38)
        .....
        at org.springframework.amqp.rabbit.listener.adapter.HandlerAdapter.invoke(HandlerAdapter.java:49)
        at org.springframework.amqp.rabbit.listener.adapter.MessagingMessageListenerAdapter.invokeHandler(MessagingMessageListenerAdapter.java:126)
        at org.springframework.amqp.rabbit.listener.adapter.MessagingMessageListenerAdapter.onMessage(MessagingMessageListenerAdapter.java:106)
        at org.springframework.amqp.rabbit.listener.AbstractMessageListenerContainer.doInvokeListener(AbstractMessageListenerContainer.java:822)
        at org.springframework.amqp.rabbit.listener.AbstractMessageListenerContainer.invokeListener(AbstractMessageListenerContainer.java:745)
        at org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer.access$001(SimpleMessageListenerContainer.java:97)
        at org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer$1.invokeListener(SimpleMessageListenerContainer.java:189)
        at org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer.invokeListener(SimpleMessageListenerContainer.java:1276)
        at org.springframework.amqp.rabbit.listener.AbstractMessageListenerContainer.executeListener(AbstractMessageListenerContainer.java:726)
        at org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer.doReceiveAndExecute(SimpleMessageListenerContainer.java:1219)
        at org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer.receiveAndExecute(SimpleMessageListenerContainer.java:1189)
        at org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer.access$1500(SimpleMessageListenerContainer.java:97)
        at org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer$AsyncMessageProcessingConsumer.run(SimpleMessageListenerContainer.java:1421)
        at java.lang.Thread.run(Thread.java:748)
```

## 解决方式：
1. 修复问题。
2. 临时扩容消费者。
3. 消费完堆积后缩容。
