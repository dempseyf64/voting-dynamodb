# Deploy with CloudFormation (CLI)

This document explains how to launch the voting app with **AWS CloudFormation** and the **AWS CLI**. One stack creates the Polls table and the instance. Deleting the stack deletes both.


## Prerequisites

You need the **AWS CLI** installed and configured with current AWS Academy Learner Lab credentials in the `[default]` profile.


## One-Time Setup

The file `deploy/userdata.sh` is used in the deployment process, and you must change one line before you deploy.

* Open `deploy/userdata.sh` in Cursor or `nano`.
* Near the top of the file you will find the line:

  ```
  REPO_URL="https://github.com/YOUR_GITHUB_USERNAME/voting-dynamodb.git"
  ```

* Change `YOUR_GITHUB_USERNAME` to your GitHub username.
* Commit this change to the git repo, and push it back to your GitHub account.

  ```
  git add deploy/userdata.sh
  git commit -m "set github account"
  git push origin main
  ```


If the `git push` command fails, check the url of `origin` and make sure it points at your fork of the repo.

```
git remote -v
```


## Create the stack

`deploy/cloudformation.yaml` creates:

* A DynamoDB table named `Polls` with partition key `pollId`
* A security group that allows SSH, HTTP, and HTTPS
* One EC2 instance with the `LabInstanceProfile` instance profile


The instance is created after the table is active. From the root of your clone:

```bash
aws cloudformation create-stack \
  --stack-name voting-dynamodb \
  --template-body file://deploy/cloudformation.yaml \
  --parameters ParameterKey=GitHubUsername,ParameterValue=dempseyf64
```

Replace `YOUR_GITHUB_USERNAME` with your GitHub username.

Wait until the stack finishes creating:

```bash
aws cloudformation wait stack-create-complete \
  --stack-name voting-dynamodb
```

User data installs the app and starts Gunicorn. Gunicorn calls Amazon DynamoDB with `LabInstanceProfile`. It does not use the `local` profile.


## After the instance stack is complete

Read the public IP from the stack output:

```bash
aws cloudformation describe-stacks \
  --stack-name voting-dynamodb \
  --query 'Stacks[0].Outputs' \
  --output table
```

**CREATE_COMPLETE does not mean the app is ready.** User data still installs packages, clones the repo, and starts Gunicorn. Wait a few minutes, then:

```bash
curl -s http://PUBLIC_IP/health
```

A 200 means Gunicorn can describe the `Polls` table. SSH with your Academy key and check `/var/log/cloud-init-output.log` if something failed.

If you use an Elastic IP for the semester, **associate it** with this instance in the EC2 console (or with the CLI) after the instance exists, then use that address instead of the ephemeral public IP.


## Delete the stack

When you are done:

```bash
aws cloudformation delete-stack --stack-name voting-dynamodb
aws cloudformation wait stack-delete-complete --stack-name voting-dynamodb
```

Deleting this stack removes the instance, the security group, and the `Polls` table, including the polls stored in it. It does not release an Elastic IP you associated by hand.
