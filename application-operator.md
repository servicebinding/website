---
description: Service Bindings for Application Operators
permalink: /application-operator/
---

Application operators bind application workloads with services by creating
`ServiceBinding` resources. The specification's [Service Binding section][sb]
describes this in detail. The `ServiceBinding` resource need to specify the
service and workload details. The resource `kind` and `apiVersion` are required
for both the service and workload. For service, you need to specify the name of
the resource. For workload, you can specify the name of the resource or a label
selector to identify one more workloads. If label selector is used for workloads
and there are more than one workload matching the label selector, all of those
workloads are going to bound to the same service.

Here is an example `ServiceBinding` resource with workload identified by name:

```yaml
apiVersion: servicebinding.io/v1beta1
kind: ServiceBinding
metadata:
  name: account-service
spec:
  service:
    apiVersion: com.example/v1alpha1
    kind: AccountService
    name: prod-account-service
  workload:
    apiVersion: apps/v1
    kind: Deployment
    name: online-banking
```

Here is an example `ServiceBinding` resource with workload identified by label selector:

```yaml
apiVersion: servicebinding.io/v1beta1
kind: ServiceBinding
metadata:
  name: online-banking-frontend-to-account-service
spec:
  name: account-service
  service:
    apiVersion: com.example/v1alpha1
    kind: AccountService
    name: prod-account-service
  workload:
    apiVersion: apps/v1
    kind: Deployment
    selector:
      matchLabels:
        app.kubernetes.io/part-of: online-banking
        app.kubernetes.io/component: frontend
```

It is possible to create more than one `ServiceBinding` resource pointing to the
same service. Similarly you can more than one `ServiceBinding` resource pointing
to the same workload. The service, workload and binding resources must be in the same
namespace.

By default, the bindings are projected into all regular containers and init
containers. But is possible to restrict the projection based a name based
filter. You can create a `.spec.workload.containers` as a list of container
names. If it exists, then only the containers matching that name is going to be
used to project bindings.

Here is an example with restricted container names:

```yaml
apiVersion: servicebinding.io/v1beta1
kind: ServiceBinding
metadata:
  name: account-service
spec:
  service:
    apiVersion: com.example/v1alpha1
    kind: AccountService
    name: prod-account-service
  workload:
    apiVersion: apps/v1
    kind: Deployment
    name: online-banking
    containers:
    - app
    - backup
```

The ServiceBinding supports projecting values as environment variables. To
project environment variables, define `.spec.env` with a list of entries. Each
entry has a name which is going to be the environment variable and key referring
to a binding Secret key and its value is the value of environment
variable.

Here is an example with environment variables:

```yaml
apiVersion: servicebinding.io/v1beta1
kind: ServiceBinding
metadata:
  name: account-service
spec:
  service:
    apiVersion: com.example/v1alpha1
    kind: AccountService
    name: prod-account-service
  workload:
    apiVersion: apps/v1
    kind: Deployment
    name: online-banking
  env:
  - name: ACCOUNT_SERVICE_HOST
    key: host
  - name: ACCOUNT_SERVICE_USERNAME
    key: username
  - name: ACCOUNT_SERVICE_PASSWORD
    key: password
```

Note: You can use the built-in language feature of your programming language of
choice to read environment variables from the application. If there is a change
in the projected bindings, the container must restart to read updated
environment variable values.

The ServiceBinding controller implements a reconciler to bind the Provisioned
Service binding Secret into the workload. The Secret referred to by
`.status.binding` on the resource represented by service is mounted as a volume
on the resource represented by workload. The directory name of the volume mount
is same as the value of `.metadata.name`. But it can override by setting a
`.spec.name` attribute.

The directory where the volume is mounted is decided based on the value of
environment variable `SERVICE_BINDING_ROOT`. If that environment variable is not
present, a location is assigned (often `/bindings`) and the value is set as the `SERVICE_BINDING_ROOT`
environment variable. This way application can rely on `SERVICE_BINDING_ROOT`
environment variable to lookup bindings.

The ServiceBinding resource allows to override values of `type` and `provider`
entries.  You can set it through `.spec.type` and `.spec.provider`.

# Service Binding Status

TODO

# Workload Resource Mapping

TODO

# Role-Based Access Control (RBAC)

TODO

[sb]: https://github.com/servicebinding/spec#service-binding
