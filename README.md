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

offset 是偏移地址，end 偏移地址的结束。偏移地址不可以超过255，如果超过255需要补上一个0x00FF000000000000的min block作为填充才能计算出真实的控制地址。

计算真实的控制地址只需要将所有的offset和end相加求和就能够给出

$$ A = \sum^{N}_{0}{offset_i + end_i + 1}$$

N是min_block的块数, A是控制地址的大小.

例子:

       |0x01|0x0A|0xFF0000|0x000000| // 第一个 min block
       |0x00|0xFF|0x000000|0x000000| 
       |0x00|0xFF|0x000000|0x000000|
       |0x00|0x01|0X0000FF|0x000000| // 第四个 min block

       全部 offset + end 相加得到上面的命令实际想控制的真实地址是 0 到 522，其中1 ~ 11的地址为0xFF0000(红色)，12 ~ 267的地址为0x000000(关灯)，268 ~ 523的地址也是关灯，524 ~ 525的地址是(0x0000FF)蓝色
       526 = (1 + 10)+1 + (0 + 255) + 1 + (0 + 255) + 1 + (0 + 1) + 1 
       

       
  
**HSV: 3bytes**

(0x800000 &gt;&gt; 23) 等于0使用HSV，1使用CCT
  
H = ((HSV[2] & 0x007F) << 2) | ((HSV[1] & 0xC0) >> 6)

S = (((HSV[1] & 0x3F) << 1) | ((HSV[0] & 0x80) >> 7)) / 100
  
V = (HSV[0] & 0x7F) / 100

  
  
**delay: 2bytes**
  接收到命令之后的延时执行时间, 65535 * delaySlice (ms) 最小延时时间为0，最大延时时间2.7小时左右，延时的最小粒度为 delaySlice ms。
  
**fadeTime: 2bytes**
  从当前颜色到指定颜色的渐变时间，65535 * fadeTimeSlice (ms) 最小渐变时间为0，最长可以渐变 2.7小时，渐变的最小粒度为 fadeTimeSlice ms。


### 内置模式设置


```text
                |                            block        ...   |
                |       min block         |   
 modelId | port | start |  end |  modelId |...
 0       1      2       4      6          7
```

**modelId: 1byte**
  0x12      

**port: 1byte**

  0000 0011，代表端口1和端口2都设置block内置模式。
  0x00，表示所有端口应用同一条命令。

**start: 2bytes 和 end: 2bytes**

一个block里面有 len(block) / 3 个min block。

min block 里面包含了start(起始地址)、end(结束地址)、modelId(内置模式)。
