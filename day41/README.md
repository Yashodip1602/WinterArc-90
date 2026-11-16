# Day 41 -- AWS EBS CSI Driver Lab

## Overview

In this lab, I configured the Amazon EBS CSI Driver in an Amazon EKS
cluster and tested Kubernetes dynamic volume provisioning with an AWS
EBS `gp3` volume.

The main goal was to create persistent storage for a Kubernetes Pod,
write data to it, delete and recreate the Pod, and confirm that the data
remained available.

## Environment

  Item                    Value
  ----------------------- -------------------
  EKS cluster             `day39-lab`
  AWS Region              `ap-south-1`
  Namespace               `dev`
  EBS volume type         `gp3`
  Requested volume size   `1Gi`
  StorageClass            `ebs-csi-gp3`
  PVC                     `ebs-test-pvc`
  Test Pod                `ebs-test-pod`
  CSI driver name         `ebs.csi.aws.com`

## 1. Installed and configured the EBS CSI Driver

-   Installed the EKS add-on `aws-ebs-csi-driver`.
-   Created the IAM role `AmazonEKS_EBS_CSI_DriverRole`.
-   Attached the AWS managed policy `AmazonEKS_EBS_CSI_DriverPolicy`.
-   Configured the role trust relationship for EKS Pod Identity.
-   Created a Pod Identity Association for:
    -   Cluster: `day39-lab`
    -   Namespace: `kube-system`
    -   Service account: `ebs-csi-controller-sa`

### Verification

The EBS CSI controller Pods were running with all six containers ready
(`6/6`).

The EBS CSI node plugin Pods were also running, and the CSI driver was
registered on the worker nodes.

The Pod Identity Agent logs confirmed that credentials were successfully
fetched for the role:

`AmazonEKS_EBS_CSI_DriverRole`

This confirmed that Pod Identity authentication was working.

## 2. Troubleshooting the StorageClass configuration

The first StorageClass was created with the name `ebs-csi.aws.com`, but
its provisioner value was mistakenly set to:

``` yaml
provisioner: ebs-csi.aws.com
```

The actual EBS CSI driver name is:

``` yaml
provisioner: ebs.csi.aws.com
```

The difference is important:

-   Incorrect: `ebs-csi.aws.com`
-   Correct: `ebs.csi.aws.com`

A PVC initially remained in `Pending` state because it referenced a
StorageClass that did not exist. After correcting the configuration, a
new StorageClass was created.

## 3. Created the correct StorageClass

File: `ebs-csi-gp3.yaml`

``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-csi-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

Applied it:

``` bash
kubectl apply -f ebs-csi-gp3.yaml
```

Verified it:

``` bash
kubectl get storageclass
```

The `ebs-csi-gp3` StorageClass showed the correct provisioner
`ebs.csi.aws.com`.

## 4. Created the PersistentVolumeClaim

File: `ebs-test-pvc.yaml`

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-test-pvc
  namespace: dev
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-csi-gp3
  resources:
    requests:
      storage: 1Gi
```

Applied it:

``` bash
kubectl apply -f ebs-test-pvc.yaml
```

The PVC was later successfully provisioned and reached the `Bound`
state.

## 5. Created the test Pod and mounted the volume

File: `ebs-test-pod.yaml`

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: ebs-test-pod
  namespace: dev
spec:
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - "while true; do sleep 3600; done"
      volumeMounts:
        - name: ebs-storage
          mountPath: /data
  volumes:
    - name: ebs-storage
      persistentVolumeClaim:
        claimName: ebs-test-pvc
```

Applied it:

``` bash
kubectl apply -f ebs-test-pod.yaml
```

Verified the Pod and PVC:

``` bash
kubectl get pvc,pod -n dev
```

Observed results:

-   PVC status: `Bound`
-   Pod status: `Running`
-   Volume capacity: `1Gi`
-   Access mode: `ReadWriteOnce`
-   StorageClass: `ebs-csi-gp3`

Kubernetes dynamically created a PersistentVolume backed by an EBS
volume.

## 6. Tested persistent data

Wrote a test file inside the mounted volume:

``` bash
kubectl exec -n dev ebs-test-pod -- sh -c \
  "echo 'EBS CSI Persistent Storage Working' > /data/test.txt && cat /data/test.txt"
```

Output:

``` text
EBS CSI Persistent Storage Working
```

This confirmed that the Pod could write to and read from the mounted EBS
volume.

## 7. Verified data survives Pod recreation

Deleted the test Pod:

``` bash
kubectl delete pod ebs-test-pod -n dev
```

Recreated it:

``` bash
kubectl apply -f ebs-test-pod.yaml
```

After the new Pod reached `Running`, read the existing file:

``` bash
kubectl exec -n dev ebs-test-pod -- cat /data/test.txt
```

Output:

``` text
EBS CSI Persistent Storage Working
```

The original file was still present after Pod recreation. This verified
that the data was stored on the persistent EBS volume, not only in the
Pod's temporary container filesystem.

## 8. Cleanup performed

Deleted the test Pod and PVC:

``` bash
kubectl delete pod ebs-test-pod -n dev --ignore-not-found
kubectl delete pvc ebs-test-pvc -n dev --ignore-not-found
```

Verified namespace PVCs:

``` bash
kubectl get pvc -n dev
```

At the time of cleanup, Kubernetes reported no PVC resources in the
`dev` namespace.

The PersistentVolume was first observed in `Released` state and was
subsequently no longer found when queried. The EBS CSI controller Pods
remained healthy (`6/6 Running`).

The following shared infrastructure was intentionally retained:

-   EBS CSI EKS add-on
-   IAM role and managed policy
-   EKS Pod Identity Association
-   `ebs-csi-gp3` StorageClass
-   Existing `gp2` StorageClass
-   Other workloads, including `s3-test`

## 9. Final outcome

  Task                                    Result
  --------------------------------------- ------------------------------
  EBS CSI add-on installation             Completed
  IAM role and policy configuration       Completed
  EKS Pod Identity association            Completed
  Pod Identity credential retrieval       Verified in agent logs
  CSI driver registration                 Verified
  Correct `gp3` StorageClass              Created and verified
  PVC dynamic provisioning                Successful (`Bound`)
  EBS volume mount in Pod                 Successful
  Write and read test                     Successful
  Data persistence after Pod recreation   Verified
  Test Pod and PVC cleanup                Completed
  Kubernetes PV removal                   Verified by `NotFound`
  AWS-side EBS volume deletion            **Needs final verification**

## 10. Final AWS billing check

The Kubernetes PV was removed, but AWS-side volume deletion should still
be independently confirmed.

Run:

``` bash
aws ec2 describe-volumes \
  --region ap-south-1 \
  --filters Name=tag:kubernetes.io/created-for/pvc/name,Values=ebs-test-pvc \
  --query 'Volumes[*].[VolumeId,State,Size,VolumeType]' \
  --output table
```

If the query returns no matching volumes, the test EBS volume is no
longer present under that filter. If a volume is returned, inspect its
state and confirm whether it is still attached or pending deletion
before taking action.

Remember: the `Delete` reclaim policy requests deletion of the backing
EBS volume when the PV is deleted, but AWS-side verification is still
recommended to confirm cleanup and avoid unexpected storage charges.

------------------------------------------------------------------------

## Key Learnings

1.  The EBS CSI Driver connects Kubernetes persistent volumes to AWS
    EBS.
2.  IAM permissions are required so the CSI controller can manage EBS
    volumes.
3.  EKS Pod Identity provides AWS credentials to the CSI controller
    without storing static AWS keys in the Pod.
4.  A StorageClass defines how dynamic storage is provisioned.
5.  A PVC requests storage, and Kubernetes dynamically creates a PV
    through the CSI provisioner.
6.  `WaitForFirstConsumer` delays volume provisioning/binding until a
    Pod uses the PVC, allowing Kubernetes to consider the Pod's
    scheduling topology.
7.  EBS volumes are Availability Zone specific.
8.  A Pod can be deleted and recreated while data remains on the same
    PVC-backed EBS volume.
9.  The CSI provisioner name must exactly match the registered driver
    name: `ebs.csi.aws.com`.

**Conclusion:** The EBS CSI lab successfully demonstrated dynamic EBS
volume provisioning and data persistence across Pod recreation.
Kubernetes test resources were cleaned up. The final AWS-side EBS volume
deletion check remains to be run.
