---
layout: post
title: "张江游记"
date: 2014-01-14 23:05:25 +0800
comments: true
categories: 
---

今天和James在一个关于<big>LinkedHashMap</big>的问题上产生了一些分歧，我查了一下JDK1.7的文档，首先这个LinkedHashMap肯定是存在的，不过我当时的理解也有一些偏差。所以在这里自己再次梳理一下。

LinkedHashMap是一个HashMap实现的加强版本，用来维持插入的key的顺序。先来看看LinkedHashMap是如何来保证插入顺序的。

LinkedHashMap中使用的是双端链表来实现顺序，LinkedHashMap的Entry是它改写了HashMap的Entry的实现，增加了Header,Footer用来指向头和尾

``` java
 private static class Entry<K,V> extends HashMap.Entry<K,V> {
        // These fields comprise the doubly linked list used for iteration.
        Entry<K,V> before, after;

        Entry(int hash, K key, V value, HashMap.Entry<K,V> next) {
            super(hash, key, value, next);
        }

        /**
         * Removes this entry from the linked list.
         */
        private void remove() {
            before.after = after;
            after.before = before;
        }

        /**
         * Inserts this entry before the specified existing entry in the list.
         */
        private void addBefore(Entry<K,V> existingEntry) {
            after  = existingEntry;
            before = existingEntry.before;
            before.after = this;
            after.before = this;
        }

        /**
         * This method is invoked by the superclass whenever the value
         * of a pre-existing entry is read by Map.get or modified by Map.set.
         * If the enclosing Map is access-ordered, it moves the entry
         * to the end of the list; otherwise, it does nothing.
         */
        void recordAccess(HashMap<K,V> m) {
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
            if (lm.accessOrder) {
                lm.modCount++;
                remove();
                addBefore(lm.header);
            }
        }

        void recordRemoval(HashMap<K,V> m) {
            remove();
        }
    }
```
当我们通过entrySet()的方法获取整个Map的迭代器的时候，他的返回是有序的。

不过LinkedHashMap平时毕竟用的不太多，这里就用一个例子来演示它的作用。假设有一个字符串“aabbccdeeg”,要求获取第一个不重复的字符,我们就可以用LinkedHashMap来实现

``` java

Map<Char,Integer> map = new LinkedHashMap<Char,Integer>();
for (int i =0;i<s.length;i++){
    char c = s.charAt(i);
    if map.containsKey(c) {
        int value = map.get(c);
        map.put(c,++value);
    } else {
        map.put(c,1);
    }
}

for (Entry<Char,Integer> entry: map.entrySet()) {
    if (entry.getValue() == 1) {
        System.out.println(entry.getKey());
    }
}

```
