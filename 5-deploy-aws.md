# Guide


## Deploy the aws example

This example application will read the servcie account JWT token and use it to
authenticate with Vault. It will then log the token (don't log secrets in a real
application!) and keep the token renewed. Additionally, it will make a request
to the [Vault AWS Secret Backend][vault-aws-secret-backend] dynamically get a
set of IAM credentials. The following setup requires an AWS account and all
operations are performed on the us-east-1 region. It also assumes that you have
your AWS CLI properly configured, with enough permissions to perform actions
against IAM to create a [Vault root user][vault-aws-root-creds].

## Setup the AWS account

In this step, we wil create a programatic user with enough permissions manage
IAM resources. We will be using the IAMFullAccess policy here, but a more locked
down custom policy can be provided. For an example on such policy template,
refer to the Vault documentation regarding the backend's
[root credentials][vault-aws-root-creds].

Create the user:

```sh
$ aws iam create-user \
    --user-name vault-root
```

Attach the IAMFullAccess policy to the user:

```sh
$ aws iam attach-user-policy \
    --user-name vault-root \
    --policy-arn arn:aws:iam::aws:policy/IAMFullAccess
```

Generate a access key and secret access key pair:

```sh
$ aws iam create-access-key \
    --user-name vault-root
```

## Setup Vault

Read the `app1-policy` policy to verify that the role has read access to `aws/creds/readonly`:

```sh
$ vault policies app1-policy
path "secret/creds" {
  capabilities = ["read"]
}

path "aws/creds/readonly" {
  capabilities = ["read"]
}
```

Mount the vault secret backend:

```sh
$ vault mount aws
Successfully mounted 'aws' at 'aws'!
```

Configure the aws secret backend:

```sh
$ vault write aws/config/root \
    access_key=<vault-root-access-key> \
    secret_key=<vault-root-secret-key> \
    region=us-east-1
Success! Data written to: aws/config/root
```

Create a aws secret backend role that has read-only permissions on EC2 instances
for the account using a AWS managed policy this policy:

```sh
$ vault write aws/roles/readonly \
    arn=arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess
Success! Data written to: aws/roles/readonly
```

The vault client on the example application will be using this role to request
IAM credentials during runtime.

### Build the aws example

The easiest way to build the container is to connect your local docker agent
to the remote one in kubernetes. With minikube this can be done with:

```
eval $(minikube docker-env)
```

Then we can build the container:
```
cd basic/
./build_container
```

### Run the aws example

The `VAULT_ADDR` variable in the deployment file should be updated to your vault
server address

Now we can run the application:

```
kubectl create -f deployment.yaml
```

### View the logs

```sh
kubectl logs -f $(kubectl \
    get pods -l app=aws-example \
    -o jsonpath='{.items[0].metadata.name}')
2017/11/21 22:39:10 ==> WARNING: Don't ever write secrets to logs.
2017/11/21 22:39:10 ==>          This is for demonstration only.
2017/11/21 22:39:10 Vault token: e8052101-1e9c-ef0d-d91a-be18192a37c6
2017/11/21 22:39:41 AWS Access Key: AKIAIVUVHJF3REXAMPLE
2017/11/21 22:39:41 AWS Secret Key: mkQZa9SW0c0tWn5F297FCGpPUSXYsZqvrEXAMPLE
2017/11/21 22:39:41 ==> Listing EC2 clients using generated credentials
2017/11/21 22:39:41 i-050168cbc1c0dd111
2017/11/21 22:39:41 Starting renewal loop
```

Then you should see a token renewal approximately every 20s.

### Cleanup 

When done delete the deployment and go back to the parent directory:

```
kubectl delete -f deployment.yaml
cd ..
```

[vault-aws-secret-backend]: https://www.vaultproject.io/docs/secrets/aws/index.html
[vault-aws-root-creds]: https://www.vaultproject.io/docs/secrets/aws/index.html#root-credentials-for-dynamic-iam-users

## Next Steps

Read more about the kubernetes auth backend!