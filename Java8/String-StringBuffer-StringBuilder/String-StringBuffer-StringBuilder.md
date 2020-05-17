# String-StringBuffer-StringBuild相互比较
## String
### 主要参数
```
    //实际存储结构为char数组
    private final char value[];
    private int hash; // Default to 0
```
### 构造方法

```
    //String的默认值为""
    public String() {
        this.value = "".value;
    }
    public String(String original) {
        this.value = original.value;
        this.hash = original.hash;
    }
    public String(char value[]) {
        this.value = Arrays.copyOf(value, value.length);
    }
```
### 拼接方法concat

```
    //实际底层还是对char数组进行操作，且返回的是一个新的String对象
    public String concat(String str) {
        int otherLen = str.length();
        if (otherLen == 0) {
            return this;
        }
        int len = value.length;
        char buf[] = Arrays.copyOf(value, len + otherLen);
        str.getChars(buf, len);
        return new String(buf, true);
    }
```

### equals
```
    //需要类型相同，char数组长度相同，且char数组内值相同
    public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        if (anObject instanceof String) {
            String anotherString = (String)anObject;
            int n = value.length;
            if (n == anotherString.value.length) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                        return false;
                    i++;
                }
                return true;
            }
        }
        return false;
    }
```
> ### 注意
```
substring,concat,replace,join,toLowerCase,toUpperCase,trim(substring)等方法，
返回值都是一个新的String对象。
所以String的长度不可变，发生变化都是创建了一个新的String对象。
```

## StringBuffer
### 主要参数

```
    //任何修改操作都会清空char[]为null
    //在toString时会把value的值通过Arrays.copyOfRange给到toStringCache，并创建一个String对象
    //但是在没有对StringBuffer修改时，可以一直用这个缓冲区中的char[]来实现toString
    private transient char[] toStringCache;
```
### 构造方法

```
    //默认长度为16，如果传值那就是值的长度+16
    public StringBuffer() {
        super(16);
    }
     public StringBuffer(int capacity) {
        super(capacity);
    }
    public StringBuffer(String str) {
        super(str.length() + 16);
        append(str);
    }
```
### 拼接方法append

```
    //注意，此处有同步锁且返回的是同一个对象
    public synchronized StringBuffer append(String str) {
        toStringCache = null;
        super.append(str);
        return this;
    }
```
> ### 注意

```
由于大部分方法都带有synchronized，故线程安全。
且对StringBuffer操作后不会创建新的对象（性能好于String），则没有像String一样重写equals方法
```

## StringBuilder
### 主要参数

### 构造方法

```
    //跟StringBuffer一样，默认也是16
    public StringBuilder() {
        super(16);
    }
    public StringBuilder(int capacity) {
        super(capacity);
    }
    public StringBuilder(String str) {
        super(str.length() + 16);
        append(str);
    }
```
### 拼接方法append

```
    //返回的也是同一个对象，但是却没有了同步锁
    public StringBuilder append(String str) {
        super.append(str);
        return this;
    }
```
> ### 注意

```
由于没有加synchronized，所以线程不安全。其他基本等同于StringBuffer。
```

## 总结
线程安全：
String 不安全
StringBuffer 安全
StringBuilder 不安全

性能：
通常情况下，StringBuffer>String, StringBuilder>String.

使用情况：
String：一般常量，不需要经常修改。
StringBuffer：多线程下需要经常修改。
StringBuilder：单线程下需要经常修改。

