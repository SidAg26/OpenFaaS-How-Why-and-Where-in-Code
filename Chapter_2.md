# Chapter 2: Configuration

Welcome back! In [Chapter 1: Gateway Application](./Chapter_1.md), we learned that the OpenFaaS Gateway acts as the central entry point, receiving requests and routing them to the correct functions. But how does the Gateway know *where* to send requests? How does it know important details like how long to wait for a function's response or where to find the Prometheus monitoring server?

This is where **Configuration** comes in.

## What Problem Does Configuration Solve?

Think of the Gateway application as a highly skilled employee starting a new job. Even the best employee needs instructions:
*   Who should I report to for finding functions? (The Provider)
*   How long should I wait before considering a task failed? (Timeouts)
*   Are there special rules for security (like basic authentication)?
*   Where do I send reports (metrics)?

Configuration provides these crucial instructions to the Gateway when it starts up. Without configuration, the Gateway wouldn't know how to interact with other parts of the OpenFaaS system or how to behave according to the deployment's needs.

**Central Use Case:** Our goal from Chapter 1 was to call a function like `hello-world`. For the Gateway to successfully forward your request to `hello-world`, it *must* know where the **Provider** is located, because the Provider is responsible for managing the function instances. Configuration tells the Gateway the Provider's address.

## What is Configuration?

In the context of the OpenFaaS Gateway, configuration is the process of providing operational settings to the Gateway application, typically at startup.

The most common way to configure cloud-native applications like the OpenFaaS Gateway is through **Environment Variables**.

**Why Environment Variables?**

Using environment variables for configuration offers several advantages:

*   **Portability:** The application code doesn't need to change for different environments (development, staging, production). Only the environment variables set *outside* the application change.
*   **Simplicity:** It's a straightforward way to pass values into a running process.
*   **Integration with Orchestrators:** Systems like Docker, Kubernetes, and others easily allow you to set environment variables for the containers they manage.

The Gateway application is designed to read specific environment variables to get its operational parameters.

## How Configuration Solves Our Use Case

To call the `hello-world` function via the Gateway, the Gateway needs to talk to the OpenFaaS **Provider**. The address of this Provider is one of the key pieces of information provided via configuration.

Let's say your OpenFaaS Provider is running at `http://faas-swarm:8081` (this is a common address if you're running on Docker Swarm). You would tell the Gateway this by setting an environment variable:

```bash
export functions_provider_url=http://faas-swarm:8081
```

When the Gateway starts, it reads this variable. Now it knows where to find the Provider and can ask it questions like "Where is `hello-world`?" or "Can you invoke `hello-world` for me?".

Other important configuration settings include:

| Setting Name             | Environment Variable (Example) | What it Configures                                    | Importance for Use Case? |
| :----------------------- | :----------------------------- | :---------------------------------------------------- | :----------------------- |
| **Provider URL**         | `functions_provider_url`       | Where to find the component that manages functions.   | **Crucial**              |
| Read Timeout             | `read_timeout`                 | Max time to wait for an incoming client request.      | Less Direct              |
| Write Timeout            | `write_timeout`                | Max time to wait for a function's response.           | **Important**            |
| Upstream Timeout         | `upstream_timeout`             | Max time for the Gateway's *own* call to the function/provider. | **Important**            |
| Prometheus Host/Port     | `faas_prometheus_host`, `faas_prometheus_port` | Where to send metrics.                                | Indirect                 |
| Basic Authentication     | `basic_auth`                   | Enable/disable security requiring username/password.  | Depends on deployment    |
| NATS Settings            | `faas_nats_address`, etc.      | Details for asynchronous function calls.              | Only for Async Mode      |

For our simple `hello-world` use case via synchronous HTTP, the `functions_provider_url`, `write_timeout`, and `upstream_timeout` are particularly relevant because they govern how the Gateway interacts with and waits for the function via the Provider.

## Gateway's Internal Implementation (Simplified)

How does the Gateway read these environment variables? Let's look back at the Go code we saw in Chapter 1, specifically the `main.go` file.

### Reading Configuration

The very first thing the `main` function does is read the configuration:

```go
func main() {
	// Read configuration settings for the gateway
	osEnv := types.OsEnv{}
	readConfig := types.ReadConfig{}
	config, configErr := readConfig.Read(osEnv) // <-- This is where config is read

	if configErr != nil {
		log.Fatalln(configErr) // Stop if configuration fails
	}

	// ... rest of the initialization and server start ...
}
```

**Explanation:**

*   `types.ReadConfig{}` creates an object that has the logic to read configuration.
*   `types.OsEnv{}` is a helper used to read environment variables (it wraps Go's standard library function `os.Getenv`).
*   `readConfig.Read(osEnv)` calls the `Read` method, passing the helper for accessing environment variables. This is the core function that looks for specific environment variables.
*   The result is a `config` object (of type `*types.GatewayConfig`) containing all the settings, or an error (`configErr`) if something went wrong (like an invalid URL format).

Now, let's peek inside the `Read` function itself, found in `gateway/types/readconfig.go`:

```go
// Read fetches gateway server configuration from environmental variables
func (ReadConfig) Read(hasEnv HasEnv) (*GatewayConfig, error) {
	cfg := GatewayConfig{
		PrometheusHost: "prometheus", // Set some default values first
		PrometheusPort: 9090,
	}

	defaultDuration := time.Second * 60

	// Read and parse timeouts
	cfg.ReadTimeout = parseIntOrDurationValue(hasEnv.Getenv("read_timeout"), defaultDuration)
	cfg.WriteTimeout = parseIntOrDurationValue(hasEnv.Getenv("write_timeout"), defaultDuration)
	cfg.UpstreamTimeout = parseIntOrDurationValue(hasEnv.Getenv("upstream_timeout"), defaultDuration)

	// Read the crucial functions provider URL
	if len(hasEnv.Getenv("functions_provider_url")) > 0 {
		var err error
		// os.Getenv reads the variable value
		// url.Parse tries to turn the string into a valid URL
		cfg.FunctionsProviderURL, err = url.Parse(hasEnv.Getenv("functions_provider_url"))
		if err != nil {
			return nil, fmt.Errorf("if functions_provider_url is provided, then it should be a valid URL, error: %s", err)
		}
	}

    // ... reading other configuration variables like NATS, Prometheus, etc. ...

	// Store settings like Basic Auth status
	cfg.UseBasicAuth = parseBoolValue(hasEnv.Getenv("basic_auth"))

	// Return the struct holding all the configuration
	return &cfg, nil
}
```

**Explanation:**

*   The function initializes a `GatewayConfig` struct (`cfg`) with some default values (like the default Prometheus address).
*   It then uses `hasEnv.Getenv("VARIABLE_NAME")` repeatedly to check if specific environment variables are set and read their string values.
*   Helper functions like `parseIntOrDurationValue` and `parseBoolValue` convert the string values read from environment variables into the correct data types (like `time.Duration` or `bool`).
*   For the `functions_provider_url`, it specifically uses `url.Parse` to ensure the value provided is a valid URL. If not, it returns an error, causing the Gateway to fail startup (as seen in `main.go`).
*   All the successfully read and parsed settings are stored in the `cfg` struct.
*   Finally, the function returns a pointer to this `cfg` struct, which holds the complete configuration for the Gateway instance.

### Using Configuration

Once the configuration is read and stored in the `config` object (of type `*types.GatewayConfig`), the rest of the Gateway application uses these values to set up its behavior.

For example:

*   In `main.go`, when setting up the HTTP server, it uses `config.ReadTimeout` and `config.WriteTimeout` to configure the server's timeouts.
*   The `functionProxy` handler (which we saw in Chapter 1 and will explore in [Request Handling (Sync/Async)](./Chapter_5.md)) uses `config.UpstreamTimeout` when making the call to the function.
*   Crucially, handlers that need to communicate with the **Provider** (like the `functionProxy` or handlers for listing functions) will use `config.FunctionsProviderURL` to know *where* to send their requests to the Provider. This interaction is detailed further in [Provider Interaction](./Chapter_7.md).

By reading configuration upfront, the Gateway's behavior is customized to the specific environment it's running in, without needing code changes.

## Conclusion

Configuration is the essential step of providing the Gateway application with the necessary instructions to operate within its environment. By reading settings primarily from environment variables, the Gateway learns crucial details like where to find the **Provider**, how long to wait for responses, and whether security features are enabled. This allows the Gateway to correctly route and manage requests, enabling core functions like calling `hello-world` by knowing where to interact with the component that controls it (the Provider).

Now that the Gateway is configured and ready, let's look at how it understands the *structure* of incoming requests and how it maps different URL paths to specific actions.

[Chapter 3: API Definition](./Chapter_3.md)

---

<sub><sup>Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge).</sup></sub> <sub><sup>**References**: [[1]](https://github.com/openfaas/faas/blob/7803ea1861f2a22adcbcfa8c79ed539bc6506d5b/gateway/types/readconfig.go)</sup></sub>
