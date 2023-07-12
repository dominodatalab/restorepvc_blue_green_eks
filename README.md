## Introduction

The intent of this page is to demonstrate the following on EKS based Domino installation

### What

Restore the PVC of an un-committed and stopped workspace in Domino A into Domino B and mount it on a Pod in Domino B. The result should be that the changes in the workspace made in Domino A are visible in pod of Domino B.

### Why

FINRA has been requesting Blue/Green deployments. Currently, when they upgrade, they make a fresh (Green) Domino install and migrate the metadata over. Except the workspace meta-data. This PoC aims to demonstrates a pre-requisite for successfully moving a pvc of a stopped workspace from Blue Domino to the Green version of Domino.

### Limitations

While we will demonstrate that the PVC can be successfully moved (even if its is encrypted by different KMS Keys), and mounted into a Pod in the compute namespace of Green Domino, more metadata is needed to restore the workspace. 

## Installation

Start two Domino installations using the Fleet Command. For example

- https://blue.cs.domino.tech

- https://green.cs.domino.tech

In both installations of Domino follow the steps in this Amazon article to install the Amazon EBS CSI driver feature: Kubernetes Volume Snapshots . 0The steps are ambiguous and incomplete. The missing parts are clarified below

On a machine where you can connect to both K8s instances using kubectl perform a git pull on the following git repos


```
cd $INSTALL_FOLDER
git clone https://github.com/kubernetes-csi/external-snapshotter.git
git clone https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git
```

This gives us to sub-folders
```
$INSTALL_FOLDER/aws-ebs-csi-driver
$INSTALL_FOLDER/external-snapshotter
```

Install the snapshot functionality
```
cd $INSTALL_FOLDER/external-snapshotter/client/config/crd
kubectl apply -k .
cd $INSTALL_FOLDER/external-snapshotter/deploy/kubernetes/snapshot-controller
kubectl apply -k .
```
The gist of how this process works is below:

> The volume snapshot controller watches Kubernetes Volume Snapshot CRD objects and triggers CreateSnapshot/DeleteSnapshot against a CSI endpoint. For more information, see CSI Snapshotter on GitHub.


The process of installing the CSI driver includes creating an Identity and Access Management (IAM) policy and a role that to use as part of the creation of a service account. AWS provides a managed policy for this called AmazonEBSCSIDriverPolicy. Details on how to create an IAM role, service account, and attach the required AWS managed policy can be found here.

```
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster $my-cluster-id \
    --role-name $AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
```

When you run this step you will be asked to add the OIDC provider attached to your EKS clusters to IAM.

Once the IAM Role and Service Accounts have been created, you can continue the installation Amazon EKS add-on support for the Amazon EBS CSI following the documentation. Missing parts

Do not forget to attach the Role you created above to the add-on

In this step you are doing the following:

a. Creating an AWS role

b. Attaching a managed policy AmazonEBSCSIDriverPolicy to this role

c. Attaching a custom policy which provides this role with the ability to use the KMS keys of both Domino installations. An example of such a policy is sameerw_AmazonEBSCSIDriverPolicy

At this point we are using [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) . Because the above process will create a K8s SA bound to this role

The full policy is 
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "kms:CreateGrant",
                "kms:ListGrants",
                "kms:RevokeGrant"
            ],
            "Resource": [
                "arn:aws:kms:us-west-2:946429944765:key/b85b982b-527f-4018-a5bb-6924eea4704b"
            ],
            "Condition": {
                "Bool": {
                    "kms:GrantIsForAWSResource": "true"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "kms:Encrypt",
                "kms:Decrypt",
                "kms:ReEncrypt*",
                "kms:GenerateDataKey*",
                "kms:DescribeKey"
            ],
            "Resource": [
                "arn:aws:kms:us-west-2:946429944765:key/b85b982b-527f-4018-a5bb-6924eea4704b"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "kms:CreateGrant",
                "kms:ListGrants",
                "kms:RevokeGrant"
            ],
            "Resource": [
                "arn:aws:kms:us-west-2:946429944765:key/f255c64f-a773-49bd-9b66-74f0926f5896"
            ],
            "Condition": {
                "Bool": {
                    "kms:GrantIsForAWSResource": "true"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "kms:Encrypt",
                "kms:Decrypt",
                "kms:ReEncrypt*",
                "kms:GenerateDataKey*",
                "kms:DescribeKey"
            ],
            "Resource": [
                "arn:aws:kms:us-west-2:946429944765:key/f255c64f-a773-49bd-9b66-74f0926f5896"
            ]
        },
         {
            "Effect": "Allow",
            "Action": [
                "kms:Decrypt",
                "kms:GenerateDataKeyWithoutPlaintext",
                "kms:CreateGrant"
            ],
            "Resource": "*"
        }
    ]
}
```
The part missing in the Amazon docs is adding this section which could be arguably tightened
```
{
            "Effect": "Allow",
            "Action": [
                "kms:Decrypt",
                "kms:GenerateDataKeyWithoutPlaintext",
                "kms:CreateGrant"
            ],
            "Resource": "*"
        }
```
And note that there are two sections similar to the snippet below, one for each Domino installation participating in this snapshot to pvc migrations. You can add as many participating Domino’s here
```
{
            "Effect": "Allow",
            "Action": [
                "kms:Encrypt",
                "kms:Decrypt",
                "kms:ReEncrypt*",
                "kms:GenerateDataKey*",
                "kms:DescribeKey"
            ],
            "Resource": [
                "arn:aws:kms:us-west-2:946429944765:key/b85b982b-527f-4018-a5bb-6924eea4704b"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "kms:CreateGrant",
                "kms:ListGrants",
                "kms:RevokeGrant"
            ],
            "Resource": [
                "arn:aws:kms:us-west-2:946429944765:key/f255c64f-a773-49bd-9b66-74f0926f5896"
            ],
            "Condition": {
                "Bool": {
                    "kms:GrantIsForAWSResource": "true"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "kms:Encrypt",
                "kms:Decrypt",
                "kms:ReEncrypt*",
                "kms:GenerateDataKey*",
                "kms:DescribeKey"
            ],
            "Resource": [
                "arn:aws:kms:us-west-2:946429944765:key/f255c64f-a773-49bd-9b66-74f0926f5896"
            ]
        }
```
 5. Install the volume snapshot class
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/examples/kubernetes/snapshot/manifests/classes/snapshotclass.yaml
```

or 
```
kubectl apply -f $INSTALL_FOLDER/aws-ebs-csi-driver/examples/kubernetes/snapshot/manifests/classes/
```
This completes the installation.

## Demo Walkthrough

1. Create an unsaved workspace in the blue deployment

2. In the Blue Domino installation start a workspace in the quick-start project using the standard DAD.

3. Inside the workspace create a file /mnt/bluegreen.txt and add a line to the file
   
```
echo 'This is from blue' > /mnt/bluegreen.txt
```

But do not commit the changes and STOP the workspace. This is important. Sometimes the unsaved contents are not flushed until the workspace is stopped.

4. Create a Snapshot of this pvc in the Blue Domino

There are many ways to identify the pvc of the stopped workspace. One method is described here. But it should have the same is as your workspace. Ex. `workspace-64ac594fbe2c046835547935`

Identify the un-commited workspace pvc in the blue deployment

Open `vi aws-ebs-csi-driver/examples/kubernetes/snapshot/manifests/snapshot/yaml` and modify and apply it or run the following script
```
export pvc=workspace-64ac594fbe2c046835547935
cat <<EOF | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: $pvc
  namespace: domino-compute
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: $pvc
EOF
```
replace the `workspace-64ac594fbe2c046835547935` with your own 

This will trigger a EBS Snapshot

You can verify it is created using the following command-
```
kubectl -n domino-compute get volumesnapshot
```
You get the AWS snaphot_id from the following command
```
kubectl -n domino-compute get volumesnapshotcontent
```
It should have the same name as the volumesnapshot and if you open the yaml you will see something like this
```
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  name: workspace-64ac594fbe2c046835547935
  namespace: domino-compute
spec:
  volumeSnapshotRef:
    kind: VolumeSnapshot
    name: workspace-64ac594fbe2c046835547935
    namespace: domino-compute
  source:
    snapshotHandle: snap-0be6df94c470555c5
  driver: ebs.csi.aws.com
  deletionPolicy: Delete
  volumeSnapshotClassName: csi-aws-vsc
```
This line signifies the snaptshot handle on AWS. You can go to EC2 snapshots page and find it there.
```
snapshotHandle: snap-0be6df94c470555c5
```
You should note that it is encrypted using the KMS key associated with the blue cluster

Recover this PVC in the Green Domino

Go to the following folder 
```
cd $INSTALL_FOLDER/aws-ebs-csi-driver/examples/kubernetes/snapshot/manifests/snapshot-import/static-snapshot
vi volume-snapshot-content.yaml
```
Add the following lines with your values
```
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  name: workspace-64ac594fbe2c046835547935
  namespace: domino-compute
spec:
  volumeSnapshotRef:
    kind: VolumeSnapshot
    name: workspace-64ac594fbe2c046835547935
    namespace: domino-compute
  source:
    snapshotHandle: snap-0be6df94c470555c5
  driver: ebs.csi.aws.com
  deletionPolicy: Delete
  volumeSnapshotClassName: csi-aws-vsc
```
The most important values here are the name and snapshotHandle . The former is the same name as in the blue namespace and the latter is the snapshot handle in AWS.

Next 
```
vi volume-snapshot.yaml
```
and update as follows
```
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: workspace-64ac60febe2c04683554794d
  namespace: domino-compute
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    volumeSnapshotContentName: workspace-64ac594fbe2c046835547935
```
and apply
```
cd $INSTALL_FOLDER/aws-ebs-csi-driver/examples/kubernetes/snapshot/manifests/snapshot-import/static-snapshot/
kubectl apply -f .
```

We are all set. We can now recover the pvc in the Green deployment

```
cd $INSTALL_FOLDER/aws-ebs-csi-driver/examples/kubernetes/snapshot/manifests/snapshot-import/app
vi claim.yaml
```
Update this file
```
## Define the PVC

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: workspace-64ac594fbe2c046835547935
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: dominodisk
  resources:
    requests:
      storage: 10Gi
  dataSource:
    name: workspace-64ac594fbe2c046835547935
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```
and apply
```
kubectl -n domino-compute apply -f claim.yaml
```
Verify that the PVC is in the pending state

Now create a pod and attach this pvc to it
```
cd $INSTALL_FOLDER/aws-ebs-csi-driver/examples/kubernetes/snapshot/manifests/snapshot-import/app
vi pod.yaml
```
Update as follows
```
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: domino-compute
spec:
  containers:
  - name: app
    image: centos
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date -u) >> /data/out.txt; sleep 10000000; done"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: workspace-64ac594fbe2c046835547935
```
and apply
```
kubectl -n domino-compute apply -f pod.yaml
## When the pod is running
kubectl -n domino-compute exec -it app -- cat /data/mnt/bluegreen.txt
#You should see the content from the blue domino here
```

## This is not enough

We have the pvc in the Green Domino and can see the changes from the Blue Domino. We have been able to successfully import a KMS encrypted PVC from Blue to Green Domino.

However, this is not enough. There is metadata and possibly internal git artifacts that need to be changed to fully restore the workspace.

However, this was the first step
