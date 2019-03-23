---
title: Android IPC 机制概要介绍
date: 2018-11-07 15:20:27
tags: Android IPC

---
## 目录
1. 跨进程通信具像化描述
1. 序列化与反序列化基础
2. Linux 进程通信方式
2. 多进程应用场景
2. 如何开启多进程
3. 开启多进程后会出现什么问题
4. 解决这些问题的方式
5. 使用 AIDL 生成 Binder 进行跨进程通信详解
6. 各种方式对比
7. 参考资料

## 跨进程通信具像化描述

1. 生活化实例：

北京 你  <-----某种信使------> 朋友 上海

2. 具体到程序

运行在进程#1 微信 <-----某种信使----->  QQ 运行在进程#2


各种信使有不同的特性，可能有的用起来麻烦，有的用起来简单；有的用起来开销大，有的用起来开销小；有的有某种局限不能满足某些特定场景，有些就能满足这种场景。
**不同的跨进程通信方式，实际上是寻求最优和最合适的信使的过程**

## 序列化与反序列化基础
Android 实现序列化的方式主要有两种：Serializable 和 Parcellable，特别注意 静态成员变量 和 transient 关键字标注的成员变量不参与序列化与反序列化过程

### 一、Serializable
一、静态字段 serialVersionUID说明：

1. serialVersionUID 用来辅助序列化和反序列化过程，一般情况来讲，序列化过程中，serialVersionUID 也会被序列化，只有序列化后的数据中的 serialVersionUID 和当前类的serialVersionUID 相同才能够正常的被反序列化；如果不匹配，则会反序列化失败，报异常。
2. 如果我们不手动设置 serialVersionUID 值为固定值，那么默认该值是根据类结构自动生成它的 hash 值，这样如果随着版本更迭类结构发生了变动（增删字段等），系统会重新计算该值，导致反序列化时，因为 serialVersionUID 不匹配而失败。
3. 综上，我们在使用 Serializable 接口实现序列化时，总是应该将 serialVersionUID 设置为固定值，如 1L，这样就算类结构发生了改变，也能让部分值反序列化成功。

二、序列化和反序列化过程

1. 参考代码：

```java
/**
 * 序列化过程
 */
public void serial(User user) {
    try {
        ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("temp.txt"));
        out.writeObject(user);
        out.flush();
        out.close();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
/**
 * 反序列化过程
 */
public UserSerial deserial() {
    try {
        ObjectInputStream in = new ObjectInputStream(new FileInputStream("temp.txt"));
        UserSerial newUser = (UserSerial) in.readObject();
        in.close();
        return newUser;
    } catch (IOException e) {
        e.printStackTrace();
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    }
    return null;
}
```
2.自定义序列化与反序列化过程

```java
// 在需要序列化的类中，重写 readObject(Object obj) 和 writeObject() 即可，
// ObjectOutputStream 和 ObjectInputStream 会在序列化与反序列化时优先通过反射
// 调用目标类的 read 和 write 方法，没找到时才会使用默认的 read/write 方法
// 当类结构发生变动(增删了字段，改变了变量名等)的时候，自定义序列化和反序列化尤其方便字段对应
public class User implements Serializable {
	private void writeObject(ObjectOutputStream out) {
        ObjectOutputStream.PutField putFields = null;
        try {
            putFields = out.putFields();
            putFields.put("name", name);
            // ...
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
	/**
     * 此方法没有返回值，因为反序列化分为两个阶段，
     * 1. 通过反射创建对象-所以我们如果自定义反序列化过程，应该保留默认构造方法
     * 2. 通过default 或我们定义的读取方法给对象属性赋值
     * 本方法解决的是赋值过程，对象在调用本方法前已经生成好了，所以不用返回对象
     */
    private void readObject(ObjectInputStream in) {
        try {
            ObjectInputStream.GetField readFields = in.readFields();
            name = (String) readFields.get("name", "");
            // ... 
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

 ###  二、Parcellable
 
 ```java
 public class UserParcel implements Parcelable {
    private String name;
//    private Address address;

    public UserParcel() {
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    /**
     * 内容描述，一直返回 0 即可，只有当前对象中存在 fd 对象时返回 1
     */
    @Override
    public int describeContents() {
        return 0;
    }

    /**
     * 序列化过程
     */
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(name);
//        dest.writeParcelable(address);
    }

    /**
     * 反序列化过程
     */
    public static final Parcelable.Creator<UserParcel> CREATOR = new Parcelable.Creator<UserParcel>() {

        @Override
        public UserParcel createFromParcel(Parcel source) {
            return new UserParcel(source);
        }

        @Override
        public UserParcel[] newArray(int size) {
            return new UserParcel[size];
        }
    };

    private UserParcel(Parcel in) {
        name = in.readString();
//        address = in.readParcelable(Thread.currentThread().getContextClassLoader());
    }
}
 ```
 
 ## Linux 进程通信方式
 
 1. 管道(pipe)
 
    管道是一种具有两个端点的通信通道，一个管道实际上就是存在在内存中的文件，对这个文件操作需要两个已经打开文件的标记，他们代表管道的两端，也叫两个句柄。
    
    管道这种通讯方式有两种限制，一是数据只能单向流动，二是只能在具有亲缘关系的进程间使用。进程的亲缘关系通常是指父子进程关系。流管道s_pipe: 去除了第一种限制,可以双向传输.

2. 信号

    信号，用于接受某种事件发生，除了用于进程间通信之外，进程还可以发送信号给进程本身。
    主要作为进程间以及同一进程不同线程之间的同步手段。
    
3. 消息队列

    消息队列是消息的链表，有权限的进程可以向消息队列中添加消息，有读权限的进程可以读走消息队列的消息。
    
	消息队列克服了信号承载信息量少，管道只能承载无格式字节流及缓冲区大小受限等缺陷。

4. 共享内存

    共享内存就是映射一段能被其他进程所访问的内存，这段共享内存由一个进程创建，但多个进程都可以访问。共享内存是最快的 IPC 方式，它是针对其他进程间通信方式运行效率低而专门设计的。
    
5. 信号量

	信号量是一个计数器，可以用来控制多个进程对共享资源的访问。它常作为一种锁机制，防止某进程正在访问共享资源时，其他进程也访问该资源。因此，主要作为进程间以及同一进程内不同线程之间的同步手段。
    
5. Socket 套接字
 
## 多进程的应用场景
### 应用内开启多进程
1. 因为 Android 为每个应用分配的内存是有上限的，超过这个内存上限便会 OOM，而一些特殊的场景，例如大图片加载、视频播放、大数据块儿的读取，不可避免要耗费大量内存，此时我们就可以把这些耗费内存的功能放到一个单独的进程，系统会为这个进程分配独立的内存空间(多进程开发时，每个进程相当于一个独立应用)，从而减轻主进程内存负担
2. 和其他应用有数据交换的需要

### 理解应用和系统服务交互过程的基础
1. 各种 Manager 和 ManagerService的交互过程都是通过 IPC 方式
2. Activity 是如何管理的
3. Window 是如何管理的
4. Toast
5. RemoteViews
6. ...
 

## 如何开启多进程
给四大组件在 AndroidManifest 文件配置 android:process 属性，配置方式有两种

1. android:process=":remote" 以 ":" 号开头，属于私有进程，其他应用组件不可以和他跑在同一个进程中，完整进程名是 包名:remote
2. android:process="com.xxx.remote"，全局进程，其他应用可以通过 ShareUID 的方式，和我们声明的全局进程跑在同一个进程中

## 开启多进程后会出现什么问题
1. 静态成员变量和单例模式完全失效
2. 线程同步机制完全失效
3. SharedPreferences 可靠性下降(SharedPreferences 具有内存缓存，在内存中维护有一个 Map，如果采用 apply 的方式提交更改，不会及时存储到文件；此外，多进程并发的修改容易引起内容覆盖(同一个进程操作的是同一个 Share 对象不会有这个问题))
4. Application 多次创建

## 解决这些问题的方式

1. 共享文件
2. Bundle/通过 ActivityManagerService 这个 Binder 传输
3. CotentProvider/Binder
4. Socket
5. AIDL/Binder
6. Messenger/Binder

## 使用 AIDL 生成 Binder 进行跨进程通信详解

1. 查看代码示例
2. Android 之所以选择 Binder 而不是传统的 Linux 跨进程通信方式，是因为 Binder 有更优的性能和更少的开销
2. AIDL 是 Android 提供的实现 Binder 的便捷方式，但是不使用 AIDL ，遵照一定的格式，我们仍然可以手写 Binder 类
3. 探索双向通信如何进行- RemoteCallbackList
4. 线程控制与切换，所有 Binder 都是运行在 Binder 线程池中的，所以应该注意耗时任务与UI更新时的线程控制
5. bind 异常中断管理：onDisconnected、linkToDeath 监听
5. 说明：客户端和服务端应该维护相同的 aidl 文件备份，这样才能正确的进行序列化和反序列化过程，进行跨进程通信

## 各方式对比

Bundle: 能传输的数据类型有限

文件共享：并发访问容易出错，无法做到即时通信

AIDL：功能强大，支持一对多并发通信，支持实时通信，支持 RPC

Messenger: 受限的 AIDL，支持一对多串行通信，不支持 RPC/没法接受返回值

ContentProvider：受限的 AIDL，主要是提供对数据源的 CRUD 操作，其他不支持

Socket：功能强大，使用麻烦，开销较大，主要用于网络数据交换，一般不考虑



## 参考资料
1. [Binder 驱动实现原理](https://blog.csdn.net/yangwen123/article/details/9316987)
2. [Binder 简介及使用 AIDL 实现 Binder](https://www.jianshu.com/p/b9b3051a4ff6)
3. [Google Developer AIDL 简介](https://developer.android.com/guide/components/aidl?hl=zh-cn)



