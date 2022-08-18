# VolumeReplicationGroup (VRG) CR Status Conditions

This page outlines the various conditions reported by the VolumeReplicationGroup (VRG) CR that is deployed on the DR cluster.

1. VRG reports `Data` related conditions and `ClusterData` related conditions.
   - `Data` refers to PVC data contents
   - `ClusterData` refers to PVC related etcd data
      - `PersistentVolume` or `PV`.
      - In future: `PersistentVolumeClaim` or `PVC`.
      - In future: Other kube objects, if VRG is enabled to protect them.
1. The conditions reported by the VRG depends upon its configuration.
    - The protection mode: `sync` or `async`.
    - PVC access mode: RWO or RWX of a `async` mode VRG.
    - TBD: Ideally, a Kubernetes CR should report all conditions, even if some conditions
    are not applicable (their state can be false, say with `Not applicable` reason).
1. A VRG in `async` mode is responsible for `Data` replication and `ClusterData` replication.
   Hence, both `Data` related conditions and `ClusterData` related conditions are applicable.
   - A VRG in async mode with PVCs in RWX mode reports VolSync related `Data` related conditions.
   These conditions are not yet shown below.
   - TBD: The complex state transition details within each `Data` related conditions on primary and
   secondary cluster needs to shown accurately in the figures below.
1. A VRG in `sync` mode is `not` responsible for `Data` replication but is responsible for
   `ClusterData` replication.  Hence, `Data` related conditions are not applicable in `sync` mode
   but `ClusterData` related conditions are applicable.
   TBD: A VRG in `sync` mode could set its `Data` related conditions to false state with `Not applicable` reason.

<!---
For the hub cluster related CR conditions, see [DRPC CR Status Conditions](drpc-status-conditions.md) and
[DRPolicy CR Status Conditions](drpolicy-status-conditions.md).
-->

```mermaid
stateDiagram-v2

direction LR
VolumeReplicationGroup : VolumeReplicationGroup (VRG) CR Status Overview
state VolumeReplicationGroup {
    VRGStatus : Status

    ProtectedPVCs : Status of the set of PVCs \nprotected by the VRG
    VRGLevelConditions : A set of VRG-level conditions
    PVCLevelConditions : A set of PVC-level conditions \n (one per PVC)
    VRGClusterDataRelatedConditions : Cluster Data Related Conditions
    VRGDataRelatedConditions : Data Related Conditions

    VRGStatus --> VRGLevelConditions : contains
    VRGStatus --> ProtectedPVCs: contains
    ProtectedPVCs --> PVCLevelConditions : each PVC contains

    VRGLevelConditions --> VRGClusterDataRelatedConditions : contains
    VRGLevelConditions --> VRGDataRelatedConditions : contains
    PVCLevelConditions --> VRGClusterDataRelatedConditions : contains
    PVCLevelConditions --> VRGDataRelatedConditions : contains
}

note right of VolumeReplicationGroup
    - In async mode, VRG level `Data` related
    conditions mostly depend on its PVC level
    conditions.

    - In sync mode, VRG does not control
    data replication and hence, data related
    conditions are not applicable.
end note


DataRelatedConditions : Data related conditions
state DataRelatedConditions {
    direction LR

    DataReadyCondition : DataReady condition
    DataProtectedCondition : DataProtected condition
}

ClusterDataRelatedConditions : ClusterData related conditions
state ClusterDataRelatedConditions {
    direction LR
    ClusterDataProtectedCondition : ClusterDataProtected condition
    ClusterDataReadyCondition : ClusterDataReady condition
}

state DataReadyCondition {
    direction LR
    DRFalse : False
    DRTrue : True
    DRUnknown : Unknown
    DRInitializing : Initializing
    DRReplicating : Replicating
    DRReplicated : Replicated
    DRReady : Ready
    DRProgressing : Progressing
    DRError: Error
    DRUnknownError: Unknown error

    state DRUnknown {
        [*] --> DRInitializing
    }

    state DRFalse {
        DRInitializing --> DRProgressing
        DRProgressing --> DRError
        DRError --> DRUnknownError
    }
    state DRTrue {
        DRReplicating --> DRReady
        DRReady --> DRReplicated
    }
}

state DataProtectedCondition {
    direction LR
    DPUnknown : Unknown
    DPInitializing : Initializing
    DPReplicating : Replicating
    DPProtected: DataProtected
    DPFalse : False
    DPTrue : True
    DPError: Error
    DPReady : Ready

    state DPUnknown {
        [*] --> DPInitializing
    }

    state DPFalse {
        DPInitializing --> DPError
        DPInitializing --> if_sync
    }

    state DPTrue {
        state if_primary <<choice>>
        state if_sync <<choice>>

        if_sync --> DPReady : sync
        if_sync --> if_primary : async

        if_primary --> DPProtected : primary
        if_primary --> DPReplicating : secondary
        DPProtected --> DPError
        DPReplicating --> DPError
    }

}

note right of DataRelatedConditions
    - VRG level DataReady condition becomes true
    when the DataReady condition of all the PVCs
    protected by it become true.
    - VRG level DataProtected condition becomes true
    when the DataProtected condition of all the PVCs
    protected by it become true.
end note

state ClusterDataProtectedCondition {
    direction LR
    CDPFalse : False
    CDPTrue : True
    CDPUnknown : Unknown
    CDPInitializing : Initializing
    CDPUploadError : Upload error
    CDPUploading : Uploading
    CDPUploaded : Uploaded

    state CDPUnknown {
        [*] --> CDPInitializing
    }

    state CDPFalse {
        CDPInitializing --> CDPUploadError
        CDPInitializing --> CDPUploaded
        CDPUploaded --> CDPUploadError
        CDPUploadError --> CDPUploaded
    }

    state CDPTrue {
        CDPUploaded
    }
}

state ClusterDataReadyCondition {
    direction LR
    CDRFalse : False
    CDRTrue : True
    CDRUnknown : Unknown

    state CDRUnknown {
        [*] --> CDRInitializing
    }

    state CDRFalse {
        CDRInitializing --> CDRRestored

        CDRInitializing : Initializing
    }

    state CDRTrue {
        CDRRestored : Restored
    }
}

note right of ClusterDataRelatedConditions
    ClusterDataProtected condition:

    - becomes true when VRG successfully
    uploads PV cluster data ("Uploaded").

    - "Upload error" results either when
    VRG has no S3 stores configured or when
    VRG is unable to upload to any of the S3 stores.

    - The "Uploading" state is defined but
    is currently not used, but may be used
    in future when VRG is configured to protect
    kube objects.
end note

note right of ClusterDataRelatedConditions
    ClusterDataReady condition:

    - becomes true when cluster data
    of PVs are "Restored" from the S3 store
    to the API server when VRG is created
    in "Primary" state

    - is only a VRG level condition and is
    not reported for each PVC.
end note


```

<!--
    state DataProtectedCondition {
        direction LR

        state DPUnknown {
            [*] --/ DPInitializing
        }


        state DPTrue {
            state if_sync <<choice>>
            state if_primary <<choice>>

            DPInitializing --/ if_sync
            if_sync --/ DPReady : sync
            if_sync --/ if_primary : async

            if_primary --/ DPProtected
            DPReplicating --/ DPProtected
        }

        state DPFalse {
            DPInitializing --/ DPError
            DPReplicating --/ DPError
        }

    }

    state DataReadyCondition {
        direction LR

        state DRUnknown {
            [*] --/ DRInitializing
        }

        state DRFalse {
            DRInitializing --/ DRError
            DRInitializing --/ DRReplicating
            DRReplicating --/ DRReplicated
            DRReplicating --/ DRProgressing
            DRReplicating --/ DRError
            DRReplicating --/ DRUnknownError
        }
        state DRTrue {
            DRReplicating --/ DRReady
        }

    }
}

-->
