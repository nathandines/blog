---
title: "Handling Secrets the Right Way"
date: 2019-03-29T08:28:28+11:00
draft: false
description: Exploring responsible secrets handling, and some limitations of alternative approaches
---

One thing which I often get asked by clients is "How do you handle secrets
properly in an environment?"

That question in itself is not as straightforward as it may seem on face value.
There are many factors which would influence how you manage secrets for your
application, not the least of which is the application architecture itself.

We're going to look primarily at handling secrets on persistent cloud
infrastructure here (i.e. not serverless), and there are a few different
scenarios which I'm going to run through as to how you'd access and manage
secrets securely. Let's look at the good, the bad, and the ugly options.

**Note:** These approaches assume usage of Git as the VCS, and AWS as the cloud
provider using CloudFormation for provisioning, but the approaches are fairly
transferrable to other technologies.

# The Good

Ah yes, this is how you manage secrets properly. These approaches don't require
configuration changes for the application to retrieve secrets, and two of these
will handle credential rotation automatically (application-integrated, or
templated).

![milhouse](/img/milhouse-winning.jpg)

## Have the application integrated directly with the secret management service

This is the ideal state for accessing secrets in any system, and it's very
applicable to ephemeral infrastructure in the cloud. This approach has your
application hitting the secret management service directly and caching the
secret in-memory for a pre-defined period of time (1 hour can be appropriate for
a production application, but your mileage may vary), before refreshing the
credential again.

This has numerous benefits:

1. The secret is stored in the environment, rather than anywhere in the codebase
   or deployment platform. This fits nicely in with [point
   3](https://12factor.net/config) of 12 factor app principles
1. Credential rotation is automatically occurring as frequently as your refresh
   timeout
1. Handling of the secret is independent of the application deployment
1. This can facilitate seperation of duties, a trusted individual could
   provision the secret into the secret management platform (or even better,
   have it dynamically generated for the relied upon service), and the client
   application can read the secret into memory, all without any manual
   client-side handling of the secret.

To facilitate faster development of your application, you might develop multiple
secret interfaces within your application, so on developer machines, they might
use an environment variable, whereas in production, they would use a secret
management service.

I've developed an example for an application accessing a secret which can be
found [in my GitHub
account](https://github.com/nathandines/aws-secrets-example-go/tree/1fd682fb74527874629d31b009766ce1f8d61754).

In this example, I've pre-provisioned a secret into AWS SSM Parameter Store, and
then upon starting this application, it retrieves the secret from the Parameter
Store.

In this simple example, it just prints the secret on the terminal at one second
intervals, but you could imagine that this may be database or third-party access
in a non-demo application. In this demo, the secret then refreshes after it's
aged for at least 10 seconds, but you would have this be a larger value in a
production application.

![Go Demo App Gif](/img/aws-secrets-go-demo.gif)

While the program is executing and the secret is cached, I change the credential
in AWS SSM Parameter Store, and as you can see, upon next refresh it retrieves
the new secret for use. Combine this with a clean process to rotate secrets...

1. Create secondary secret in relied upon service
1. Replace original secret in parameter store with the secondary secret
1. Wait _cache period * 1.1_ to allow the new secret to be picked up by the
   application
1. Disable the original secret from relied upon service

...and you've got a well-defined and zero downtime secret management
process; automate this and you never have to think about your secrets again!

## Use a templating tool to wrap the config and manage the service state

This approach can be good as a halfway between native application integration,
and statically persisting the secret to the host. If you're not in the position
to develop the integration of the application with a secrets management service,
this can help facilitate dynamic secret access by templating the secrets from
secret providers into the local application configuration.

To achieve this, there are tools such as
[confd](https://github.com/kelseyhightower/confd) or
[consul-template](https://github.com/hashicorp/consul-template) which can be
configured as services on your instances. Either of these would use a template
as the basis of your application configuration, populate it with values from
your secret provider, and handle the reload of your application service to
utilise the new configuration.

This one can be a bit fiddly, and does depend on how your application handles
state and reloads, but can help increase the maturity of the application
security and credential rotation capabilities overall.

## Use the secret management service APIs and store the secret on the host

If you have an instance which depends on a fairly static value for its secret
configuration, one way which this secret can be provided while minimising
exposure, is having the instance access the secret directly (rather than perhaps
passing the secret in through deployment tooling).

To access the secret directly, the instance would need to have permission to
access the secret. Using an IAM role with SSM parameter store fits perfectly for
this scenario. Here is an example IAM policy which will allow the instance to
access a parameter containing its secret:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ssm:GetParameter"],
      "Resource": [
        "arn:aws:ssm:ap-southeast-2:123456789012:parameter/app/some-secret"
      ]
    }
  ]
}
```

And then in conjunction with that IAM policy (which should be applied to the IAM
role which is applied to the instance profile which is applied to the EC2
instance), the following could be executed in _User Data_ in order to access the
secret and store it on the host:

```sh
secret_value="$(aws ssm get-parameter --with-decryption \
  --name 'arn:aws:ssm:ap-southeast-2:123456789012:parameter/app/some-secret' \
  --output text --query 'Parameter.Value')"

# Store secret in config file
printf 'secret_key = "%s"\n' "$secret_value" >> '/opt/app/etc/config.toml'

# or save secret as an environment variable to be used when executing an
# application
# Usage: `. '/opt/app/environment' && /opt/app/bin/app`
printf 'export SECRET_KEY="%s"\n' "$secret_value" >> '/opt/app/environment'
```

# The Bad

These mechanisms are not great, but can be used if you're in need of a temporary
fix. Be warned, if you implement one of these, you should add these to your tech
debt backlog for remediation as soon as possible.

![okay](/img/its-okay-i-guess.jpg)

## Pass in the secret as a parameter to your CloudFormation template

This resolves the issue of secrets being stored in version control 🎉. That's a
win, right?

Passing secrets into a CloudFormation template as a parameter permits you to
consistently use the same template, and keep the secrets out of the code. The
source of the parameter might come from AWS SSM Parameter Store, or an
environment variable stored in your CI platform.

This parameter should be passed in with the `NoEcho` flag set to true,
preventing visibility of the secret in CloudFormation. Unfortunately, this
mechanism doesn't restrict visibility below the CloudFormation level of the
stack. If passing the secret through to `AWS::CloudFormation::Init` or EC2 User
Data, for example, the secret is visible in the Launch Configuration and EC2
User Data of the instances.

Additionally, rotation of this secret requires a redeployment of the
CloudFormation templates, and this will likely need to redeploy your compute
infrastructure unless you use something like `cfn-hup` (which I feel is a bad
idea, but I'll leave that discussion for another day). Redeploying your compute
infrastructure could introduce instability or inconsistencies in your
application (depending on the app).

## Manually provision the secret onto your instances

Well, pretty much the only use case for this is if you have long-lived,
non-ephemeral instances in your environment, which don't really handle change
very well (I'm looking at you "lift-n-shift"). While this mechanism is far from
preferred, the secret should only reside on the instance which it is provisioned
on, which should only be accessed by authorised users (which should have a
narrower audit trail).

I've included this in "the bad", rather than "the ugly", as there are some
reasonable use cases for this, particularly applying a secret to a legacy
application which needs to continue living out its days in the cloud.
Refactoring these applications in order to access a secret just may not be worth
the investment to the organisation (the application might have a target date for
decommission), and the infrastructure isn't ephemeral.

The primary limitation of this is the requirement for manual rotation of the
secret, which exposes it to an administrator working on the host.

# The Ugly

These are things you should never do when managing secrets. Just, never. I'm not
going to spend too much time on these. There's no good reason to do any of these
in a production application.

![ugly-dog](/img/ugly-dog.jpg)

## Hardcode the secret in your CloudFormation templates and store it in Git

Why this is a bad idea:

1. It exposes your secret in multiple areas. Git, CloudFormation, Launch
   Configurations, EC2 User Data. There is no explicit audit trail for access to
   the secrets in these resources (exposure of a secret could be a result of
   anything accessing these services).
1. When using Git, the secret cannot be removed from history without rewriting
   all of the commit history post-adding the secret. Rotation is required once
   added here.
1. Additionally, when using Git, a copy of this secret is located on every
   machine that the git repo is cloned to (yes, every developer machine working
   on this repo). If any of these machines are comprimised, your system is
   comprimised.
1. A complete redeployment of the infrastructure and application is required to
   rotate the secret. This might not be a quick or straightforward process,
   depending on your application.

## Hardcode the secret in your application source code

Why this is a bad idea:

1. See points 2 and 3 above 👆
1. Not only does rotation of the secret require a complete redeploy of the
   application, but it also requires a complete rebuild and release! Not great,
   and brings with it a risk of introducing potentially breaking changes to the
   application.

## Bake the secret into your machine images (e.g. AMIs)

Essentially, this is a combination of the above two approaches. Changing a
secret would require a rebuild of the AMI (which may have the secret defined in
the code, if it's not coming from the deployment environment), and a redeploy of
the infrastructure. Not trivial, and comes with a whole heap of risk.

# In closing...

As with anything, there are many ways to access secrets for your application.
The most important things to consider when it comes to accessing and managing
secrets are:

1. Minimise the scope within which the secret can be accessed
1. Ease the process of secret rotation on both the server and client side
1. Minimise/eliminate the need for humans to interface directly with secrets

I hope this post has provided some insights into how secrets can be handled by
your applications, and that you've gained some insight as to the strengths and
weaknesses of different approaches.

Wherever possible, integrate your application directly with the secret
management platform using clearly defined and interchangable interfaces!
