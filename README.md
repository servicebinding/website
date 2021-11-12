---
title: Service Binding for Kubernetes
permalink: /
---

Today in Kubernetes, the exposure of secrets for connecting application workloads to external services such as REST APIs, databases, event buses, and many more is manual and bespoke.  Each service provider suggests a different way to access their secrets, and each application developer consumes those secrets in a custom way to their workloads.  While there is a good deal of value to this flexibility level, large development teams lose overall velocity dealing with each unique solution.  To combat this, we already see teams adopting internal patterns for how to achieve this workload-to-service linkage.

This project specifies a Kubernetes-wide specification for communicating service secrets to workloads in an automated way.  It aims to create a widely applicable mechanism but _without_ excluding other strategies for systems that it does not fit easily.  The benefit of Kubernetes-wide specification is that all of the actors in an ecosystem can work towards a clearly defined abstraction at the edge of their expertise and depend on other parties to complete the chain.

* Application Developers expect their secrets to be exposed consistently and predictably.
* Service Providers expect their secrets to be collected and exposed to users consistently and predictably.
* Platforms expect to retrieve secrets from Service Providers and expose them to Application Developers consistently and predictably.

<a name="community"></a>
# Community, discussion, contribution, and support

The Service Binding for Kubernetes project is a community lead effort.
A bi-weekly [working group call][working-group] is open to the public.
Discussions occur [on GitHub][github] and on the [#bindings-discuss channel in the Kubernetes Slack][slack].

## Code of conduct

Participation in the Kubernetes community is governed by the [Kubernetes Code of Conduct][code-of-conduct].

[working-group]: https://docs.google.com/document/d/1rR0qLpsjU38nRXxeich7F5QUy73RHJ90hnZiFIQ-JJ8/edit#heading=h.ar8ibc31ux6f
[slack]: https://kubernetes.slack.com/archives/C012F2GPMTQ
[github]: https://github.com/servicebinding
[code-of-conduct]: https://git.k8s.io/community/code-of-conduct.md

# Consuming the Bindings from Workloads
The [Workload Projection section](/spec/core/1.0.0-rc3/#workload-projection) of the specification describes how bindings are projected into the workload.  The primary mechanism of projection is through files mounted at a specific directory.  The bindings directory path is discovered through the mandatory `$SERVICE_BINDING_ROOT` environment variable set on all containers where bindings are created.

Within this service binding root directory, multiple Service Bindings may be projected.  For example, a workload that requires both a database and event stream will declare one `ServiceBinding` for the database, a second `ServiceBinding` for the event stream, and both bindings will be projected as subdirectories of the root.

Let's take a look at the example given in the spec:

```
$SERVICE_BINDING_ROOT
├── account-database
│   ├── type
│   ├── provider
│   ├── uri
│   ├── username
│   └── password
└── transaction-event-stream
    ├── type
    ├── connection-count
    ├── uri
    ├── certificates
    └── private-key
```
In the above example, there are two bindings under the `$SERVICE_BINDING_ROOT` directory with the names `account-database` and `transaction-event-stream`.  In order for a workload to configure itself, it must select the proper binding for each client type.  Each binding directory has a special file named `type` and you can use the content of that file to identify the type of the binding projected into that directory (e.g. `mysql`, `kafka`).  Some bindings optionally also contain another special file named `provider` which is an additional identifier used to further narrow down ambiguous types.  Choosing a binding "by-name" is not considered good practice as it makes your workload less portable (although it may be unavoidable).  Wherever possible use the `type` field and, if necessary, `provider` to select a binding.

Usually, operators use `ServiceBinding` resource name (`.metadata.name`) as the bindings directory name, but the spec also provides a way to override that name through the `.spec.name` field. So, there is a chance for bindings name collision.  However, due to the nature of the volume mount in Kubernetes, the bindings directory will contain values from only one of the Secret resources.  This is a caveat of using the bindings directory name to look up the bindings.

### Purpose of the type and the provider fields in the workload projection

The specification mandates the `type` field and recommends `provider` field in the projected binding.  In many cases the `type` field should be good enough to select the appropriate binding.  In cases where it is not (e.g. when there are different providers for the same Provisioned Service), the `provider` field may be used.  For example, when the type is `mysql`, the `provider` value might be `mariadb`, `oracle`, `bitnami`, `aws-rds`, etc.  When the workload is selecting a binding, if necessary, it could consider `type` and `provider` as a composite key to avoid ambiguity.  This could be helpful if a workload needs to choose a particular provider based on the deployment environment.  In the deployment environment (`stage`, `prod`, etc.), at any given time, you need to ensure only one binding projection exists for a given `type` or `type` and `provider` -- unless your workload needs to connect to all the services.

### Environment Variables

The specification also has support for projecting binding values as environment variables.  You can use the built-in language feature of your programming language of choice to read environment variables.  The container must restart to read updated environment variable values.

# Programmatic Language Bindings
While the projection of a binding into a `Pod` can be consumed directly with features typically found in any programming language, it is often preferable to use a language binding that adds semantic meaning to the interaction.  For example:

### Java
_(Example taken from <https://github.com/nebhale/client-jvm>)_

```java
import com.nebhale.bindings.Binding;
import com.nebhale.bindings.Bindings;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class Application {

    public static void main(String[] args) {
        Binding[] bindings = Bindings.fromServiceBindingRoot();
        bindings = Bindings.filter(bindings, "postgresql");
        if (bindings.length != 1) {
            System.err.printf("Incorrect number of PostgreSQL drivers: %d\n", bindings.length);
            System.exit(1);
        }

        String url = bindings[0].get("url");
        if (url == null) {
            System.err.println("No URL in binding");
            System.exit(1);
        }

        Connection conn;
        try {
            conn = DriverManager.getConnection(url);
        } catch (SQLException e) {
            System.err.printf("Unable to connect to database: %s", e);
            System.exit(1);
        }

        // ...
    }

}
```

### Go
_(Example taken from <https://github.com/baijum/servicebinding>)_

```go
import (
	"context"
	"fmt"
	"github.com/jackc/pgx/v4"
	"github.com/baijum/servicebinding/binding"
	"os"
)

func main() {
	sb, err := bindings.FromServiceBindingRoot()
	if err != nil {
		_, _ = fmt.Fprintln(os.Stderr, "Could not read service bindings")
		os.Exit(1)
	}

	b, err := sb.Bindings("postgresql")
	if err != nil {
		_, _ = fmt.Fprintln(os.Stderr, "Unable to find postgresql binding")
		os.Exit(1)
	}
	if len(b) != 1 {
		_, _ = fmt.Fprintf(os.Stderr, "Incorrect number of PostgreSQL bindings: %d\n", len(b))
		os.Exit(1)
	}

	u, ok := b[0]["url"]
	if !ok {
		_, _ = fmt.Fprintln(os.Stderr, "No URL in binding")
		os.Exit(1)
	}

	conn, err := pgx.Connect(context.Background(), u)
	if err != nil {
		_, _ = fmt.Fprintf(os.Stderr, "Unable to connect to database: %v\n", err)
		os.Exit(1)
	}
	defer conn.Close(context.Background())

	// ...
}
```

## Known Language Bindings
_(If you've created an implementation, please [submit a PR][p] for inclusion)_

[p]: https://github.com/servicebinding/website/pulls

* .NET
	* [`donschenck/dotnetservicebinding`](https://github.com/donschenck/dotnetservicebinding)
* Go
	* [`baijum/servicebinding`](https://github.com/baijum/servicebinding)
	* [`nebhale/client-go`](https://github.com/nebhale/client-go)
* JVM
	* [`nebhale/client-jvm`](https://github.com/nebhale/client-jvm)
	* [Quarkus](https://quarkus.io/guides/deploying-to-kubernetes#service-binding)
	* [Spring Cloud Bindings](https://github.com/spring-cloud/spring-cloud-bindings)
* NodeJS
	* [`nebhale/client-nodejs`](https://github.com/nebhale/client-nodejs)
	* [`nodeshift/kube-service-bindings`](https://github.com/nodeshift/kube-service-bindings)
* Python
	* [`baijum/pyservicebinding`](https://github.com/baijum/pyservicebinding)
	* [`nebhale/client-python`](https://github.com/nebhale/client-python)
* Ruby
	* [`nebhale/client-ruby`](https://github.com/nebhale/client-ruby)
* Rust
	* [`nebhale/client-rust`](https://github.com/nebhale/client-rust)

# Specification
* Core
  * [1.0.0-rc3](/spec/core/1.0.0-rc3/) (pre-release)
  * [1.0.0-rc2](/spec/core/1.0.0-rc2/) (pre-release)
  * [1.0.0-rc1](/spec/core/1.0.0-rc1/) (pre-release)
