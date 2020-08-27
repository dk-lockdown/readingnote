#### Informer

Informer 高级错误处理行为：

+ 当长时间运行的监听连接断开时，会尝试另一个监听请求进行恢复，包装时间流不丢失任何事件。

+ 如果连接断开时间很长，并且 APIServer 丢失了时间，因为 ETCD 在新的监听请求成功之前已从数据库清除了事件，通知程序将重新列出所有对象。

  在重新列出所有对象之后，有一个可配置的重新同步时间段，用于协调内存中的缓存和业务逻辑：每次超过配置时间段时，将为所有对象调用已注册的时间处理器（常用单位分钟）。

每个 GroupVersionResource 中，⼀个⼆进制⽂件应该仅实例化⼀个通知程序。 为了使通知者易于共享，我们可以使⽤ SharedInformerFactory 实例化通知者。SharedInformerFactory 允许在应用程序中相同的资源共享一个 informer。

```informerFactory := informers.NewSharedInformerFactory(clientset,time.Second*30)```

重新同步机制将一组完整的事件发送到已注册的 UpdateFunc，以便控制器逻辑能够将其状态与 APIServer 的状态进行同步。通过比较 ObjectMeta.ResourceVersion 字段，可以将真正的更新与重新同步区分开。

更改对象之前思考：

+ informer 和列表器持有他们所返回的对象。因此，消费者必须在改变对象前进行深拷贝。
+ 客户端返回调用方拥有的新对象。
+ 转换返回共享对象。如果调用者确实拥有输入对象，则它不拥有输出对象。

```NewSharedInformerFactory``` 将资源的所有对象缓存在存储中的所有命名空间中。可使用灵活性更大的构造函数：

```
func NewFilteredSharedInformerFactory(
 client versioned.Interface, defaultResync time.Duration,
 namespace string,
 tweakListOptions internalinterfaces.TweakListOptionsFunc
) SharedInformerFactor
```



#### Reflector

Reflector 用于监控制定资源的 Kubernetes 资源，当监控资源发生变化时，触发相应的变更事件，例如 Added 事件、Updated 事件、Deleted 事件，并将其资源对象存放到本地缓存 DeltaFIFO 中。

Watch 操作通过 HTTP 协议与 Kubernetes API Server 建立长连接，接收 Kubernetes API Server 发来的资源变更事件。

k8s.io/client-go/1.4/tools/cache/reflector.go:

```go
// watchHandler watches w and keeps *resourceVersion up to date.
func (r *Reflector) watchHandler(w watch.Interface, resourceVersion *string, errc chan error, stopCh <-chan struct{}) error {
	start := time.Now()
	eventCount := 0

	// Stopping the watcher should be idempotent and if we return from this function there's no way
	// we're coming back in with the same watch interface.
	defer w.Stop()

loop:
	for {
		select {
		case <-stopCh:
			return errorStopRequested
		case err := <-errc:
			return err
		case event, ok := <-w.ResultChan():
			if !ok {
				break loop
			}
			if event.Type == watch.Error {
				return apierrs.FromObject(event.Object)
			}
			if e, a := r.expectedType, reflect.TypeOf(event.Object); e != nil && e != a {
				utilruntime.HandleError(fmt.Errorf("%s: expected type %v, but watch event object had type %v", r.name, e, a))
				continue
			}
			meta, err := meta.Accessor(event.Object)
			if err != nil {
				utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", r.name, event))
				continue
			}
			newResourceVersion := meta.GetResourceVersion()
			switch event.Type {
			case watch.Added:
				r.store.Add(event.Object)
			case watch.Modified:
				r.store.Update(event.Object)
			case watch.Deleted:
				// TODO: Will any consumers need access to the "last known
				// state", which is passed in event.Object? If so, may need
				// to change this.
				r.store.Delete(event.Object)
			default:
				utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", r.name, event))
			}
			*resourceVersion = newResourceVersion
			r.setLastSyncResourceVersion(newResourceVersion)
			eventCount++
		}
	}

	watchDuration := time.Now().Sub(start)
	if watchDuration < 1*time.Second && eventCount == 0 {
		glog.V(4).Infof("%s: Unexpected watch close - watch lasted less than a second and no items received", r.name)
		return errors.New("very short watch")
	}
	glog.V(4).Infof("%s: Watch close - %v total %v items received", r.name, r.expectedType, eventCount)
	return nil
}
```



#### DeltaFIFO

DeltaFIFO 可以分开理解，FIFO 是一个先进先出的队列，它拥有队列操作的基本方法，例如 Add、Update、Delete、List、Pop、Close 等，而 Delta 是一个资源对象存储，它可以保存资源对象的操作类型，例如 Added 操作类型、Updated 操作类型、Deleted 操作类型、Sync 操作类型等。

DeltaFIFO 队列中的资源对象在 Added 事件、Updated 事件、Deleted 事件中都调用了 queueActionLocked 函数，它是 DeltaFIFO 实现的关键。

k8s.io/client-go/1.4/tools/cache/delta_fifo.go:

```go
// queueActionLocked appends to the delta list for the object, calling
// f.deltaCompressor if needed. Caller must lock first.
func (f *DeltaFIFO) queueActionLocked(actionType DeltaType, obj interface{}) error {
	id, err := f.KeyOf(obj)
	if err != nil {
		return KeyError{obj, err}
	}

	// If object is supposed to be deleted (last event is Deleted),
	// then we should ignore Sync events, because it would result in
	// recreation of this object.
	if actionType == Sync && f.willObjectBeDeletedLocked(id) {
		return nil
	}

	newDeltas := append(f.items[id], Delta{actionType, obj})
	newDeltas = dedupDeltas(newDeltas)
	if f.deltaCompressor != nil {
		newDeltas = f.deltaCompressor.Compress(newDeltas)
	}

	_, exists := f.items[id]
	if len(newDeltas) > 0 {
		if !exists {
			f.queue = append(f.queue, id)
		}
		f.items[id] = newDeltas
		f.cond.Broadcast()
	} else if exists {
		// The compression step removed all deltas, so
		// we need to remove this from our map (extra items
		// in the queue are ignored if they are not in the
		// map).
		delete(f.items, id)
	}
	return nil
}
```

如果操作类型为 Sync，则标识该数据来源于 Indexer (本地存储)。

Pop 方法作为消费者方法使用。当队列中没有数据时，通过 f.cond.wait 阻塞等待数据，只有收到 cond.Broadcast 时才说明有数据被添加，解除当前阻塞状态。如果队列中不为空，取出 f.queue 的头部数据，将该对象传入 process 回调函数，由上层消费者进行处理。如果 process 回调函数处理出错，则将该对象重新存入队列。

```go
// Pop blocks until an item is added to the queue, and then returns it.  If
// multiple items are ready, they are returned in the order in which they were
// added/updated. The item is removed from the queue (and the store) before it
// is returned, so if you don't successfully process it, you need to add it back
// with AddIfNotPresent().
// process function is called under lock, so it is safe update data structures
// in it that need to be in sync with the queue (e.g. knownKeys). The PopProcessFunc
// may return an instance of ErrRequeue with a nested error to indicate the current
// item should be requeued (equivalent to calling AddIfNotPresent under the lock).
//
// Pop returns a 'Deltas', which has a complete list of all the things
// that happened to the object (deltas) while it was sitting in the queue.
func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error) {
	f.lock.Lock()
	defer f.lock.Unlock()
	for {
		for len(f.queue) == 0 {
			f.cond.Wait()
		}
		id := f.queue[0]
		f.queue = f.queue[1:]
		item, ok := f.items[id]
		if f.initialPopulationCount > 0 {
			f.initialPopulationCount--
		}
		if !ok {
			// Item may have been deleted subsequently.
			continue
		}
		delete(f.items, id)
		err := process(item)
		if e, ok := err.(ErrRequeue); ok {
			f.addIfNotPresent(id, item)
			err = e.Err
		}
		// Don't need to copyDeltas here, because we're transferring
		// ownership to the caller.
		return item, err
	}
}
```



Resync 机制会将 Indexer 本地存储中的资源对象同步到 DeltaFIFO 中，并将这些资源对象设置为 Sync 的操作类型。

```go
func (f *DeltaFIFO) syncKey(key string) error {
	f.lock.Lock()
	defer f.lock.Unlock()
	obj, exists, err := f.knownObjects.GetByKey(key)
	if err != nil {
		glog.Errorf("Unexpected error %v during lookup of key %v, unable to queue object for sync", err, key)
		return nil
	} else if !exists {
		glog.Infof("Key %v does not exist in known objects store, unable to queue object for sync", key)
		return nil
	}

	// If we are doing Resync() and there is already an event queued for that object,
	// we ignore the Resync for it. This is to avoid the race, in which the resync
	// comes with the previous value of object (since queueing an event for the object
	// doesn't trigger changing the underlying store <knownObjects>.
	id, err := f.KeyOf(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	if len(f.items[id]) > 0 {
		return nil
	}

	if err := f.queueActionLocked(Sync, obj); err != nil {
		return fmt.Errorf("couldn't queue object: %v", err)
	}
	return nil
}
```



#### Indexer

Indexer 是 client-go 用来存储资源对象并自带索引功能的本地存储，Reflector 从 DeltaFIFO 中将消费出来的资源对象存储至 Indexer。

Indexer 有 4 个非常重要的数据结构，分别是 Indices、Index、Indexers 及 IndexFunc。

```go
// IndexFunc knows how to provide an indexed value for an object.
type IndexFunc func(obj interface{}) ([]string, error)

// Index maps the indexed value to a set of keys in the store that match on that value
type Index map[string]sets.String

// Indexers maps a name to a IndexFunc
type Indexers map[string]IndexFunc

// Indices maps a name to an Index
type Indices map[string]Index
```

