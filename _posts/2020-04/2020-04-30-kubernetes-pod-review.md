---
layout: post
title:  "Kubernetes Pod复习及结构分析"
date:   2020-04-30 22:03:00 +0800
categories: Kubernetes
tags: Kubernetes-Pod
excerpt: Kubernetes Pod复习及结构分析
mathjax: true
typora-root-url: ../
---

# Kubernetes Pod struct

```go
// Pod is a collection of containers that can run on a host. This resource is created
// by clients and scheduled onto hosts.
type Pod struct {
   metav1.TypeMeta `json:",inline"`
   // Standard object's metadata.
   // More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
   // +optional
   metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

   // Specification of the desired behavior of the pod.
   // More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
   // +optional
   Spec PodSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`

   // Most recently observed status of the pod.
   // This data may not be up to date.
   // Populated by the system.
   // Read-only.
   // More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
   // +optional
   Status PodStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
```

根据Pod的定义可以看到，分成三个部分：

* metadata
* spec
* status

## metadata

### TypeMeta

```go
// TypeMeta describes an individual object in an API response or request
// with strings representing the type of the object and its API schema version.
// Structures that are versioned or persisted should inline TypeMeta.
//
// +k8s:deepcopy-gen=false
type TypeMeta struct {
   // Kind is a string value representing the REST resource this object represents.
   // Servers may infer this from the endpoint the client submits requests to.
   // Cannot be updated.
   // In CamelCase.
   // More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds
   // +optional
   Kind string `json:"kind,omitempty" protobuf:"bytes,1,opt,name=kind"`

   // APIVersion defines the versioned schema of this representation of an object.
   // Servers should convert recognized schemas to the latest internal value, and
   // may reject unrecognized values.
   // More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources
   // +optional
   APIVersion string `json:"apiVersion,omitempty" protobuf:"bytes,2,opt,name=apiVersion"`
}
```

这部分是Kind和APIVersion：

* Kind：resource的类型
* APIVersion：resource版本

### ObjectMeta

```go
// ObjectMeta is metadata that all persisted resources must have, which includes all objects
// users must create.
type ObjectMeta struct {
   // Name must be unique within a namespace. Is required when creating resources, although
   // some resources may allow a client to request the generation of an appropriate name
   // automatically. Name is primarily intended for creation idempotence and configuration
   // definition.
   // Cannot be updated.
   // More info: http://kubernetes.io/docs/user-guide/identifiers#names
   // +optional
   Name string `json:"name,omitempty" protobuf:"bytes,1,opt,name=name"`

   // GenerateName is an optional prefix, used by the server, to generate a unique
   // name ONLY IF the Name field has not been provided.
   // If this field is used, the name returned to the client will be different
   // than the name passed. This value will also be combined with a unique suffix.
   // The provided value has the same validation rules as the Name field,
   // and may be truncated by the length of the suffix required to make the value
   // unique on the server.
   //
   // If this field is specified and the generated name exists, the server will
   // NOT return a 409 - instead, it will either return 201 Created or 500 with Reason
   // ServerTimeout indicating a unique name could not be found in the time allotted, and the client
   // should retry (optionally after the time indicated in the Retry-After header).
   //
   // Applied only if Name is not specified.
   // More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#idempotency
   // +optional
   GenerateName string `json:"generateName,omitempty" protobuf:"bytes,2,opt,name=generateName"`

   // Namespace defines the space within each name must be unique. An empty namespace is
   // equivalent to the "default" namespace, but "default" is the canonical representation.
   // Not all objects are required to be scoped to a namespace - the value of this field for
   // those objects will be empty.
   //
   // Must be a DNS_LABEL.
   // Cannot be updated.
   // More info: http://kubernetes.io/docs/user-guide/namespaces
   // +optional
   Namespace string `json:"namespace,omitempty" protobuf:"bytes,3,opt,name=namespace"`

   // SelfLink is a URL representing this object.
   // Populated by the system.
   // Read-only.
   //
   // DEPRECATED
   // Kubernetes will stop propagating this field in 1.20 release and the field is planned
   // to be removed in 1.21 release.
   // +optional
   SelfLink string `json:"selfLink,omitempty" protobuf:"bytes,4,opt,name=selfLink"`

   // UID is the unique in time and space value for this object. It is typically generated by
   // the server on successful creation of a resource and is not allowed to change on PUT
   // operations.
   //
   // Populated by the system.
   // Read-only.
   // More info: http://kubernetes.io/docs/user-guide/identifiers#uids
   // +optional
   UID types.UID `json:"uid,omitempty" protobuf:"bytes,5,opt,name=uid,casttype=k8s.io/kubernetes/pkg/types.UID"`

   // An opaque value that represents the internal version of this object that can
   // be used by clients to determine when objects have changed. May be used for optimistic
   // concurrency, change detection, and the watch operation on a resource or set of resources.
   // Clients must treat these values as opaque and passed unmodified back to the server.
   // They may only be valid for a particular resource or set of resources.
   //
   // Populated by the system.
   // Read-only.
   // Value must be treated as opaque by clients and .
   // More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#concurrency-control-and-consistency
   // +optional
   ResourceVersion string `json:"resourceVersion,omitempty" protobuf:"bytes,6,opt,name=resourceVersion"`

   // A sequence number representing a specific generation of the desired state.
   // Populated by the system. Read-only.
   // +optional
   Generation int64 `json:"generation,omitempty" protobuf:"varint,7,opt,name=generation"`

   // CreationTimestamp is a timestamp representing the server time when this object was
   // created. It is not guaranteed to be set in happens-before order across separate operations.
   // Clients may not set this value. It is represented in RFC3339 form and is in UTC.
   //
   // Populated by the system.
   // Read-only.
   // Null for lists.
   // More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
   // +optional
   CreationTimestamp Time `json:"creationTimestamp,omitempty" protobuf:"bytes,8,opt,name=creationTimestamp"`

   // DeletionTimestamp is RFC 3339 date and time at which this resource will be deleted. This
   // field is set by the server when a graceful deletion is requested by the user, and is not
   // directly settable by a client. The resource is expected to be deleted (no longer visible
   // from resource lists, and not reachable by name) after the time in this field, once the
   // finalizers list is empty. As long as the finalizers list contains items, deletion is blocked.
   // Once the deletionTimestamp is set, this value may not be unset or be set further into the
   // future, although it may be shortened or the resource may be deleted prior to this time.
   // For example, a user may request that a pod is deleted in 30 seconds. The Kubelet will react
   // by sending a graceful termination signal to the containers in the pod. After that 30 seconds,
   // the Kubelet will send a hard termination signal (SIGKILL) to the container and after cleanup,
   // remove the pod from the API. In the presence of network partitions, this object may still
   // exist after this timestamp, until an administrator or automated process can determine the
   // resource is fully terminated.
   // If not set, graceful deletion of the object has not been requested.
   //
   // Populated by the system when a graceful deletion is requested.
   // Read-only.
   // More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
   // +optional
   DeletionTimestamp *Time `json:"deletionTimestamp,omitempty" protobuf:"bytes,9,opt,name=deletionTimestamp"`

   // Number of seconds allowed for this object to gracefully terminate before
   // it will be removed from the system. Only set when deletionTimestamp is also set.
   // May only be shortened.
   // Read-only.
   // +optional
   DeletionGracePeriodSeconds *int64 `json:"deletionGracePeriodSeconds,omitempty" protobuf:"varint,10,opt,name=deletionGracePeriodSeconds"`

   // Map of string keys and values that can be used to organize and categorize
   // (scope and select) objects. May match selectors of replication controllers
   // and services.
   // More info: http://kubernetes.io/docs/user-guide/labels
   // +optional
   Labels map[string]string `json:"labels,omitempty" protobuf:"bytes,11,rep,name=labels"`

   // Annotations is an unstructured key value map stored with a resource that may be
   // set by external tools to store and retrieve arbitrary metadata. They are not
   // queryable and should be preserved when modifying objects.
   // More info: http://kubernetes.io/docs/user-guide/annotations
   // +optional
   Annotations map[string]string `json:"annotations,omitempty" protobuf:"bytes,12,rep,name=annotations"`

   // List of objects depended by this object. If ALL objects in the list have
   // been deleted, this object will be garbage collected. If this object is managed by a controller,
   // then an entry in this list will point to this controller, with the controller field set to true.
   // There cannot be more than one managing controller.
   // +optional
   // +patchMergeKey=uid
   // +patchStrategy=merge
   OwnerReferences []OwnerReference `json:"ownerReferences,omitempty" patchStrategy:"merge" patchMergeKey:"uid" protobuf:"bytes,13,rep,name=ownerReferences"`

   // Must be empty before the object is deleted from the registry. Each entry
   // is an identifier for the responsible component that will remove the entry
   // from the list. If the deletionTimestamp of the object is non-nil, entries
   // in this list can only be removed.
   // Finalizers may be processed and removed in any order.  Order is NOT enforced
   // because it introduces significant risk of stuck finalizers.
   // finalizers is a shared field, any actor with permission can reorder it.
   // If the finalizer list is processed in order, then this can lead to a situation
   // in which the component responsible for the first finalizer in the list is
   // waiting for a signal (field value, external system, or other) produced by a
   // component responsible for a finalizer later in the list, resulting in a deadlock.
   // Without enforced ordering finalizers are free to order amongst themselves and
   // are not vulnerable to ordering changes in the list.
   // +optional
   // +patchStrategy=merge
   Finalizers []string `json:"finalizers,omitempty" patchStrategy:"merge" protobuf:"bytes,14,rep,name=finalizers"`

   // The name of the cluster which the object belongs to.
   // This is used to distinguish resources with same name and namespace in different clusters.
   // This field is not set anywhere right now and apiserver is going to ignore it if set in create or update request.
   // +optional
   ClusterName string `json:"clusterName,omitempty" protobuf:"bytes,15,opt,name=clusterName"`

   // ManagedFields maps workflow-id and version to the set of fields
   // that are managed by that workflow. This is mostly for internal
   // housekeeping, and users typically shouldn't need to set or
   // understand this field. A workflow can be the user's name, a
   // controller's name, or the name of a specific apply path like
   // "ci-cd". The set of fields is always in the version that the
   // workflow used when modifying the object.
   //
   // +optional
   ManagedFields []ManagedFieldsEntry `json:"managedFields,omitempty" protobuf:"bytes,17,rep,name=managedFields"`
}
```

这部分是一些原数据

* Name：在一个namespace里Pod Name是唯一的（不能被更新）
* GenerateName：如果没有提供Name字段，server会自动提供一个唯一的generatename
* Namespace：定义在哪个space下name应该是唯一的（如果不提供，默认在default namespace下）
* SelfLink：（1.21会被remove）
* UID：对象被成功创建后，由系统分配的unique in time and space value（不能被更新）
* ResourceVersion：系统生成的对象内部的version，client可以用来查看对象是否被changed
* Generation：系统生成的，只读的，代表specific generation of the desired state的序列号
* CreationTimeStamp：系统生成的，只读的，对象创建的时间
* DeletionTimestamp：当对象graceful deletion is requested的时候由系统生成，只读（这个时间代表了resource将会被删除的时间，当graceful deletion被请求的时候，在这个时间点后，一旦finalizers list是空了，对象就会被删除，resource list里已经查询不到，通过name也查询不到了。但是finalizers list如果还存在items，那么删除就被会blocked。一旦DeletionTimestamp被设置了，就不能被unset或者修改，虽然真正的删除时间可能会不一样）
* DeletionGracePeriodSeconds：允许的gracefully terminate的时间（秒），只在DeletionTimestamp被设置的时候才会被设置
* Labels：key-value的map，organize and categorize (scope and select) objects
* Annotations：非结构化的key-value map，由外部工具设置以存储和检索任意元数据。不能被查询，修改对象的时候这个也会保留
* OwnerReferences：该对象所依赖的对象列表。如果对象列表中所有的对象都被删除了，那么这个对象会被垃圾回收。如果对象是被controller管理的，那么这里会指向controller
*  Finalizers：当 Finalizers字段存在时，相关资源不允许被强制删除。存在 Finalizers 字段的的资源对象接收的第一个删除请求设置deletionTimestamp 字段的值，但不删除具体资源，在该字段设置后， finalizer 列表中的对象只能被删除，不能做其他操作
* ClusterName：对象归属的cluster
* ManagedFields：

## PodSpec

```go
// PodSpec is a description of a pod.
type PodSpec struct {
   // List of volumes that can be mounted by containers belonging to the pod.
   // More info: https://kubernetes.io/docs/concepts/storage/volumes
   // +optional
   // +patchMergeKey=name
   // +patchStrategy=merge,retainKeys
   Volumes []Volume `json:"volumes,omitempty" patchStrategy:"merge,retainKeys" patchMergeKey:"name" protobuf:"bytes,1,rep,name=volumes"`
   // List of initialization containers belonging to the pod.
   // Init containers are executed in order prior to containers being started. If any
   // init container fails, the pod is considered to have failed and is handled according
   // to its restartPolicy. The name for an init container or normal container must be
   // unique among all containers.
   // Init containers may not have Lifecycle actions, Readiness probes, Liveness probes, or Startup probes.
   // The resourceRequirements of an init container are taken into account during scheduling
   // by finding the highest request/limit for each resource type, and then using the max of
   // of that value or the sum of the normal containers. Limits are applied to init containers
   // in a similar fashion.
   // Init containers cannot currently be added or removed.
   // Cannot be updated.
   // More info: https://kubernetes.io/docs/concepts/workloads/pods/init-containers/
   // +patchMergeKey=name
   // +patchStrategy=merge
   InitContainers []Container `json:"initContainers,omitempty" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,20,rep,name=initContainers"`
   // List of containers belonging to the pod.
   // Containers cannot currently be added or removed.
   // There must be at least one container in a Pod.
   // Cannot be updated.
   // +patchMergeKey=name
   // +patchStrategy=merge
   Containers []Container `json:"containers" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,2,rep,name=containers"`
   // List of ephemeral containers run in this pod. Ephemeral containers may be run in an existing
   // pod to perform user-initiated actions such as debugging. This list cannot be specified when
   // creating a pod, and it cannot be modified by updating the pod spec. In order to add an
   // ephemeral container to an existing pod, use the pod's ephemeralcontainers subresource.
   // This field is alpha-level and is only honored by servers that enable the EphemeralContainers feature.
   // +optional
   // +patchMergeKey=name
   // +patchStrategy=merge
   EphemeralContainers []EphemeralContainer `json:"ephemeralContainers,omitempty" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,34,rep,name=ephemeralContainers"`
   // Restart policy for all containers within the pod.
   // One of Always, OnFailure, Never.
   // Default to Always.
   // More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy
   // +optional
   RestartPolicy RestartPolicy `json:"restartPolicy,omitempty" protobuf:"bytes,3,opt,name=restartPolicy,casttype=RestartPolicy"`
   // Optional duration in seconds the pod needs to terminate gracefully. May be decreased in delete request.
   // Value must be non-negative integer. The value zero indicates delete immediately.
   // If this value is nil, the default grace period will be used instead.
   // The grace period is the duration in seconds after the processes running in the pod are sent
   // a termination signal and the time when the processes are forcibly halted with a kill signal.
   // Set this value longer than the expected cleanup time for your process.
   // Defaults to 30 seconds.
   // +optional
   TerminationGracePeriodSeconds *int64 `json:"terminationGracePeriodSeconds,omitempty" protobuf:"varint,4,opt,name=terminationGracePeriodSeconds"`
   // Optional duration in seconds the pod may be active on the node relative to
   // StartTime before the system will actively try to mark it failed and kill associated containers.
   // Value must be a positive integer.
   // +optional
   ActiveDeadlineSeconds *int64 `json:"activeDeadlineSeconds,omitempty" protobuf:"varint,5,opt,name=activeDeadlineSeconds"`
   // Set DNS policy for the pod.
   // Defaults to "ClusterFirst".
   // Valid values are 'ClusterFirstWithHostNet', 'ClusterFirst', 'Default' or 'None'.
   // DNS parameters given in DNSConfig will be merged with the policy selected with DNSPolicy.
   // To have DNS options set along with hostNetwork, you have to specify DNS policy
   // explicitly to 'ClusterFirstWithHostNet'.
   // +optional
   DNSPolicy DNSPolicy `json:"dnsPolicy,omitempty" protobuf:"bytes,6,opt,name=dnsPolicy,casttype=DNSPolicy"`
   // NodeSelector is a selector which must be true for the pod to fit on a node.
   // Selector which must match a node's labels for the pod to be scheduled on that node.
   // More info: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
   // +optional
   NodeSelector map[string]string `json:"nodeSelector,omitempty" protobuf:"bytes,7,rep,name=nodeSelector"`

   // ServiceAccountName is the name of the ServiceAccount to use to run this pod.
   // More info: https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/
   // +optional
   ServiceAccountName string `json:"serviceAccountName,omitempty" protobuf:"bytes,8,opt,name=serviceAccountName"`
   // DeprecatedServiceAccount is a depreciated alias for ServiceAccountName.
   // Deprecated: Use serviceAccountName instead.
   // +k8s:conversion-gen=false
   // +optional
   DeprecatedServiceAccount string `json:"serviceAccount,omitempty" protobuf:"bytes,9,opt,name=serviceAccount"`
   // AutomountServiceAccountToken indicates whether a service account token should be automatically mounted.
   // +optional
   AutomountServiceAccountToken *bool `json:"automountServiceAccountToken,omitempty" protobuf:"varint,21,opt,name=automountServiceAccountToken"`

   // NodeName is a request to schedule this pod onto a specific node. If it is non-empty,
   // the scheduler simply schedules this pod onto that node, assuming that it fits resource
   // requirements.
   // +optional
   NodeName string `json:"nodeName,omitempty" protobuf:"bytes,10,opt,name=nodeName"`
   // Host networking requested for this pod. Use the host's network namespace.
   // If this option is set, the ports that will be used must be specified.
   // Default to false.
   // +k8s:conversion-gen=false
   // +optional
   HostNetwork bool `json:"hostNetwork,omitempty" protobuf:"varint,11,opt,name=hostNetwork"`
   // Use the host's pid namespace.
   // Optional: Default to false.
   // +k8s:conversion-gen=false
   // +optional
   HostPID bool `json:"hostPID,omitempty" protobuf:"varint,12,opt,name=hostPID"`
   // Use the host's ipc namespace.
   // Optional: Default to false.
   // +k8s:conversion-gen=false
   // +optional
   HostIPC bool `json:"hostIPC,omitempty" protobuf:"varint,13,opt,name=hostIPC"`
   // Share a single process namespace between all of the containers in a pod.
   // When this is set containers will be able to view and signal processes from other containers
   // in the same pod, and the first process in each container will not be assigned PID 1.
   // HostPID and ShareProcessNamespace cannot both be set.
   // Optional: Default to false.
   // This field is beta-level and may be disabled with the PodShareProcessNamespace feature.
   // +k8s:conversion-gen=false
   // +optional
   ShareProcessNamespace *bool `json:"shareProcessNamespace,omitempty" protobuf:"varint,27,opt,name=shareProcessNamespace"`
   // SecurityContext holds pod-level security attributes and common container settings.
   // Optional: Defaults to empty.  See type description for default values of each field.
   // +optional
   SecurityContext *PodSecurityContext `json:"securityContext,omitempty" protobuf:"bytes,14,opt,name=securityContext"`
   // ImagePullSecrets is an optional list of references to secrets in the same namespace to use for pulling any of the images used by this PodSpec.
   // If specified, these secrets will be passed to individual puller implementations for them to use. For example,
   // in the case of docker, only DockerConfig type secrets are honored.
   // More info: https://kubernetes.io/docs/concepts/containers/images#specifying-imagepullsecrets-on-a-pod
   // +optional
   // +patchMergeKey=name
   // +patchStrategy=merge
   ImagePullSecrets []LocalObjectReference `json:"imagePullSecrets,omitempty" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,15,rep,name=imagePullSecrets"`
   // Specifies the hostname of the Pod
   // If not specified, the pod's hostname will be set to a system-defined value.
   // +optional
   Hostname string `json:"hostname,omitempty" protobuf:"bytes,16,opt,name=hostname"`
   // If specified, the fully qualified Pod hostname will be "<hostname>.<subdomain>.<pod namespace>.svc.<cluster domain>".
   // If not specified, the pod will not have a domainname at all.
   // +optional
   Subdomain string `json:"subdomain,omitempty" protobuf:"bytes,17,opt,name=subdomain"`
   // If specified, the pod's scheduling constraints
   // +optional
   Affinity *Affinity `json:"affinity,omitempty" protobuf:"bytes,18,opt,name=affinity"`
   // If specified, the pod will be dispatched by specified scheduler.
   // If not specified, the pod will be dispatched by default scheduler.
   // +optional
   SchedulerName string `json:"schedulerName,omitempty" protobuf:"bytes,19,opt,name=schedulerName"`
   // If specified, the pod's tolerations.
   // +optional
   Tolerations []Toleration `json:"tolerations,omitempty" protobuf:"bytes,22,opt,name=tolerations"`
   // HostAliases is an optional list of hosts and IPs that will be injected into the pod's hosts
   // file if specified. This is only valid for non-hostNetwork pods.
   // +optional
   // +patchMergeKey=ip
   // +patchStrategy=merge
   HostAliases []HostAlias `json:"hostAliases,omitempty" patchStrategy:"merge" patchMergeKey:"ip" protobuf:"bytes,23,rep,name=hostAliases"`
   // If specified, indicates the pod's priority. "system-node-critical" and
   // "system-cluster-critical" are two special keywords which indicate the
   // highest priorities with the former being the highest priority. Any other
   // name must be defined by creating a PriorityClass object with that name.
   // If not specified, the pod priority will be default or zero if there is no
   // default.
   // +optional
   PriorityClassName string `json:"priorityClassName,omitempty" protobuf:"bytes,24,opt,name=priorityClassName"`
   // The priority value. Various system components use this field to find the
   // priority of the pod. When Priority Admission Controller is enabled, it
   // prevents users from setting this field. The admission controller populates
   // this field from PriorityClassName.
   // The higher the value, the higher the priority.
   // +optional
   Priority *int32 `json:"priority,omitempty" protobuf:"bytes,25,opt,name=priority"`
   // Specifies the DNS parameters of a pod.
   // Parameters specified here will be merged to the generated DNS
   // configuration based on DNSPolicy.
   // +optional
   DNSConfig *PodDNSConfig `json:"dnsConfig,omitempty" protobuf:"bytes,26,opt,name=dnsConfig"`
   // If specified, all readiness gates will be evaluated for pod readiness.
   // A pod is ready when all its containers are ready AND
   // all conditions specified in the readiness gates have status equal to "True"
   // More info: https://git.k8s.io/enhancements/keps/sig-network/0007-pod-ready%2B%2B.md
   // +optional
   ReadinessGates []PodReadinessGate `json:"readinessGates,omitempty" protobuf:"bytes,28,opt,name=readinessGates"`
   // RuntimeClassName refers to a RuntimeClass object in the node.k8s.io group, which should be used
   // to run this pod.  If no RuntimeClass resource matches the named class, the pod will not be run.
   // If unset or empty, the "legacy" RuntimeClass will be used, which is an implicit class with an
   // empty definition that uses the default runtime handler.
   // More info: https://git.k8s.io/enhancements/keps/sig-node/runtime-class.md
   // This is a beta feature as of Kubernetes v1.14.
   // +optional
   RuntimeClassName *string `json:"runtimeClassName,omitempty" protobuf:"bytes,29,opt,name=runtimeClassName"`
   // EnableServiceLinks indicates whether information about services should be injected into pod's
   // environment variables, matching the syntax of Docker links.
   // Optional: Defaults to true.
   // +optional
   EnableServiceLinks *bool `json:"enableServiceLinks,omitempty" protobuf:"varint,30,opt,name=enableServiceLinks"`
   // PreemptionPolicy is the Policy for preempting pods with lower priority.
   // One of Never, PreemptLowerPriority.
   // Defaults to PreemptLowerPriority if unset.
   // This field is alpha-level and is only honored by servers that enable the NonPreemptingPriority feature.
   // +optional
   PreemptionPolicy *PreemptionPolicy `json:"preemptionPolicy,omitempty" protobuf:"bytes,31,opt,name=preemptionPolicy"`
   // Overhead represents the resource overhead associated with running a pod for a given RuntimeClass.
   // This field will be autopopulated at admission time by the RuntimeClass admission controller. If
   // the RuntimeClass admission controller is enabled, overhead must not be set in Pod create requests.
   // The RuntimeClass admission controller will reject Pod create requests which have the overhead already
   // set. If RuntimeClass is configured and selected in the PodSpec, Overhead will be set to the value
   // defined in the corresponding RuntimeClass, otherwise it will remain unset and treated as zero.
   // More info: https://git.k8s.io/enhancements/keps/sig-node/20190226-pod-overhead.md
   // This field is alpha-level as of Kubernetes v1.16, and is only honored by servers that enable the PodOverhead feature.
   // +optional
   Overhead ResourceList `json:"overhead,omitempty" protobuf:"bytes,32,opt,name=overhead"`
   // TopologySpreadConstraints describes how a group of pods ought to spread across topology
   // domains. Scheduler will schedule pods in a way which abides by the constraints.
   // This field is alpha-level and is only honored by clusters that enables the EvenPodsSpread
   // feature.
   // All topologySpreadConstraints are ANDed.
   // +optional
   // +patchMergeKey=topologyKey
   // +patchStrategy=merge
   // +listType=map
   // +listMapKey=topologyKey
   // +listMapKey=whenUnsatisfiable
   TopologySpreadConstraints []TopologySpreadConstraint `json:"topologySpreadConstraints,omitempty" patchStrategy:"merge" patchMergeKey:"topologyKey" protobuf:"bytes,33,opt,name=topologySpreadConstraints"`
}
```

* Volumes：volume的列表，这里的volume可以被pod里的container mount
* InitContainers：在container之前跑的InitContainers
* Containers
* EphemeralContainers：临时容器，用于故障排查的
* RestartPolicy：重启策略（Always, OnFailure, Never）
* TerminationGracePeriodSeconds：pod terminate gracefully的时候K8S 会给POD发送SIGTERM信号，并且等待terminationGracePeriodSeconds这么长的时间，超过terminationGracePeriodSeconds等待时间后会强制kill
* ActiveDeadlineSeconds：标志失败Pod的重试最大时间，超过这个时间不会继续重试
* DNSPolicy：默认是ClusterFirst，可选值是'ClusterFirstWithHostNet', 'ClusterFirst', 'Default' or 'None'，如果要让pod dns options跟hostnetwork一致，需要设置成ClusterFirstWithHostNet
* NodeSelector：决定pod是否fit on a node
* ServiceAccountName
* AutomountServiceAccountToken：决定是否auto mount一个service account token
* NodeName：如果NodeName非空，scheduler会schedule pod到那个node
* HostNetwork：使用host network namespace
* HostPID：使用host pid namespace
* HostIPC：使用host ipc namespace
* ShareProcessNamespace：在pod的所有containers中进程命名空间共享使用
* SecurityContext：安全上下文，用于定义Pod或Container的权限和访问控制设置
* ImagePullSecrets
* Hostname：pod的hostname，如果没有定义，就是metadata.name
* Subdomain：`<hostname>.<subdomain>.<pod namespace>.svc.<cluster domain>`
* Affinity：scheduling的约束定义
* SchedulerName：分配到指定scheduler
* Tolerations：配置Pod可以容忍哪些污点
* HostAliases：只用于non-hostnetwork pod，这里配置的会被放到pod的/etc/hosts里
* PriorityClassName：pod的优先级，如果设置成system-cluster-critical` 或 `system-node-critical，后者是整个群集的最高级别。
* Priority：优先级的值如果Priority Admission Controller被启动，这个值是没有用的
* DNSConfig：pod的DNS配置
* ReadinessGates：如果ReadinessGates被设置，同时满足两个条件pod才会变成ready：1. pod中所有contaienr状态都是ready，2. 所有pod spec定义的ReadinessGates状态都是true
* RuntimeClassName：容器运行时类，如果未指定 `runtimeClassName` ，则将使用默认的RuntimeHandler，相当于禁用 RuntimeClass 功能特性
* EnableServiceLinks：是否service的信息需要被注入pod的环境变量
* PreemptionPolicy：`PreemptionPolicy`，当设置为`Never`时，该Pod将不会抢占比它优先级低的Pod，只是调度的时候，会优先调度（PriorityClass的value）
* Overhead：用于PodOverhead
* TopologySpreadConstraints：拓扑结构约束

## PodStatus

```go
// PodStatus represents information about the status of a pod. Status may trail the actual
// state of a system, especially if the node that hosts the pod cannot contact the control
// plane.
type PodStatus struct {
   // The phase of a Pod is a simple, high-level summary of where the Pod is in its lifecycle.
   // The conditions array, the reason and message fields, and the individual container status
   // arrays contain more detail about the pod's status.
   // There are five possible phase values:
   //
   // Pending: The pod has been accepted by the Kubernetes system, but one or more of the
   // container images has not been created. This includes time before being scheduled as
   // well as time spent downloading images over the network, which could take a while.
   // Running: The pod has been bound to a node, and all of the containers have been created.
   // At least one container is still running, or is in the process of starting or restarting.
   // Succeeded: All containers in the pod have terminated in success, and will not be restarted.
   // Failed: All containers in the pod have terminated, and at least one container has
   // terminated in failure. The container either exited with non-zero status or was terminated
   // by the system.
   // Unknown: For some reason the state of the pod could not be obtained, typically due to an
   // error in communicating with the host of the pod.
   //
   // More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#pod-phase
   // +optional
   Phase PodPhase `json:"phase,omitempty" protobuf:"bytes,1,opt,name=phase,casttype=PodPhase"`
   // Current service state of pod.
   // More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#pod-conditions
   // +optional
   // +patchMergeKey=type
   // +patchStrategy=merge
   Conditions []PodCondition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type" protobuf:"bytes,2,rep,name=conditions"`
   // A human readable message indicating details about why the pod is in this condition.
   // +optional
   Message string `json:"message,omitempty" protobuf:"bytes,3,opt,name=message"`
   // A brief CamelCase message indicating details about why the pod is in this state.
   // e.g. 'Evicted'
   // +optional
   Reason string `json:"reason,omitempty" protobuf:"bytes,4,opt,name=reason"`
   // nominatedNodeName is set only when this pod preempts other pods on the node, but it cannot be
   // scheduled right away as preemption victims receive their graceful termination periods.
   // This field does not guarantee that the pod will be scheduled on this node. Scheduler may decide
   // to place the pod elsewhere if other nodes become available sooner. Scheduler may also decide to
   // give the resources on this node to a higher priority pod that is created after preemption.
   // As a result, this field may be different than PodSpec.nodeName when the pod is
   // scheduled.
   // +optional
   NominatedNodeName string `json:"nominatedNodeName,omitempty" protobuf:"bytes,11,opt,name=nominatedNodeName"`

   // IP address of the host to which the pod is assigned. Empty if not yet scheduled.
   // +optional
   HostIP string `json:"hostIP,omitempty" protobuf:"bytes,5,opt,name=hostIP"`
   // IP address allocated to the pod. Routable at least within the cluster.
   // Empty if not yet allocated.
   // +optional
   PodIP string `json:"podIP,omitempty" protobuf:"bytes,6,opt,name=podIP"`

   // podIPs holds the IP addresses allocated to the pod. If this field is specified, the 0th entry must
   // match the podIP field. Pods may be allocated at most 1 value for each of IPv4 and IPv6. This list
   // is empty if no IPs have been allocated yet.
   // +optional
   // +patchStrategy=merge
   // +patchMergeKey=ip
   PodIPs []PodIP `json:"podIPs,omitempty" protobuf:"bytes,12,rep,name=podIPs" patchStrategy:"merge" patchMergeKey:"ip"`

   // RFC 3339 date and time at which the object was acknowledged by the Kubelet.
   // This is before the Kubelet pulled the container image(s) for the pod.
   // +optional
   StartTime *metav1.Time `json:"startTime,omitempty" protobuf:"bytes,7,opt,name=startTime"`

   // The list has one entry per init container in the manifest. The most recent successful
   // init container will have ready = true, the most recently started container will have
   // startTime set.
   // More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#pod-and-container-status
   InitContainerStatuses []ContainerStatus `json:"initContainerStatuses,omitempty" protobuf:"bytes,10,rep,name=initContainerStatuses"`

   // The list has one entry per container in the manifest. Each entry is currently the output
   // of `docker inspect`.
   // More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#pod-and-container-status
   // +optional
   ContainerStatuses []ContainerStatus `json:"containerStatuses,omitempty" protobuf:"bytes,8,rep,name=containerStatuses"`
   // The Quality of Service (QOS) classification assigned to the pod based on resource requirements
   // See PodQOSClass type for available QOS classes
   // More info: https://git.k8s.io/community/contributors/design-proposals/node/resource-qos.md
   // +optional
   QOSClass PodQOSClass `json:"qosClass,omitempty" protobuf:"bytes,9,rep,name=qosClass"`
   // Status for any ephemeral containers that have run in this pod.
   // This field is alpha-level and is only populated by servers that enable the EphemeralContainers feature.
   // +optional
   EphemeralContainerStatuses []ContainerStatus `json:"ephemeralContainerStatuses,omitempty" protobuf:"bytes,13,rep,name=ephemeralContainerStatuses"`
}
```

* Phase：Pod生命周期的定义
* Conditions：pod的当前状态
* Message：detail信息为啥pod处于当前的condition
* Reason：简短的CamelCase信息，为啥pod处在当前state
* NominatedNodeName：如果抢占算法返回节点不为空，会更新到NominatedNodeName
* HostIP：pod的host的ip
* PodIP：pod的ip
* PodIPs：podIPs保存分配给pod的IP地址
* StartTime：被kubelet确认的时间（在pull container image之前）
* InitContainerStatuses：InitContainer的状态
* ContainerStatuses：Container的状态
* QOSClass：pod的服务质量（Guaranteed, Burstable, BestEffort）
* EphemeralContainerStatuses：临时容器的状态

