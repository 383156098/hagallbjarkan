### 获取MODE ID

```text
|opcode|            modelId0 - modelIdN             |
0      1                                            N
```

**opcode: 0xA1**

**model id 0 \~ model id N:**  
  modelId 代表设备有什么能力 (开关，调色，色温，等等.),一个modelId占用一个byte，每个设备只能拥有255个modelId

### 设置时间粒度

```text
       |             block              |
|opcode|modelId|delaySlice|fadeTimeSlice|...
0      1       2          4             6
```

**opcode: 0xA1**

**block:**
  每个block包含了以下的字段，一条数据可以有多个block
  modelId: 需要设备时间粒度的modelId
  delaySlice: delay字段的时间粒度
  fadeTimeSlice: fadeTime字段的时间粒度

### 设置开关和亮度

```text
        |        block            | ....
|modelId|port|power|delay|fadeTime| ....
0       1    2     3     5        7
```

**modelId: 1 bytes**
  0x10
  
**port: 1 bytes**
  每个bit代表一个端口,  例如: 0000 0001 表示只控制端口 1。0000 0011 表示端口1和端口2同时控制。
  0x00, 表示所有端口应用同一条命令。
  block 的块数必须与控制端口一直，例如: 0000 0011 那么block就有2块，每块的都包含了power、delay、fadeTime。第一的block包含了端口1的控制参数，第二的block包含了端口2的控制参数
  
**power: 1 byte**
  (power & 0x80) &gt;&gt; 7 等于 1是开，0是关，如果是关亮度值可以无视。
  (power & 0x7F) 是开起时的亮度
  
**delay: 2 bytes**
  接收到命令之后的延时执行时间, 65535 \* delaySlice (ms) 最小延时时间为0，最大延时时间2.7小时左右，延时的最小粒度为 delaySlice ms。
  
**fadeTime: 2 bytes**
  从当前亮度到指定亮度的渐变时间，65535 \* fadeTimeSlice / abs(current_power - power & 0x7F) 最小渐变时间为0，最长可以渐变 2.7小时，渐变的最小粒度为 fadeTimeSlice ms。

### 设置颜色

```text
                |                            block                                 ...   |
                      |                   min block                              |   
 modelId | port | len | offset | end |   hsv   |      delay       |      fadeTime | ...
 0       1      2     4        5     6         9       12bit              12bit   12
```

**modelId: 1 bytes**
  0x11
  
**port: 1 bytes**
  每个bit代表一个端口, 例如: 0000 0001 表示只控制端口 1。0000 0011 表示端口1和端口2同时控制。
  0x00，表示所有端口应用同一条命令。

       
**len: 2 byte**

       len是block的数据长度，block的数量应该和port一致。
       如果port是0x00，那么block只会存在一个，所有端口都应用这一个block的数据。
       如果port不为0x00，那么block的数量应该和port是一致。
       
       len最坏的情况下可以控制8191个点(每个点颜色不一样情况下)

**offset: 1byte 和 end: 1byte**

       |0x01|0x0A|RED|
       |0x00|0xFF|0x00|
       |0x00|0xFF|0X00|
       |0x00|0x01|BLUE|
       offset 是偏移地址，end是实际地址地址偏移。第一个min block的end加上剩余的min block的offset和end算出真实需要控制的地址大小。
       
       
$$\sum^{x \to \infty}_{y \to 0}{\frac{x}{y}}$$
  
**HSV: 3 byte**
  (0x800000 &gt;&gt; 23) 等于0使用HSV，1使用CCT
  H = ((HSV[2] & 0x007F) << 2) | ((HSV[1] & 0xC0) >> 6)
  S = (((HSV[1] & 0x3F) << 1) | ((HSV[0] & 0x80) >> 7)) / 100
  V = (HSV[0] & 0x7F) / 100

  
  
**delay: 2 bytes**
  接收到命令之后的延时执行时间, 65535 * delaySlice (ms) 最小延时时间为0，最大延时时间2.7小时左右，延时的最小粒度为 delaySlice ms。
  
**fadeTime: 2 bytes**
  从当前颜色到指定颜色的渐变时间，65535 * fadeTimeSlice (ms) 最小渐变时间为0，最长可以渐变 2.7小时，渐变的最小粒度为 fadeTimeSlice ms。
