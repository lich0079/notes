# java 对象内存占用情况 

## 工具


```
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.14</version>
    <scope>provided</scope>
</dependency>


```
## code
```
import org.openjdk.jol.info.ClassLayout;

public class Jol {
    public static void main(String[] args) {
        Person o = new Person();
        System.out.println(ClassLayout.parseInstance(o).toPrintable());
    }
}

class Person {
    int age = 256;
    String name = "ssss";
}

```
## Output
```
Person object internals:
 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
      0     4                    (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                    (object header)                           43 c0 00 f8 (01000011 11000000 00000000 11111000) (-134168509)
     12     4                int Person.age                                256
     16     4   java.lang.String Person.name                               (object)
     20     4                    (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

### 为什么Java HotSpot(TM) 64-Bit Server  跑出来 指针是32位?

因为存在ptr压缩，求证：  
```
-XX:InitialHeapSize=268435456 -XX:MaxHeapSize=4294967296 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC

-XX:InitialHeapSize=268435456 -XX:MaxHeapSize=4294967296 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC 
java version "1.8.0_241"
```    

### 为什么64位OS指针只用48位？

当前版本的AMD64架构就规定了只用48位地址；一个表示虚拟内存地址的64位指针只有低48位有效并带符号扩展到64位——换句话说，其高16位必须是全1或全0，而且必须与低48位的最高位（第47位）一致，否则通过该地址访问内存会产生#GP异常（general-protection exception）。只用48位的原因很简单：因为现在还用不到完整的64位寻址空间，所以硬件也没必要支持那么多位的地址。

