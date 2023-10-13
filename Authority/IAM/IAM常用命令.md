---
title: IAM 常用命令
categories: iam
tags: 
    - iam policy
---
IAM常用命令:
```bash
# 查看身份
aws sts get-caller-identity

# 过滤策略
policyArn=$(aws iam list-policies --output text --query 'Policies[?PolicyName == `S3-Delete-Bucket-Policy`].Arn')

# 查看策略
aws iam get-policy-version --policy-arn $policyArn --version-id v1

# 给role绑定策略
aws iam attach-role-policy --policy-arn $policyArn --role-name notes-application-role

# 查看role的策略
aws iam list-attached-role-policies --role-name notes-application-role

# 给iam绑定策略
aws iam attach-role-policy --policy-arn $policyArn --user-name notes-application-role
```