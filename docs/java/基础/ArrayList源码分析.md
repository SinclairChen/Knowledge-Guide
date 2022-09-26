## 1. ArrayList的优缺点

ArrayList底层是基于数组来实现的，所以优缺点也是非常的明显

- 优点：
  - 可以通过数组下标，来实现随机的读取数组中的某个元素，这个性能是比较高的

- 缺点：
  - 数组是固定长度的，当不停的往ArrayList中插入数据，超出数组容量的时候，就需要进行扩容和拷贝，这个性能是非常差的
  - 在数组中间插入数据的时候，需要将这个新增元素后面的所有数据后移一位，这个性能也是比较差的。
  - 同理删除数组中间元素的的时候，也需要进行拷贝，需要将后面的元素往前移动一位



## 2. 核心方法源码分析

### 2.1 ArrayList() 构造方法

```java
//默认大小
private static final int DEFAULT_CAPACITY = 10;

//构建一个Object的空数组
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

//建议创建ArrayList的时候给定一个容量大小
//避免在使用的时候频繁的扩容和拷贝
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
```



### 2.2 add() 添加元素

```java
//直接添加一个元素
public boolean add(E e) {
    //判断是否需要进行扩容
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    
    //核心就是先在当前位置插入元素，然后将size进行++处理
    elementData[size++] = e;
    return true;
}

//通过数组下标添加元素
public void add(int index, E element) {
    //判断是否越界
    rangeCheckForAdd(index);

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    
    //进行数组拷贝
    //从这个数组的第几个元素开始，移动到这个数组的什么位置，移动多少个元素
    //例如在下标1这个位置添加一个元素“赵六”，会先进行数组拷贝
    //{“张三”，“李四”，“王五”} => {“张三”，“李四”，“李四”，“王五”}
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    
    //然后用新的元素替换当前这个位置的元素
    //{“张三”，“赵六”，“李四”，“王五”}
    elementData[index] = element;
    size++;
}
```



### 2.3 ensureCapacityInternal() 判断是否需要进行扩容

```java
//每次往ArrayList都会判断一下，当前数组中的元素是否需要进行扩容
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

//判断是否超出默认数组大小的容量，如果没有超出，返回默认容量大小10
//如果超出，直接返回这个值
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    //判断是否需要进行扩容
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

//具体进行扩容的方法
private void grow(int minCapacity) {
    // overflow-conscious code
    //获取原来数组中元素的个数
    int oldCapacity = elementData.length;
    
    //新的数组的大小，是原来数组大小加上原来数组大小的一半
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    
    // minCapacity is usually close to size, so this is a win:
    //然后把原来数组中的元素，拷贝到新的数组中去
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```



### 2.4 set() 插入元素

```java
public E set(int index, E element) {
    rangeCheck(index);
	
    //获取当前位置原来的元素，并返回
    E oldValue = elementData(index);
    
    //替换这个位置的元素
    elementData[index] = element;
    return oldValue;
}

//判断是否越界，如果是抛出异常
private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```



### 2.5 get() 获取元素

```java
public E get(int index) {
    rangeCheck(index);

    //直接通过下标返回当前数组中的元素
    return elementData(index);
}
```



### 2.6 remove() 删除元素

```java
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    
    //拿到当前位置的元素，返回
    E oldValue = elementData(index);

    //需要移动的元素的数量
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
    
    //最后，将最后一位设置为null，然后将size进行--操作
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

