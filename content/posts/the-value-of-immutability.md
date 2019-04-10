---
title: "The Value of Immutability"
date: 2019-04-07T10:59:02+11:00
draft: false
description: What is an immutable artefact, and what value does it provide a project?
---

So I feel like this blog post needs to happen… it's 2019, yet I've found that
there are far too many people who either a) don't see the value in immutability;
b) don't understand what immutability means; or c) feel that introducing
immutability slows the development process down. We're going to address all of
these points, and run through why developing immutable artefacts can help you
move fast without fear of breaking things.

An immutable artefact is a resource which you can deploy into an environment
which will always deploy with the exact same result, each and every time you
deploy it. The idea behind this is that you can re-deploy an artefact days,
weeks, or even years from today, with the exact same outcome. This is critically
important in a cloud based environment where dependencies can change without
much notice.

# The immutability test (Is my artefact immutable?)

Essentially, the sanity check for whether your resource is immutable is "will
this resource deploy successfully if it's completely isolated from anything
else?"

Let's take an EC2 instance, for example: if you deploy this EC2 instance from an
AMI without any network connectivity, will your instance deploy successfully? If
the answer is "yes", then congratulations, your AMI is immutable. If the answer
is "no", then it's not.

This example can extend to other resource types, but it's a nice and simple way
to represent the concept. Containers are another resource that you could apply
this example to (and should, containers by nature are meant to be independent
deployable units).

# Does immutability matter for my project?

Unless you're working on something purely experimental, then yes, immutability
does matter to you.

Right now, you might be thinking "we're building, testing, and redeploying our
resources every single day, and not seeing any issues; what's the value
proposition here?" The value proposition is that one day, this product that
you're developing may have a regression introduced to it from a third-party
dependency, or your product will go into maintenance mode, and to cater for both
of these scenarios, you want all non-configuration aspects of your application
to remain static after a release. You don't want a version bump of a third-party
dependency to break everything, or worse still, the dependency to now be
unavailable. Fixing these issues would require investment in development effort,
which the business might not be in a position to provide; not just that, but
this business-critical service will probably be offline until redevelopment has
occurred. That's not a fun situation to be in.

## Non-business critical applications

If your application is not considered to be "business critical", then you might
think that immutability doesn't apply to it, but be sure to remember that fixing
any issue which might arise will require an undetermined amount of development
time before declaring this.

# Partial Immutability

So we need to remember that the value of immutability is to reduce the risk of
something outside of your control resulting in an unprecedented change, or
breakage of your application.

While the immutability test described above is extreme, one aspect of your
artefact which should remain mutable is its configuration. In the immutability
test, you might leverage a service such as AWS Systems Manager Parameter Store,
or Amazon S3 to provide configuration to your application, which would fail the
"no network connectivity" test, _but_ in an AWS environment, this would be a
conscious decision, and you might decide that VPC networking could be considered
safe to rely upon.

The configuration for your application would not be immutable in this scenario,
but the configuration which your application depends on is within your
environment and within your control. This is perfectly acceptable for an
immutable deployment, as the risk which immutable artefacts mitigate are
dependencies which change application behaviour, not application configuration.

# Starting with immutability: Break in the build, not in production

When developing an immutable resource for an application, you want this to be
part of your continuous delivery pipeline. This should be one of the first
things which you setup for your project, and will ensure that there is less
friction later in the project to switch provisioning mechanisms. Everything is
captured in code, and can be rebuilt repeatedly based off of that code; no need
to manually image instances which might have instance-specific configuration on
them, or stale application data on them.

Another benefit of creating immutable artefacts in your CD pipeline is that if
there is an issue with a third-party dependency the build will fail, rather than
the runtime application provisioning process failing. This means that while no
new releases could be deployed for the application, the existing build of the
application will continue to run successfully, and the development teams can
take the required time to properly develop a solution to address the issue in
build.

# Testing and Building

One complaint which I've often heard around immutable artefacts, particularly
around machine images, is that it takes too long to test: "I need to bake the
image, then I need to launch the image in my IaC template, then I need to log
onto the instance to check the state of the image: it takes too long"

Well to this, I say: "there's a better way!!!"

## Machine Images

To continue with the example of immutable machine images, you should be using a
configuration management language to configure your images (allowing for
consistent execution within different tools), and then leverage the tooling
available in that ecosystem to test your code.

Let's say that you're using Chef as your configuration management tooling to
provision your machine images: you can utilise chefspec to perform unit tests
against your cookbooks, test-kitchen to stand up an independent EC2 instance for
testing against, which can use inspec to test the state of the instance after
provisioning. This means you can have test-driven infrastructure development,
and standing up the test environment is consistent for all developers. You then
use the same cookbooks in a tool such as Hashicorp Packer to bake your AMI
(after they've already been tested), and run the inspec tests again within the
bake to be sure that the artefact is as you expect it.

## Containers

For containers, you might build a container image using build stages, performing
your testing in the build stage, and then using the artefact in the final stage.
Your Dockerfile might look something like below:

```dockerfile
FROM golang:1.12 as builder
WORKDIR /workspace
COPY ./ /workspace
RUN go mod vendor

# Run tests here
RUN go test -v -cover -race

# Build
RUN CGO_ENABLED=0 GOOS=linux go build -mod=vendor -o app .

FROM alpine:latest
RUN apk --no-cache add ca-certificates
COPY --from=builder /workspace/app /app
CMD ["/app"]
```

## Application Binaries and Packages

Similar to the Docker example above application binaries and packages should be
as immutable as possible. For a compiled language, such as _Go_, this is simple,
as a project just needs to build and release the application binaries, and the
dependencies will be used at compile time to build an immutable release. This is
the nature of a compiled application binary.

But what about an interpreted language, such as Python? Well in this scenario,
your safest bet is to document all of your dependencies in a `requirements.txt`
file (`python -m pip freeze > requirements.txt`), and at "build time" pull in
your dependencies, and package them into your deployable artefact. You might do
something like below (the below code is untested and for example purposes only,
YMMV):

```sh
WORKDIR="${PWD}"

# Run your unit tests
nose2 -v

# Package your application
TMP_PACKAGE_PATH="$(mktemp -d 'tmp/pypackage.XXXXXXXX')"
python -m pip install -r requirements.txt --target "$TMP_PACKAGE_PATH"
cp app.py "${TMP_PACKAGE_PATH}/"
pushd "$TMP_PACKAGE_PATH"
zip -Xr9 "${WORKDIR}/python_package.zip" .
popd
```

The above packaging process in your build environment will ensure that all of
your dependencies are local to your deployment package, and you just have to
make sure you're running the same version of Python in each environment (which
can be defined elsewhere)

# But Security Patches

One of the "risks" which are associated with immutable artefacts are the lack of
security patches. Put succinctly, if you have a mature enough level of testing
to be confident that each and every build will be a reliable release, then you
should be able to simply schedule a recurring build and release of your
artefact, which will ensure that security patches will only be as old as the
frequency of your build.

In addition to mature testing, a team could choose to gate the scheduled release
and send a Slack notification to the team, allowing for manual testing before
the artefact gets released.

# AWS Lambda: A Hidden "Gotcha"

There's an interesting scenario which I've come by recently when using AWS
Lambda. Often teams believe that they can utilise serverless to abstract away
all management of any operating environment dependencies, but there was a new
scenario which I've seen arise: the deprecation of Lambda runtimes.

As of writing this blog post, AWS will deprecate Lambda runtimes when the
upstream language maintainer stops providing support for it. This is currently
occurring with Node.js v6.10. What this means is that the runtime will continue
to operate for existing deployments, but any new deployments will need to be
developed to leverage an alternative runtime. This could be troublesome for any
serverless applications which are now operating in maintenance mode, but may
need a patch in the future.

Chatting with a friend, we conceptualised a way to mitigate the risk of this,
but it does reduce some of the benefits of serverless: maintain the custom
runtime yourself in an immutable AWS Lambda Layer. I might write about this in
another post one day, but for now, that's just something to be aware of when
developing serverless applications.

# That's all folks

By ensuring that all of your deployable artefacts are immutable, from
application binary/package through to infrastructure release, you're ensuring
the supportability of the application for years to come. Using the tools
natively available in your ecosystem, you can get fast feedback when developing
in your stack, and with a mature enough level of testing, you can continue
introducing security patches for the application, while producing a reliable
release and deploying it through to production.

Feel free to message me [on twitter](https://twitter.com/nathan_dines) if you've
got any thoughts on this.
