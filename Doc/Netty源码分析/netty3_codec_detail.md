上篇提到 了`ByteToMessageDecoder`中的调用流程，其中`decode`作为抽象方法，留给具体子类实现。

本篇我们关注几个核心的子类实现：

# 一. `FixedLengthFrameDecoder`

固定长度的解码器，如下图会将不定长度的数组，组织成固定长度的数组。

> ```
> * +---+----+------+----+
> * | A | BC | DEFG | HI |
> * +---+----+------+----+
> --->
>  * +-----+-----+-----+
>  * | ABC | DEF | GHI |
>  * +-----+-----+-----+
> ```

```java
public class FixedLengthFrameDecoder extends ByteToMessageDecoder {
	//固定帧长度
    private final int frameLength;
    
     @Override
    protected final void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        //解析对象不为null，则放入out中
        Object decoded = decode(ctx, in);
        if (decoded != null) {
            out.add(decoded);
        }
    }

    protected Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
        //可读字节<固定帧长度，则返回null
        if (in.readableBytes() < frameLength) {
            return null;
        } else {
            //直接从bytebuf中截取固定长度，并返回一个新的bytebuf
            //ps:in的readindex会后移framelength
            return in.readRetainedSlice(frameLength);
        }
    }
    
}

```

# 二. `LineBasedFrameDecoder`

行解码器，将多个字节数组，分解成以`\r`或`\r\n`为间隔的数组，

可以推测至少有以下两个属性

1. 一个最大长度的属性，不然就会无限制读取，造成内存泄漏。
2. 还应该有属性表明是否需要包含行分隔符本身

```java
//当超过最大长度时是否丢弃已读取字节
private boolean discarding;
//是否剥离\r或\r\n分隔符
private final boolean stripDelimiter;
//累计丢弃字节数
private int discardedBytes;
protected Object decode(ChannelHandlerContext ctx, ByteBuf buffer) throws Exception {
    final int eol = findEndOfLine(buffer);
    if (!discarding) {
        //非丢弃模式
        if (eol >= 0) {
            final ByteBuf frame;
            //length：可读取的长度
            final int length = eol - buffer.readerIndex();
            //delimLength:界定符的长度
            final int delimLength = buffer.getByte(eol) == '\r'? 2 : 1;

            if (length > maxLength) {
                //超出最大长度，则丢弃之前读到的数据：将readInex置为\r\n之后
                buffer.readerIndex(eol + delimLength);
                fail(ctx, length);
                return null;
            }

            if (stripDelimiter) {
                //frame是不包括分隔符的可读数组
                frame = buffer.readRetainedSlice(length);
                //越过分隔符
                buffer.skipBytes(delimLength);
            } else {
                //frame是包括分隔符的可读数组
                frame = buffer.readRetainedSlice(length + delimLength);
            }

            return frame;
        } else {
            //没有找到分隔符，
            final int length = buffer.readableBytes();
            if (length > maxLength) {
                //丢弃所有可读长度
                discardedBytes = length;
                //将readindex置为writeIndex
                buffer.readerIndex(buffer.writerIndex());
                discarding = true;
                offset = 0;
                if (failFast) {
                    fail(ctx, "over " + discardedBytes);
                }
            }
            return null;
        }
    } else {
        //丢弃模式
        if (eol >= 0) {
            //找到分隔符
            final int length = discardedBytes + eol - buffer.readerIndex();
            final int delimLength = buffer.getByte(eol) == '\r'? 2 : 1;
            //将buffer的readIndex置为分隔符之后
            buffer.readerIndex(eol + delimLength);
            discardedBytes = 0;
            discarding = false;
            if (!failFast) {
                fail(ctx, length);
            }
        } else {
            //累计丢弃了多少字节
            discardedBytes += buffer.readableBytes();
            buffer.readerIndex(buffer.writerIndex());
            // We skip everything in the buffer, we need to set the offset to 0 again.
            offset = 0;
        }
        return null;
    }
}

//偏移量：buffer中最后一次读取的位置
private int offset;

private int findEndOfLine(final ByteBuf buffer) {
    int totalLength = buffer.readableBytes();
    //根据偏移量找到\n的位置
    int i = buffer.forEachByte(buffer.readerIndex() + offset, totalLength - offset, ByteProcessor.FIND_LF);
    if (i >= 0) {
        //偏移量置为0
        offset = 0;
        //如果前一个字符为\r,则将序号定位的\r\n中\r的位置
        if (i > 0 && buffer.getByte(i - 1) == '\r') {
            i--;
        }
    } else {
        //没有找到，则偏移量=最终长度
        offset = totalLength;
    }
    return i;
}
```

# 三. `DelimiterBasedFrameDecoder`

指定分隔符解码器，`LineBasedFrameDecoder`应该是该分隔符的一个子类，行分隔只是众多分隔符中的一个特例。

`DelimiterBasedFrameDecoder`和行分隔符类似，

```java
//maxFrameLength:允许读取的最大长度
//stripDelimiter:解析数据中是否包含间隔符
//delimiter：指定间隔符，多个
public DelimiterBasedFrameDecoder(
        int maxFrameLength, boolean stripDelimiter, ByteBuf delimiter) {
    this(maxFrameLength, stripDelimiter, true, delimiter);
}
//行解码器
private final LineBasedFrameDecoder lineBasedDecoder;
public DelimiterBasedFrameDecoder(
    int maxFrameLength, boolean stripDelimiter, boolean failFast, ByteBuf... delimiters) {
    validateMaxFrameLength(maxFrameLength);
    ObjectUtil.checkNonEmpty(delimiters, "delimiters");
	//当分隔符是\r\n时，则采用内部的行分隔符解码器解析
    if (isLineBased(delimiters) && !isSubclass()) {
        lineBasedDecoder = new LineBasedFrameDecoder(maxFrameLength, stripDelimiter, failFast);
        this.delimiters = null;
    } else {
        this.delimiters = new ByteBuf[delimiters.length];
        for (int i = 0; i < delimiters.length; i ++) {
            ByteBuf d = delimiters[i];
            validateDelimiter(d);
            this.delimiters[i] = d.slice(d.readerIndex(), d.readableBytes());
        }
        lineBasedDecoder = null;
    }
    this.maxFrameLength = maxFrameLength;
    this.stripDelimiter = stripDelimiter;
    this.failFast = failFast;
}
```

下面看具体的解码：

```java
protected Object decode(ChannelHandlerContext ctx, ByteBuf buffer) throws Exception {
    //采用行解码器
    if (lineBasedDecoder != null) {
        return lineBasedDecoder.decode(ctx, buffer);
    }
    int minFrameLength = Integer.MAX_VALUE;
    ByteBuf minDelim = null;
    //尝试所有的分隔符，找到分隔数据最短的分隔符。
    //如分隔符为&和），则AAA)BB&,则选择）分隔符
    for (ByteBuf delim: delimiters) {
        int frameLength = indexOf(buffer, delim);
        if (frameLength >= 0 && frameLength < minFrameLength) {
            minFrameLength = frameLength;
            minDelim = delim;
        }
    }

    if (minDelim != null) {
        int minDelimLength = minDelim.capacity();
        ByteBuf frame;

        //是否需要丢弃过长的数据
        if (discardingTooLongFrame) {
            discardingTooLongFrame = false;
            //丢弃
            buffer.skipBytes(minFrameLength + minDelimLength);

            int tooLongFrameLength = this.tooLongFrameLength;
            this.tooLongFrameLength = 0;
            if (!failFast) {
                fail(tooLongFrameLength);
            }
            return null;
        }

        if (minFrameLength > maxFrameLength) {
            //数据包大于最大长度，则丢弃数据包+分隔符长度
            buffer.skipBytes(minFrameLength + minDelimLength);
            fail(minFrameLength);
            return null;
        }
		//是否丢弃分隔符
        if (stripDelimiter) {
            frame = buffer.readRetainedSlice(minFrameLength);
            buffer.skipBytes(minDelimLength);
        } else {
            frame = buffer.readRetainedSlice(minFrameLength + minDelimLength);
        }

        return frame;
    } else {
        //没有找到分隔符
        if (!discardingTooLongFrame) {
            if (buffer.readableBytes() > maxFrameLength) {
                // Discard the content of the buffer until a delimiter is found.
                tooLongFrameLength = buffer.readableBytes();
                buffer.skipBytes(buffer.readableBytes());
                discardingTooLongFrame = true;
                if (failFast) {
                    fail(tooLongFrameLength);
                }
            }
        } else {
            // Still discarding the buffer since a delimiter is not found.
            tooLongFrameLength += buffer.readableBytes();
            buffer.skipBytes(buffer.readableBytes());
        }
        return null;
    }
}

```

