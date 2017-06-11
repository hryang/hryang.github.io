在分布式系统里，出于可靠性（reliability）或者性能的考虑，数据通常会被复制为多个副本。因此，系统需要定义一组协议，来规定用户读写多副本时的行为。这组协议称之为 [consistency model](https://en.wikipedia.org/wiki/Consistency_model)。
最终一致性（eventually consistency），linearizability（atomic consistency）是不同类型的一致性模型。下图是最终一致性模型的示例，用户 B/C 在读取不同的副本时看到了不同的 x 的值。请注意，用户A，B，C的操作是完全串行执行的，即操作在时间轴上没有重叠。C 读取操作发生在 B 之后，却读到了过期的值。可见，最终一致性是一个较弱的一致性模型，用户需要自己解决读取到过期数据的问题。

![eventually_consistency](https://yqfile.alicdn.com/c3b0a374da12554749991927584a2c267a3161cc.png)

接下来，让我们看看一个中间状态的一致性模型，它比最终一致性强，但是比 linearizability 弱。从时间轴上可以看到，B0 发生在 A0 之前，读取到的 x 值为0。B2 发生在 A0 之后，读取到的 x 值为1。而读操作 B1，C0，C1 与写操作 A0 在时间轴上有重叠，因此他们可能读取到旧的值0，也可能读取到新的值1。注意，C1 发生在 B1 之后（二者在时间轴上没有重叠），但是 B1 看到 x 的新值，C1 反而看到的是旧值。即对用户来说，x 的值发生了回跳。

![casual_consistency](https://yqfile.alicdn.com/6ebc483cb645bc2ef98bbd3fbd95422a4f23d721.png)

Linearizability，则提供了更强的一致性保证。注意下图中的 C1 和 B1 操作，C1 发生在 B1 之后，但是与 A0 并行。在 Linearizable 系统中，如果 B1 看到的 x 值为1，则 C1 看到的值也一定为1。而在前两种一致性模型中，同样的场景，C1 看到的 x 值既可能是0，也可能是1。在 Linearizable 的系统中，任何操作在该系统生效的时刻都对应时间轴上的一个点。如果我们把这些时刻连接起来，如下图中紫线所示，则这条线会一直沿时间轴向前，不会反向回跳。所以，在 Linearizable 系统中，**任何操作**都可以互相比较决定谁发生在前，谁发生在后。例如 B1 发生在 A0 前，C1 发生在 A0 后。而在前面较弱的一致性模型中，我们无法比较诸如 B1 和 A0 的先后关系。对这个性质更正式的表述是：在 Linearizable 系统中，我们可以获得操作的[全序](https://www.quora.com/How-can-you-explain-partial-order-and-total-order-in-simple-terms)（total order）。而在提供较弱一致性的系统中，我们只能获得操作的偏序（partial order）。

![linearizability](https://yqfile.alicdn.com/582959f8441236fa3a8243b11aceb183edc705d1.png)

> 注意：
> 
> 由于网络延时以及系统本身执行请求的不确定性，请求发起得早不意味着在服务端被执行得早。例如 A0 比 B1 在客户端发起得早，但 A0 在存储系统生效晚于 B1。反之，在服务端被执行得早的请求也有可能很晚客户端才收到结果，例如 C0。

需要 Linearizability 语义的一个典型场景是实现分布式锁。例如当一个锁被 client A获取后，则希望后续看到该锁的状态都是被A locked，而不是其他状态。

### 附录

#### A. Linearizability 与 Serializability 的差别

关于二者的差别，可以参考这一篇[文章](https://zhuanlan.zhihu.com/p/26893940)。
