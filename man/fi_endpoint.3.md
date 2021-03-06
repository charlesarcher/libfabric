---
layout: page
title: fi_endpoint(3)
tagline: Libfabric Programmer's Manual
---
{% include JB/setup %}

# NAME

fi_endpoint \- Fabric endpoint operations

fi_endpoint / fi_pendpoint / fi_close
:   Allocate or close an endpoint.

fi_ep_bind
:   Associate an endpoint with an event queue, completion queue, address
    vector, or memory region

fi_enable
:   Transitions an endpoint into an active state.

fi_cancel
:   Cancel a pending asynchronous data transfer

fi_alias
:   Create an alias to the endpoint

fi_control
:   Control endpoint operation.

fi_getopt / fi_setopt
:   Get or set endpoint options.

fi_rx_context / fi_tx_context / fi_srx_context  / fi_stx_context
:   Open a transmit or receive context.

# SYNOPSIS

{% highlight c %}
#include <rdma/fabric.h>

#include <rdma/fi_endpoint.h>

int fi_endpoint(struct fid_domain *domain, struct fi_info *info,
    struct fid_ep **ep, void *context);

int fi_pendpoint(struct fi_fabric *fabric, struct fi_info *info,
    struct fid_pep **pep, void *context);

int fi_tx_context(struct fid_ep *ep, int index,
    struct fi_tx_ctx_attr *attr, struct fid_ep **tx_ep,
    void *context);

int fi_rx_context(struct fid_ep *ep, int index,
    struct fi_rx_ctx_attr *attr, struct fid_ep **rx_ep,
    void *context);

int fi_stx_context(struct fid_domain *domain,
    struct fi_tx_ctx_attr *attr, struct fid_stx **stx,
    void *context);

int fi_srx_context(struct fid_domain *domain,
    struct fi_rx_ctx_attr *attr, struct fid_ep **rx_ep,
    void *context);

int fi_close(struct fid *ep);

int fi_ep_bind(struct fid_ep *ep, struct fid *fid, uint64_t flags);

int fi_enable(struct fid_ep *ep);

int fi_cancel(struct fid_ep *ep, void *context);

int fi_alias(struct fid_ep *ep, fid_t *alias_ep, uint64_t flags);

int fi_control(struct fid *ep, int command, void *arg);

int fi_getopt(struct fid_ *ep, int level, int optname,
    void *optval, size_t *optlen);

int fi_setopt(struct fid *ep, int level, int optname,
    const void *optval, size_t optlen);
{% endhighlight %}

# ARGUMENTS

*fid*
: On creation, specifies a fabric or access domain.  On bind,
  identifies the event queue, completion queue or address vector to
  bind to the endpoint.

*info*
: Details about the fabric interface endpoint to be opened, obtained
  from fi_getinfo.

*ep*
: A fabric endpoint.

*fid*
: Fabric identifier of an associated resource.

*context*
: Context associated with the endpoint or asynchronous operation.

*flags*
: Additional flags to apply to the operation.

*command*
: Command of control operation to perform on endpoint.

*arg*
: Optional control argument

*level*
: Protocol level at which the desired option resides.

*optname*
: The protocol option to read or set.

*optval*
: The option value that was read or to set.

*optlen*
: The size of the optval buffer.

# DESCRIPTION

Endpoints are transport level communication portals.  There are two
types of endpoints: active and passive.  Passive endpoints belong to a
fabric domain and are used to listen for incoming connection requests.
Active endpoints belong to access domains and can perform data
transfers.

Data transfer interfaces are bound to active endpoints.  Active
endpoints may be connection-oriented or connectionless, and may
provide data reliability.

Active endpoints are created in the disabled state.  They must
transition into an enabled state before accepting data transfer
operations, including posting of receive buffers.  The fi_enable call
is used to transition an endpoint into an active enabled state.  The
fi_connect and fi_accept calls will also transition an endpoint into
the enabled state, if it is not already active.

In order to transition an endpoint into an enabled state, it must be
bound to one or more fabric resources.  An endpoint that will generate
asynchronous completions, either through data transfer operations or
communication establishment events, must be bound to the appropriate
completion queues or event queues before being enabled.

Once an endpoint has been activated, it may be associated with memory
regions and address vectors.  Receive buffers may be posted to it, and
calls may be made to connection establishment routines.
Connectionless endpoints may also perform data transfers.

The behavior of an endpoint may be adjusted by setting its control
data and protocol options.  This allows the underlying provider to
redirect function calls to implementations optimized to meet the
desired application behavior.

## fi_endpoint / fi_pendpoint

fi_endpoint allocates a new active endpoint.  fi_pendpoint allocates a
new passive endpoint.  The properties and behavior of the endpoint are
defined based on the provided struct fi_info.  See fi_getinfo for
additional details on fi_info.  fi_info flags that control the
operation of an endpoint are defined below.

If an active endpoint is associated with a connection request, the
fi_info connreq must reference the corresponding request.

## fi_close

Closes an endpoint and release all resources associated with it.

## fi_ep_bind

fi_ep_bind is used to associate an endpoint with hardware resources.
The common use of fi_ep_bind is to direct asynchronous operations
associated with an endpoint to a completion queue.  An endpoint must
be bound with CQs capable of reporting completions for any
asynchronous operation initiated on the endpoint.  This is true even
for endpoints which are configured to suppress successful completions,
in order that operations that complete in error may be reported to the
user.  For passive endpoints, this requires binding the endpoint with
an EQ that supports the communication management (CM) domain.

An active endpoint may direct asynchronous completions to different
CQs, based on the type of operation.  This is specified using
fi_ep_bind flags.  The following flags may be used separately or OR'ed
together when binding an endpoint to a completion domain CQ.

*FI_SEND*
: Directs the completion of outbound data transfer requests to the
  specified completion queue.  This includes send message, RMA, and
  atomic operations.

*FI_RECV*
: Directs the notification of inbound data transfers to the specified
  completion queue.  This includes received messages.

*FI_COMPLETION*
: If FI_COMPLETION is specified, the indicated data transfer
  operations won't generate entries for successful completions in the
  completion queue unless FI_COMPLETION is set for that specific
  operation.  FI_COMPLETION must be OR'ed with FI_SEND and/or FI_RECV
  flags.

  When set the user must determine when a request that does NOT have
  FI_COMPLETION set has completed indirectly, usually based on the
  completion of a subsequent operation.  Use of this flag may improve
  performance by allowing the provider to avoid writing a completion
  entry for every operation.

  The use of FI_COMPLETION is often paired with the call fi_sync.
  FI_COMPLETION allows the user to suppress completions from being
  generated.  In order for the application to ensure that all previous
  operations have completed, the application may call fi_sync.  The
  successful completion of fi_sync indicates that all prior operations
  have completed successfully.

An endpoint may also, or instead, be bound to a fabric counter.  When
binding an endpoint to a counter, the following flags may be specified.

*FI_SEND*
: Increments the specified counter whenever a successful message is
  transferred over the endpoint.  Sent messages include both tagged
  and normal message operations.

*FI_RECV*
: Increments the specified counter whenever a successful message is
  received over the endpoint.  Received messages include both tagged
  and normal message operations.

*FI_READ*
: Increments the specified counter whenever a successful RMA read or
  atomic fetch operation is initiated from the endpoint.

*FI_WRITE*
: Increments the specified counter whenever a successful RMA write or
  atomic operation is initiated from the endpoint.

*FI_REMOTE_READ*
: Increments the specified counter whenever a successful RMA read or
  atomic fetch operation is initiated from a remote endpoint that
  targets the given endpoint.

*FI_REMOTE_WRITE*
: Increments the specified counter whenever a successful RMA write or
  atomic operation is initiated from a remote endpoint that targets
  the given endpoint.

Connectionless endpoints must be bound to a single address vector.

## fi_enable

This call transitions the endpoint into an enabled state.  An endpoint
must be enabled before it may be used to perform data transfers.
Enabling an endpoint typically results in hardware resources being
assigned to it.

Calling connect or accept on an endpoint will implicitly enable an
endpoint if it has not already been enabled.

## fi_cancel

fi_cancel attempts to cancel an outstanding asynchronous operation.
The endpoint must have been configured to support cancelable
operations -- see FI_CANCEL flag -- in order for this call to succeed.
Canceling an operation causes the fabric provider to search for the
operation and, if it is still pending, complete it as having been
canceled.  The cancel operation will complete within a bounded period
of time.

## fi_alias

This call creates an alias to the specified endpoint.  Conceptually,
an endpoint alias provides an alternate software path from the
application to the underlying provider hardware.  Applications
configure an alias endpoint with data transfer flags, specified
through the fi_alias call.  Typically, the data transfer flags will be
different than those assigned to the actual endpoint.  The alias
mechanism allows a single endpoint to have multiple optimized software
interfaces.  All allocated aliases must be closed for the underlying
endpoint to be released.

## fi_control

The control operation is used to adjust the default behavior of an
endpoint.  It allows the underlying provider to redirect function
calls to implementations optimized to meet the desired application
behavior.  As a result, calls to fi_ep_control must be serialized
against all other calls to an endpoint.

The base operation of an endpoint is selected during creation using
struct fi_info.  The following control commands and arguments may be
assigned to an endpoint.

**FI_GETOPSFLAG -- uint64_t *flags**
: Used to retrieve the current value of flags associated with data
  transfer operations initiated on the endpoint.  See below for a list
  of control flags.

**FI_SETOPSFLAG -- uint64_t *flags**
: Used to change the data transfer operation flags associated with an
  endpoint.  The FI_READ, FI_WRITE, FI_SEND, FI_RECV flags indicate
  the type of data transfer that the flags should apply to, with other
  flags OR'ed in.  Valid control flags are defined below.

## fi_getopt / fi_setopt

Endpoint protocol operations may be retrieved using fi_getopt or set
using fi_setopt.  Applications specify the level that a desired option
exists, identify the option, and provide input/output buffers to get
or set the option.  fi_setopt provides an application a way to adjust
low-level protocol and implementation specific details of an endpoint.

The following option levels and option names and parameters are defined.

*FI_OPT_ENDPOINT*

- *FI_OPT_MIN_MULTI_RECV - size_t*
: Defines the minimum receive buffer space available when the receive
  buffer is automatically freed (see FI_MULTI_RECV).

# ENDPOINT ATTRIBUTES

The fi_ep_attr structure defines the set of attributes associated with
an endpoint.

{% highlight c %}
struct fi_ep_attr {
	uint64_t  protocol;
	size_t    max_msg_size;
	size_t    inject_size;
	size_t    total_buffered_recv;
	size_t    msg_prefix_size;
	size_t    max_order_raw_size;
	size_t    max_order_war_size;
	size_t    max_order_waw_size;
	uint64_t  mem_tag_format;
	uint64_t  msg_order;
	size_t    tx_ctx_cnt;
	size_t    rx_ctx_cnt;
};
{% endhighlight %}

## Protocol

Specifies the low-level end to end protocol employed by the provider.
A matching protocol must be used by communicating endpoints to ensure
interoperability.  The following protocol values are defined.
Provider specific protocols are also allowed.  Provider specific
protocols will be indicated by having the upper 3 bytes of the
protocol value set to the vendor OUI.

*FI_PROTO_UNSPEC*
: The protocol is not specified.  This is usually provided as input,
  with other attributes of the socket or the provider selecting the
  actual protocol.

*FI_PROTO_RDMA_CM_IB_RC*
: The protocol runs over Infiniband reliable-connected queue pairs,
  using the RDMA CM protocol for connection establishment.

*FI_PROTO_IWARP*
: The protocol runs over the Internet wide area RDMA protocol transport.

*FI_PROTO_IB_UD*
: The protocol runs over Infiniband unreliable datagram queue pairs.

*FI_PROTO_PSMX*
: The protocol is based on an Intel proprietary protocol known as PSM,
  performance scaled messaging.  PSMX is an extended version of the
  PSM protocol to support the libfabric interfaces.

## max_msg_size - Max Message Size

Defines the maximum size for an application data transfer as a single
operation.

## inject_size - Inject Size

Defines the default inject operation size (see the FI_INJECT flag)
that an endpoint will support.  This value applies per send operation.

## total_buffered_recv - Total Buffered Receive

Defines the total available space allocated by the provider to buffer
received messages (see the FI_BUFFERED_RECV flag).

## msg_prefix_size - Message Prefix Size

Specifies the size of any required message prefix buffer space.  This
field will be 0 unless the FI_MSG_PREFIX mode is enabled.  If
msg_prefix_size is > 0 the specified value will be a multiple of
8-bytes.

## Max RMA Ordered Size

The maximum ordered size specifies the delivery order of transport
data into target memory for RMA and atomic operations.  Data ordering
is separate, but dependent on message ordering (defined below).  Data
ordering is unspecified where message order is not defined.

Data ordering refers to the access of target memory by subsequent
operations.  When back to back RMA read or write operations access the
same registered memory location, data ordering indicates whether the
second operation reads or writes the target memory after the first
operation has completed.  Because RMA ordering applies between two
operations, and not within a single data transfer, ordering is defined
per byte-addressable memory location.  I.e.  ordering specifies
whether location X is accessed by the second operation after the first
operation.  Nothing is implied about the completion of the first
operation before the second operation is initiated.

In order to support large data transfers being broken into multiple packets
and sent using multiple paths through the fabric, data ordering may be
limited to transfers of a specific size or less.  Providers specify when
data ordering is maintained through the following values.  Note that even
if data ordering is not maintained, message ordering may be.

*max_order_raw_size*
: Read after write size.  If set, an RMA or atomic read operation
  issued after an RMA or atomic write operation, both of which are
  smaller than the size, will be ordered.  The RMA or atomic read
  operation will see the results of the previous RMA or atomic write.

*max_order_war_size*
: Write after read size.  If set, an RMA or atomic write operation
  issued after an RMA or atomic read operation, both of which are
  smaller than the size, will be ordered.  The RMA or atomic read
  operation will see the initial value of the target memory region
  before a subsequent RMA or atomic write updates the value.

*max_order_waw_size*
: Write after write size.  If set, an RMA or atomic write operation
  issued after an RMA or atomic write operation, both of which are
  smaller than the size, will be ordered.  The target memory region
  will reflect the results of the second RMA or atomic write.

An order size value of 0 indicates that ordering is not guaranteed.
A value of -1 guarantees ordering for any data size.

## mem_tag_format - Memory Tag Format

The memory tag format is a bit array used to convey the number of
tagged bits supported by a provider.  Additionally, it may be used to
divide the bit array into separate fields.  The mem_tag_format
optionally begins with a series of bits set to 0, to signify bits
which are ignored by the provider.  Following the initial prefix of
ignored bits, the array will consist of alternating groups of bits set
to all 1's or all 0's.  Each group of bits corresponds to a tagged
field.  The implication of defining a tagged field is that when a mask
is applied to the tagged bit array, all bits belonging to a single
field will either be set to 1 or 0, collectively.

For example, a mem_tag_format of 0x30FF indicates support for 14
tagged bits, separated into 3 fields.  The first field consists of
2-bits, the second field 4-bits, and the final field 8-bits.  Valid
masks for such a tagged field would be a bitwise OR'ing of zero or
more of the following values: 0x3000, 0x0F00, and 0x00FF.

By identifying fields within a tag, a provider may be able to optimize
their search routines.  An application which requests tag fields must
provide tag masks that either set all mask bits corresponding to a
field to all 0 or all 1.  When negotiating tag fields, an application
can request a specific number of fields of a given size.  A provider
must return a tag format that supports the requested number of fields,
with each field being at least the size requested, or fail the
request.  A provider may increase the size of the fields.

It is recommended that field sizes be ordered from smallest to
largest.  A generic, unstructured tag and mask can be achieved by
requesting a bit array consisting of alternating 1's and 0's.

## msg_order - Message Ordering

Message ordering refers to the order in which transport layer headers
(as viewed by the application) are processed.  Relaxed message order
enables data transfers to be sent and received out of order, which may
improve performance by utilizing multiple paths through the fabric
from the initiating endpoint to a target endpoint.  Message order
applies only between a single source and destination endpoint pair.
Ordering between different target endpoints is not defined.

Message order is determined using a set of ordering bits.  Each set
bit indicates that ordering is maintained between data transfers of
the specified type.  Message order is defined for [read | write |
send] operations submitted by an application after [read | write |
send] operations.

Message ordering only applies to the processing of transport headers.
Message ordering is necessary, but does not guarantee the order in
which data is sent or received by the transport layer.

*FI_ORDER_RAR*
: Read after read.  If set, RMA and atomic read operations are
  processed in the order submitted relative to other RMA and atomic
  read operations.  If not set, RMA and atomic reads may be processed
  out of order from their submission.

*FI_ORDER_RAW*
: Read after write.  If set, RMA and atomic read operations are
  processed in the order submitted relative to RMA and atomic write
  operations.  If not set, RMA and atomic reads may be processed ahead
  of RMA and atomic writes.

*FI_ORDER_RAS*
: Read after send.  If set, RMA and atomic read operations are
  processed in the order submitted relative to message send
  operations, including tagged sends.  If not set, RMA and atomic
  reads may be processed ahead of sends.

*FI_ORDER_WAR*
: Write after read.  If set, RMA and atomic write operations are
  processed in the order submitted relative to RMA and atomic read
  operations.  If not set, RMA and atomic writes may be processed
  ahead of RMA and atomic reads.

*FI_ORDER_WAW*
: Write after write.  If set, RMA and atomic write operations are
  processed in the order submitted relative to other RMA and atomic
  write operations.  If not set, RMA and atomic writes may be
  processed out of order from their submission.

*FI_ORDER_WAS*
: Write after send.  If set, RMA and atomic write operations are
  processed in the order submitted relative to message send
  operations, including tagged sends.  If not set, RMA and atomic
  writes may be processed ahead of sends.

*FI_ORDER_SAR*
: Send after read.  If set, message send operations, including tagged
  sends, are processed in order submitted relative to RMA and atomic
  read operations.  If not set, message sends may be processed ahead
  of RMA and atomic reads.

*FI_ORDER_SAW*
: Send after write.  If set, message send operations, including tagged
  sends, are processed in order submitted relative to RMA and atomic
  write operations.  If not set, message sends may be processed ahead
  of RMA and atomic writes.

*FI_ORDER_SAS*
: Send after send.  If set, message send operations, including tagged
  sends, are processed in the order submitted relative to other
  message send.  If not set, message sends may be processed out of
  order from their submission.

## tx_ctx_cnt - Transmit Context Count

Number of transmit contexts to associate with the endpoint.  If not
specified (0), 1 context will be assigned if the endpoint supports
outbound transfers.  Transmit contexts are independent command queues
that may be separately configured.  Each transmit context may be bound
to a separate CQ, and no ordering is defined between contexts.
Additionally, no synchronization is needed when accessing contexts in
parallel.

If the count is set to the value FI_SHARED_CONTEXT, the endpoint will
be configured to use a shared transmit context, if supported by the
provider.  Providers that do not support shared transmit contexts will
fail the request.

See the scalable endpoint and shared contexts sections for additional
details.

## rx_ctx_cnt - Receive Context Count

Number of receive contexts to associate with the endpoint.  If not
specified, 1 context will be assigned if the endpoint supports inbound
transfers.  Receive contexts are independent processing queues that
may be separately configured.  Each receive context may be bound to a
separate CQ, and no ordering is defined between contexts.
Additionally, no synchronization is needed when accessing contexts in
parallel.

If the count is set to the value FI_SHARED_CONTEXT, the endpoint will
be configured to use a shared receive context, if supported by the
provider.  Providers that do not support shared receive contexts will
fail the request.

See the scalable endpoint and shared contexts sections for additional
details.

# SCALABLE ENDPOINTS

A scalable endpoint is a communication portal that supports multiple
transmit and receive contexts.  Scalable endpoints are loosely modeled
after the networking concept of transmit/receive side scaling, also
known as multi-queue.  By default, an endpoint is associated with a
single transmit and receive context.  Support for scalable endpoints
is domain specific.  Scalable endpoints may improve the performance of
multi-threaded and parallel applications, by allowing threads to
access independent transmit and receive queues.  A scalable endpoint
has a single transport level address, which can reduce the memory
requirements needed to store remote addressing data, versus using
standard endpoints.

## fi_tx_context

Transmit contexts are independent command queues.  Ordering and
synchronization between contexts are not defined.  Conceptually a
transmit context behaves similar to a send-only endpoint.  A transmit
context may be configured with relaxed capabilities, and has its own
completion queue.  The number of transmit contexts associated with an
endpoint is specified during endpoint creation.

The fi_tx_context call is used to retrieve a specific context,
identified by an index.  Providers may dynamically allocate contexts
when fi_tx_context is called, or may statically create all contexts
when fi_endpoint is invoked.  By default, a transmit context inherits
the properties of its associated endpoint.  However, applications may
request context specific attributes through the attr parameter.
Support for per transmit context attributes is provider specific and
not guaranteed.  Providers will return the actual attributes assigned
to the context through the attr parameter, if provided.

{% highlight c %}
struct fi_tx_ctx_attr {
	uint64_t  caps;
	uint64_t  mode;
	uint64_t  op_flags;
	uint64_t  msg_order;
	size_t    inject_size;
	size_t    size;
	size_t    iov_limit;
};
{% endhighlight %}

*caps*
: The requested capabilities of the context.  The capabilities must be
  a subset of those requested of the associated endpoint.  See the
  CAPABILITIES section if fi_getinfo(3) for capability details.

*mode*
: The operational mode bits of the context.  The mode bits will be a
  subset of those associated with the endpoint.  See the MODE section
  of fi_getinfo(3) for details.

*op_flags*
: Flags that control the operation of operations submitted against the
  context.  Applicable flags are listed in the Operation Flags
  section.

*msg_order*
: The message ordering requirements of the context.  The message
  ordering must be the same or more relaxed than those specified of
  the associated endpoint.  See the fi_endpoint Message Ordering
  section.

*inject_size*
: The requested inject operation size (see the FI_INJECT flag) that
  the context will support.  This value must be equal to or less than
  the inject_size of the associated endpoint.  See the fi_endpoint
  Inject Size section.

*size*
: The size of the context, in bytes.  The size is usually used as an
  output value by applications wishing to track if sufficient space is
  available in the local queue to post a new operation.

*iov_limit*
: This is the maximum number of IO vectors (scatter-gather elements)
  that a single posted operation may reference.

## fi_rx_context

Receive contexts are independent command queues for receiving incoming
data.  Ordering and synchronization between contexts are not
guaranteed.  Conceptually a receive context behaves similar to a
receive-only endpoint.  A receive context may be configured with
relaxed endpoint capabilities, and has its own completion queue.  The
number of receive contexts associated with an endpoint is specified
during endpoint creation.

Receive contexts are often associated with steering flows, that
specify which incoming packets targeting a scalable endpoint to
process.  However, receive contexts may be targeted directly by the
initiator, if supported by the underlying protocol.  Such contexts are
referred to as 'named'.  Support for named contexts must be indicated
by setting the caps FI_NAMED_RX_CTX capability when the corresponding
endpoint is created.  Support for named receive contexts is
coordinated with address vectors.  See fi_av(3) and fi_rx_addr(3).

The fi_rx_context call is used to retrieve a specific context,
identified by an index.  Providers may dynamically allocate contexts
when fi_rx_context is called, or may statically create all contexts
when fi_endpoint is invoked.  By default, a receive context inherits
the properties of its associated endpoint.  However, applications may
request context specific attributes through the attr parameter.
Support for per receive context attributes is provider specific and
not guaranteed.  Providers will return the actual attributes assigned
to the context through the attr parameter, if provided.

{% highlight c %}
struct fi_rx_ctx_attr {
	uint64_t  caps;
	uint64_t  mode;
	uint64_t  op_flags;
	uint64_t  msg_order;
	size_t    total_buffered_recv;
	size_t    size;
	size_t    iov_limit;
};
{% endhighlight %}

*caps*
: The requested capabilities of the context.  The capabilities must be
  a subset of those requested of the associated endpoint.  See the
  CAPABILITIES section if fi_getinfo(3) for capability details.

*mode*
: The operational mode bits of the context.  The mode bits will be a
  subset of those associated with the endpoint.  See the MODE section
  of fi_getinfo(3) for details.

*op_flags*
: Flags that control the operation of operations submitted against the
  context.  Applicable flags are listed in the Operation Flags
  section.

*msg_order*
: The message ordering requirements of the context.  The message
  ordering must be the same or more relaxed than those specified of
  the associated endpoint.  See the fi_endpoint Message Ordering
  section.

*total_buffered_recv*
: Defines the total available space allocated by the provider to
  buffer received messages on the context.  This value must be less
  than or equal to that specified for the associated endpoint.  See
  the fi_endpoint Total Buffered Receive section.

*size*
: The size of the context, in bytes.  The size is usually used as an
  output value by applications wishing to track if sufficient space is
  available in the local queue to post a new operation.

*iov_limit*
: This is the maximum number of IO vectors (scatter-gather elements)
  that a single posted operating may reference.

# SHARED CONTEXTS

Shared contexts are transmit and receive contexts explicitly shared
among one or more endpoints.  A sharable context allows an application
to use a single dedicated provider resource among multiple transport
addressable endpoints.  This can greatly reduce the resources needed
to manage communication over multiple endpoints by multiplexing
transmit and/or receive processing, with the potential cost of
serializing access across multiple endpoints.  Support for sharable
contexts is domain specific.

Conceptually, sharable contexts are command queues that may be
accessed by many endpoints.  The use of a shared transmit context is
mostly opaque to an application.  Applications must allocate and bind
shared transmit contexts to endpoints, but otherwise transmit
operations are posted directly to the endpoint.  An endpoint may only
be associated with a single shared transmit context.

Unlike shared transmit contexts, applications interact directly with
shared receive contexts.  Users post receive buffers directly to a
shared receive context, with the buffers usable by any endpoint bound
to the shared receive context.  An endpoint may only be associated
with a single receive context.

Endpoints associated with a shared transmit context may use dedicated
receive contexts, and vice-versa.  Or an endpoint may use shared
transmit and receive contexts.  And there is no requirement that the
same group of endpoints sharing a context of one type also share the
context of an alternate type.  Furthermore, an endpoint may use a
shared context of one type, but a scalable set of contexts of the
alternate type.

## fi_stx_context

This call is used to open a sharable transmit context.  See
fi_tx_context call under the SCALABLE ENDPOINTS section for details on
the transit context attributes.  The exception is that endpoints
attached to a shared transmit context must use a subset of the
transmit context attributes.  This is opposite of the requirement for
scalable endpoints.

## fi_srx_context

This allocates a sharable receive context.  See fi_rx_context call
under SCALABLE ENDPOINTS section for details on the receive context
attributes.  The exception is that endpoints attached to a shared
receive context must use a subset of the receive context attributes.
This is opposite of the requirement for scalable endpoints.

# OPERATION FLAGS

Operation flags are obtained by OR-ing the following flags together.
Operation flags define the default flags applied to an endpoint's data
transfer operations, where a flags parameter is not available.  Data
transfer operations that take flags as input override the op_flags
value of an endpoint.

*FI_INJECT*
: Indicates that all outbound data buffer should be returned to the
  user's control immediately after a data transfer call returns, even
  if the operation is handled asynchronously.  This may require that
  the provider copy the data into a local buffer and transfer out of
  that buffer.  A provider may limit the total amount of send data
  that may be buffered and/or the size of a single send.  Applications
  may discover and modify these limits using the endpoint's getopt and
  setopt interfaces.

*FI_MULTI_RECV*
: Applies to posted receive operations.  This flag allows the user to
  post a single buffer that will receive multiple incoming messages.
  Received messages will be packed into the receive buffer until the
  buffer has been consumed.  Use of this flag may cause a single
  posted receive operation to generate multiple completions as
  messages are placed into the buffer.  The placement of received data
  into the buffer may be subjected to provider specific alignment
  restrictions.  The buffer will be freed from the endpoint when a
  message is received that cannot fit into the remaining free buffer
  space.

*FI_BUFFERED_RECV*
: If set, the communication interface implementation should attempt to
  queue inbound data that arrives before a receive buffer has been
  posted.  In the absence of this flag, any messages that arrive
  before a receive is posted are lost.

*FI_COMPLETION*
: Indicates that a completion entry should be generated for data
  transfer operations.

*FI_REMOTE_SIGNAL*
: Indicates that a completion entry at the target process should be
  generated for the given operation.  The remote endpoint must be
  configured with FI_REMOTE_SIGNAL, or this flag will be ignored by
  the target.  The local endpoint must be configured with the
  FI_REMOTE_SIGNAL capability in order to specify this flag.

*FI_REMOTE_COMPLETE*
: Indicates that local completions should not be generated until the
  operation has completed on the remote side.  When set, if the target
  endpoint experiences an error receiving the transferred data, that
  error will be reported back to the initiator of the request.  This
  includes errors which may not normally be reported to the initiator.
  For example, if the receive data is truncated at the target because
  the provided receive buffer is too small, the initiator will be
  notified of the truncation.

*FI_READ*
: Indicates that the user wants to initiate reads against remote
  memory regions.  Remote reads include some RMA and atomic
  operations.

*FI_WRITE*
: Indicates that the user wants to initiate writes against remote
  memory regions.  Remote writes include some RMA and most atomic
  operations.

*FI_SEND*
: Indicates that the endpoint will be used to send message data
  transfers.  Message transfers include base message operations as
  well as tagged message functionality.

*FI_RECV*
: Indicates that the endpoint will be used to receive message data
  transfers.  Message transfers include base message operations as
  well as tagged message functionality.

*FI_REMOTE_READ*
: Indicates that the endpoint should allow remote endpoints to read
  memory regions exposed by this endpoint.  Remote read operations
  include some RMA and atomic operations.

*FI_REMOTE_WRITE*
: Indicates that the endpoint should allow remote endpoints to write
  to memory regions exposed by this endpoint.  Remote write operations
  include some RMA operations and most atomic operations.

# NOTES

Users should call fi_close to release all resources allocated to the
fabric endpoint.

Endpoints allocated with the FI_CONTEXT mode set must typically
provide struct fi_context as their per operation context parameter.
(See fi_getinfo.3 for details.)  However, when FI_COMPLETION is
enabled to suppress completion entries, and an operation is initiated
without FI_COMPLETION flag set, then the context parameter is ignored.
An application does not need to pass in a valid struct fi_context into
such data transfers.

Operations that complete in error that are not associated with valid
operational context will use the endpoint context in any error
reporting structures.

# RETURN VALUES

Returns 0 on success.  On error, a negative value corresponding to
fabric errno is returned.

Fabric errno values are defined in `rdma/fi_errno.h`.

# ERRORS

*-FI_EDOMAIN*
: A resource domain was not bound to the endpoint or an attempt was
  made to bind multiple domains.

*-FI_ENOCQ*
: The endpoint has not been configured with necessary event queue.

*-FI_EOPBADSTATE*
: The endpoint's state does not permit the requested operation.

# SEE ALSO

[`fi_getinfo`(3)](fi_getinfo.3.html),
[`fi_domain`(3)](fi_domain.3.html),
[`fi_msg`(3)](fi_msg.3.html),
[`fi_tagged`(3)](fi_tagged.3.html),
[`fi_rma`(3)](fi_rma.3.html)
