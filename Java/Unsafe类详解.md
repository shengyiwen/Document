### Java的Unsafe类

Unsafe类属于sun.misc包下，不属于Java标准，但是很多Java的基础类库，包括一些被广泛使用到的高性能
开发库(netty、hadoop、kafka)等都基于Unsafe类开发的。Unsafe类在提升Java运行效率，增强Java语言底层操作
能力方面起到了很大的作用。

Unsafe类可以让Java拥有了像C一样操作内存空间的能力，但是同时带来了指针的问题。过度使用Unsafe类会大大
增加出错的可能性，而且Java屏蔽了指针，所以官方也不建议使用Unsafe类。我们业务代码中最好也不要使用Unsafe类，
除非你对它很了解才行。

Unsafe类是单例模式，通过静态方法getUnsafe()获取，但是Unsafe做了限制，如果普通调用的话，会出现SecurityException。
因此如果我们想使用，就需要使用反射的方法获取

    
    Field f = Unsafe.class.getDeclaredField("theUnsafe");
    f.setAccessible(true);
    Unsafe unsafe = (Unsafe)f.get(null)

### 主要功能

- 内存管理

    - allocateMemory 
    - reallocateMemory 
    - copyMemory 
    - freeMemory 
    - getAddress 
    - addressSize
    - pageSize 
    - getInt
    - getIntVolatile
    - putInt
    - putIntVolatile
    - putOrderedInt

- 非常规的对象实例化

    - allocateInstance

- 操作类、对象、变量

    - staticFieldOffset
    - defineClass
    - defineAnonymousClass
    - ensureClassInitialized
    - objectFieldOffset

- 数组操作

    - arrayBaseOffset
    - arrayIndexScale

- 多线程同步，包括锁机制以及Cas操作等
  
    - monitorEnter
    - tryMonitorEnter
    - monitorExit
    - compareAndSwapInt
    - compareAndSwap
      
- 挂起和恢复

    - park
    - unpark

- 内存屏障，避免指令重排问题

    - loadFence
    - storeFence
    - fullFence


### 源码解读

    public final class Unsafe {

        // 获取单例对象Unsafe类
        public static Unsafe getUnsafe()

        // 获取对象在offset偏移地址对应int类型字段的属性值
        public native int getInt(Object var1, long var2);

        // 设置对象在offset偏移地址对应int类型字段的属性值
        public native void putInt(Object var1, long var2, int var4);

        // 获取对象在offset偏移地址对应object类型字段的属性值
        public native Object getObject(Object var1, long var2);

        // 设置对象在offset偏移地址对应object类型字段的属性值
        public native void putObject(Object var1, long var2, Object var4);
        
        // 获取对象在offset偏移地址对应boolean类型字段的属性值
        public native boolean getBoolean(Object var1, long var2);

        // 设置对象在offset偏移地址对应boolean类型字段的属性值
        public native void putBoolean(Object var1, long var2, boolean var4);

        // 获取对象在offset偏移地址对应byte类型字段的属性值
        public native byte getByte(Object var1, long var2);
        
        // 设置对象在offset偏移地址对应byte类型字段的属性值
        public native void putByte(Object var1, long var2, byte var4);

        // 获取对象在offset偏移地址对应short类型字段的属性值
        public native short getShort(Object var1, long var2);

        // 设置对象在offset偏移地址对应short类型字段的属性值
        public native void putShort(Object var1, long var2, short var4);
        
        // 获取对象在offset偏移地址对应char类型字段的属性值
        public native char getChar(Object var1, long var2);
        
        // 设置对象在offset偏移地址对应char类型字段的属性值
        public native void putChar(Object var1, long var2, char var4);

        // 获取对象在offset偏移地址对应long类型字段的属性值
        public native long getLong(Object var1, long var2);

        // 设置对象在offset偏移地址对应long类型字段的属性值
        public native void putLong(Object var1, long var2, long var4);

        // 获取对象在offset偏移地址对应float类型字段的属性值
        public native float getFloat(Object var1, long var2);

        // 设置对象在offset偏移地址对应float类型字段的属性值
        public native void putFloat(Object var1, long var2, float var4);

        // 获取对象在offset偏移地址对应float类型字段的属性值
        public native double getDouble(Object var1, long var2);

        // 设置对象在offset偏移地址对应float类型字段的属性值
        public native void putDouble(Object var1, long var2, double var4);

        // 获取在offset偏移地址对应byte类型字段的属性值
        public native byte getByte(long var1);

        // 设置在offset偏移地址对应float类型字段的属性值
        public native void putByte(long var1, byte var3);

        // 获取在offset偏移地址对应int类型字段的属性值
        public native int getInt(long var1);

        // 设置在offset偏移地址对应int类型字段的属性值
        public native void putInt(long var1, int var3);

        // 设置在offset偏移地址对应long类型字段的属性值
        public native long getLong(long var1);

        // 设置在offset偏移地址对应long类型字段的属性值
        public native void putLong(long var1, long var3);

        // 从一个给定的内存地址获取本地指针，如果不是allocateMemory方法，结果将不确定
        public native long getAddress(long var1);

        // 设置一个本地指针到一个给定的内存地址，如果不是allocateMemory方法结果将不确定
        public native void putAddress(long var1, long var3);

        // 分配内存，size是大小
        public native long allocateMemory(long size);
        
        // 重新分配内存，在offset的位置，重新分配内存
        public native long reallocateMemory(long var1, long var3);

        // 在给定内存块中设置值
        public native void setMemory(Object var1, long var2, long var4, byte var6);

        // 在给定内存块中设置值
        public void setMemory(long var1, long var3, byte var5) {
            this.setMemory((Object)null, var1, var3, var5);
        }
        
        // 从一块内存复制到另一块内存
        public native void copyMemory(Object var1, long var2, Object var4, long var5, long var7);

        // 从一块内存复制到另一块内存
        public void copyMemory(long var1, long var3, long var5) {
            this.copyMemory((Object)null, var1, (Object)null, var3, var5);
        }

        // 释放内存
        public native void freeMemory(long var1);

        // 获取静态字段在内存地址的偏移量，对于特定的Field的值是唯一的
        public native long staticFieldOffset(Field var1);

        // 获取对象字段在内存地址的偏移量，对于特定的Field的值是唯一的
        public native long objectFieldOffset(Field var1);

        // 获取一个给定字段的位置
        public native Object staticFieldBase(Field var1);

        public native boolean shouldBeInitialized(Class<?> var1);

        // 确保给定的Class是否被初始化
        public native void ensureClassInitialized(Class<?> var1);

        // 返回给定数组类的第一个地址的偏移量，为了获取数组类的元素，这和偏移量地址会和arrayIndexScale的非0值返回一起使用
        public native int arrayBaseOffset(Class<?> var1);

        // 返回给定素组类的寻址的换算因子，一个正确的换算因子不能返回类型的时候会返回0，这个返回值可以和arrayBaseOffset一起使用获取元素的值
        public native int arrayIndexScale(Class<?> var1);

        public native int addressSize();

        // 获取本机内存的页数，这个值永远都是2的幂次方
        public native int pageSize();

        // 告诉虚拟机定义了一个没有安全检查的类
        public native Class<?> defineClass(String var1, byte[] var2, int var3, int var4, ClassLoader var5, ProtectionDomain var6);
         
        // 定义一个类，但是不让它知道类加载器和系统字典
        public native Class<?> defineAnonymousClass(Class<?> var1, byte[] var2, Object[] var3);

        // 创建一个类的实例，不需要调用它的构造函数、初使化代码、各种JVM安全检查以及其它的一些底层的东西。即使构造函数是私有，我们也可以通过这个方法创建它的实例,对于单例模式
        public native Object allocateInstance(Class<?> var1) throws InstantiationException;  

        @Deprecated
        // 锁定对象，必须是没有被锁的
        public native void monitorEnter(Object var1);

        /** @deprecated */
        // 解锁对象
        @Deprecated
        public native void monitorExit(Object var1);
    
        /** @deprecated */
        // 试图锁定对象，返回true或false是否锁定成功，如果锁定，必须用monitorExit解锁  
        @Deprecated
        public native boolean tryMonitorEnter(Object var1);
        
        // 引发异常，没有通知
        public native void throwException(Throwable var1);

        // 在对象offset的位置比较object类型字段和期望的值，如果相同就更新
        // 这个方法的操作是原子的
        public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);
    
        // 在对象offset的位置比较integer类型字段和期望的值，如果相同就更新
        // 这个方法的操作是原子的
        public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
    
        // 在对象offset的位置比较long类型字段和期望的值，如果相同就更新
        // 这个方法的操作是原子的
        public final native boolean compareAndSwapLong(Object var1, long var2, long var4, long var6);
    
        // 获取对象在offset偏移地址地址对应的object类型字段值，支持volatile load语义
        public native Object getObjectVolatile(Object var1, long var2);
    
         // 设置对象在offset的偏移地址对应的object类型字段属性值，支持volatile store语义
        public native void putObjectVolatile(Object var1, long var2, Object var4);
    
        // 获取对象在offset偏移地址地址对应的int类型字段值，支持volatile load语义
        public native int getIntVolatile(Object var1, long var2);
    
        // 设置对象在offset的偏移地址对应的int类型字段属性值，支持volatile store语义
        public native void putIntVolatile(Object var1, long var2, int var4);
    
        // 获取对象在offset偏移地址地址对应的long类型字段值，支持volatile load语义
        public native long getLongVolatile(Object var1, long var2);
    
        // 设置对象在offset的偏移地址对应的long类型字段属性值，支持volatile store语义
        public native void putLongVolatile(Object var1, long var2, long var4);
    
        // 设置对象在offset地址对应object类型字段的值，这是一个有序的或者有延迟的方法，
        // 并不能保证值在发生修改后能被其他线程看到，只有字段被volatile修饰并且期望被意外修改的时候使用才有用
        public native void putOrderedObject(Object var1, long var2, Object var4);
    
        // 设置对象在offset地址对应int类型字段的值，这是一个有序的或者有延迟的方法，
        // 并不能保证值在发生修改后能被其他线程看到，只有字段被volatile修饰并且期望被意外修改的时候使用才有用
        public native void putOrderedInt(Object var1, long var2, int var4);
    
        // 设置对象在offset地址对应long类型字段的值，这是一个有序的或者有延迟的方法，
        // 并不能保证值在发生修改后能被其他线程看到，只有字段被volatile修饰并且期望被意外修改的时候使用才有用
        public native void putOrderedLong(Object var1, long var2, long var4);
    
        // 释放被阻塞的线程，当unpark的时候permit置为0，这个时候线程会解除阻塞
        public native void unpark(Object var1);
    
        // 阻塞一个线程知道unpark出现、线程中断、或者timeout时间到期。park的时候permit会被置为1
        public native void park(boolean var1, long var2);
    
        // 获取系统在不同时间系统的负载情况  
        public native int getLoadAverage(double[] var1, int var2);
    
        // 内存屏障，禁止load操作重排序。屏障前的load操作不能被重排序到屏障后，屏障后的load操作不能被重排序到屏障前
        public native void loadFence();

        // 内存屏障，禁止store操作重排序。屏障前的store操作不能被重排序到屏障后，屏障后的store操作不能被重排序到屏障前
        public native void storeFence();

        // 内存屏障，进制load、store操作重排序
        public native void fullFence();
    }

### 测试代码

    public class UnsafeTest {
    
        static class Test {
            private int i = 1;
            private Long l;
            private String s;
    
            private Test() {
            }
    
            public Test(int i, long l, String s) {
                this.i = i;
                this.l = l;
                this.s = s;
            }
    
            public int getI() {
                return i;
            }
    
            public Long getL() {
                return l;
            }
    
            public String getS() {
                return s;
            }
        }
    
        private static Unsafe unsafe = null;
    
        public static void main(String[] args) throws Exception {
    
            // =============== 操作Object类 ===============
    
            // 直接通过class对象实例化对象，就算不适用任何构造器都能创建
            Test test = (Test) getUnsafe().allocateInstance(Test.class);
            // 输出0，即使test.i = 1，但是仍然输出的是0！！！因为类没有经过显示的初始化
            System.out.println("test.i value: " + test.getI());
            // 获取字段i的偏移量地址
            long iOffset = getUnsafe().objectFieldOffset(Test.class.getDeclaredField("i"));
            unsafe.getAndSetInt(test, iOffset, 100);
            // 输出100
            System.out.println("test.i value：" + test.getI());
    
            // ============= 操作数组 ==============
            int[] data = new int[10];
            // [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
            System.out.println("array value:" + Arrays.toString(data));
    
            // 获取int.class的offset
            int arrayBaseOffset = getUnsafe().arrayBaseOffset(int[].class);
            System.out.println("array base offset:" + arrayBaseOffset);
    
            // 将第一个值设置为1
            unsafe.putInt(data, (long)arrayBaseOffset, 1);
            // 将最后一个值设置为11
            unsafe.putInt(data, (long)arrayBaseOffset + 4 * 9, 1);
            // array value:[1, 0, 0, 0, 0, 0, 0, 0, 1, 0]
            System.out.println("array value:" + Arrays.toString(data));
    
            // ============= 操作CAS ==============
            boolean success = unsafe.compareAndSwapInt(test, iOffset, 100, 2);
            if (success) {
                System.out.println("交换成功");
            }
            System.out.println(test.getI());
            
        }
    
        private static Unsafe getUnsafe() {
            if (unsafe != null) {
                return unsafe;
            }
            try {
                Field field = Unsafe.class.getDeclaredField("theUnsafe");
                field.setAccessible(true);
                unsafe = (Unsafe) field.get(null);
                return unsafe;
            } catch (Exception e) {
                throw new RuntimeException();
            }
        }
    }
