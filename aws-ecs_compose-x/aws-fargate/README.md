
# Deploy to AWS ECS on Fargate.

## Prepare your machine

First of all, prepare your environment and install ECS Compose-X.

```shell
pip install --user ecs-composex
```

Alternatively, in a python sandbox/virtual environment

```shell
pip install pip -U
python -m venv aws-ecs_compose-x
source aws-ecs_compose-x/bin/activate
pip install pip -U; pip install ecs-aws-ecs_compose-x
```

If your account is new, or you never used ECS, run the following command.

```shell
ecs-compose-x init
```

This will create a new S3 bucket to store templates into, and configure ECS to enable essential features in your account.
All of this is free.

## Deploy to AWS publicly addressed with NGINX for HTTPs.

This deployment puts the Conduktor platform containers in a public subnet with a public IP address to be reached directly
from anywhere. To improve security, we add NGINX to enable TLS, with a self-signed certificate.

As to why bother with SSL? If you are planning on testing against your real workload, you don't want the content of
your messages non-encrypted transiting over the internet.

Assuming you went through the [Initial Setup](../README.md) and have your Conduktor admin secrets, along with Confluent cluster
details, all we now have to do to deploy is to run.

Set the `ORG_NAME` environment variable to the same value used when creating the license.

```shell
# Set the organization name
export ORG_NAME=<your_org_name>

# Render the templates, ensuring there is no typo, and configuration works.
ecs-compose-x render -d templates -p conduktor-platform -f aws-fargate/docker-compose.yaml -f aws-fargate/public.yaml

# Deploy, simply changing `render` to `up`
ecs-compose-x render -d templates -p conduktor-platform -f aws-fargate/docker-compose.yaml -f aws-fargate/public.yaml
```

To access the conduktor platform, identify the ECS Service task public IP address, and head to `https://<IP>/`


## Deploy to AWS behind Load Balancer.

Using the same overall command as above, but using a different override compose file, we deploy the conduktor platform
in a private subnet, and expose it on the internet via an Application Load Balancer which will terminate the SSL connection
for us.

The pre-requisite to this working is to have an already defined AWS Route53 hosted zone with your domain name that we
can use in order to access the service and generate our ACM certificate.

```shell
# First, export your domain name into an environment variable
export DomainName=my-route53-domain.net

# Set the organization name
export ORG_NAME=<your_org_name>

# Render the templates, ensuring there is no typo
ecs-compose-x render -d templates -p conduktor-platform -f aws-fargate/docker-compose.yaml -f aws-fargate/public-alb.yaml

# Deploy, simply changing `render` to `up`
ecs-compose-x render -d templates -p conduktor-platform -f aws-fargate/docker-compose.yaml -f aws-fargate/public-alb.yaml

```

Once complete, you can now access your conduktor-platform at `https://conduktor.${DomainName}`
