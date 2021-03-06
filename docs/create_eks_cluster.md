# Create EKS Cluster
Kubernetes requires a cluster for deployments. This will walk through creating a cluster in AWS with `eksct

### 1. Install CLIs
  1. awscli
    
      ```
      brew install python3
      pip3 install awscli --upgrade --user
      ```

  2. kubectl: https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-macos
      
      ```
      curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl

      chmod +x ./kubectl

      sudo mv ./kubectl /usr/local/bin/kubectl

      KUBECONFIG="$HOME/.kube/config"

      kubectl version
      ```

  3. eksctl
      ```
      curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

      sudo mv /tmp/eksctl /usr/local/bin
      
      # OR

      brew tap weaveworks/tap

      brew install weaveworks/tap/eksctl
      ```
  4. kubectx
      ```
      brew install kubectx
      ```

### 2. Get AWS access key/secret.
> Note: If you have an admin key and secret already, skip to step 3.

  1. Create IAM user. Use admin permissions for now. https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console
  2. Copy/save key and secret that is generated. 

### 3. Configure aws cli
```
aws configure --profile aws_admin
```
Default Region: {leave_empty}
Default output format: json

Then, set created profile as default for current shell: 
```
export AWS_DEFAULT_PROFILE="aws_admin"
```


### 4. Set up eks_cluster.yaml file
For eksctl, you can create a cluster with a command, or with a file. 
I've found the file to be better because:
- you have a stored representation of your cluster
- you can edit the yaml in the future and apply the changes (example, to add a new node-group)


sample yaml file: 
```yaml
## This is a SAMPLE eksctl deploy file. find more at: 
# https://github.com/weaveworks/eksctl/tree/master/examples
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: cluster-name
  region: us-east-1

# vpc:
#   id: "vpc-0ff086f1cae7defff"  # (optional, must match VPC ID used for each subnet below)
#   subnets:
#     # must provide 'private' and/or 'public' subnets by availibility zone as shown
#     public:
#       us-west-2d:
#         id: "subnet-0030b2ed693c99fff"
#       us-west-2b:
#         id: "subnet-08fadddc16ea55fff"
#       us-west-2c:
#         id: "subnet-05d8af2ff3770afff"

nodeGroups:
  - name: base
    instanceType: t3.xlarge
    desiredCapacity: 2
    availabilityZones: ["us-east-2a", "us-east-2b","us-east-2c"]
    iam:
      withAddonPolicies:
        imageBuilder: true
        autoScaler: true
        externalDNS: true
        certManager: true
        appMesh: true
        ebs: true
        fsx: true
        efs: true
        albIngress: true
        xRay: true
        cloudWatch: true

fargateProfiles:
  - name: fp-default
    selectors:
      # All workloads in the "default" Kubernetes namespace will be
      # scheduled onto Fargate:
      - namespace: default
      # All workloads in the "kube-system" Kubernetes namespace will be
      # scheduled onto Fargate:
      - namespace: kube-system
  - name: fp-dev
    selectors:
      # All workloads in the "dev" Kubernetes namespace matching the following
      # label selectors will be scheduled onto Fargate:
      - namespace: dev
        labels:
          env: dev
          checks: passed

```

view additional settings from: https://eksctl.io/
and additional fargate settings from: https://eksctl.io/usage/fargate-support/

once you like your yaml, run: 
```
eksctl create cluster --config-file=<path> \
  --kubeconfig=$HOME/.kube/config \
  --set-kubeconfig-context=true \
  --profile=${AWS_DEFAULT_PROFILE}
```
>**NOTE** This will take some time ~15-30mins
```
##SAMPLE OUTPUT
#[ℹ]  using region us-east-1
#[ℹ]  subnets for us-east-1a - public:192.168.0.0/19 private:192.168.96.0/19
#[ℹ]  subnets for us-east-1b - public:192.168.32.0/19 private:192.168.128.0/19
#[ℹ]  subnets for us-east-1d - public:192.168.64.0/19 private:192.168.160.0/19
#[ℹ]  nodegroup "ng-9d03bfd5" will use "ami-0abcb9f9190e867ab" [AmazonLinux2/1.12]
#[ℹ]  creating EKS cluster "sg-eks-test-1" in "us-east-1" region
#...
#[✔]  EKS cluster "sg-eks-test-1" in "us-east-1" region is ready
```

### 5. test kubectl with new context
First, make sure you are using the right context. 
```
#view available contexts
kubectx

#rename your context to a friendly name
kubectx <NEW_NAME>=<NAME> 

#use that context
kubectx <NEW_NAME>
```

Then test it!

```
kubectl create deployment pingdataconsole --image pingidentity/pingdataconsole:edge

kubectl get all

kubectl delete deployment pingdataconsole
```

### 6. Create a "namespace" 
```
kubectl create namespace ping-poc-base
```

### 7. Create RBAC roles

- [for a poc](#for-a-poc)
- [for a multi-tenant cluster](for-a-multi-tenant-cluster)

#### For a POC: 
1. For this, whichever user creates the EKS cluster (aws_admin), should create a second IAM user with privileges for **EKS only**. 

    - Follow https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console 
    - When selecting permissions: 1. "Attach Existing Policies directly" 2. search for "eks", add the 4 that appear. 
      > **Note**: hold on to the key/secret!!

2. Add this user must be added to the cluster's "config-map":
    1. Log in to `awscli` with the new user and get the user's info

        ```
        aws configure --profile base_eks_user
        export AWS_DEFAULT_PROFILE="base_eks_user"
        aws sts get-caller-identity
        ```
        > save output from this ^

    2. Switch back to the original user that created the cluster: 
        ```
        export AWS_DEFAULT_PROFILE=aws_admin
        kubectl edit -n kube-system configmap/aws-auth
        ```
    3. This opens an editor.  You will need to add a "mapUsers" section with some data. Sample file for mapping below:

          ```
          apiVersion: v1
          data:
            mapRoles: |
              - groups:
                - system:bootstrappers
                - system:nodes
                rolearn: arn:aws:iam::916400927078:role/eksctl-sg-eks-test-1-nodegroup-ng-NodeInstanceRole-1VU4TGY8HOH79
                username: system:node:{{EC2PrivateDNSName}}
            mapUsers: |
              - userarn: arn:aws:iam::916400927078:user/ping_eks_user
                username: ping_eks_user
                groups: 
                  - system:masters
          kind: ConfigMap
          metadata:
            creationTimestamp: "2019-07-15T14:04:49Z"
            name: aws-auth
            namespace: kube-system
            resourceVersion: "17834"
            selfLink: /api/v1/namespaces/kube-system/configmaps/aws-auth
            uid: 8259e69f-a709-11e9-9281-0e94073a7804
          ```


#### For a Mult-Tenant Cluster

the following steps will create user-accounts and a namespace for every person that is to use the cluster. 

1. create cluserrole: https://gitlab.corp.pingidentity.com/devops-program/gte-devops-scripts/blob/master/rsa-cluster-rbac/cluster-role.yaml

2.  create an AWS IAM group for EKS users. 

3. Create an AWS IAM user for each user and add them to the group. 

4. see the scripts at: https://gitlab.corp.pingidentity.com/devops-program/gte-devops-scripts/blob/master/rsa-cluster-rbac/create-identity-mapping.sh

>make sure your AWS profile is the one that created the EKS Cluster, then run this script for each user:

```
./create-identity-mapping.sh <aws-iam-username>
```

## Next Steps: 
- be prepared to provide the user/key info to the users.
- may need to add URLs and hosted zones to route53. 

