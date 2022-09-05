# 高级篇（五）Redis 原理篇（三）通信协议


## 1. RESP 协议

Redis是一个CS架构的软件，通信一般分两步（不包括pipeline和PubSub）：

- 客户端（client）向服务端（server）发送一条命令
- 服务端解析并执行命令，返回响应结果给客户端

客户端发送命令的格式、服务端响应结果的格式必须有一个规范，这个规范就是**通信协议**。

在Redis中采用的是RESP（Redis Serialization Protocol）协议：

- Redis 1.2版本引入了RESP协议
- Redis 2.0版本中成为与Redis服务端通信的标准，称为RESP2
- Redis 6.0版本中，从RESP2升级到了RESP3协议，增加了更多数据类型并且支持6.0的新特性--客户端缓存

> 目前，默认使用的依然是RESP2协议，也是我们要学习的协议版本（以下简称RESP）。

在RESP中，通过首字节的字符来区分不同数据类型，常用的数据类型包括5种：



- 单行字符串：首字节是 ‘+’ ，后面跟上单行字符串，以CRLF（ "\r\n" ）结尾（二进制不安全：字符串本身不能带 CRLF）。例如返回"OK"： "+OK\r\n"
- 错误（Errors）：首字节是 ‘-’ ，与单行字符串格式一样，只是字符串是异常信息。例如："-Error message\r\n"
- 数值：首字节是 ‘:’ ，后面跟上数字格式的字符串，以CRLF结尾。例如：":10\r\n"
- 多行字符串：首字节是 ‘$’，后面跟字节数，CRLF 开始和结束，表示二进制安全的字符串，最大支持512MB：
  - 如果大小为0，则代表空字符串："$0\r\n\r\n"
  - 如果大小为-1，则代表不存在："$-1\r\n"
- 数组：首字节是 ‘*’，后面跟上数组元素个数，再跟上元素，CRLF结尾，元素数据类型不限。

## 2. 基于 python socket 模拟客户端

```python
import socket


class RedisClient:

    def __init__(self):
        self.sock = socket.socket()
        self.socket_writer = self.sock.makefile(mode='wb')  # makefile() 返回 file object 来操作 socket
        self.socket_reader = self.sock.makefile(mode='rb')

    def send_request(self, *args):
        """
        发送命令
        :param args:
        :return:
        """
        self.socket_writer.write(('*' + str(len(args)) + '\r\n').encode())

        for command in args:
            self.socket_writer.write(('$' + str(len(command.encode())) + '\r\n' + command + '\r\n').encode())

        self.socket_writer.flush()

    def handle_response(self):
        """
        处理响应
        :return:
        """
        prefix = self.socket_reader.read(1)
        if prefix == b'+':
            return self.socket_reader.readline().decode()
        if prefix == b'-':
            raise RuntimeError(self.socket_reader.readline().decode())
        if prefix == b':':
            return int(self.socket_reader.readline().decode())
        if prefix == b'$':
            length = int(self.socket_reader.readline().decode())
            if length == -1:
                return None
            if length == 0:
                return ''
            data = self.socket_reader.read(length).decode()
            self.socket_reader.read(2)  # 读取剩下的 \r\n
            return data
        if prefix == b'*':
            return self.read_bulk_string()

        raise RuntimeError('错误的数据格式')

    def read_bulk_string(self):
        """
        读数组响应
        :return:
        """
        resp = []
        # 获取数组大小
        length = int(self.socket_reader.readline().decode())
        if length <= 0:
            return None
        # 遍历读取元素
        for i in range(length):
            resp.append(self.handle_response())

        return resp

    def main(self):
        try:
            # 1. 建立连接
            self.sock.connect(('192.168.31.57', 6379))
            # 2. 发出请求
            self.send_request('set', 'name', '虎哥')
            # 3. 解析响应
            obj = self.handle_response()
            print(obj)

            # 2. 发出请求
            self.send_request('get', 'name')
            # 3. 解析响应
            obj = self.handle_response()
            print(obj)

            # 2. 发出请求
            self.send_request('set', 'name', 'bob')
            # 3. 解析响应
            obj = self.handle_response()
            print(obj)

            # 2. 发出请求
            self.send_request('get', 'name')
            # 3. 解析响应
            obj = self.handle_response()
            print(obj)
        finally:
            # 4. 释放连接
            self.sock.close()


if __name__ == '__main__':
    c = RedisClient()
    c.main()


# stdout    
> OK
> 
> 虎哥
> OK
>
> bob

```


---

> 作者: [黄波](https://boh5.com)  
> URL: https://boh5.com/posts/notes/databases/redis/itheima_redis_lesson/advanced/5-redis-principle-3-resp-protocol/  

