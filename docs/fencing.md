# Fencing

## How do we verify the real primary?
We start out evaluating the cluster state by checking each registered standby for connectivity and asking who their primary is.

The "clusters state" is represented across a few different dimensions:

**Total members**
Number of registered members, including the primary.

**Total active members**
Number of members that are responsive.  This includes the primary we are evaluating, so this will never be less than one.

**Total inactive members**
Number of registered members that are non-responsive.

**Conflict map**
The conflict map is a `map[string]int` that tracks conflicting primary's queried from our standbys and the number of occurrences a given primary was referenced.

As an example, say we have a 3 member cluster and both of the standby's indicate their registered primary does not match.  This will be recorded as:
```
map[string]int{
  "fdaa:0:2e26:a7b:8c31:bf37:488c:2": 2
}
```

The real primary is resolvable so long as the majority of members can agree on who it is.  Quorum being defined as `total_members / 2 + 1`.

**There is one exception to note here. When the primary being evaluated meets quorum, it will still be fenced in the event a conflict is found.  This is to protect against a possible race condition where the old primary comes back up during an active failover.**

Tests can be found here: https://github.com/fly-apps/postgres-flex/pull/49/files#diff-3d71960ff7855f775cb257a74643d67d2636b354c9d485d10c2ded2426a7f362

## What if the real primary can't be resolved or doesn't match the booting primary?

In both of these instances the primary member will be fenced.

**If the real primary is resolvable**
The cluster will be made read-only, the PGBouncer will be reconfigured to target the "real" primary and the ip address is written to a `zombie.lock` file.  The PGBouncer reconfiguration will ensure that any connections hitting this member will be routed to the real primary in order to minimize interruptions.  Once completed there will be panic to force a full member restart. When the member is restarted, we will read the ip address from the `zombie.lock` file and use that to attempt to rejoin the cluster we diverged from.  If we are successful, the `zombie.lock` is cleared and we will boot as a standby.

**Note: We will not attempt to rejoin a cluster if the resolved primary resides in a region that differs from the `PRIMARY_REGION` environment variable set on self.  The `PRIMARY_REGION` will need to be updated before a rejoin will be attempted.**

**If the real primary is NOT resolvable**
The cluster will be made read-only, PGBouncer will remain disabled and a `zombie.lock` file will be created without a value.  When the member reboots, we will read the `zombie.lock` file and see that it's empty.  This indicates that we've entered a failure mode that can't be recovered automatically.  This could be an issue where previously deleted members were not properly unregistered, or the primary's state has diverged to a point where its registered members have been cycled out.