---
title: "RPC远程过程调用"
date: 2022-03-06T18:40:55+08:00
author : "LongyimHung"
draft: false
categories: ["RPC"]
tags: ["RPC"]
---

# RPC (Remote Procedure Call)

## 写在前面

随着业务发展，单机应用会越来越力不从心，势必会走向分布式。而从单机走向分布式，一台机器如何才能调用另一台机器上的方法呢？这就涉及到分布式通信方式，从最底层的TCP/UDP到耳熟能详的RestFul，RPC，这中间出现了很多令人眼花缭乱的专有名词：

1. **TCP/UDP**

   指TCP/UDP协议；数据以二进制进行传输，是其他协议的基础。

2. **CORBA**

   Common Object Request Broker Architecture, 古老而复杂的，支持面向对象的通信协议

3. **Web Service** (SOA SOAP RDDI WSDL...)

   基于http+xml的标准化Web API

4. **Restful**(Representational State Transfer)

   回归简单化本源的Web API的事实标准

   http+json

5. **RMI**

   Remote Method Invocation

   Java内部的分布式通信协议

6. **JMS** Java Message Service

   JavaEE中的消息框架标准，为很多MQ所支持

7. **RPC**

   远程过程调用，重点在于方法调用(不支持对象)，具体实现可以用RMI，Restful实现，但RMI不能跨语言，Restful效率太低。

 ## 什么是RPC？

这里讲的RPC不是指RPC框架，而是一种**分布式通信方式**：RPC，Remote Procedure Call，即远程过程调用，简单地说就是它能使我们可以像调用本地函数一样调用远程服务器的函数。

接下来从实践入手，从最简单的分布式通信开始，一点点完善，从中理解RPC的概念。

## RPC演进

### 从单机到分布式

随着业务发展，单机应用解决不了问题了，就可以扩展更多台机器，局域网内机器解决不了的，再广域网扩展更多台，让这些机器共同对外提供服务。所以这些机器之间，需要有关联，需要互相访问，互相通信，这就涉及到分布式通信。

机器直接通信的基础：二进制数据传输（TCP/IP）

但是操作系统是没有直接暴露TCP/IP接口的，而是通过Socket抽象了TCP/IP接口，所以我们可以使用Socket来实现远程方法调用。

### 最原始的分布式通信

![image-20220306122525394](https://s2.loli.net/2022/03/06/me6DYpzdhAKOTw9.png)

Server端：

 ```java
 public class Server {
 
     private static boolean running = true;
 
     public static void main(String[] args) throws Exception{
         ServerSocket ss = new ServerSocket(8888);
         while (running){
             Socket s = ss.accept();
             process(s);
             s.close();
         }
         ss.close();
     }
 
     private static void process(Socket s) throws Exception{
         InputStream in = s.getInputStream();
         OutputStream out = s.getOutputStream();
         DataInputStream dis = new DataInputStream(in);
         DataOutputStream dos = new DataOutputStream(out);
 
         int id = dis.readInt();
         IUserService service = new UserServiceImpl();
         User user = service.findUserById(id);
 
         dos.writeInt(user.getId());
         dos.writeUTF(user.getName());
         dos.flush();
 
     }
 
 ```

Client端：

```java
public class Client {
    public static void main(String[] args) throws Exception{
        Socket s = new Socket("127.0.0.1", 8888);
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        DataOutputStream dos = new DataOutputStream(baos);
        dos.writeInt(123);

        s.getOutputStream().write(baos.toByteArray());
        s.getOutputStream().flush();

        DataInputStream dis = new DataInputStream(s.getInputStream());
        int id = dis.readInt();
        String name = dis.readUTF();
        User user = new User(id, name);

        System.out.println(user);

        dos.close();
        s.close();
    }
}
```

这种方式非常不灵活，Server和Client必须了解被传输对象的一切细节，并且数据的传输过程和业务逻辑代码全部混在一起，一旦有地方需要改动，带来的影响是十分大的。

### 使用Stub把网络传输的过程屏蔽掉

使用静态代理的思想，把原来Client内容封装到Stub中，让Stub把网络通信部分对Client隐藏。

![image-20220306140422646](https://s2.loli.net/2022/03/06/niGKmsvLZ185IHT.png)

Server端：

保持不变

Client端：

```java
public class Client {
    public static void main(String[] args) throws Exception{
        Stub stub = new Stub();
        System.out.println(stub.findUserByID(123));
    }
}
```

Stub：

```java
public class Stub {

    public User findUserByID(Integer id) throws Exception{
        Socket s = new Socket("127.0.0.1", 8888);
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        DataOutputStream dos = new DataOutputStream(baos);
        dos.writeInt(id);

        s.getOutputStream().write(baos.toByteArray());
        s.getOutputStream().flush();

        DataInputStream dis = new DataInputStream(s.getInputStream());
        int receivedId = dis.readInt();
        String name = dis.readUTF();
        User user = new User(receivedId, name);

        dos.close();
        s.close();
        return user;
    }
}
```

但是这样简单的封装还存在很多问题，目前这个Stub只能代理findUserById这一个方法，只能返回一个类，如果有其他的方法也需要代理，就需要写更多的静态代理类，这是不合理的。

### 静态代理到动态代理

让Stub的getStub提供一个类，使用这个类远程访问方法。

Client端：

```java
public class Client {
    public static void main(String[] args) throws Exception{
        IUserService service = Stub.getStub();
        System.out.println(service.findUserById(123));
    }
}
```

Stub：

```java
public class Stub {

    public static IUserService getStub() {
        InvocationHandler h = new InvocationHandler() {
            //此处未对method方法进行判断，对接口中所有方法执行统一的invoke实现
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                Socket s = new Socket("127.0.0.1", 8888);
                ByteArrayOutputStream baos = new ByteArrayOutputStream();
                DataOutputStream dos = new DataOutputStream(baos);
                dos.writeInt((Integer) args[0]);

                s.getOutputStream().write(baos.toByteArray());
                s.getOutputStream().flush();

                DataInputStream dis = new DataInputStream(s.getInputStream());
                int receivedId = dis.readInt();
                String name = dis.readUTF();
                User user = new User(receivedId, name);

                dos.close();
                s.close();
                return user;
            }
        };
        //生成IUserService的动态代理类，对接口中的每个方法调用时执行代理方法
        Object o = Proxy.newProxyInstance(IUserService.class.getClassLoader(), new Class[]{IUserService.class}, h);
        System.out.println(o.getClass().getName());
        System.out.println(o.getClass().getInterfaces()[0]);
        return (IUserService)o;
    }
}
```

实际上，这里改进只是将静态代理换成动态代理，还没有达到我们对任意method支持的目的，还是写死的只调用一个方法。

下一版本的改进

### 改成对任意方法都支持

Client端：

```java
public class Client {
    public static void main(String[] args) throws Exception{
        IUserService service = Stub.getStub();
        System.out.println(service.findUserById(123));
    }
}
```

将method名，参数类型传给server端。

Stub：

```java
public class Stub {

    public static IUserService getStub() {
        InvocationHandler h = new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                Socket s = new Socket("127.0.0.1", 8888);

                ObjectOutputStream oos = new ObjectOutputStream(s.getOutputStream());

                //将方法名称，参数类型(防止只传方法名有重载问题)，参数值写给server端
                String methodName = method.getName();
                Class<?>[] parameterTypes = method.getParameterTypes();
                oos.writeUTF(methodName);
                oos.writeObject(parameterTypes);
                oos.writeObject(args);
                oos.flush();

                DataInputStream dis = new DataInputStream(s.getInputStream());
                int receivedId = dis.readInt();
                String name = dis.readUTF();
                User user = new User(receivedId, name);

                oos.close();
                s.close();
                return user;
            }
        };
        //生成IUserService的动态代理类，对接口中的每个方法调用时执行代理方法
        Object o = Proxy.newProxyInstance(IUserService.class.getClassLoader(), new Class[]{IUserService.class}, h);
        System.out.println(o.getClass().getName());
        System.out.println(o.getClass().getInterfaces()[0]);
        return (IUserService)o;
    }
}

```

Server端通过method名和参数类型确定需要被调用的方法，传递参数列表，将返回值写回Client端。

Server端：

```java
public class Server {

    private static boolean running = true;

    public static void main(String[] args) throws Exception{
        ServerSocket ss = new ServerSocket(8888);
        while (running){
            Socket s = ss.accept();
            process(s);
            s.close();
        }
        ss.close();
    }

    private static void process(Socket s) throws Exception{
        InputStream in = s.getInputStream();
        OutputStream out = s.getOutputStream();
        //进行相应的修改
        ObjectInputStream ois = new ObjectInputStream(in);
        DataOutputStream dos = new DataOutputStream(out);

        String methodName = ois.readUTF();
        Class[] parameterTypes = (Class[]) ois.readObject();
        Object[] args = (Object[]) ois.readObject();

        //使用反射获取实现类的指定方法，进行调用
        IUserService service = new UserServiceImpl();
        Method method = service.getClass().getDeclaredMethod(methodName, parameterTypes);
        User user = (User) method.invoke(service, args);

        dos.writeInt(user.getId());
        dos.writeUTF(user.getName());
        dos.flush();
    }
}
```

现在我们可以实现对同一接口里的所有方法的支持。但是，这里的service实现类还是写死的IUserService实现类，不能对其他接口支持。

### 不再拆解方法返回的对象，直接writeObject

其实我们前面还有一些遗留问题，比如user往回写时，我们是把user的所有属性传过去，现在我们直接使用writeObject写回user对象。

 只更改了Server端：

```java
public class Server {

    private static boolean running = true;

    public static void main(String[] args) throws Exception{
        ServerSocket ss = new ServerSocket(8888);
        while (running){
            Socket s = ss.accept();
            process(s);
            s.close();
        }
        ss.close();
    }

    private static void process(Socket s) throws Exception{
        InputStream in = s.getInputStream();
        OutputStream out = s.getOutputStream();
        //进行相应的修改
        ObjectInputStream ois = new ObjectInputStream(in);
        ObjectOutputStream oos = new ObjectOutputStream(out);

        String methodName = ois.readUTF();
        Class[] parameterTypes = (Class[]) ois.readObject();
        Object[] args = (Object[]) ois.readObject();

        //使用反射获取实现类的指定方法，进行调用
        IUserService service = new UserServiceImpl();
        Method method = service.getClass().getDeclaredMethod(methodName, parameterTypes);
        User user = (User) method.invoke(service, args);

        oos.writeObject(user);
        oos.flush();
    }
}
```

继续改进

### 改成对任意接口任意方法都支持

我们想要让Stub帮我们生成更多类型的接口，延续之前的思路，我们这次把className也写到Server端去。

Client端：

```java
public class Client {
    public static void main(String[] args) throws Exception{
        IUserService service = (IUserService) Stub.getStub(IUserService.class);
        System.out.println(service.findUserById(123));
    }
}
```

将Class名字写给Server端，返回值改为Object

Stub：

```java
public class Stub {

    public static Object getStub(Class clazz) {
        InvocationHandler h = new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                Socket s = new Socket("127.0.0.1", 8888);

                ObjectOutputStream oos = new ObjectOutputStream(s.getOutputStream());

                String clazzName = clazz.getName();
                String methodName = method.getName();
                Class<?>[] parameterTypes = method.getParameterTypes();

                oos.writeUTF(clazzName);
                oos.writeUTF(methodName);
                oos.writeObject(parameterTypes);
                oos.writeObject(args);
                oos.flush();

                //序列化传输
                ObjectInputStream ois = new ObjectInputStream(s.getInputStream());
                Object o = ois.readObject();

                oos.close();
                s.close();
                return o;
            }
        };
        //生成IUserService的动态代理类，对接口中的每个方法调用时执行代理方法
        Object o = Proxy.newProxyInstance(clazz.getClassLoader(), new Class[]{clazz}, h);
        return o;
    }
}
```

读到className从服务注册表找到具体的类(或使用注册中心，或直接spring注入)

Server端：

```java
public class Server {

    private static boolean running = true;

    public static void main(String[] args) throws Exception{
        ServerSocket ss = new ServerSocket(8888);
        while (running){
            Socket s = ss.accept();
            process(s);
            s.close();
        }
        ss.close();
    }

    private static void process(Socket s) throws Exception{
        InputStream in = s.getInputStream();
        OutputStream out = s.getOutputStream();
        ObjectInputStream ois = new ObjectInputStream(in);
        ObjectOutputStream oos = new ObjectOutputStream(out);

        String clazzName = ois.readUTF();
        String methodName = ois.readUTF();
        Class[] parameterTypes = (Class[]) ois.readObject();
        Object[] args = (Object[]) ois.readObject();

        Class clazz = null;

        //从服务注册表找到具体的类
        clazz = findClassByName(clazzName);

        Method method = clazz.getMethod(methodName, parameterTypes);
        Object o = method.invoke(clazz.getDeclaredConstructor().newInstance(), args);

        oos.writeObject(o);
        oos.flush();
    }
    /**
     * 根据className寻找class，过程有各种各样，此处简单直接赋值
     */
    private static Class findClassByName(String clazzName) throws Exception{
        if (Class.forName(clazzName).isAssignableFrom(UserServiceImpl.class)) {
            return UserServiceImpl.class;
        }
        return Object.class;
    }
}
```

到现在，我们的Stub已经可以支持所有的接口以及所有的方法了。现在Client只需要知道Server端提供了什么方法，将对应的Class传给Stub，得到一个动态生成的类，再像本地调用一样，调用这个类的相应方法，就能得到他想要的返回了，中间的细节已经被完全屏蔽掉了。

这就是Remote Procedure Call！

## RPC序列化框架

现在这个demo用的序列化方式是JDK的Serializable。

还有其他流行序列化框架：

1.  Google protobuf

2. Facebook Thrift

3. Hessian

4. kyro

5. fst

6. json序列化框架

   Jackson

   Google Gson

   Ali FastJson

7. xmlrpc(xstream)

8. ...

## 图解RPC

看完上面的demo后，再来看看RPC的图解：

![image-20220306174233579](https://s2.loli.net/2022/03/06/j9xwSRclprzIvb4.png)

![image-20220306174220985](https://s2.loli.net/2022/03/06/VfvWXlhRnQzd1pa.png)

使用注册中心的情况：

![image-20220306174344019](https://s2.loli.net/2022/03/06/kFYpqXmKoQhZnCg.png)





最后一个小demo附在后面：

## 选用比较简单的Hessian序列化框架替换掉JDK的Serializable。

HassianUtil

```java
public class HessianUtil {

    public static byte[] serialize(Object o) throws Exception {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        Hessian2Output output = new Hessian2Output(baos);
        output.writeObject(o);
        output.flush();
        byte[] bytes = baos.toByteArray();
        baos.close();
        output.close();

        return bytes;
    }

    public static Object deserialize(byte[] bytes) throws Exception {
        ByteArrayInputStream bais = new ByteArrayInputStream(bytes);
        Hessian2Input input = new Hessian2Input(bais);
        Object o = input.readObject();
        bais.close();
        input.close();
        return o;
    }

}

```

Client

```java
public class Client {
    public static void main(String[] args) throws Exception{
        IUserService service = (IUserService) Stub.getStub(IUserService.class);
        System.out.println(service.findUserById(123));
    }
}
```

Stub

```java
public class Stub {

    public static Object getStub(Class clazz) {
        InvocationHandler h = new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                Socket s = new Socket("127.0.0.1", 8888);

                Hessian2Output oos = new Hessian2Output(s.getOutputStream());

                String clazzName = clazz.getName();
                String methodName = method.getName();
                Class<?>[] parameterTypes = method.getParameterTypes();

                oos.writeString(clazzName);
                oos.writeString(methodName);
                oos.writeObject(parameterTypes);
                oos.writeObject(args);
                oos.flush();

                //序列化传输
                Hessian2Input ois = new Hessian2Input(s.getInputStream());
                Object o = HessianUtil.deserialize(ois.readBytes());

                oos.close();
                s.close();
                return o;
            }
        };
        //生成IUserService的动态代理类，对接口中的每个方法调用时执行代理方法
        Object o = Proxy.newProxyInstance(clazz.getClassLoader(), new Class[]{clazz}, h);
        return o;
    }
}
```

Server

```java
public class Server {

    private static boolean running = true;

    public static void main(String[] args) throws Exception{
        ServerSocket ss = new ServerSocket(8888);
        while (running){
            Socket s = ss.accept();
            process(s);
            s.close();
        }
        ss.close();
    }

    private static void process(Socket s) throws Exception{
        InputStream in = s.getInputStream();
        OutputStream out = s.getOutputStream();
        Hessian2Input ois = new Hessian2Input(in);
        Hessian2Output oos = new Hessian2Output(out);

        String clazzName = ois.readString();
        String methodName = ois.readString();
        Class[] parameterTypes = (Class[]) ois.readObject();
        Object[] args = (Object[]) ois.readObject();

        Class clazz = null;

        //从服务注册表找到具体的类
        clazz = findClassByName(clazzName);;

        Method method = clazz.getMethod(methodName, parameterTypes);
        Object o = method.invoke(clazz.getDeclaredConstructor().newInstance(), args);

        oos.writeBytes(HessianUtil.serialize(o));
        oos.flush();
    }
    /**
     * 根据className寻找class，过程有各种各样，此处简单直接赋值
     */
    private static Class findClassByName(String clazzName) throws Exception{
        if (Class.forName(clazzName).isAssignableFrom(UserServiceImpl.class)) {
            return UserServiceImpl.class;
        }
        return Object.class;
    }
}
```



## Reference

本文大部分代码，描述基于视频 [马士兵：30行代码透彻解析RPC，包你听懂](https://www.bilibili.com/video/BV17Z4y1s7cG)
