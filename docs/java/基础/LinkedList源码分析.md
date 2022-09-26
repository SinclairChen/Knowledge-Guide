## 1. LinkedList的优缺点

LinkedList的底层是基于双向链表来实现的

- 优点：
  - 在LinkedList中增删元素的时候，和ArrayList不一样的是，不需要移动其他元素，只需要改变节点的指向即可
- 缺点：
  - 不适合随机获取某个位置的元素，因为每次获取元素都需要从链表的头开始遍历，直到找到这个元素



## 2. 核心方法源码分析

### 2.1 插入元素

```java
//添加一个元素，在队尾插入
public boolean add(E e) {
    linkLast(e);
    return true;
}

//在队尾插入一个元素的具体方法
void linkLast(E e) {
    //先获取last节点
    final Node<E> l = last;
    
    //封装一个新的node节点，这个节点的prev指向之前那个last节点
    final Node<E> newNode = new Node<>(l, e, null);
    
    //将last指针指向这个新的节点
    last = newNode;
    
    //如果之前的那个last节点没有node，就这个新的node节点指定为first节点
    //反之，如果last节点有node，将这个node的next指向新的node
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}

//在链表中间插入一个元素
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element);
    else
        //
        linkBefore(element, node(index));
}

//判断插入在这个链表的前半部分还是后半部分
Node<E> node(int index) {
    // assert isElementIndex(index);

    //如果是插在前半部分，从队头开始遍历找到那个节点
    //如果是插在后半部分，从队尾开始遍历找到那个节点
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}

//在链表中间插入元素的具体方法
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    //拿到当前位置这个节点的前一个节点的node
    final Node<E> pred = succ.prev;
    
    //封装一个新的node节点，并将这个节点的prev指向之前那个节点的前一个node，next指向为之前的那个节点
    final Node<E> newNode = new Node<>(pred, e, succ);
    
    //将之前那个node接地那的prev指向这个新封装的node
    succ.prev = newNode;
    
    //判断之前那个node节点是否有前一个node，如果没有将first指针指向新封装的node
    //如果有，就需要将前一个node的next指向新封装的那个node
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}

//在队头插入一个元素
public void addFirst(E e) {
    linkFirst(e);
}

//在队头插入的具体方法
private void linkFirst(E e) {
    //首先获取first节点的node元素
    final Node<E> f = first;
    
    //封装一个新的node，并将这个node节点的next指向之前first节点的那个node
    final Node<E> newNode = new Node<>(null, e, f);
    
    //将first指针指向新的node节点
    first = newNode;
    
    //判断之前first节点上有没有node
    //如果没有将last指针，指向这个新的node
    //反之如果有，需要将原来的那个node节点的prev指向新的node
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}

//在队尾插入一个元素，和add方法一样
public void addLast(E e) {
    linkLast(e);
}
```



### 2.2 获取元素

```java
//获取队头的元素
public E getFirst() {
    
    //直接返回first指针指向的node
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return f.item;
}

//等同于getFirst
//和getFirst的区别在于getFirst如果没有元素，会抛异常，而peek会直接返回null
public E peek() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}

//获取队尾的元素
public E getLast() {
    
    //直接返回last指针指向的node
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return l.item;
}


//通过下标获取元素
//会先判断一下，从链表的前半部分获取还是从链表的后半部分获取
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
```



### 2.3 删除元素

```java
//删除元素，调用的是removeFirst方法
public E remove() {
    return removeFirst();
}

//删除链表中的第一个元素
public E removeFirst() {
    
    //获取first指针指向的node元素，如果为null直接抛出异常
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}

//删除链表中第一个元素
private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    //获取这个node节点中的item和next的指向的参数
    //将这个node的item和next的参数设置为null
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    
    //将first指针指向之前拿到的next的node
    //如果next没有node，就需要将last设置为null
    //如果有node，就需要将这个node的prev改为null
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}

//删除链表中的最后一个元素
public E removeLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return unlinkLast(l);
}

//删除链表中最后一个元素的具体方法
private E unlinkLast(Node<E> l) {
    // assert l == last && l != null;
    
   	//拿到最后一个node节点的item和prev参数
    final E element = l.item;
    final Node<E> prev = l.prev;
    
    //将这个node的item和prev设置为nul
    l.item = null;
    l.prev = null; // help GC
    
    //将last指针指向刚刚拿到的prev的node
    last = prev;
    
    //如果前面没有node，就需要将first改为null
    //如果有就需要将拿到的node的next指向改为null
    if (prev == null)
        first = null;
    else
        prev.next = null;
    size--;
    modCount++;
    return element;
}

//等同于removeFirst
//区别在于poll如果没有获取到元素不会抛出异常，会直接返回null
public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}
```

