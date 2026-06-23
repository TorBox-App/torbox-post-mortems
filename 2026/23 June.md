# Post Mortem Covering June 2026

During June 2026, the service experience multiple outages, affecting several TorBox functions. Some of these outages caused TorBox to become entirely unavailable to users for extended periods of time and exposed critical weaknesses in our network posture.

We are once again sorry for letting you down and have compiled this post to outline exactly what happened and why. We will discuss what steps were taken to prevent these types of outages from happening again.

To compensate our valued users for this downtime, will be credeting all paid plans with an additional seven days of time automatically. Keep an eye on our [𝕏.com](https://x.com/torbox) for the details.

<sub>*This post is segmented into sections to improve readability with extra details provided for high severity incidents.*</sub>

## Establishing the timeline

### Persistent network attacks (Major)
*(Note: certain key facts have been intentionally left out to prevent those responsible from learning information about our mitigation that could enable future attacks)*

In early June we continued to experience periodic (small) outages arising from what appeared to have been attempts at mapping weak spots within certain segments of the networks we use. In consultation with our upstream the response was two-fold: utilise an external scrubbing service (Path.net) and bring online on-premise scrubbing applicances. Once online, the on-premise appliances began causing the core routing engines to reboot sporadically. The issue was traced to a bug causing Flowspec rules to crash our routing engines when mitigation was applied. An attempt was made to upgrade the JunOS firmware on these routing engines but it was unsuccessful prompting replacement hardware which was prepped and shipped to the datacentre. In the interim we continued using Path.net and service was stable.

<sub>Ref: </sub>
<sub>https://status.torbox.app/incident/911580</sub>
<sub>https://status.torbox.app/incident/912014</sub>

### [Replacement routing engines](https://status.torbox.app/maintenance/918940) (Minor)

During the second week of June the replacement hardware arrived and the routing engines were swapped out successfully. The entire process completed in under twenty minutes and the issue has not occured since.

### API outages (Major)

Two seperate issues surfaced in June involving the Redis cache cluster we use for our APIs:
1. A misconfiguration within the cache cluster itself would cause seemingly random timeouts. Poor error logging made this issue invisible to our monitoring systems.
2. The API cluster was calling the cache cluster in a manner than compounded the effects of the misconfiguration. Our initial response was unsuccessful as unrelated causes (detailed above) had to be ruled out before the direct cause was determined.

A fix was applied and the issue has not reoccured.

<sub>Ref: </sub>
<sub>https://status.torbox.app/incident/922335</sub>
<sub>https://status.torbox.app/incident/926041</sub>

### [WEUR mitigation profiles](https://status.torbox.app/incident/922861) (Minor)

Around mid-June, our WEUR location began tripping mitigation filters due to a newly installed mitigation profile previously put in place to guard against genuine attacks. This profile was unfortunately set too aggresively and packets from our own storage cores were being dropped. In consultation with our upstream provider the profile was refined to only detect and filter out undesired traffic. This issue was poorly detected by our volume based traffic monitoring as it was filtering out actual attack traffic (thus mitigation notices were lost in the noise) as well as some legitimate traffic, but not enough to raise an alarm.

