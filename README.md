# IaC-Playground

Spin up the infra:
```
cd cloudformation
export AWS_PROFILE=sysops-admin
aws cloudformation deploy --stack-name common --template-file common.yaml --parameter-overrides file://vars.json
aws cloudformation deploy --stack-name management --template-file management.yaml --parameter-overrides file://vars.json
```

Tear it down:
```
aws cloudformation delete-stack --stack-name k8s
aws cloudformation delete-stack --stack-name management
aws cloudformation delete-stack --stack-name common
```