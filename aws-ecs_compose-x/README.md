
# Conduktor Platform to AWS with ECS Compose-X

Deploy Conduktor platform to AWS ECS in a few commands.

## Pre-requisites

You will need to have an AWS Account with working credentials in order to deploy the application to AWS.
See [Install AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) to get yourself
started.

If you plan on using MSK, the cluster deployment is not at this time covered in this guide. For simplicity, we will
first test with a confluent kafka cluster that you can create for 0$ for this demo.

Once you have access to confluent cloud, create a set of credentials to connect to the cluster.
For improved security, we recommend to create a new service account and grant it the ClusterAdmin role,
or lower if need be. You can evolve these in the future.

We want to accent that if you use your admin credentials, make sure not to publish it in configuration or otherwise.

## Prepare your AWS environment.

Depending on the Kafka cluster vendor, follow the below steps to prepare your environment. Once you have obtained these
credentials, we configure our AWS environment.

### Conduktor platform secrets

First of all, we create 2 secrets in AWS to store 1. the license 2. the admin user email / password.
Optionally you can also configure your SSO client secrets. Otherwise, leave these values to `none`.

```shell
aws cloudformation deploy --stack-name conduktor-platform-secrets --template common/conduktor-admin.template \
  --parameter-overrides \
  OrganizationName=<org_name> \
  ConduktorLicense=<license_key> \
  PlatformAdminEmail=<admin_email> \
  PlatformAdminPassword=<admin_password>
```

Note that the value for license and password will not be visible anywhere but in the AWS Secret.

## Deployments to AWS

* [Deploy to AWS Fargate to public subnet](./aws-fargate/README.md)


## Kafka cloud vendors configuration

From that point onwards, configuration is down to the Kafka cloud vendor that you use to configure the rest.

### SASL Credentials secret for Confluent

```shell
aws cloudformation deploy --stack-name conduktor-kafka-creds --template common/confluent_credentials.template \
  --parameter-overrides \
    ClusterId=<confluent_cluster_id> \
    BootstrapEndpoint=<endpoint> \
    BootstrapPort=9092 \
    ConsumerGroupName=conduktor-platform \ # Or alternative name
    ConsumerGroupUsername=<api_key_id> \
    ConsumerGroupPassword=<api_key_secret> \
    SchemaRegistryUrl=<sr_url> \
    SchemaRegistryGroupUsername=<sr_api_key_id> \
    SchemaRegistryGroupPassword=<sr_api_key_secret>
```

#### Prepare confluent service account

```shell
# Create service account
confluent iam service-account create conduktor --description "Conduktor platform"

# Create API key for your cluster
confluent kafka cluster list
confluent kafka cluster describe # Note cluster ID, bootstrap endpoint and so on
confluent schema-registry cluster describe # Note SR URL & ID

# Kafka cluster SASL Credentials
confluent api-key create --service-account <sa-xxxx> --resource <cluster_id>
# Schema Registry API Key
confluent api-key create --service-account <sa-xxxx> --resource <schema_registry_id>
```

As for ACLs, you have the option to either configure permissions using Confluent's own RBAC or Kafka native ACLs.
To manage everything in that cluster with Confluent RBAC, make the service account a `ClusterAdmin`

With Kafka ACLs, you can do

```shell
confluent kafka acl create --service-account <sa-xxxx> --allow --read --topics '*'
confluent kafka acl create --service-account <sa-xxxx> --allow --read --consumer-groups '*'
```

Further ACLs can be set, depending on the level of access you are comfortable granting the Conduktor platform user/service-account.
