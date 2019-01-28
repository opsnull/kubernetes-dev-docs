# Kubernetes API Client

	资源对象的 schema（位于 k8s.io/api 目录的各 resource group 目录下）；
		定义对象 struct；
		生成 deepcopy，实现 Runtime.Object 接口；
		注册到 Schema，可能是 clientset 使用的全局 Schema，也可能是 apiserver 使用
	资源对象的 Lister
	资源对象的 Informer，Informer 基于 Indexer 实现，可以生成 Lister；
		一般使用 InformerFactory 创建 Informer，再获取 Lister；
	操作资源对象的 clientset，各种类型 client 的集合，所以称为 set；自动编解码；

## 注册资源类型 Scheme

Scheme 定义了资源对象的类型，和编解码方式。

scheme package（k8s.io/client-go/kubernetes/scheme/）的 `init()` 函数将所有 K8S 内置资源对象的 Scheme 注册到创建的全局变量 `Scheme` 对象上，并用 Scheme 对象创建编解码对象 `Codecs` 和参数编解码对象 `ParameterCodec`。

``` go
// 来源于 k8s.io/client-go/kubernetes/scheme/register.go
// 新建一个 Scheme，后续所有 K8S 类型均添加到该 Scheme；
var Scheme = runtime.NewScheme()
// 为 Scheme 中的所有类型创建一个编解码工厂；
var Codecs = serializer.NewCodecFactory(Scheme)
// 为 Scheme 中的所有类型创建一个参数编解码工厂
var ParameterCodec = runtime.NewParameterCodec(Scheme)

var localSchemeBuilder = runtime.SchemeBuilder{
	admissionregistrationv1alpha1.AddToScheme,
    ...
	extensionsv1beta1.AddToScheme,
    ...
}
var AddToScheme = localSchemeBuilder.AddToScheme

func init() {
    v1.AddToGroupVersion(Scheme, schema.GroupVersion{Version: "v1"})
    // 将所有 K8S Scheme 注册到 Scheme
	utilruntime.Must(AddToScheme(Scheme))
}
```

## 创建 Kubernetes Clientset

``` go
cfg, err := clientcmd.BuildConfigFromFlags(masterURL, kubeconfig)
if err != nil {
    klog.Fatalf("Error building kubeconfig: %s", err.Error())
}

kubeClient, err := kubernetes.NewForConfig(cfg)
if err != nil {
    klog.Fatalf("Error building kubernetes clientset: %s", err.Error())
}
```

## Kubernetes 的 Clientset

``` go
// 来源于 k8s.io/client-go/kubernetes/clientset.go
// 传入的 rest.Config 包含 apiserver 服务器和认证信息
func NewForConfig(c *rest.Config) (*Clientset, error) {
	configShallowCopy := *c
    ...
    // 透传 rest.COnfig，调用具体分组和版本的资源类型的 ClientSet 构造函数
	cs.extensionsV1beta1, err = extensionsv1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
    ...
	cs.DiscoveryClient, err = discovery.NewDiscoveryClientForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	return &cs, nil
}

// ExtensionsV1beta1 retrieves the ExtensionsV1beta1Client
func (c *Clientset) ExtensionsV1beta1() extensionsv1beta1.ExtensionsV1beta1Interface {
	return c.extensionsV1beta1
}
```

## 创建具体资源类型的 Clientset

``` go
// 来源于 k8s.io/client-go/kubernetes/typed/extensions/v1beta1/extensions_client.go
// 传入的 rest.Config 包含 apiserver 服务器和认证信息
func NewForConfig(c *rest.Config) (*ExtensionsV1beta1Client, error) {
    config := *c
    // 为 rest.Config 设置资源对象相关的参数
	if err := setConfigDefaults(&config); err != nil {
		return nil, err
    }
    // 创建 ExtensionsV1beta1 的 RestClient
	client, err := rest.RESTClientFor(&config)
	if err != nil {
		return nil, err
	}
	return &ExtensionsV1beta1Client{client}, nil
}

func setConfigDefaults(config *rest.Config) error {
    // 资源对象的 GroupVersion
	gv := v1beta1.SchemeGroupVersion
    config.GroupVersion = &gv
    // 资源对象的 root path
    config.APIPath = "/apis"
    // 使用注册的资源类型 Schema 对请求和响应进行编解码
    // scheme 为前面分析过的 k8s.io/client-go/kubernetes/scheme package
	config.NegotiatedSerializer = serializer.DirectCodecFactory{CodecFactory: scheme.Codecs}

	if config.UserAgent == "" {
		config.UserAgent = rest.DefaultKubernetesUserAgent()
	}

	return nil
}

func (c *ExtensionsV1beta1Client) Deployments(namespace string) DeploymentInterface {
	return newDeployments(c, namespace)
}
```

## 特定资源类型的 Rest 方法实现

``` go
// 来源于 k8s.io/client-go/kubernetes/typed/extensions/v1beta1/deployment.go
// newDeployments returns a Deployments
func newDeployments(c *ExtensionsV1beta1Client, namespace string) *deployments {
	return &deployments{
		client: c.RESTClient(),
		ns:     namespace,
	}
}

// Get takes name of the deployment, and returns the corresponding deployment object, and an error if there is any.
func (c *deployments) Get(name string, options v1.GetOptions) (result *v1beta1.Deployment, err error) {
	result = &v1beta1.Deployment{}
	err = c.client.Get().
		Namespace(c.ns).
		Resource("deployments").
		Name(name).
		VersionedParams(&options, scheme.ParameterCodec).
		Do().
		Into(result)
	return
}
```