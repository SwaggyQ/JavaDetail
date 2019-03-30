# java的自动装箱和自动拆箱
## 正文
#### 今天看到一个很有意思的事情，也就是关于在java中的基础类型的自动装箱和自动拆箱的事情。之前可能就是一直在用，但是也不知道为什么可以这样。例如，下面一个最简单的例子
```
	public static void main(String[] args) {
        Integer a = 10;
        int b = a;
        System.out.println(a);
        System.out.println(b);
    }
```
#### 平时可能非常理所当然的就这么用了。但是如果问你，这个为什么可以这样子呢？或者给你出这么一道题目
```
	public static void main(String[] args) {
        Integer x1 = 10;
        Integer x2 = 10;
        Integer x3 = 200;
        Integer x4 = 200;
        System.out.println(x1 == x2); 
        System.out.println(x3 == x4);

    }
```
#### 这两个的返回值按照常理来说都是一样的情况，应该要么都是false或者都是true。但是不是的，返回值第一个是true，第二个是false。

