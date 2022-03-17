---
description: Service Bindings for Service Providers
permalink: /service-provider/
---

Service providers expose bindings through a `Secret` resource with data required for connectivity.  The specification's [Provisioned Service section][provisioned-service] describes this in detail.  Alternatively, the specification also support [Direct Secret Reference][direct-secret-reference].  The only important part required for Direct Secret Reference is a `Secret` resource with data required for connectivity.  But if you are creating a Provisioned Service (preferred approach), you also need a custom resource with `.status.binding.name` attribute pointing to the `Secret` resource.

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

In the above example, `production-db-secret` is the name of the `Secret` resource with data entries required for connectivity.  The `Secret` resource should contain a `type` entry that can be used for identifying the service.  It helps the application to indentify the service as a relational database, key-value store, or a cache server.  There is no standardization on the value for `type`, but you can see some good examples in the [Spring Cloud Bindings][spring-cloud-bindings].  A few examples:

- `cassandra`
- `couchbase`
- `db2`
- `elasticsearch`
- `kafka`
- `ldap`
- `mongodb`
- `mysql`
- `neo4j`
- `oracle`
- `postgresql`
- `rabbitmq`
- `redis`
- `sqlserver`
- `vault`

The type of the `Secret` resource should have a special value based on the `type` entry value.  The format is `servicebinding.io/{type}`.  For example, if the type entry is `mysql` the Secret's `type` value should be `servicebinding.io/mysql`.  This recommendation helps to query `Secret` resources of particular type using field-selector. For example:

```bash
kubectl get secrets --field-selector="type=servicebinding.io/type"
```

will give the `Secret` resources of `mysql` type.

Similar to `type` entry, spec also recommends to add a `provider` entry to identify the provider.  The `provider` entry is a further classification of the type.

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

# Well-known Secret Entries

Apart from the special `type`, and `provider` entries in the Secret data, there are few special words, if used must follow certain restrictions for the values.  These are called well-known entries.

| Name | Requirements
| ---- | ------------
| `host` | A DNS-resolvable host name or IP address
| `port` | A valid port number
| `uri` | A valid URI as defined by [RFC3986](https://tools.ietf.org/html/rfc3986)
| `username` | A string-based username credential
| `password` | A string-based password credential
| `certificates` | A collection of PEM-encoded X.509 certificates, representing a certificate chain used in mTLS client authentication
| `private-key` | A PEM-encoded private key used in mTLS client authentication


If there is any entry that doesn’t follow the given requirement, you can choose different names. For example, if there is a URI-like string but not a valid one, as per RFC-3986, use another name (e.g., "custom-uri").

# Considerations for Role-Based Access Control (RBAC)

As a service provider, you can create a `ClusterRole` with the label
`servicebinding.io/controller=true` and the verbs `get`, `list`, and `watch`
listed in the rules.  Here is an example `ClusterRole`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: examlples-service-bindings
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
change those values as per your Provisioned Service.  While your operator is
getting installed, make sure this cluster role is also installed.

# Real World Examples

1. RabbitMQ Operator: https://github.com/rabbitmq/cluster-operator/pull/615
2. Kafka Access Operator: https://github.com/strimzi/kafka-access-operator

[provisioned-service]: https://github.com/servicebinding/spec#provisioned-service
[direct-secret-reference]: https://github.com/servicebinding/spec#direct-secret-reference
[spring-cloud-bindings]: https://github.com/spring-cloud/spring-cloud-bindings
