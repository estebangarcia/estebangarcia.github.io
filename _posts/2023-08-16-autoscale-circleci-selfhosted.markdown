---
layout: post
title:  "Autoscale your CircleCI self-hosted runners"
date:   2023-08-16 16:00:00 +0100
categories: ci, k8s, ec2, circleci
permalink: /autoscale-circleci-self-hosted-runners/
comments: true
issue_number: 4
---

Greetings, fellow builders of continuous integration dreams! Today, we're diving into the exhilarating world of scaling, but not just any scaling – we're talking about autoscaling your CircleCI self-hosted runners. If you've ever wished for a magic potion to automatically expand your runner fleet to handle surges in your CI/CD demands, you're in luck! In this post, we're going to uncover the secrets of unleashing the power of autoscaling your runners. So, fasten your seatbelts and prepare to embark on an adventure into the realm of dynamic and efficient CI/CD scalability! 🚀🔥

I already created a post about our decision at Vela Games of moving off of Jenkins to CircleCI and the engineering behind our game build pipeline for Evercore Heroes. If you haven’t seen it yet you can [read it here.](https://estebangarcia.io/optimize-unreal-engine-builds/)

In this post, as ChatGPT very creatively explained in the introduction, I want to share our reasoning behind how we handled auto-scaling our CircleCI self-hosted runners and I’m happy to announce we are [open-sourcing the code](https://github.com/estebangarcia/circleci-runner-autoscaler) behind this so anyone can benefit from it.

When we started our journey using the self-hosted runners there was no out-of-the-box way to dynamically scale them, so we started using a [scheduled scaling policy](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-simple-step.html) for each of our autoscaling groups, we scaled-out just before the work day would start and scaled-in after the work day was finished. This mostly worked fine but had a couple of downsides:
1. We use auto-merging on our Github Pull-requests to merge only when CI checks are successful. This meant that when the scheduled policy to scale-in kicked, all the PRs that depended on running pipelines would fail and expected changes we wanted to playtest the next day would not be in.
2. At the same time, all CI checks for new code committed while the agents were down had to wait until the next day to run. This caused a bottleneck during the first hours of the working day when people started to commit new code to the game’s repository.
3. The last point and one of the most important ones for us as a startup was infrastructure costs. Our schedule policy would set the ASG’s desired capacity to its max as there was no way for us to know the demand for agents on a given day. And because we need big EC2 instances to build our game, this meant our spending was significantly up.

Because of these reasons, we determined auto-scaling based on job queue demand was a must for us. So we set out on the task of creating a service that would handle this.

This service would need to:
* Automatically discover all of our resource classes.
* Call CircleCI’s self-hosted runner API to know how many unclaimed tasks are waiting for a runner
* Increase the desired capacity of the ASG associated with the resource class to match the number of unclaimed tasks until the maximum is reached
* Wait for all instances to be up before running the scaling loop again.

This would handle the scaling-out of the runners, for the scale-in we relied on an [available configuration of the agent](https://circleci.com/docs/runner-config-reference/#runner-idle-timeout), that kills the process if it hasn’t received a task after a certain amount of time. So the idea would be to:
1. Change the launch template to terminate the instance on shutdown
2. Change our user-data script in Terraform to inject the timeout for the agent
3. After the process is killed use the AWS CLI to detach the instance from its ASG
4. Shutdown the instance

This simplifies the complexity of the autoscaling service as it didn't need to feed itself with information on how long the agent has been idle to shut it down and kill it.

## CircleCI API Client

The [Runners API](https://circleci.com/docs/runner-api) is a separate API from the main CircleCI API. It has only 3 endpoints:
* `GET /api/v3/runner` - Lists all the registered self-hosted runners
* `GET /api/v3/runner/tasks` - Gives you the number of tasks for a particular resource class that is waiting for a free runner.
* `GET /api/v3/runner/running` - Gives you the number of tasks for a particular resource class that running.

For our project, we only need the first two endpoints to check if a runner is up and to know how many runners we need to create based on the unclaimed tasks count.

There's no GoLang client for this API, so we needed to create one. As the Runner API is very simple and to simplify the development of the client, we relied on the [OpenAPI's Client Generator](https://github.com/deepmap/oapi-codegen).

We created an [OpenAPI YAML definition](https://github.com/vela-games/circleci-runner-autoscaler/blob/main/.openapi/definition.yml) for the API and used `oapi-codegen` to auto-generate a [client](https://github.com/estebangarcia/circleci-runner-autoscaler/blob/main/client/client.go). 


## Dispatcher pattern

Our service continously runs two different tasks:
* Discover auto-scaling groups associated with CircleCI's resource classes
* Handle the auto-scaling for each one of the discovered resource classes

So to handle these tasks, we decided on using something very similar to the dispatcher pattern. We have 2 different types of workers `DiscoveryWorker` and `ScalingWorker`, and a `WorkerDispatcher` that would manage the execution of each one of these every couple of seconds.

These are the interfaces to be implemented by Workers and the Dispatcher:
```golang
// Interface for all workers to implement
type Worker interface {
	Handle(context.Context)
}

type Dispatcher interface {
	Start(context.Context, Worker)
}
```

And here is the core of how the Dispatcher works:

```golang
type WorkerDispatcher struct {
	RunEvery time.Duration
	Group    *errgroup.Group
}

func (w *WorkerDispatcher) Start(ctx context.Context, worker Worker) {
	w.Group.Go(func() error {
		for {
			worker.Handle(ctx)
			select {
			case <-time.After(w.RunEvery):
				continue
			case <-ctx.Done():
				log.Printf("exiting %T", worker)
				return nil
			}
		}
	})
}
```

When the service starts the `WorkerDispatcher` starts the `DiscoveryWorker`, this will discover auto-scaling groups and communicate to the dispatcher that a `ScalingWorker` for each discovered resource class has to be started.

The `ScalingWorker` will take care of calling the CircleCI API to check for Unclaimed Tasks and add to the Desired Capacity of the ASG then wait for instances to be ready and registered against CircleCI before trying to scale again.

## Container-based runners on Kubernetes

The autoscaler also supports creating container runners on Kubernetes, we created this feature as a POC to move certain jobs off of expensive EC2 instances to ephemeral containers on Kubernetes that would significantly reduce costs for us. At the time we did this the [container runner](https://circleci.com/docs/container-runner-installation/) that CircleCI now has didn't exist.

So I'm marking our solutions for containers as DEPRECATED and you shouldn't use it, please use the official solution developed by CircleCI.

In any case, as the code is still there, functional and battle-tested with thousands of jobs served, I'm going to briefly explain our approach for the POC and how to use it.

Our main idea here was when a job was pending to run, the autoscaler would detect that, create a pod to run the job and after that single job has finished the pod terminates and is cleaned up.

We didn't want to create a Custom Resource nor a separate k8s controller for this version of the POC, as we just wanted to prove that this was useful for us and that we could use container runners for the purposes we needed. But we still needed a way to:
* Manage different resource classes in YAML, with potentially different pod templates so we could use different docker images or mount different volumes for each depending on the requirements.
* Be able to discover resource classes templates
* Reliably run the pods to completion, restarting them if the agent failed to start
* Cleanup history of completed pods

A Kubernetes [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) would fit our use-case and we could use a suspended `CronJob` as the template for our jobs. Using a `CronJob` like this is unconventional but it gave us the ability to manage in YAML all the resource classes templates and also it would take care of automatically cleaning up completed pods without having to develop a custom operator from scratch as a first iteration.

The CronJob definition would look like this:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: small-runner
  namespace: circleci-runners
  labels:
    resource-class-org: "vela-games"
    resource-class-name: "k8s-small"
spec:
  suspend: true
  schedule: "* * * * *"
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 5
  jobTemplate:
    spec:
      ttlSecondsAfterFinished: 21600
      template:
        metadata:
          labels:
            resource-class-org: "vela-games"
            resource-class-name: "k8s-small"
        spec:
          containers:
          - name: ci-small
            image: circleci-image:latest
            command: ["/opt/circleci/start.sh"]
            env:
            - name: LAUNCH_AGENT_RUNNER_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: LAUNCH_AGENT_API_AUTH_TOKEN
              valueFrom:
                secretKeyRef:
                  name: api-token
                  key: LAUNCH_AGENT_API_AUTH_TOKEN
                  optional: false
            imagePullPolicy: IfNotPresent
            resources:
              limits:
                cpu: 2000m
                memory: 1Gi
                ephemeral-storage: "15Gi"
              requests:
                cpu: 2000m
                memory: 1Gi
                ephemeral-storage: "15Gi"
          restartPolicy: OnFailure
```

In the same way as the EC2 autoscaler, we have 2 workers, Discovery and Scaling workers. The main difference is that the `DiscoveryWorker` is checking for CronJobs within the namespace that has the labels `resource-class-org` and `resource-class-name` and starts a `ScalingWorker` that creates a `Job` per unclaimed task and waits for the pods to register against CircleCI's runner API.

In case you want to use this there's a [Dockerfile](https://github.com/estebangarcia/circleci-runner-autoscaler/blob/main/install/k8s-image/Dockerfile) in the repository to build the runner image.

## Deploying the autoscaler

Currently, I have neither a public helm repository nor a Docker repository to provide public images (something I'm currently looking at) but I'm providing in the repository a [Dockerfile](https://github.com/estebangarcia/circleci-runner-autoscaler/blob/main/Dockerfile), [Helm chart](https://github.com/estebangarcia/circleci-runner-autoscaler/tree/main/install/helm/circleci-runner-autoscaler) and [terraform module](https://github.com/estebangarcia/circleci-runner-autoscaler/tree/main/install/terraform/circleci-runner-autoscaler) for the necessary IAM permission for the Kubernetes service account.

You can use these to deploy the autoscaler on your infrastructure. The only prerequisites are that:
* Either the Kubernetes service account mounted in the pods or EC2 instances have an IAM role attached with rights to allow autoscaling actions
* Configure the CircleCI Token in the `APP_CIRCLE_TOKEN` environment variable
* Configure the Runner Namespace in the `APP_CIRCLE_RESOURCE_NAMESPACE` environment variable

And there you have it, dear builders of continuous integration greatness! With the knowledge and insights gained from this journey, you're now equipped to master the art of autoscaling in your CircleCI self-hosted environment. As you watch your runner fleet expand and contract effortlessly to accommodate your CI/CD needs, remember that the power of autoscaling is in your hands. May your builds be swift, your tests be thorough, and your deployments be as smooth as a perfectly orchestrated symphony. Until we embark on our next scaling adventure, keep coding, keep building, and keep scaling with the magic of CircleCI by your side! 🚀🔥🛠️