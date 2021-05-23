### AQS中的tryAcquire方法

首先，我们的问题是AQS中的tryAcquire方法为什么不定义成抽象方法，而是定义成一个实现的方法，但并没有真正实现。

```java
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
```

### 思考过程

#### 思考一

如果定义成抽象方法，意味着什么？ 或者说抽象方法和非抽象方法的区别在哪里？

- 定义成抽象方法，意味着子类必须要实现。
- 定义成非抽象方法，子类可以重写，也可以不重写

#### 思考二

既然设计者定义成允许子类不重写的方法，说明意图是子类在继承AQS的时候，是允许tryAcquire不重写

那么，什么情况下不需要重写呢？

#### 思考三

回想到AQS 支持两种锁，

1. 排他锁
2. 共享锁

而tryAcquire是实现排他锁的一个方法，那另外一个方法是tryAcquireShared

是不是意味着共享锁是不需要实现tryAcquire方法的呢？ 基于这个猜想，我们来求证一下

然后思考，哪些类是实现了共享锁呢？于是想到了`CountDownLatch`

于是找到CountDownLatch中锁的一个实现，发现果然能够证明我们的猜想。这个类中并没有实现tryAcquire。

```java
private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
```

### 总结

技术这个领域，它是一种逻辑的关系的存在， 大家把自己的思维打开一些， 就是如果你是设计者，你为什么这么考虑？如果不这么考虑会存在什么问题。

当你开始这么思考的时候，你就发现技术这些东西，都是有严密的逻辑性。那么我们在学习的过程中，也就更加得心应手了。