之前的几篇文章分析了通讯发起的链路流程.

后续分析下`netty`的编解码器是如何实现的，

所有的数据读取都会通过`ChannelRead`方法处理，编解码也不例外。



这里主要关注下`ByteToMessageDecoder`

```java

/**
上一次读取的数据
*/
ByteBuf cumulation;
 
/**
下面两个Cumulator是处理ByteBuf的不同策略
*/
public static final Cumulator MERGE_CUMULATOR;
public static final Cumulator COMPOSITE_CUMULATOR;
```



关注下`channelRead`方法

```java

public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        if (msg instanceof ByteBuf) {
            //生成一个对象容器，用于保存解析后的数据
            CodecOutputList out = CodecOutputList.newInstance();
            try {
                //cumulation保存了上次读取记录，为null表示第一次读取
                first = cumulation == null;
                //1.通过cumulator合并上次的数据cumulation和本次的数据msg
                cumulation = cumulator.cumulate(ctx.alloc(),
                        first ? Unpooled.EMPTY_BUFFER : cumulation, (ByteBuf) msg);
                //2.子类解析，cumulation是个累计数据，包含上N次的数据
                callDecode(ctx, cumulation, out);
            } catch (DecoderException e) {
                throw e;
            } catch (Exception e) {
                throw new DecoderException(e);
            } finally {
                try {
                    //如果上一次读取数据不为null并且没有可读取的字节，则释放cumulation
                    if (cumulation != null && !cumulation.isReadable()) {
                        numReads = 0;
                        cumulation.release();
                        cumulation = null;
                    } else if (++numReads >= discardAfterReads) {
                        //当调用channelread此处超过discardAfterReads(默认16)时丢弃已读数据
                        // We did enough reads already try to discard some bytes so we not risk to see a OOME.
                        // See https://github.com/netty/netty/issues/4275
                        numReads = 0;
                        discardSomeReadBytes();
                    }
					
                    //传递read事件
                    int size = out.size();
                    firedChannelRead |= out.insertSinceRecycled();
                    fireChannelRead(ctx, out, size);
                } finally {
                    out.recycle();
                }
            }
        } else {
            ctx.fireChannelRead(msg);
        }
    }
```

继续跟进`callDecode`

```java
protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    try {
        while (in.isReadable()) {
            //已解析成功的对象数量，首次为0
            int outSize = out.size();

            //如果有解析成功的数据，则立即通过channelread传递
            if (outSize > 0) {
                fireChannelRead(ctx, out, outSize);
                out.clear();

                // Check if this handler was removed before continuing with decoding.
                // If it was removed, it is not safe to continue to operate on the buffer.
                //
                // See:
                // - https://github.com/netty/netty/issues/4635
                if (ctx.isRemoved()) {
                    break;
                }
                //已传递对象，所以已解析对象数量清零
                outSize = 0;
            }
			//剩余可读取数据
            int oldInputLength = in.readableBytes();
            decodeRemovalReentryProtection(ctx, in, out);

            // Check if this handler was removed before continuing the loop.
            // If it was removed, it is not safe to continue to operate on the buffer.
            //
            // See https://github.com/netty/netty/issues/1664
            if (ctx.isRemoved()) {
                break;
            }
			//上述decodeRemovalReentryProtection可能会影响out容器内对象的数量
            //如果outSize=out.size，表明没有解析成功的对象
            if (outSize == out.size()) {
                //如果剩余可读取数据不变，表明没有读取到数据，跳出等待下次读取
                if (oldInputLength == in.readableBytes()) {
                    break;
                } else {
                    //否则表明有数据被读取了(没有解析到对象，但被读取处理了)，则继续读取数据
                    continue;
                }
            }

            //有解析成功的对象，但是没有从缓冲区读取？
            if (oldInputLength == in.readableBytes()) {
                throw new DecoderException(
                    StringUtil.simpleClassName(getClass()) +
                    ".decode() did not read anything but decoded a message.");
            }

            if (isSingleDecode()) {
                break;
            }
        }
    } catch (DecoderException e) {
        throw e;
    } catch (Exception cause) {
        throw new DecoderException(cause);
    }
}

//此处调用了抽象方法decode
final void decodeRemovalReentryProtection(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)
    throws Exception {
    decodeState = STATE_CALLING_CHILD_DECODE;
    try {
        decode(ctx, in, out);
    } finally {
        boolean removePending = decodeState == STATE_HANDLER_REMOVED_PENDING;
        decodeState = STATE_INIT;
        if (removePending) {
            fireChannelRead(ctx, out, out.size());
            out.clear();
            handlerRemoved(ctx);
        }
    }
}

```

