1. Decoder是指入栈Handler，指从网络IO-->用户处理器,read()发起
2. Encoder是指出栈Handler，指从用户处理器-->网络IO,write()结束
3. `LengthFieldBasedFrameDecoder`中常见的几个属性

`lengthFieldOffset`: 指偏移多少字节才能得到**长度字段**

`lengthFieldLength`: **长度字段**占用多少字节

`lengthAdjustment`:  **长度字段值** 调整(添加或删除)多少个字节才能得到正确的**数据长度**

`initialBytesToStrip`:  需要略过的字节数


eg:
```
解码前 (16 bytes)                               解码后 (13 bytes)
+------+--------+------+----------------+      +------+----------------+
| HDR1 | Length | HDR2 | Actual Content |----->| HDR2 | Actual Content |
| 0xCA | 0x0010 | 0xFE | "HELLO, WORLD" |      | 0xFE | "HELLO, WORLD" |
+------+--------+------+----------------+      +------+----------------+
```

    lengthFieldOffset   =  1;（因为长度字段前开头有一个字节的HDR1，所以需要偏移一个字节）
    lengthFieldLength   =  2;（长度字段占用Length两个字节长度）
    lengthAdjustment    = -3;（长度字段值=0x10=16,实际长度是12，所以需要-3）
    initialBytesToStrip =  3;（需要略过HDR1，Length，HDR2 3个字节）