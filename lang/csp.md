## 使用通信来共享内存

相信学过Go的同学都会对这一句话很熟悉：不要通过共享内存来通信，应该使用通信来共享内存。

这句话很好理解，但是Go为什么要推崇这种设计哲学呢？

其实，通过通信来共享内存不只是Go推崇对哲学，Erlang也遵循了同样对设计理念，只不过实现上有所区别，前者深受CSP模型的影响，使用通信顺序进程；
而Erlang使用的是Actor模型设计。Go语言的并发模型深受CSP模型的影响

不管是哪种实现，它们的作用都是在不同的线程或者协程之间交换信息。往深了说，计算机上的线程和协程同步信息都是通过共享内存进行的，因为不管是使用哪种通信
模型，线程和协程最终都会从内存获取数据。说到这里，面对信息的同步，为什么是使用类似channel这样的发送信息的方式来同步，而不是直接使用多个线程或者协程来直接共享内存呢？

#### 更好的抽象

不管是采用发送信息还是直接共享内存，本质上看这两者其实都是作为信息传递的方式，可以理解为是不同等级的抽象，不同语言在实现这些抽象的时候
都会使用操作系统提供的锁机制，因为存在并发问题，共享内存这种相对原始的信息传递的方式就是通过使用锁来处理并发的。这里提到了不同等级的抽象，可以简单理解
为，更高级的信息传递方式其实只是对低抽象等级的接口的组合和封装（- -｜别误会，这里不是在说面向对象），在Go中，channel就给协程提供了信息传递的方式，内部实现就
广泛使用到了共享内存和锁，不管是一开始的悲观锁实现，还是后面的乐观锁实现，通过对共享内存和锁进行组合，对外也就提供了更为高级的同步机制，当然，对于语言的使用者来说，
不用去考虑封装下面内容，但是也存在不适合的场景，比如需要更细粒度的掌控或者说是有更高的性能要求的时候。

#### 更少的耦合

这个就比较浅显了，假设一个场景，当我们需要在多个协程之间传递信息的时候，我们采用共享内存，那么会遇到什么问题呢？每个协程都有可能是彼此的生产者和消费者，
这个时候，在读取/写入的时候就需要对资源进行加锁操作了。

相反，如果我们将信息通过发送的形式，这些协程或者说线程就可以做到解耦，同时协程彼此也不需要自己手动去处理资源的获取和释放，同时，针对需要新增的消费者/生产者这种情况，也不需要做
过多的处理，在channel两端直接增加就好了。

#### 更少的竞争

Go在实现上通过Channel保证被共享的变量不会同时被多个活跃的协程访问，一旦某个消息被发送到了Channel中，我们就失去了当前消息的控制权，
作为接受者的Goroutine在收到这条消息之后就可以根据该消息进行一些计算任务；从这个过程来看，消息在被发送前只由发送方进行访问，在发
送之后仅可被唯一的接受者访问，所以从这个设计上来看我们就避免了线程竞争。可以简单说，就是Go的这种设计，能够天然地避免线程竞争和数据冲突的问题。
（当然，如果发送的是指针，仍然会造成数据冲突）。当然这么设计只能说是 降低而不能完全避免。有些场景不得不使用较为底层的互斥锁，这无可厚非，不过在大多数情况下，一昧的添加互斥锁并不一定是正确的设计。


