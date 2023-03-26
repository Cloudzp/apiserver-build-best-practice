# apiserver-build-best-practice
使用apiserver-builder工具的最佳实践


## 初始化项目
完成 apiserver-boot 安装后，可通过如下命令来初始化一个 Aggregated APIServer  项目：
```
$ mkdir demo-animal
$ cd demo-animal 
$ apiserver-boot init repo --domain demo.io
```
执行后会生成如下目录：

```
$ tree                                       
.
├── bin
├── cmd
│   ├── apiserver
│   │   └── main.go
│   └── manager
│       └── main.go -> ../../main.go
├── Dockerfile
├── go.mod
├── hack
│   └── boilerplate.go.txt
├── main.go
├── Makefile
└── pkg
    └── apis
        └── doc.go

7 directories, 8 files
```
- hack 目录存放自动脚本
- cmd/apiserver 是 aggregated server的启动入口
- cmd/manager 是 controller 的启动入口
- pkg/apis 存放 CR 相关的结构体定义，会在下一步自动生成

## 生成自定义资源
```
$ apiserver-boot create group version resource --group animal --version v1alpha1 --kind Cat --non-namespaced=false
Create Resource [y/n]
y
Create Controller [y/n]
n
```
可根据自己的需求选择是否生成 Controller，我们这里暂时选择不生成, 对于需要通过 namespace 隔离的 resource 需要增加 --non-namespaced=false 的参数，默认都是 true。

执行完成后代码结构如下：
```
$ tree
.
├── bin
├── cmd
│   ├── apiserver
│   │   └── main.go
│   └── manager
│       └── main.go -> ../../main.go
├── Dockerfile
├── go.mod
├── hack
│   └── boilerplate.go.txt
├── main.go
├── Makefile
└── pkg
    └── apis
        ├── animal
        │   ├── doc.go
        │   └── v1alpha1
        │       ├── cat_types.go
        │       ├── doc.go
        │       └── register.go
        └── doc.go

9 directories, 12 files
```
可以看到在 pkg/apis 下生成了 animal 的 group 并在 v1alpha1 版本下新增了 `cat_types.go` 文件，此文件包含了我们资源的基础定义，我们在 spec 中增加字段定义，并在已经实现的 `Validate` 方法中完成基础字段的校验。
```
// Cat
// +k8s:openapi-gen=true
type Cat struct {
        metav1.TypeMeta   `json:",inline"`
        metav1.ObjectMeta `json:"metadata,omitempty"`

        Spec   CatSpec   `json:"spec,omitempty"`
        Status CatStatus `json:"status,omitempty"`
}

// CatSpec defines the desired state of Cat
type CatSpec struct {
   Name string `json:"name"`
}

func (in *Cat) Validate(ctx context.Context) field.ErrorList {
   allErrs := field.ErrorList{}

   if len(in.Spec.Name) == 0 {
      allErrs = append(allErrs, field.Invalid(field.NewPath("spec", "name"), in.Spec.Name, "must be specify"))
   }
   return allErrs
}
```

在 main 方法中进行资源注册。
```
func main() {
   err := builder.APIServer.
      WithResource(&animalv1alpha1.Cat{}).  // 资源注册。
      WithLocalDebugExtension().            // 本地调试。        
      DisableAuthorization().
      WithOptionsFns(func(options *builder.ServerOptions) *builder.ServerOptions {
         options.RecommendedOptions.CoreAPI = nil
         options.RecommendedOptions.Admission = nil
         return options
      }). // 控制参数， 不依赖 kube-apiserver 的 admission
      // WithoutEtcd(). // 不依赖 etcd 
      Execute()

   if err != nil {
      klog.Fatal(err)
   }
}
```

其他资源注册函数： 

-  [WithResourceAndStorage](https://github.com/kubernetes-sigs/apiserver-builder-alpha/blob/ba22a63c6f739eef533979e20427bd470de1909f/example/kine/cmd/apiserver/main.go): 向apiserver注册资源，创建一个新的etcd备份存储，GroupResource使用提供的策略。在大多数情况下，调用者应该使用WithResource 实现“apiserver-runtime/pkg/builder/rest”中定义的接口来控制策略。 

```
func main() {
   mysqlHost := os.Getenv("MYSQL_HOST")
   mysqlPort, _ := strconv.Atoi(os.Getenv("MYSQL_PORT"))
   mysqlUser := os.Getenv("MYSQL_USERNAME")
   mysqlPasswd := os.Getenv("MYSQL_PASSWORD")
   mysqlDatabase := os.Getenv("MYSQL_DATABASE")
   err := builder.APIServer.
      WithResourceAndStorage(&mysqlv1.Tiger{}, mysql.NewMysqlStorageProvider(
         mysqlHost,
         int32(mysqlPort),
         mysqlUser,
         mysqlPasswd,
         mysqlDatabase,
      )). // namespaced resource
      WithoutEtcd().
      WithLocalDebugExtension().
      Execute()
   if err != nil {
      klog.Fatal(err)
   }
}
```

- [WithResourceAndHandler](https://github.com/kubernetes-sigs/apiserver-builder-alpha/blob/ba22a63c6f739eef533979e20427bd470de1909f/example/basic/cmd/apiserver/main.go): 注册一个请求处理程序，而不是默认的 etcd备份的存储。

``` golang
package main

import (
        "k8s.io/klog"
        "sigs.k8s.io/apiserver-runtime/pkg/builder"
           "sigs.k8s.io/apiserver-runtime/pkg/experimental/storage/filepath"

        // +kubebuilder:scaffold:resource-imports
        animalv1alpha1 "github.com/cloudzp/demo-animal/pkg/apis/animal/v1alpha1"
)

func main() {
     err := builder.APIServer.
      // writes burger resources as static files under the "data" folder in the working directory.
      WithResourceAndHandler(&animalv1alpha1.Cat{}, filepath.NewJSONFilepathStorageProvider(&animalv1alpha1.Cat{}, "data")).
      WithLocalDebugExtension().
      WithoutEtcd().
      Execute()
   if err != nil {
      klog.Fatal(err)
   }
}
```
## 生成`deepcopy.go` 文件
```
$ go mod tidy
$ make generate
$ tree
$ tree
.
├── bin
│   └── controller-gen
├── cmd
│   ├── apiserver
│   │   └── main.go
│   └── manager
│       └── main.go -> ../../main.go
├── Dockerfile
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
├── main.go
├── Makefile
└── pkg
    └── apis
        ├── animal
        │   ├── doc.go
        │   └── v1alpha1
        │       ├── cat_types.go
        │       ├── doc.go
        │       ├── register.go
        │       └── zz_generated.deepcopy.go
        └── doc.go

9 directories, 15 files 
```

## 本地运行
```
$ apiserver-boot run local
I0326 23:17:30.109033 3244801 local.go:244] Failed to run bin/apiserver, error: exit status 255
I0326 23:17:32.080642 3244801 local.go:132] Aggregated apiserver successfully started
I0326 23:17:32.080683 3244801 local.go:138] Controller manager successfully started
I0326 23:17:32.080699 3244801 local.go:141] 
==================================================
| Now you're all set!
| To test the server, try the following commands:
|
| >> "kubectl --kubeconfig kubeconfig api-versions" or "KUBECONFIG=kubeconfig kubectl api-resources"
|
==================================================
I0326 23:17:32.080716 3244801 local.go:240] Starting local component: bin/controller-manager --kubeconfig=kubeconfig
I0326 23:17:32.080725 3244801 local.go:116] Cleaning up processes
I0326 23:17:32.080773 3244801 local.go:289] Completed etcd
I0326 23:17:32.080787 3244801 local.go:284] Waiting for process of bin/apiserver (pid=3245132) to be completed
I0326 23:17:32.080793 3244801 local.go:289] Completed bin/apiserver
I0326 23:17:32.080799 3244801 local.go:289] Completed bin/controller-manager
```

##  查看资源
```
$ kubectl api-versions |grep animal
animal.demo.io/v1alpha1
```
