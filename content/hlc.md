[接上篇](local-timestamps-transaction.md)，由于逻辑时钟不具备一个全局的性质，我们无法用某个时间戳拿到一个快照。混合逻辑时钟可以用于解决这个问题。

混合逻辑时钟存储两部分信息，一部分取值来源于本地时钟(物理部分)，另一部分取值来自于计数器(逻辑部分)。来源于物理的部分，里面保存的其实是当前所有参与节点的本地时钟的最大值，另一部分则是每次事件或者消息通信时加加，类似逻辑时钟里每次事件都加加。将这两部分称为l和c。用一个混合逻辑时钟跟本地时钟比较大小时，先用比较l部分，如果相等再看c是否为零。

混合逻辑时钟的算法描述如下：

本地事件或者发送消息时，如果本地时钟pt大于当前的混合逻辑时钟的l，则将l更新成本地时钟，将c清零。否则，l保持不变，将c加1。

    l'.j = l.j;
    l.j = max(l'.j, pt.j);
    if l.j = l'.j then c.j := c.j + 1 else c.j = 0
    return l.j c.j

收到消息时，l在 当前的逻辑时钟的l、机器的本地时钟pt、收到消息里面带的l，三者中取最大的。如果l部分是更新为本地时钟了，则将c清零。否则，c取较大的那个l对应到的c加1。

    l'.j = l.j;
    l.j = max(l'.j, l.m, pt.j);
    if l.j = l'.j = m.j then c.j = max(c.j, c.m) + 1
    else if l.j = l'.j then c.j = c.j + 1
    else if l.j = l.m then c.j = c.m + 1
    else c.j = 0
    return l.j c.j

通俗理解就是，每个节点都维护了一个自己混合逻辑时钟的当前值。每当发生事件或者消息通信时，这个值都会更新。发出去的消息都会带上自己的时钟。如果收到了消息里面包含更大的本地时钟，则将当前物理部分拉到更大的值。如果消息的物理时间并不比自己大，则根据本地时钟更新。如果物理部分相等，则更新逻辑部分。各节点相互通信的最终结果是，来源于本地时钟的物理部分，最终记录的是所有参与者中最大的本地时钟。

举几个例子，假设本地时钟以秒为单位。当前的混合逻辑时钟到了13秒，计数器在10，当前值记为13.10。几种情况：一种是发生事件本地到14秒了，那么将计数器清零变成14.0。一种是收到更小的消息，比如12.22，那么时间取较大的仍然是13，计数器加一后到了13.11。一种是收到时间相同但计数器更大的消息，比如13.17，那么将当前值更新到13.18。再有一种比如有机器上面时间飘移到20秒，收到它的消息20.0，当前值也会被拉到20，那么就是20.1。

接下来看一下混合逻辑时钟有什么性质。

首先，跟逻辑时钟一样，如果事件e happens before f，那么一定有事件e的混合逻辑时钟小于事件f的混合逻辑时钟：

    e hb f => (l.e, c.e) < (l.f, c.f)
    
这条性质不用多说，能理解逻辑时钟就能理解这个。

性质二，对于任何事件f，l.f >= pt.f 。好理解，似乎没啥用。

性质三，l.f代表了f所知道的各节点的本地时钟的最大值。解释过了，只要遇到消息里面的本地时钟比当前大，就会将当前值拉到那个值。

推论，对于任何事件f，l.f跟pt.f是的差值有上界的。这条是假设机器之间通过NTP对时，可以保证机器之前的本地时钟差值是在一定范围内（NTP对时的精度相关），而l.f其实近似最大的那个本地时钟，所以l.f跟pt.f的误差范围也是有界的。

性质四，如果有事件f并且c.f大于0，那么一定存在某个事件g(l.g=l.f，c.g<c.f)，满足g happens before f。这个简单解释就是说，混合逻辑时钟里面，计数器当前值不可能凭空的变到大于0，肯定是发生了某个事件，那个事件必然是happens before当前的这个计数器不为零的事件。

好，最关键的部分来了，混合逻辑时钟如何用来实现读快照。

假设e和f事件发生成同一节点上，`l.e < l.f` ，我们引入一个虚拟事件g，`l.e+1 <= l.g <= l.f` ，并且 `c.g = 0`，这样的虚拟事件肯定是能找到的，并且引入一个虚构事件并不影响真实事件发生的先后关系。注意了！这个虚拟事件的时钟其实是可以跟全局的节点比较时序的`(l.g, 0)` ! 也就意味着，我们可以拿一个本地时钟去确定一个快照了！

快照是只要happens before我的，我一定能够看到。混合逻辑时钟通过保留了物理时钟部分，使得拿到'全局'的时间戳成为可能，而逻辑时钟里面happens before的因果关系仍然可以保留。

其实理一下思路还是很真观的：想实现事务会需要时钟。本地时钟能不能比较先后？不能。逻辑时钟行不行？能判定happens before关系但无法获得一个全局快照。混合逻辑时钟可以解决这个问题，over。