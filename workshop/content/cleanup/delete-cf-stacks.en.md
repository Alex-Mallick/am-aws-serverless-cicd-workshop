+++
title = "Delete CFN Stacks"
date =  2020-07-06T18:10:22-04:00
weight = 6
+++

All of the resources deployed in this workshop are via CloudFormation. Run the following script
which will delete all of the CloudFormation stacks.

```bash
curl -s https://cicd.serverlessworkshops.io/assets/cleanup-cicd.sh | bash
```

You may also manually [download the script](/assets/cleanup-cicd.sh) and run it on your Cloud9 instance or local
computer. Note that this is a `bash` script and requires a terminal. This should work without
modification when using Cloud9. If you are using your own computer and personal AWS account you may
need to make adjustments. The cleanup script in its entirety is shown below.

```bash
#!/bin/bash

ROLE_NAME=Cloud9-AwesomeAdmin

# Function to get the status of a CFN template
get_stack_status () {
    status=$(aws cloudformation describe-stacks --stack-name $1)
    if [ $? -eq 0 ]; then
        echo "$status" | jq -r ".Stacks[].StackStatus"
    else
        echo "DELETE_COMPLETE"
    fi
}

cat > cfn-policy.json<< EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudformation.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF

ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)

## Create a temporary Admin role
aws iam create-role --role-name "$ROLE_NAME" --assume-role-policy-document file://cfn-policy.json
aws iam attach-role-policy --role-name "$ROLE_NAME" --policy-arn "arn:aws:iam::aws:policy/AdministratorAccess"
rm cfn-policy.json

## Wait a bit for this role to stick. It cannot be used immediately after creation.
sleep 10

## Delete stacks
stacks=(sam-app-prod sam-app-dev sam-app-pipeline aws-sam-cli-managed-prod-pipeline-resources aws-sam-cli-managed-dev-pipeline-resources)

for stack_name in "${stacks[@]}"
do
    echo "Deleting stack: $stack_name..."

    aws cloudformation delete-stack \
        --role-arn "arn:aws:iam::$ACCOUNT_ID:role/$ROLE_NAME" \
        --stack-name "$stack_name"

    status="$(get_stack_status $stack_name)"

    # Wait until the stack is actually deleted
    while [ "$status" != "DELETE_COMPLETE" ]
    do
        sleep 5
        status="$(get_stack_status $stack_name)"
        echo "$stack_name: $status"
    done

done

## Delete the temporary Admin role
aws iam detach-role-policy --role-name "$ROLE_NAME" --policy-arn "arn:aws:iam::aws:policy/AdministratorAccess"
aws iam delete-role --role-name "$ROLE_NAME"
```
