---
title: "Controller-Runtime: The Manager"
categories: [Kubernetes, Operators]
tags: [kubernetes,operators,controller-runtime]
description: A deep dive into the Manager component of the controller-runtime library.
---


## Introduction:
I've been using controller-runtime for quite a while, and like most users, I scaffold the project with kubebuilder and I head straight to the reconciliation loop to do the magic,  Which is honestly great, since I don't need to care about the heavy lifting that was needed before in the early client-go style controllers. But as with any other abstraction in software, there's always some machinery under the hood that is valuable, and sometimes even mandatory, to understand if you want to scale and write more complex projects.

In this blog series I will try to explain some of the machinery behind controller-runtime's magic, starting with an essential component, ***The Manager***.

## What's a Manager:

Let me first give you my definition of a Manager, then we can check the official one:

To put it simply, a **Manager** is responsible for managing the lifecycle of multiple dependant components, such as **Controllers**, **Webhooks**, **Http Servers**, **Leader Elections**, and it's also responsible for providing required dependencies, that are often shared between these components, such as caches, clients and more.
> We'll see more about **Servers** and **Leader Elections** later on.
{: .prompt-info}
The [godoc](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.23.3/pkg/manager#Manager) definition states that a : 
>   Manager initializes shared dependencies such as Caches and Clients, and provides them to Runnables. A Manager is required to create Controllers.


You can see that the latter doesn't differ a lot from the former, but it did mention something about **Runnables**? If you compare the two definitons, you can probably guess that **Runnables** refer to components such as Controllers, Webhooks etc, this will become clearer as we move on, but for now, and to better understand the overall architecture of the Manager, let's dig into its definition.


## The Manager Interface:

The key to understanding the manager is to understand its structure, therefore we should investigate the [Manager interface](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.23.3/pkg/manager#Manager):

> I omitted some irrelevant fields from the interface for brevity.
{: .prompt-info}

```go
type Manager interface {
	// Cluster holds a variety of methods to interact with a cluster.
	cluster.Cluster

	// Add will set requested dependencies on the component, and cause the component to be
	// started when Start is called.
	// Depending on if a Runnable implements LeaderElectionRunnable interface, a Runnable can be run in either
	// non-leaderelection mode (always running) or leader election mode (managed by leader election if enabled).
	Add(Runnable) error

	// Elected is closed when this manager is elected leader of a group of
	// managers, either because it won a leader election or because no leader
	// election was configured.
	Elected() <-chan struct{}

	// AddMetricsServerExtraHandler adds an extra handler served on path to the http server that serves metrics.
	// Might be useful to register some diagnostic endpoints e.g. pprof.
	//
	// Note that these endpoints are meant to be sensitive and shouldn't be exposed publicly.
	//
	// If the simple path -> handler mapping offered here is not enough,
	// a new http server/listener should be added as Runnable to the manager via Add method.
	AddMetricsServerExtraHandler(path string, handler http.Handler) error 

	// Start starts all registered Controllers and blocks until the context is cancelled.
	// Returns an error if there is an error starting any controller.
	//
	// If LeaderElection is used, the binary must be exited immediately after this returns,
	// otherwise components that need leader election might continue to run after the leader
	// lock was lost.
	Start(ctx context.Context) error
}
```
Now let's try to digest it one bit at a time.

### The Cluster:
Honestly the **Cluster** component deserves its own blog, so I'll try to keep it brief. Remember when we said that the Manager initializes shared dependencies like clients and caches, well the cluster is the one who provide these dependencies in the first place.

According to the [godoc](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.23.3/pkg/cluster#Cluster) definition:
> Cluster provides various methods to interact with a cluster.

The [Cluster](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.23.3/pkg/cluster#Cluster) interface isn't that huge, nevertheless it carries a big responsibility:

```go
type Cluster interface {
	recorder.Provider

	// GetHTTPClient returns an HTTP client that can be used to talk to the apiserver
	GetHTTPClient() *http.Client

	// GetConfig returns an initialized Config
	GetConfig() *rest.Config

	// GetCache returns a cache.Cache
	GetCache() cache.Cache

	// GetScheme returns an initialized Scheme
	GetScheme() *runtime.Scheme

	// GetClient returns a client configured with the Config. This client may
	// not be a fully "direct" client -- it may read from a cache, for
	// instance.  See Options.NewClient for more information on how the default
	// implementation works.
	GetClient() client.Client

	// GetFieldIndexer returns a client.FieldIndexer configured with the client
	GetFieldIndexer() client.FieldIndexer

	// GetRESTMapper returns a RESTMapper
	GetRESTMapper() meta.RESTMapper

	// GetAPIReader returns a reader that will be configured to use the API server directly.
	// This should be used sparingly and only when the cached client does not fit your
	// use case.
	GetAPIReader() client.Reader

	// Start starts the cluster
	Start(ctx context.Context) error
}
```

We can observe many interesting, and somewhat expected fields. The cluster has what's needed for a seamless communication with the API Server.A [Provider](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/recorder#Provider) to generate [event recorders](https://pkg.go.dev/k8s.io/client-go/tools/record#EventRecorder), essentially to record and report kubernetes events. It has a raw http ,a cached and an APIReader clients (I will explain the differences and when to use each one in the dedicated cluster blog), and it also provides getters for a [schema](https://pkg.go.dev/k8s.io/apimachinery/pkg/runtime#Scheme) and for a [rest mapper](https://pkg.go.dev/k8s.io/apimachinery/pkg/api/meta#RESTMapper).

The cluster provides access to a cache as well, via the `GetCache()` method. At this point, something should be bothering you, didn't we say that the **Cluster** provides <ins>**shared**</ins> dependencies? but what do we mean by <ins>**shared**</ins>?  Basically that boils down to the fact that we are using [SharedInformers](https://pkg.go.dev/k8s.io/client-go/tools/cache#SharedInformer) (or [SharedIndexInformers](https://pkg.go.dev/k8s.io/client-go/tools/cache#SharedIndexInformer)) instead of the legacy Informer (respectively IndexInformer). Again this deserves its own blog, but sharing the cache between controllers significantly reduces the load on the APIServer (by reducing the lists and watches) and it also lowers the memory consumption (since we'll need to store only one instance of the cache in memory).


Cluster also has a Start method, which creates the caches and the clients when called, and in fact, it makes the Cluster a <ins>**Runnable**</ins>.

### But what's a Runnable?

In controller-runtime, [Runnable](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.23.3/pkg/manager#Runnable) is a pretty simple interface:

```go
type Runnable interface {
	// Start starts running the component.  The component will stop running
	// when the context is closed. Start blocks until the context is closed or
	// an error occurs.
	Start(context.Context) error
}
```
and this is what the documentation says:
> Runnable allows a component to be started .........

I guess it's self explanatory, but what types implements this interface?

After digging a bit I found more Runnables in the controller-runtime than I have initially expected. Probably the most important ones are: 

- [Cluster](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.23.3/pkg/cluster#Cluster)
- [Controller](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.23.3/pkg/controller#Controller)
- [Server](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.23.3/pkg/manager#Server)
    > If you are wondering about this Server type, it's just a typical http server that is Runnable by the manager. It can be used to serve internal http handlers for a variety of needs such as profiling[^1] or health probes. This is mentionned also in the [`AddMetricsServerExtraHandler`](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.23.3/pkg/manager/manager.go#L77) field documentaion in the Manager, where a Server can be used to add more complex metrics setups.
    
- [Informers](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/cache#Informers)
- Different cache types: [delegatingByGVKCache](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.23.3/pkg/cache/delegating_by_gvk_cache.go#L34) and [multiNamespaceCache](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.23.3/pkg/cache/multi_namespace_cache.go#L69)

> Caches are probably the most interesting thing to discuss in controller-runtime, we'll visit them in future blogs.
{: .prompt-info}

**Note** that **Manager** itslef is a Runnable and by starting it, you effectively start the added Runnables, and this flow can be controlled by the [runnableGroup](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.23.3/pkg/manager/runnable_group.go#L108) and by implementing the [LeaderElectionRunnable](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.23.3/pkg/manager#LeaderElectionRunnable) interface.


### Note about leader election:

You probably know that kubernets controllers can be deployed in HA (high availability) mode, this can be achieved easily by increasing the `replica` field in the controller's deployment manifest. HA will be discussed in length in the next blog, but some basic understanding will be quite useful at this stage.

Deploying multiple pods of the controller, means deploying multiple managers and if you have basic distributed systems knowledge, you'll probably guess that we'll need to elect a leader. This has direct implications on the **Runnables** since that depends on the managers. Some Runnables will require their manager to be the leader, hence they should wait for their manager to be elected before calling the `Start` method, meanwhile other Runnables doesn't really care about that, thus they will call the `Start` method immediately when their manager start.

This behaviour is controller by implementing the [LeaderElectionRunnable](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.23.3/pkg/manager#LeaderElectionRunnable) interface:

```go
type LeaderElectionRunnable interface {
	// NeedLeaderElection returns true if the Runnable needs to be run in the leader election mode.
	// e.g. controllers need to be run in leader election mode, while webhook server doesn't.
	NeedLeaderElection() bool
}
```

The above code comment gives us a pretty good example for a component that needs to be run in leader election mode, which is the controller.
> Note that isn't always the case, as we'll see with warm replicas in the next blog.
{: .prompt-warning}


## A concrete example, the controllerManager:

We've been discussing the **Manager** interface, which explained the contract needed to create a manager, but studying a concrete example will make things much more clearer, and will provide us with a reference implementation for the methods we've encountered.

The controller-runtime ships with a ready to use **Manager**, which is the [controllerManager](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.23.3/pkg/manager/internal.go#L65).


The implementation is a bit lengthy, so I'll leave reading the code as an exercise for you, but there's a part that is too important to skip.

> The rest of the struct was omitted for brevity.
{: .prompt-info}
```go
type controllerManager struct {
	sync.Mutex
	started bool

	stopProcedureEngaged *int64
	errChan              chan error
	runnables            *runnables
.................
}
```

We have an internal type called [runnables](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.23.3/pkg/manager/runnable_group.go#L31), let's see what it hides.

```go
// runnables handles all the runnables for a manager by grouping them accordingly to their
// type (webhooks, caches etc.).
type runnables struct {
	HTTPServers    *runnableGroup
	Webhooks       *runnableGroup
	Caches         *runnableGroup
	LeaderElection *runnableGroup
	Warmup         *runnableGroup
	Others         *runnableGroup
}
```
Hmmm, interesting. Hopefully things are starting to click on your side, remember when I said earlier that we can control the flow and order of the Runnables startup? Well, this is the **how** :).

So essentially the controllerManager will group different runnables that shares common behaviour into the same [runnable groups](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.23.3/pkg/manager/runnable_group.go#L108), this way it can coordinate the startup of these runnables.

If we follow the code and comments of the controllerManager [Start](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.23.3/pkg/manager/internal.go#L347) function, we can get a rough idea of that flow.

> As usual, irrelevant code is omitted
{: .prompt-info}
```go
// Start starts the manager and waits indefinitely.
// There is only two ways to have start return:
// An error has occurred during in one of the internal operations,
// such as leader election, cache start, webhooks, and so on.
// Or, the context is cancelled.
func (cm *controllerManager) Start(ctx context.Context) (err error) {


	// First start any HTTP servers, which includes health probes, metrics and profiling if enabled.
	//
	// WARNING: HTTPServers includes the health probes, which MUST start before any cache is populated, otherwise
	// it would block conversion webhooks to be ready for serving which make the cache never get ready.
	logCtx := logr.NewContext(cm.internalCtx, cm.logger)
	if err := cm.runnables.HTTPServers.Start(logCtx); err != nil {
		return fmt.Errorf("failed to start HTTP servers: %w", err)
	}

	// Start any webhook servers, which includes conversion, validation, and defaulting
	// webhooks that are registered.
	//
	// WARNING: Webhooks MUST start before any cache is populated, otherwise there is a race condition
	// between conversion webhooks and the cache sync (usually initial list) which causes the webhooks
	// to never start because no cache can be populated.
	if err := cm.runnables.Webhooks.Start(cm.internalCtx); err != nil {
		return fmt.Errorf("failed to start webhooks: %w", err)
	}

	// Start and wait for caches.
	if err := cm.runnables.Caches.Start(cm.internalCtx); err != nil {
		return fmt.Errorf("failed to start caches: %w", err)
	}

	// Start the non-leaderelection Runnables after the cache has synced.
	if err := cm.runnables.Others.Start(cm.internalCtx); err != nil {
		return fmt.Errorf("failed to start other runnables: %w", err)
	}

	// Start WarmupRunnables and wait for warmup to complete.
	if err := cm.runnables.Warmup.Start(cm.internalCtx); err != nil {
		return fmt.Errorf("failed to start warmup runnables: %w", err)
	}

	// Start the leader election and all required runnables.
	{
		// Create a context that inherits all keys from the parent context
		// but can be cancelled independently for leader election management
		baseCtx := context.WithoutCancel(ctx)
		leaderCtx, cancel := context.WithCancel(baseCtx)
		cm.leaderElectionCancel = cancel
		if leaderElector != nil {
			// Start the leader elector process
			go func() {
				leaderElector.Run(leaderCtx)
				<-leaderCtx.Done()
				close(cm.leaderElectionStopped)
			}()
		} else {
			go func() {
				// Treat not having leader election enabled the same as being elected.
				if err := cm.startLeaderElectionRunnables(); err != nil {
					cm.errChan <- err
				}
				close(cm.elected)
			}()
		}
	}
}
```
As you can see, runnable Groups are used to control the startup order of the Runnables:

 HTTPServers-->Webhooks-->Caches-->Others-->Warmup-->Leader Elections


Another interesting piece of code is the [**Add**](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.23.3/pkg/manager/internal.go#L171) function, which adds the Runnables to the Manager. Let's see the controllerManager's implementation for that one.

The controllerManager Add function will call the [Add](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.23.3/pkg/manager/runnable_group.go#L68) function of its **runnables** field:

```go
// Add adds a runnable to closest group of runnable that they belong to.
//
// Add should be able to be called before and after Start, but not after StopAndWait.
// Add should return an error when called during StopAndWait.
// The runnables added before Start are started when Start is called.
// The runnables added after Start are started directly.
func (r *runnables) Add(fn Runnable) error {
	switch runnable := fn.(type) {
	case *Server:
		if runnable.NeedLeaderElection() {
			return r.LeaderElection.Add(fn, nil)
		}
		return r.HTTPServers.Add(fn, nil)
	case hasCache:
		return r.Caches.Add(fn, func(ctx context.Context) bool {
			return runnable.GetCache().WaitForCacheSync(ctx)
		})
	case webhook.Server:
		return r.Webhooks.Add(fn, nil)
	case warmupRunnable, LeaderElectionRunnable:
		if warmupRunnable, ok := fn.(warmupRunnable); ok {
			if err := r.Warmup.Add(RunnableFunc(warmupRunnable.Warmup), nil); err != nil {
				return err
			}
		}

		leaderElectionRunnable, ok := fn.(LeaderElectionRunnable)
		if !ok {
			// If the runnable is not a LeaderElectionRunnable, add it to the leader election group for backwards compatibility
			return r.LeaderElection.Add(fn, nil)
		}

		if !leaderElectionRunnable.NeedLeaderElection() {
			return r.Others.Add(fn, nil)
		}
		return r.LeaderElection.Add(fn, nil)
	default:
		return r.LeaderElection.Add(fn, nil)
	}
}
```

Here we use a type switch to assign **Runnables** to the correct **runnableGroup**, if you follow the switch statement, it's obvious that webhooks will be assigned to the Webhook runnableGroup, and that servers will be added to the HttpServers runnableGroup, but what about the rest? 

It actually goes as follows:

| Component    | runnableGroup |
| -------- | ------- |
| Cluster  | Caches    |
| Controller | LeaderElection/Warmup/Others   |

The cluster actually implements the [hasCache](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.23.3/pkg/manager/internal.go#L171) interface, so it goes in the Caches group, as for the Controller, it can belong to one of the above 3 groups, depeding on whether the controller require leader election or not, and whether it should be warmed up.



## Conclusion:

The goal of this blog was to give readers a basic understanding of the **Manager** component in controller-runtime. We still have much more to cover, including a followup blog that goes through **Manager's HA**, **Leader Election**, **Failover**, and **Warm Replicas**. In addition, different dependant components that was mentionned throughout the blog deserves their own walkthroughs.




[^1]: You can find a short read about pprof profiling [here](https://eng.d2iq.com/blog/profiling-kubernetes-controllers-with-pprof/) 