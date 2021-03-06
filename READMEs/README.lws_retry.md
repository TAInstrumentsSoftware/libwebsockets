# `lws_retry_bo_t` client connection management

This struct sets the policy for delays between retries, and for
how long a connection may be 'idle' before it first tries to
ping / pong on it to confirm it's up, or drops the connection
if still idle.

## Retry rate limiting

You can define a table of ms-resolution delays indexed by which
connection attempt number is ongoing, this is pointed to by
`.retry_ms_table` with `.retry_ms_table_count` containing the
count of table entries.

`.conceal_count` is the number of retries that should be allowed
before informing the parent that the connection has failed.  If it's
greater than the number of entries in the table, the last entry is
reused for the additional attempts.

`.jitter_percent` controls how much additional random delay is
added to the actual interval to be used... this stops a lot of
devices all synchronizing when they try to connect after a single
trigger event and DDoS-ing the server.

The struct and apis are provided for user implementations, lws does
not offer reconnection itself.

## Connection validity management

Lws has a sophisticated idea of connection validity and the need to
reconfirm that a connection is still operable if proof of validity
has not been seen for some time.  It concerns itself only with network
connections rather than streams, for example, it only checks h2
network connections rather than the individual streams inside (which
is consistent with h2 PING frames only working at the network stream
level itself).

Connections may fail in a variety of ways, these include that no traffic
at all is passing, or, eg, incoming traffic may be received but no
outbound traffic is making it to the network, and vice versa.  In the
case that tx is not failing at any point but just isn't getting sent,
endpoints can potentially kid themselves that since "they are sending"
and they are seeing RX, the combination means the connection is valid.
This can potentially continue for a long time if the peer is not
performing keepalives.

"Connection validity" is proven when one side sends something and later
receives a response that can only have been generated by the peer
receiving what was just sent.  This can happen for some kinds of user
transactions on any stream using the connection, or by sending PING /
PONG protocol packets where the PONG is only returned for a received PING.

To ensure that the generated traffic is only sent when necessary, user
code can report for any stream that it has observed a transaction amounting
to a proof of connection validity using an api.  This resets the timer for
the associated network connection before the validity is considered
expired.

`.secs_since_valid_ping` in the retry struct sets the number of seconds since
the last validity after which lws will issue a protocol-specific PING of some
kind on the connection.  `.secs_since_valid_hangup` specifies how long lws
will allow the connection to go without a confirmation of validity before
simply hanging up on it.

## Defaults

The context defaults to having a 5m valid ping interval and 5m10s hangup interval,
ie, it'll send a ping at 5m idle if the protocol supports it, and if no response
validating the connection arrives in another 10s, hang up the connection.

User code can set this in the context creation info and can individually set the
retry policy per vhost for server connections.  Client connections can set it
per connection in the client creation info `.retry_and_idle_policy`.

## Checking for h2 and ws

Check using paired minimal examples with the -v flag on one or both sides to get a
small validity check period set of 3s / 10s

Also give, eg, -d1039 to see info level debug logging

### h2

```
$ lws-minimal-http-server-h2-long-poll -v

$ lws-minimal-http-client -l -v
```

### ws

```
$ lws-minimal-ws-server-h2 -s -v

$ lws-minimal-ws-client-ping -n --server 127.0.0.1 --port 7681 -v
```


