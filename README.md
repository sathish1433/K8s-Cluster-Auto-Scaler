The following is an example to make use of the AWS IAM OIDC with the Cluster Autoscaler in an EKS cluster.
Prerequisites

    * An Active EKS cluster (1.14 preferred since it is the latest) against which the user is able to run kubectl commands.
    * Cluster must consist of at least one worker node ASG.
    * Create an IAM OIDC identity provider for your cluster with the AWS Management Console using the documentation 
            Create OIDC provider (AWS Console)
            
            - Open the Amazon EKS console
            - In the left pane, select Clusters, and then select the name of your cluster on the Clusters page.
            - In the Details section on the Overview tab, note the value of the OpenID Connect provider URL.
            - Open the IAM console at https://console.aws.amazon.com/iam/
            - In the left navigation pane, choose Identity Providers under Access management. If a Provider is listed that matches the URL for your cluster, then you already have a provider for your cluster. If a provider isn’t listed that matches the URL for your cluster, then you must create one.
            - To create a provider, choose Add provider.
            - For Provider type, select OpenID Connect.
            - For Provider URL, enter the OIDC provider URL for your cluster.
            - For Audience, enter sts.amazonaws.com. (Optional) Add any tags, for example a tag to identify which cluster is for this provider.
            Choose Add provider.
    * Create a test IAM policy for your service accounts.
    
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

Note: it for oidc role creating purpose only.it an mandatory to assign policy pupose.

  * Create an IAM role for your service accounts in the console.
      

    Retrieve the OIDC issuer URL from the Amazon EKS console description of your cluster . It will look something identical to: 'https://oidc.eks.us-east-1.amazonaws.com/id/xxxxxxxxxx'

    While creating a new IAM role, In the "Select type of trusted entity" section, choose "Web identity".

    In the "Choose a web identity provider" section: For Identity provider, choose the URL for your cluster. For Audience, type sts.amazonaws.com.

    In the "Attach Policy" section, select the policy to use for your service account, that you created in Section B above.

    After the role is created, choose the role in the console to open it for editing.

    Choose the "Trust relationships" tab, and then choose "Edit trust relationship". Edit the OIDC provider suffix and change it from :aud to :sub. Replace sts.amazonaws.com to your service account ID.
    ex. "oidc.eks.ap-south-1.amazonaws.com/id/<eks-cluster-id>:aud": "sts.amazonaws.com"   to  "oidc.eks.ap-south-1.amazonaws.com/id/FDCB5F706CD563B6A2866E80217B25FF:sub": "system:serviceaccount:kube-system:cluster-autoscaler"   

    Update trust policy to finish.
    
  * Set up Cluster Autoscaler Auto-Discovery using the tutorial .

    Open the Amazon EC2 console, and then choose EKS worker node Auto Scaling Groups from the navigation pane.
    In the "Add/Edit Auto Scaling Group Tags" window, please make sure you enter the following tags by replacing 'awsExampleClusterName' with the name of your EKS cluster. Then, choose "Save".

        Plugin 	 README
        Key: 	  k8s.io/cluster-autoscaler/enabled
        Key: 	  k8s.io/cluster-autoscaler/'awsExampleClusterName'

  * Create an IAM Policy for cluster autoscaler and to enable AutoDiscovery as well as discovery of instance types.


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

    Attach the above created policy to the instance role that's attached to your Amazon EKS worker nodes and this oidc role.
    
    Download a deployment example file provided by the Cluster Autoscaler project on GitHub, run the following command:
    
    $ wget https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

    the values for AWS_REGION, AWS_ROLE_ARN and AWS_WEB_IDENTITY_TOKEN_FILE where the role arn must be the same as the role provided in the service account annotations.

    $ kubectl apply -f .yml

    The cluster autoscaler scaling the worker nodes can also be tested:

    $ kubectl scale deployment autoscaler-demo --replicas=50
deployment.extensions/autoscaler-demo scaled
 
$ kubectl get deployment
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
autoscaler-demo   55/55   55           55          143m

Snippet of the cluster-autoscaler pod logs while scaling:

I1025 13:48:42.975037       1 scale_up.go:529] Final scale-up plan: [{eksctl-xxx-xxx-xxx-nodegroup-ng-xxxxx-NodeGroup-xxxxxxxxxx 2->3 (max: 8)}]

    

