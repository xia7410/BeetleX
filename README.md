# BeetleX
beetleX是基于dotnet core实现的轻量级高性能的TCP通讯组件，使用方便、性能高效和安全可靠是组件设计的出发点！开发人员可以在Beetlx组件的支持下快带地构建高性能的TCP通讯服务程序，在安全通讯方面只需要简单地设置一下SSL信息即可实现可靠安全的SSL服务。

### [基于BeetleX实现的官网](http://www.ikende.com)

### 使用方便性
beetleX网络流读写是基于Stream标准来构建，仅仅基于Stream的基础读写对于应用者来说还是过于繁琐；组件为了更方便进行网络数据处理在Stream的基础之上扩展了一系列的读写规则：ReadLine、ReadInt、WriteLine、WriteInt等一系列简便方法，在这些方法的支持下使用者就可以更轻松地处理数据；为了在网络通讯中更好的兼容其他平台协议以上方法都兼容Big-Endian和Little-Endian不同方式。为了更好地利用现有序列化组件，组件通过IPacket接口规范消息扩展，通过实现不同的Packet解释器，即可以实现基于Protobuf,json和Msgpack等方式的对象数据传输。
### 高性能特性
beetleX的高性能是建立在内部一个数据流处理对象PipeStream，它是构建在Stream标准之上；它和.NET内置的NetworkStream最大的差别是PipeStream的读写基于SocketAsyncEventArgs实现，这正是在编写高性能网络数据处理所提倡的模式。PipeStream不仅在网络数据处理模式上有着性能的优势，在内存读写上和MemoryStream也有着很大的区别；由于PipeStream的内存块是以一个基于链表的SocketAsyncEventArgs Buffer 组成，因此PipeStream在写入大数据的情况并不存在内存扩容和复制的问题；因为PipeStream基础内存是SocketAsyncEventArgs Buffer，所以在数据和网络缓存读写并不存在内存块复制过程。如果在应用中中使用PipeStream相应的BinaryReader和IBinaryWriter读写规范，那大部分数据处理基本不存在内存复制过程，从而让数据处理性能更高效。

以下是PipeStream的结构：
![PipeStream](https://i.imgur.com/16wjO0R.png) 

### 性能
beetleX的性能到底怎样呢，以下简单和DotNetty进行一个网络数据交换的性能测试,分别是1K,5K和10K连接数下数据请求并发测试
#### DotNetty测试代码
```
        public override void ChannelRead(IChannelHandlerContext context, object message)
        {
            var buffer = message as IByteBuffer;
            context.WriteAsync(message);
        }
```
#### Beetlex 测试代码
```
        public override void SessionReceive(IServer server, SessionReceiveEventArgs e)
        {
            server.Send(e.Stream.ToPipeStream().GetReadBuffers(), e.Session);
            base.SessionReceive(server, e);
        }
```
### 测试结果
#### 1K connections
![PipeStream](https://i.imgur.com/XlKeV9c.png) 
![PipeStream](https://i.imgur.com/JIaqGPD.png) 
#### 5K connections
![PipeStream](https://i.imgur.com/KzeUtOv.png) 
![PipeStream](https://i.imgur.com/ZBndSS6.png) 
#### 10K connections
![PipeStream](https://i.imgur.com/bc3UMeM.png) 
![PipeStream](https://i.imgur.com/VrffHGR.png) 
### 构建TCP Server
```
    class Program : ServerHandlerBase
    {
        private static IServer server;

        public static void Main(string[] args)
        {
            NetConfig config = new NetConfig();
            //ssl
            //config.SSL = true;
            //config.CertificateFile = @"c:\ssltest.pfx";
            //config.CertificatePassword = "123456";
            server = SocketFactory.CreateTcpServer<Program>(config);
            server.Open();
            Console.Write(server);
            Console.Read();
        }
        public override void SessionReceive(IServer server, SessionReceiveEventArgs e)
        {
            string name = e.Stream.ToPipeStream().ReadLine();
            Console.WriteLine(name);
            e.Session.Stream.ToPipeStream().WriteLine("hello " + name);
            e.Session.Stream.Flush();
            base.SessionReceive(server, e);
        }
    }
```
### 构建TCP Client
```
    class Program
    {
        static void Main(string[] args)
        {
            TcpClient client = SocketFactory.CreateClient<TcpClient>("127.0.0.1", 9090);
            //ssl
            //TcpClient client = SocketFactory.CreateSslClient<TcpClient>("127.0.0.1", 9090, "localhost");
            while (true)
            {
                Console.Write("Enter Name:");
                var line = Console.ReadLine();
                client.Stream.ToPipeStream().WriteLine(line);
                client.Stream.Flush();
                var reader = client.Read();
                line = reader.ToPipeStream().ReadLine();
                Console.WriteLine(line);
            }
            Console.WriteLine("Hello World!");
        }
    }
```
### 异步Client
```
    class Program
    {
        static void Main(string[] args)
        {
            AsyncTcpClient client = SocketFactory.CreateClient<AsyncTcpClient>("127.0.0.1", 9090);
            //SSL
            //AsyncTcpClient client = SocketFactory.CreateSslClient<AsyncTcpClient>("127.0.0.1", 9090, "serviceName");
            client.ClientError = (o, e) =>
            {
                Console.WriteLine("client error {0}@{1}", e.Message, e.Error);
            };
            client.Receive = (o, e) =>
            {
                Console.WriteLine(e.Stream.ToPipeStream().ReadLine());
            };
            var pipestream = client.Stream.ToPipeStream();
            pipestream.WriteLine("hello henry");
            client.Stream.Flush();
            Console.Read();
        }
    }
```
### 实现一个Protobuf对象解释器
```
    public class Packet : FixedHeaderPacket
    {
        public Packet()
        {
            TypeHeader = new TypeHandler();
        }

        private PacketDecodeCompletedEventArgs mCompletedEventArgs = new PacketDecodeCompletedEventArgs();

        public void Register(params Assembly[] assemblies)
        {
            TypeHeader.Register(assemblies);
        }

        public IMessageTypeHeader TypeHeader { get; set; }

        public override IPacket Clone()
        {
            Packet result = new Packet();
            result.TypeHeader = TypeHeader;
            return result;
        }

        protected override object OnReader(ISession session, PipeStream reader)
        {
            Type type = TypeHeader.ReadType(reader);
            int bodySize = reader.ReadInt32();
            return reader.Stream.Deserialize(bodySize, type);
        }

        protected override void OnWrite(ISession session, object data, PipeStream writer)
        {
            TypeHeader.WriteType(data, writer);
            MemoryBlockCollection bodysize = writer.Allocate(4);
            int bodyStartlegnth = (int)writer.CacheLength;
            ProtoBuf.Meta.RuntimeTypeModel.Default.Serialize(writer.Stream, data);
            bodysize.Full((int)writer.CacheLength - bodyStartlegnth);
        }
    }
```

