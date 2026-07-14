# In-memory session state doesn't survive scale-to-zero

## The trap

Deploying an MCP server to a serverless/autoscaled platform (Cloud Run, and similarly-shaped platforms) with the default autoscaling settings means the platform will scale the service down to **zero running instances** after a period of no traffic, and cold-start a fresh one on the next request. That's the expected, cost-saving behavior for a stateless HTTP API — and actively harmful for an MCP server, because MCP sessions (and the long-lived Streamable HTTP `GET` stream a session keeps open for server-initiated messages) are normally held **in process memory**, not in any external store.

Every time the instance scales to zero:
- Any open long-lived stream gets killed out from under the client.
- All in-memory session state for every active conversation is gone.
- The next request from a client that thinks it still has a valid session gets nothing sensible back, because the server that issued that session no longer exists.

From the host's side, this looks exactly like a generic connectivity failure (see `host-error-diagnosis.md`) — there's no clean error, just a session that stopped working. It's also intermittent and hard to reproduce on demand: it only shows up after however long the platform's idle timeout is, which makes it easy to mistake for a one-off flake rather than a structural scaling problem.

Two related but distinct constraints show up in this class of server:
- **Max instances must often be capped at 1** if session state isn't shared across instances (no external session store) — otherwise requests from the same client can land on a different instance than the one holding their session.
- **Min instances must be at least 1** for the same underlying reason — otherwise the *one* instance you're relying on periodically ceases to exist.

Capping max-instances without also flooring min-instances only solves the "wrong instance" half of the problem; the "instance disappeared entirely" half is still live.

## The fix

Set an explicit floor on running instances so the process (and its in-memory sessions) never gets torn down between requests:

```bash
gcloud run services update <service> \
  --region <region> \
  --min-instances 1
```

This is a service-level scaling setting, not tied to a particular image — it can be applied independently of any rebuild/redeploy, and takes effect immediately on the existing running revision.

The tradeoff is real and worth being upfront about: the instance now runs continuously instead of scaling to zero when idle, which means small but ongoing compute cost instead of near-zero cost during idle periods. For a server used interactively by real users (as opposed to a personal/local-only tool), that tradeoff is usually worth it — an intermittently-broken shared tool erodes trust faster than a few dollars a month in idle compute costs.

## Aftermath

If the platform enforces a hard per-request timeout (Cloud Run's default is 300s, matching how long a Streamable HTTP `GET` stream can stay open before being forcibly cut), a `min-instances 1` floor doesn't eliminate stream churn entirely — the client transport should already handle a stream reconnect gracefully as part of normal operation. What the floor actually fixes is the *session* surviving that churn, since the process holding it never goes away. If reconnects still fail even with a floor in place, that's a separate bug in the client's reconnect handling, not a scaling config problem.
