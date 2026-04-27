# Post Mortem Covering 11 April to 23 April

During April 2026, the service experienced multiple major outages affecting several TorBox functions. Some of these outages caused TorBox to become entirely unavailable to users for extended periods of time and exposed critical gaps in our monitoring and response apparatus.

We are sorry for letting you down and have compiled this post to outline exactly what happened and why. We will also discuss the steps taken to prevent this from happening again.

To compensate our valued users for this downtime, we will be crediting all paid plans with an additional twelve days of time automatically. Keep an eye on our [𝕏.com](https://x.com/torbox) for the details.

<sub>*This post is segmented into sections to improve readability with extra details provided for high severity incidents.*</sub>

## Establishing the timeline

### [Elevated error rates returned by Usenet via API](https://status.torbox.app/incident/869293) (Minor)

On 11 April 2026, we began seeing higher than normal error rates returned to users by our API when Usenet downloads were added, due to a misconfiguration of our Usenet clients. This issue was promptly resolved.

### Elevated rates of 404s returned to users by CDNs (Major)

On 13 April 2026, a subset of CDN servers were configured to use an experimental performance enhancing change. We intended for this to provide us with crucial performance data to finetune the changes before being pushed to our global CDN fleet. This backfired when a lack of logging caused errors to go unnoticed for several hours before the change was reverted.

The next day, on 14 April 2026, we again tested an updated configuration on a smaller sample of CDN servers with constant human supervision. The test revealed poorer than anticipated performance and the change was rolled back. This unfortunately required links to be invalidated and regenerated globally and we did not take into account the link caching done by many end-user devices. Many users continued to report broken links as their devices were still using the previously invalidated URL tokens necessitating device restarts.

### [Add-on outage](https://status.torbox.app/incident/872542) (Upstream dependency failure) (Minor)

On 15 April 2026, an upstream dependency powering the search function used by our Stremio add-on began returning unusable results. After some time investigating it became clear that this was an intentional change made to prevent automated scrapers from using those results which incidentally also prevented TorBox from using this data to power our add-on. Our team responded by switching the upstream source entirely which required rewriting a non-trivial amount of backend code.

### [API outage (part one)](https://status.torbox.app/incident/871794) (Major)

Also on 15 April 2026, our database became CPU locked causing a total service outage. Our team began investigating within minutes to identify the cause. A lack of logging and visibility delayed the investigation for several hours as multiple resolution paths were considered, tested, and ruled out. We later discovered the true cause: at 2:10am SAST the nightly database vacuum cronjob had caused a cascade of timeouts that compounded to eventually lock up the entire database. Once the cause was validated a fix was promptly made to restore operations.

### Cloudflare CDN outage (Minor)

On 16 April 2026, to enhance download speeds for users in difficult to serve areas, we began routing fallback link generation to our ERTH (Cloudflare) CDN as default, rather than serving files from the next closest storage region. Initially all internal tests passed and human supervision was undertaken without problems, however during Oceaniac peak traffic hours, users began reporting 429 rate limit errors being returned to them when accessing Cloudflare served files due to a configuration oversight. We immediately reverted to our previously validated configuration.

An incidental DNS outage was also noted for approximately ten minutes but was deemed unrelated.

### [WebDAV outage](https://status.torbox.app/incident/874700) (Testing Oversight) (Minor)

On April 18 2026, the response code returned by our WebDAV service was changed to match RFC 2616 standards, which was validated internally by our development team using a sample of browsers. Immediately after deployment, users began reporting errors as their browsers and players would not allow them to access their WebDAV content. After testing more players and browsers, the development team resolved the issue by not strictly keeping to standard, as not all clients accepted said standards.

### Degraded createTorrent endpoint (Testing Oversight) (Minor)

On 23 April 2026, we shipped a fix to address a race condition that caused downloads by different users for the same file to be added to our database twice. This caused an unintentional delay when files were added via our API that did not present in testing. This issue was subsequently reported to us by a community developer and the fix was promptly rolled-back. (Thank you [MunifTanjim](https://github.com/MunifTanjim))

### Denial of Service (Minor)

On 23 April 2026, a denial of service attack targeted at our web infrastructure overlapped with an API degradation event. This caused our response resources to dilute between possible areas of cause. The attack itself would normally not have been a problem but combined with a more relaxed firewall policy we introduced recently, and the compounded prior outages, it caught us off guard.

### [API outage (part two)](https://status.torbox.app/incident/871794) (Major)

On 23 April 2026, 1AM SAST, our alerts fired and the team began investigating the first complete service outage of the day. Coincidentally, this happened around the same time (midnight SAST) as the previous API outage but was not related to the database this time. We had just performed a minor API update that routinely clears a large volume of keys from our internal Redis cluster. This caused a cache stampede between our API and storage clusters resulting in response time spikes that snowballed out of control. We hurriedly pushed an untested fix in a misguided attempt to restore services. This worked and provided the team with a false sense of security as the issue reoccurred that evening (8PM SAST) when a short network hiccup upset an already fragile system.

This second outage lasted several hours as the team discarded any prior assumptions and worked throughout the night to ensure stability which was partially successful as no further outage was observed through the weekend. We’re still in the process of hardening against these types of issues which will require major architectural revisions over the coming weeks.

## What is being done about this?

In the coming weeks we will be setting aside maintenance windows for stress testing our systems for robustness. These will be conducted during off-peak hours to ensure any downtime experienced is minimised and sufficient notice will be provided via our status page.

We will also begin seeding new updates to a small number of servers at random to smoke test for stability and bugs before committing them at scale. This will be in addition to the internal testing we already do and will be scoped to a very small percentage of requests to minimise the blast radius of issues when they happen.

Lastly we have already improved, and will continue to improve, our internal alerting systems to ensure the relevant teams can be reached at all times of the day by any other member of the team.

## Why did all this happen in such a short time?

The blunt truth is that our pursuit of fast development velocity introduced multiple weak points that, when combined with rapid user growth, created the ideal conditions for chaos. We cannot blame vibe coding with AI, irresponsible features and sloppy bug fixes, nor can we deflect the blame onto bad luck. Our team is solely responsible and we dropped the ball, it bounced, and hit us in the face. We’re sorry and promise to do better.

Thank you for your trust and support,
TorBox Team
