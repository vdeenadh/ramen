# VolumeReplicationGroup (VRG) Sequence Diagram

The sequence below shows DR protection, failover and relocate of application.  This is a VRG focussed diagram and for simplicity (to reduce clutter), this diagram views the actions performed by ACM, DRPC or user as if they are performed by a single actor or entity.

This version of the diagram only shows `ClusterData` related VRG conditions and does not show `Data` related VRG conditions.

```mermaid
sequenceDiagram
autonumber
participant C1App1 as App1 in ClusterA
participant C1App1 as App1 in ClusterA
participant C1VRG1 as ns1/vrg1 in ClusterA
Actor User as ACM/DRPC/User
participant C2VRG1 as ns1/vrg1 in ClusterB
participant C2App1 as App1 in ClusterB
% rect rgb(100,200,250)
    loop for all required apps
        note left of User: Deploy app in Cluster1 and DR protect it
        User ->> C1App1 : Deploy app in ns1
        activate C1VRG1
        User -) +C1VRG1:Create VRG1 with [spec.state = Primary] in ns1
        C1VRG1 ->> C1VRG1: Protect App1
        % C1VRG1 -) User:VRG1.status.Cond.DataProtected = True
        C1VRG1 -) User:VRG1.status.Cond.ClusterDataProtected = True
        % C1VRG1 -) User:App Protected [spec.status = Primary]
        note left of User: App is DR protected
    end
% end

    note left of User: Fence Cluster1 in preparation to failover to Cluster2
    User ->> User: Fence Cluster1's access to Cluster2's replica store (for both sync and async modes)
    User ->> User: Fence Cluster2's access to Cluster1's replica store (for both sync and async modes)
    User ->> User: Fence Cluster1's access to Cluster2's volume data (for sync mode only)
    User ->> User: Fence Cluster2's access to Cluster1's volume data (for sync mode only)

% rect rgb(250,125,0)
    note right of User: Failover DR protected apps to Cluster2
    loop for all required apps
        activate C2VRG1
        User -) C2VRG1:Create VRG1 with [spec.state = Primary] in ns1
        % C2VRG1 -) User:VRG1.status.Cond.DataReady = True
        C2VRG1 ->> C2App1: Restore app
        activate C2App1
        C2VRG1 -) User:VRG1.status.Cond.ClusterDataReady = True
        note right of User: App is restored
        C2VRG1 ->> C2VRG1: Protect App1
        C2VRG1 -) User:VRG1.status.Cond.ClusterDataProtected = True
        note right of User: App is DR protected
    end
% end

% rect
    note left of User: Unfence Cluster1 in preparation to failback to Cluster1
    loop for all apps failed over to Cluster2
        User ->> C1VRG1: Update VRG1 with [spec.state = Secondary] in ns1 in preparation to unfence
        User ->> C1App1: Undeploy app
        User -x C1VRG1: Delete VRG1
    end
    User ->> User: Unfence Cluster1 (only for MetroDR)

% end

% rect rgb(200,200,100)
    note left of User: Relocate DR protected apps to Cluster1
    loop for all required apps
        User ->> C2VRG1: Update VRG1 with [spec.state = Secondary] in ns1
        User -x C2App1: Undeploy the app
        C2VRG1 ->> User: VRG1.status.state = Secondary
        note left of User: Regional DR resync requirement
        User ->> C1VRG1: Create VRG1 with [spec.state = Secondary] in ns1 to start a resync
        User -x C2VRG1: Delete VRG1 with [spec.state = Secondary] in ns1
        activate C1VRG1
        User -) C1VRG1:Update VRG1 with [spec.state = Primary] in ns1
        % C1VRG1 -) User:VRG1.status.Cond.DataReady = True
        C1VRG1 ->> C1App1: Restore app
        activate C1App1
        C1VRG1 -) User:VRG1.status.Cond.ClusterDataReady = True
        note left of User: App is restored
        C1VRG1 ->> C1VRG1: Protect App1
        C1VRG1 -) User:VRG1.status.Cond.ClusterDataProtected = True
        note left of User: App is DR protected
    end
% end
```
