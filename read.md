# 记录

websocket接口流程:
- 从数据流中parse出JSON对象`command_list`, 为JSON的Array类型
- `command_list`中依次存着command/value pair, 所以循环的时候`i+=2`, 循环中取出相应的command和value, 根据command和value的类型执行相应逻辑

command类型有以下这些:
1. subscribe
  - 要求value必须是Array类型
  - value里的元素是`{"scene","renderer","transport-frame","source-metering","master-metering","output-activity","cpu-load"}`之一
2. unsubscribe
  - 同上
3. state: 要求value是一个JSON Object, 格式如下
```
{
    "auto-rotate-sources" :          Bool,
    "ref-pos" :                      [x,y,z], // 三维坐标
    "ref-rot" :                      [x,y,z,w], // 旋转矩阵?
    "master-volume" :                Number,
    "decay-exponent" :               Number,
    "amplitude-reference-distance" : Number,
    "processing" :                   Bool,
    "ref-pos-offset" :               [x,y,z],
    "transport-rolling" :            Bool,
    "frame" :                        Int,
    "tracker" :                      String,
}
```
4. new-src, 要求value是Array类型, Array里的元素为JSON Object, 格式如下
```
{
    "id": String, // source ID
    "name": String, // source name
    "model": String, // source model
    "audio-file": String,
    "port-number": Int,
    "channel": Int,
    "pos": [x,y,z],
    "rot": [x,y,z,w],
    "fixed": Bool,
    "volume": Number,
    "mute": Bool,
    "properties-file": String,
    // audio_file 和 port_number中只能有一个
    // port_number && channel中只能有一个
    
}
```
5. mod-src
6. del-src
7. load-scene
8. save-scene 未实现
9. time 未实现

**疑惑**: websocket/connection.h 687行, 此段代码的作用?
```cpp
if (!control)
{
    control = _controller.take_control();
}
```