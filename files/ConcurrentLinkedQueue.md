## ConcurrentLinkedQueue
``` java
public E poll() {
    restartFromHead: for (;;) {
        for (Node<E> h = head, p = h, q;; p = q) {
            final E item;
						// p不是空的，那么将其拿下
            if ((item = p.item) != null && p.casItem(item, null)) {
                // Successful CAS is the linearization point
                // for item to be removed from this queue.
                // 第一次遍历不会走到这里
								if (p != h) // hop two nodes at a time
                  // 对应head节点为空，需要出队的情况 
									// 如果q不为空，就是当第一个元素出队后，后面还有元素，就使用q当head，这个head不为null,否则用p当head，这个head就是null的 
									updateHead(h, ((q = p.next) != null) ? q : p);
                // 直接返回的话，对应head节点不为空的情况
								return item;
            }
						// 特殊情况1：只有一个节点，而且为null节点，则更新h为p
						// 代表队列中没有元素
						// 队列中只有一个节点，延迟更新head
            else if ((q = p.next) == null) {
								// 什么情况会导致h!=p?
								// 第二次循环，一开始的head就是空的，现在p也是空了，那么
								// 说明其他线程把p出队了，但是head还指向p的prev，这里延
								// 迟将head设为p。
                updateHead(h, p);
                return null;
            }
						// p == p.next,说明头节点是空头，重新开始，使用新的head
            else if (p == q)
                continue restartFromHead;
						// p是空头，那么，p指向p=next，head被抛弃
						// else p = q;
        }
    }
}
```
![](files\pic\ConcurrentLinkedQueuePoll.png)
```java
public boolean offer(E e) {
    final Node<E> newNode = new Node<E>(Objects.requireNonNull(e));

    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {
            // p is last node
            if (NEXT.compareAndSet(p, null, newNode)) {
                // Successful CAS is the linearization point
                // for e to become an element of this queue,
                // and for newNode to become "live".
								// 如果一次就入队成功，那么延迟改变tail，减少cas操作
								// 多次才成功，更改tail为自身
                if (p != t) // hop two nodes at a time; failure is OK
										TAIL.weakCompareAndSet(this, t, newNode);
                return true;
            }
            // Lost CAS race to another thread; re-read next
        }
				// 由于poll产生的特殊情况
        else if (p == q)
            // We have fallen off list.  If tail is unchanged, it
            // will also be off-list, in which case we need to
            // jump to head, from which all live nodes are always
            // reachable.  Else the new tail is a better bet.
            p = (t != (t = tail)) ? t : head;
        // tail不是队尾
				else
            // Check for tail updates after two hops.
						// 否则对应单线程情况，向下遍历找到尾节点
						// 第一次遍历必然有 p == t
						// 第二次遍历 p != t 并且 t != (t = tail)
						// means if t != tail then t = tail
						// 如果t不等于tail，说明tail已经被改变，那么直接指向最新的tail并返回
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```
![](files\pic\ConcurrentLinkedQueueOffer.png)
