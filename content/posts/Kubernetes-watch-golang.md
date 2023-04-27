---
title: "Kubernetes-watch-golang实现"
date: 2020-06-23T00:00:00+08:00
---

##### 第一种方法

使用`k8s.io/client-go/tools/cache`

```go
func StartWatchingServices(c *Client) {
	watchlist := cache.NewListWatchFromClient(
		c.K8sClientset.CoreV1().RESTClient(),
		string(v1.ResourceServices),
		v1.NamespaceAll,
		fields.Everything(),
	)

	_, controller := cache.NewInformer(
		watchlist,
		&v1.Service{},
		0, //Duration is int64
		cache.ResourceEventHandlerFuncs{
			AddFunc: func(obj interface{}) {
				fmt.Printf("added: %s \n", obj)
			},
			DeleteFunc: func(obj interface{}) {
				fmt.Printf(deleted: %s \n", obj)
			},
			UpdateFunc: func(oldObj, newObj interface{}) {
				fmt.Printf("changed \n")
			},
		},
	)

	stop := make(chan struct{})
	defer close(stop)
	go controller.Run(stop)
	for {
		time.Sleep(time.Second)
	}
}
```

##### 第二种方法

使用`k8s.io/client-go/informers`，可以 watch CRD 资源但需要使用 code-generator 生成 clientset 和 informer(指 CRD 资源)。

```go
func StartWatchingServices2(c *Client) {
	kubeInformerFactory := informers.NewSharedInformerFactory(c.K8sClientset, time.Second*30)
	svcInformer := kubeInformerFactory.Core().V1().Services().Informer()

	svcInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			fmt.Printf("added: %s \n", obj)
		},
		DeleteFunc: func(obj interface{}) {
			fmt.Printf("deleted: %s \n", obj)
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			fmt.Printf("changed: %s \n", newObj)
		},
	})

	stop := make(chan struct{})
	defer close(stop)
	kubeInformerFactory.Start(stop)
	for {
		time.Sleep(time.Second)
	}
}
```

##### 第三种方法

使用`	runtimeCache "sigs.k8s.io/controller-runtime/pkg/cache"`, 可以 watch CRD 资源，无需特殊处理。下例`v1alpha1.Component`是 CRD 资源。

```go
func StartWatchingComponents(c *Client) {
	informerCache, err := runtimeCache.New(c.RestConfig, runtimeCache.Options{})
	if err != nil {
		panic(err)
	}

	informer, err := informerCache.GetInformer(&v1alpha1.Component{})
	if err != nil {
		panic(err)
	}

	informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
  		AddFunc: func(obj interface{}) {
			fmt.Printf("added: %s \n", obj)
		},
		DeleteFunc: func(obj interface{}) {
			fmt.Printf("deleted: %s \n", obj)
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			fmt.Printf("changed: %s \n", newObj)
		},
        })

	stop := make(chan struct{})
	defer close(stop)
	informerCache.Start(stop)
	// for {
	// 	time.Sleep(time.Second)
	// }
}
```
