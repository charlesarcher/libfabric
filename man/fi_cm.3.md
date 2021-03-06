---
layout: page
title: fi_cm(3)
tagline: Libfabric Programmer's Manual
---
{% include JB/setup %}

# NAME

fi_cm - Connection management operations

fi_connect / fi_listen / fi_accept / fi_reject / fi_shutdown
: Manage endpoint connection state.

fi_getname / fi_getpeer
: Return local or peer endpoint address

fi_join / fi_leave
: Have an endpoint join or leave a multicast group.

# SYNOPSIS

{% highlight c %}
#include <rdma/fi_cm.h>

int fi_connect(struct fid_ep *ep, const void *addr,
    const void *param, size_t paramlen);

int fi_listen(struct fid_pep *pep);

int fi_accept(struct fid_ep *ep, fi_connreq_t connreq,
    const void *param, size_t paramlen);

int fi_reject(struct fid_pep *pep, fi_connreq_t connreq,
    const void *param, size_t paramlen);

int fi_shutdown(struct fid_ep *ep, uint64_t flags);

int fi_getname(fid_t fid, void *addr, size_t *addrlen);

int fi_getpeer(struct fid_ep *ep, void *addr, size_t *addrlen);

int fi_join(struct fid_ep *ep, void *addr, fi_addr_t *fi_addr,
    uint64_t flags, void *context);

int fi_leave(struct fid_ep *ep, void *addr, fi_addr_t fi_addr,
    uint64_t flags);
{% endhighlight %}

# ARGUMENTS

*ep / pep*
: Fabric endpoint on which to change connection state.

*addr*
: Buffer to store queried address (get), or address to
  connect/join/leave.  The address must be in the same format as that
  specified using fi_info: addr_format when the endpoint was created.

*addrlen*
: On input, specifies size of addr buffer.  On output, stores number
  of bytes written to addr buffer.

*param*
: User-specified data exchanged as part of the connection exchange.

*paramlen*
: Size of param buffer.

*info*
: Fabric information associated with a connection request.

*fi_addr*
: Fabric address associated with a multicast address.

*flags*
: Additional flags for controlling connection operation.

*context*
: User context associated with the request.

# DESCRIPTION

Connection management functions are used to connect an endpoint to a
remote address (in the case of a connectionless endpoint) or a peer
endpoint (for connection-oriented endpoints).

## fi_listen

The fi_listen call indicates that the specified endpoint should be
transitioned into a passive connection state, allowing it to accept
incoming connection requests.  Connection requests against a listening
endpoint are reported asynchronously to the user through a bound CM
event queue using the FI_CONNREQ event type.  The number of outstanding
connection requests that can be queued at an endpoint is limited by the
listening endpoint's backlog parameter.  The backlog is initialized
based on administrative configuration values, but may be adjusted
through the fi_control call.

## fi_connect

For a connection-oriented endpoint, fi_connect initiates a connection
request to the destination address.  For a connectionless endpoint,
fi_connect specifies the destination address that future data transfer
operations will target.  This avoids the need for the user to specify the
address as part of the data transfer.

## fi_accept / fi_reject

The fi_accept and fi_reject calls are used on the passive (listening)
side of a connection to accept or reject a connection request,
respectively.  To accept a connection, the listening application first
waits for a connection request event.  After receiving such an event, it
allocates a new endpoint to accept the connection.  fi_accept is invoked
with the newly allocated endpoint passed in as the fid parameter.  If
the listening application wishes to reject a connection request, it calls
fi_reject with the listening endpoint passed in as the fid.
fi_reject takes a reference to the connection request as an input parameter.

A successfully accepted connection request will result in the active
(connecting) endpoint seeing an FI_CONNECTED event on its associated
event queue.  A rejected or failed connection request will generate an
error event.  The error entry will provide additional details describing
the reason for the failed attempt.

An FI_CONNECTED event will also be generated on the passive side for the
accepting endpoint once the connection has been properly established.
Outbound data transfers cannot be initiated on a connection-oriented
endpoint until an FI_CONNECTED event has been generated.  However, receive
buffers may be associated with an endpoint anytime.

For connection-oriented endpoints, the param buffer will be sent as
part of the connection request or response, subject to the constraints of
the underlying connection protocol.  Applications may use fi_control
to determine the size of application data that may be exchanged as
part of a connection request or response.  The fi_connect, fi_accept, and
fi_reject calls will silently truncate any application data which cannot
fit into underlying protocol messages.

## fi_shutdown

The fi_shutdown call is used to gracefully disconnect an endpoint from
its peer.  If shutdown flags are 0, the endpoint is fully disconnected,
and no additional data transfers will be possible.  Flags may also be
used to indicate that only outbound (FI_WRITE) or inbound (FI_READ) data
transfers should be disconnected.  Regardless of the shutdown option
selected, any queued completions associated with asynchronous operations
may still be retrieved from the corresponding event queues.

An FI_SHUTDOWN event will be generated for an endpoint when the remote
peer issues a disconnect using fi_shutdown or abruptly closes the endpoint.

## fi_getname / fi_getpeer

The fi_getname and fi_getpeer calls may be used to retrieve the local or
peer endpoint address, respectively.  On input, the addrlen parameter should
indicate the size of the addr buffer.  If the actual address is larger than
what can fit into the buffer, it will be truncated.  On output, addrlen
is set to the size of the buffer needed to store the address, which may
be larger than the input value.

## fi_join / fi_leave

fi_join and fi_leave are use to associate or dissociate an endpoint
with a multicast group.  Join operations complete asynchronously, with
the completion reported through the event queue associated with the
endpoint or domain, if an event queue has not been bound to the
endpoint.

A fabric address will be provided as part of the join request.  The
address will be written to the memory location referenced by the
fi_addr parameter.  This address must be used when issuing data
transfer operations to the multicast group.  Because join operations
are asynchronous, the memory location referenced by the fi_addr
parameter must remain valid until an event associated with the join is
reported, or a corresponding call to leave the multicast group
returns.  Fi_addr is not guaranteed to be set upon return from
fi_join, and it is strongly recommended that fi_addr not be declared
on the stack, as data corruption may result.

The fi_leave call will result in an endpoint leaving a multicast
group.  The fi_leave call may be called even if the join operation has
not completed, in which case the join will be canceled if it has not
yet completed.

# FLAGS

The fi_join call allows the user to specify flags requesting the type of
join operation being requested.  Flags for fi_leave must be 0.

*FI_SEND*
: Setting FI_SEND, but not FI_RECV, indicates that the endpoint should
  join the multicast group as a send-only member.  If FI_RECV is also
  set or neither FI_SEND or FI_RECV are set, then the endpoint will
  join the group with send and receive capabilities.

*FI_RECV*
: Setting FI_RECV, but not FI_SEND, indicates that the endpoint should
  join the multicast group as a receive-only member.  If FI_SEND is
  also set or neither FI_SEND or FI_RECV are set, then the endpoint
  will join the group with send and receive capabilities.

# RETURN VALUE

Returns 0 on success. On error, a negative value corresponding to fabric
errno is returned. Fabric errno values are defined in
`rdma/fi_errno.h`.

# ERRORS


# NOTES


# SEE ALSO

[`fi_getinfo`(3)](fi_getinfo.3.html),
[`fi_endpoint`(3)](fi_endpoint.3.html),
[`fi_domain`(3)](fi_domain.3.html),
[`fi_eq`(3)](fi_eq.3.html)
