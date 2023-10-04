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
same service. Similarly you can create more than one `ServiceBinding` resource
pointing to the same workload. The service, workload and ServiceBinding
resources must be in the same namespace.

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
present, a location is assigned (often `/bindings`) and the value is set as the
`SERVICE_BINDING_ROOT` environment variable. This way application can rely on
`SERVICE_BINDING_ROOT` environment variable to lookup bindings.

The ServiceBinding resource allows to override values of `type` and `provider`
entries.  You can set it through `.spec.type` and `.spec.provider` attributes.

# Service Binding Status

TODO

# Workload Resource Mapping

By default, an implementation assumes that it is projecting binding information
into workloads with a shape similar to `Deployment.apps` resources.  For
instance, it may assume that it can find volume information at
`.spec.template.spec.volumes`.

This assumption does not work with workloads of a different resource shape,
such as a `CronJob`.  To support these cases, you can use a
`ClusterWorkloadResourceMapping.servicebinding.io` resource to inform the
`ServiceBinding` controller how it can project information into these resources.

## Resource Schema

`ClusterWorkloadResourceMapping` resources have the following shape:

```yaml
apiVersion: servicebinding.io/v1beta1
kind: ClusterWorkloadResourceMapping
metadata:
  name: # string (1)
spec:
  versions:         # []MappingTemplate (2)
  - version:        # string (3)
    annotations:    # string (Fixed JSONPath), optional (4)
    containers:     # []MappingContainer, optional (5)
    - path:         # string (JSONPath) (6)
      name:         # string (Fixed JSONPath), optional (7)
      env:          # string (Fixed JSONPath), optional (8)
      volumeMounts: # string (Fixed JSONPath), optional (9)
    volumes:        # string (Fixed JSONPath), optional (10)
```

Some notes:

1. These resources need to be named after the plural name of the resource.  For
   instance, to define mappings for `CronJob` resources, the mapping needs to
   be named `cronjobs.batch`.  This is the same pattern used to name
   `CustomResourceDefinitions`s.
2. Entries in this array define the resource version that this template
   handles.  Each mapping supports all versions of the target resource.
3. The version here must either match a version exactly (e.g. `v1alpha1`) or
   can be `*` to match all versions that don't have an exact match.  A workload
   that does not match a defined mapping version reverts to the default
   mapping.
4. Some fields in the mapping must conform to what the spec calls a "Fixed
   JSONPath".  This is a JSONPath, but restricted to dot (`.`) and array access
   (`foo['bar']`) operators.  For more information, please refer to [the
   spec](https://github.com/servicebinding/spec#fixed-jsonpath).

   This field indicates where annotations for the generated pod might be found.
   Defaults to `.spec.template.metadata.annotations`.
5. Entries in this array define where a container might be found.  This lets a
   single version refer to multiple container fields, such as containers and
   init containers.  Arrays of containers must be expanded to resolve sets of
   individual containers, for example `.spec.template.spec.containers[*]`.
6. Defines the root of where container information may be found.
7. Indicates the name of the container.  If not specified, selective binding to
   containers by name is not supported for this mapping.
8. Indicates where environment variables can be set in the container.  Defaults
   to `.env`.
9. Indicates where volume mount information can be set in the container.
   Defaults to `.volumeMounts`.
10. Indicates where volumes can be specified for the generated pod.  Defaults
    to `.spec.template.spec.volumes`

## Examples

For instance, `cronjobs.batch` resources could be made bindable with the
following mapping:

```yaml
apiVersion: servicebinding.io/v1beta1
kind: ClusterWorkloadResourceMapping
metadata:
  name: cronjobs.batch
spec:
  versions:
  - version: "*"
    annotations: .spec.jobTemplate.spec.template.metadata.annotations
    containers:
    - path: .spec.jobTemplate.spec.template.spec.containers[*]
    - path: .spec.jobTemplate.spec.template.spec.initContainers[*]
    volumes: .spec.jobTemplate.spec.template.spec.volumes
```

# Role-Based Access Control (RBAC)

TODO

[sb]: https://github.com/servicebinding/spec#service-binding
