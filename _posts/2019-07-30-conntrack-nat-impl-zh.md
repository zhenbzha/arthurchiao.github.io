---
layout    : post
title     : "连接跟踪与 NAT 在 Linux 内核中的实现"
date      : 2019-07-30
lastupdate: 2019-07-30
categories: netfilter
---

### 摘要

本文对连接跟踪（connection tracking）和 NAT 做一个详细的介绍。

代码分析基于内核 `4.19`。为使行文简洁，本文所贴的大部分代码都做了删减，但每段
代码都给出了其所在的源文件，如有需要请查阅完整代码。

## 引言

Netfilter 是一个对数据包进行**控制、修改和过滤**（manipulation and filtering）的
框架。它在内核协议栈中提供了若干 hook 点，以此对数据包进行拦截、过滤或其他处理。

Netfilter 由几个模块构成，其中最主要的是**连接跟踪**（Connection Tracking，CT）
模块和**网络地址转换**（Network Address Translation，NAT）模块。

连接跟踪，顾名思义，就是**跟踪并记录连接的状态**。但**需要注意的是**，这里的“连
接”和 TCP/IP 协议中所说的“面向连接”（connection oriented）的“连接”并不是同一个概
念：

* TCP/IP 协议中，TCP 是有连接的，UDP 是无连接的
* 连接跟踪系统中，一个元组（tuple）定义的 flow 就表示一条连接。我
  们后面会看到 UDP 甚至是 ICMP 也都是有连接的

在本文中，如无特殊说明，我们所说的连接都是指后者。

连接跟踪最重要的**使用场景**是 NAT。

## Netfilter 框架

Netfilter 在内核协议栈的包处理路径上提供了 5 个 hook 点，分别是：

1. `NF_IP_PRE_ROUTING`
1. `NF_IP_LOCAL_IN`
1. `NF_IP_LOCAL_OUT`
1. `NF_IP_FORWARD`
1. `NF_IP_POST_ROUTING`

定义在 `include/uapi/linux/netfilter_ipv4.h`：

```c
/* only for userspace compatibility */
#ifndef __KERNEL__

/* IP Hooks */
#define NF_IP_PRE_ROUTING	0 /* After promisc drops, checksum checks. */
#define NF_IP_LOCAL_IN		1 /* If the packet is destined for this box. */
#define NF_IP_FORWARD		2 /* If the packet is destined for another interface. */
#define NF_IP_LOCAL_OUT		3 /* Packets coming from a local process. */
#define NF_IP_POST_ROUTING	4 /* Packets about to hit the wire. */
#define NF_IP_NUMHOOKS		5

#endif /* ! __KERNEL__ */
```

另外还有一套 `NF_INET_` 开头的定义，`include/uapi/linux/netfilter.h`：

```c
enum nf_inet_hooks {
	NF_INET_PRE_ROUTING,
	NF_INET_LOCAL_IN,
	NF_INET_FORWARD,
	NF_INET_LOCAL_OUT,
	NF_INET_POST_ROUTING,
	NF_INET_NUMHOOKS
};
```

这两套是等价的，从注释看，`NF_IP_` 开头的定义可能是为了保持兼容性。

hook 的返回值类型（`include/uapi/linux/netfilter.h`）：

1. `NF_ACCEPT`：接受这个包，继续下一步处理
1. `NF_DROP`：丢弃这个包
1. `NF_STOLEN`：当前处理函数已经消费了这个包，后面的处理函数不用处理了
1. `NF_QUEUE`：应当将包放到队列
1. `NF_REPEAT`：当前处理函数应当被再次调用

<p align="center"><img src="/assets/img/netfilter-conntrack-nat-impl/1.png" width="80%" height="80%"></p>
<p align="center">Fig.1. Netfilter IPv4 Hook Traversal</p>

注册到 hook 点的处理函数必须指定**优先级**，这样当触发 hook 时能够根据优先级依次
调用处理函数。

本文主要关注 `NF_IP_PRE_ROUTING` 和 `NF_IP_POST_ROUTING`，这是连接跟踪和 NAT 实
现的地方。

#### 流量类型 1：外部 -> 宿主机程序

首先来看一个以本地为目的的数据包，它要经过以下步骤才能到达要接收它的程序

| Step | Chain | Table | What Happens |
| :--  | :--   | :--   | :--          |
| 1 |              |          | 包到达宿主机网卡 |
| 2 | `PREROUTING` | `mangle` | 修改包的一些头字段，例如 TTL、TOS 等 |
| 3 | `PREROUTING` | `nat`    | 如果需要，做 DNAT `[注 a]` |
| 4 |              |          | 路由判断：目的是本机还是其他其他机器（需要转发） |
| 5 | `INPUT`      | `mangle` | 修改包的一些头字段 `[注 b]`|
| 6 | `INPUT`      | `filter` | 对包过滤 `[注 c]` |
| 7 |              |          | 到达本地程序 |

* `注 a`：例如，如果包的最终目的地是宿主机上的某个容器，那这个包当前的 `dst_ip`
  就是宿主机 IP；接下来宿主机会将它改为容器 IP，然后转发给容器。docker 默认的网
  络模式就是这种。
* `注 b`：路由之后，被送往本地程序之前，修改头字段
* `注 c`：所有以本地为目的的包都要经过这个链，不管它们从哪儿来，对这些包的过滤条
  件就设在这里

#### 流量类型 2：宿主机程序 -> 外部

接着看看以以本地为源的数据包，它需要经过下面的步骤才能发送出去

| Step | Chain | Table | What Happens |
| :--  | :--   | :--   | :--          |
| 1    |               |            | 本地程序发出数据包 |
| 2    |               |            | 路由判断 |
| 3    | `OUTPUT`      | `mangle`   | 修改数据包头 |
| 4    | `OUTPUT`      | `nat`      | 如果需要，做 DNAT `[注 a]` |
| 5    | `OUTPUT`      | `filter`   | 对本地发出的包过滤 |
| 6    | `POSTROUTING` | `mangle`   | `[注 b]` |
| 7    | `POSTROUTING` | `nat`      | 在这里做 SNAT。`[注 c]` |
| 8    |               |            | 从网卡发出 |

* `注 a`：例如，
* `注 b`：这条链主要在包DNAT之后(译者注：作者把这一次DNAT称作实际的路由，虽然在
  前面有一次路由。对于本地的包，一旦它被生成，就必须经过路由代码的处理，但这个包
  具体到哪儿去，要由NAT代码处理之后才能确定。所以把这称作实际的路由。)，离开本地
  之前，对包mangle。有两种包会经过这里，防火墙所在机子本身产生的包，还有被转发的
  包。
* `注 c`：但不要在这里做过滤，因为有副作用，而且有些包是会溜过去的，即使你用了DROP策略。

#### 流量类型 3：外部 -> 宿主机转发 -> 外部

目的是另一个网络中的一台机子

| Step | Chain | Table | What Happens |
| :--  | :--   | :--   | :--          |
| 1    |                |          | 包被网卡接收 |
| 2    | `PREROUTING`   | `mangle` | 修改数据包的一些头字段，例如TTL、TOS等 |
| 3    | `PREROUTING`   | `nat`    | 做DNAT。例如包的最终目的地是宿主机上的走NAT网络的容器 |
| 4    |                |          | 路由判断：目的是本机还是其他其他机器（需要转发） |
| 5    | `FORWARD`      | `mangle` | 包继续被发送至mangle表的FORWARD链，这是非常特殊的情况才会用到的。在这里，包被mangle。这次mangle发生在最初的路由判断之后，在最后一次更改包的目的之前（译者注：就是下面的FORWARD链所做的，因其过滤功能，可能会改变一些包的目的地，如丢弃包）。 |
| 6    | `FORWARD`      | `filter` | 包继续被发送至这条FORWARD链。只有需要转发的包才会走到这里，并且针对这些包的所有过滤也在这里进行。注意，所有要转发的包都要经过这里，不管是外网到内网的还是内网到外网的。在你自己书写规则时，要考虑到这一点。 |
| 7    | `POSTROUTING`  | `mangle` | 这个链也是针对一些特殊类型的包（译者注：参考第6步，我们可以发现，在转发包时，mangle表的两个链都用在特殊的应用上）。这一步mangle是在所有更改包的目的地址的操作完成之后做的，但这时包还在本地上。 |
| 8    | `POSTROUTING`  | `nat`    | 这个链就是用来做SNAT的，当然也包括Masquerade（伪装）。但不要在这儿做过滤，因为某些包即使不满足条件也会通过。 |
| 9    |                |          | 从网卡发出


## 连接跟踪（CT）

连接跟踪模块用于维护**可跟踪协议**（trackable protocols）的连接的状态。也就是说
，连接跟踪针对的是特定协议的包，而不是所有协议的包。我们稍后会看到它支持哪些协议
。

连接跟踪模块和 NAT 模块是独立的，但**其主要用途就是做 NAT**。

### 元组（Tuple）

Tuple 是连接跟踪中最重要的概念，一个 tuple 定义一个单向（unidirectional）flow。
`include/net/netfilter/nf_conntrack_tuple.h` 中有如下注释：

> A `tuple` is a structure containing the information to uniquely
> identify a connection.  ie. if two packets have the same tuple, they
> are in the same connection; if not, they are not.

结构体定义 `struct nf_conntrack_tuple`，
`include/net/netfilter/nf_conntrack_tuple.h`:

```c
// We divide the structure along "manipulatable" and
// "non-manipulatable" lines, for the benefit of the NAT code.

struct nf_conntrack_man { /* The manipulable part of the tuple. */
	union nf_inet_addr u3;
	union nf_conntrack_man_proto u;
	u_int16_t l3num; /* Layer 3 protocol */
};

struct nf_conntrack_tuple { /* This contains the information to distinguish a connection. */
	struct nf_conntrack_man src;
	struct { /* These are the parts of the tuple which are fixed. */
		union nf_inet_addr u3;
		union {
			/* Add other protocols here. */
			__be16 all;

			struct { __be16 port;         } tcp;
			struct { __be16 port;         } udp;
			struct { u_int8_t type, code; } icmp;
			struct { __be16 port;         } dccp;
			struct { __be16 port;         } sctp;
			struct { __be16 key;          } gre;
		} u;
		u_int8_t protonum; /* The protocol. */
		u_int8_t dir; /* The direction (for tuplehash) */
	} dst;
};
```

以 IPv4 UDP 为例，五元组分别保存在如下字段：

* `dst.protonum`：协议类型
* `src.u3.ip`：源 IP 地址
* `dst.u3.ip`：目的 IP 地址
* `src.u.udp.port`：源端口号
* `dst.u.udp.port`：目的端口号

其中的 `struct nf_conntrack_man_proto` 和 `struct nf_inet_addr` 两个结构体的定义：

`include/uapi/linux/netfilter/nf_conntrack_tuple_common.h`:

```c
/* The protocol-specific manipulable parts of the tuple: always in network order */
union nf_conntrack_man_proto {
	/* Add other protocols here. */
	__be16 all;

	struct { __be16 port; } tcp;
	struct { __be16 port; } udp;
	struct { __be16 id;   } icmp;
	struct { __be16 port; } dccp;
	struct { __be16 port; } sctp;
	struct { __be16 key;  } gre;	/* GRE key is 32bit, PPtP only uses 16bit */
};
```

`include/uapi/linux/netfilter.h`:

```c
union nf_inet_addr {
	__u32		all[4];
	__be32		ip;
	__be32		ip6[4];
	struct in_addr	in;
	struct in6_addr	in6;
};
```

另外从以上定义可以看到，连接跟踪模块目前只支持以下六种协议：

* TCP
* UDP
* ICMP
* DCCP
* SCTP
* GRE

**注意其中的 ICMP 协议**。很多人可能会认为，连接跟踪模块依据包的三层和四层信息做
哈希，而 ICMP 是三层协议，没有四层信息，因此 ICMP 肯定不会被 CT 记录。但**实际上
是会的**，上面代码可以看到，ICMP 使用了其头信息中的 ICMP `type`和 `code` 字段来
定义 tuple。

支持连接跟踪的协议都需要实现 `struct nf_conntrack_l4proto` 结构体（
`include/net/netfilter/nf_conntrack_l4proto.h`）中定义的方法，例如 `pkt_to_tuple()`。

由于传输层协议可能是面向连接的，因此 `struct nf_conntrack_l4proto` 中还包含了一
些和连接状态相关的方法。例如：

* `get_timeouts()`：获取协议的 timeout 值
* `error()`：检查当前数据包能否被连接跟踪
* `new()`：当检测到是一个新连接的时候调用
* `packet()`：经过 `error()` 判断数据包可以跟踪，接下来就调用这个方法
* `destroy()`：删除一条连接的时候调用

### Hashing

将活动连接的状态存储在一张哈希表中。

`hash_conntrack_raw` 根据 tuple 计算出一个 32 位的哈希值（
`net/netfilter/nf_conntrack_core.c`）：

```c
static u32 hash_conntrack_raw(const struct nf_conntrack_tuple *tuple, const struct net *net)
```

`nf_conntrack_tuple_hash` 是哈希表中的表项（
`include/net/netfilter/nf_conntrack_tuple.h`）:

```c
/* Connections have two entries in the hash table: one for each way */
struct nf_conntrack_tuple_hash {
    struct hlist_nulls_node hnnode;
    struct nf_conntrack_tuple tuple;
};
```

其中的 `hnnode` 用于解决哈希冲突。

### Connection

在 Netfilter 中每个 flow 都称为一个 connection，即使是对那些非面向连接的协议（例
如 UDP）。每个 connection 用 `struct nf_conn` 表示（
`include/net/netfilter/nf_conntrack.h`），主要字段如下：

```c
struct nf_conn {
	struct nf_conntrack ct_general;

	struct nf_conntrack_zone zone;
	struct nf_conntrack_tuple_hash tuplehash[IP_CT_DIR_MAX];

	unsigned long status; /* Have we seen traffic both ways yet? (bitset) */
	u32 timeout;          /* jiffies32 when this ct is considered dead */

	possible_net_t ct_net;

	struct hlist_node	nat_bysource;

	struct nf_conn *master; /* If we were expected by an expectation, this will be it */

	u_int32_t mark;
	u_int32_t secmark;

	union nf_conntrack_proto proto; /* Storage reserved for other modules, must be the last member */
};
```

其中：

1. `tuplehash[IP_CT_DIR_MAX]` 分别表示两个方向的 flow
1. `timeout` 是这个连接的状态（connection state）的定时器
1. `status` 每个协议使用的 bit 不一样，表示状态信息（`enum ip_conntrack_status`）

连接跟踪的状态集合 `ip_conntrack_status` 定义在
`include/uapi/linux/netfilter/nf_conntrack_common.h`：

```c
/* Bitset representing status of connection. */
enum ip_conntrack_status {
	IPS_EXPECTED      = (1 << IPS_EXPECTED_BIT),
	IPS_SEEN_REPLY    = (1 << IPS_SEEN_REPLY_BIT),
	IPS_ASSURED       = (1 << IPS_ASSURED_BIT),
	IPS_CONFIRMED     = (1 << IPS_CONFIRMED_BIT),
	IPS_SRC_NAT       = (1 << IPS_SRC_NAT_BIT),
	IPS_DST_NAT       = (1 << IPS_DST_NAT_BIT),
	IPS_NAT_MASK      = (IPS_DST_NAT | IPS_SRC_NAT),
	IPS_SEQ_ADJUST    = (1 << IPS_SEQ_ADJUST_BIT),
	IPS_SRC_NAT_DONE  = (1 << IPS_SRC_NAT_DONE_BIT),
	IPS_DST_NAT_DONE  = (1 << IPS_DST_NAT_DONE_BIT),
	IPS_NAT_DONE_MASK = (IPS_DST_NAT_DONE | IPS_SRC_NAT_DONE),
	IPS_DYING         = (1 << IPS_DYING_BIT),
	IPS_FIXED_TIMEOUT = (1 << IPS_FIXED_TIMEOUT_BIT),
	IPS_TEMPLATE      = (1 << IPS_TEMPLATE_BIT),
	IPS_UNTRACKED     = (1 << IPS_UNTRACKED_BIT),
	IPS_HELPER        = (1 << IPS_HELPER_BIT),
	IPS_OFFLOAD       = (1 << IPS_OFFLOAD_BIT),

	IPS_UNCHANGEABLE_MASK = (IPS_NAT_DONE_MASK | IPS_NAT_MASK |
				 IPS_EXPECTED | IPS_CONFIRMED | IPS_DYING |
				 IPS_SEQ_ADJUST | IPS_TEMPLATE | IPS_OFFLOAD),
};
```

### Tracking

Netfilter 使用三个 Hook 对包进行跟踪：

1. `NF_INET_PRE_ROUTING`：调用 `nf_conntrack_in`
1. `NF_INET_LOCAL_OUT`：调用 `nf_conntrack_in`
1. `NF_INET_POST_ROUTING`：调用 `nf_conntrack_confirm`

**`nf_conntrack_in` 函数是连接跟踪模块的核心**，定义在
`net/netfilter/nf_conntrack_core.c`。下面的代码删去了一些错误处理部分，只展示了
正常的流程：

```c
unsigned int
nf_conntrack_in(struct net *net, u_int8_t pf, unsigned int hooknum, struct sk_buff *skb)
{
	tmpl = nf_ct_get(skb, &ctinfo); // 获取 skb 对应的 conntrack_info 和 conntrack 记录 struct nf_conn *tmpl
	if (tmpl || ctinfo == IP_CT_UNTRACKED) { // 如果记录存在，或者是不需要跟踪的类型
		if ((tmpl && !nf_ct_is_template(tmpl)) || ctinfo == IP_CT_UNTRACKED) {
			NF_CT_STAT_INC_ATOMIC(net, ignore);   // 判断是不需要跟踪的类型，增加 ignore 类型计数
			return NF_ACCEPT;                     // 返回 NF_ACCEPT，继续后面的处理
		}
		skb->_nfct = 0;  // 如果 skb 不属于 ignore 类型，则计数器置零，准备做后续处理
	}

	dataoff = get_l4proto(skb, skb_network_offset(skb), pf, &protonum); // 找到 skb 的 L4 header 地址偏置
	l4proto = __nf_ct_l4proto_find(pf, protonum); // 提取头信息，初始化一个协议相关的 struct nf_conntrack_l4proto 变量
	if (l4proto->error != NULL) {
		l4proto->error(net, tmpl, skb, dataoff, pf, hooknum); // skb 的完整性和合法性验证
	}
repeat:
	// 开始连接跟踪：提取 tuple；创建新连接记录，或者更新已有连接的状态
	ret = resolve_normal_ct(net, tmpl, skb, dataoff, pf, protonum, l4proto);

	l4proto->packet(ct, skb, dataoff, ctinfo); // 进行一些协议相关的处理，例如 UDP 会更新 timeout
	if (ctinfo == IP_CT_ESTABLISHED_REPLY && !test_and_set_bit(IPS_SEEN_REPLY_BIT, &ct->status))
		nf_conntrack_event_cache(IPCT_REPLY, ct);
out:
	if (tmpl)
		nf_ct_put(tmpl); // 解除对 conntrack 记录 tmpl 的引用
	return ret;
}
```

大致流程：

1. 首先尝试获取这个 skb 对应的连接跟踪记录
1. 然后检查是否需要对这个包做连接跟踪，如果不需要，就更新 ignore 计数，返回
   `NF_ACCEPT`；如果需要跟踪，就清空计数器
1. 然后找到这个 skb 的四层头地址，从中提取信息，初始化一个协议相关的 `struct
   nf_conntrack_l4proto` 类型变量。这个结构体我们前面介绍过了，每种支持连接跟踪
   的协议会实现它规定的方法
1. 调用协议相关的 `error()` 检查包的完整性、校验和等信息
1. 调用 `resolve_normal_ct()` 开始进行连接跟踪处理，它会创建 tuple，创建新连接的
   记录，或者更新已有连接的状态（`pkg_to_tuple`、`init_conntrack()`）
1. 调用协议相关的`packet()` 方法进行一些协议相关的处理，例如对于 UDP，如果
   status bit 里面设置了 `IPS_SEEN_REPLY` 位，就会更新 timeout。timeout 大小和协
   议相关，越小越越可以防止 DoS 攻击（DoS 的基本原理就是将机器的可用连接耗尽）

#### 创建新连接记录

连接不存在的时候（flow 的第一个包），`resolve_normal_ct` 会调用 `init_conntrack1`
创建一个新的连接记录。

如果当前包会影响到后面的包的状态判断，那 `init_conntrack()`函数会设置 `nf_conn`
的 `master` 字段。面向连接的协议会用到这个特性，例如 TCP。新创建的记录会插入到一
个 **未确认连接**（unconfirmed connection）列表。如果这个包之后没有被丢弃，那它
在经过 `NF_INET_POST_ROUTING` hook 时被 `nf_conntrack_confirm`方法调用。这个函数
用于确认包确实是发到网络上了，而没有被其他模块丢弃。经过这个函数之后，状态就变为
了 `IPS_CONFIRMED`，并且连接记录从**未确认列表**移到正常的列表。

## NAT

NAT 是与连接跟踪独立的模块。NAT 分为两种：

* SNAT：对源 IP、源 Port 进行修改（转换）
* DNAT：对目的 IP、目的 Port 进行修改（转换）

每种支持 NAT 的协议需要实现 `struct nf_nat_l3proto` 中的方法，
`include/net/netfilter/nf_nat_l3proto.h`：

```c
struct nf_nat_l3proto {
	u8	l3proto;

	bool	(*in_range)(const struct nf_conntrack_tuple *t,
                 const struct nf_nat_range2 *range);

	u32 	(*secure_port)(const struct nf_conntrack_tuple *t, __be16);

	bool	(*manip_pkt)(struct sk_buff *skb, unsigned int iphdroff,
			     const struct nf_nat_l4proto *l4proto,
			     const struct nf_conntrack_tuple *target,
			     enum nf_nat_manip_type maniptype);

	void	(*csum_update)(struct sk_buff *skb, unsigned int iphdroff,
			       __sum16 *check, const struct nf_conntrack_tuple *t,
			       enum nf_nat_manip_type maniptype);

	void	(*csum_recalc)(struct sk_buff *skb, u8 proto,
			       void *data, __sum16 *check, int datalen, int oldlen);

	void	(*decode_session)(struct sk_buff *skb, const struct nf_conn *ct,
				  enum ip_conntrack_dir dir, unsigned long statusbit,
				  struct flowi *fl);

	int	(*nlattr_to_range)(struct nlattr *tb[], struct nf_nat_range2 *range);
};
```

`struct nf_nat_l4proto` 中的方法，
`include/net/netfilter/nf_nat_l4proto.h`：

```c
struct nf_nat_l4proto {
	/* Protocol number. */
	u8 l4proto;

	/* Translate a packet to the target according to manip type. */
	bool (*manip_pkt)(struct sk_buff *skb,
			  const struct nf_nat_l3proto *l3proto,
			  unsigned int iphdroff, unsigned int hdroff,
			  const struct nf_conntrack_tuple *tuple,
			  enum nf_nat_manip_type maniptype);

	/* Is the manipable part of the tuple between min and max incl? */
	bool (*in_range)(const struct nf_conntrack_tuple *tuple,
			 enum nf_nat_manip_type maniptype,
			 const union nf_conntrack_man_proto *min,
			 const union nf_conntrack_man_proto *max);

	/* Alter the per-proto part of the tuple (depending on
	 * maniptype), to give a unique tuple in the given range if
	 * possible.  Per-protocol part of tuple is initialized to the
	 * incoming packet.  */
	void (*unique_tuple)(const struct nf_nat_l3proto *l3proto,
			     struct nf_conntrack_tuple *tuple,
			     const struct nf_nat_range2 *range,
			     enum nf_nat_manip_type maniptype,
			     const struct nf_conn *ct);

	int (*nlattr_to_range)(struct nlattr *tb[], struct nf_nat_range2 *range);
};
```

1. `manip_pkt` 方法根据传入的 tuple 和 NAT 类型（SNAT/DNAT）修改包的 L3/L4 头
1. `unique_tuple` 方法分配可用的 ID 信息，例如对 UDP 协议，分配一个 16 bit 可用
   的端口号

**NAT 的核心函数是 `nf_nat_inet_fn()`**，它会在以下 hook 点被调用：

* `NF_INET_PRE_ROUTING`
* `NF_INET_POST_ROUTING`
* `NF_INET_LOCAL_OUT`
* `NF_INET_LOCAL_IN`

也就是**除了 `NF_INET_FORWARD` 之外其他 hook 点都会被调用**。在这些 hook 点的优
先级：**Conntrack > NAT > Packet Filtering**。**连接跟踪的优先级高于 NAT 是因为
NAT 依赖连接跟踪的结果**。

```c
unsigned int
nf_nat_inet_fn(void *priv, struct sk_buff *skb, const struct nf_hook_state *state)
{
	maniptype = HOOK2MANIP(state->hook); /* maniptype == SRC for postrouting. */
	ct = nf_ct_get(skb, &ctinfo);
	if (!ct) // 如果 conntrack 不存在就做不了 NAT，直接返回
		return NF_ACCEPT;

	nat = nfct_nat(ct);

	switch (ctinfo) {
	case IP_CT_RELATED:
	case IP_CT_RELATED_REPLY: /* Only ICMPs can be IP_CT_IS_REPLY.  Fallthrough */
	case IP_CT_NEW: /* Seen it before? This can happen for loopback, retrans, or local packets. */
		if (!nf_nat_initialized(ct, maniptype)) {
			struct nf_hook_entries *e = rcu_dereference(lpriv->entries);
			if (!e)
				goto null_bind;

			for (i = 0; i < e->num_hook_entries; i++) {
				ret = e->hooks[i].hook(e->hooks[i].priv, skb, state);
				if (ret != NF_ACCEPT)
					return ret;
				if (nf_nat_initialized(ct, maniptype))
					goto do_nat;
			}
null_bind:
			nf_nat_alloc_null_binding(ct, state->hook);
		} else { // Already setup manip
			if (nf_nat_oif_changed(state->hook, ctinfo, nat, state->out))
				goto oif_changed;
		}
		break;
	default: /* ESTABLISHED */
		if (nf_nat_oif_changed(state->hook, ctinfo, nat, state->out))
			goto oif_changed;
	}
do_nat:
	return nf_nat_packet(ct, ctinfo, state->hook, skb);
oif_changed:
	nf_ct_kill_acct(ct, ctinfo, skb);
	return NF_DROP;
}
```

首先查询 conntrack 记录，如果不存在，就意味着无法跟踪这个连接，那就更不可能做
NAT 了，因此直接返回。

如果找到了 conntrack 记录，并且是 `IP_CT_RELATED`、`IP_CT_RELATED_REPLY` 或
`IP_CT_NEW` 状态，就去获取 NAT 规则。如果没有规则，直接返回 `NF_ACCEPT`，对包不
做任何改动；如果有规则，最后执行 `nf_nat_packet`，这个函数会进一步调用
`manip_pkt` 完成对包的修改，如果失败，包将被丢弃。

Masquerade 模块。

## References

1. Boye, Magnus. ["Netfilter connection tracking and NAT implementation"](https://wiki.aalto.fi/download/attachments/69901948/netfilter-paper.pdf). Proc.
   Seminar on Network Protocols in Operating Systems, Dept. Commun. and
   Networking, Aalto Univ. 2013.
