# Using AWS App Mesh with Fargate

## Overview

I gave a walkthrough in my [previous article] on how to deploy a simple microservice application to ECS and configure [AWS App Mesh] to provide traffic control and observability for it. In this article, we are going to start to explore what it means when we say that App Mesh is a service mesh that lets you control and monitor services spanning multiple AWS compute environments. We'll start with using Fargate as an ECS launch type to deploy a specific version of our `colorteller` service before we move on and explore distributing traffic across other environments, such as EC2 and EKS.

This is intentionally meant to be a simple example for clarity, but in the real world there are many use cases where creating a service mesh that can bridge different compute environments becomes very useful. Fargate is a compute service for AWS that helps you run containerized tasks using the primitives (tasks and services) of an ECS application without the need to directly configure and manage EC2 instances. Our contrived example in this article demonstrates a scenario in which you already have a containerized application running on ECS, but want to shift your workloads to use [Fargate] so that you can evolve your application without the need to manage compute infrastructure directly. 

Our strategy will be to deploy a new version of our `colorteller` service with Fargate and begin shifting traffic to it. If all goes well, then we will continue to shift more traffic to the new version until it is serving 100% of all requests. For this demo, we'll use "blue" to represent the original version and "green" to represent the new version.

As a refresher, this is what the programming model for the Color App looks like:

![](img/appmesh-fargate-colorapp-demo-1.png)
<p align="center"><b><i>Figure 1.</i></b> Programmer perspective of the Color App.</p>

In terms of App Mesh configuration, we will want to begin shifting traffic over from version 1 (represented by `colorteller-blue` in the following diagram) over to version 2 (represented by `colorteller-green`). Remember, in App Mesh, every version of a service is ultimately backed by actual running code somewhere (in this case ECS/Fargate tasks), so each service will have it's own *virtual node* representation in the mesh that provides this conduit.

![](img/appmesh-fargate-colorapp-demo-2.png)
<p align="center"><b><i>Figure 2.</i></b> App Mesh configuration of the Color App.</p>

Finally, there is the physical deployment of the application itself to a compute environment. In this demo, `colorteller-blue` runs on ECS using the EC2 launch type and `colorteller-green` will run on ECS using the Fargate launch type. Our goal is to test with a portion of traffic going to `colorteller-blue`, ultimately increasing to 100% of traffic going to this version.

![](img/appmesh-fargate-colorapp-demo-3.png)
<p align="center"><b><i>Figure 3.</i></b> AWS deployment perspective of the Color App.</p>

## Prerequisites

1. You have successfully set up the prerequisites and deployed the Color App as described in the previous [walkthrough].

## Configuration

### Initial configuration

Once you have deployed the Color App (see #prerequisites), configure the app so that 100% of traffic goes to `colorteller-blue` for now. The blue color will represent version 1 of our colorteller service. There are several ways you can accomplish this and I'll point them out, but then will recommend that you use the last approach.

Log into the App Mesh console and drill down into "Virtual routers" for the mesh. Configure the HTTP route to send 100% of traffic to the `colorteller-blue` virtual node.

![](../appmesh-colorteller-route-1.png)
<p align="center"><b><i>Figure 4.</i></b> Routes in the App Mesh console.</p>

Test the service and confirm in X-Ray that the traffic flows through the `colorteller-blue` as expected with no errors.

![](../appmesh-xray-tracing-1.png)
<p align="center"><b><i>Figure 5.</i></b> Tracing the colorgateway virtual node.</p>

### Deploy the new colorteller to Fargate

For this configuration, we will deploy `colorteller-green`, which represents version 2 of our colorteller service. Initally, we will only send 30% of our traffic over to it. If our monitoring indicates that the service is healthy, we'll increase it to 60%, then finally to 100%. In the real world, you might choose more granular increases with automated rollout (and rollback if issues are indicated), but we're keeping things simple for the demo.

As part of the original [walkthrough] we pushed the `gateway` and `colorteller` images to ECR (see [Deploy Images]) and then launched ECS tasks with these images. We will now launch an ECS task using the Fargate launch type with the same `colorteller` and `envoy` images. When the task is deployed, the running `envoy` container will be a sidecar for the `colorteller` container. Even with the Fargate launch type where we don't manually configure EC2 instances, a sidecar container will always be co-located on the same physical instance and its lifecycle coupled to the lifecycle of the primary application container (see [Sidecar Pattern]).

1. Update the mesh configuration. Our updated CloudFormation templates are located in [this repo].

This updated mesh configuration adds a new virtual node (`colorteller-green-vn`) and updates the virtual router (`colorteller-vr`) for the `colorteller` virtual service, so that traffic will be distributed between the blue and green virtual nodes at a 2:1 ratio (i.e., the green node will receive one third of the traffic).

```
$ ./appmesh-colorapp.sh
...
Waiting for changeset to be created..
Waiting for stack create/update to complete
...
Successfully created/updated stack - DEMO-appmesh-colorapp
$
```

2. Deploy the green task to Fargate. The `fargate-colorteller.sh` script creates parameterized template definitions before deploying the `fargate-colorteller.yaml` CloudFormation template. The change to launch a colorteller task as a Fargate task is in `fargate-colorteller-task-def.json`. 

```
$ ./fargate-colorteller.sh
...

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - DEMO-fargate-colorteller
$
```

### Verifying the Fargate deployment

The endpoint for the ColorApp is one of the CloudFormation template's outputs. You can view it in the stack output in the CloudFormation console, or fetch it with the AWS CLI:

```
$ colorapp=$(aws cloudformation describe-stacks --stack-name=$ENVIRONMENT_NAME-ecs-colorapp --query="Stacks[0
].Outputs[?OutputKey=='ColorAppEndpoint'].OutputValue" --output=text); echo $colorapp> ].Outputs[?OutputKey=='ColorAppEndpoint'].OutputValue" --output=text); echo $colorapp
http://DEMO-Publi-YGZIJQXL5U7S-471987363.us-west-2.elb.amazonaws.com
```

We assigned the endpoint to the `colorapp` environment variable so we can use it for a few curl requests:

```
$ curl $colorapp/color
{"color":"blue", "stats": {"blue":1}}
$

Since the weight of blue to green is 2:1, the result is not unsurprising. Let's run it a few times until we get a green result:

```
$ for ((n=0;n<200;n++)); do echo "$n: $(curl -s $colorapp/color)"; done
0: {"color":"blue", "stats": {"blue":1}}
1: {"color":"green", "stats": {"blue":0.5,"green":0.5}}
2: {"color":"blue", "stats": {"blue":0.67,"green":0.33}}
3: {"color":"green", "stats": {"blue":0.5,"green":0.5}}
4: {"color":"blue", "stats": {"blue":0.6,"green":0.4}}
5: {"color":"green", "stats": {"blue":0.5,"green":0.5}}
6: {"color":"blue", "stats": {"blue":0.57,"green":0.43}}
7: {"color":"blue", "stats": {"blue":0.63,"green":0.38}}
8: {"color":"green", "stats": {"blue":0.56,"green":0.44}}
...
199: {"color":"blue", "stats": {"blue":0.66,"green":0.34}}
```

So far so good; this looks like what we expected for a 2:1 ratio.

Let's take a look at our X-Ray console:

![](img/appmesh-fargate-xray-blue-green.png)
<p align="center"><b><i>Figure 5.</i></b> X-Ray console map after initial testing.</p>




















[A/B testing]: https://en.wikipedia.org/wiki/A/B_testing
[previous article]: ../walkthrough.md
[AWS App Mesh]: https://aws.amazon.com/app-mesh/
[Deploy Images]: https://medium.com/p/de3452846e9d#0d56
[Fargate]: https://aws.amazon.com/fargate/
[Sidecar Pattern]: https://www.oreilly.com/library/view/designing-distributed-systems/9781491983638/ch02.html
[this repo]: https://github.com/subfuzion/appmesh-blog/tree/master/colorapp/fargate
[walkthrough]: ../walkthrough.md
[walkthrough prerequisites]: https://medium.com/containers-on-aws/aws-app-mesh-walkthrough-deploy-the-color-app-on-amazon-ecs-de3452846e9d#42cf