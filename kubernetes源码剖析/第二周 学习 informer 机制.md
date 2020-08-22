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





#### DeltaFIFO



#### Indexer

