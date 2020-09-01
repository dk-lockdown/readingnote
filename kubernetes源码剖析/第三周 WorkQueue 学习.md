WorkQueue 支持如下特性：

+ 有序

  Interface 按照添加顺序处理元素。但 DelayingInterface、RateLimitingInterface 会先按照 delay 时间排序等待。

+ 去重

+ 并发性：多生产者和多消费者。

+ 标记机制：标记一个元素是否被处理。

+ 通知机制：ShutDown 方法通过信号量通知队列不再接收新的元素。

+ 延迟

+ 限速

+ Metrics

WorkQueue 支持三种队列：

+ Interface：FIFO 队列
+ DelayingInterface：延迟队列
+ RateLimitingInterface：限速队列

源码位置：kubernetes/pkg/util/workqueue

DelayingInterface：

```go
func (q *delayingType) waitingLoop() {
	defer utilruntime.HandleCrash()

	// Make a placeholder channel to use when there are no items in our list
	never := make(<-chan time.Time)

	for {
		if q.Interface.ShuttingDown() {
			// discard waiting entries
			q.waitingForAdd = nil
			q.waitingTimeByEntry = nil
			return
		}

		now := q.clock.Now()

		// Add ready entries
		readyEntries := 0
		for _, entry := range q.waitingForAdd {
			if entry.readyAt.After(now) {
				break
			}
			q.Add(entry.data)
			delete(q.waitingTimeByEntry, entry.data)
			readyEntries++
		}
		q.waitingForAdd = q.waitingForAdd[readyEntries:]

		// Set up a wait for the first item's readyAt (if one exists)
		nextReadyAt := never
		if len(q.waitingForAdd) > 0 {
			nextReadyAt = q.clock.After(q.waitingForAdd[0].readyAt.Sub(now))
		}

		select {
		case <-q.stopCh:
			return

		case <-q.heartbeat:
			// continue the loop, which will add ready items

		case <-nextReadyAt:
			// continue the loop, which will add ready items

		case waitEntry := <-q.waitingForAddCh:
      // 尚未到延迟时间，继续延迟
			if waitEntry.readyAt.After(q.clock.Now()) {
				q.waitingForAdd = insert(q.waitingForAdd, q.waitingTimeByEntry, waitEntry)
			} else {
				q.Add(waitEntry.data)
			}

			drained := false
			for !drained {
				select {
				case waitEntry := <-q.waitingForAddCh:
					if waitEntry.readyAt.After(q.clock.Now()) {
            // 会按照延迟时间排序
						q.waitingForAdd = insert(q.waitingForAdd, q.waitingTimeByEntry, waitEntry)
					} else {
						q.Add(waitEntry.data)
					}
				default:
					drained = true
				}
			}
		}
	}
}
```

