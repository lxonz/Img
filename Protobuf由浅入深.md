---
title: Protobuf由浅入深还原数据定义
date: 2025-08-06 16:01:30
description: 在目前智能网联汽车中使用protobuf的频率还是比较高的，例如dds、Apollo自动驾驶、某些汽车数据本地TCP数据传输，都采用protobuf定义相关数据格式，一般来讲，车厂使用protobuf进行序列化之后，并不会对数据进行加密，因此如何将数据还原成我们看得懂的状态（反序列化）是一个值得分析事情。
categories: 
  - 汽车安全

---

# 一、背景

在目前智能网联汽车中使用protobuf的频率还是比较高的，例如dds、Apollo自动驾驶、某些汽车数据本地TCP数据传输，都采用protobuf定义相关数据格式，一般来讲，车厂使用protobuf进行序列化之后，并不会对数据进行加密，因此如何将数据还原成我们看得懂的状态（反序列化）是一个值得分析事情。

# 二、开发环境（C++）

## Mac os M1

别想了，因为架构问题，absl不太支持，环境起不来

```Go
Undefined symbols for architecture arm64:
  "_AbslInternalSpinLockWake", referenced from:
```

## Linux

```Plain
git clone https://github.com/protocolbuffers/protobuf.git
```

将abseil-cpp放进protobuf中third\_party下

```Plain
git clone https://github.com/abseil/abseil-cpp
```

回到protobuf目录下进行cmake ./ && make 然后 make install

![1.png](Protobuf由浅入深/1.png)

```SQL
root@iZ2ze7lesc0k6jujoawx4uZ:~/protobuf# protoc --version
libprotoc 23.0
```

查看版本，这就算环境安装成功了

# 三、demo编写

proto描述文件，定义数据格式

```SQL
syntax = "proto3";

message Test {
    string name = 1;
    int32 age = 2;
}
```

源文件

```SQL
#include "stdio.h"
#include "test.pb.h"
int main()
{
        Test *test = new Test();
        test->set_name("123");
        test->set_age(12345);
        printf("test  !!!!!\n");
        return 0;
}
```

使用protoc工具对proto文件进行编译，会生成两个文件

```Plain
protoc -I=./ --cpp_out=./ ./test.proto
 test.pb.cc  test.pb.h
```

编译

```Plain
g++ -o test test.cpp test.pb.cc -I /usr/local/include -L /usr/local/lib `pkg-config --cflags --libs protobuf`
```

问：为什么要做个C++的protobuf环境呢？

答：在没有相应的protobuf数据只有binary时，我们自己编写demo，通过对比ida中的区别，确定相关代码在ida中的格式进行比较与学习。

# 四、分析对比

## 案例1

ida显示

![2.png](./Protobuf由浅入深/2.png)

源码

```SQL
#include "stdio.h"
#include "test.pb.h"
int main()
{
        Test *test = new Test();
        test->set_name("123");
        test->set_age(12345);
        printf("test  !!!!!\n");
        return 0;
}
```

从ida中可以看出，设置“123”字符串与set\_age设置12345，能大致判断类型分别为string和int32，相应的就能写出对应的proto文件，但在实际业务中并不会这么简单，下面我们看一个相对复杂的。

## 案例2

![3.png](./Protobuf由浅入深/3.png)

案例2的逻辑不是在main中直接的，进到sendHeart看一下

![4.png](./Protobuf由浅入深/4.png)

这里首先可以看到有一个TopMessage的消息类型，他对应一个set\_message\_type的字段，并且该字段的值为1，至于类型未知，简单还原一下目前的proto文件可能长什么样子

```Bash
message TopMessage
{
    String message_type = 1;  //这里1不是赋值，是编号
}
```

![5.png](./Protobuf由浅入深/5.png)

这段代码多出MsgResult，值也为1，传入的v5为MsgResult的字符串指针，因此setBytes的部分，也属于MsgResult的字段，因此还原如下：

```Bash
message TopMessage
{
    int32 message_type = 1;
}
message MsgResult
{
    int32 result = 1;
    bytes error_code = 2;
}
```

![6.png](./Protobuf由浅入深/6.png)

进到receHeartResp进行分析，发现msg\_result是TopMessage的成员函数，并且打印了error\_code的值，根据这条信息，我们对proto的信息进行修改

```Bash
message TopMessage
{
    int32 message_type = 1;
    MsgResult msg_result = 2;
}
message MsgResult
{
    int32 result = 1;
    bytes error_code = 2;
}
```

由于message\_type只为1或者2，因此可以在proto文件中将其值固定，最终还原的proto文件如下所示：

```Bash
enum Messagetype
{
    NONE = 0; //enum中要求第一个值必须为0
    SIGNAL = 1;
    RESULT = 2;
}
message TopMessage
{
    Messagetype message_type = 1;
    MsgResult msg_result = 2;
}
message MsgResult
{
    int32 result = 1;
    bytes error_code = 2;
}
```

# 五、实战分析（长安深蓝）

我们以长安深蓝的binary Base\_process进行分析，确定USB\_MgrMessage的数据定义

![7.png](./Protobuf由浅入深/7.png)

```Bash
enum Messagetype
{
    NONE = 0;
}


message USB_MgrMessage
{
    Messagetype message_id = 1;
    Msg_usb_mgr_start_service_req msg_usb_mgr_start_service_req = 2;
    
}
message Msg_usb_mgr_start_service_req
{
    USB_MGR_startService_REQ usb_mgr_startService_REQ = 1;
}

message USB_MGR_startService_REQ
{
    uint64 type = 1;
}
```

从上图所示的代码中，我们能恢复的部分为这些，为了更快速的还原数据定义，直接在函数中，找到相关set进行逐步还原

![8.png](./Protobuf由浅入深/8.png)

先将所有mutable的字段全部筛选出来

![9.png](./Protobuf由浅入深/9.png)

**注：mutable方法可以直接修改内部字段的字段的值，理解成将二级指针赋予给当前的指针，我们就有操作别的数据字段的能力了**

```Bash
enum Messagetype
{
    START_SERVICE = 0;
    STATUS_NTF = 1;
    TIMEOUT_NTF = 2;
    STOP_SERVICE = 3;
    RECIEVE_MSG = 4;
    SEND_MSG = 5;
}


message USB_MgrMessage
{
    optional Messagetype message_id = 1;
    optional Msg_usb_mgr_start_service_req msg_usb_mgr_start_service_req = 2;
    optional Msg_usb_mgr_sock_connect_status_ntf msg_usb_mgr_sock_connect_status_ntf = 3；
    optional Msg_usb_mgr_timer_timeout_ntf msg_usb_mgr_timer_timeout_ntf = 4;
    optional Msg_usb_mgr_stop_service_req msg_usb_mgr_stop_service_req = 5;
    optional Msg_usb_mgr_recieve_msg_ntf msg_usb_mgr_recieve_msg_ntf = 6;
    optional Msg_usb_mgr_send_msg_ntf msg_usb_mgr_send_msg_ntf = 7;
}
message Msg_usb_mgr_start_service_req
{
    optional USB_MGR_startService_REQ usb_mgr_startService_req = 1;
}

message USB_MGR_startService_REQ
{
    optional uint64 type = 1;
}

message Msg_usb_mgr_sock_connect_status_ntf
{
    optional USB_MGR_SockConnectStatus_NTF usb_mgr_sockconnectstatus_ntf = 1;
}
message USB_MGR_SockConnectStatus_NTF
{
    optional unit32 direcrion = 1;
    optional unit32 status = 2;
}
message Msg_usb_mgr_timer_timeout_ntf
{
    optional USB_MGR_TimerTimeout_NTF usb_mgr_timertimeout_nef = 1;
}
message USB_MGR_TimerTimeout_NTF
{
    optional uint32 timeid = 1;
}
message Msg_usb_mgr_stop_service_req
{
    optional USB_MGR_stopService_REQ usb_mgr_stop_service_req = 1;
}
message USB_MGR_stopService_REQ
{ 
    optional uint64 type = 1;
}
message Msg_usb_mgr_recieve_msg_ntf
{
    optional USB_MGR_RecieveMsg_NTF recievemsg = 1;    
}
message USB_MGR_RecieveMsg_NTF
{
    optional string data = 1;
    optional uint32 len = 2;
}
message Msg_usb_mgr_send_msg_ntf
{
    optional USB_MGR_SendMsg_NTF sendmsg = 1;
}
message USB_MGR_SendMsg_NTF
{
    optional uint32 protocol_header = 1;
    optional string data = 2;
    optional uint32 len = 3;
    optional unit32 result = 4;
}
```

还原出来大致如上，由于没有真实数据，因此无法验证逆向的准确度，但整体上大差不差，看懂protobuf在ida中的表现形式，即可完成相关逆向工作。
