# backoffice-email-infra

CloudFormation infrastructure for backoffice email open tracking.

## Stack contents

[cloudformation/email-tracking.yaml](cloudformation/email-tracking.yaml):

- `Domain` parameter → subscription endpoint `https://${Domain}/emailopen/`
- SNS topic `backoffice-email-open` + topic policy allowing `ses.amazonaws.com` to publish, scoped to this account
- SES config set `backoffice-email-tracking` with `open` event destination → SNS (with `DependsOn` on the topic policy so SES's publish check passes during create)
- HTTPS subscription wired to an SQS DLQ (`backoffice-email-open-dlq`, 14-day retention) via `RedrivePolicy`; DLQ queue policy allows SNS to redrive
- CloudWatch alarms on `NumberOfNotificationsFailed` and `NumberOfNotificationsFilteredOut-InvalidAttributes`
- Outputs expose `ConfigurationSetName` (with the `MAILER_DSN` reminder in its description), topic ARN, DLQ URL/ARN

## Deploy

```
aws cloudformation deploy --region us-west-2 \
  --stack-name backoffice-email-tracking \
  --template-file cloudformation/email-tracking.yaml \
  --parameter-overrides Domain=your.host.example
```

On first deploy the HTTPS subscription stays `PendingConfirmation` until your webhook forwards the `SubscribeURL` email and an admin clicks it.

## Application config

After the stack is live, append `&config_set_name=backoffice-email-tracking` to `MAILER_DSN` so outbound SES sends are tagged with the configuration set and SES injects the tracking pixel.
