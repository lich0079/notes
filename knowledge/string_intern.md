
# String.intern


* 会带来系统耗时的增加， 因为intern()的过程是去StringPool查找是否有该string对象存在，存在则返回引用，不存在则copy堆上对象储存一份在StringPool， 这里的查找是有额外耗时的

* 如果之前的string对象不在堆上，那么就不能节省内存的。如果是new String()的对象   new String().intern() 能节省内存

* 那么什么时候的string 对象是存在堆上呢， 比如new String出来的， 反序列化出来的， 或者从网络读到的

* new String的时候每次都是有新对象产生的， 1万次 String.valueOf(1) 是会有1万个对象

* 当保存string 的时候要小心它的来源，即使他们的值一样，但可能来自不能的对象，这时可以用intern优化