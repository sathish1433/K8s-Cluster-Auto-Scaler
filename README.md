# Configure Cluster Autoscaler with AWS IAM OIDC on Amazon EKS

This guide outlines the steps required to set up the [Cluster Autoscaler](https://github.com/kubernetes/autoscaler) in an Amazon EKS cluster using AWS IAM OIDC authentication.

---

## Prerequisites

- Active Amazon EKS cluster (v1.14 or newer)
- At least one worker node ASG
- `kubectl` configured and working
- AWS CLI and necessary IAM permissions

---

## Step 1: Create IAM OIDC Identity Provider

1. Open the Amazon EKS console.
2. Select **Clusters** > your cluster.
3. Copy the **OpenID Connect provider URL**.
4. Open the [IAM Console](https://console.aws.amazon.com/iam/).
5. Go to **Access Management > Identity Providers**.
6. If OIDC provider not listed:
   - Click **Add provider**
   - **Type:** OpenID Connect
   - **Provider URL:** `<your-cluster-oidc-url>`
   - **Audience:** `sts.amazonaws.com`
   - Add optional tags
   - Click **Add provider**

---

## Step 2: Create IAM Policy for OIDC Role (Test Policy)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-pod-secrets-bucket/*"
      ]
    }
  ]
}
```

> Note: Only required for OIDC role creation, not for assignment.

---

## Step 3: Create IAM Role for Service Account

1. Go to IAM console → Roles → **Create role**
2. **Trusted entity type:** Web identity
3. **Provider:** `<your OIDC provider>`
4. **Audience:** `sts.amazonaws.com`
5. Attach the policy created in Step 2
6. Edit **Trust relationships**:
   - Replace `:aud` with `:sub`
   - Replace value with your service account:
     ```json
     "oidc.eks.<region>.amazonaws.com/id/<EKS_ID>:sub": "system:serviceaccount:kube-system:cluster-autoscaler"
     ```

---

## Step 4: Tag the Auto Scaling Group

1. Open the EC2 Console → **Auto Scaling Groups**
2. Add the following tags:

| Key                                               | Value |
|----------------------------------------------------|--------|
| `k8s.io/cluster-autoscaler/enabled`                | `true` |
| `k8s.io/cluster-autoscaler/<EKS-Cluster-Name>`     | `owned` |

---

## Step 5: Create IAM Policy for Cluster Autoscaler

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeLaunchTemplateVersions"
      ],
      "Resource": ["*"]
    },
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup"
      ],
      "Resource": ["*"]
    }
  ]
}
```

- Attach this policy to:
  - The IAM role used by your EKS worker node group
  - The IAM OIDC role for Cluster Autoscaler

---

## Step 6: Deploy Cluster Autoscaler

1. Download deployment manifest:
   ```sh
   wget https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
   ```

2. Update these fields in the manifest:
   - `AWS_REGION`
   - `AWS_ROLE_ARN`
   - `AWS_WEB_IDENTITY_TOKEN_FILE`

3. Apply the manifest:
   ```sh
   kubectl apply -f cluster-autoscaler-autodiscover.yaml
   ```

---

## Step 7: Test Autoscaler

1. Scale a test deployment:
   ```sh
   kubectl scale deployment autoscaler-demo --replicas=50
   ```

2. Verify deployments:
   ```sh
   kubectl get deployment
   ```

3. View autoscaler logs:
   ```sh
   kubectl logs -n kube-system deployment/cluster-autoscaler
   ```

> Sample log snippet:
```
I1025 13:48:42.975037       1 scale_up.go:529] Final scale-up plan: [{eksctl-xxx-xxx-xxx-nodegroup-ng-xxxxx-NodeGroup-xxxxxxxxxx 2->3 (max: 8)}]
```

---

## References

- [Cluster Autoscaler GitHub](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)
- [Amazon EKS IAM OIDC Documentation](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)
