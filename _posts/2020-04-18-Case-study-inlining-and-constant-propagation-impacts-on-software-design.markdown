---
layout: post
title:  "Case study: How Inlining and constant propagation helps software design"
date:   2021-04-18 10:35:47 -0300
categories: gcc llvm compilers compiler inlining inline optimization propagation constant folding
image: "/assets/img/2021-04-18-PANDA-asm-img.png"
share-img: "/assets/img/2021-04-18-PANDA-asm-shareimg.png"
author: Felipe Magno de Almeida
---

Performance is a big part of software design, and does have impact on
software architecture and API design. However, we will see how we can
leverage everyday optimizations such as inlining to accomplish
performance similar or better to hand-written software in extensible
and maintainable designs.

# Case study

We will talk about the [PANDA from
SiPanda](https://github.com/panda-net/panda) project. Expertise
Solutions wrote the PANDA Compiler that:

* Reads a C file
* extracts network parser relationships into a Graph
* Use this information to generate C code that allowed inlining and
constant propagation to optimize it to similar and in some cases
better performance than hand-written network parser code.
  
The PANDA project goal is to be the fastest network parser without
sacrificing extensibility and readability.

# Software Design

The PANDA Project is written in C, and is extensible by using macros
that define global structures that define hooks for a specific parser
and the relationship between other parsers.

The following code defines a "Parser Node", which is a parser that
handles a specific protocol.

{% highlight C %}
PANDA_MAKE_PARSE_NODE(ether_node, panda_parse_ether, ether_metadata,
		      panda_null_handle_proto, ether_table);
{% endhighlight %}

The `panda_parse_ether` is a global structure that defines various
hooks and constant information about how to parse an ethernet packet and is defined below:

{% highlight C %}
static const struct panda_proto_node panda_parse_ether __unused() = {
	.name = "Ethernet",
	.min_len = sizeof(struct ethhdr),
	.ops.next_proto = ether_proto,
};
{% endhighlight %}

The relationship between protocols is defined as tables below:

{% highlight C %}
PANDA_MAKE_PROTO_TABLE(ether_table,
	{ __cpu_to_be16(ETH_P_IP), &ipv4_check_node },
	{ __cpu_to_be16(ETH_P_IPV6), &ipv6_check_node },
	{ __cpu_to_be16(ETH_P_8021AD), &e8021AD_node },
	{ __cpu_to_be16(ETH_P_8021Q), &e8021Q_node },
	{ __cpu_to_be16(ETH_P_MPLS_UC), &mpls_node },
	{ __cpu_to_be16(ETH_P_MPLS_MC), &mpls_node },
	{ __cpu_to_be16(ETH_P_ARP), &arp_node },
	{ __cpu_to_be16(ETH_P_RARP), &rarp_node },
	{ __cpu_to_be16(ETH_P_TIPC), &tipc_node },
	{ __cpu_to_be16(ETH_P_BATMAN), &batman_node },
	{ __cpu_to_be16(ETH_P_FCOE), &fcoe_node },
	{ __cpu_to_be16(ETH_P_PPP_SES), &pppoe_node },
);
{% endhighlight %}

## Extending

Hence, extending the parser to other protocols becomes trivial. The
user creates another parse node with its own protocol definitions, and
adds this protocol to the tables which can contain this new protocol
in its paylod.

## Meta parser

The user can define its own parser from start to end, by creating new
parsers with different root nodes and defining his own tables and
parse nodes. The user can reuse protocol definitions in the PANDA
library or he can customize it to fit his particular needs.

In that way, the PANDA library doesn't just defiine a generic network
packet parser, but it defines the APIs for users to create their own
network specific packet parsers.

Let's say a user needs to parse just TCP and UDP over IP. It doesn't
make sense for the user that the parser tries to parse SCTP, ICMP and
other packets. By removing those options, the parser can be smaller,
which improves code locality and instruction cache hits and can also
eliminate a huge amount of branch instructions, which helps CPUs do a
better job at branch prediction and also has the CPU run less
instructions for data the user won't be able to use it.

## The Non-optimized implementation

The PANDA project created this API on top of a non-optimized
implementation which looped over each protocol boundary in the packet
parsing process and called the hooks on each parse node which was
looked-up through the tables based on "next protocol" fields.

This was slow because, firstly, looping requires more bookkeeping and
more instructions to run and secondly because every hook, no matter
how small it was, required an indirect function call.

## The PANDA Compiler

We at Expertise Solutions, as experts in Modern C++ programming
already knew, inlining is king for performance. There is some
misunderstanding as to how inlining can be beneficial to performance
because people tend to focus on the performance gains on the `call`
instruction itself, which is not by itself too expensive.

However, inlining allows the compiler to see the target function's
definition and do several interprocedural optimizations that would
otherwise be impossible if the call is made to an unknown function
from the compiller's perspective.

For example, the Ethernet parser node defines a `ether_proto` hook
which retrieves the `id` of the next protocol as defined in its
packet header below:

{% highlight C %}
static inline int ether_proto(const void *veth)
{
	return ((struct ethhdr *)veth)->h_proto;
}
{% endhighlight %}

As you can see, this is a very simple function and it is very likely
that the `veth` is already in one of the registers in the target CPU
when this functions is called. Which, when inlined the compiler can
transform this function call to a _single_ instruction in X86-64
platforms:

{% highlight nasm %}
    movzwl	12(%rdi), %eax	# MEM[(struct ethhdr *)hdr_2(D)].h_proto, _38
{% endhighlight %}

It was not the case, but the value could have already been loaded by
another register load before and so the call would be just basically
free.

And if you actually look at the context, you can see that this
_almost_ happens.

{% highlight nasm %}
	movzwl	12(%rdi), %eax
	leaq	14(%rdi), %r11	#, hdr
	leaq	-14(%rsi), %r10	#, len
	movw	%ax, 88(%r8)
	movq	(%rdi), %rax
	movq	%rax, 2(%r8)
	movl	8(%rdi), %eax
	movl	%eax, 10(%r8)
	movzwl	12(%rdi), %eax
	cmpw	$18312, %ax	#, _38
	je	.L194	#,
	ja	.L183	#,
	cmpw	$1544, %ax	#, _38
	je	.L191	#,
	ja	.L185	#,
	cmpw	$129, %ax	#, _38
	je	.L186	#,
	cmpw	$1347, %ax	#, _38
	jne	.L242	#,
{% endhighlight %}

You can see that the compiler reads from `12(%rdi)` twice in the
assembly above. This probably happens because of register
pressure. The data is loaded before because of metadata extraction but
it could be reused later for the next protocol branching if there were
more registers available.

### Constant propagation

As you can see on the assembly snippet above, there's no function
calls made in the output instructions.

This was, as we already mentioned, because of inlining. However,
inlining is not the only optimization at play. Without constant
propagation no inlining could've happened. As we showed in
`panda_parse_ether` definition, we still set fields with
pointer-to-functions, and as such the compiler still sees an indirect
call, as we can see in the C code output from the PANDA compiler below:


{% highlight C %}
	int ret;
	int type; (void)type;
	const struct panda_parse_node* parse_node =
         (const struct panda_parse_node*)&ether_node;
	const struct panda_proto_node* proto_node =
         parse_node->proto_node;
	ssize_t hlen;
	if ((ret = check_pkt_len(hdr, parse_node->proto_node, len, &hlen))
                  != PANDA_OKAY)
		return ret;
	if (parse_node->ops.extract_metadata)
		parse_node->ops.extract_metadata (hdr, frame);
	if (proto_node->encap && (ret = panda_encap_layer
                        (metadata, max_encaps, &frame, &frame_num)) != 0)
		return ret;

	type = proto_node->ops.next_proto (hdr);
	if (type < 0)
		return type;
	if (!proto_node->overlay) {
		hdr += hlen;
		len -= hlen;
	}
	switch (type) {
	case __cpu_to_be16(ETH_P_IP):
    // ... more switch cases
{% endhighlight %}

As you can see the C code still uses indirect calls and does checks
that do not appear on the final assembly output. This is because the
struct definition is visible to the compiler and is defined as
constant, which guarantees no mutability. Thus, the compiler is able
to make constant propagation for `proto_node->overlay` but also to
`proto_node->ops.next_proto` and so the compiler can substitute this
indirect call to a direct call to `ether_proto` and eliminate the
check for `proto_node->overlay`. The same thing happens to
`extract_metadata`, `check_pkt_len`, `parse_node->min_len`,
`proto_node->encap`, `panda_encap_layer` and etc.

With the indirect call substituted by a direct call, then the compiler
can inline the code as usual. And, as you can see the assembly is even
smaller than the C code.

# Conclusion

If we understand how and what the compiler is capable, we can use this
to our benefit in software design by allowing us to compromise less
and achieve better, more extensible and maintainable designs _without_
sacrificing performance.

Check it out the PANDA Project at their
[website](https://www.sipanda.io) and their
[GitHub](https://github.com/panda-net/panda) repository.
