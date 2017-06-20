# 服务器

> [项目地址](https://github.com/CrazyArcade/room-service)

## 技术选型

### 传输协议

#### UDP or TCP

原版泡泡堂是基于 UDP 在客户端互传数据的，这样做速度虽然很快，但是有一个弊端就是跨运营商传输慢，而且 UDP 不可靠，可能会出现人物瞬移的情况。不仅如此，使用 UDP 互传数据加大了反作弊的难度。

因此，最终我选择 TCP 作为传输协议，客户端先把数据传到服务器，然后服务器进行一定的判断，再回传。

#### Socket or WebSocket

现在还有一个问题，是选择原生的 Socket 还是 WebSocket。

不得不说，Socket 是目前游戏的主流方案。但是由于 cocos2d-x 本身没有对 Socket 进行封装，如果要做跨平台，就要针对不同平台提供不同的解决方案，开发难度较大。

而 cocos2d-x 本身提供了对 WebSocket 的支持，开发起来较为容易。而且，在数据量较小的情况下，二者数据包的大小基本没有区别。所以，考虑到开发效率，最终我选择 WebSocket。

#### Socket.io or Pure WebSocket

刚开始看到 cocos2d-x 还集成了 Socket.io，本来已经打算选择 Socket.io，可是一问助教，服务端只能用 c++，写。。虽然说，可以照着它的[协议](https://github.com/socketio/socket.io-protocol)撸一个 c++ 版的，不过还是那句话，开发效率。

所以我果断选择了 Pure WebSocket，使用的是[uWebSockets](https://github.com/uNetworking/uWebSockets)这个框架。

### 消息类型

选完了传输协议，现在还有一个问题，就是使用什么格式来传输数据，纯文本？JSON？或者二进制？

查了相关资料后发现，cocos2d-x 集成了 [rapidjson](https://github.com/miloyip/rapidjson) 和 [flatbuffers](https://github.com/google/flatbuffers)。

看了 flatbuffers 介绍，二进制、序列化速度快（接近原生），最终选择了 flatbuffers。

## 服务器架构

### 目录结构

```
data/       游戏的数据：地图、角色属性
include/    第三方项目
scripts/    编译脚本（已废弃）
src/        项目源码
.../api/    flatbuffers 的数据结构表，以及自动生成的头文件
.../model/  游戏对象：玩家、泡泡、道具
.../utils/  辅助方法：日志输出、uuid 生成压缩
```

### 文件分工

#### main.cpp

程序入口，初始化程序

#### common.h

程序共同需要的结构体、函数。

比如，Vec2, Size, random...

#### server.h / .cpp

对 uWebSockets 进行二次封装，实现了不同的消息类型执行不同函数的功能。主要原理是使用 `std::function` 和 `std::bind` 实现回调函数的功能。

#### room.h / .cpp

处理游戏的逻辑，玩家切换角色、切换准备状态、位置移动、放置泡泡等游戏的功能。

大部分的游戏逻辑都是在这里进行的，只有玩家的移动是在客户端本地运行的。这么做主要是因为，玩家移动的逻辑比较消耗资源（虽然目前支持 4 人游戏，消耗这点资源可以忽略）。而把玩家移动的逻辑放在客户端，服务器只对其的合法性进行验证，能减轻服务器的压力（省钱。

在这个文件可以看到和 [server.h / .cpp](#server.h-cpp) 进行绑定的函数。

``` c++
#define ON(code, funcName) server->on(code, CALLBACK(Room::funcName, this))

void Room::initServer() {
    ON(-2, onUserJoin);
    ON(-1, onUserLeave);

    ON(MsgType_GotIt, onGotIt);

    ON(MsgType_JoinRoom, onJoinRoom);
    ON(MsgType_UserChangeRole, onUserChangeRole);
    ON(MsgType_UserChangeStats, onUserChangeStats);
    ON(MsgType_RoomInfoUpdate, onRoomInfoUpdate);
    ON(MsgType_Chat, onChat);

    ON(MsgType_PlayerPosChange, onPlayerPosChange);
    ON(MsgType_PlayerSetBubble, onPlayerSetBubble);
}
```

其他的就是各种逻辑了。。

#### model/*

model 下的文件主要是游戏的对象，用来保存对象的状态和属性。

## 部署

因为 c/c++ 的特殊性，导致跨平台部署不是很方便。因此我采用的是 Docker(Moby) 容器化服务端。

[docker repo](https://hub.docker.com/r/crazyarcade/room-service/)

只要两行命令就可以运行这个服务器了（前提是要装好 Docker

```
docker pull crazyarcade/room-service
docker run -d --name crazyarcade-room-01 -p 4000:4000 crazyarcade/room-service
```

同时，因为使用了 Docker，大大提高了扩展性。例如，可以写一个调度程序来管理创建房间