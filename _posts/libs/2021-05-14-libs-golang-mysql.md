---
title: golang中的mysql driver是怎样接收多条结果的？
date: 2021-05-14 09:26:00 +0800
category: Libs
---
当golang中使用sql查询数据库获取多条结果时，一般采用db.Query() + rows.Next() + rows.Scan()三个函数调用，那每个函数完成的具体功能到底是啥呢？

### 接口与driver
golang中数据库的实现方式为：[database/sql](https://golang.org/pkg/database/sql/)提供公共的数据库操作接口，具体的数据库驱动由各个数据库自己实现，比如[go-sql-driver/mysql](https://github.com/go-sql-driver/mysql)，关系图如下：<br/>
![sql-and-database.png](/assets/images/lib/sql-and-database.png)<br/>
图片来源：<https://draveness.me/golang/docs/part4-advanced/ch09-stdlib/golang-database-sql/>

### mysql客户端-服务端交互
```
1. 三次握手建立 TCP 连接。

2. 建立 MySQL 连接，也就是认证阶段。
    服务端 -> 客户端：发送握手初始化包 (Handshake Initialization Packet)。
    客户端 -> 服务端：发送验证包 (Client Authentication Packet)。
    服务端 -> 客户端：认证结果消息。

3. 认证通过之后，客户端开始与服务端之间交互，也就是命令执行阶段。
    客户端 -> 服务端：发送命令包 (Command Packet)。
    服务端 -> 客户端：发送回应包 (OK Packet, or Error Packet, or Result Set Packet)。

4. 断开 MySQL 连接。
    客户端 -> 服务器：发送退出命令包。

5. 四次握手断开 TCP 连接。
```
引用：<https://gohalo.me/post/mysql-protocol.html>

### 响应报文格式
响应报文采用统一格式，如下：<br/>
```
+-------------------+--------------+---------------------------------------------------+
|      3 Bytes      |    1 Byte    |                   N Bytes                         |
+-------------------+--------------+---------------------------------------------------+
|<= length of msg =>|<= sequence =>|<==================== data =======================>|
|<============= header ===========>|<==================== body =======================>|
```
>说明：
1. 3字节：报文长度；
2. 1字节：序号，每次客户端发起请求时，序号值都会从 0 开始计算。当一个请求由多个响应报文组成时，用来确定报文顺序，如果接收报文失序，则报错。
3. N字节：具体数据部分。

引用：<https://gohalo.me/post/mysql-protocol.html>

### ResultSet报文
响应报文包含三种：OK报文, Error报文, Result Set报文，下面主要说明最复杂的ResultSet报文：
![sql-response-pack.png](/assets/images/lib/sql-response-pack.png)
>说明：
图表包含了sql请求的所有响应情况，其中最左边方框+EOF/ERR报文就表示一个完整的ResultSet报文，主要包括五组报文：
1. ResultSetHeaderPacket：结果集头报文，包含列的个数信息，报文个数=1；
2. ColumnsPackets：列说明报文，包含每个列的具体说明，报文个数=column_count；
3. EOFPacket：表示列说明报文结束，报文个数=1；
4. RowsPackets：行内容报文，包含一行的具体内容，报文个数=row_num；
5. EOFPacket/ERRPacket：表示整个Result Set报文的结束，报文个数=1；

参考：<https://dev.mysql.com/doc/internals/en/com-query-response.html#packet-ProtocolText::Resultset>

### ResultSet报文例子
```
01 00 00 01 01|27 00 00    02 03 64 65 66 00 00 00    .....'....def...
11 40 40 76 65 72 73 69    6f 6e 5f 63 6f 6d 6d 65    .@@version_comme
6e 74 00 0c 08 00 1c 00    00 00 fd 00 00 1f 00 00|   nt..............
05 00 00 03 fe 00 00 02    00|1d 00 00 04 1c 4d 79    ..............My
53 51 4c 20 43 6f 6d 6d    75 6e 69 74 79 20 53 65    SQL Community Se
72 76 65 72 20 28 47 50    4c 29|05 00 00 05 fe 00    rver (GPL)......
00 02 00                                              ...
```
说明：
包含5种报文：
1. length = 01 00 00, sequence_id = 01([ResultSetHeaderPacket](https://dev.mysql.com/doc/internals/en/integer.html#packet-Protocol::LengthEncodedInteger))

    + column_count = 01 (1)

2. length = 27 00 00, sequence_id = 02([ColumnsPackets](https://dev.mysql.com/doc/internals/en/com-query-response.html#packet-Protocol::ColumnDefinition))

    + catalog = 03 64 65 66 ("def")

    + schema = 00 ("")

    + table = 00 ("")

    + org_table = 00 ("")

    + name = 11 40 40 76 65 72 73 69 6f 6e 5f 63 6f 6d 6d 65 6e 74 ("@@version_comment")

    + org_name = 00 ("")

    + filler_1 = 0c

    + character_set = 08 00 (latin1_swedish_ci)

    + column_length = 1c 00 00 00 (28)

    + column_type = fd (Protocol::MYSQL_TYPE_VAR_STRING)

    + flags = 00 00

    + decimals = 1f (127)

    + filler_2 00 00

3. length = 05 00 00, sequence_id = 03([EOFPacket](https://dev.mysql.com/doc/internals/en/packet-EOF_Packet.html))

    + fe (EOF indicator)

    + warning_count = 00 00 (0)

    + status_flags = 02 00 (Protocol::StatusFlags = SERVER_STATUS_AUTOCOMMIT )

4. length = 05 00 00, sequence_id = 04([RowsPackets](https://dev.mysql.com/doc/internals/en/com-query-response.html#packet-ProtocolText::ResultsetRow))

    + (ProtocolText::ResultsetRow)

    + 1c 4d 79 53 51 4c 20 43 6f 6d 6d 75 6e 69 74 79 20 53 65 72 76 65 72 20 28 47 50 4c 29 (length = 28, string = "MySQL Community Server (GPL)")

5. length = 05 00 00, sequence_id = 05([EOFPacket](https://dev.mysql.com/doc/internals/en/packet-EOF_Packet.html))

    + fe (EOF indicator)

    + warning_count = 00 00 (0)

    + status_flags = 02 00 (Protocol::StatusFlags = SERVER_STATUS_AUTOCOMMIT )

参考：<https://dev.mysql.com/doc/internals/en/protocoltext-resultset.html>

### db.Query
```                                                                             
接口调用：db.Query ===> db.QueryContext ===> db.query ===> db.conn(获取连接) ===> db.queryDC ===> db.ctxDriverQuery ===> 
driver调用：driver.QueryContext ===> driver.query ===> driver.writeCommandPacketStr(发送sql请求) ===> driver.readResultSetHeaderPacket(读取并解析ResultSetHeaderPacket) ===> driver.readColumns(读取并解析ColumnsPackets) <===> until EOFPacket(读到EOF报文表示列解析完成).
```

### rows.Next
```                                                                             
接口调用：rows.Next ===> rows.nextLocked 
driver调用：rows.Next ===> rows.readRow ===> rows.readRowPacket(读取并解析RowsPackets) <===> until EOFPacket((读到EOF报文表示行解析完).
```

### rows.Scan
```                                                                             
接口调用：rows.Scan ===> convertAssignRows(将数据库行转为golang内部结构体).
```