### 简介

Kubernetes提供了一个端到端（E2E，从用户而非开发人员的角度）的测试框架，确保K8S代码库的行为一致、可靠。在分布式系统中，通过单元测试/集成测试用例，而端到端行为异常的情况不少见。E2E框架基于[Ginkgo](https://blog.gmem.cc/ginkgo-study-note)、Gomega构建。

除了保证测试覆盖率，编写E2E测试的另一个目标是Flaky Test ——  大多数情况下能通过，但是间歇性的、因为难以调试的原因而失败的场景。

#### 如何进行E2E测试

Kubernetes的E2E测试流程包含几个阶段：

1. 实现测试套件，需要利用基于Ginkgo/Gomega的E2E Framework，以及client-go

2. 利用kubetest创建并启动一个测试集群，或者使用现有集群

3. 在测试集群中运行E2E测试套件。使用kubetest、go test、ginkgo命令均可以。K8S E2E测试会连接到默认（根据环境变量KUBECONFIG确定）集群

### kubetest

运行E2E测试的方式有多种，典型的方式是通过kubetest命令。

#### 安装

```shell
go get -u k8s.io/test-infra/kubetest
```


#### 运行测试

端到端测试可以启动Master以及Worker节点、执行某些测试，最后清理掉临时的K8S集群。为了自动化创建K8S集群，你需要提供`--provider`参数，其默认值是gce。

在测试之前，你可以先构建以下Kubernetes项目的e2e框架：

```shell
cd $GOPATH/src/k8s.io/kubernetes
go install ./test/e2e
```


以确保能编译通过。

构建Kubernetes、启动一个集群、运行测试、清理，这一系列阶段可以通过下面的命令完成：

```shell
kubetest --build --up --test --down
```


如果你仅仅想执行部分阶段，可以：

```shell
# 构建
kubetest --build
 
# 启动空白集群，如果存在先删除之
kubetest --up
 
# 运行所有测试
kubetest --test
 
# 运行匹配的测试
kubetest --test --test_args="--ginkgo.focus=\[Feature:Performance\]" --provider=local
 
# 跳过指定的测试
kubetest --test --test_args="--ginkgo.skip=Pods.*env"
 
# 并行测试，跳过不支持并行的那些用例
GINKGO_PARALLEL=y kubetest --test --test_args="--ginkgo.skip=\[Serial\]"
 
# 指定云提供商
kubetest --provider=aws --build --up --test --down
 
 
# 针对临时集群调用kubectl
kubetest -ctl='get events'
kubetest -ctl='delete pod foobar'
 
 
# 清理
kubetest --down
```


#### 测试特定版本

利用kubetest你可以下载任意版本的K8S，包括服务器组件、客户端、测试二进制文件。

```shell
kubetest --extract=v1.5.1 --up  # 部署1.5.1
kubetest --extract=v1.5.2-beta.0  --up  # 部署 1.5.2-beta.0
```


#### 传递选项给Ginkgo

使用`--ginkgo.xxx`可以向Ginkgo传递命令行参数。

#### 使用本地集群

##### 选项

```shell
export KUBECONFIG=/path/to/kubeconfig
kubetest --provider=local --test
 
--host=""         # 指定API Servier
--kubeconfig=""   # 指定Kubeconfig
```


##### 清理

如果使用本地集群进行反复测试，你可能需要周期性的进行某些手工清理：

1. 执行`rm -rf /var/run/kubernetes`删除K8S生成的凭证文件，某些情况下上次测试遗留的凭证文件会导致问题

2. 执行`sudo iptables -F`清空kube-proxy生成的Iptables规则

#### 指定K8S代码库

使用选项`--repo-root="../../"`，可以指定K8S代码库的根目录，E2E测试文件将从中寻找。

### E2E Framework

[此框架](https://godoc.org/k8s.io/kubernetes/test/e2e/framework)提供了云提供商无关的助手代码，用于构建和运行E2E测试。

使用此包需要引入`import "k8s.io/kubernetes/test/e2e/framework"`。

#### 如何工作

查看Kubernetes源码即可了解如何使用E2E框架。入口点：

```go
func TestE2E(t *testing.T) {
    RunE2ETests(t)
}
```


可以看到这是一个标准的Go Test，它调用e2e.go中定义的：

```go
import (
    "k8s.io/klog"
 
    "github.com/onsi/ginkgo"
    "github.com/onsi/ginkgo/config"
    "github.com/onsi/ginkgo/reporters"
    "github.com/onsi/gomega"
 
    v1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    runtimeutils "k8s.io/apimachinery/pkg/util/runtime"
    "k8s.io/component-base/logs"
    "k8s.io/component-base/version"
    commontest "k8s.io/kubernetes/test/e2e/common"
    "k8s.io/kubernetes/test/e2e/framework"
    e2elog "k8s.io/kubernetes/test/e2e/framework/log"
    e2enode "k8s.io/kubernetes/test/e2e/framework/node"
    e2epod "k8s.io/kubernetes/test/e2e/framework/pod"
    "k8s.io/kubernetes/test/e2e/manifest"
    e2ereporters "k8s.io/kubernetes/test/e2e/reporters"
    testutils "k8s.io/kubernetes/test/utils"
    utilnet "k8s.io/utils/net"
 
    clientset "k8s.io/client-go/kubernetes"
    // 确保Auth插件加载
    _ "k8s.io/client-go/plugin/pkg/client/auth"
 
    // ensure that cloud providers are loaded
    _ "k8s.io/kubernetes/test/e2e/framework/providers/aws"
    _ "k8s.io/kubernetes/test/e2e/framework/providers/azure"
    _ "k8s.io/kubernetes/test/e2e/framework/providers/gce"
    _ "k8s.io/kubernetes/test/e2e/framework/providers/kubemark"
    _ "k8s.io/kubernetes/test/e2e/framework/providers/openstack"
    _ "k8s.io/kubernetes/test/e2e/framework/providers/vsphere"
)
 
func RunE2ETests(t *testing.T) {
    // 控制HandleCrash函数的行为，设置为True导致panic
    runtimeutils.ReallyCrash = true
    // 初始化日志
    logs.InitLogs()
    // 刷空日志
    defer logs.FlushLogs()
    // 断言失败处理
    gomega.RegisterFailHandler(e2elog.Fail)
    // 除非明确通过命令行参数要求，否则跳过测试
    if config.GinkgoConfig.FocusString == "" && config.GinkgoConfig.SkipString == "" {
        config.GinkgoConfig.SkipString = `\[Flaky\]|\[Feature:.+\]`
    }
 
    // 初始化Reporter
    var r []ginkgo.Reporter
    if framework.TestContext.ReportDir != "" {
        if err := os.MkdirAll(framework.TestContext.ReportDir, 0755); err != nil {
            klog.Errorf("Failed creating report directory: %v", err)
        } else {
            r = append(r, reporters.NewJUnitReporter(path.Join(framework.TestContext.ReportDir, fmt.Sprintf("junit_%v%02d.xml", framework.TestContext.ReportPrefix, config.GinkgoConfig.ParallelNode))))
        }
    }
 
    // 测试进度信息输出到控制台，以及可选的外部URL
    r = append(r, e2ereporters.NewProgressReporter(framework.TestContext.ProgressReportURL))
    
    // 启动测试套件，使用Ginkgo默认Reporter + 自定义的Reporter
    ginkgo.RunSpecsWithDefaultAndCustomReporters(t, "Kubernetes e2e suite", r)
}
```


上述代码主要就是启动测试套件。实际执行Spec之前会调用下面的函数启动进行准备工作：

```go
func setupSuite() {
    // Run only on Ginkgo node 1
 
    // GCE/GKE特殊处理
    switch framework.TestContext.Provider {
    case "gce", "gke":
        framework.LogClusterImageSources()
    }
    // 创建Clientset
    c, err := framework.LoadClientset()
    if err != nil {
        klog.Fatal("Error loading client: ", err)
    }
 
    // 删除所有K8S自带的命名空间，清除上次测试的残余
    if framework.TestContext.CleanStart {
                        // E2E框架提供大量便捷的API
        deleted, err := framework.DeleteNamespaces(c, nil, /* 支持过滤器 */
            []string{
                metav1.NamespaceSystem,
                metav1.NamespaceDefault,
                metav1.NamespacePublic,
                v1.NamespaceNodeLease,
            })
        if err != nil {
            // 直接失败
            framework.Failf("Error deleting orphaned namespaces: %v", err)
        }
        klog.Infof("Waiting for deletion of the following namespaces: %v", deleted)
                            // 有很多类似的，等待操作完成的函数
                                                                // 很多可用的常量
        if err := framework.WaitForNamespacesDeleted(c, deleted, framework.NamespaceCleanupTimeout); err != nil {
            framework.Failf("Failed to delete orphaned namespaces %v: %v", deleted, err)
        }
    }
 
    // 对于大型集群，执行到这里时，可能很多节点的路由还没有同步，因此不支持调度
    // 下面的方法等待直到所有节点可调度
    framework.ExpectNoError(framework.WaitForAllNodesSchedulable(c, framework.TestContext.NodeSchedulableTimeout))
 
    // 如果没有指定节点数量，自动计算
    if framework.TestContext.CloudConfig.NumNodes == framework.DefaultNumNodes {
        //            获取可调度节点数
        nodes, err := e2enode.GetReadySchedulableNodes(c)
        // 断言
        framework.ExpectNoError(err)
        framework.TestContext.CloudConfig.NumNodes = len(nodes.Items)
    }
 
    // 在测试之前，确保所有Pod已经Running/Ready，否则没有就绪的集群基础设施Pod可能阻止测试Pod
    // 运行
    podStartupTimeout := framework.TestContext.SystemPodsStartupTimeout
    if err := e2epod.WaitForPodsRunningReady(c, metav1.NamespaceSystem, int32(framework.TestContext.MinStartupPods), int32(framework.TestContext.AllowedNotReadyNodes), podStartupTimeout, map[string]string{}); err != nil {
        // Dump出指定命名空间的事件、Pod、节点信息
        framework.DumpAllNamespaceInfo(c, metav1.NamespaceSystem)
        // 对失败的容器执行kubectl logs                             Logf为Info级别日志器
        framework.LogFailedContainers(c, metav1.NamespaceSystem, framework.Logf)
        // 运行一个测试容器，尝试连接到API Server，等待此容器Ready，打印其标准输出，退出
        runKubernetesServiceTestContainer(c, metav1.NamespaceDefault)
        framework.Failf("Error waiting for all pods to be running and ready: %v", err)
    }
    //                  等待DaemonSets全部就绪
    if err := framework.WaitForDaemonSets(c, metav1.NamespaceSystem, int32(framework.TestContext.AllowedNotReadyNodes), framework.TestContext.SystemDaemonsetStartupTimeout); err != nil {
        framework.Logf("WARNING: Waiting for all daemonsets to be ready failed: %v", err)
    }
 
    // 打印服务器和客户端版本信息
    framework.Logf("e2e test version: %s", version.Get().GitVersion)
 
    dc := c.DiscoveryClient
 
    serverVersion, serverErr := dc.ServerVersion()
    if serverErr != nil {
        framework.Logf("Unexpected server error retrieving version: %v", serverErr)
    }
    if serverVersion != nil {
        framework.Logf("kube-apiserver version: %s", serverVersion.GitVersion)
    }
 
    if framework.TestContext.NodeKiller.Enabled {
        nodeKiller := framework.NewNodeKiller(framework.TestContext.NodeKiller, c, framework.TestContext.Provider)
        // NodeKiller负责周期性的模拟节点失败
        go nodeKiller.Run(framework.TestContext.NodeKiller.NodeKillerStopCh)
    }
}
```


setupSuite执行完毕之后，Ginkgo会运行子目录中的数千个Specs。

#### 编写Spec

Kubernetes项目中有大量例子可以参考，例如：

```go
package e2e
 
import (
    "fmt"
    "path/filepath"
    "sync"
    "time"
 
    rbacv1 "k8s.io/api/rbac/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime/schema"
    "k8s.io/apiserver/pkg/authentication/serviceaccount"
    clientset "k8s.io/client-go/kubernetes"
    podutil "k8s.io/kubernetes/pkg/api/v1/pod"
    commonutils "k8s.io/kubernetes/test/e2e/common"
    "k8s.io/kubernetes/test/e2e/framework"
    "k8s.io/kubernetes/test/e2e/framework/auth"
    e2epod "k8s.io/kubernetes/test/e2e/framework/pod"
    "k8s.io/kubernetes/test/e2e/framework/testfiles"
 
    "github.com/onsi/ginkgo"
)
 
const (
    serverStartTimeout = framework.PodStartTimeout + 3*time.Minute
)
 
// 声明一个ginkgo.Describe块，自动添加[k8s.io] 标签
var _ = framework.KubeDescribe("[Feature:Example]", func() {
    // 创建一个新的Framework对象，自动提供：
    //   BeforeEach：创建K8S客户端、创建命名空间、启动资源用量收集器、指标收集器
    //   AfterEach：调用cleanupHandle、删除命名空间
    f := framework.NewDefaultFramework("examples")
 
    var c clientset.Interface
    var ns string
    // 自己可以添加额外的Setup/Teardown块
    ginkgo.BeforeEach(func() {
        // 获取客户端、使用的命名空间
        c = f.ClientSet
        ns = f.Namespace.Name
 
        // 在命名空间级别绑定RBAC权限，给default服务账号授权
        err := auth.BindClusterRoleInNamespace(c.RbacV1(), "edit", f.Namespace.Name,
            rbacv1.Subject{Kind: rbacv1.ServiceAccountKind, Namespace: f.Namespace.Name, Name: "default"})
        framework.ExpectNoError(err)
        // 等待操作完成
        err = auth.WaitForAuthorizationUpdate(c.AuthorizationV1(),
            serviceaccount.MakeUsername(f.Namespace.Name, "default"),
            f.Namespace.Name, "create", schema.GroupResource{Resource: "pods"}, true)
        // 断言
        framework.ExpectNoError(err)
    })
 
    // 嵌套的Describe
    framework.KubeDescribe("Liveness", func() {
        // 第一个Spec：测试健康检查失败的Pod能否自动重启
        ginkgo.It("liveness pods should be automatically restarted", func() {
            test := "test/fixtures/doc-yaml/user-guide/liveness"
            // 读取文件，Go Template形式，并解析为YAML资源清单
            execYaml := readFile(test, "exec-liveness.yaml.in")
            httpYaml := readFile(test, "http-liveness.yaml.in")
            nsFlag := fmt.Sprintf("--namespace=%v", ns)
 
            // 调用Kubectl来创建资源
            framework.RunKubectlOrDieInput(execYaml, "create", "-f", "-", nsFlag)
            framework.RunKubectlOrDieInput(httpYaml, "create", "-f", "-", nsFlag)
 
            // 并行测试
            var wg sync.WaitGroup
            passed := true
            // 此函数检查发生了重启
            checkRestart := func(podName string, timeout time.Duration) {
                // 等待Pod就绪
                err := e2epod.WaitForPodNameRunningInNamespace(c, podName, ns)
                framework.ExpectNoError(err)
                // 轮询知道重启次数大于0
                for t := time.Now(); time.Since(t) < timeout; time.Sleep(framework.Poll) {
                    pod, err := c.CoreV1().Pods(ns).Get(podName, metav1.GetOptions{})
                    framework.ExpectNoError(err, fmt.Sprintf("getting pod %s", podName))
                    stat := podutil.GetExistingContainerStatus(pod.Status.ContainerStatuses, podName)
                    framework.Logf("Pod: %s, restart count:%d", stat.Name, stat.RestartCount)
                    if stat.RestartCount > 0 {
                        framework.Logf("Saw %v restart, succeeded...", podName)
                        wg.Done()
                        return
                    }
                }
                framework.Logf("Failed waiting for %v restart! ", podName)
                passed = false
                wg.Done()
            }
            // By用于添加一段文档说明
            ginkgo.By("Check restarts")
 
            // 检查两个Pod
            wg.Add(2)
            for _, c := range []string{"liveness-http", "liveness-exec"} {
                go checkRestart(c, 2*time.Minute)
            }
            wg.Wait()
            // 断言
            if !passed {
                framework.Failf("At least one liveness example failed.  See the logs above.")
            }
        })
    })
 
    framework.KubeDescribe("Secret", func() {
        // 第二个Spec，测试Pod能读取一个保密字典
        ginkgo.It("should create a pod that reads a secret", func() {
            test := "test/fixtures/doc-yaml/user-guide/secrets"
            secretYaml := readFile(test, "secret.yaml")
            podYaml := readFile(test, "secret-pod.yaml.in")
 
            nsFlag := fmt.Sprintf("--namespace=%v", ns)
            podName := "secret-test-pod"
 
            ginkgo.By("creating secret and pod")
            // 创建一个Secret，以及会读取此Secret并打印的Pod
            framework.RunKubectlOrDieInput(secretYaml, "create", "-f", "-", nsFlag)
            framework.RunKubectlOrDieInput(podYaml, "create", "-f", "-", nsFlag)
            // 等待Pod退出
            err := e2epod.WaitForPodNoLongerRunningInNamespace(c, podName, ns)
            framework.ExpectNoError(err)
 
            ginkgo.By("checking if secret was read correctly")
            // 检查Pod日志
            _, err = framework.LookForStringInLog(ns, "secret-test-pod", "test-container", "value-1", serverStartTimeout)
            framework.ExpectNoError(err)
        })
    })
 
})
 
func readFile(test, file string) string {
    from := filepath.Join(test, file)
    return commonutils.SubstituteImageName(string(testfiles.ReadOrDie(from)))
}
```


#### 测试分类

可以为E2E测试添加标签，以区分类别：

| 标签 |说明 |
| ----------- |----------- |
| 无 |测试可以快速（5m以内）完成，支持并行测试，具有一致性 |
| [Slow] |运行时间超过5分钟 |
| [Serial] |不支持和其它测试并行执行 |
| [Disruptive] |可能影响（例如重启组件、Taint节点）不是该测试自己创建的工作负载。任何Disruptive测试自动是Serial的 |
| [Flaky] |标记测试中的问题难以短期修复。这种测试默认情况下不会运行，除非使用focus/skip参数 |
| [Feature:.+] |如果一个测试运行/处理非核心功能，因此需要排除出标准测试套件，使用此标签。 |
| [LinuxOnly] |需要使用Linux特有的特性 |
此外，任何测试都必须归属于某个SIG，并具有对应的`[sig-<name>]`标签。每个e2e的子包都在framework.go中SIGDescribe函数，来添加此标签：

```go
// SIGDescribe annotates the test with the SIG label.
func SIGDescribe(text string, body func()) bool {
    return framework.KubeDescribe("[sig-node] "+text, body)
}
```


测试可以具有多个标签，使用空格分隔即可。

#### 1.13新特性

##### 隔离提供商相关测试

1.12以及更老版本的E2E框架难以使用的原因是，它依赖于大量云提供商的私有SDK，这需要拉取大量的包，甚至编译通过都很困难。这些包里面，很多都仅仅被一部分测试所需要，因此从1.13开始和云提供商相关的测试，和**K8S核心的测试被隔离**开来，前者现在转移到`test/e2e/framework/providers`包中，E2E框架通过接口[ProviderInterface](https://github.com/kubernetes/kubernetes/blob/6c1e64b94a3e111199c934c39a0c25bc219ed5f9/test/e2e/framework/provider.go#L79-L99)和这些云提供商相关测试。

测试套件的实现者决定需要引入哪些提供商的包，并且配合kubetest的命令行选项`--provider`激活之。1.13-1.14的二进制文件e2e.test支持1.12的所有providers，如果你不引用任何提供商的包，则仅仅通用providers可用，包括：

1. skeleton，仅仅通过K8S API访问集群，没有其它方式

2. local，类似local，但是支持通过脚本kubernetes/kubernetes/cluster中的脚本拉取日志

##### 外部文件

很多测试套件需要在运行时读取额外的文件，例如YAML清单。 e2e.test二进制文件倾向于是自包含的，以提升可移植性。在之前，所有test/e2e/testing-manifests中的文件会被打包（使用[go-bindata](https://github.com/jteeuwen/go-bindata)）到e2e.test中，现在则是可选的。当通过[testfiles](https://github.com/kubernetes/kubernetes/blob/v1.13.0/test/e2e/framework/testfiles/testfiles.go)包访问文件时，e2e.test从不同地方获取文件：

1. 想对于`--repo-root`参数所指定的目录

2. 从0-N个bindata块中获取

##### 从YAML创建资源

在1.12中，你可以从YAML中加载一个个的资源，但是必须手工创建它。现在提供了新的[方法](https://github.com/kubernetes/kubernetes/blob/v1.13.0/test/e2e/framework/create.go)从YAML中加载多个资源并Patch之（例如设置命名空间）、创建之。

### E2E Utils库

此库是定义在[kubernetes/test/e2e/framework/util.go](https://github.com/kubernetes/kubernetes/blob/master/test/e2e/framework/util.go)中的一系列常量、变量、函数：

1. 常量：各种操作的超时值、集群节点数量、CPU剖析采样间隔

2. 变量：各种常用镜像的URL，例如BusyBox

3. 函数：

    1. 命名空间操控：

        1. CreateTestingNS：创建一个新的，供当前测试使用的命名空间

        2. DeleteNamespaces：删除命名空间

        3. CheckTestingNSDeletedExcept：检查所有e2e测试创建的命名空间出于Terminating状态，并且阻塞直到删除

        4. Cleanup：读取文件中的清单，从指定命名空间中删除它们，并检查命名空间中匹配指定selector的资源正确停止

    2. 节点操控：

        1. 设置Label、设置Taint

        2. RemoveLabelOffNode、RemoveTaintOffNode 删除Label/Taint

        3. AllNodesReady：检查所有节点是否就绪

        4. GetMasterHost：获取哦Master的主机名

        5. NodeHasTaint：判断节点是否具有Taint

    3. Pod操控：

        1. CreateEmptyFileOnPod：在Pod中创建文件

        2. LookForStringInPodExec：在Pod中执行命令并搜索输出

        3. LookForStringInLog：搜索Pod日志

        4. WaitForAllNodesSchedulabl：e等待节点可调度

    4. 日志和调试：

        1. CoreDump：登陆所有节点并保存日志到指定目录

        2. DumpDebugInfo：输出测试的调试信息

        3. DumpNodeDebugInfo：输出节点的调试信息

    5. 网络操控：

        1. BlockNetwork：通过操控Iptables阻塞两个节点之间的网络

        2. UnblockNetwork：解除阻塞

    6. 控制平面操控：

        1. RestartApiserver：重启API Server

        2. RestartKubelet：重启Kubelet

    7. 断言，若干Expect***、Fail***函数

    8. 收集CPU、内存等剖析信息Gather***

    9. 执行Kubectl命令：KubectlCmd

    10. 创建K8S客户端：

        1. LoadConfig：加载K8S配置

        2. LoadClientset：创建客户端对象

    11. Ginkgo API封装：

        1. KubeDescribe

    12. 等待各种资源达到某种状态：Wait***

    13. 其它杂项：

        1. OpenWebSocketForURL：打开WebSocket连接

        2. PrettyPrintJSON：格式化JSON

        3. Run***：运行命令

其文档位于[https://godoc.org/k8s.io/kubernetes/test/e2e/framework](https://godoc.org/k8s.io/kubernetes/test/e2e/framework)。

### 最佳实践

#### 可调试性

如果测试失败，应当提供尽可能详细的错误信息：

```go
// 没有错误信息
Expect(err).NotTo(HaveOccurred())
 
// 足够的错误信息
Expect(err).NotTo(HaveOccurred(), "Failed to create %d foobars, only created %d", foobarsReqd, foobarsCreated)
```


另一方面，不要打印过多冗长的日志，可能会干扰错误原因的定位。

#### 支持非专用集群

在K8S项目的CI生命周期中，为了减少延迟、提高资源利用率，可能会复用已有的K8S集群，大规模并行的运行E2E测试。因此你：

1. 不能假设集群中仅仅运行你的测试用例。如果的确需要独占集群，使用`[Serial]`标签

2. 应当避免在测试用例中对集群进行一些可能影响其它测试可靠运行的变更，例如重启节点、断开网络接口，升级集群软件。如果的确需要进行破坏性变更，使用`[Disruptive]`标签，避免并行测试

3. 不要使用没有在API规范中明确声明的Kubernetes API

#### 减少执行时间

在K8S项目的CI生命周期中，有数百个E2E用例需要执行，其中一部分还必须串行化的执行，这导致E2E测试非常耗时。

建议尽所有可能保证你的用例可以在2m以内完成。

#### 容错能力

E2E测试可能运行在不同的云提供商中、不同的负载状况下，底层存储有可能是最终一致性的。因此你的测试用例应该能够容忍偶然发生的，基础设施小故障或延迟：

1. 如果一个资源创建请求是异步的，那么即使在绝大部分情况下它实际上都是“同步”就完成了，也应当坚持假设它是异步的

2. 在高负载时，某些请求可能超时，在Fail测试用例之前，应该考虑Retry几次

#### 关于E2E Framework

此框架能够自动创建名字独特的命名空间，在其中进行测试，并且最终清理一切（删除命名空间）。

需要注意的是，删除命名空间是成本比较高的操作，你创建的资源越少，则清理工作越简单，测试运行的也就越快。

#### 关于E2E Utils库

此库提供了大量可重用的测试相关代码，包括等待资源进入特定的状态、安全和一致性的重试失败操作。



**文章出自 https://blog.gmem.cc/kubernetes-e2e-test，见文章分析透彻，收录以便不时查看。**