// 来源：https://blog.openshift.com/kubernetes-deep-dive-code-generation-customresources/

The more Kubernetes turns into a platform for distributed applications, the more projects will use the provided extension points to build software on a higher level in the stack. CustomResourceDefinitions (CRDs) as introduced in Kubernetes 1.7 as alpha and promoted to beta in 1.8 are a natural building block for many use-cases, especially those which implement the controller (or sometimes called operator) pattern in some way. Moreover, CustomResourceDefinitions are so easy to create and to use.

With Kubernetes 1.8, their use in golang-based projects also becomes much more natural: With user-provided CustomResources we can utilize the same code-generation tools that are used inside of Kubernetes itself or in OpenShift. This post shows how the code-generators work and how you apply them in your own project with minimal lines of code, giving you generated deepcopy functions, typed clients, listers and informers, all with one shell script call and a couple of code annotations. A complete project suitable as a blueprint is available in the openshift-evangelists/crd-code-generation repo.

Code Generation – Why?
Those who have used ThirdPartyResources or CustomResourceDefinition natively in golang might be surprised that suddenly in Kubernetes 1.8, client-go code-generation is required. More specifically, client-go requires that runtime.Object types (CustomResources in golang have to implement the runtime.Object interface) must have DeepCopy methods. Here code-generation comes into play via the deepcopy-gen generator, which can be found in the k8s.io/code-generator repository.

Next to deepcopy-gen there a handful of code-generators that most users of CustomResources want to use:

deepcopy-gen—creates a method func (t* T) DeepCopy() *T for each type T
client-gen—creates typed clientsets for CustomResource APIGroups
informer-gen—creates informers for CustomResources which offer an event based interface to react on changes of CustomResources on the server
lister-gen—creates listers for CustomResources which offer a read-only caching layer for GET and LIST requests.
The last two are the basis for building controllers (or operators as some people call them). In a follow-up blog post we will look at controllers in more in detail. These four code-generator make up a powerful basis to build full-featured, production-ready controllers, using the same mechanisms and packages that the Kubernetes upstream controllers are using.

There are some more generators in k8s.io/code-generator for other contexts, for example, if you build your own aggregated API server, you will work with internal types in addition to versioned types. Conversion-gen will create conversions functions between these internal and external types. Defaulter-gen will take care of defaulting certain fields.

Calling the Code-Generators in Your Project
All the Kubernetes code-generators are implemented on-top of k8s.io/gengo. They share a number of common command line flags. Basically, all the generators get a list of input packages (--input-dirs) which they go through type by type, and output the generated code for. The generated code:

Either goes to the same directory as the input files like for deepcopy-gen (with --output-file-base “zz_generated.deepcopy” to define the file name).
Or they generate into one or multiple output packages (with --output-package) like client-, informer- and lister-gen do (usually generating into pkg/client).
The upper description might sound like long fiddling with command line arguments is necessary to get started, but this is luckily not true: k8s.io/code-generator ships a shell script generator-group.sh that does the heavy lifting of calling the generators with all their special small requirements for the use-case of CustomResources. All you have to do in your project boils down to a one-liner, usually inside of hack/update-codegen.sh:

$ vendor/k8s.io/code-generator/generate-groups.sh all \
github.com/openshift-evangelist/crd-code-generation/pkg/client \ github.com/openshift-evangelist/crd-code-generation/pkg/apis \
example.com:v1
It runs against a project which is set up like the following package tree:


All the APIs are expected below pkg/apis and the clientsets, informers, and listers are created inside pkg/client. In other words, pkg/client is completely generated as is the zz_generated.deepcopy.go file next to the types.go file which contains our CustomResource golang types. Both are not supposed to be modified manually, but created by running:

$ hack/update-codegen.sh
Usually, next to it there is a hack/verify-codegen.sh script as well, which terminates with a non-zero return code if any of the generated files is not up-to-date. This is very helpful to put into a CI script: if a developer modified the files by accident or if the files are just outdated, CI will notice and complain.

Controlling the Generated Code – Tags
While some behaviour of the code-generators is controlled via command line flags as described above (especially the packages to process), a lot more properties are controlled via tags in your golang files.

There are two kind of tags:

Global tags above package in doc.go
Local tags above a type that is processed
Tags in general have the shape // +tag-name or // +tag-name=value, that is, they are written into comments. Depending on the tags, the position of the comment might be important. There are a number of tags which must be in a comment directly above a type (or the package line for a global tag), others must be separated from the type (pr the package line) with at least one empty line in-between. We are working on making this more consistent and less error-prone in the 1.9 release cycle (pull request #53579 and issue #53893). Just be prepared that an empty line might matter. Better follow an example and copy the basic shape.

Global Tags
Global tags are written into the doc.go file of a package. A typical pkg/apis/<apigroup>/<version>/doc.go looks like this:

// +k8s:deepcopy-gen=package,register


// Package v1 is the v1 version of the API.
// +groupName=example.com
package v1
It tells deepcopy-gen to create deepcopy methods by default for every type in that package. If you have types where deepcopy is not necessary or not desired, you can opt-out for such a type with a local tag // +k8s:deepcopy-gen=false. If you do not enable package-wide deepcopy, you have to opt-in to deepcopy for each desired type via // +k8s:deepcopy-gen=true.

Note: The register keyword in the value of the upper example will enable the registration of deepcopy methods into the scheme. This will completely go away in Kubernetes 1.9 because the scheme won’t be responsible anymore to do deepcopies of runtime.Objects. Instead just call yourobject.DeepCopy() or yourobject.DeepCopyObject(). You can and you should do that already today in 1.8-based projects as it is faster and less error-prone. Moreover, you will be prepared for 1.9 which will require this pattern.

Finally, the // +groupName=example.com defines the fully qualified API group name. If you get that wrong, client-gen will produce wrong code. Be warned that this tag must be in the comment block just above package (see Issue #53893).

Local Tags
Local tags are written either directly above an API type or in the second comment block above it. Here is an example types.go for the golang types of our API server deep dive series about CustomResources:

// +genclient
// +genclient:noStatus
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// Database describes a database.
type Database struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec DatabaseSpec `json:"spec"`
}

// DatabaseSpec is the spec for a Foo resource
type DatabaseSpec struct {
    User     string `json:"user"`
    Password string `json:"password"`
    Encoding string `json:"encoding,omitempty"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// DatabaseList is a list of Database resources
type DatabaseList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata"`

    Items []Database `json:"items"`
}
Note that we have enabled deepcopy for all types by default, that is, with possible opt-out. These types, though, are all API types and need the deepcopy. Therefore, we don’t have to switch deepcopy on or off in this example types.go, but only on package-wide in doc.go.

RUNTIME.OBJECT AND DEEPCOPYOBJECT
There is a special deepcopy tag which needs more explanation:

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
If you have tried to use CustomResources with client-go based on Kubernetes 1.8—some people might have had the pleasure already because they accidentally vendored a k8s.op/apimachinery of the master branch—you have hit the compiler error that the CustomResource type does not implement runtime.Object because DeepCopyObject() runtime.Object is not defined on your type. The reason is that in 1.8 the runtime.Object interface was extended with this method signature and hence every runtime.Object has to implement DeepCopyObject. The implementation of DeepCopyObject() runtime.Object is trivial:

func (in *T) DeepCopyObject() runtime.Object {
    if c := in.DeepCopy(); c != nil {
        return c
    } else {
        return nil
    }
}
But luckily you don’t have to implement this for every one of your types, but just put the following local tag above your top-level API types:

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
In our example above both Database and DatabaseList are top-level types because they are used as runtime.Objects. As a rule of thumb, top-level types are those which have metav1.TypeMeta embedded. Also, those are the types which clients are create for using client-gen.

Note, that the // +k8s:deepcopy-gen:interfaces tag can and should also be used in cases where you define API types that have fields of some interface type, for example, field SomeInterface. Then // +k8s:deepcopy-gen:interfaces=example.com/pkg/apis/example.SomeInterface will lead to the generation of a DeepCopySomeInterface() SomeInterface method. This allows it to deepcopy those fields in a type-correct way.

CLIENT-GEN TAGS
Finally, there are a number of tags to control client-gen, two of them we see in our example:

// +genclient
// +genclient:noStatus
The first tag tells client-gen to create a client for this type (this is always opt-in). Note that you don’t have to and indeed MUST not put it above the List type of the API objects.

The second tag tells client-gen that this type is not using spec-status separation via the /status subresource. The resulting client will not have the UpdateStatus method (client-gen would generate that blindly otherwise as soon as it finds a Status field in your struct). A /status subresource is only possible in 1.8 for natively (in golang) implemented resources. But this might change soon as subresources are discussed for CustomResources in PR 913.

For cluster-wide resources, you have to use the tag:

// +genclient:nonNamespaced
For special purpose clients, you might also want to control in detail which HTTP methods are offered by the client. This can be done using a couple of tags, for example:

// +genclient:noVerbs
// +genclient:onlyVerbs=create,delete
// +genclient:skipVerbs=get,list,create,update,patch,delete,deleteCollection,watch
// +genclient:method=Create,verb=create,result=k8s.io/apimachinery/pkg/apis/meta/v1.Status
The first three should be pretty self-explanatory, but the last one needs some explanation. The type where this tag is written above will be create-only and will not return the API type itself, but a metav1.Status. For CustomResources this does not make much sense, but for user-provided API servers written in golang those resources can exist and they do in practice, for example, in the OpenShift API.

A Main Function Using the Types Clients
While most examples based on Kubernetes 1.7 and older used the client-go dynamic client for CustomResources, native Kubernetes API types had a much more convenient typed client for a long time. This changed in 1.8: client-gen as described above creates a native, full-featured, and easy to use typed client also for your custom types. Actually, client-gen does not know whether you are applying it to a CustomResource type or a native one.
Hence, using this client turns out to be exactly equivalent to using a client-go Kubernetes client. Here is a very simple example:

import (
    ...
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/tools/clientcmd"
    examplecomclientset "github.com/openshift-evangelist/crd-code-generation/pkg/client/clientset/versioned"
)

var (
    kuberconfig = flag.String("kubeconfig", "", "Path to a kubeconfig. Only required if out-of-cluster.")
    master      = flag.String("master", "", "The address of the Kubernetes API server. Overrides any value in kubeconfig. Only required if out-of-cluster.")
)

func main() {
    flag.Parse()

    cfg, err := clientcmd.BuildConfigFromFlags(*master, *kuberconfig)
    if err != nil {
        glog.Fatalf("Error building kubeconfig: %v", err)
    }

    exampleClient, err := examplecomclientset.NewForConfig(cfg)
    if err != nil {
        glog.Fatalf("Error building example clientset: %v", err)
    }

    list, err := exampleClient.ExampleV1().Databases("default").List(metav1.ListOptions{})
    if err != nil {
        glog.Fatalf("Error listing all databases: %v", err)
    }

    for _, db := range list.Items {
        fmt.Printf("database %s with user %q\n", db.Name, db.Spec.User)
    }
}
It works with a kubeconfig file, in fact the same that can be used with kubectl and the Kubernetes clients.

In contrast to old legacy TPR or CustomResource code with the dynamic client, you don’t have to type-cast. Instead, the actual client call looks completely native and it is:

list, err := exampleClient.ExampleV1().Databases("default").List(metav1.ListOptions{})
The result is a DatabaseList in this example of all databases in your cluster. If you switch your type to cluster-wide (i.e. without namespaces; don’t forget to tell client-gen with the // +genclient:nonNamespaced tag!), the calls turns into

list, err := exampleClient.ExampleV1().Databases().List(metav1.ListOptions{})
Creating a CustomResourceDefinition Programmatically in Golang
As this question comes up quite often, a few words about how to create a CRD programmatically from your golang code.

Client-gen always creates so-called clientsets. Clientsets bundle one or more API groups into one client. Usually, these API groups come from one repository and a placed inside a base package, for example, your pkg/apis as in the example in this blog post, or from k8s.io/api in the case of Kubernetes.

CustomResourceDefinitions are provided by the
[kubernetes/apiextensions-apiserver repository](https://github.com/kubernetes/apiextensions-apiserver repository). This API server (that can also be launched stand-alone) is embedded by kube-apiserver, such that CRDs are available on every Kubernetes cluster. But the client to create CRDs is created into the apiextensions-apiserver repository, of course also using client-gen. After having read this blog, it should not surprise you to find the client at kubernetes/apiextensions-apiserver/tree/master/pkg/client, nor should it be unexpected what it looks like to create a client instance and how to create the CRD:

import (
    ...
    apiextensionsclientset "k8s.io/apiextensions-apiserver/pkg/client/clientset/clientset”
)

apiextensionsClient, err := apiextensionsclientset.NewForConfig(cfg)
...
createdCRD, err := apiextensionsClient.ApiextensionsV1beta1().CustomResourceDefinitions().Create(yourCRD)
Note that after this creation you will have to wait that the Established condition is set on the new CRD. Only then will the kube-apiserver start to serve the resource. If you don’t wait for that condition, every CR operation will return a 404 HTTP status code.

Further Material
The documentation of the Kubernetes generators has a lot of room for improvement, currently, and any kind of help is very welcome. They are freshly extracted from the Kubernetes project into k8s.io/code-generator for public consumption by CustomResource users. The documentation will certainly improve over time and this blog post also aims to contribute to this.

For further information about the different generators it is often good to look at examples inside Kubernetes itself (for example, in k8s.io/api), into OpenShift, which has many advanced use-cases, and into the generators themselves:

Deepcopy-gen has some documentation available inside its main.go file.
Client-gen has some documentation available here.
Informer-gen and lister-gen currently don’t posses further documentation, however generate-groups.sh shows how each is invoked.
All the examples in this blog post are available as a fully functioning repository, which can easily serve as a blueprint for your own experiments:

https://github.com/openshift-evangelists/crd-code-generation.