# ArrayList-Note
ArrayList集合源码分析
# 集合ArrayList源码分析总结（基于JDK1.8）
## 一、总结
（1）ArrayList随机元素时间复杂度O(1)，插入删除操作需大量移动元素，效率较低

（2）为了节约内存，当新建容器为空时，会共享Object[] EMPTY_ELEMENTDATA = {}和 Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {}空数组

（3）容器底层采用数组存储，每次扩容为1.5倍

（4）ArrayList的实现中大量地调用了Arrays.copyof()和System.arraycopy()方法，其实Arrays.copyof()内部也是调用System.arraycopy()。System.arraycopy()为Native方法

（5）两个ToArray方法

Object[] toArray()方法。该方法有可能会抛出java.lang.ClassCastException异常
<T> T[] toArray(T[] a)方法。该方法可以直接将ArrayList转换得到的Array进行整体向下转型
（6）ArrayList可以存储null值

（7）ArrayList每次修改（增加、删除）容器时，都是修改自身的modCount；在生成迭代器时，迭代器会保存该modCount值，迭代器每次获取元素时，会比较自身的modCount与ArrayList的modCount是否相等，来判断容器是否已经被修改，如果被修改了则抛出异常（fast-fail机制）。


## 二、代码分析

### ArrayList底层是使用一个Object类型的数组来存放数据的，size变量代表List实际存放元素的数量

### add,remove,get,set,contains操作
### get和set方法，都是通过数组下标，直接操作数据的，时间复杂度为O(1)

### contains方法和indexOf方法
```
public boolean contains(Object o) {
    return indexOf(o) >= 0;
}

public int indexOf(Object o) {
    // 遍历所有元素找到相同的元素，返回元素的下标，
    // 如果是元素为null，则直接比较地址，否则使用equals的方法比较
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}

```
### add方法

```
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // 扩容检测
    elementData[size++] = e;  //新增元素添加到末尾
    return true;
}

public void add(int index, E element) {
    rangeCheckForAdd(index);

    ensureCapacityInternal(size + 1);  // 扩容检测
    // 使用System.arraycopy的方法，将index后面元素往后移动1位
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    elementData[index] = element;  // 存放元素到index位置
    size++;
}
```
### remove方法

```
public E remove(int index) {
    rangeCheck(index);  //越界检测

    modCount++;
    E oldValue = elementData(index);  //旧值

    int numMoved = size - index - 1;  // 需要移动元素的数量
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // 方便JVM进行GC操作，避免出现泄露

    return oldValue;
}

public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
    
    
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
    
    protected void removeRange(int fromIndex, int toIndex) {
        modCount++;
        int numMoved = size - toIndex;
        System.arraycopy(elementData, toIndex, elementData, fromIndex,
                         numMoved);

        // clear to let GC do its work
        int newSize = size - (toIndex-fromIndex);
        for (int i = newSize; i < size; i++) {
            elementData[i] = null;
        }
        size = newSize;
    }
    
    public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, false);
    }
    private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;
        boolean modified = false;
        try {
            for (; r < size; r++)
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            if (r != size) {
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                w += size - r;
            }
            if (w != size) {
                // clear to let GC do its work
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
        return modified;
    }
```

### retainAll方法

```
public boolean retainAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, true);
    }
```

### get方法

```
public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }
```
### set方法

```
public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
```
### trimToSize()方法
由于elementData的长度会被拓展，size标记的是其中包含的元素的个数。所以会出现size很小但elementData.length很大的情况，将出现空间的浪费。trimToSize将返回一个新的数组给elementData，元素内容保持不变，length很size相同，节省空间。

```
public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
    }
```
### clear方法

```
public void clear() {
        modCount++;

        // clear to let GC do its work
        for (int i = 0; i < size; i++)
            elementData[i] = null;

        size = 0;
    }
```
### 扩容策略
ArrayList底层是使用数组存储的，当数组大小不足存放新增元素的时候，才会发生扩容。

在add操作中，ArrayList首先会调用ensureCapacityInternal方法进行扩容检测的。

如果数组大小不足，则会自动扩容；如果扩容后的大小超出数组最大的大小，则会抛出异常。

ensureCapacityInternal(size + 1);
ArrayList扩容方案，主要有两个步骤：1.大小检测，2.扩容

#### 大小检测：

检测数组大小是否为0，如果是，则使用默认的扩容大小10

检测是否需要扩容，只有当数组最小需要容量大小大于当前数组大小时，才会进行扩容

#### 扩容：grow和hugeCapacity

进行数组越界判断

拷贝原始数据到新的数组中


```
private void ensureCapacityInternal(int minCapacity) {
    // 通过ArrayList<Integer> a = new ArrayList<Integer>()或者通过序列化读取，元素大小为0时，底层数组才会为null数组
    // 如果底层数组大小为0，则使用默认的容量大小10
    if (elementData == EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
    // 数据结构发生改变，和fail-fast机制有关，在使用迭代器过程中，只能通过迭代器的方法（比如迭代器中add，remove等），修改List的数据结构，
    // 如果使用List的方法（比如List中的add，remove等），修改List的数据结构，会抛出ConcurrentModificationException
    modCount++;  


    // 当前数组容量大小不足时，才会调用grow方法，自动扩容
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;


private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    // 新的容量大小 = 原容量大小的1.5倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0) //溢出判断，比如minCapacity = Integer.MAX_VALUE / 2, oldCapacity = minCapacity - 1
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```
### fail-fast机制的实现
fail-fast机制也叫作”快速失败”机制，是Java集合中的一种错误检测机制。

在对集合进行迭代过程中，除了迭代器可以对集合进行数据结构上进行修改，其他的对集合的数据结构进行修改，都会抛出ConcurrentModificationException错误。

这里，所谓的进行数据结构上进行修改,是指对存储的对象，进行add,set,remove操作，进而对数据发生改变。

ArrayList中，有个modCount的变量，每次进行add，set，remove等操作，都会执行modCount++。

在获取ArrayList的迭代器时，会将ArrayList中的modCount保存在迭代中，

每次执行add,set,remove等操作，都会执行一次检查，调用checkForComodification方法，对modCount进行比较。

如果迭代器中的modCount和List中的modCount不同，则抛出ConcurrentModificationException


```
final void checkForComodification() {
    if (expectedModCount != ArrayList.this.modCount)
        throw new ConcurrentModificationException();
}
```

```
     private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        public boolean hasNext() {
            return cursor != size;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        @Override
        @SuppressWarnings("unchecked")
        public void forEachRemaining(Consumer<? super E> consumer) {
            Objects.requireNonNull(consumer);
            final int size = ArrayList.this.size;
            int i = cursor;
            if (i >= size) {
                return;
            }
            final Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length) {
                throw new ConcurrentModificationException();
            }
            while (i != size && modCount == expectedModCount) {
                consumer.accept((E) elementData[i++]);
            }
            // update once at end of iteration to reduce heap write traffic
            cursor = i;
            lastRet = i - 1;
            checkForComodification();
        }

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```

```
private class ListItr extends Itr implements ListIterator<E> {
        ListItr(int index) {
            super();
            cursor = index;
        }

        public boolean hasPrevious() {
            return cursor != 0;
        }

        public int nextIndex() {
            return cursor;
        }

        public int previousIndex() {
            return cursor - 1;
        }

        @SuppressWarnings("unchecked")
        public E previous() {
            checkForComodification();
            int i = cursor - 1;
            if (i < 0)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i;
            return (E) elementData[lastRet = i];
        }

        public void set(E e) {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                ArrayList.this.set(lastRet, e);
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        public void add(E e) {
            checkForComodification();

            try {
                int i = cursor;
                ArrayList.this.add(i, e);
                cursor = i + 1;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
    }
```

### 序列化


```
transient Object[] elementData; 
// non-private to simplify nested class access
```
transient修饰符让elementData无法自动序列化，这样的原因是，数组内存储的的元素其实只是一个引用，单单序列化一个引用没有任何意义，反序列化后这些引用都无法在指向原来的对象。ArrayList使用writeObject()实现手工序列化数组内的元素。


### 使用场景

ArrayList的使用场景主要从其优缺点来考虑的：

#### 优点：

get，set，时间复杂度为O(1)

add（一般都是在末尾插入），时间复杂度为O(1)，最差情况下（往头部插入数据），时间复杂度O(n)

##### 数据存储是顺序的


#### 缺点：

remove，时间复杂度为O(n)，最优情况下（移除末尾元素），时间复杂度为O(1)

ArrayList底层使用数组存储数据，数组是不能自动扩容的，因此在发生扩容的情况下，需要移动大量的元素。

ArrayList大小很大的时候，会存在空间浪费（可以通过trimToSize方法，清除空闲空间）

数组大小是由限制的，受jvm和机器的影响，当扩容超出上限时，ArrayList会抛出异常

插入操作多，数据量不大，顺序存储时，可以考虑使用ArrayList

#### 多线程情况下：

ArrayList所有的操作，都不是同步的，因此ArrayList不是线程安全的。

如果考虑到线程安全的话，可以使用CopyOnWriteArrayList或者外部同步ArrayList（List list = Collections.synchronizedList(new ArrayList(…));）



### 思考
#### 1.remove方法中，为什么会将数组对应的元素置为null？

ArrayList内部使用数组实现一套管理对象的机制，remove操作中，已经将元素的数量-1了，ArrayList认为该对象已经被移除了，应该被jvm回收。

但是，对于jvm来说，该值仍然保存在数组中，ArrayList持有这个对象的引用，在jvm发生GC时，这个对象是不对被jvm回收，这样就会造成内存泄露了。

#### 2.查找元素的方法中（比如indexOf），为什么需要对元素进行null值判断？

判断对象是否相等，有两个方面，1.对象存储的地址；2.对象的内容。

==，是用来比较两个对象的地址是否相等，一般来说，两个对象的地址相同，那么这两个对象可以认为是相同的对象

equals方法，是用来比较对象内容的，当然，也可以重载该方法，直接比较对象地址；Object对象的equals方法，是比较地址的。

一般来说，重载equals方法的同时，也要重载hashCode方法的，重载hashCode方法，必须得遵守6个原则：

自反性：对于任何非null的引用值x，x.equals(x),必须返回true

传递性：对于任何非null的引用值x,y,z，如果x.equals(y) 为true，且y.equals(z)为true，那么x.equals(z)必须为true

对称性：对于任何非null的引用值x,y，如果x.equals(y)为true，那么y.equals(x)必须为true

非空性：对于任何非null的引用值x,x.equals(null)必须为false

一致性：对于任何非null的引用值x，y，如果多次调用equals方法，如果x和y比较的值没有改变，那么x.equals(y)就会一致性返回true或者false

为什么重载equals方法，一般要重载hashCode方法？ 
重载equals方法，可以不重载hashCode方法，但是一般情况，不建议这么做。 
hashCode方法，使用来求出对象的Hash值， 
重载hashCode方法主要是为了提高一些容器（比如HashMap，Hashtable）进行hash运算的效率，而且也可以避免出现一些错误（比如HashSet容器的操作）

对于元素进行null值判断，我认为主要是为了效率考虑，如果是null值的话，可以直接比较地址，而非空值，则需要通过equals方法来比较，由于ArrayList是泛型的，

所以其添加的元素，可能重载equals方法，自定义了判断的原则。

#### 3.grow方法中，对新容量大小进行判断，为什么会定义MAX_ARRAY_SIZE的？

ArrayList底层存储是使用数组来实现的，所以ArrayList存储文件的大小必定受数组大小的限制，所以在扩容中，可以看到ArrayList对新容量大小进行逻辑判断。

影响数组最大值：

理论上最大值为Integer.MAX_VALUE(2^32 - 1)

对象头限制，不同类型的元素，可创建数组的最大值是不同的，byte是1字节，int是4字节

比如jvm可用内存为1M，32位机器下，

　　
int[] bytes = new int[1024 * 1024 / 4];
byte[] bytes = new byte[1024 * 1024];
jvm可用内存大小限制

比如jvm可用内存为1M，32位机器下，

byte[] bytes = byte[1024 * 1024]


