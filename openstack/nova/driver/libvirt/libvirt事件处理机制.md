# 事件处理机制

## 背景

Openstack Nova计算模块在启动的时注册Libvirt模块提供的虚拟机生命周期事件, 然后调用event loop来将libvirt产生的事件发送到nova，nova根据接收到的事件来同步虚拟机的状态。

## 事件注册时序图

~~~ mermaid
sequenceDiagram
    participant  M as manager
    participant LD as libvirt_driver
    participant LH as libvirt_host
    participant L as libvirt
    M->>LD: Loading libvirt Driver
    LD->>LH: Loading libvrit host
    LH-->>LD: Loading libvirt host finished
    LD-->>M: loading libivrt drvier finished
    M->>LD: init_host()
    LD->>+LH: initialize()
    LH->>LH: 注册事件实现virEventRegisterDefaultImpl
    Note right of LH: 在建立连接之前注册
    LH->>LH: 初始化Event Pipe
    Note right of LH: 非阻塞GreenPipe
    loop 启动本地线程处理事件
    	LH->>LH: virEventRunDefaultImpl
    end
        loop 启动绿色线程分发事件
    	LH->>LH: _dispatch_events
    end
    LH-->>-LD: initialize finished
    LD->>+LH: get_capabilities
    LH->>+L: 与libvirt建立链接
    L-->>-LH: connect finished
    LH->>+L: 注册生命周期事件和事件回调函数
    L-->>-LH: register finished
    LH->>+L: 注册链接断开回调函数
    L-->>LH: register finished
    LH-->>LD: init host finished
    LD-->>M: init host finished
    
    M->>+LD: 注册事件监听函数
    LD->>LD: _compute_event_callback=handle_events
    LD-->>-M: 注册完成
    
    
    
~~~

## 事件初始化

注册流程分成两个阶段，阶段一是在与libvirt建立链接之前的注册，阶段二是与libvirt建立链接之后的注册。

### 建立链接前

#### 注册事件实现机制

服务启动后，加载完成libvirt driver对象后，manager通过`init_host()`函数来进行一些初始化的配置。此时**libvirt host**模块会调用***libvirt***模块的两个函数默认的事件机制实现，同时注册了一个libvirt异常处理函数。这样后面就可以调用 [virEventRunDefaultImpl](https://libvirt.org/html/libvirt-libvirt-event.html#virEventRunDefaultImpl)()函数来遍历处理libvirt发送的事件。

> 这个必须在libvirt链接建立之前注册

```Python
# NOTE(dkliban): Error handler needs to be registered before libvirt
#                connection is used for the first time.  Otherwise, the
#                handler does not get registered.
libvirt.registerErrorHandler(self._libvirt_error_handler, None)
libvirt.virEventRegisterDefaultImpl()
```



#### 创建GreenPipe

查看libvirt的接口文档，文档建议使用的是通过监控文件描述符来进行事件的处理。因为如果没有注册事件的时候，调用`virEventRunDefaultImpl`是一直阻塞的。所以采用通过一个pip-to-self[^1]的方式来实现。

> Furthermore, it is wise to set up a pipe-to-self handler (via [virEventAddHandle](https://libvirt.org/html/libvirt-libvirt-event.html#virEventAddHandle)()) or a timeout (via [virEventAddTimeout](https://libvirt.org/html/libvirt-libvirt-event.html#virEventAddTimeout)()) before calling this function, as it will block forever if there are no registered events. 

由于文件描述符实现的Pipe方式是阻塞性的。所以对于并发性性的事件处理具有阻塞性。因此为了无阻塞的处理libvirt事件，对处理流程进行了优化，使用无阻塞的GreenPipe来处理并发性的事件。

GreenPipe实际上是对文件描述符实现Pipe的一个优化，最重要的一个特性是增加了非阻塞性的功能。

创建GreenPipe的流程如下。

```Python
import os
from eventlet import greenio
......

rpipe, wpipe = os.pipe()
self._event_notify_send = greenio.GreenPipe(wpipe, 'wb', 0)
self._event_notify_recv = greenio.GreenPipe(rpipe, 'rb', 0)
```

创建了GreenPipe后，就可以通过写文件描述符写数据，然后通过读文件描述符读出写入的数据，来完成事件通知功能。

#### 遍历处理事件

注册完成事件处理机制后，需要启动一个线程来循环遍历来处理事件。Libvirt官方建议如下。

> Run one iteration of the event loop. Applications will generally want to have a thread which invokes this method in an infinite loop.

nova启动了一个本地线程，来无限循环来处理事件。因为线程已经被绿化，因此需要通过eventlet来获取本地线程。loop函数如下。调用libvirt接口`virEventRunDefaultImpl`。

```python
def _native_thread(self):
"""Receives async events coming in from libvirtd.

This is a native thread which runs the default
libvirt event loop implementation. This processes
any incoming async events from libvirtd and queues
them for later dispatch. This thread is only
permitted to use libvirt python APIs, and the
driver.queue_event method. In particular any use
of logging is forbidden, since it will confuse
eventlet's greenthread integration
"""

    while True:
    libvirt.virEventRunDefaultImpl()
    
```

这个函数要放在一个本地线程中执行，避免阻塞绿色线程。

```Python
from eventlet import patcher

native_threading = patcher.original("threading")

LOG.debug("Starting native event thread")
self._event_thread = native_threading.Thread(
target=self._native_thread)
self._event_thread.setDaemon(True)
self._event_thread.start()
```



#### 事件分发处理

启动一个绿色线程来进行事件的分发处理。该绿色线程从GreenPipe中读取数据，如果读到指定的字符串，则认为已经接收到了libvirt发送的事件，目前只处理`LifecycleEvent`类型的事件。事件是存放在本地队列`_event_queue`中，从`_event_queue`获取到事件后，将事件发送到**Manager**进行状态同步处理。

值得注意的地方是，这里有个特殊的延迟分发事件的处理。在事件类型为`EVENT_LIFECYCLE_STOPPED`时，会等待 *15*s的时间再发送该事件，因为收到`EVENT_LIFECYCLE_STOPPED`后，很可能会马上收到一个`EVENT_LIFECYCLE_STARTED`的事件。这种场景一般是在重启的场景下发生。这样在后一个事件接受到时将取消`EVENT_LIFECYCLE_STOPPED`事件，**Manager**就只会收到一个条`EVENT_LIFECYCLE_STARTED`的最终事件，不处理中间暂时的`EVENT_LIFECYCLE_STOPPED`事件。但是这样也会导致如果只是在关闭虚拟机操作的情况下，`EVENT_LIFECYCLE_STOPPED`事件延迟*15*s发送给**Manager**处理。

### 建立链接后

以上 的初始化操作都是在nova与libvirt建立链接之前完成的。只是完成了准备接受事件的一些准备。但是产生事件还需要建立libvirt链接后注册这些需要处理的生命周期事件。建立链接并注册生命周期事件和事件处理回调函数。

```python
        wrapped_conn = self._connect(self._uri, self._read_only)

        try:
            LOG.debug("Registering for lifecycle events %s", self)
            wrapped_conn.domainEventRegisterAny(
                None,
                libvirt.VIR_DOMAIN_EVENT_ID_LIFECYCLE,
                self._event_lifecycle_callback,
                self)
        except Exception as e:
            LOG.warning("URI %(uri)s does not support events: %(error)s",
                        {'uri': self._uri, 'error': e})

```

通过`domainEventRegisterAny`注册了`VIR_DOMAIN_EVENT_ID_LIFECYCLE`事件，就能接收到该类型的虚拟机事件。收到该类型事件后，会触发`_event_lifecycle_callback`回调函数的执行。该回调函数主要是将事件加入到`_event_queue`队列中，然后写入GreenPipe特定的字符串通知[事件分发处理](#事件分发处理)。代码实现如下。

* 收到事件后，进行转换后，添加到队列中。

```python
    def _event_lifecycle_callback(conn, dom, event, detail, opaque):
        """Receives lifecycle events from libvirt.

        NB: this method is executing in a native thread, not
        an eventlet coroutine. It can only invoke other libvirt
        APIs, or use self._queue_event(). Any use of logging APIs
        in particular is forbidden.
        """

        self = opaque

        uuid = dom.UUIDString()
        transition = None
        if event == libvirt.VIR_DOMAIN_EVENT_STOPPED:
            transition = virtevent.EVENT_LIFECYCLE_STOPPED
        elif event == libvirt.VIR_DOMAIN_EVENT_STARTED:
            transition = virtevent.EVENT_LIFECYCLE_STARTED
        elif event == libvirt.VIR_DOMAIN_EVENT_SUSPENDED:
            transition = virtevent.EVENT_LIFECYCLE_PAUSED
        elif event == libvirt.VIR_DOMAIN_EVENT_RESUMED:
            transition = virtevent.EVENT_LIFECYCLE_RESUMED

        if transition is not None:
            self._queue_event(virtevent.LifecycleEvent(uuid, transition))
```

* 将事件加入队列后，向GreenPipe中写入一个空格字符，告知已经接收到事件，进行处理。

```python
def _queue_event(self, event):
        """Puts an event on the queue for dispatch.

        This method is called by the native event thread to
        put events on the queue for later dispatch by the
        green thread. Any use of logging APIs is forbidden.
        """

        if self._event_queue is None:
            return

        # Queue the event...
        self._event_queue.put(event)

        # ...then wakeup the green thread to dispatch it
        c = ' '.encode()
        self._event_notify_send.write(c)
        self._event_notify_send.flush()
```



## 事件处理

**Libvirt Driver**接收到事件后，最终会发送到**Manager**处理。**Manager**主要是根据接收到的事件对虚拟机的状态进行同步。**Manager**中的`handle_events()`方法根据接收到的消息最终调用`_sync_instance_power_state()`方法进行状态同步。

```python
    def handle_events(self, event):
        if isinstance(event, virtevent.LifecycleEvent):
            try:
                self.handle_lifecycle_event(event)
            except exception.InstanceNotFound:
                LOG.debug("Event %s arrived for non-existent instance. The "
                          "instance was probably deleted.", event)
        else:
            LOG.debug("Ignoring event %s", event)P
```

### 时间处理时序图

~~~ mermaid
sequenceDiagram
    participant L as libvirt
    participant LH as libvirt_host
    participant LD as libvirt_driver
    participant  M as manager
    L->>LH: Trigger an Event
    loop 遍历事件产生
    	LH->>+LH: virEventRunDefaultImpl
    end
    LH->>LH: 将Event加入到_event_queue
    LH->>-LH: 写GreenPipe
    
    loop 获取并分发事件
    	alt GreenPipe读取到数据
    		LH->>+LH: 读取_event_queue中Event
    		LH->>LD: _event_emit
    	else GreenPipe中未读取到数据
    		LH->>-LH: loop
    	end	
    end
 
    LD->>+M: _compute_event_callback
    alt power_state == vm_power_state
    	M->>M: _sync_instance_power_state
    else power_state != vm_power_state
    	M->>-M: do nothing
    end
    
    	
~~~



[^1]: https://docs.python.org/2/library/os.html#os.pipe
[^2]: 

