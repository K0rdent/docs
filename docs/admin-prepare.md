# Prepare k0rdent to create managed clusters on multiple providers

Managed clusters can be hosted on a number of different platforms. At the time of this writing, those platforms include:

- Amazon Web Services
- Microsoft Azure
- OpenStack
- VMware

## AWS

k0rdent is able to deploy managed clusters as both EC2-based Kubernetes clusters and EKS clusters. In both cases, you'll need to create the relevant credentials, and to do that you'll need to configure an IAM user. Follow these steps to make it possible to deploy to AWS:

1. Install k0rdent

    Follow the instructions in [Install k0rdent](admin-installation.md) to create a management cluster with k0rdent running.

2. Install `clusterawsadm`

    k0rdent uses the Cluster API (CAPI) to marshal clouds and infrastructures. For AWS, this means using the components from the Cluster API Provider AWS (CAPA) project. clusterawsadm, a CLI tool created by CAPA project, helps with AWS-specific tasks such as creating IAM roles and policies, as well as credential configuration. To install clusterawsadm on Ubuntu on x86 hardware, execute these commands:

    ```shell
    curl -LO https://github.com/kubernetes-sigs/cluster-api-provider-aws/releases/download/v2.7.1/clusterawsadm-linux-amd64
    sudo install -o root -g root -m 0755 clusterawsadm-linux-amd64 /usr/local/bin/clusterawsadm
    ```

3. Configure AWS IAM

    Next you'll need to create the IAM policies and service account k0rdent will use to take action within the AWS infrastructure. (Note that you only need to do this once.)

    The first step is to crete the IAM CloudFormation stack based on your admin user. Start by specifying the environment variables clusterawsadm will use as AWS credentials:

    ```shell
    export AWS_REGION=<EXAMPLE_AWS_REGION>
    export AWS_ACCESS_KEY_ID=<EXAMPLE_ACCESS_KEY_ID>
    export AWS_SECRET_ACCESS_KEY=<EXAMPLE_SECRET_ACCESS_KEY>
    export AWS_SESSION_TOKEN=<YOUR_SESSION_TOKEN> # Optional. If you are using Multi-Factor Auth.
    ```

4. Create the IAM CloudFormation stack

	Now use `clusterawsadm` to create the IAM CloudFormation stack:

	```shell
	clusterawsadm bootstrap iam create-cloudformation-stack
	```
	
5. Install the AWS CLI
 
	With the stack in place you can create the AWS IAM user. You can do this in the UI, but it's also possible to do it from the command line using the aws CLI tool.  Start by installing it, if you haven't already:

	```shell
	sudo apt install unzip
	curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" 
	unzip awscliv2.zip 
	sudo ./aws/install
	```

	The tool recognizes the environment variables you created earlier, so there's no need to login.

6. Create the IAM user. 

	The actual `user-name` parameter is arbitrary; you can specify it as anything you like:

	```shell
	aws iam create-user --user-name k0rdentQuickstart
	```
	```console
	{
	  "User": {
		  "Path": "/",
		  "UserName": "k0rdentQuickstart",
		  "UserId": "EXAMPLE_USER_ID",
		  "Arn": "arn:aws:iam::FAKE_ARN_123:user/k0rdentQuickstart",
		  "CreateDate": "2025-01-18T08:15:27+00:00"
	  }
	}
	```

7. Assign the relevant policies
 
	You'll need to assign the following policies to the user you just created:

	```shell
	control-plane.cluster-api-provider-aws.sigs.k8s.io
	controllers.cluster-api-provider-aws.sigs.k8s.io
	nodes.cluster-api-provider-aws.sigs.k8s.io
	```
	To do that, you'll need the ARNs for each policy.  You can get them with the `list-policies` command, as in:

	```shell
	aws iam list-policies --scope Local
	```
	```console
	{
	  "Policies": [
		  {
			  "PolicyName": "controllers-eks.cluster-api-provider-aws.sigs.k8s.io",
			  "PolicyId": "ANPA22CF4NNF3VUDTMH3N",
			  "Arn": "arn:aws:iam::FAKE_ARN_123:policy/controllers-eks.cluster-api-provider-aws.sigs.k8s.io",
			  "Path": "/",
			  "DefaultVersionId": "v1",
			  "AttachmentCount": 2,
			  "PermissionsBoundaryUsageCount": 0,
			  "IsAttachable": true,
			  "CreateDate": "2025-01-01T18:47:43+00:00",
			  "UpdateDate": "2025-01-01T18:47:43+00:00"
		  },
		  {
			  "PolicyName": "nodes.cluster-api-provider-aws.sigs.k8s.io",
			  "PolicyId": "ANPA22CF4NNF5TAKL44PU",
			  "Arn": "arn:aws:iam::FAKE_ARN_123:policy/nodes.cluster-api-provider-aws.sigs.k8s.io",
			  "Path": "/",
			  "DefaultVersionId": "v1",
			  "AttachmentCount": 3,
			  "PermissionsBoundaryUsageCount": 0,
			  "IsAttachable": true,
			  "CreateDate": "2025-01-01T18:47:44+00:00",
			  "UpdateDate": "2025-01-01T18:47:44+00:00"
		  },
		  {
			  "PolicyName": "controllers.cluster-api-provider-aws.sigs.k8s.io",
			  "PolicyId": "ANPA22CF4NNFVO6OHIQOE",
			  "Arn": "arn:aws:iam::FAKE_ARN_123:policy/controllers.cluster-api-provider-aws.sigs.k8s.io",
			  "Path": "/",
			  "DefaultVersionId": "v1",
			  "AttachmentCount": 3,
			  "PermissionsBoundaryUsageCount": 0,
			  "IsAttachable": true,
			  "CreateDate": "2025-01-01T18:47:43+00:00",
			  "UpdateDate": "2025-01-01T18:47:43+00:00"
		  },
		  {
			  "PolicyName": "control-plane.cluster-api-provider-aws.sigs.k8s.io",
			  "PolicyId": "ANPA22CF4NNFY4FJ3DA2E",
			  "Arn": "arn:aws:iam::FAKE_ARN_123:policy/control-plane.cluster-api-provider-aws.sigs.k8s.io",
			  "Path": "/",
			  "DefaultVersionId": "v1",
			  "AttachmentCount": 2,
			  "PermissionsBoundaryUsageCount": 0,
			  "IsAttachable": true,
			  "CreateDate": "2025-01-01T18:47:43+00:00",
			  "UpdateDate": "2025-01-01T18:47:43+00:00"
		  }
	  ]
	}
	```

	Now you can add the policies using the `attach-user-policy` command and the ARNs you retrieved in the previous step:

	```shell
	aws iam attach-user-policy --user-name k0rdentQuickstart --policy-arn arn:aws:iam::FAKE_ARN_123:policy/controllers-eks.cluster-api-provider-aws.sigs.k8s.io
	aws iam attach-user-policy --user-name k0rdentQuickstart --policy-arn arn:aws:iam::FAKE_ARN_123:policy/controllers.cluster-api-provider-aws.sigs.k8s.io
	aws iam attach-user-policy --user-name k0rdentQuickstart --policy-arn arn:aws:iam::FAKE_ARN_123:policy/control-plane.cluster-api-provider-aws.sigs.k8s.io
    aws iam attach-user-policy --user-name k0rdentQuickstart --policy-arn arn:aws:iam::FAKE_ARN_123:policy/nodes.cluster-api-provider-aws.sigs.k8s.io
	```
	
8. Create an access key and secret

	To access AWS as this new user, you'll need to create an access key:

	```shell
	aws iam create-access-key --user-name k0rdentQuickstart 
	```
	```console
	{
	  "AccessKey": {
		  "UserName": "k0rdentQuickstart",
		  "AccessKeyId": "EXAMPLE_ACCESS_KEY_ID",
		  "Status": "Active",
		  "SecretAccessKey": "EXAMPLE_SECRET_ACCESS_KEY",
		  "CreateDate": "2025-01-18T08:33:35+00:00"
	  }
	}
	```

9. Create the IAM Credentials Secret on the k0rdent Management Cluster

	Create a YAML file called aws-cluster-identity-secret.yaml and add the following text, including the `AccessKeyId` and `SecretAccessKey` you created in the previous step:

	```yaml
	apiVersion: v1
	kind: Secret
	metadata:
	  name: aws-cluster-identity-secret
	  namespace: kcm-system
	type: Opaque
	stringData:
	  AccessKeyID: EXAMPLE_ACCESS_KEY_ID
	  SecretAccessKey: EXAMPLE_SECRET_ACCESS_KEY
	```

	Apply the YAML to your cluster, making sure to add it to the namespace where the CAPA provider is running (currently `kcm-system`) so the controller can read it:

	```shell
	kubectl apply -f aws-cluster-identity-secret.yaml -n kcm-system
	```

10. Create the `AWSClusterStaticIdentity`

	Create the `AWSClusterStaticIdentity` object in a file named `aws-cluster-identity.yaml`:

	```shell
	kind: AWSClusterStaticIdentity
	metadata:
	  name: aws-cluster-identity
	spec:
	  secretRef: aws-cluster-identity-secret
	  allowedNamespaces:
	    selector:
		  matchLabels: {}
	```

	Notice that the `secretRef` references the `Secret` you created in the previous step.

	Apply the YAML to your cluster, again adding it to the `kcm-system` namespace.

	```shell
	kubectl apply -f aws-cluster-identity.yaml  -n kcm-system
	```

11. Create the `Credential`

	Finally, create the kcm `Credential` object, making sure to reference the `AWSClusterStaticIdentity` you just created:

	```yaml
	apiVersion: k0rdent.mirantis.com/v1alpha1
	kind: Credential
	metadata:
	  name: aws-cluster-identity-cred
	  namespace: kcm-system
	spec:
	description: "Credential Example"
	identityRef:
	  apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
	  kind: AWSClusterStaticIdentity
	  name: aws-cluster-identity
	```
	Apply the YAML to your cluster, again keeping it in the `kcm-system` namespace:

	```shell
	kubectl apply -f aws-cluster-identity-cred.yaml -n kcm-system
	```

12. Deploy a cluster

	Make sure everything is configured properly by creating a `ClusterDeployment`. Start with a YAML file specifying the `ClusterDeployment`, as in:

	```yaml
	apiVersion: k0rdent.mirantis.com/v1alpha1
	kind: ClusterDeployment
	metadata:
      name: my-aws-clusterdeployment1
	  namespace: kcm-system
	spec:
	  template: aws-standalone-cp-0-0-5
	  credential: aws-cluster-identity-cred
    config:
      clusterLabels: {}
      region: us-east-2
      controlPlane:
        instanceType: t3.small
      worker:
        instanceType: t3.small
	```

  > NOTE:
  > - You're giving it an arbitrary name in `.metadata.name` (`my-aws-clusterdeployment1`)
  > - You're referencing the credential you created in the previous step, `aws-cluster-identity-cred`. This enables you to set up a system where users can take advantage of having access to the credentials to the AWS account without actually having those credentials in hand.
  > - You need to choose a template to use for the cluster, in this case `aws-standalone-cp-0-0-5`. You can get a list of available templates using:

  ```shell
  kubectl get clustertemplate -n kcm-system
  ```

	```console
	NAMESPACE    NAME                            VALID
	kcm-system   adopted-cluster-0-0-2           true
	kcm-system   aws-eks-0-0-3                   true
	kcm-system   aws-hosted-cp-0-0-4             true
	kcm-system   aws-standalone-cp-0-0-5         true
	kcm-system   azure-aks-0-0-2                 true
	kcm-system   azure-hosted-cp-0-0-4           true
	kcm-system   azure-standalone-cp-0-0-5       true
	kcm-system   openstack-standalone-cp-0-0-2   true
	kcm-system   vsphere-hosted-cp-0-0-5         true
	kcm-system   vsphere-standalone-cp-0-0-5     true
	```

	Apply the YAML to your management cluster:

	```shell
	kubectl apply -f my-aws-clusterdeployment1.yaml
	```
	```console
	clusterdeployment.k0rdent.mirantis.com/my-aws-clusterdeployment1 created
	```
	
    As before, there will be a delay as the cluster finishes provisioning. Follow the provisioning process with:

	```shell
	kubectl -n kcm-system get clusterdeployment.k0rdent.mirantis.com my-aws-clusterdeployment1 --watch
	```
	```console
	NAME                        READY   STATUS
	my-aws-clusterdeployment1   True    ClusterDeployment is ready
	```

	When the cluster is `Ready`, you can access it via the kubeconfig, as in:

	```shell
	kubectl -n kcm-system get secret my-aws-clusterdeployment1-kubeconfig -o jsonpath='{.data.value}' | base64 -d > my-aws-clusterdeployment1-kubeconfig.kubeconfig
	KUBECONFIG="my-aws-clusterdeployment1-kubeconfig.kubeconfig" kubectl get pods -A
	```

14. Cleanup

	When you've established that it's working properly. you can delete the managed cluster and its AWS objects:

	```shell
	kubectl delete ClusterDeployment my-aws-clusterdeployment1 
	```

## Azure

Standalone clusters can be deployed on Azure instances. Follow these steps to make Azure clusters available to your users:

1. Install k0rdent

    Follow the instructions in [Install k0rdent](admin-installation.md) to create a management cluster with k0rdent running.

2. The Azure CLI

    The Azure CLI (az) is required to interact with Azure resources. You can install it on Ubuntu as follows:

    ```shell
    curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
    ```

3. Log in to Azure

    Run the `az login` command to authenticate your session with Azure:

    ```shell
    az login
    ```

    Make sure that the account you're using has at least one active subscription.

4. Register resource providers

    In order for k0rdent to deploy and manage clusters, it needs to be able to work with Azure resources such as 
    compute, network, and identity. Make sure the subscription you're using has the following resource providers registered:

    ```shell
    Microsoft.Compute
    Microsoft.Network
    Microsoft.ContainerService
    Microsoft.ManagedIdentity
    Microsoft.Authorization
    ```

    To register these providers, run the following commands in the Azure CLI:

    ```shell
    az provider register --namespace Microsoft.Compute
    az provider register --namespace Microsoft.Network
    az provider register --namespace Microsoft.ContainerService
    az provider register --namespace Microsoft.ManagedIdentity
    az provider register --namespace Microsoft.Authorization
    ```

5. Find Your Subscription ID

    Creating a managed cluster requires a structure of credentials that link to user identities on the provider system without
    exposing the actual username and password to users. You can find more information on [k0rdent 
    Credentials](admin-credentials.md), but for Azure, this involves creating an `AzureClusterIdentity` and a 
    Service Principal (SP) to let CAPZ (Cluster API Azure) communicate with the cloud provider. 

    On Azure, the lowest level of this hierarchy is the subscription, which ties to your billing information Azure.Your Azure must have at least one subscription for you to use it with k0rdent, so if you're working with a new
    account make sure to [create a new subscription with billing information](https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/create-subscription) before you start.

    To get the information you need, list all your Azure subscriptions: 

    ```shell
    az account list -o table
    ```
    ```console
    Name                     SubscriptionId                        TenantId
    -----------------------  -------------------------------------  --------------------------------
    My Azure Subscription    SUBSCRIPTION_ID_SUBSCRIPTION_ID        TENANT_ID_TENANT_ID_TENANT_ID
    ```

    Make note of the `SubscriptionId` for the subscription you want to use.

6. Create a Service Principal (SP)

    The Service Principal is like a password-protected user that CAPZ will use to manage resources on Azure.
    Create the Service Principal, making sure to replace <SUBSCRIPTION_ID_SUBSCRIPTION_ID> with the `SubscriptionId` from step 1.

    ```shell
    az ad sp create-for-rbac --role contributor --scopes="/subscriptions/<SUBSCRIPTION_ID_SUBSCRIPTION_ID>"
    ```
    ```console
    {
    "appId": "SP_APP_ID_SP_APP_ID",
    "displayName": "azure-cli-2024-10-24-17-36-47",
    "password": "SP_PASSWORD_SP_PASSWORD",
    "tenant": "SP_TENANT_SP_TENANT"
    }
    ```
    Note that this information gives you access to your Azure account, so make sure to treat these strings 
    like passwords. Do not share them or check them into a repository.

7. Use the password to create a `Secret` object

    The `Secret` stores the `clientSecret` (password) from the Service Principal.
    Save the `Secret` YAML in a file called `azure-cluster-identity-secret.yaml`:

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: azure-cluster-identity-secret
      namespace: kcm-system
    stringData:
      clientSecret: <SP_PASSWORD_SP_PASSWORD> # Password retrieved from the Service Principal
    type: Opaque
    ```

    You can then apply the YAML to your cluster:

    ```shell
    kubectl apply -f azure-cluster-identity-secret.yaml
    ```

8. Create the `AzureClusterIdentity` objects

    The `AzureClusterIdentity` object defines the credentials CAPZ uses to manage Azure resources. 
    It references the `Secret` you just created, so make sure that `.spec.clientSecret.name` matches 
    the name of that `Secret`.

    Save the following YAML into a file named `azure-cluster-identity.yaml`:

    ```yaml
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AzureClusterIdentity
    metadata:
      labels:
        clusterctl.cluster.x-k8s.io/move-hierarchy: "true"
      name: azure-cluster-identity
      namespace: kcm-system
    spec:
      allowedNamespaces: {}
      clientID: <SP_APP_ID_SP_APP_ID> # The App ID retrieved from the Service Principal above in Step 2
      clientSecret:
        name: azure-cluster-identity-secret
        namespace: kcm-system
      tenantID: <SP_TENANT_SP_TENANT> # The Tenant ID retrieved from the Service Principal above in Step 2
      type: ServicePrincipal
    ```

    Apply the YAML to your cluster:

    ```shell
    kubectl apply -f azure-cluster-identity.yaml
    ```
    ```console
    azureclusteridentity.infrastructure.cluster.x-k8s.io/azure-cluster-identity created
    ```

9. Create the k0rdent `Credential` Object

    Create the YAML for the specification of the `Credential` and save it as `azure-cluster-identity-cred.yaml`.

    ```YAML
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: Credential
    metadata:
      name: azure-cluster-identity-cred
      namespace: kcm-system
    spec:
      identityRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: AzureClusterIdentity
        name: azure-cluster-identity
        namespace: kcm-system
    ```

    You're referencing the `AzureClusterIdentity` object you just created, so make sure that `.spec.name` matches 
    `.metadata.name` of that object. Also, note that while the overall object's `kind` is `Credential`, the 
    `.spec.identityRef.kind` must be `AzureClusterIdentity` to match that object.

    Apply the YAML to your cluster:

    ```shell
    kubectl apply -f azure-cluster-identity-cred.yaml
    ```
    ```console
    credential.k0rdent.mirantis.com/azure-cluster-identity-cred created
    ```

Now you're ready to deploy the cluster.

10. Create a `ClusterDeployment`

    To test the configuration, deploy a managed cluster by following these steps:

    First get a list of available locations/regions:

    ```shell
    az account list-locations -o table
    ```
    ```console
    DisplayName               Name                 RegionalDisplayName
    ------------------------  -------------------  -------------------------------------
    East US                   eastus               (US) East US
    South Central US          southcentralus       (US) South Central US
    West US 2                 westus2              (US) West US 2
    West US 3                 westus3              (US) West US 3
    Australia East            australiaeast        (Asia Pacific) Australia East
    …
    ```

    Make note of the location you want to use, such as `eastus`.

    To create the actual managed cluster, create a `ClusterDeployment` that references the appropriate template
    as well as the location, credentials, and `subscriptionId`.

    You can see the available templates by listing them:

    ```shell
    kubectl get clustertemplate -n kcm-system
    ```

    Create the yaml:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ClusterDeployment
    metadata:
      name: my-azure-clusterdeployment1
      namespace: kcm-system
    spec:
      template: azure-standalone-cp-0-0-5
      credential: azure-cluster-identity-cred
      config:
        location: "westus" # Select your desired Azure Location (find it via `az account list-locations -o table`)
        subscriptionID: <SUBSCRIPTION_ID_SUBSCRIPTION_ID> # Enter the Subscription ID used earlier
        controlPlane:
          vmSize: Standard_A4_v2
        worker:
          vmSize: Standard_A4_v2
    ```

    Apply the YAML to your management cluster:

    ```shell
    kubectl apply -f my-azure-clusterdeployment1.yaml
    ```
    ```console
    clusterdeployment.k0rdent.mirantis.com/my-azure-clusterdeployment1 created
    ```

    Note that although the `ClusterDeployment` object has been created, there will be a delay as actual Azure instances
    are provisioned and added to the cluster. You can follow the provisioning process:

    ```shell
    kubectl -n kcm-system get clusterdeployment.k0rdent.mirantis.com my-azure-clusterdeployment1 --watch
    ```

    If the provisioning process continues for a more than a few minutes, check to make sure k0rdent isn't trying to
    exceed your quotas. If you are near the top of your quotas, requesting an increase can "unstick" the provisioning process.

    After the cluster is `Ready`, you can access it via the kubeconfig:

    ```shell
    kubectl -n kcm-system get secret my-azure-clusterdeployment1-kubeconfig -o jsonpath='{.data.value}' | base64 -d > my-azure-clusterdeployment1-kubeconfig.kubeconfig
    KUBECONFIG="my-azure-clusterdeployment1-kubeconfig.kubeconfig" kubectl get pods -A
    ```

11. Cleanup

    To clean up Azure resources, delete the managed cluster by deleting the `ClusterDeployment`:

    ```shell
    kubectl get ClusterDeployments -A
    ```
    ```console
    NAMESPACE    NAME                          READY   STATUS
    kcm-system   my-azure-clusterdeployment1   True    ClusterDeployment is ready
    ```
    ```shell
    kubectl delete ClusterDeployment my-azure-clusterdeployment1 -n kcm-system
    ```
    ```console
    clusterdeployment.k0rdent.mirantis.com "my-azure-clusterdeployment1" deleted
    ```

## OpenStack

k0rdent is able to deploy managed clusters on OpenStack virtual machines. Follow these steps to make it possible to deploy to OpenStack:

1. Install k0rdent

    Follow the instructions in [Install k0rdent](admin-installation.md) to create a management cluster with k0rdent running.

2. OpenStack CLI (optional)

    If you plan to access OpenStack directly, go ahead and 
    [install the OpenStack CLI](https://docs.openstack.org/newton/user-guide/common/cli-install-openstack-command-line-clients.html).

3. Configure the OpenStack Application Credential

    This credential should include:

    ```shell
    OS_AUTH_URL
    OS_APPLICATION_CREDENTIAL_ID
    OS_APPLICATION_CREDENTIAL_SECRET
    OS_REGION_NAME
    OS_INTERFACE
    OS_IDENTITY_API_VERSION
    OS_AUTH_TYPE (e.g. v3applicationcredential)
    ```

    While it's possible to use a username and password instead of the Application Credential &mdash; adjust your YAML accordingly &mdash; an 
    Application Credential is strongly recommended because it limits scope and improves security over a raw username/password approach.

4. Create the OpenStack Credentials Secret

    Create a Kubernetes `Secret` containing the `clouds.yaml` that defines your OpenStack environment, substituting real values
    where appropriate. Save this as `openstack-cloud-config.yaml`:

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: openstack-cloud-config
      namespace: kcm-system
    stringData:
      clouds.yaml: |
        clouds:
          openstack:
            auth:
              auth_url: <OS_AUTH_URL>
              application_credential_id: <OS_APPLICATION_CREDENTIAL_ID>
              application_credential_secret: <OS_APPLICATION_CREDENTIAL_SECRET>
            region_name: <OS_REGION_NAME>
            interface: <OS_INTERFACE>
            identity_api_version: <OS_IDENTITY_API_VERSION>
            auth_type: <OS_AUTH_TYPE>
    ```

    Apply the YAML to your cluster:

    ```shell
    kubectl apply -f openstack-cloud-config.yaml
    ```

5. Create the k0rdent Credential Object

    Next, define a `Credential` that references the `Secret` from the previous step.
    Save this as `openstack-cluster-identity-cred.yaml`:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: Credential
    metadata:
      name: openstack-cluster-identity-cred
      namespace: kcm-system
    spec:
      description: "OpenStack credentials"
      identityRef:
        apiVersion: v1
        kind: Secret
        name: openstack-cloud-config
        namespace: kcm-system
    ```

    Apply the YAML to your cluster:

    ```shell
    kubectl apply -f openstack-cluster-identity-cred.yaml
    ```

    Note that `.spec.identityRef.name` must match the `Secret` you created in the previous step, and 
    `.spec.identityRef.namespace` must be the same as the `Secret`’s namespace (`kcm-system`).

6. Create Your First Managed Cluster

    To test the configuration, create a YAML file with the specification of your Managed Cluster and save it as
    `my-openstack-cluster-deployment.yaml`.  Note that you can see the available templates by listing them:

    ```shell
	kubectl get clustertemplate -n kcm-system
	```
	```console
	NAMESPACE    NAME                            VALID
	kcm-system   adopted-cluster-0-0-2           true
	kcm-system   aws-eks-0-0-3                   true
	kcm-system   aws-hosted-cp-0-0-4             true
	kcm-system   aws-standalone-cp-0-0-5         true
	kcm-system   azure-aks-0-0-2                 true
	kcm-system   azure-hosted-cp-0-0-4           true
	kcm-system   azure-standalone-cp-0-0-5       true
	kcm-system   openstack-standalone-cp-0-0-2   true
	kcm-system   vsphere-hosted-cp-0-0-5         true
	kcm-system   vsphere-standalone-cp-0-0-5     true
	```

    The `ClusterDeployment` should look something like this:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ClusterDeployment
    metadata:
      name: my-openstack-cluster-deployment
      namespace: kcm-system
    spec:
      template: openstack-standalone-cp-0-0-1
      credential: openstack-cluster-identity-cred
      config:
        controlPlaneNumber: 1
        workersNumber: 1
        controlPlane:
          flavor: m1.medium
          image:
            filter:
              name: ubuntu-22.04-x86_64
        worker:
          flavor: m1.medium
          image:
            filter:
              name: ubuntu-22.04-x86_64
        authURL: <OS_AUTH_URL>
    ```
    You can adjust `flavor`, `image` name, and `authURL` to match your OpenStack environment. For more information about the configuration options, see the [OpenStack Template Parameters Reference](template-openstack.md).

    Apply the YAML to your management cluster:

    ```shell
    kubectl apply -f my-openstack-cluster-deployment.yaml
    ```

    There will be a delay as the cluster finishes provisioning. You can follow the
    provisioning process:

    ```shell
    kubectl -n kcm-system get clusterdeployment.k0rdent.mirantis.com my-openstack-cluster-deployment --watch
    ```

    After the cluster is `Ready`, you can access it via the kubeconfig, just like any other Kubernetes cluster:

    ```shell
    kubectl -n kcm-system get secret my-openstack-cluster-deployment-kubeconfig -o jsonpath='{.data.value>' | base64 -d > my-openstack-cluster-deployment-kubeconfig.kubeconfig
    KUBECONFIG="my-openstack-cluster-deployment-kubeconfig.kubeconfig" kubectl get pods -A
    ```

7. Cleanup

    To clean up OpenStack resources, delete the managed cluster by deleting the `ClusterDeployment`:

    ```shell
    kubectl get ClusterDeployments -A
    ```
    ```console
    NAMESPACE    NAME                          READY   STATUS
    kcm-system   my-openstack-cluster-deployment   True    ClusterDeployment is ready
    ```
    ```shell
    kubectl delete ClusterDeployment my-openstack-cluster-deployment -n kcm-system
    ```
    ```console
    clusterdeployment.k0rdent.mirantis.com "my-openstack-cluster-deployment" deleted
    ```


## vSphere

To enable users to deploy managed clusers on vSphere, follow these steps:

1. Create a k0rdent management cluster

    Follow the instructions in [Install k0rdent](admin-installation.md) to create a management cluster with k0rdent running, as well as a local install of `kubectl`.

2. Install a vSphere instance version `6.7.0` or higher.

3. Create a vSphere account with appropriate privileges

    To function properly, the user assigned to the vSphere Provider should be able
    to manipulate vSphere resources. The user should have the following 
    required privileges:

    ```shell
    Virtual machine: Full permissions are required
    Network: Assign network is sufficient
    Datastore: The user should be able to manipulate virtual machine files and metadata
    ```

    In addition to that, specific CSI driver permissions are required. See
    [the official doc](https://docs.vmware.com/en/VMware-vSphere-Container-Storage-Plug-in/2.0/vmware-vsphere-csp-getting-started/GUID-0AB6E692-AA47-4B6A-8CEA-38B754E16567.html)
    for more information on CSI-specific permissions.

4. Image template

    You can use pre-built image templates from the
    [CAPV project](https://github.com/kubernetes-sigs/cluster-api-provider-vsphere/blob/main/README.md#kubernetes-versions-with-published-ovas)
    or build your own.

    When building your own image, make sure that VMware tools and cloud-init are
    installed and properly configured.

    You can follow the [official open-vm-tools guide](https://docs.vmware.com/en/VMware-Tools/11.0.0/com.vmware.vsphere.vmwaretools.doc/GUID-C48E1F14-240D-4DD1-8D4C-25B6EBE4BB0F.html)
    on how to correctly install VMware tools.

    When setting up cloud-init, you can refer to the [official docs](https://cloudinit.readthedocs.io/en/latest/index.html)
    and specifically the [VMware datasource docs](https://cloudinit.readthedocs.io/en/latest/reference/datasources/vmware.html)
    for extended information regarding cloud-init on vSphere.

5. vSphere network

    When creating a network, make sure that it has the DHCP service.

    Also, ensure that part of your network is out of the DHCP range (for example, the network
    `172.16.0.0/24` should have a DHCP range of `172.16.0.100-172.16.0.254` only) so that LoadBalancer services will not create any IP conflicts in the
    network.

6. vSphere Credentials

    To enable k0rdent to access vSphere resources, create the appropriate credentials objects. For a full explanation of how Credentials work, see the [main Credentials chapter](admin-credentials.md) but for now, follow these steps:

    Create a `Secret` Object with the username and password

    The `Secret` stores the username and password for your vSphere instance. Save the `Secret` YAML in a file named `vsphere-cluster-identity-secret.yaml`:

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: vsphere-cluster-identity-secret
      namespace: kcm-system
    stringData:
      username: <USERNAME>
      password: <PASSWORD>
    type: Opaque
    ```

    Apply the YAML to your cluster:

    ```shell
    kubectl apply -f vsphere-cluster-identity-secret.yaml
    ```

7. Create the `VSphereClusterIdentity` Object

    The `VSphereClusterIdentity` object defines the credentials CAPV will use to manage vSphere resources.

    Save the `VSphereClusterIdentity` YAML into a file named `vsphere-cluster-identity.yaml`:

    ```yaml
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: VSphereClusterIdentity
    metadata:
      name: vsphere-cluster-identity
    spec:
      secretName: vsphere-cluster-identity-secret
      allowedNamespaces:
        selector:
          matchLabels: {}
    ```

    The `VSphereClusterIdentity` object references the `Secret` you created in the previous step, so `.spec.secretName` 
    needs to match the `.metadata.name` for the `Secret`.

    Apply the YAML to your cluster:

    ```shell
    kubectl apply -f vsphere-cluster-identity.yaml
    ```

8. Create the `Credential` Object

    Create a YAML with the specification of our credential and save it as
    `vsphere-cluster-identity-cred.yaml`

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: Credential
    metadata:
      name: vsphere-cluster-identity-cred
      namespace: kcm-system
    spec:
      identityRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: VSphereClusterIdentity
        name: vsphere-cluster-identity
    ```
    Again, `.spec.identityRef.name` must match the `.metadata.name` of the `VSphereClusterIdentity` object you just created.

    Apply the YAML to your cluster:

    ```shell
    kubectl apply -f vsphere-cluster-identity-cred.yaml
    ```

9. Create your first Cluster Deployment

    Test the configuration by deploying a cluster. Create a YAML document with the specification of your Cluster Deployment and save it as `my-vsphere-clusterdeployment1.yaml`.

    You can get a list of available templates:


    ```shell
    kubectl get clustertemplate -n kcm-system
    ```
    ```console
    NAMESPACE    NAME                            VALID
    kcm-system   adopted-cluster-0-0-2           true
    kcm-system   aws-eks-0-0-3                   true
    kcm-system   aws-hosted-cp-0-0-4             true
    kcm-system   aws-standalone-cp-0-0-5         true
    kcm-system   azure-aks-0-0-2                 true
    kcm-system   azure-hosted-cp-0-0-4           true
    kcm-system   azure-standalone-cp-0-0-5       true
    kcm-system   openstack-standalone-cp-0-0-2   true
    kcm-system   vsphere-hosted-cp-0-0-5         true
    kcm-system   vsphere-standalone-cp-0-0-5     true
    ```

    The `ClusterDeployment` YAML file should look something like this. Make sure the replace the placeholders with your
    specific information:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ClusterDeployment
    metadata:
      name: my-vsphere-clusterdeployment1
      namespace: kcm-system
    spec:
      template: vsphere-standalone-cp-0-0-5
      credential: vsphere-cluster-identity-cred
      config:
        vsphere:
          server: <VSPHERE_SERVER>
          thumbprint: <VSPHERE_THUMBPRINT>
          datacenter: <VSPHERE_DATACENTER>
          datastore: <VSPHERE_DATASTORE>
          resourcePool: <VSPHERE_RESOURCEPOOL>
          folder: <VSPHERE_FOLDER>
        controlPlaneEndpointIP: <VSPHERE_CONTROL_PLANE_ENDPOINT>
        controlPlane:
          ssh:
            user: ubuntu
            publicKey: <VSPHERE_SSH_KEY>
          vmTemplate: <VSPHERE_VM_TEMPLATE>
          network: <VSPHERE_NETWORK>
        worker:
          ssh:
            user: ubuntu
            publicKey: <VSPHERE_SSH_KEY>
          vmTemplate: <VSPHERE_VM_TEMPLATE>
          network: <VSPHERE_NETWORK>
    ```

    For more information about the available configuration options, see the [vSphere Template Parameters](template-openstack.md).

    Apply the YAML to your management cluster:

    ```shell
    kubectl apply -f my-vsphere-clusterdeployment1.yaml
    ```

    There will be a delay as the cluster finishes provisioning. Follow the
    provisioning process with the following command:

    ```shell
    kubectl -n kcm-system get clusterdeployment.k0rdent.mirantis.com my-vsphere-clusterdeployment1 --watch
    ```

    After the cluster is `Ready`, you can access it via the kubeconfig, like this:

    ```shell
    kubectl -n kcm-system get secret my-vsphere-clusterdeployment1-kubeconfig -o jsonpath='{.data.value}' | base64 -d > my-vsphere-clusterdeployment1-kubeconfig.kubeconfig
    KUBECONFIG="my-vsphere-clusterdeployment1-kubeconfig.kubeconfig" kubectl get pods -A
    ```

10. Cleanup

    To delete the provisioned cluster and free consumed vSphere resources run:

    ```shell
    kubectl -n kcm-system delete cluster my-vsphere-clusterdeployment1
    ```
