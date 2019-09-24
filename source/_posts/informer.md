---
title: k8s Informer机制源码揭秘
date: 2019-06-26 09:26:46
tags: [kube-controller-manager,Informer]
categories: kubernetes
---
Etcd 存储集群的数据信息，apiserver 作为统一入口，任何对数据的操作都必须经过 apiserver。客户端(kubelet/scheduler/ontroller-manager)通过 list-watch 监听 apiserver 中资源(pod/rs/rc 等等)的 create, update 和 delete 事件，并针对事件类型调用相应的事件处理函数。
<!--more-->
那么 list-watch 具体是什么呢，顾名思义，list-watch 有两部分组成，分别是 list 和 watch。list 非常好理解，就是调用资源的 list API 罗列资源，基于HTTP短链接实现；watch 则是调用资源的 watch API 监听资源变更事件，基于HTTP长链接实现.

Informer 在初始化时，先调用 Kubernetes List API 获得某种 resource 的全部 Object，缓存在内存中; 然后，调用 Watch API 去 watch 这种 resource，去维护这份缓存; 最后，Informer 就不再调用 Kubernetes 的任何 API。

### list流程

```golang
// 版本号设置为0
options := metav1.ListOptions{ResourceVersion: "0"}
// list接口
list = Reflector.listerWatcher.List(options)

// 获取版本号
resourceVersion = listMetaInterface.GetResourceVersion()
// 将list的内容提取成对象列表
items, err := meta.ExtractList(list)
// 将list中对象列表的内容和版本号存储到本地的缓存store中，并全量替换已有的store的内容。
Reflector.syncWith(items, resourceVersion)   <==> Reflector.store.Replace(found range items, resourceVersion)

//最后设置最新的资源版本号。

Reflector.setLastSyncResourceVersion(resourceVersion)   <==> Reflector.lastSyncResourceVersion = resourceVersion
```


### watch流程
```golang
watch = Reflector.listerWatcher.Watch()     

event = watch.ResultChan()

switch event.type: {
    case Add: Reflector.store.Add(event.Object)
    case Modified: Reflector.store.Update(event.Object)
    case Deleted:  Reflector.store.Delete(event.Object)
}

```
watch把监听到的事件存到Reflector.store中
阅读源代码得知r.store由DeltaFIFO赋值：
```golang
Reflector.store = controller.config.Queue = DeltaFIFO
```
DeltaFIFO 再Pop事件到 Controller 中

Controller收到这个事件，会触发 Processor 的回调函数
```golang
func (c *controller) processLoop() {
	for {
		obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
    }
}
func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error) {
	for {
		id := f.queue[0]
		f.queue = f.queue[1:]
		item, ok := f.items[id]
		if !ok {
			continue
		}
		delete(f.items, id)
		err := process(item)   //##########
		return item, err
	}
}
```
### process

```golang

cfg := &Config{
        ...
		Process: s.HandleDeltas,
    }
```

```golang
func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
	for _, d := range obj.(Deltas) {
		switch d.Type {
		case Sync, Added, Updated:
			isSync := d.Type == Sync
			s.cacheMutationDetector.AddObject(d.Object)
			if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
				if err := s.indexer.Update(d.Object); err != nil {
					return err
				}
				s.processor.distribute(updateNotification{oldObj: old, newObj: d.Object}, isSync)
			} else {
				if err := s.indexer.Add(d.Object); err != nil {
					return err
				}
				s.processor.distribute(addNotification{newObj: d.Object}, isSync)
			}
		case Deleted:
			if err := s.indexer.Delete(d.Object); err != nil {
				return err
			}
			s.processor.distribute(deleteNotification{oldObj: d.Object}, false)
		}
	}
	return nil
}
```

```golang
func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()

	if sync {
		for _, listener := range p.syncingListeners {
			listener.add(obj)
		}
	} else {
		for _, listener := range p.listeners {
			listener.add(obj)
		}
	}
}
```

```golang
func (p *processorListener) add(notification interface{}) {
	p.addCh <- notification
}
```

### Processor

processor的主要功能就是记录了所有的回调函数实例(即 ResourceEventHandler 实例)，并负责触发这些函数。在sharedIndexInformer.Run部分会调用processor.run。

流程：

listenser的add函数负责将notify装进 pendingNotifications (参加上面一个代码块)
pop函数取出pendingNotifications的第一个nofify,输出到 nextCh channel。
run函数则负责取出notify，然后根据notify的类型(增加、删除、更新)触发相应的处理函数，这些函数是在不同的NewXxxcontroller实现中注册的。

```golang
wg.StartWithChannel(processorStopCh, s.processor.run)
```

```golang
func (p *sharedProcessor) run(stopCh <-chan struct{}) {
	func() {
		for _, listener := range p.listeners {
			p.wg.Start(listener.run)
			p.wg.Start(listener.pop)
		}
		p.listenersStarted = true
	}()
}
```

```golang
func (p *processorListener) pop() {
	var nextCh chan<- interface{}
	var notification interface{}
	for {
		select {
		case nextCh <- notification:
			var ok bool
			notification, ok = p.pendingNotifications.ReadOne()
			if !ok { // Nothing to pop
				nextCh = nil 
			}
		case notificationToAdd, ok := <-p.addCh:
			if !ok {
				return
			}
			if notification == nil { 
				notification = notificationToAdd
				nextCh = p.nextCh
			} else { 
				p.pendingNotifications.WriteOne(notificationToAdd)
			}
		}
	}
}

func (p *processorListener) run() {
    for next := range p.nextCh {
        switch notification := next.(type) {
            case updateNotification:
                p.handler.OnUpdate(notification.oldObj, notification.newObj)
            case addNotification:
                p.handler.OnAdd(notification.newObj)
            case deleteNotification:
                p.handler.OnDelete(notification.oldObj)  
        }   
    }	
}

```

NewXxxcontroller实现中注册函数OnUpdate、OnAdd、Ondelete等函数逻辑可以参见另外一篇博客：[kubernetes之Job技术内幕]（https://luohua13.github.io/2019/04/25/job/) 进行具体的分析。

### 二级缓存
二级缓存属于 Informer 的底层缓存机制，这两级缓存分别是 DeltaFIFO 和 LocalStore。

这两级缓存的用途各不相同。DeltaFIFO 用来存储 Watch API 返回的各种事件 ，LocalStore 只会被 Lister 的 List/Get 方法访问 。
虽然 Informer 和 Kubernetes 之间没有 resync 机制，但 Informer 内部的这两级缓存之间存在 resync 机制。

### 后记
关于二级缓存中的localStore代码部分还不是很明确，本人猜想LocalStore是经过sharedIndexInformer.Indexer处理的，具体可见HandleDeltas函数。各位大侠若有这方面的正确理解可在下方留言。

### 参考链接
- https://www.huweihuang.com/kubernetes-notes/code-analysis/kube-controller-manager/sharedIndexInformer.html
- http://wsfdl.com/kubernetes/2019/01/10/list_watch_in_k8s.html
- https://caicloud.io/blog/5a1b83a975d69a0681e19584
