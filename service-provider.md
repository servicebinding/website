---
description: Service Bindings for Service Providers
permalink: /service-provider/
---

Service providers expose bindings through a `Secret` resource with data required for connectivity.  The specification's [Provisioned Service section][provisioned-service] describes this in detail.  Alternatively, the specification also supports [Direct Secret Reference][direct-secret-reference].  The only requirement for Direct Secret Reference is a `Secret` resource with data required for connectivity.  Alternatively, if you are creating a Provisioned Service (preferred approach), you also need a custom resource with `.status.binding.name` attribute pointing to the `Secret` resource.

Here is an example custom resource:

```yaml
apiVersion: example.dev/v1beta1
kind: Database
metadata:
  name: database-service
...
status:
  binding:
    name: production-db-secret
```


{% include note.html content="Implementing the Provisioned Service contract frees the `ServiceBinding` creator from needing to know about the name of the `Secret` holding the credentials.  The service can update the secret name exposed over time.  This behavior not only decouples the ServiceBinding and the workload from needing to know the name of the Secret, it also enables use of [immutable secrets](https://kubernetes.io/docs/concepts/configuration/secret/#secret-immutable).  An immutable secret cannot be updated after it is created.  To rotate the credentials, a new secret can be created and update the same in the Provisioned Service resource's `.status.binding.name` attribute." %}

In the previous example, `production-db-secret` is the name of the `Secret` resource with data entries required for connectivity.  The `Secret` resource should contain a `type` entry that can be used for identifying the service.  It helps the application to identify the service as a relational database, key-value store, or a cache server.  There is no standardization on the value for `type`, but you can see some good examples in the [Spring Cloud Bindings][spring-cloud-bindings].  A few examples:

- `postgresql`
- `mysql`
- `redis`
- `mongodb`

The `Secret` resource's [`type` field](https://kubernetes.io/docs/concepts/configuration/secret/#secret-types) should have a special value based on the `type` data entry value.  The expected format is `servicebinding.io/{type}`.  For example, if the value for `type` data entry is `mysql`, then the `Secret` resource's `type` field value should be `servicebinding.io/mysql`.  This recommendation helps to query `Secret` resources of particular type using field-selector. For example:

```bash
kubectl get secrets --field-selector="type=servicebinding.io/mysql"
```

will give the `Secret` resources of `mysql` type.

Similar to the `type` data entry, spec also recommends to add a `provider` entry to identify the provider.  The `provider` data entry is a further classification of the type.

Here is an example `Secret`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: production-db-secret
type: servicebinding.io/mysql
stringData:
  type: mysql
  provider: bitnami
  host: localhost
  port: 3306
  username: root
  password: root
```

In this example, `bitnami` is the provider indicating the service was provisioned from the Bitnami catalog.  Use any appropriate value for services you provision that indicates to workloads how to consume the service. For a known type and provider, a workload should be able to know how to consume the service.

{% include tip.html content="It is possible to override the `type` and `provider` values with the `ServiceBinding` resource. This capability is intended for users to consume existing `Secrets` that predate the Service Binding Specification. Service providers should set these values so that users don't need to do it themselves manually." %}

# Well-known Secret Entries

Apart from the special `type`, and `provider` entries in the `Secret` data, there are few special words, if used must follow certain restrictions for the values.  These are called well-known entries.

| Name | Requirements
| ---- | ------------
| `host` | A DNS-resolvable host name or IP address
| `port` | A valid port number
| `uri` | A valid URI as defined by [RFC3986](https://tools.ietf.org/html/rfc3986)
| `username` | A string-based username credential
| `password` | A string-based password credential
| `certificates` | A collection of PEM-encoded X.509 certificates, representing a certificate chain used in mTLS client authentication
| `private-key` | A PEM-encoded private key used in mTLS client authentication


If there is any entry that doesn’t follow the given requirement, you can choose different names. For example, if there is a URI-like string but not a valid one, as per RFC-3986, use another name (e.g., "custom-uri"). If a service has multiple values for a key, consider a dot-delimited prefix (e.g. "amqp.port" or "mqtt.port") reusing the prefix with other keys as appropriate.

# Considerations for Role-Based Access Control (RBAC)

As a service provider, you can create a `ClusterRole` with the label
`servicebinding.io/controller=true` and the verbs `get`, `list`, and `watch`
listed in the rules.  Service Binding implementations use these permissions to lookup the service resource and read the binding secret name from its status when referenced by a ServiceBinding.  Here is an example `ClusterRole`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: examples-service-bindings
  labels:
    servicebinding.io/controller: "true" # matches the aggregation rule selector
rules:
- apiGroups:
  - example.dev
  resources:
  - databases
  verbs:
  - get
  - list
  - watch
```

In the above example, the API group for the backing service CRD is
`example.dev` and the resource name (plural form) is `databases`.  You can
change those values as per your Provisioned Service.  When your operator is
getting installed, make sure this cluster role is also installed.

# Real World Examples

1. [RabbitMQ Operator](https://github.com/rabbitmq/cluster-operator/pull/615)
2. [Kafka Access Operator](https://github.com/strimzi/kafka-access-operator)
3. [External Secrets Operator](https://external-secrets.io/)

[provisioned-service]: https://github.com/servicebinding/spec#provisioned-service
[direct-secret-reference]: https://github.com/servicebinding/spec#direct-secret-reference
[spring-cloud-bindings]: https://github.com/spring-cloud/spring-cloud-bindings
