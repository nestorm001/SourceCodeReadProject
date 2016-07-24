### HashMap源码阅读

#### 前言
HashMap作为一种常用的集合类型，其中的设计必然有许多值得学习的地方，阅读其源码不仅可以学习设计思路，更可以帮助我们更好的使用HashMap  
由于HashMap中涉及的点较多，一次性看完写完也不太现实，坑就慢慢填吧...

#### 源码阅读
##### 从HashMap的存储table开始说起
```java
    /**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     */
    transient Node<K,V>[] table;
```
对于HashMap和HashSet(内部由HashMap实现)，由于通过数组来进行Key的散列存储，每次需要扩容时，需要进行一次内存分配和数组的拷贝(每次翻倍，大小为2的幂次方)，因此在提前知道大小的情况下，直接指定大小可以提高不少的效率。

Node数组table为HashMap存储的实现

下面来看看内部类Node的实现
```java
	static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```

从上面的实现可以看到，Node类内部存储了hash值(作为table中的index使用)，key、value为entry本身的值   
由Node next可以看出，Node类本身还可以构成一个链表，当多个entry的hash值相等时，使用链表进行维护   
以上便是HashMap内部存储只要实现的原理，剩下的坑慢慢填，慢慢填...   

