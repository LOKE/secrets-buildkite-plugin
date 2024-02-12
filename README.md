# secrets-buildkite-plugin

A plugin the uses buildkite oicd tokens to fetch secrets from ssm parameter
store and inject them into the build environment.

Example

```yaml
- command: do_thing
  plugins:
    - LOKE/secrets#v0.0.2:
        role-arn: arn:aws:iam::<aws-account-id>:role/<iam-role>
        env:
          SECRET: ssm-param-store-id
```

You need an IAM role with trust of the oicd token, the trust policy should be
something like

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<aws-account-id>:oidc-provider/agent.buildkite.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "agent.buildkite.com:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "agent.buildkite.com:sub": [
                        "organization:<org-slug>:pipeline:<pipeline-slug>:*"
                    ]
                }
            }
        }
    ]
}
```
