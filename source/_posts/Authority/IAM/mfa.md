---
title: mfa 失效
categories: iam
tags: 
    - mfa
---
经常会遇到mfa失效的时候，
尝试同步mfa

```bash
aws iam resync-mfa-device --user-name kail --serial-number arn:aws-cn:iam::208892152133:mfa/kail --authentication-code1 653006 --authentication-code2 742895
```

如果同步mfa无效，那么使用CLI删除重新绑定
```bash
aws iam deactivate-mfa-device --user-name kail --serial-number arn:aws-cn:iam::208892152133:mfa/kail
```
