# TorBox Outage Post Mortem - August 2026

During August 2026, TorBox experienced multiple service outages involving both our platform and its network dependencies. Some issues prevented users from accessing TorBox functions while others exposed blind spots in how we detect and recover from problems.

We are sorry for the troublesome month. This post explains what happened, what we did, and the work we still have to do.

<sub>*Below times are UTC unless otherwise stated. Monitoring windows are not always the full duration of user impact.*</sub>

<img width="1169" height="685" alt="10-api-error-rate-overview" src="https://github.com/user-attachments/assets/ffb09dc4-eb15-4686-8aae-4312215f2b0f" />

<sub>The above chart shows the sudden onset of HTTP >=500 codes returned to users.</sub>

## Establishing the timeline

### Early August

| Incident | Brief |
|---|---|
| [3 August — general outage](https://status.torbox.app/incident/994569) | Severe errors began around 20:50pm. Median response time jumped from about 100ms to >200 seconds. Improvements were felt at 22:55pm, but long tail slow requests continued. This was resolved at 21:38pm |
| [8 August — general outage](https://status.torbox.app/incident/1005499) | Latency started rising from about 11:30am. Errors improved around 12:10pm. |
| 13 August — API monitor timeout | A five minute incident started at 07:01am after a probe received no response headers. This was a false positive. |
| [18 August — API timeouts](https://status.torbox.app/incident/1019820) | Degradation reported around 18:45pm and latency returned to baseline around 19:40pm. This was caused by a flood of requests sent to our database and we tightened edge rate limits. We believe this started, or at least directly contributed to, the following outage waves.|


### [20–24 August: Absolute chaos](https://status.torbox.app/incident/1022223)

This was the start of the outage/recovery waves that involved our API, database, relays, and caused other unrelated and unaffected services to fire alerts that made identifying the primary cause difficult. 

On **20 August**, we discovered host conntrack table exhaustion (kernel active connection records) on some API servers. Our Redis cluster also experienced a stampede of clients reconnecting after outage waves further complicating things. We made adjustments to account for the spikes in traffic and tightened our edge firewall to block as much of the bad traffic before it reached our servers.

On **21 August**, we began seeing massive cache check requests hitting our API causing critical requests to fail, we quickly made changes to better handle both how these requests are handled and prevent those requests from starving our database connection pool. We rolled out the change to all API servers and this improved the sampled error rates immediately.

We also scaled our API cluster to account for this increase in requests but the additional worker count led to database connection limits being reached. Due to poor telemetry from our database provider this remained invisible to us for some time (more on this issue later).

A **23 August rollback** then restored a much larger worker configuration across all API servers, which was more than double what we had previously. This was a multiplicative effect as shown below. 

Each worker had its own connection pool:

```text
12 hosts × 32 workers × 64 connections = 24 576 possible connections
Pooler client limit                   =   20 000
```
<sub>*Values for illustrative purposes only. This shows the theoretical application capacity and not the true number of connections.*</sub>

This caused a slow increase in connection errors as the number of connections made by the API cluster exceeded the available connections offered by our database pooler. Our Redis cluster started thrashing and alerts started firing. We both scaled out our Redis cluster and put in place mechanisms to warn us of similar failures before they happen.

Supabase also reported a PostgREST failure during these waves which contributed to the chaos. 
[Provider update](https://status.supabase.com/incidents/6q5902p2xd9f).

### 25–26 August: Databases, Cloudflare, and you

In response to various attacks on our network, we tightened mitigation filters to be more strict which caused traffic between us, our database provider, and Cloudflare to fail. This failure was not immediately obvious as it only broke TLS connections during peak traffic periods as our baseline mitigation sensitivity was high enough to not cause problems except during the 1-2 hour highest traffic periods. This was no doubt the worst time to have things break and due to the nature of the failure, the cause was not immediately obvious. We worked with our upstream providers to identify and correct the mitigation rule in question.

### 27 August: Attacks cascade

While the attacks continued, Supabase started falling over which caused stalled requests to snowball. We experienced a number of short but high impact outages. At 17:44pm alerts started firing and the team was deployed. This was resolved shortly after.

[Supabase latency incident](https://status.supabase.com/incidents/bv4ntm4x0btf), [linked rollout account](https://status.supabase.com/incidents/6q5902p2xd9f).

### [27–28 August: planned network maintenance](https://status.torbox.app/maintenance/1031021)

Later that evening planned maintenance was carried out and is unrelated to the events of the week. It is mentioned here only for completeness sake. 

### 28 August: Script kiddies and botnets

<img width="1391" height="714" alt="12-http-ddos-mitigation" src="https://github.com/user-attachments/assets/47e23f14-f75f-4c71-b8eb-7ea4acaf3109" />

<sub>*This is a rolling 21-day view extending into September. It is not an August-only total or a count of requests reaching our origins. *</sub>

We have chosen to not disclose any additional details but give thanks to our upstream partners for assisting us in defending against these attacks. 

### 29 August: WebDAV errors

To better absorb attacks and traffic stampedes our cloud Redis cluster was brought on-net, this cluster served requests for WebDAV and T3 - we started observing errors stemming from bad ETag generation, which is used in identifying file metadata. This bad data was being passed to the below code snippet which was promptly fixed:

```diff
- xxhash.xxh64(value, seed=0).hexdigest()
+ xxhash.xxh64(value.encode("utf-8"), seed=0).hexdigest()
```
The essential correction was explicit UTF-8 encoding, shown here with the input shortened to `value`:

Telemetry showed this affected 316,378 requests between 11:03 and 11:42. The fix was made live and errors quickly dropped.

A separate WebDAV probe timeout occurred at **14:06–14:09**. This was corrected within minutes.

## What is being done about this?

Without giving away too many intricate details that may invite future attackers, we can say that we learnt a lot from these outages and spent significant time improving both monitoring and response resources to detect issues sooner and fix them quicker. We will continue to invest in our network and people to withstand attacks and become more robust in the future. 

## Why did all this happen in such a short time?

We were caught with our pants down and did not spend enough time testing fixes after they were made live. Additionally we simply were not prepared to respond to the layered attacks on our infrastructure which compounded the problems and prolonged them. We promise to do better.

Thank you for your patience and unwavering support,

The TorBox Team
