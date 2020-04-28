今天面试的时候，被问到了serialize和Externalizable的区别

这两种方式都是java序列化的手段。
# 1. Serializable序列化

首先我们采用如下方式序列化
```java
  public static void main(String[] args){  
        Box myBox = new Box();  
        myBox.setWidth(50);  
        myBox.setHeight(30);  
  
        try{  
            FileOutputStream fs = new FileOutputStream("foo.ser");  
            ObjectOutputStream os =  new ObjectOutputStream(fs);  
            os.writeObject(myBox);  
            os.close();  
        }catch(Exception ex){  
            ex.printStackTrace();  
        }  
    }  
```

# 2. 源码跟踪
我们跟踪下`ObjectOutpuStream`的`writeObject`方法
```java
   if (obj instanceof String) {
        writeString((String) obj, unshared);
    } else if (cl.isArray()) {
        writeArray(obj, desc, unshared);
    } else if (obj instanceof Enum) {
        writeEnum((Enum<?>) obj, desc, unshared);
    } else if (obj instanceof Serializable) {
        writeOrdinaryObject(obj, desc, unshared);
    } else {
        if (extendedDebugInfo) {
            throw new NotSerializableException(
                cl.getName() + "\n" + debugInfoStack.toString());
        } else {
            throw new NotSerializableException(cl.getName());
        }
    }
```

可以看到内部有判断是否实现 `Serializable` 接口，并进入到 `writeOrdinaryObject` 方法中
```java
private void writeOrdinaryObject(Object obj,
                                    ObjectStreamClass desc,
                                    boolean unshared)throws IOException
{
    if (extendedDebugInfo) {
        debugInfoStack.push(
            (depth == 1 ? "root " : "") + "object (class \"" +
            obj.getClass().getName() + "\", " + obj.toString() + ")");
    }
    try {
        desc.checkSerialize();

        bout.writeByte(TC_OBJECT);
        writeClassDesc(desc, false);
        handles.assign(unshared ? null : obj);
        //是否继承Externalizable接口，由此可以推断Externalizable继承Serializable
        if (desc.isExternalizable() && !desc.isProxy()) {
            writeExternalData((Externalizable) obj);
        } else {
            writeSerialData(obj, desc);
        }
    } finally {
        if (extendedDebugInfo) {
            debugInfoStack.pop();
        }
    }
}

public interface Externalizable extends java.io.Serializable {

    void writeExternal(ObjectOutput out) throws IOException;

    void readExternal(ObjectInput in) throws IOException,ClassNotFoundException;
}


// 继承Serializable接口的处理方法
   private void writeSerialData(Object obj, ObjectStreamClass desc)
        throws IOException
    {
        ObjectStreamClass.ClassDataSlot[] slots = desc.getClassDataLayout();
        for (int i = 0; i < slots.length; i++) {
            ObjectStreamClass slotDesc = slots[i].desc;
            //1.判断是否有writeObject方法，有则调用writeObject
            if (slotDesc.hasWriteObjectMethod()) {
                PutFieldImpl oldPut = curPut;
                curPut = null;
                SerialCallbackContext oldContext = curContext;

                if (extendedDebugInfo) {
                    debugInfoStack.push(
                        "custom writeObject data (class \"" +
                        slotDesc.getName() + "\")");
                }
                try {
                    curContext = new SerialCallbackContext(obj, slotDesc);
                    bout.setBlockDataMode(true);
                    slotDesc.invokeWriteObject(obj, this);
                    bout.setBlockDataMode(false);
                    bout.writeByte(TC_ENDBLOCKDATA);
                } finally {
                    curContext.setUsed();
                    curContext = oldContext;
                    if (extendedDebugInfo) {
                        debugInfoStack.pop();
                    }
                }

                curPut = oldPut;
            } else {
                //2. 没有writeObject方法则采用默认保存所有字段
                defaultWriteFields(obj, slotDesc);
            }
        }
    }
```



1) 只针对对象的属性保存，方法不保存
2) 父类实现了序列化，子类自耦等实现序列化
3) transient：当某个字段被声明为transient后，默认序列化机制就会忽略该字段
4) writeObject/readObject:从上述源码可以看到，如果序列化对象中包含`writeObject/readObject`方法，则可以在该方法中实现自定义的序列化逻辑
```java

public class Person implements Serializable {
    ...
    transient private Integer age = null;
    ...
 
    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject();
        out.writeInt(age);
    }
 
    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        in.defaultReadObject();
        age = in.readInt();
    }

```

5. Exteralizable 的序列化逻辑在上述源码`writeExternalData`中，

## 总结

 Serializable 和 Exteralizable 最大的不同在于：

 1. 实现了Serializable的对象，默认不需要实现任务方法就可以序列化对象中的所有字段。
 2. 实现了Exteralizable的对象，则需要手动实现`writeExternal`和`readExternal`，否则不会序列化任何字段
 3.  实现了Serializable的对象，也可以通过添加`writeObject`和`readObject`手动控制序列化逻辑