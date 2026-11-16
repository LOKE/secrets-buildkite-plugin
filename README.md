# secrets-buildkite-plugin

A plugin the uses buildkite oicd tokens to fetch secrets from ssm parameter
store and inject them into the build environment.

## Examples

Pull secrets stored under the pipeline's convention path:

```yaml
- command: do_thing
  plugins:
    - LOKE/secrets#v0.1.0:
        role-arn: arn:aws:iam::<aws-account-id>:role/<iam-role>
        vars: [DB_PASSWORD, API_TOKEN]
```

Each name in `vars` is read from
`/buildkite-secrets/<pipeline-slug>/env/<NAME>` (the pipeline slug comes from
`BUILDKITE_PIPELINE_SLUG`) and exported under that name. For the pipeline
`tidy-api`, `DB_PASSWORD` comes from `/buildkite-secrets/tidy-api/env/DB_PASSWORD`.
If any listed parameter doesn't exist the plugin fails and names the missing
parameters.

Pull secrets from any parameter with your own role:

```yaml
- command: do_thing
  plugins:
    - LOKE/secrets#v0.1.0:
        role-arn: arn:aws:iam::<aws-account-id>:role/<iam-role>
        env:
          SECRET: ssm-param-store-id
```

## IAM

The plugin assumes the role with `sts:AssumeRoleWithWebIdentity`, so the
account needs an IAM OIDC identity provider for `https://agent.buildkite.com`
with the audience `sts.amazonaws.com`.

### Your own role

A role used with `role-arn` and `env` needs a trust policy like this, and
`ssm:GetParameter` on the parameters it reads:

```json
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

### A role for `vars`

When `vars` is set, the plugin requests the OIDC token with the
`organization_slug` and `pipeline_slug` session tags (buildkite-agent v3.83.0 or
later). One role can then serve every pipeline in the Buildkite organization:
its trust policy allows `sts:TagSession`, so the job's `pipeline_slug` becomes
a principal tag:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<aws-account-id>:oidc-provider/agent.buildkite.com"
            },
            "Action": [
                "sts:AssumeRoleWithWebIdentity",
                "sts:TagSession"
            ],
            "Condition": {
                "StringEquals": {
                    "agent.buildkite.com:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "agent.buildkite.com:sub": "organization:<org-slug>:*"
                }
            }
        }
    ]
}
```

Its permissions policy limits each job to its own pipeline's parameters:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "ssm:GetParameters",
            "Resource": "arn:aws:ssm:*:*:parameter/buildkite-secrets/${aws:PrincipalTag/pipeline_slug}/env/*"
        }
    ]
}
```

`SecureString` parameters encrypted with the default `aws/ssm` key need no
extra permission. For a customer managed KMS key, also allow `kms:Decrypt` on
that key.
