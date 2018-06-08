# Authentication &Authorization setup guide.

Authentication&Authorization setup is divided into multiple steps:

- enable RBAC in Kubernetes (k8s) cluster with kops
- define roles and role bindings (connect roles with k8s user groups)
- define AWS iam roles that corresponds to k8s groups from rolebindings (see previous step)
- define IAM policies to allow users to assume aforementioned roles
- Install heptio and configure it using IAM roles and k8s user groups
- Install locally token generator.

[toc]

## Before you begin

Initially you have to prepare your environment by setting necessary variables, switch to proper AWS profile and kubectl context

```
export ENV_NAME=<your env, stage or prod>
export NAME=<your_cluster_name>
export KOPS_STATE_STORE=<your_cluster_state_store>
export AWS_PROFILE=<name of AWS profile where your k8s cluster>
kubectl config use-context <k8s-cluster-context>
```

## Enable RBAC in k8s cluster with kops

Edit kops cluster config and add `authorization: { rbac: {} }` (or substitute `authorization: {alwaysAllow: {} }`)

```bash
kops edit cluster ${NAME}
```

```yaml
    apiVersion: kops/v1alpha2
    kind: Cluster
    spec:
      authorization:
        rbac: {}
```

After that run `kops update cluster ${NAME} --yes` and `rolling-update cluster ${NAME} --yes` to apply changes.

**Note that you might need to update your existing k8s entities according to new rbac permission model (redeploy helm, ingress etc.)**

## Define k8s roles and role bindings

Here you simply create YAML files with Role(ClusterRole), RoleBinding(ClusterRoleBinding) definitions according your permission model needs. You can see the example of [readonly ClusterRole(and it's readonly user group binding)](server-setup/group-n-roles.yaml)

For simplicity lets stay with Kubernetes default `system:masters` user group and our `readonly` user group.

## Define AWS iam roles that corresponds to k8s groups from rolebindings

On the previous step we've created k8s `readonly` user group and decided to use internal `system:masters` group for root access. Now lets create AWS IAM roles to map IAM users to k8s groups.

Lets create 2 IAM roles: `k8s-admin` (for root access) and `k8s-readonly` (for radonly access). If you have multiple clusters or multiple envs, you can create additional `k8s-stage-admin`/`k8s-release-admin` etc. roles.

Here is an example how to create role in terminal

```bash
# get your account ID
ACCOUNT_ID=$(aws sts get-caller-identity --output text --query 'Account')

# define a role trust policy that opens the role to users in your account (limited by IAM policy)
POLICY=$(echo -n '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":"arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':root"},"Action":"sts:AssumeRole","Condition":{}}]}')

# create a role named k8s-admin-prod (will print the new role's ARN)
aws iam create-role \
  --role-name k8s-admin-prod \
  --description "Kubernetes administrator production role (for Heptio Authenticator for AWS)." \
  --assume-role-policy-document "$POLICY" \
  --output text \
  --query 'Role.Arn'
```

Please note that here we created role which can be assumed only user from original account. You can also allow users from different accounts to assume this role, but it's out of the scope of our guide.

Here is also example of Terraform definition of the role, where the role can be assumed by users from two accounts (ACC_1 and ACC_2)

```terraform
resource "aws_iam_role" "k8s-prod-admin" {
  name = "k8s-prod-admin"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACC_1:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {}
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACC_2:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {}
    }
  ]
}
EOF
}
```

## Define IAM policies to allow users to assume aforementioned roles

We won't stop here a lot. Here is just Terraform definition of such a policy and policy attachment.

```terraform
resource "aws_iam_policy" "k8s-auth-admin" {
  name = "k8s-auth-admin"

  policy = <<EOF
{
    "Version": "2012-10-17",
    "Statement": {
        "Effect": "Allow",
        "Action": "sts:AssumeRole",
        "Resource": "${aws_iam_role.k8s-admin.arn}"
    }
}
EOF
}

resource "aws_iam_group_policy_attachment" "k8s-admins-access" {
  group      = "${aws_iam_group.k8s-admins.name}"
  policy_arn = "${aws_iam_policy.k8s-auth-admin.arn}"
}
```

## Install heptio and configure it using IAM roles and k8s user groups

To install Heption Authenticator on a Kops cluster we need to do following steps:

0. Choose `ClusterID` for your k8s cluster. Here we simply assign clustername as cluster id.

```bash
export CLUSTER_ID=${NAME}
```

1.  Pre-generate the certificate, key, and kubeconfig and upload them to the kops state store.

    ```
    docker run --name=heptio-init gcr.io/heptio-images/authenticator:v0.1.0 init -i ${CLUSTER_ID}
    docker cp heptio-init:/cert.pem ./cert.pem
    docker cp heptio-init:/key.pem ./key.pem
    docker cp heptio-init:/heptio-authenticator-aws.kubeconfig ./heptio-authenticator-aws.kubeconfig

    aws s3 cp cert.pem ${KOPS_STATE_STORE}/${NAME}/addons/authenticator/cert.pem;
    aws s3 cp key.pem ${KOPS_STATE_STORE}/${NAME}/addons/authenticator/key.pem;
    aws s3 cp heptio-authenticator-aws.kubeconfig ${KOPS_STATE_STORE}/${NAME}/addons/authenticator/kubeconfig.yaml;
    ```

2.  Add the following sections to the cluster spec, either using `kops edit cluster ${NAME}` or editing the manifest yaml file.

    Be sure to replace `KOPS_STATE_STORE` and `NAME` with their appropriate values since those environment variables are not available at runtime on Kubernetes instances.
    This downloads the files from the state store on masters to a directory that is volume mounted by kube-apiserver.
    Kops does not support adding additional volumes to kube-apiserver so we must reuse the existing `/srv/kubernetes` hostPath volume.

    Also please note we added `authorization: { rbac: {} }` here.

    ```
    apiVersion: kops/v1alpha2
    kind: Cluster
    spec:
      authorization:
        rbac: {}
      kubeAPIServer:
        authenticationTokenWebhookConfigFile: /srv/kubernetes/heptio-authenticator-aws/kubeconfig.yaml
      hooks:
      - name: kops-hook-authenticator-config.service
        before:
          - kubelet.service
        roles: [Master]
        manifest: |
          [Unit]
          Description=Download Heptio AWS Authenticator configs from S3
          [Service]
          Type=oneshot
          ExecStart=/bin/mkdir -p /srv/kubernetes/heptio-authenticator-aws
          ExecStart=/usr/local/bin/aws s3 cp  --region=us-east-1 --recursive s3://KOPS_STATE_STORE/NAME/addons/authenticator /srv/kubernetes/heptio-authenticator-aws/
    ```

3. Apply the changes with `kops update cluster ${NAME}`.
   If the cluster already exists, roll the cluster with `kops rolling-update cluster ${NAME}` in order to recreate the master nodes.

4. Create heptio authenticator daemon set. You can see example of DaemonSet definition [here](server-setup/heptio-daemonset.yaml). Be sure to replace `clusterID` value with your cluster name – `${CLUSTER_ID}`
