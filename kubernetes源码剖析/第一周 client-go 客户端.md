### Client 客户端对象

RESTClient 最基础的客户端。派生出下列对象：

+ ClientSet

  只能处理 Kubernetes 内置资源，不能直接访问 CRD 资源，可通过 client-gen 在 ClientSet 集合中自动生成 CRD 操作相关的接口。

+ DynamicClient

  能够处理 CRD 自定义资源，将 Resource 转换成 Unstructured 结构类型（ma p[string]interface{}），再将 Unstructured 转换成对应的资源对象。

+ DiscoveryClient

  主要用于发现 Kubernetes API Server 所支持的资源组、资源版本、资源信息。

RESTClient、ClientSet、DynamicClient 可对资源执行 Create、Update、Delete、Get、List、Watch、Patch 等操作。