# External Secret Operator
[![CircleCI](https://circleci.com/gh/ContainerSolutions/externalsecret-operator.svg?style=svg)](https://circleci.com/gh/ContainerSolutions/externalsecret-operator) [![Go Report Card](https://goreportcard.com/badge/github.com/ContainerSolutions/externalsecret-operator)](https://goreportcard.com/report/github.com/ContainerSolutions/externalsecret-operator) [![codecov](https://codecov.io/gh/ContainerSolutions/externalsecret-operator/branch/master/graph/badge.svg)](https://codecov.io/gh/ContainerSolutions/externalsecret-operator)

This operator reads information from a third party service
like [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/) or [AWS SSM](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-paramstore.html) and automatically injects the values as [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/).

## Quick start

If you want to jump right into action you can deploy the External Secrets Operator using the provided [helm chart](./deployments/helm/externalsecret-operator/README.md) or [manifests](./deploy). The following examples are specific to the AWS Secret Manager backend.

### Helm

Here's how you can deploy the External Secret Operator in the `default` namespace.

```shell
export AWS_ACCESS_KEY_ID="AKIAYOURSECRETKEYID"
export AWS_DEFAULT_REGION="eu-west-1"
export AWS_SECRET_ACCESS_KEY="OoXie5Mai6Qu3fakemeezoo4ahfoo6IHahch0rai"
helm upgrade --install asm1 --wait \
    --set operatorName="asm-example" \
    --set secret.data.Type="asm" \
    --set secret.data.Parameters.accessKeyID="$AWS_ACCESS_KEY_ID" \
    --set secret.data.Parameters.region="$AWS_DEFAULT_REGION" \
    --set secret.data.Parameters.secretAccessKey="$AWS_SECRET_ACCESS_KEY" \
    ./deployments/helm/externalsecret-operator/.
```

It will watch for `ExternalSecrets` with `Backend: asm-example` resources in the `default` namespace and it will inject a corresponding `Secret` with the value retrieved from AWS Secret Manager.

Look for more deployment options in the [README.md](./deployments/helm/externalsecret-operator/README.md) of the helm chart.

### Manifests

The `deploy` target in the Makefile will substiute variables and deploy the
manifests for you. The following command will deploy the operator in the
`default` namespace:

```shell
export AWS_ACCESS_KEY_ID="AKIAYOURSECRETKEYID"
export AWS_DEFAULT_REGION="eu-west-1"
export AWS_SECRET_ACCESS_KEY="OoXie5Mai6Qu3fakemeezoo4ahfoo6IHahch0rai"
export OPERATOR_NAME=asm-example
export BACKEND=asm
make deploy
```
It will watch for `ExternalSecrets` with `Backend: asm-example` resources in the `default` namespace and it will inject a corresponding `Secret` with the value retrieved from AWS Secret Manager.

## What does it do?

Given a secret defined in AWS Secrets Manager:

```shell
% aws secretsmanager create-secret \
  --name=example-externalsecret-key \
  --secret-string='this string is a secret'
```

and an `ExternalSecret` resource definition like this one:

```yaml
% cat ./deployments/crds/examples/externalsecret-asm.yaml
apiVersion: externalsecret-operator.container-solutions.com/v1alpha1
kind: ExternalSecret
metadata:
  name: example-externalsecret
spec:
  Key: example-externalsecret-key
  Backend: asm-example
```

The operator fetches the secret from AWS Secrets Manager and injects it as a
secret:

```shell
% kubectl apply -f ./deployments/crds/examples/externalsecret-asm.yaml
% kubectl get secret example-externalsecret \
  -o jsonpath='{.data.example-externalsecret-key}' | base64 -d
this string is a secret
```

## Secrets Backends

We would like to support as many backend as possible and it should be rather easy to write new ones. Currently supported or planned backends are:

* AWS Secrets Manager
* 1Password
* Keybase
* Git

A contributing guide is coming soon!

### 1Password

#### Prerequisites

* An existing 1Password team account.
* A 1Password account specifically for the operator. Tip: Setup an email with the `+` convention: `john.doe+operator@example.org`
* Store the _secret key_, _master password_, _email_ and _url_ of the _operator_ account in your existing 1Password account. This screenshot shows which fields should be used to store this information.
* Our naming convention for the item account is 'External Secret Operator' concatenated with name of the Kubernetes cluster for instance 'External Secret Operator minikube'. This item name is also used for development.
  
![1Password operator account](https://raw.githubusercontent.com/containersolutions/externalsecret-operator/master/assets/1password-operator-account.png)

#### Integration Test 

The integration `secrets/onepassword/backend_integration_test.go` test checks whether a secret stored in 1Password can be read via the operator.

Create a secret in 1Password as follow. Create a vault called `test vault one`. Now add a new `Login` item with name `testkey`. Set its `password` field to `testvalue`. See the screenshot below.

![1Password secret](https://raw.githubusercontent.com/containersolutions/externalsecret-operator/master/assets/1password-secret.png)

To run the integration test do the following.

1. Sign in to your _existing_ 1password

```
$ eval $(op signin)
```

2. Set the `ITEM_VAULT` and `ITEM_NAME` environment variables to select the right 1Password item that contains credentials fo your _operator_ 1Password account.

```
$ export ITEM_NAME=External Secret Operator mykubernetescluster
$ export ITEM_VAULT=myvault
```

Now load the 1Password credentials of your _operator_ account into the environment

```
$ . deployments/source-onepassword-secrets.sh
```

Run the tests including the integration test with

```
$ go test -v ./pkg/onepassword/
```

#### Operator Deployment

To deploy the operator do the following.

1. Sign in to your _existing_ 1password

```
$ eval $(op signin)
```

2. Load the 1Password credentials of your _operator_ account into the environment

```
$ source deployments/source-onepassword-secrets.sh
```

4.  Deploy the operator

```
$ make deploy-onepassword
```

## What's next

This project is just at its beginning. See
[Issues](https://github.com/ContainerSolutions/externalsecret-operator/issues)
for planned improvements and additions.
