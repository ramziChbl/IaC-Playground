# IaC-Playground

The following setup deploys the following instances:
1. orchestrator: Configuration management server and a bastion. It has a public IP
2. k8s-master: Where K8s will be installed, doesn't have an public gateway but still needs an Internet Gateway for internet access 

Spin up the infra:
```
cd cloudformation
export AWS_PROFILE=sysops-admin
aws cloudformation deploy --stack-name common --template-file common.yaml --parameter-overrides file://vars.json
aws cloudformation deploy --stack-name management --template-file management.yaml --parameter-overrides file://vars.json
aws cloudformation deploy --stack-name k8s --template-file k8s.yaml --parameter-overrides file://vars.json
aws cloudformation deploy --stack-name vpc-peering --template-file vpc-peering.yaml --parameter-overrides file://vars.json
```

Tear it down:
```
aws cloudformation delete-stack --stack-name vpc-peering
aws cloudformation delete-stack --stack-name k8s
aws cloudformation delete-stack --stack-name management
aws cloudformation delete-stack --stack-name common
```

## IP Blocks and Routes

| VPC        | Subnet              | CIDR        | AZ         |
|------------|---------------------|-------------|------------|
| management | mgmt-default-subnet | 10.0.0.0/16 | us-east-1b |
| k8s        | k8s-az1             | 10.1.0.0/16 | us-east-1a |


| Route Table | Destination | Target                        |
|-------------|-------------|-------------------------------|
| mgmt-rt     | 10.0.0.0/16 | Local                         |
|             | 10.1.0.0/16 | !Ref MgmtK8sPeeringConnection |
|             | 0.0.0.0/0   | !Ref MgmtInternetGateway      |
| k8s-rt      | 10.1.0.0/16 | Local                         |
|             | 10.0.0.0/16 | !Ref MgmtK8sPeeringConnection |
|             | 0.0.0.0/0   | !Ref K8sInternetGateway       |
