---
layout: post
title:  "Using reactors for unit testing in golang your k8s-powered code"
date:   2023-08-03 16:00:00 +0100
categories: golang k8s
permalink: /unit-testing-k8s-golang/
comments: true
issue_number: 3
---

Welcome to a wizard's guide to unit testing in the world of Golang and Kubernetes! Get ready to wield the sorcery of the k8s fake client to ensure your Golang application is battle-tested and ready to conquer any challenge. Grab your wands (or keyboards), as we embark on an enchanting adventure through the realm of unit testing in Kubernetes-powered Golang applications! 🧙🚀🔮

The ChatGPT introductions to my posts are now a norm, even though they are super cheesy and make me cringe sometimes. This post will be a short one in comparison to my last one. I was recently working on a Golang service to autoscale our CircleCI agents (stay tuned for a post on this) based on how many jobs were pending execution, and we decided to support both EC2 and Kubernetes-based agents, these are short-lived and only run a single job before terminating.

So naturally the service uses the Golang Kubernetes client to be able to create these pods, and when I got to the moment when I wanted to create a unit test I had a bit of a hard time trying to find how to do this. Hopefully, this post will be useful to you if you find yourself in the same situation.

The example code I'm going to show is going to create a pod if another pod named `testing` exists in the same namespace.

This is not a real-life scenario is only to show you how in the unit tests we can:
* Return a fake list of pods
* Verify if the create pod action was executed

Let's start with creating a quick struct that tries to get the `testing` pod and creates another one if it exists:
```golang
package blog

import (
	"context"
	"errors"

	corev1 "k8s.io/api/core/v1"
	v1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
)

type PodCreator struct {
	Namespace string
	ClientSet kubernetes.Interface
}

func (p *PodCreator) CreatePod(ctx context.Context) error {
	pod, err := p.ClientSet.CoreV1().Pods(p.Namespace).Get(ctx, "testing", v1.GetOptions{})
	if err != nil {
		return err
	}

	if pod == nil {
		return errors.New("pod not found")
	}

	podToCreate := &corev1.Pod{
		ObjectMeta: v1.ObjectMeta{
			Name:      "creating-a-new-pod",
			Namespace: p.Namespace,
		},
		Spec: corev1.PodSpec{
			Containers: []corev1.Container{
				{
					Name:  "ubuntu",
					Image: "ubuntu:latest",
				},
			},
		},
	}

	_, err = p.ClientSet.CoreV1().Pods(p.Namespace).Create(ctx, podToCreate, v1.CreateOptions{})
	if err != nil {
		return err
	}

	return nil
}
```

Now we create our test to verify our code works. The test is going to use a fake client that is provided in the k8s golang client library, this client implements the `kubernetes.Interface` interface, on creation we set the resources that are available on our fake cluster.

```golang
package blog

import (
	"context"
	"testing"

	"github.com/stretchr/testify/assert"
	corev1 "k8s.io/api/core/v1"
	v1 "k8s.io/apimachinery/pkg/apis/meta/v1"

	testclient "k8s.io/client-go/kubernetes/fake"
)

func TestPodCreation(t *testing.T) {
	t.Run("it should return error because pod testing doesn't exist", func(t *testing.T) {
		// We create the fake client with a PodList containing a Pod with the name "t"
		k8sClient := testclient.NewSimpleClientset(&corev1.PodList{
			Items: []corev1.Pod{
				{
					ObjectMeta: v1.ObjectMeta{
						Name:      "t",
						Namespace: "default",
					},
					Spec: corev1.PodSpec{
						Containers: []corev1.Container{
							{
								Name:  "ubuntu-testing",
								Image: "ubuntu:latest",
							},
						},
					},
				},
			},
		})

		podCreator := PodCreator{
			Namespace: "default",
			ClientSet: k8sClient,
		}

		err := podCreator.CreatePod(context.TODO())
		assert.Errorf(t, err, "pods \"testing\" not found")
	})
}
```

The previous only tests the scenario where the `testing` pod doesn't exist, but it shows you how to create the fake client and how to expose fake resources.

To be able to test if our create action is called and the properties of the pod being created match what we need, we have to add to the fake client a `Reactor`. As the name suggests a `Reactor` here "reacts" to different actions being executed by the k8s client and calls a callback `func` we specify.

This is a basic test to just assert if the create pod action was executed:

```golang
t.Run("it should create our pod", func(t *testing.T) {
  // We create the fake client with a PodList containing a Pod with the name "testing"
  k8sClient := testclient.NewSimpleClientset(&corev1.PodList{
    Items: []corev1.Pod{
      {
        ObjectMeta: v1.ObjectMeta{
          Name:      "testing",
          Namespace: "default",
        },
        Spec: corev1.PodSpec{
          Containers: []corev1.Container{
            {
              Name:  "ubuntu-testing",
              Image: "ubuntu:latest",
            },
          },
        },
      },
    },
  })

  createPodCalled := false
  k8sClient.PrependReactor("create", "pods", func(action k8stesting.Action) (bool, runtime.Object, error) {
    createPodCalled = true
    return true, nil, nil
  })

  podCreator := PodCreator{
    Namespace: "default",
    ClientSet: k8sClient,
  }

  err := podCreator.CreatePod(context.TODO())
  assert.NoError(t, err)
  assert.True(t, createPodCalled)
})
```

And this is how we can access the pod object related to this action to check its properties:

```golang
k8sClient.PrependReactor("create", "pods", func(action k8stesting.Action) (bool, runtime.Object, error) {
    obj := action.(k8stesting.CreateAction).GetObject()
    pod, ok := obj.(*corev1.Pod)
    assert.True(t, ok)
    assert.Equal(t, "creating-a-new-pod", pod.Name)
    assert.Equal(t, "default", pod.Namespace)
    return true, nil, nil
})
```
And with that, our magical journey through the world of unit testing in Golang and Kubernetes comes to an end. Armed with the knowledge of the fake client, you now possess the power to ensure your applications are robust, reliable, and ready for any quest that lies ahead. So go forth, fellow wizards of code, and let the magic of unit testing guide your every step. May your tests always be passing, your deployments smooth as a spell, and your applications enchant users far and wide. Until our next adventure together, keep coding and conjuring greatness!