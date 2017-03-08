---
layout: post 
author: shalou
title:  "k8s的HorizontalPodAutoscaler实现机制" 
category: 容器技术
tag: [k8s,autoscale]
---

*本文基于kubernetes v1.5.1*

## 1. HorizontalPodAutoscaler简介
HorizontalPodAutoscaler是k8s提供的自动扩展机制，他可以根据Pod的负载进行实时的自动伸缩，目前支持cpu这一个指标以及自定义的指标，进行自动伸缩。cpu的检测指标是通过CPU获取的。下面简单介绍其用法

首先创建一个deployments

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: test
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: test
  spec:
    containers:
    - name: test
      image: kubernetes/redis:v1
      resources:
        requests:
          cpu: 100m
          memory: 100Mi
```

使用autoscale命令，创建一个HorizontalPodAutoscaler

```go
kubectl autoscale deployment test --min=2 --max=5 --cpu-percent=70
```

这里我们针对test这个deployment创建一个HorizontalPodAutoscaler，--min指定最小的复制数，--max指定最大的复制数，--cpu-percent指定cpu发起伸缩的阈值。

接下来可以对pod进行压力测试，你会发现Pod会增加，当负载下降时，pod数目会自动较小


**注意**：pod能够进行自动伸缩，必须给pod配置resources.requests.cpu，此外还需要安装heapster这个监控服务。默认的cpu伸缩阈值为80%

## 2. 实现机制

### 2.1 API变化

为了支持自动扩展，k8s首先增加了在extensions这个APIGroup里面增加了三类子资源：

* deployments/scale
* replicationcontrollers/scale
* replicasets/scale

deployments这个资源不用说，位于extensions这个APIGROUP,但replicationcontrollers位于core这个APIGROUP，其实extensions里面也定义了这个API，称作ReplicationControllerDummy，单纯为了scale而存在

```go
type Scale struct {
    unversioned.TypeMeta

    api.ObjectMeta

    Spec ScaleSpec 

    Status ScaleStatus
}

type ScaleSpec struct {
    Replicas int32
}
```

当需要对Pod执行自动扩展时，需要调用replicationcontrollers/scale的UPDATE接口

**pkg/registry/extensions/controller/etcd/etcd.go +73**

```
func (r *ScaleREST) Update(ctx api.Context, name string, objInfo rest.UpdatedObjectInfo) (runtime.Object, bool, error) {
    rc, err := (*r.registry).GetController(ctx, name)
    if err != nil {
        return nil, false, errors.NewNotFound(extensions.Resource("replicationcontrollers/scale"), name)
    }   
    oldScale := scaleFromRC(rc)

    obj, err := objInfo.UpdatedObject(ctx, oldScale)

    if obj == nil {
        return nil, false, errors.NewBadRequest(fmt.Sprintf("nil update passed to Scale"))
    }   
    scale, ok := obj.(*extensions.Scale)
    if !ok {
        return nil, false, errors.NewBadRequest(fmt.Sprintf("wrong object passed to Scale update: %v", obj))
    }   

    if errs := extvalidation.ValidateScale(scale); len(errs) > 0 { 
        return nil, false, errors.NewInvalid(extensions.Kind("Scale"), scale.Name, errs)
    }   

    rc.Spec.Replicas = scale.Spec.Replicas
    rc.ResourceVersion = scale.ResourceVersion
    rc, err = (*r.registry).UpdateController(ctx, rc) 
    if err != nil {
        return nil, false, errors.NewConflict(extensions.Resource("replicationcontrollers/scale"), scale.Name, err)
    }   
    return scaleFromRC(rc), false, nil 
}
```
该接口本质上是通过scale中的name获取相应的replicationcontroller，rc, err := (*r.registry).GetController(ctx, name)。然后比较scale和replicationcontroller的复制数，更新replicationcontroller的复制数，从而引起Pod的伸缩


为了支持自动扩展，k8s还在autoscaling这个APIGROUP引入了HorizontalPodAutoscaler这个对象

```
type HorizontalPodAutoscaler struct {
    unversioned.TypeMeta

    v1.ObjectMeta

    Spec HorizontalPodAutoscalerSpec

    Status HorizontalPodAutoscalerStatus
}


type HorizontalPodAutoscalerSpec struct {
    ScaleTargetRef CrossVersionObjectReference

    MinReplicas *int32

    MaxReplicas int32

    TargetCPUUtilizationPercentage *int32 
}

type CrossVersionObjectReference struct {
    Kind string 
    
    Name string
    
    APIVersion string
}
```

其实这个资源就是我们通过kubectl autoscale创建的对象。HorizontalPodAutoscalerSpec里面有一个ScaleTargetRef的成员，它指定了我们是针对那类资源进行自动扩展，目前支持replicationcontroller、deployment和replicaset这三类资源

### 2.2 客户端逻辑

客户端逻辑主要指kubectl autoscale的实现，主要有RunAutoscale这个函数实现

**pkg/kubectl/cmd/autoscale.go +81**

```go
func RunAutoscale(f cmdutil.Factory, out io.Writer, cmd *cobra.Command, args []string, options *resource.FilenameOptions) error {
    ...
    mapper, typer := f.Object()
    r := resource.NewBuilder(mapper, typer, resource.ClientMapperFunc(f.ClientForMapping), f.Decoder(true)).
        ContinueOnError().
        NamespaceParam(namespace).DefaultNamespace().
        FilenameParam(enforceNamespace, options).
        ResourceTypeOrNameArgs(false, args...).
        Flatten().
        Do()
    ...
    err = r.Visit(func(info *resource.Info, err error) error {
        if err != nil {
            return err
        }

        mapping := info.ResourceMapping()
        if err := f.CanBeAutoscaled(mapping.GroupVersionKind.GroupKind()); err != nil {
            return err
        }

        name := info.Name
        params := kubectl.MakeParams(cmd, names)
        params["default-name"] = name

        params["scaleRef-kind"] = mapping.GroupVersionKind.Kind
        params["scaleRef-name"] = name
        params["scaleRef-apiVersion"] = mapping.GroupVersionKind.GroupVersion().String()

        ...
        object, err := generator.Generate(params) //生成hpa的各个参数
       
        resourceMapper := &resource.Mapper{
            ObjectTyper:  typer,
            RESTMapper:   mapper,
            ClientMapper: resource.ClientMapperFunc(f.ClientForMapping),
            Decoder:      f.Decoder(true),
        }
        hpa, err := resourceMapper.InfoForObject(object, nil)
        if err != nil {
            return err
        }
        if cmdutil.ShouldRecord(cmd, hpa) {
            if err := cmdutil.RecordChangeCause(hpa.Object, f.Command()); err != nil {
                return err
            }
            object = hpa.Object
        }
        if cmdutil.GetDryRunFlag(cmd) {
            return f.PrintObject(cmd, mapper, object, out)
        }

        if err := kubectl.CreateOrUpdateAnnotation(cmdutil.GetFlagBool(cmd, cmdutil.ApplyAnnotationsFlag), hpa, f.JSONEncoder()); err != nil {
            return err
        }

        object, err = resource.NewHelper(hpa.Client, hpa.Mapping).Create(namespace, false, object) //创建hpa
        ...
    })
    ...
}
```

以上函数的逻辑其实主要构建一个horizontalPodAutoscaler对象，持久化到etcd中去

### 2.3 Server端逻辑

Server端的逻辑主要存在于kube-apiserver和kube-controller-manager，kube-apiserver的任务很明显，主要是负责接收客户端的请求，将horizontalPodAutoscaler持久化到etcd中去，主要的控制逻辑放在kube-controller-manager，和实现其他功能类似，kube-controller-manager构建了一个专门的逻辑用来实现Pod的自动扩展

**cmd/kube-controller-manager/app/controllermanager.go +434**

```go
groupVersion = "autoscaling/v1"                                                                                                                                        
resources, found = resourceMap[groupVersion]
glog.Infof("Attempting to start horizontal pod autoscaler controller, full resource map %+v", resourceMap)                                                             
if containsVersion(versions, groupVersion) && found {
    glog.Infof("Starting %s apis", groupVersion)
    if containsResource(resources, "horizontalpodautoscalers") {
        glog.Infof("Starting horizontal pod controller.")
        hpaClient := client("horizontal-pod-autoscaler")
        metricsClient := metrics.NewHeapsterMetricsClient(
            hpaClient,            
            metrics.DefaultHeapsterNamespace,
            metrics.DefaultHeapsterScheme,
            metrics.DefaultHeapsterService,
            metrics.DefaultHeapsterPort,
        )
        replicaCalc := podautoscaler.NewReplicaCalculator(metricsClient, hpaClient.Core())                                                                             
        go podautoscaler.NewHorizontalController(hpaClient.Core(), hpaClient.Extensions(), hpaClient.Autoscaling(), replicaCalc, s.HorizontalPodAutoscalerSyncPeriod.Duration).
            Run(wait.NeverStop)
        time.Sleep(wait.Jitter(s.ControllerStartInterval.Duration, ControllerStartJitter))                                                                             
    }
}
```
NewHorizontalController创建了一个控制器，传入了5个参数：hpaClient.Core()表示Core这个apigroup的client，hpaClient.Extensions()表示extensions这个apigroup的client，hpaClient.Autoscaling()表示autoscaling这个apigroup的client，replicaCalc用来计算Pod的复制数，实现自动扩展，s.HorizontalPodAutoscalerSyncPeriod.Duration默认的值为30秒，也就是每个30s会执行一次自动扩展。

replicaCalc的构建传入了一个metricsClient对象，metricsClient其实就是相当于heapster的客户端，用于从heapster哪里获得自动扩展的各项指标。我们主要关注HorizontalController这个控制器

**pkg/controller/podautoscaler/horizontal.go +112**

```go
func NewHorizontalController(evtNamespacer unversionedcore.EventsGetter, scaleNamespacer unversionedextensions.ScalesGetter, hpaNamespacer unversionedautoscaling.HorizontalPodAutoscalersGetter, replicaCalc *ReplicaCalculator, resyncPeriod time.Duration) *HorizontalController {
    broadcaster := record.NewBroadcaster()
    broadcaster.StartRecordingToSink(&unversionedcore.EventSinkImpl{Interface: evtNamespacer.Events("")})
    recorder := broadcaster.NewRecorder(api.EventSource{Component: "horizontal-pod-autoscaler"})

    controller := &HorizontalController{
        replicaCalc:     replicaCalc,
        eventRecorder:   recorder,
        scaleNamespacer: scaleNamespacer,
        hpaNamespacer:   hpaNamespacer,
    }   
    store, frameworkController := newInformer(controller, resyncPeriod)
    controller.store = store
    controller.controller = frameworkController

    return controller
}

func (a *HorizontalController) Run(stopCh <-chan struct{}) {
    defer utilruntime.HandleCrash()
    glog.Infof("Starting HPA Controller")
    go a.controller.Run(stopCh)
    <-stopCh
    glog.Infof("Shutting down HPA Controller")
}
```
构建了一个HorizontalController，HorizontalController还有一个成员为controller，这个是真正的控制逻辑，利用newInformer函数创建

**pkg/controller/podautoscaler/horizontal.go +75**

```
func newInformer(controller *HorizontalController, resyncPeriod time.Duration) (cache.Store, *cache.Controller) {
    return cache.NewInformer(
        &cache.ListWatch{
            ListFunc: func(options api.ListOptions) (runtime.Object, error) {
                return controller.hpaNamespacer.HorizontalPodAutoscalers(api.NamespaceAll).List(options)
            },  
            WatchFunc: func(options api.ListOptions) (watch.Interface, error) {
                return controller.hpaNamespacer.HorizontalPodAutoscalers(api.NamespaceAll).Watch(options)
            },  
        },  
        &autoscaling.HorizontalPodAutoscaler{},
        resyncPeriod,
        cache.ResourceEventHandlerFuncs{
            AddFunc: func(obj interface{}) {
                hpa := obj.(*autoscaling.HorizontalPodAutoscaler)
                hasCPUPolicy := hpa.Spec.TargetCPUUtilizationPercentage != nil 
                _, hasCustomMetricsPolicy := hpa.Annotations[HpaCustomMetricsTargetAnnotationName]
                if !hasCPUPolicy && !hasCustomMetricsPolicy {
                    controller.eventRecorder.Event(hpa, api.EventTypeNormal, "DefaultPolicy", "No scaling policy specified - will use default one. See documentation for details")
                }   
                err := controller.reconcileAutoscaler(hpa)
                if err != nil {
                    glog.Warningf("Failed to reconcile %s: %v", hpa.Name, err)
                }   
            },  
            UpdateFunc: func(old, cur interface{}) {
                hpa := cur.(*autoscaling.HorizontalPodAutoscaler)
                err := controller.reconcileAutoscaler(hpa)
                if err != nil {
                    glog.Warningf("Failed to reconcile %s: %v", hpa.Name, err)
                }
            },
            // We are not interested in deletions.
        },
    )
}
```

该控制器主要负责监听HorizontalPodAutoscalers对象，当出现该对象的增加、更新时，调用reconcileAutoscaler函数去协调，当出现删除时，什么事情也不做

**pkg/controller/podautoscaler/horizontal.go +272**

```go
func (a *HorizontalController) reconcileAutoscaler(hpa *autoscaling.HorizontalPodAutoscaler) error {
    reference := fmt.Sprintf("%s/%s/%s", hpa.Spec.ScaleTargetRef.Kind, hpa.Namespace, hpa.Spec.ScaleTargetRef.Name)
    
    //利用hpa，获取scale子资源
    scale, err := a.scaleNamespacer.Scales(hpa.Namespace).Get(hpa.Spec.ScaleTargetRef.Kind, hpa.Spec.ScaleTargetRef.Name)
    ...  
    currentReplicas := scale.Status.Replicas

    cpuDesiredReplicas := int32(0)
    cpuCurrentUtilization := new(int32)
    cpuTimestamp := time.Time{}

    cmDesiredReplicas := int32(0)
    cmMetric := ""
    cmStatus := ""
    cmTimestamp := time.Time{}

    desiredReplicas := int32(0)
    rescaleReason := ""
    timestamp := time.Now()

    rescale := true

    if scale.Spec.Replicas == 0 { 
        // 假如scale的复制数为0，则说明不需要scale
        desiredReplicas = 0 
        rescale = false
    } else if currentReplicas > hpa.Spec.MaxReplicas {
        //现有复制数大于最大复制数，必须scale
        desiredReplicas = hpa.Spec.MaxReplicas
    } else if hpa.Spec.MinReplicas != nil && currentReplicas < *hpa.Spec.MinReplicas {
        //先有复制数小于最小复制数，必须scale
        desiredReplicas = *hpa.Spec.MinReplicas
    } else if currentReplicas == 0 { 
        //hpa对应的pod复制数必须大于0，因此为0时，需要扩展到1
        desiredReplicas = 1 
    } else {
        //查看annotation，看是否有自定义的metrics
        cmAnnotation, cmAnnotationFound := hpa.Annotations[HpaCustomMetricsTargetAnnotationName]

        if hpa.Spec.TargetCPUUtilizationPercentage != nil || !cmAnnotationFound {
            //根据cpu指标，计算scale的备份数
            cpuDesiredReplicas, cpuCurrentUtilization, cpuTimestamp, err = a.computeReplicasForCPUUtilization(hpa, scale)
            ...
        }

        if cmAnnotationFound {
            //根据自定义的指标，计算scale的备份数
            cmDesiredReplicas, cmMetric, cmStatus, cmTimestamp, err = a.computeReplicasForCustomMetrics(hpa, scale, cmAnnotation)
            ...
        }

        rescaleMetric := ""
        if cpuDesiredReplicas > desiredReplicas {
            desiredReplicas = cpuDesiredReplicas
            timestamp = cpuTimestamp
            rescaleMetric = "CPU utilization"
        }
        if cmDesiredReplicas > desiredReplicas {
            desiredReplicas = cmDesiredReplicas
            timestamp = cmTimestamp
            rescaleMetric = cmMetric
        }
        if desiredReplicas > currentReplicas {
            rescaleReason = fmt.Sprintf("%s above target", rescaleMetric)
        }
        if desiredReplicas < currentReplicas {
            rescaleReason = "All metrics below target"
        }

        if hpa.Spec.MinReplicas != nil && desiredReplicas < *hpa.Spec.MinReplicas {
            desiredReplicas = *hpa.Spec.MinReplicas
        }

        //  never scale down to 0, reserved for disabling autoscaling
        if desiredReplicas == 0 {
            desiredReplicas = 1
        }

        if desiredReplicas > hpa.Spec.MaxReplicas {
            desiredReplicas = hpa.Spec.MaxReplicas
        }

        if desiredReplicas > calculateScaleUpLimit(currentReplicas) {
            desiredReplicas = calculateScaleUpLimit(currentReplicas)
        }

        //查看是否允许扩展，距离上次向下扩展(downscale)5分钟以内或向上扩展(upscale)3分钟以内，禁止scale，主要为了减小误差
        rescale = shouldScale(hpa, currentReplicas, desiredReplicas, timestamp)
    }

    if rescale {
        //调用scale子资源的update接口进行扩展，我们在2.1小节里面介绍过
        scale.Spec.Replicas = desiredReplicas
        _, err = a.scaleNamespacer.Scales(hpa.Namespace).Update(hpa.Spec.ScaleTargetRef.Kind, scale)
        ...
    } else {
        desiredReplicas = currentReplicas
    }
    ...
}
```

从上面的分析可以知道computeReplicasForCPUUtilization和computeReplicasForCustomMetrics两个函数很重要，他们用来计算Pod的复制数，这里我们以CPU指标为例

**pkg/controller/podautoscaler/horizontal.go +147**

```
func (a *HorizontalController) computeReplicasForCPUUtilization(hpa *autoscaling.HorizontalPodAutoscaler, scale *extensions.Scale) (int32, *int32, time.Time, error) {
    //默认的CPU阈值为80%
    targetUtilization := int32(defaultTargetCPUUtilizationPercentage)
    if hpa.Spec.TargetCPUUtilizationPercentage != nil {
        targetUtilization = *hpa.Spec.TargetCPUUtilizationPercentage
    }   
    currentReplicas := scale.Status.Replicas
    ...
    selector, err := unversioned.LabelSelectorAsSelector(scale.Status.Selector)
    ... 
    desiredReplicas, utilization, timestamp, err := a.replicaCalc.GetResourceReplicas(currentReplicas, targetUtilization, api.ResourceCPU, hpa.Namespace, selector)
    ...
    return desiredReplicas, &utilization, timestamp, nil
}
```

**pkg/controller/podautoscaler/replica_calculator.go +45**

```go
func (c *ReplicaCalculator) GetResourceReplicas(currentReplicas int32, targetUtilization int32, resource api.ResourceName, namespace string, selector labels.Selector) (replicaCount int32, utilization int32, timestamp time.Time, err error) {
    //获取metrics
    metrics, timestamp, err := c.metricsClient.GetResourceMetric(resource, namespace, selector)
    ...
    podList, err := c.podsGetter.Pods(namespace).List(api.ListOptions{LabelSelector: selector})
    ...  
    requests := make(map[string]int64, len(podList.Items))
    readyPodCount := 0
    unreadyPods := sets.NewString()
    missingPods := sets.NewString()

    for _, pod := range podList.Items {
        podSum := int64(0)
        for _, container := range pod.Spec.Containers {
            if containerRequest, ok := container.Resources.Requests[resource]; ok {
                podSum += containerRequest.MilliValue()
            } else {
                return 0, 0, time.Time{}, fmt.Errorf("missing request for %s on container %s in pod %s/%s", resource, container.Name, namespace, pod.Name)
            }   
        }
        
        requests[pod.Name] = podSum

        if pod.Status.Phase != api.PodRunning || !api.IsPodReady(&pod) {
            unreadyPods.Insert(pod.Name)
            delete(metrics, pod.Name)
            continue
        }

        if _, found := metrics[pod.Name]; !found {
            missingPods.Insert(pod.Name)
            continue
        }

        readyPodCount++
    }

    ...

    usageRatio, utilization, err := metricsclient.GetResourceUtilizationRatio(metrics, requests, targetUtilization)
    if err != nil {
        return 0, 0, time.Time{}, err
    }

    rebalanceUnready := len(unreadyPods) > 0 && usageRatio > 1.0
    if !rebalanceUnready && len(missingPods) == 0 {
        //当usageRatio大于0.9且小于1.1时，认为这是干扰，不做调整
        if math.Abs(1.0-usageRatio) <= tolerance {
            // return the current replicas if the change would be too small
            return currentReplicas, utilization, timestamp, nil
        }

        return int32(math.Ceil(usageRatio * float64(readyPodCount))), utilization, timestamp, nil
    }

    if len(missingPods) > 0 {
        if usageRatio < 1.0 {
            // on a scale-down, treat missing pods as using 100% of the resource request
            for podName := range missingPods {
                metrics[podName] = requests[podName]
            }
        } else if usageRatio > 1.0 {
            // on a scale-up, treat missing pods as using 0% of the resource request
            for podName := range missingPods {
                metrics[podName] = 0
            }
        }
    }

    if rebalanceUnready {
        // on a scale-up, treat unready pods as using 0% of the resource request
        for podName := range unreadyPods {
            metrics[podName] = 0
        }
    }

    // re-run the utilization calculation with our new numbers
    newUsageRatio, _, err := metricsclient.GetResourceUtilizationRatio(metrics, requests, targetUtilization)
    if err != nil {
        return 0, utilization, time.Time{}, err
    }

    if math.Abs(1.0-newUsageRatio) <= tolerance || (usageRatio < 1.0 && newUsageRatio > 1.0) || (usageRatio > 1.0 && newUsageRatio < 1.0) {
        return currentReplicas, utilization, timestamp, nil
    }

    return int32(math.Ceil(newUsageRatio * float64(len(metrics)))), utilization, timestamp, nil
}
```

scale的策略还是挺有意思的，当usageRatio位于0.9和1.1之间的，认为这是干扰，因此不调整，而当出现有Pod不健康或者没有获取到Pod的metrics，根据是upscale或downscale采取的策略也不一致。

我们比较感兴趣的是如何向heapster获取CPU指标的，由c.metricsClient.GetResourceMetric函数实现，我们看看该函数，这个函数用到了一类特殊的restful接口，即proxy

**pkg/controller/podautoscaler/metrics/metrics_client.go +83**

```go
func (h *HeapsterMetricsClient) GetResourceMetric(resource api.ResourceName, namespace string, selector labels.Selector) (PodResourceInfo, time.Time, error) {
    //构建metricPath
    metricPath := fmt.Sprintf("/apis/metrics/v1alpha1/namespaces/%s/pods", namespace)
    params := map[string]string{"labelSelector": selector.String()}

    //发送请求获取，获取metrics
    resultRaw, err := h.services.
        ProxyGet(h.heapsterScheme, h.heapsterService, h.heapsterPort, metricPath, params).
        DoRaw()
    ...  

    glog.V(4).Infof("Heapster metrics result: %s", string(resultRaw))

    metrics := metrics_api.PodMetricsList{}
    err = json.Unmarshal(resultRaw, &metrics)
    if err != nil {
        return nil, time.Time{}, fmt.Errorf("failed to unmarshal heapster response: %v", err)
    }   

    if len(metrics.Items) == 0 { 
        return nil, time.Time{}, fmt.Errorf("no metrics returned from heapster")
    }   

    res := make(PodResourceInfo, len(metrics.Items))

    for _, m := range metrics.Items {
        podSum := int64(0)
        missing := len(m.Containers) == 0
        for _, c := range m.Containers {
            resValue, found := c.Usage[v1.ResourceName(resource)]
            if !found {
                missing = true
                glog.V(2).Infof("missing resource metric %v for container %s in pod %s/%s", resource, c.Name, namespace, m.Name)
                continue
            }   
            podSum += resValue.MilliValue()
        }   

        if !missing {
            res[m.Name] = int64(podSum)
        }   
    }
    timestamp := time.Time{}
    if len(metrics.Items) > 0 {
        timestamp = metrics.Items[0].Timestamp.Time
    }

    return res, timestamp, nil
}
```

h.services.
        ProxyGet(h.heapsterScheme, h.heapsterService, h.heapsterPort, metricPath, params).
        DoRaw()表示的就是proxy请求，之所以称为proxy，是因为我们向kube-apiserver发出请求，kube-apiserver将请求发送至heapster这个service。它本质上发送了以下请求
 
```go    
curl http://127.0.0.1:8080/api/v1/namespaces/kube-system/services/heapster/proxy/apis/metrics/v1alpha1/namespaces/upchat/pods
```

他表示像kube-system下的heapster服务发起请求，proxy的url路径为/apis/metrics/v1alpha1/namespaces/upchat/pods，即获取upchat这个namespaces的Pod的指标，实际上还可以传入params,params包括selector,即只请求我们感兴趣的Pod。

目前只有services资源实现了proxy接口


## 3. 总结
本文我们介绍了k8s中HorizontalPodAutoscaler的实现机制，其实现机制和k8s中其他功能的实现差不多，都是把主要的逻辑专门作为kube-controller-manager的控制器实现。其功能有如下特点：

* 自动扩展的指标包括cpu和自定义的指标
* 每隔30秒，就会执行一次自动扩展逻辑
* 对于扩展率位于0.9和1.1中的，认为是干扰，因此不做扩展
* updown的三分钟之内或downscale的五分钟内，不做扩展
* 每次扩展的数目不应超过现有备份数的两倍，不然会照成统计数据的不准
