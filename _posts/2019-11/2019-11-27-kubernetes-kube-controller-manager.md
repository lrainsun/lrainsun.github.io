---
layout: post
title:  "Kubernetes kube-controller-manager启动过程分析"
date:   2019-11-27 11:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: Kubernetes kube-controller-manager启动过程分析
mathjax: true
typora-root-url: ../
---

# kube-controller-manager启动

大部分的controller都包含在kube-controller-manager里，所以我们回到controller manager启动的最初：

*k8s.io/kubernetes/cmd/kube-controller-manager/controller-manager.go*

```go
func main() {
	rand.Seed(time.Now().UnixNano())

	command := app.NewControllerManagerCommand()

	// TODO: once we switch everything over to Cobra commands, we can go back to calling
	// utilflag.InitFlags() (by removing its pflag.Parse() call). For now, we have to set the
	// normalize func and add the go flag set by hand.
	// utilflag.InitFlags()
	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}
```

而在newControllerManagerCommand()里会创建Run command，而最终会执行Run函数

```go
// NewControllerManagerCommand creates a *cobra.Command object with default parameters
func NewControllerManagerCommand() *cobra.Command {
	s, err := options.NewKubeControllerManagerOptions()
	if err != nil {
		klog.Fatalf("unable to initialize command options: %v", err)
	}

	cmd := &cobra.Command{
		Use: "kube-controller-manager",
		Long: `The Kubernetes controller manager is a daemon that embeds
the core control loops shipped with Kubernetes. In applications of robotics and
automation, a control loop is a non-terminating loop that regulates the state of
the system. In Kubernetes, a controller is a control loop that watches the shared
state of the cluster through the apiserver and makes changes attempting to move the
current state towards the desired state. Examples of controllers that ship with
Kubernetes today are the replication controller, endpoints controller, namespace
controller, and serviceaccounts controller.`,
		Run: func(cmd *cobra.Command, args []string) {
			verflag.PrintAndExitIfRequested()
			utilflag.PrintFlags(cmd.Flags())

			c, err := s.Config(KnownControllers(), ControllersDisabledByDefault.List())
			if err != nil {
				fmt.Fprintf(os.Stderr, "%v\n", err)
				os.Exit(1)
			}

			if err := Run(c.Complete(), wait.NeverStop); err != nil {
				fmt.Fprintf(os.Stderr, "%v\n", err)
				os.Exit(1)
			}
		},
	}

// Run runs the KubeControllerManagerOptions.  This should never exit.
func Run(c *config.CompletedConfig, stopCh <-chan struct{}) error {
}
```

# Run函数

在Run函数里会做这么几件事情：

* 参数注册

  ```go
  if cfgz, err := configz.New(ConfigzName); err == nil {
     cfgz.Set(c.ComponentConfig)
  } else {
     klog.Errorf("unable to register configz: %v", err)
  }
  ```

* 设置健康检查

  ```go
  // Setup any healthz checks we will want to use.
  var checks []healthz.HealthChecker
  var electionChecker *leaderelection.HealthzAdaptor
  if c.ComponentConfig.Generic.LeaderElection.LeaderElect {
     electionChecker = leaderelection.NewLeaderHealthzAdaptor(time.Second * 20)
     checks = append(checks, electionChecker)
  }
  ```

* 启动controller manager http server

  ```go
  // Start the controller manager HTTP server
  // unsecuredMux is the handler for these controller *after* authn/authz filters have been applied
  var unsecuredMux *mux.PathRecorderMux
  if c.SecureServing != nil {
     unsecuredMux = genericcontrollermanager.NewBaseHandler(&c.ComponentConfig.Generic.Debugging, checks...)
     handler := genericcontrollermanager.BuildHandlerChain(unsecuredMux, &c.Authorization, &c.Authentication)
     // TODO: handle stoppedCh returned by c.SecureServing.Serve
     if _, err := c.SecureServing.Serve(handler, 0, stopCh); err != nil {
        return err
     }
  }
  if c.InsecureServing != nil {
     unsecuredMux = genericcontrollermanager.NewBaseHandler(&c.ComponentConfig.Generic.Debugging, checks...)
     insecureSuperuserAuthn := server.AuthenticationInfo{Authenticator: &server.InsecureSuperuser{}}
     handler := genericcontrollermanager.BuildHandlerChain(unsecuredMux, nil, &insecureSuperuserAuthn)
     if err := c.InsecureServing.Serve(handler, 0, stopCh); err != nil {
        return err
     }
  }
  ```

* 初始化ControllerContext，内容大概包括clientBuilder, InformerFactory等等，client是api server的客户端，用来通过rest的方式访问api server的服务

  ```go
  // CreateControllerContext creates a context struct containing references to resources needed by the
  // controllers such as the cloud provider and clientBuilder. rootClientBuilder is only used for
  // the shared-informers client and token controller.
  func CreateControllerContext(s *config.CompletedConfig, rootClientBuilder, clientBuilder controller.ControllerClientBuilder, stop <-chan struct{}) (ControllerContext, error) {
     ctx := ControllerContext{
        ClientBuilder:                   clientBuilder,
        InformerFactory:                 sharedInformers,
        ObjectOrMetadataInformerFactory: controller.NewInformerFactory(sharedInformers, metadataInformers),
        ComponentConfig:                 s.ComponentConfig,
        RESTMapper:                      restMapper,
        AvailableResources:              availableResources,
        Cloud:                           cloud,
        LoopMode:                        loopMode,
        Stop:                            stop,
        InformersStarted:                make(chan struct{}),
        ResyncPeriod:                    ResyncPeriod(s),
     }
     return ctx, nil
  }
  ```

* 首先启动serviceAccountTokenController，因为serviceAccountTokenController比较特殊，它需要先启动来为其他controller设置permissions，使用的是rootClientBuilder

  ```go
  saTokenControllerInitFunc := serviceAccountTokenControllerStarter{rootClientBuilder: rootClientBuilder}.startServiceAccountTokenController
  
  // serviceAccountTokenControllerStarter is special because it must run first to set up permissions for other controllers.
  // It cannot use the "normal" client builder, so it tracks its own. It must also avoid being included in the "normal"
  // init map so that it can always run first.
  type serviceAccountTokenControllerStarter struct {
  	rootClientBuilder controller.ControllerClientBuilder
  }
  
  func (c serviceAccountTokenControllerStarter) startServiceAccountTokenController(ctx ControllerContext) (http.Handler, bool, error) {
    	controller, err := serviceaccountcontroller.NewTokensController(
  		ctx.InformerFactory.Core().V1().ServiceAccounts(),
  		ctx.InformerFactory.Core().V1().Secrets(),
  		c.rootClientBuilder.ClientOrDie("tokens-controller"),
  		serviceaccountcontroller.TokensControllerOptions{
  			TokenGenerator: tokenGenerator,
  			RootCA:         rootCA,
  		},
  	)
  	if err != nil {
  		return nil, true, fmt.Errorf("error creating Tokens controller: %v", err)
  	}
  	go controller.Run(int(ctx.ComponentConfig.SAController.ConcurrentSATokenSyncs), ctx.Stop)
  
  	// start the first set of informers now so that other controllers can start
  	ctx.InformerFactory.Start(ctx.Stop)
  
  	return nil, true, nil
  }
  ```

* 启动controllers

  ```go
  if err := StartControllers(controllerContext, saTokenControllerInitFunc, NewControllerInitializers(controllerContext.LoopMode), unsecuredMux); err != nil {
     klog.Fatalf("error starting controllers: %v", err)
  }
  
  // StartControllers starts a set of controllers with a specified ControllerContext
  func StartControllers(ctx ControllerContext, startSATokenController InitFunc, controllers map[string]InitFunc, unsecuredMux *mux.PathRecorderMux) error {
    
  }
  ```

  * 看看startcontrollers的入参，一个是刚刚说的controllerContext，serviceaccounttokenConroller的initFunc，还有一个controllers的InitFunc的map -> 这里包含了所有要包含在kube-controller-manager里的controllers。之前学习的replicaset自然也在里面: *controllers["replicaset"] = startReplicaSetController*

    ```go
    // NewControllerInitializers is a public map of named controller groups (you can start more than one in an init func)
    // paired to their InitFunc.  This allows for structured downstream composition and subdivision.
    func NewControllerInitializers(loopMode ControllerLoopMode) map[string]InitFunc {
       controllers := map[string]InitFunc{}
       controllers["endpoint"] = startEndpointController
       controllers["endpointslice"] = startEndpointSliceController
       controllers["replicationcontroller"] = startReplicationController
       controllers["podgc"] = startPodGCController
       controllers["resourcequota"] = startResourceQuotaController
       controllers["namespace"] = startNamespaceController
       controllers["serviceaccount"] = startServiceAccountController
       controllers["garbagecollector"] = startGarbageCollectorController
       controllers["daemonset"] = startDaemonSetController
       controllers["job"] = startJobController
       controllers["deployment"] = startDeploymentController
       controllers["replicaset"] = startReplicaSetController
       controllers["horizontalpodautoscaling"] = startHPAController
       controllers["disruption"] = startDisruptionController
       controllers["statefulset"] = startStatefulSetController
       controllers["cronjob"] = startCronJobController
       controllers["csrsigning"] = startCSRSigningController
       controllers["csrapproving"] = startCSRApprovingController
       controllers["csrcleaner"] = startCSRCleanerController
       controllers["ttl"] = startTTLController
       controllers["bootstrapsigner"] = startBootstrapSignerController
       controllers["tokencleaner"] = startTokenCleanerController
       controllers["nodeipam"] = startNodeIpamController
       controllers["nodelifecycle"] = startNodeLifecycleController
       if loopMode == IncludeCloudLoops {
          controllers["service"] = startServiceController
          controllers["route"] = startRouteController
          controllers["cloud-node-lifecycle"] = startCloudNodeLifecycleController
          // TODO: volume controller into the IncludeCloudLoops only set.
       }
       controllers["persistentvolume-binder"] = startPersistentVolumeBinderController
       controllers["attachdetach"] = startAttachDetachController
       controllers["persistentvolume-expander"] = startVolumeExpandController
       controllers["clusterrole-aggregation"] = startClusterRoleAggregrationController
       controllers["pvc-protection"] = startPVCProtectionController
       controllers["pv-protection"] = startPVProtectionController
       controllers["ttl-after-finished"] = startTTLAfterFinishedController
       controllers["root-ca-cert-publisher"] = startRootCACertPublisher
    
       return controllers
    }
    ```

  * 在startControllers里，会把上面map里的每个controller都启动起来，比如repliasetcontroller，就会调用*startReplicaSetController*，后面就是[replicaset controller启动过程](https://lrainsun.github.io/2019/11/24/kubernetes-relicaset-code/)

    ```go
    for controllerName, initFn := range controllers {
       if !ctx.IsControllerEnabled(controllerName) {
          klog.Warningf("%q is disabled", controllerName)
          continue
       }
    
       time.Sleep(wait.Jitter(ctx.ComponentConfig.Generic.ControllerStartInterval.Duration, ControllerStartJitter))
    
       klog.V(1).Infof("Starting %q", controllerName)
       debugHandler, started, err := initFn(ctx)
       if err != nil {
          klog.Errorf("Error starting %q", controllerName)
          return err
       }
       if !started {
          klog.Warningf("Skipping %q", controllerName)
          continue
       }
       if debugHandler != nil && unsecuredMux != nil {
          basePath := "/debug/controllers/" + controllerName
          unsecuredMux.UnlistedHandle(basePath, http.StripPrefix(basePath, debugHandler))
          unsecuredMux.UnlistedHandlePrefix(basePath+"/", http.StripPrefix(basePath, debugHandler))
       }
       klog.Infof("Started %q", controllerName)
    }
    ```

* InformerFactory.Start，会把controllerContext里构造的sharedInformers start。

  ```go
  controllerContext.InformerFactory.Start(controllerContext.Stop)
  
  func CreateControllerContext(s *config.CompletedConfig, rootClientBuilder, clientBuilder controller.ControllerClientBuilder, stop <-chan struct{}) (ControllerContext, error) {
  	versionedClient := rootClientBuilder.ClientOrDie("shared-informers")
  	sharedInformers := informers.NewSharedInformerFactory(versionedClient, ResyncPeriod(s)())
  }
  
  		InformerFactory:                 sharedInformers,
  ```

  看一下sharedInformers的构造，有两个参数：

  * versionedClient，所有的其他contronllers，用的都是rootClient来与api server交互

  * ResyncPeriod(s)，返回的是一个函数，这个函数用来生成一个时间段，这样，多个controller就不会陷入lock-step，而且不会同时发起list请求来增加apiserver的压力

    ```go
    // ResyncPeriod returns a function which generates a duration each time it is
    // invoked; this is so that multiple controllers don't get into lock-step and all
    // hammer the apiserver with list requests simultaneously.
    func ResyncPeriod(c *config.CompletedConfig) func() time.Duration {
       return func() time.Duration {
          factor := rand.Float64() + 1
          return time.Duration(float64(c.ComponentConfig.Generic.MinResyncPeriod.Nanoseconds()) * factor)
       }
    }
    ```

  * sharedInformers会new一个SharedInformerFactory，看下SharedInformerFactory的定义，这里其实包含了所有api group verions里资源的共享的informer定义，比如*ctx.InformerFactory.Apps().V1().ReplicaSets()*

    ```go
    // SharedInformerFactory provides shared informers for resources in all known
    // API group versions.
    type SharedInformerFactory interface {
       internalinterfaces.SharedInformerFactory
       ForResource(resource schema.GroupVersionResource) (GenericInformer, error)
       WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool
    
       Admissionregistration() admissionregistration.Interface
       Apps() apps.Interface
       Auditregistration() auditregistration.Interface
       Autoscaling() autoscaling.Interface
       Batch() batch.Interface
       Certificates() certificates.Interface
       Coordination() coordination.Interface
       Core() core.Interface
       Discovery() discovery.Interface
       Events() events.Interface
       Extensions() extensions.Interface
       Flowcontrol() flowcontrol.Interface
       Networking() networking.Interface
       Node() node.Interface
       Policy() policy.Interface
       Rbac() rbac.Interface
       Scheduling() scheduling.Interface
       Settings() settings.Interface
       Storage() storage.Interface
    }
    
    // Interface provides access to each of this group's versions.
    type Interface interface {
    	// V1 provides access to shared informers for resources in V1.
    	V1() v1.Interface
    	// V1beta1 provides access to shared informers for resources in V1beta1.
    	V1beta1() v1beta1.Interface
    	// V1beta2 provides access to shared informers for resources in V1beta2.
    	V1beta2() v1beta2.Interface
    }
    
    // Interface provides access to all the informers in this group version.
    type Interface interface {
    	// ControllerRevisions returns a ControllerRevisionInformer.
    	ControllerRevisions() ControllerRevisionInformer
    	// DaemonSets returns a DaemonSetInformer.
    	DaemonSets() DaemonSetInformer
    	// Deployments returns a DeploymentInformer.
    	Deployments() DeploymentInformer
    	// ReplicaSets returns a ReplicaSetInformer.
    	ReplicaSets() ReplicaSetInformer
    	// StatefulSets returns a StatefulSetInformer.
    	StatefulSets() StatefulSetInformer
    }
    ```

  * 最终会调用SharedInformerFactory的start，这里有个for循环，意味着，上面所有定义在factory里的informers，都会被Run。informer具体run之后发生的故事，留在后面分析reflactor&fifoqueue的时候仔细研究

    ```go
    // Start initializes all requested informers.
    func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
       f.lock.Lock()
       defer f.lock.Unlock()
    
       for informerType, informer := range f.informers {
          if !f.startedInformers[informerType] {
             go informer.Run(stopCh)
             f.startedInformers[informerType] = true
          }
       }
    }
    ```

* ObjectMetadataInformerFactory.Start，ObjectOrMetadataInformerFactory是give access to informers for typed resources and dynamic resources by their metadata

  ```go
  		controllerContext.ObjectOrMetadataInformerFactory.Start(controllerContext.Stop)
  
  	// ObjectOrMetadataInformerFactory gives access to informers for typed resources
  	// and dynamic resources by their metadata. All generic controllers currently use
  	// object metadata - if a future controller needs access to the full object this
  	// would become GenericInformerFactory and take a dynamic client.
  	ObjectOrMetadataInformerFactory controller.InformerFactory
  ```

* 高可用的一些初始化工作，需要LeaderElection，具体先不分析了

# Controller

![image-20191127112249873](/../assets/images/image-20191127112249873.png)

这个图展示了好多遍了。。但是所有的controller起来之后，基本上都是按照这个图来运作的

左侧部分是生产者，右侧是消费者，每个部分都会在其他文章里再具体分析