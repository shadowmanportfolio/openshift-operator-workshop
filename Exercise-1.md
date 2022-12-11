# Operator SDK with Go

## 为什么要用Operator?

Operator使得在Kubernetes上管理复杂的有状态应用程序变得容易。然而，由于使用底层API、编写模板以及缺乏重复的模块化等挑战，导致编写Operator可能非常困难。

## 什么是 Operator SDK?

提供Operator SDK来使用控制器运行库框架，可以让编写Operator更简单:

- 高级API和抽象，以便更直观地编写操作逻辑。
- 用于快速引导新项目的搭建和代码生成工具。
- 扩展以涵盖常见的Operator用例。

## 怎么使用 Operator SDK?

以下是一个用Operator SDK新建基于 **Go** 的Operator工作流程:

1. 使用SDK CLI创建一个新的Operator项目。
2. 使用SDK命令行创建一个新的自定义资源定义API类型。
3. 将您的自定义资源定义(CRD)添加到活动Kubernetes集群中。
4. 定义您的自定义资源规格和状态。
5. 为自定义资源定义API创建一个新的控制器。
6. 为控制器编写调度逻辑。
7. 在本地运行Operator，在活动的Kubernetes集群上测试代码。
8. 将您的自定义资源(CR)添加到您的实时Kubernetes集群，并观看您的Operator行动!
9. 在您对自己的工作感到满意之后，运行一些Makefile命令来构建和生成Operator Deployment清单。
10. 使用SDK CLI可选地添加额外的API和控制器。

## PodSet Operator

在本教程中，我们将创建一个名为PodSet的operator。PodSet是一个管理pod的简单Controller/Operator。

用户提供了在 `spec.replicas`中指定的多个pod。PodSet还方便地输出当前在 `status.PodNames` 字段中由PodSet控制的所有pod的名称。

让我们开始！

### 新建项目

让我们从连接到OpenShift开始，可从web console中获得具体的登录链接。例如:

```shell
oc login --token=7DMj4CRxuUCzXXXXXXXXXX --server=https://XXXXX.com:30596
```

现在让我们创建一个新的项目，接下来我们将在这里进行工作:

```
oc new-project myproject
```

现在让我们为我们的项目创建一个新目录:

```
mkdir -p $HOME/projects/podset-operator
```

导航到目录:

```
cd $HOME/projects/podset-operator
```

为了方便，我们用以下命令初始化了一个基于go operator SDK的PodSet operator新项目。

```
operator-sdk init --domain=example.com --repo=github.com/redhat/podset-operator
```

可以列出使用该命令生成的所有文件:

```
ls -lh
```

### 创建API和控制器

添加一个新的自定义资源定义(CRD) API，名为PodSet, APIVersion为 `app.example.com/v1alpha1` ， Kind为 `PodSet`。该命令还将创建我们的引用控制器逻辑和[Kustomize](https://kustomize.io/)配置文件。

```
cd $HOME/projects/podset-operator && \
  operator-sdk create api --group=app --version=v1alpha1 --kind=PodSet --resource --controller
```

我们现在应该看到 `/api`、 `/config` 和 `/controllers` 目录。

### 定义 Spec（规格） 和 Status（状态）

让我们从检查新生成的 `api/v1alpha1/podset_types.go` 文件开始。从 **Visual Editor** 选项卡打开PodSet API，或者直接运行终端:

```
cd $HOME/projects/podset-operator && \
  cat api/v1alpha1/podset_types.go
```

在Kubernetes中，每个函数对象（除了一些例外，例如ConfigMap）都包含 `spec` 和 `status`。Kubernetes通过协期望状态(Spec)和实际集群状态来发挥作用。然后记录观察到的情况（Status）。

还要注意在整个文件中发现的 `+kubebuilder` 注释标记。 `operator-sdk` 使用一个名为[controller- gen](https://github.com/kubernetes-sigs/controller-tools)的工具(来自[controller-tools](https://github.com/kubernetes-sigs/controller-tools)项目)来生成实用程序代码和Kubernetes YAML。更多关于配置/代码生成标记的信息可以在这里(https://book.kubebuilder.io/reference/markers.html)找到。

现在让我们修改 `api/v1alpha1/podset_types.go` 文件中 `PodSet` 自定义资源(CR)的 `PodSetSpec` 和 `PodSetStatus` 。

```
/*


Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package v1alpha1

import (
        metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// PodSetSpec defines the desired state of PodSet
type PodSetSpec struct {
        // Replicas is the desired number of pods for the PodSet
        // +kubebuilder:validation:Minimum=1
        // +kubebuilder:validation:Maximum=10
        Replicas int32 `json:"replicas,omitempty"`
}

// PodSetStatus defines the current status of PodSet
type PodSetStatus struct {
        PodNames          []string `json:"podNames"`
        AvailableReplicas int32    `json:"availableReplicas"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status

// PodSet is the Schema for the podsets API
// +kubebuilder:printcolumn:JSONPath=".spec.replicas",name=Desired,type=string
// +kubebuilder:printcolumn:JSONPath=".status.availableReplicas",name=Available,type=string
type PodSet struct {
        metav1.TypeMeta   `json:",inline"`
        metav1.ObjectMeta `json:"metadata,omitempty"`

        Spec   PodSetSpec   `json:"spec,omitempty"`
        Status PodSetStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// PodSetList contains a list of PodSet
type PodSetList struct {
        metav1.TypeMeta `json:",inline"`
        metav1.ListMeta `json:"metadata,omitempty"`
        Items           []PodSet `json:"items"`
}

func init() {
        SchemeBuilder.Register(&PodSet{}, &PodSetList{})
}
```

在修改 `*_types.go` 文件之后，总是运行以下命令来更新 `zz_generated.deepcopy.go` 文件:

```
make generate
```

现在我们可以运行 `make manifests` 命令来生成定制的CRD和额外的对象YAML。

```
make manifests
```

多亏了我们的注释标记，我们现在有了一个新生成的CRD yaml，它反映 `spec.replicas` 和 `status.podNames` 的OpenAPI v3模式验证和定制打印列。

```
cat config/crd/bases/app.example.com_podsets.yaml
```

将您的PodSet自定义资源定义部署到活动的OpenShift集群:

```
oc apply -f config/crd/bases/app.example.com_podsets.yaml
```

确认CRD已成功创建:

```
oc get crd podsets.app.example.com -o yaml
```

### 自定义operator逻辑

现在让我们观察默认的 `controllers/podset_controller.go` 文件:

```
cd $HOME/projects/podset-operator && \
  cat controllers/podset_controller.go
```

这个默认控制器需要额外的逻辑，所以每当 `kind: PodSet` 对象被添加、更新或删除时，我们就可以触发协调器。我们还希望在添加、更新和删除给定PodSet所拥有的Pods时触发协调器。我们用修改控制器的 `SetupWithManager` 方法来完成这一任务。

在 `controllers/podset_controller.go` 处修改PodSet控制器逻辑:

```
/*
Copyright 2021.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package controllers

import (
        "context"
        "reflect"

        "k8s.io/apimachinery/pkg/runtime"
        ctrl "sigs.k8s.io/controller-runtime"
        "sigs.k8s.io/controller-runtime/pkg/client"
        ctrllog "sigs.k8s.io/controller-runtime/pkg/log"

        appv1alpha1 "github.com/redhat/podset-operator/api/v1alpha1"
        corev1 "k8s.io/api/core/v1"
        metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

        "k8s.io/apimachinery/pkg/api/errors"
        "k8s.io/apimachinery/pkg/labels"
        "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
)

// PodSetReconciler reconciles a PodSet object
type PodSetReconciler struct {
        client.Client
        Scheme *runtime.Scheme
}

//+kubebuilder:rbac:groups=app.example.com,resources=podsets,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=app.example.com,resources=podsets/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=app.example.com,resources=podsets/finalizers,verbs=update

// Reconcile is part of the main kubernetes reconciliation loop which aims to
// move the current state of the cluster closer to the desired state.
// TODO(user): Modify the Reconcile function to compare the state specified by
// the PodSet object against the actual cluster state, and then
// perform operations to make the cluster state reflect the state specified by
// the user.
//
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.8.3/pkg/reconcile
func (r *PodSetReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
        log := ctrllog.FromContext(ctx)

        // Fetch the PodSet instance
        instance := &appv1alpha1.PodSet{}
        err := r.Get(context.TODO(), req.NamespacedName, instance)
        if err != nil {
                if errors.IsNotFound(err) {
                        // Request object not found, could have been deleted after reconcile request.
                        // Owned objects are automatically garbage collected. For additional cleanup logic use finalizers.
                        // Return and don't requeue
                        return ctrl.Result{}, nil
                }
                // Error reading the object - requeue the request.
                return ctrl.Result{}, err

        }

        // List all pods owned by this PodSet instance
        podSet := instance
        podList := &corev1.PodList{}
        lbs := map[string]string{
                "app":     podSet.Name,
                "version": "v0.1",
        }
        labelSelector := labels.SelectorFromSet(lbs)
        listOps := &client.ListOptions{Namespace: podSet.Namespace, LabelSelector: labelSelector}
        if err = r.List(context.TODO(), podList, listOps); err != nil {
                return ctrl.Result{}, err
        }

        // Count the pods that are pending or running as available
        var available []corev1.Pod
        for _, pod := range podList.Items {
                if pod.ObjectMeta.DeletionTimestamp != nil {
                        continue
                }
                if pod.Status.Phase == corev1.PodRunning || pod.Status.Phase == corev1.PodPending {
                        available = append(available, pod)
                }
        }
        numAvailable := int32(len(available))
        availableNames := []string{}
        for _, pod := range available {
                availableNames = append(availableNames, pod.ObjectMeta.Name)
        }

        // Update the status if necessary
        status := appv1alpha1.PodSetStatus{
                PodNames:          availableNames,
                AvailableReplicas: numAvailable,
        }
        if !reflect.DeepEqual(podSet.Status, status) {
                podSet.Status = status
                err = r.Status().Update(context.TODO(), podSet)
                if err != nil {
                        log.Error(err, "Failed to update PodSet status")
                        return ctrl.Result{}, err
                }
        }

        if numAvailable > podSet.Spec.Replicas {
                log.Info("Scaling down pods", "Currently available", numAvailable, "Required replicas", podSet.Spec.Replicas)
                diff := numAvailable - podSet.Spec.Replicas
                dpods := available[:diff]
                for _, dpod := range dpods {
                        err = r.Delete(context.TODO(), &dpod)
                        if err != nil {
                                log.Error(err, "Failed to delete pod", "pod.name", dpod.Name)
                                return ctrl.Result{}, err
                        }
                }
                return ctrl.Result{Requeue: true}, nil
        }

        if numAvailable < podSet.Spec.Replicas {
                log.Info("Scaling up pods", "Currently available", numAvailable, "Required replicas", podSet.Spec.Replicas)
                // Define a new Pod object
                pod := newPodForCR(podSet)
                // Set PodSet instance as the owner and controller
                if err := controllerutil.SetControllerReference(podSet, pod, r.Scheme); err != nil {
                        return ctrl.Result{}, err
                }
                err = r.Create(context.TODO(), pod)
                if err != nil {
                        log.Error(err, "Failed to create pod", "pod.name", pod.Name)
                        return ctrl.Result{}, err
                }
                return ctrl.Result{Requeue: true}, nil
        }

        return ctrl.Result{}, nil
}

// newPodForCR returns a busybox pod with the same name/namespace as the cr
func newPodForCR(cr *appv1alpha1.PodSet) *corev1.Pod {
        labels := map[string]string{
                "app":     cr.Name,
                "version": "v0.1",
        }
        return &corev1.Pod{
                ObjectMeta: metav1.ObjectMeta{
                        GenerateName: cr.Name + "-pod",
                        Namespace:    cr.Namespace,
                        Labels:       labels,
                },
                Spec: corev1.PodSpec{
                        Containers: []corev1.Container{
                                {
                                        Name:    "busybox",
                                        Image:   "busybox",
                                        Command: []string{"sleep", "3600"},
                                },
                        },
                },
        }
}

// SetupWithManager sets up the controller with the Manager.
func (r *PodSetReconciler) SetupWithManager(mgr ctrl.Manager) error {
        return ctrl.NewControllerManagedBy(mgr).
                For(&appv1alpha1.PodSet{}).
                Owns(&corev1.Pod{}).
                Complete(r)
}


```

`go mod tidy` 确保了go.mod文件与模块中的源代码相匹配。它添加构建当前模块的包和依赖项所需的任何缺失的模块需求，并删除对不提供任何相关包的模块的需求。它还向 go.sum 添加任何丢失的条目，并删除不必要的项。

```
go mod tidy
```

### 本地运行operator(集群外部)

一旦CRD被注册，有两种方式来运行Operator:

- 作为Kubernetes集群中的Pod。
- 作为一个使用Operator-SDK的集群外的Go程序。这对于你的operator的本地开发是很好的。

为了本教程的目的，我们将使用 Operator-SDK 和我们的 `kubeconfig` 凭证在集群外运行Operator作为Go程序

该命令一旦运行，将阻塞当前会话。您可以通过打开一个新的终端窗口来继续与OpenShift集群交互。您可以通过按 `CTRL + C`。

```
cd $HOME/projects/podset-operator && \
  MATCH_NAMESPACE=myproject KUBECONFIG=$HOME/.kube/config make run
```

在一个新的终端中，检查Custom Resource清单:

```
cd $HOME/projects/podset-operator
cat config/samples/app_v1alpha1_podset.yaml
```

确保您的 `kind: PodSet` 自定义资源(CR)的' spec.replicas '已经被更新

```
apiVersion: app.example.com/v1alpha1
kind: PodSet
metadata:
  name: podset-sample
spec:
  replicas: 1
```

确保你当前的作用域是 `myproject` 命名空间:

```
oc project myproject
```

将您的PodSet自定义资源部署到活动的OpenShift集群:

```
oc create -f config/samples/app_v1alpha1_podset.yaml
```

验证Podset是否存在:

```
oc get podset
```

验证PodSetoperator已经创建了1个pod:

```
oc get pods
```

验证status显示了PodSet当前拥有的pods的名称:

```
oc get podset podset-sample -o yaml
```

增加PodSet拥有的副本数量:

```
oc patch podset podset-sample --type='json' -p '[{"op": "replace", "path": "/spec/replicas", "value":3}]'
```

验证出现3个正在运行的pod

```
oc get pods
```

### 删除PodSet自定义资源

我们的PodSet控制器在它们的 `metadata` 部分创建了包含 ownerreference 的pod。这确保了它们将在删除 `podset-sample` CR时被删除。

观察Podset的pod上的OwnerReference集合:

```
oc get pods -o yaml | grep ownerReferences -A10
```

删除podset-sample自定义资源:

```
oc delete podset podset-sample
```

多亏了OwnerReferences，所有的pod都应该被删除:

```
oc get pods
```
