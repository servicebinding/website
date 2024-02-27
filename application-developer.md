---
description: Service Bindings for Application Developers
permalink: /application-developer/
---

Application developers consume service bindings by reading [volume mounted `Secret`s][vm] from a directory.  The specification's [Workload Projection section][wp] 
describes this in detail but there are three important parts:

[vm]: https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-files-from-a-pod
[wp]: {{ site.spec.core }}#workload-projection

1. A `SERVICE_BINDING_ROOT` environment variable that specifies the directory
2. A subdirectory for each  binding with a name matching the binding name
3. A file for each `Secret` entry with a name matching the entry key and content matching the entry value

Let's take a look at the [example][eds] given in the spec:

[eds]: {{ site.spec.core }}#example-directory-structure

```plain
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

In the example, there are two bindings under the `$SERVICE_BINDING_ROOT` directory with the names `account-database` and `transaction-event-stream`.  In order for a 
workload to configure itself, it must select the proper binding for each client type.  Each binding directory has a special file named `type` that is used to identify 
the type of the binding projected into that directory (e.g. `mysql`, `kafka`).

In most cases, the `type` entry should be enough to select the appropriate binding.  In the cases where it is not (e.g. when there are different providers for the same 
service type), the optional but strongly encouraged `provider` entry should be used to further differentiate bindings.  For example, when the `type` is `mysql`, 
`provider` values of `mariadb`, `oracle`, `bitnami`, or `aws-rds` can be used to choose a binding.

{% include tip.html content="Wherever possible select a binding using the `type` entry and, if necessary, the `provider` entry.  Consider using the name of the binding 
only as a last resort." %}

# Language-specific Libraries
A binding can almost always be consumed directly with features found in any programming language.  However, it's often preferable to use a language-specific library that 
adds semantic meaning to the code.  There's no "right" way to interact with a binding, so here's a partial list of libraries you might use:

* .NET
	* [`donschenck/dotnetservicebinding`](https://github.com/donschenck/dotnetservicebinding)
	* [Steeltoe](https://docs.steeltoe.io/api/v3/connectors/)
* Go
	* [`baijum/servicebinding`](https://github.com/baijum/servicebinding)
	* [`nebhale/client-go`](https://github.com/nebhale/client-go)
	* [`rhjensen79/files2env`](https://github.com/rhjensen79/files2env)
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

{% include note.html content="If you've created an implementation, please [submit a PR](https://github.com/servicebinding/website/pulls) for inclusion." %}

To illustrate the advantages of of using a library, here are a few examples:

### Java
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
_(Example taken from <https://github.com/nebhale/client-jvm>)_


### Go
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
_(Example taken from <https://github.com/baijum/servicebinding>)_
