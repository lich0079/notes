# Java 内部类 存在的理由/为什么要设计Java内部类

本文目的在于思考Java 内部类存在的意义，即哪些技术场景是必须用内部类实现，其他手段替代不了。static inner class 不在本文的讨论范畴中。

## 定义

 * JAVA中，内部类可以访问到外围类的变量、方法或者其它内部类等所有成员，即使它被定义成private了，但是外部类不能访问内部类中的变量。这样通过内部类就可以提供一种代码隐藏和代码组织的机制，并且这些被组织的代码段还可以自由地访 问到包含该内部类的外围上下文环境。


 * 实现举例


```
public class ArrayList<E> {

    //something else...
    
    public Iterator<E> iterator() {
        return new Itr();
    }
    
    private class Itr implements Iterator<E> {
        int cursor = 0;

        int lastRet = -1;

        int expectedModCount = modCount; // modCount是外部类私有变量

        public boolean hasNext() {//...}
        
        public E next() {//...}
        
        public void remove() {//...}    
    }
}

```


## 内部类的潜在替换方案

 * 接口
    * 比如 ArrayList.Itr 这个内部类貌似也可以通过在 ArrayList 上去实现接口 Iterator 来完成遍历的需要， 就是外部类直接暴露出 Iterator 需要的方法。
    ```
        public class ArrayList<E> implements Iterator<E> {

            public boolean hasNext() {//...}
                
            public E next() {//...}
                
            public void remove() {//...} 

        }

    ```
 * 内部类外置化
    * 内部类的一大特性是可以访问外部的私有变量， 但当我把内部类外置化后， 也可以通过在构造时传参的方式把要用到的私有变量传进去，来达到近似的效果。


### 接口为什么不能替代内部类

 * 对于接口能不能替代，网上的观点说用接口替换内部类在需要实现的接口很多的情况下，类会很乱，不知道哪个接口属于哪个方法，这个说法有道理，但不够充分。

* 有一个理由在我看来是内部类对接口绝对胜出的情形，就是内部类可以维护自己的内部变量，你是我的，我是我的; 接口的话则数据全在一起。更关键的是一个外部类可以对应多个内部类，就是1对N的关系, 每个N有自己的内部变量，互不干扰。比如内部类可以很轻松实现并发 Iterator，试想下如果是接口的方式去实现并发 Iterator， 那个变量维护...

* 内部类是懒加载，只有在用到的时候才会去开闭空间，如果是实现接口则没法省掉。

### 内部类外置化为什么不能替代内部类

 * 内部类可以让代码结构更紧凑，不会出现一大堆类。 内部类没必要让外部知道。