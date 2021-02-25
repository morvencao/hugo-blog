---
title: "深入 kubernetes API 的源码实现"
date: 2021-02-25
categories: ['note', 'tech']
draft: false
---

很多同学应该像我一样，第一次打开 Github 上面 kubernetes 项目源码的时候就被各种仓库搞晕了，kuberentes 组织下有很多个仓库，包括 kubernetes、client-go、api、apimachinery 等，该从哪儿仓库看起？kubernetes 仓库应该是 kubernetes 项目的核心仓库，它包含 kubernetes 控制平面核心组件的源码；client-go 从名字也不难看出是操作 kubernetes API 的 go 语言客户端；api 与 apimachinery 应该是与 kubernetes API 相关的仓库，但它们俩为啥要分成两个不同的仓库？这些代码仓库之间如何交互？apimachinery仓库中还有 api、apis 两个包，里面定义了各种复杂的接口与实现，清楚这些复杂接口对于扩展 kubernetes API 大有裨益。所以，这篇文章就重点关注 api 与 apimachinery 这两个仓库。

## api

我们知道 kubernetes 官方提供了多种多样的的 API 资源类型，它们被定义在 k8s.io/api 这个仓库中，作为 kubernetes API 定义的规范地址。实际上，最开始这个仓库只是 kubernetes 核心仓库的一部分，后来 kubernetes API 定义规范被越来越多的其他仓库使用，例如 k8s.io/client-go、k8s.io/apimachinery、k8s.io/apiserver 等，为了避免交叉依赖，所以才把 api 拿出来作为单独的仓库。k8s.io/api 仓库是只读仓库，所有代码都同步自 https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/api 核心仓库。

在 k8s.io/api 仓库定义的kubernetes API 规范中，Pod 作为最基础的资源类型，一个典型的 YAML 形式的序列化 pod 对象如下所示：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webserver
  labels:
    app: webserver
spec:
  containers:
  - name: webserver
    image: nginx
    ports:
    - containerPort: 80
```

从编程的角度来看，序列化的 pod 对象最终会被发送到 API-Server 并解码为 Pod 类型的 Go 结构体，同时 YAML 中的各个字段会被赋值给该 Go 结构体。那么，Pod 类型在 Go 语言结构体中是怎么定义的呢？

```golang
// source code from https://github.com/kubernetes/api/blob/master/core/v1/types.go
type Pod struct {
    // 从TypeMeta字段名可以看出该字段定义Pod类型的元信息，类似于面向对象编程里面
    // Class本身的元信息，类似于Pod类型的API分组、API版本等
    metav1.TypeMeta `json:",inline"`
    // ObjectMeta字段定义单个Pod对象的元信息。每个kubernetes资源对象都有自己的元信息，
    // 例如名字、命名空间、标签、注释等等，kuberentes把这些公共的属性提取出来就是
    // metav1.ObjectMeta，成为了API对象类型的父类
    metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`
    // PodSpec表示Pod类型的对象定义规范，最为代表性的就是CPU、内存的资源使用。
    // 这个字段和YAML中spec字段对应
    Spec PodSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`
    // PodStatus表示Pod的状态，比如是运行还是挂起、Pod的IP等等。Kubernetes会根据pod在
    // 集群中的实际状态来更新PodStatus字段
    Status PodStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
```

从上面 Pod 定义的结构体可以看出，它继承了 metav1.TypeMeta 和 metav1.ObjectMeta 两个类型，metav1.TypeMeta 对应 YAML 中的 kind 与 apiVersion 段，而 metav1.ObjectMeta 则对应 metadata 字段。这其实也可以从 Go 结构体的字段 json 标签看得出来。除了 metav1.TypeMeta 和 metav1.ObjectMeta 字段，Pod 结构体同时还定义了 Spec 和 Status 两个成员变量。如果去查看 k8s.io/api 仓库中其他 API 资源结构体的定义就会发现 kubernetes 绝大部分 API 资源类型都是这样的结构，这也就是说 kubernetes API 资源类型都继承 metav1.TypeMeta 和 metav1.ObjectMeta，前者用于定义资源类型的属性，后者用于定义资源对象的公共属性；Spec 用于定义 API 资源类型的私有属性，也是不同 API 资源类型之间的区别所在；Status 则是用于描述每个资源对象的状态，这和每个资源类型紧密相关的。

关于 metav1.TypeMeta 和 metav1.ObjectMeta 字段从语义上也很好理解，这两个类型作为所有 kubernetes API 资源对象的基类，每个 API 资源对象需要 metav1.TypeMeta 字段用于描述自己是什么类型，这样才能构造相应类型的对象，所以相同类型的所有资源对象的 metav1.TypeMeta 字段都是相同的，但是 metav1.ObjectMeta 则不同，它是定义资源对象实例的属性，即所有资源对象都应该具备的属性。这部分就是和对象本身相关，和类型无关，所以相同类型的资源对象的 metav1.ObjectMeta 可能是不同的。

在 kubernetes 的 API 资源对象中除了单体对象外，还有对象列表类型，用于描述一组相同类型的对象列表。对象列表的典型应用场景就是列举，对象列表就可以表达一组资源对象。可能有些读者会问为什么不用对象的slice，例如[]Pod，伴随着笔者对对象列表的解释读者就会理解，此处以PodList为例进行分析：

```golang
// source code from https://github.com/kubernetes/api/blob/master/core/v1/types.go
type PodList struct {
    // PodList也需要继承metav1.TypeMeta，毕竟对象列表也好、单体对象也好都需要类型属性。
    // PodList比[]Pod类型在yaml或者json表达上多了类型描述，当需要根据YAML构建对象列表的时候，
    // 就可以根据类型描述反序列成为PodList。而[]Pod则不可以，必须确保YAML就是[]Pod序列化的
    // 结果，否则就会报错。这就无法实现一个通用的对象序列化/反序列化。
    metav1.TypeMeta `json:",inline"`
    // 与Pod不同，PodList继承了metav1.ListMeta，metav1.ListMeta是所有资源对象列表类型的父类，
    // ListMeta定义了所有对象列表类型实例的公共属性。
    metav1.ListMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`
    // Items字段则是PodList定义的本质，表示Pod资源对象的列表，所以说PodList就是[]Pod基础上加了一些
    // 跟类型和对象列表相关的元信息
    Items []Pod `json:"items" protobuf:"bytes,2,rep,name=items"`
```

在开始下一节的内容之前，我们先做个小结：

1. metav1.TypeMeta 和 metav1.ObjectMeta 是所有 API 单体资源对象的父类；
2. metav1.TypeMeta 和 metav1.ListMeta 是所有 API 资源对象列表的父类；
3. metav1.TypeMeta 是所有 API 资源对象的父类，因为所有的资源对象都要说明表示是什么类型；

## metav1

这里的 metav1 是包 k8s.io/apimachinery/pkg/apis/meta/v1 的别名，本文其他部分的将用 metav1 指代。

### metav1.TypeMeta

metav1.TypeMeta 用来描述 kubernetes API 资源对象类型的元信息，包括资源类型的名字以及对应 API 的 schema。这里的 schema 指的是资源类型 API 分组以及版本。

```golang
// source code from https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go
//
// TypeMeta describes an individual object in an API response or request
// with strings representing the type of the object and its API schema version.
// Structures that are versioned or persisted should inline TypeMeta.
type TypeMeta struct {
	// Kind is a string value representing the REST resource this object represents.
	// Servers may infer this from the endpoint the client submits requests to.
	// Cannot be updated.
	Kind string `json:"kind,omitempty" protobuf:"bytes,1,opt,name=kind"`

	// APIVersion defines the versioned schema of this representation of an object.
	// Servers should convert recognized schemas to the latest internal value, and
	// may reject unrecognized values.
	APIVersion string `json:"apiVersion,omitempty" protobuf:"bytes,2,opt,name=apiVersion"`
}
```

细心的同学还会发现 metav1.TypeMeta 实现了 schema.ObjectKind 接口，schema.ObjectKind 接口了所有序列化对象怎么解码与编码资源类型信息的方法，它的完整定义如下：

```golang
// source code from https://github.com/kubernetes/apimachinery/blob/master/pkg/runtime/schema/interfaces.go
//
// All objects that are serialized from a Scheme encode their type information. This interface is used
// by serialization to set type information from the Scheme onto the serialized version of an object.
// For objects that cannot be serialized or have unique requirements, this interface may be a no-op.
type ObjectKind interface {
	// SetGroupVersionKind sets or clears the intended serialized kind of an object. Passing kind nil
	// should clear the current setting.
	SetGroupVersionKind(kind GroupVersionKind)
	// GroupVersionKind returns the stored group, version, and kind of an object, or an empty struct
	// if the object does not expose or provide these fields.
	GroupVersionKind() GroupVersionKind
}
```

从 metav1.TypeMeta 对象的实例（也就是任何 kubernetes API 资源对象）都可以通过 `GetObjectKind()` 方法获取到 schema.ObjectKind 类型对象，而 TypeMeta 对象的实例本身也实现了 schema.ObjectKind 接口：

```golang
// source code from https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go

func (obj *TypeMeta) GetObjectKind() schema.ObjectKind { return obj }

// SetGroupVersionKind satisfies the ObjectKind interface for all objects that embed TypeMeta
func (obj *TypeMeta) SetGroupVersionKind(gvk schema.GroupVersionKind) {
	obj.APIVersion, obj.Kind = gvk.ToAPIVersionAndKind()
}

// GroupVersionKind satisfies the ObjectKind interface for all objects that embed TypeMeta
func (obj *TypeMeta) GroupVersionKind() schema.GroupVersionKind {
	return schema.FromAPIVersionAndKind(obj.APIVersion, obj.Kind)
}
```

### metav1.ObjectMeta

metav1.ObjectMeta 则用来定义资源对象实例的属性，即所有资源对象都应该具备的属性。这部分就是和对象本身相关，和类型无关，所以相同类型的资源对象的 metav1.ObjectMeta 可能是不同的。

```golang
// source code from https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go
//
// ObjectMeta is metadata that all persisted resources must have, which includes all objects
// users must create.
type ObjectMeta struct {
	// Name must be unique within a namespace. Is required when creating resources, although
	// some resources may allow a client to request the generation of an appropriate name
	// automatically. Name is primarily intended for creation idempotence and configuration
	// definition.
	Name string `json:"name,omitempty" protobuf:"bytes,1,opt,name=name"`

	// GenerateName is an optional prefix, used by the server, to generate a unique
    // name ONLY IF the Name field has not been provided.
    // Populated by the system. Read-only.
	GenerateName string `json:"generateName,omitempty" protobuf:"bytes,2,opt,name=generateName"`

	// Namespace defines the space within which each name must be unique. An empty namespace is
	// equivalent to the "default" namespace, but "default" is the canonical representation.
	// Not all objects are required to be scoped to a namespace - the value of this field for
	// those objects will be empty.
	Namespace string `json:"namespace,omitempty" protobuf:"bytes,3,opt,name=namespace"`

    // SelfLink is a URL representing this object.
    // Populated by the system. Read-only.
	SelfLink string `json:"selfLink,omitempty" protobuf:"bytes,4,opt,name=selfLink"`

	// UID is the unique in time and space value for this object. It is typically generated by
	// the server on successful creation of a resource and is not allowed to change on PUT
    // operations.
    // Populated by the system. Read-only.
	UID types.UID `json:"uid,omitempty" protobuf:"bytes,5,opt,name=uid,casttype=k8s.io/kubernetes/pkg/types.UID"`

	// An opaque value that represents the internal version of this object that can
	// be used by clients to determine when objects have changed. May be used for optimistic
	// concurrency, change detection, and the watch operation on a resource or set of resources.
	// Clients must treat these values as opaque and passed unmodified back to the server.
    // They may only be valid for a particular resource or set of resources.
    // Populated by the system. Read-only.
	ResourceVersion string `json:"resourceVersion,omitempty" protobuf:"bytes,6,opt,name=resourceVersion"`

	// A sequence number representing a specific generation of the desired state.
	// Populated by the system. Read-only.
	Generation int64 `json:"generation,omitempty" protobuf:"varint,7,opt,name=generation"`

	// CreationTimestamp is a timestamp representing the server time when this object was
	// created. It is not guaranteed to be set in happens-before order across separate operations.
	// Clients may not set this value. It is represented in RFC3339 form and is in UTC.
    // Populated by the system. Read-only.
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
	DeletionTimestamp *Time `json:"deletionTimestamp,omitempty" protobuf:"bytes,9,opt,name=deletionTimestamp"`

	// Number of seconds allowed for this object to gracefully terminate before
	// it will be removed from the system. Only set when deletionTimestamp is also set.
	// May only be shortened.
	// Read-only.
	DeletionGracePeriodSeconds *int64 `json:"deletionGracePeriodSeconds,omitempty" protobuf:"varint,10,opt,name=deletionGracePeriodSeconds"`

	// Map of string keys and values that can be used to organize and categorize
	// (scope and select) objects. May match selectors of replication controllers
	// and services.
	Labels map[string]string `json:"labels,omitempty" protobuf:"bytes,11,rep,name=labels"`

	// Annotations is an unstructured key value map stored with a resource that may be
	// set by external tools to store and retrieve arbitrary metadata. They are not
	// queryable and should be preserved when modifying objects.
	Annotations map[string]string `json:"annotations,omitempty" protobuf:"bytes,12,rep,name=annotations"`

	// List of objects depended by this object. If ALL objects in the list have
	// been deleted, this object will be garbage collected. If this object is managed by a controller,
	// then an entry in this list will point to this controller, with the controller field set to true.
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
	Finalizers []string `json:"finalizers,omitempty" patchStrategy:"merge" protobuf:"bytes,14,rep,name=finalizers"`

	// The name of the cluster which the object belongs to.
	// This is used to distinguish resources with same name and namespace in different clusters.
	// This field is not set anywhere right now and apiserver is going to ignore it if set in create or update request.
	ClusterName string `json:"clusterName,omitempty" protobuf:"bytes,15,opt,name=clusterName"`

	// ManagedFields maps workflow-id and version to the set of fields
	// that are managed by that workflow. This is mostly for internal
	// housekeeping, and users typically shouldn't need to set or
	// understand this field. A workflow can be the user's name, a
	// controller's name, or the name of a specific apply path like
	// "ci-cd". The set of fields is always in the version that the
	// workflow used when modifying the object.
	ManagedFields []ManagedFieldsEntry `json:"managedFields,omitempty" protobuf:"bytes,17,rep,name=managedFields"`
}
```

metav1.ObjectMeta 还实现了 metav1.Object 与 metav1.MetaAccessor 这两个接口，其中 metav1.Object 接口定义了获取单个资源对象各种元信息的 Get 与 Set 方法：

```golang
// source code from https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/meta.go
//
// Object lets you work with object metadata from any of the versioned or
// internal API objects. Attempting to set or retrieve a field on an object that does
// not support that field (Name, UID, Namespace on lists) will be a no-op and return
// a default value.
type Object interface {
	GetNamespace() string
	SetNamespace(namespace string)
	GetName() string
	SetName(name string)
	GetGenerateName() string
	SetGenerateName(name string)
	GetUID() types.UID
	SetUID(uid types.UID)
	GetResourceVersion() string
	SetResourceVersion(version string)
	GetGeneration() int64
	SetGeneration(generation int64)
	GetSelfLink() string
	SetSelfLink(selfLink string)
	GetCreationTimestamp() Time
	SetCreationTimestamp(timestamp Time)
	GetDeletionTimestamp() *Time
	SetDeletionTimestamp(timestamp *Time)
	GetDeletionGracePeriodSeconds() *int64
	SetDeletionGracePeriodSeconds(*int64)
	GetLabels() map[string]string
	SetLabels(labels map[string]string)
	GetAnnotations() map[string]string
	SetAnnotations(annotations map[string]string)
	GetFinalizers() []string
	SetFinalizers(finalizers []string)
	GetOwnerReferences() []OwnerReference
	SetOwnerReferences([]OwnerReference)
	GetClusterName() string
	SetClusterName(clusterName string)
	GetManagedFields() []ManagedFieldsEntry
	SetManagedFields(managedFields []ManagedFieldsEntry)
}
```

metav1.MetaAccessor 接口则定义了获取资源对象存取器的方法：

```golang
// source code from https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/meta.go
//
type ObjectMetaAccessor interface {
	GetObjectMeta() Object
}
```

因为 kubernetes 所有单体资源对象都继承了 metav1.ObjectMeta，那么所有的 API 资源对象就都实现了 metav1.Object 和 metav1.MetaAccessor 接口。kubernetes 中有很多地方访问 API 资源对象的元信息并且不区分对象类型，只要是 metav1.Object 接口类型的对象都可以访问。

### metav1.ListMeta

metav1.ListMeta 定义了所有对象列表类型实例的公共属性。

```golang
// source code from https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go
//
// ListMeta describes metadata that synthetic resources must have, including lists and
// various status objects. A resource may have only one of {ObjectMeta, ListMeta}.
type ListMeta struct {
	// selfLink is a URL representing this object.
	// Populated by the system.
	// Read-only.
	SelfLink string `json:"selfLink,omitempty" protobuf:"bytes,1,opt,name=selfLink"`

	// String that identifies the server's internal version of this object that
	// can be used by clients to determine when objects have changed.
	// Value must be treated as opaque by clients and passed unmodified back to the server.
	// Populated by the system.
	// Read-only.
	ResourceVersion string `json:"resourceVersion,omitempty" protobuf:"bytes,2,opt,name=resourceVersion"`

	// continue may be set if the user set a limit on the number of items returned, and indicates that
	// the server has more data available. The value is opaque and may be used to issue another request
	// to the endpoint that served this list to retrieve the next set of available objects. Continuing a
	// consistent list may not be possible if the server configuration has changed or more than a few
	// minutes have passed. The resourceVersion field returned when using this continue value will be
	// identical to the value in the first response, unless you have received this token from an error
	// message.
	Continue string `json:"continue,omitempty" protobuf:"bytes,3,opt,name=continue"`

	// remainingItemCount is the number of subsequent items in the list which are not included in this
	// list response. If the list request contained label or field selectors, then the number of
	// remaining items is unknown and the field will be left unset and omitted during serialization.
	// If the list is complete (either because it is not chunking or because this is the last chunk),
	// then there are no more remaining items and this field will be left unset and omitted during
	// serialization.
	// Servers older than v1.15 do not set this field.
	// The intended use of the remainingItemCount is *estimating* the size of a collection. Clients
	// should not rely on the remainingItemCount to be set or to be exact.
	RemainingItemCount *int64 `json:"remainingItemCount,omitempty" protobuf:"bytes,4,opt,name=remainingItemCount"`
}
```

类似于与 metav1.ObjectMeta 结构体，metav1.ListMeta 还实现了 metav1.ListInterface 与 metav1.ListMetaAccessor 这两个接口，其中 metav1.ListInterface 接口定义了获取资源对象列表各种元信息的 Get 与 Set 方法：

```golang
// source code from https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/meta.go
//
// ListInterface lets you work with list metadata from any of the versioned or
// internal API objects. Attempting to set or retrieve a field on an object that does
// not support that field will be a no-op and return a default value.
type ListInterface interface {
	GetResourceVersion() string
	SetResourceVersion(version string)
	GetSelfLink() string
	SetSelfLink(selfLink string)
	GetContinue() string
	SetContinue(c string)
	GetRemainingItemCount() *int64
	SetRemainingItemCount(c *int64)
}
```

metav1.ListMetaAccessor 接口则定义了获取资源对象列表存取器的方法：

```golang
// source code from https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/meta.go
//
// ListMetaAccessor retrieves the list interface from an object
type ListMetaAccessor interface {
	GetListMeta() ListInterface
}
```

### runtime.Object

前面在介绍 metav1.TypeMeta 与 metav1.ObjectMeta 的时候我们发现 schema.ObjecKind 是所有 API 资源类型的抽象，metav1.Object 是所有 API 单体资源对象属性的抽象，那么同时实现这两个接口的类型对象不就可以访问任何 API 对象的公共属性了吗？是的，对于每一个特定的类型，如 Pod、Deployment 等，它们确实可以获取当前 API 对象的公共属性。有没有一种所有特定类型的统一父类，同时拥有 schema.ObjecKind 和 metav1.Object 两个接口，这样就可以表示任何特定类型的对象。这就是本节要讨论 runtime.Object 接口。

先来看看 runtime.Object 接口定义：

```golang
// source code from: https://github.com/kubernetes/apimachinery/blob/master/pkg/runtime/interfaces.go
//
// Object interface must be supported by all API types registered with Scheme. Since objects in a scheme are
// expected to be serialized to the wire, the interface an Object must provide to the Scheme allows
// serializers to set the kind, version, and group the object is represented as. An Object may choose
// to return a no-op ObjectKindAccessor in cases where it is not expected to be serialized.
type Object interface {
    // used to access type metadata(GVK)
    GetObjectKind() schema.ObjectKind
    // DeepCopyObject needed to implemented by each kubernetes API type definition,
    // usually by automatically generated code.
	DeepCopyObject() Object
}
```

为什么 runtime.Object 接口只有这两个方法，不应该有 GetObjectMeta() 方法来获取 metav1.ObjectMeta 对象吗？仔细往下看的话会发现，这里使用了不一样的实现方式：

```golang
// source code from: https://github.com/kubernetes/apimachinery/blob/master/pkg/api/meta/meta.go
//
// Accessor takes an arbitrary object pointer and returns meta.Interface.
// obj must be a pointer to an API type. An error is returned if the minimum
// required fields are missing. Fields that are not required return the default
// value and are a no-op if set.
func Accessor(obj interface{}) (metav1.Object, error) {
	switch t := obj.(type) {
	case metav1.Object:
		return t, nil
	case metav1.ObjectMetaAccessor:
		if m := t.GetObjectMeta(); m != nil {
			return m, nil
		}
		return nil, errNotObject
	default:
		return nil, errNotObject
	}
}
```

Accessor 方法可以讲任何的类型 metav1.Object 或者返回错误信息，这样就避免了每个 API 资源类型都需要实现 GetObjectMeta() 方法了。

还有个问题是为什么没有看到 API 资源类型实现 runtime.Object.DeepCopyObject() 方法？那是因为深拷贝方法是具体 API 资源类型需要重载实现的，存在类型依赖，作为 API 资源类型的父类不能统一实现。一般来说，深拷贝方法是由工具自动生成的，定义在 `zz_generated.deepcopy.go` 文件中，以 configMap 为例：

```golang
// source code from https://github.com/kubernetes/api/blob/master/core/v1/zz_generated.deepcopy.go
//
// DeepCopyObject is an autogenerated deepcopy function, copying the receiver, creating a new runtime.Object.
func (in *ConfigMap) DeepCopyObject() runtime.Object {
	if c := in.DeepCopy(); c != nil {
		return c
	}
	return nil
}
```

## 小结

为了便于记忆，现在对前面介绍的各种接口以及实现做一个小结：

![k8s-apimachinery.jpg](https://i.loli.net/2021/02/25/TE1IG6mqYi89jfv.jpg)

1. runtime.Object 接口是所有 API 单体资源对象的根类，各个 API 对象的编码与解码依赖于该接口类型；
2. schema.ObjectKind 接口是对 API 资源对象类型的抽象，可以用来获取或者设置 GVK；
3. metav1.Object 接口是 API 资源对象属性的抽象，用来存取资源对象的属性；
4. metav1.ListInterface 接口是 API 对象列表属性的抽象，用来存取资源对象列表的属性；
5. metav1.TypeMeta 结构体实现了 schema.ObjectKind 接口，所有的 API 资源类型继承它；
6. metav1.ObjectMeta 结构体实现了 metav1.Object 接口，所有的 API 资源类型继承它；
7. metav1.ListMeta 结构体实现了 metav1.ListInterface 接口，所有的 API 资源对象列表类型继承它；
