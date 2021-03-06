Operator tutorial
===================

In this tutorial we will learn how to create an opeator using Kooper.

If you didn't read the [previous tutorial](controller-tutorial.md) about controllers you should read that first.

Lets start by remembering what is an operator... Depending who or where you ask you will get different answers, very similar but not the same, but everyone concludes that an operator is just a controller that bases its flow on CRDs or custom resources created by us instead of Kubernetes core resources like pods, services, deployments...

In this tutorial we will make an operator that will terminate pods, something like [chaos monkey](https://github.com/Netflix/chaosmonkey) but for pods and very very simple.

## 01 - Description.

Our Operator will apply [chaos engineering](http://principlesofchaos.org/) at a very low level. It will delete pods based on the time (interval), quantity, and filtered by labels.

It will lack a lot of things and the main purpose of this tutorial is not the domain logic of the operator (the pod terminator, that is cool also and you should read it :)) but how you bootstrap a full working operator with its CRDs using Kooper.

The full controller is in [examples/pod-terminator-operator](https://github.com/spotahome/kooper/tree/master/examples/pod-terminator-operator).

## 02 - Operator structure.

Our example structure is simple

```
./examples/pod-terminator-operator/                                                                               
├── apis                                                      
│   └── chaos          
│       ├── register.go
│       └── v1alpha1               
│           ├── doc.go 
│           ├── register.go
│           ├── types.go
│           └── zz_generated.deepcopy.go [AUTOGENERATED CODE]
├── client             
│   └── k8s            
│       └── clientset [AUTOGENERATED CODE]
├── cmd
│   ├── flags.go
│   └── main.go
├── log
│   └── log.go
├── Makefile
├── manifest-examples
│   ├── nginx-controller.yaml
│   └── pause.yaml
├── operator
│   ├── config.go
│   ├── crd.go
│   ├── factory.go
│   └── handler.go
└── service
    └── chaos
        ├── chaos.go
        ├── podkill.go
        └── podkill_test.go

```

* apis: There will reside our custom Kubernetes API objects (crds).
* client/k8s/clientset: This is a k8s client for our CRDs so it is easy to use them with kubernetes. This code is autogenerated by [code-generator](https://github.com/kubernetes/code-generator).
* cmd: The main app that will run our operator.
* log: Our logger.
* Makefile: Commands to automate stuff for the example (in this case autogenerate our CRD code).
* manifest-examples: Some yaml examples of our CRD.
* operator: All the kooper library usage to create the operator.
* service: our services where also will be our operator domain logic (chaos).


### Unit testing.

Testing is important. As you see this project has few unit tests. This is because of two things:

One, the project is very simple.

Two, you can trust Kubernetes and Kooper libraries, they are already tested, you don't need to test these, but you should test your domain logic (and if you want, main and glue code also). In this operator we just tested the service that has our domain logic (the logic that kills the pods).

## 03 - Designing our CRD.

The first thing that we need to do is to design our CRD. In the case of this example the definition of the resource needs to be something that need to select pods based on labels and periodically kill them, also needs to respect a safe quantity of minimum pods running and lastly a dry run option to check if it's selecting the correct pods to kill. Something like this:

```yaml
apiVersion: chaos.spotahome.com/v1alpha1
kind: PodTerminator
metadata:
  name: nginx-controller
  labels:
    example: pod-terminator-operator
    operator: pod-terminator-operator
spec:
  selector:
    k8s-app: nginx-ingress-controller
  periodSeconds: 120
  TerminationPercent: 25
  MinimumInstances: 2
  DryRun: true
```

We called `PodTerminator` and its group will be `chaos.spotahome.com/v1alpha1` we selected `v1alpha1` as the version of the group because is the first iteration of our resources, eventually it will end up being `v1` when its stable, but in this example it will be there forever. You can imagine doing this when the example is up and running:

```
$ kubectl get podterminators
NAME               AGE
nginx-controller   1h
```

### Implementin the CRD.

We have designed the CRD, but now we need to implement. All of the operator resources are in [apis](https://github.com/spotahome/kooper/tree/master/examples/pod-terminator-operator/apis) directory inside the example, the structure of the CRD follows Kubernetes conventions (group/version). How this structure is placed and how it's implemented its out of the scope of this tutorial, you can check [kubernetes api](https://github.com/kubernetes/api) repository and [this blog post](https://blog.openshift.com/kubernetes-deep-dive-code-generation-customresources/)

To summarize a little bit, we have this API structure:

```
./examples/pod-terminator-operator/apis/
└── chaos
    ├── register.go
    └── v1alpha1
        ├── doc.go
        ├── register.go
        ├── types.go
        └── zz_generated.deepcopy.go
```

in `types` is our [PodTerminator Go object](https://github.com/spotahome/kooper/blob/master/examples/pod-terminator-operator/apis/chaos/v1alpha1/types.go) that describes de API and in `register.go` files there is data to register this types in kubernetes client.

With these files kubernetes code-generator will generate the required code for the [clients](https://github.com/spotahome/kooper/tree/master/examples/pod-terminator-operator/client/k8s/clientset) and deepcopy methods ([deepcopy](https://github.com/spotahome/kooper/blob/master/examples/pod-terminator-operator/apis/chaos/v1alpha1/zz_generated.deepcopy.go) methods are required by all the kubernetes objects). You can see how its used in the [Makefile](https://github.com/spotahome/kooper/blob/master/examples/pod-terminator-operator/Makefile). This will generate all the boilerplate code that is the same in all the kubernetes objects (crds and not crds).

At this point we have our PodTerminator CRD go code ready to work with (create, get, delete, list, watch... using kubernetes client).

## 04 - Chaos service

Let's start with our domain logic. Our domain logic is a single service called `Chaos`. This service will be responsible for running a number of `podKillers`, and these podkillers will kill pods at regular intervals based on the filters and description described on our manifests (`PodTerminator` CRD).

We will not explain the logic of this service as is out of the scope of this operator tutorial. But you can take a look [here]((https://github.com/spotahome/kooper/tree/master/examples/pod-terminator-operator/service/chaos) if you think is interesting.

Our service has these methods and are the ones that will be used by our operator.

```golang
type ChaosSyncer interface {
	EnsurePodTerminator(pt *chaosv1alpha1.PodTerminator) error
	DeletePodTerminator(name string) error
}
```

For us, create and update are the same. This means that we need to think as "ensure" or "sync" verbs instead of create/update.

## 05 - Operator

### Config

This operator only has the resync period (the interval kubernetes will return us all the resources we are listening to)

This can be found in [operator/config.go](https://github.com/spotahome/kooper/blob/master/examples/pod-terminator-operator/operator/config.go)
```
type Config struct {
	// ResyncPeriod is the resync period of the operator.
	ResyncPeriod time.Duration
}
```

### CRD

The first thing that the operator needs is the CRD, the operator will ensure CRD (`PodTerminator`) is registered on kubernetes and also will be reacting to this resource changes (add/update and delete).

This can be found in [operator/crd.go](https://github.com/spotahome/kooper/blob/master/examples/pod-terminator-operator/operator/crd.go).

For the initialization kooper [CRD client](https://github.com/spotahome/kooper/blob/master/client/crd/crd.go) is used.
All the stuff required by the `crd.Conf` was added on the implementation and design of the CRD in previous steps.

```
func (p *podTerminatorCRD) Initialize() error {
	crd := crd.Conf{
		Kind:       chaosv1alpha1.PodTerminatorKind,
		NamePlural: chaosv1alpha1.PodTerminatorNamePlural,
		Group:      chaosv1alpha1.SchemeGroupVersion.Group,
		Version:    chaosv1alpha1.SchemeGroupVersion.Version,
		Scope:      chaosv1alpha1.PodTerminatorScope,
	}

	return p.crdCli.EnsurePresent(crd)
}
```

You will notice that the lister watcher and the object is like a retriever. This is because a CRD is a retriever but it knows how to initialize, nothing more. As you see on the lister watcher we are using the client that code-generator generated for us in previous steps.

```golang
func (p *podTerminatorCRD) GetListerWatcher() cache.ListerWatcher {
	return &cache.ListWatch{
		ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
			return p.podTermCli.ChaosV1alpha1().PodTerminators().List(options)
		},
		WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
			return p.podTermCli.ChaosV1alpha1().PodTerminators().Watch(options)
		},
	}
}

func (p *podTerminatorCRD) GetObject() runtime.Object {
	return &chaosv1alpha1.PodTerminator{}
}
```

### Handler

The handler is how the operator will react to Kubernetes resource changes/notifications. In this case is very simple because all the domain logic is inside our services (decouple your domain logic and responsibilities :)).

This can be found in [operator/handler.go](https://github.com/spotahome/kooper/blob/master/examples/pod-terminator-operator/operator/handler.go).

```golang
func (h *handler) Add(obj runtime.Object) error {
	pt, ok := obj.(*chaosv1alpha1.PodTerminator)
	if !ok {
		return fmt.Errorf("%v is not a pod terminator object", obj.GetObjectKind())
	}

	return h.chaosService.EnsurePodTerminator(pt)
}

func (h *handler) Delete(name string) error {
	return h.chaosService.DeletePodTerminator(name)
}
```

### Factory.

All the pieces are ready, let's glue all together.


This can be found in [operator/factory.go](https://github.com/spotahome/kooper/blob/master/examples/pod-terminator-operator/operator/factory.go).
```golang
func New(cfg Config, podTermCli podtermk8scli.Interface, crdCli crd.Interface, kubeCli kubernetes.Interface, logger log.Logger) (operator.Operator, error) {

	// Create crd.
	ptCRD := newPodTermiantorCRD(podTermCli, crdCli, kubeCli)

	// Create handler.
	handler := newHandler(kubeCli, logger)

	// Create controller.
	ctrl := controller.NewSequential(cfg.ResyncPeriod, handler, ptCRD, logger)

	// Assemble CRD and controller to create the operator.
	return operator.NewOperator(ptCRD, ctrl, logger), nil
}
```

First the CRD instance is created, next the handler, then with the handler and the CRD the controller (remember that an operator is just a controller +  CRD) and finally the operator is created using the controller and the CRD.


## 06 - Finishing.

After all these steps, a fully operational chaos operator is ready to destroy pods. You can check the [main](https://github.com/spotahome/kooper/blob/master/examples/pod-terminator-operator/cmd/main.go) where the operator is instantiated.

Run everything with:

```bash
$ go run ./examples/pod-terminator-operator/cmd/* --help
```

And you can test the CRD with the examples or creating yours!

```bash
$ kubectl apply -f ./examples/pod-terminator-operator/manifest-examples/nginx-controller.yaml 
```

