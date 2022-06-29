title:: 深入理解 golang 的 sync.Pool

- tags:: [[golang]]
- # 什么是sync.Pool?
	- 它是一种资源复用的机制。类似连接池、线程池。可以让我们尽可能少地创建新资源，复用以前的资源。
- # 为什么要使用sync.Pool?
	- 这在资源频繁创建的场景下十分有用，能够显著提高程序的性能。针对那种生命周期很短的对象很有用。如果你的对象是长期存在的，那么可能没有必要使用sync.Pool。
- # 内部实现
	- sync.Pool是对象复用的一种技术，它是协程安全的。值得注意的是它在清除对象是自动的（在GC时清除），且不会通知用户，如果此时对象只被Pool持有，那么可能会被释放（存放在victim的资源可能在GC时会被释放）。
	- sync.Pool的定义如下：
	  ```go
	  type Pool struct {
	  	noCopy noCopy
	  	local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
	  	localSize uintptr        // size of the local array
	  	victim     unsafe.Pointer // local from previous cycle
	  	victimSize uintptr        // size of victims array
	  	New func() any
	  }
	  ```
	  
	  其中`noCopy`表示在第一次使用后就不能拷贝，如果出现拷贝，可以配合go vet检测出。`local`其实指向了一个`poolLocal`的数组。长度为P（GMP模型）的数量。每个index下保存着当前P所持有的对象池。`victimSize`保存了上一次清理前的`local`所持有的对象池，这个会在Get、GC的时候会用到，对于平滑性能很重要。`New`是唯一向用户暴露的字段，它表示如果当前pool中没有可用的资源时，会调用该函数去创建新对象然后返回给用户。
	- ## poolLocal
		- `poolLocal`是跟P绑定的，用来存放所有资源对象的结构体。`sync.Pool`**能够在多个goroutinue中并发访问也是因为每个G只访问当前P的poolLocal，单个P一次只能运行一个G，所以在P的视角上是单线程，所以是线程安全的。
		- 在sync.Pool暴露的Put和Get方法中，都会调用`pin`这个方法。因为go支持抢占式调度，所以在拿poolLocal的时候需要禁用抢占，直到操作完成后才把抢占打开。（根据goroutine的特性，G会在各个P中流转，**如果不禁用抢占，可能在get poolLocal前，当前G1被其他G抢占，等到G1在再运行时就跑到了其他的P上，导致拿到的poolLocal不是当前的P所持有的poolLocal**）。代码如下：
		  
		  ```go
		  func (p *Pool) Put(x any) {
		  	// ...
		  	l, _ := p.pin()  // 调用抢占，获取当前P的poolLocal
		  	if l.private == nil {
		  l.private = x
		  x = nil
		  	}
		  	if x != nil {
		  l.shared.pushHead(x)
		  	}
		  	runtime_procUnpin() // 操作完成后，开启抢占
		  	if race.Enabled {
		  race.Enable()
		  	}
		  }
		  ```
		  
		  那`p.pin()`是如何知道poolLocal的呢？上面提到的`local`其实等价于`[P]poolLocal`，使用时根据当前的pid（P id）拿到对应的poolLocal，`p.pin()`中会调用`indexLocal()`获取当前的P，代码如下：
		  
		  ```go
		  func indexLocal(l unsafe.Pointer, i int) *poolLocal {
		  	lp := unsafe.Pointer(uintptr(l) + uintptr(i)*unsafe.Sizeof(poolLocal{}))
		  	return (*poolLocal)(lp)
		  }
		  ```
		  
		  接下来看下poolLocal的数据结构：
		  
		  ```go
		  type poolLocal struct {
		  	poolLocalInternal
		  	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
		  }
		  ```
		  
		  其中`pad`字段是一个填充字节，主要是为了能在一个cache line中。因为sync.Pool是在go runtime内部运行的，所以要求数据的存取尽可能的高性能。`poolLocalInternal`是保存数据真正的结构。
	- ## poolLocalInternal
		- `poolLocalInternal`是存放对象的数据结构。结构如下：
		  
		  ```go
		  type poolLocalInternal struct {
		  	private any       // Can be used only by the respective P.
		  	shared  poolChain // Local P can pushHead/popHead; any P can popTail.
		  }
		  ```
		  
		  `private`字段会作为sync.Pool的Put、Get的第一个操作对象，避免频繁操作`shared`数据结构，也是一种提高性能的手段。
		  
		  poolChain是一个双向链表，链表的每个节点都是环形数组。结构如下：
		  
		  ```go
		  type poolChain struct {
		  	head *poolChainElt
		  	tail *poolChainElt
		  }
		  type poolChainElt struct {
		  	poolDequeue
		  	next, prev *poolChainElt
		  }
		  type poolDequeue struct {
		  	headTail uint64
		  	vals []eface
		  }
		  
		  ```
		  
		  `poolChainElt`中的`poolDequeue`是一个环形数组。
# 如何存取对象
	- ## 存对象
		- 存对象相对简单，上面讲到其实存储对象的数据结构是一个双向链表，链表节点是一个双端队列。首先会在队列首找是否有有效的节点，没有的话新建一个节点，然后把值插入进去。
		  ```go
		  func (c *poolChain) pushHead(val any) {
		  	d := c.head
		  	if d == nil {
		  d = new(poolChainElt)
		  d.vals = make([]eface, initSize)
		  c.head = d
		  storePoolChainElt(&c.tail, d)
		  	}
		  
		  	if d.pushHead(val) {
		  return
		  	}
		  	// 每次新建时都以两倍大小扩充
		  	newSize := len(d.vals) * 2
		  	if newSize >= dequeueLimit {
		  // Can't make it any bigger.
		  newSize = dequeueLimit
		  	}
		  	d2 := &poolChainElt{prev: d}
		  	d2.vals = make([]eface, newSize)
		  	c.head = d2
		  	storePoolChainElt(&d.next, d2)
		  	d2.pushHead(val)
		  }
		  ```
	- ## 取对象
		- 取对象相对复杂一点，除了在本地取，它还会尝试从其他地方中窃取，具体顺序如下：
		  1. private中取。
		  2. private中没有，从本地P的poolLocal取。
		  3. 本地P没有，从其他P的poolLocal中取。
		  4. 其他P中没有，从victim中取。
		  
		  ```go
		  func (p *Pool) Get() any {
		  
		  	l, pid := p.pin()
		  	x := l.private
		  	l.private = nil
		  	if x == nil {
		  x, _ = l.shared.popHead()
		  if x == nil {
		  	// 尝试从其他local、本地的victim窃取
		  	x = p.getSlow(pid)
		  }
		  	}
		  	runtime_procUnpin()
		  	if x == nil && p.New != nil {
		  x = p.New()
		  	}
		  	return x
		  }
		  ```
		  
		  ```go
		  func (p *Pool) getSlow(pid int) any {
		  	// See the comment in pin regarding ordering of the loads.
		  	size := runtime_LoadAcquintptr(&p.localSize) // load-acquire
		  	locals := p.local                            // load-consume
		  	// 尝试从其他的P中窃取
		  	for i := 0; i < int(size); i++ {
		  l := indexLocal(locals, (pid+i+1)%int(size))
		  if x, _ := l.shared.popTail(); x != nil {
		  	return x
		  }
		  	}
		  	// 在其他P中拿不到对象后，会尝试从victim中获取。保证这样顺序是因为，尽可能让vicim的对象
		  	// 能够被GC。
		  	size = atomic.LoadUintptr(&p.victimSize)
		  	if uintptr(pid) >= size {
		  return nil
		  	}
		  	locals = p.victim
		  	l := indexLocal(locals, pid)
		  	if x := l.private; x != nil {
		  l.private = nil
		  return x
		  	}
		  	for i := 0; i < int(size); i++ {
		  l := indexLocal(locals, (pid+i)%int(size))
		  if x, _ := l.shared.popTail(); x != nil {
		  	return x
		  }
		  	}
		  	atomic.StoreUintptr(&p.victimSize, 0)
		  
		  	return nil
		  }
		  ```
- # 对象的清理时机
	- `poolCleanup`负责清理所有P上面的poolLocal，因为在GC的时候，整个程序处于STW，所以可以直接不加锁直接调用这个函数清理所有的poolLocal。这里值得注意的是当前的poolLocal没有直接释放，而是存放到了victim这个变量中。这样做的理由是防止每次GC都把对象清理完，导致GC后的Get需要创建新的对象。有了这一步操作，可以让整个程序性能表现得更加的平滑。
	  
	  ```go
	  func poolCleanup() {
	  	// This function is called with the world stopped, at the beginning of a garbage collection.
	  	// It must not allocate and probably should not call any runtime functions.
	  
	  	// Because the world is stopped, no pool user can be in a
	  	// pinned section (in effect, this has all Ps pinned).
	  	// Drop victim caches from all pools.
	  	for _, p := range oldPools {
	  p.victim = nil
	  p.victimSize = 0
	  	}
	  	// Move primary cache to victim cache.
	  	for _, p := range allPools {
	  p.victim = p.local
	  p.victimSize = p.localSize
	  p.local = nil
	  p.localSize = 0
	  	}
	  	// The pools with non-empty primary caches now have non-empty
	  	// victim caches and no pools have primary caches.
	  	oldPools, allPools = allPools, nil
	  }
	  ```
- # 参考
	- [Deep analysis of Golang sync.Pool underlying principles - SoByte](https://www.sobyte.net/post/2022-03/think-in-sync-pool/)
	- [深度解密Go语言之sync.pool | qcrao](https://www.qcrao.com/2020/04/20/dive-into-go-sync-pool/#pack-unpack)