资源对象的 schema（位于 k8s.io/api 目录的各 resource group 目录下）；
    定义对象 struct；
    生成 deepcopy，实现 Runtime.Object 接口；
    注册到 Schema，可能是 clientset 使用的全局 Schema，也可能是 apiserver 使用
资源对象的 Lister
资源对象的 Informer，Informer 基于 Indexer 实现，可以生成 Lister；
    一般使用 InformerFactory 创建 Informer，再获取 Lister；
操作资源对象的 clientset，各种类型 client 的集合，所以称为 set；自动编解码；