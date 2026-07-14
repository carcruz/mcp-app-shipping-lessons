# Diagnosing generic host errors like "Unable to reach `<server>`"

## The trap

MCP hosts surface connector-level failures with a single generic message — "Unable to reach `<server-name>`," or similar — regardless of *why* the call actually failed. That message is consistent with several completely different root causes:

- The server process is genuinely down or unreachable (network, DNS, TLS).
- The server is up, but autoscaling/cold-start churn wiped session state mid-conversation.
- The server is up, the transport succeeded, but the specific JSON-RPC call the host made returned an *error response* — not a transport failure at all.

Only the first two are actually connectivity problems. The third is a normal application-level error dressed up as a connectivity error by the host's generic messaging, and chasing infrastructure fixes (restarting the service, adjusting autoscaling, re-adding the connector) will not fix it, because nothing is actually unreachable.

## The fix

Before touching infrastructure, check the client-side transport log for the actual JSON-RPC exchange. Hosts that bridge to a remote MCP server (e.g. Claude Desktop via `mcp-remote`) typically log every message per configured server:

```
~/Library/Logs/Claude/mcp-server-<connector-name>.log
```

Look at the sequence of calls around the failure, not just the last line. A pattern like:

```
Message from client: method="tools/call" id=8 ...
Message from server: id=8 result(1 blocks) ...
Message from client: method="resources/read" id=9 ...
Message from server: id=9 error(code=-32603) ...
```

tells you the tool call succeeded and a *specific* follow-up call (here, reading the UI resource the tool referenced) is what's actually failing, with a real JSON-RPC error code. That rules out "the server is unreachable" outright — it was reached, twice, and answered both times. The bug is in whatever handler produces that specific response, not in networking, sessions, or autoscaling.

Cross-check against the server's own request logs (e.g. `gcloud logging read` for a Cloud Run service). A request that HTTP-succeeded (`200`) with a suspiciously small response body compared to sibling requests is the server-side fingerprint of the same thing: the transport-level status doesn't reflect an application-level error wrapped inside it. Don't let a row of `200`s in infrastructure logs rule out an application bug — check response sizes, or better, grep the server's own stdout/stderr for the error path, not just the request access log.

## Aftermath

Once you've confirmed via the client log which specific call is failing (and that others are succeeding), you've narrowed "unreachable" down to one handler in the server. From there, treat it as an ordinary application bug: find the handler for that method/resource, and check what it does differently from the calls that are succeeding — a different code path, a file it reads that others don't, an external dependency the succeeding calls don't touch. That's usually a much faster path to the actual root cause than anything infrastructure-side.
