# Deploy S3 CSI Driver on AWS China

### 1、创建IAM Policy
使用如下json创建名为AmazonS3CSIDriverPolicy的IAM Policy
BUCKET_NAME需要替换
```
{
   "Version": "2012-10-17",
   "Statement": [
        {
            "Sid": "MountpointFullBucketAccess",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws-cn:s3:::BUCKET_NAME"
            ]
        },
        {
            "Sid": "MountpointFullObjectAccess",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:AbortMultipartUpload",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws-cn:s3:::BUCKET_NAME/*"
            ]
        }
   ]
}
```
### 2、创建IAM ServiceAccount
```
CLUSTER_NAME=test
REGION=cn-northwest-1
ROLE_NAME=AmazonEKS_S3_CSI_DriverRole
## ACCOUNT_ID需要替换
POLICY_ARN=arn:aws-cn:iam::ACCOUNT_ID:policy/AmazonS3CSIDriverPolicy

eksctl create iamserviceaccount \
    --name s3-csi-driver-sa \
    --namespace kube-system \
    --cluster $CLUSTER_NAME \
    --attach-policy-arn $POLICY_ARN \
    --approve \
    --role-name $ROLE_NAME \
    --region $REGION \
    --role-only
```
### 3、Helm Chart安装S3 CSI Driver
### 3.1 安装Helm    
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
### 3.2 Helm安装s3 csi driver
```
helm repo add aws-mountpoint-s3-csi-driver https://awslabs.github.io/mountpoint-s3-csi-driver
helm repo update

helm upgrade --install aws-mountpoint-s3-csi-driver \
   --namespace kube-system \
   --set node.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws-cn:iam::$ACCOUNT_ID:role/AmazonEKS_S3_CSI_DriverRole" \
   aws-mountpoint-s3-csi-driver/aws-mountpoint-s3-csi-driver 

# 查看s3 csi driver pod状态
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-mountpoint-s3-csi-driver
NAME                READY   STATUS    RESTARTS   AGE
s3-csi-node-4psk7   3/3     Running   0          23m
```

### 4、按照如下创建pv，pvc和pod进行测试
```
# REGION_CODE和BUCKET_NAME需要替换

cat << EOF > static_provisioning.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: s3-pv
spec:
  capacity:
    storage: 1200Gi # ignored, required
  accessModes:
    - ReadWriteMany # supported options: ReadWriteMany / ReadOnlyMany
  mountOptions:
    - allow-delete
    - region REGION_CODE
  csi:
    driver: s3.csi.aws.com # required
    volumeHandle: s3-csi-driver-volume
    volumeAttributes:
      bucketName: BUCKET_NAME
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: s3-claim
spec:
  accessModes:
    - ReadWriteMany # supported options: ReadWriteMany / ReadOnlyMany
  storageClassName: "" # required for static provisioning
  resources:
    requests:
      storage: 1200Gi # ignored, required
  volumeName: s3-pv
---
apiVersion: v1
kind: Pod
metadata:
  name: s3-app
spec:
  containers:
    - name: app
      image: public.ecr.aws/docker/library/centos:centos7
      command: ["/bin/sh"]
      args: ["-c", "echo 'Hello from the container!' >> /data/$(date -u).txt; tail -f /dev/null"]
      volumeMounts:
        - name: persistent-storage
          mountPath: /data
  volumes:
    - name: persistent-storage
      persistentVolumeClaim:
        claimName: s3-claim
EOF

# 部署应用 
kubectl apply -f static_provisioning.yaml
 
# 查看Pod
kubectl get pod s3-app
NAME     READY   STATUS    RESTARTS   AGE
s3-app   1/1     Running   0          9m11s

# 查看s3是否有文件生成
aws s3 ls <BUCKET_NAME>
2024-08-16 08:47:20         26 Fri Aug 16 08:47:19 UTC 2024.txt
```
