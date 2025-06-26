# Chapter 3: API Definition

Welcome back! In [Chapter 1: Gateway Application](./Chapter_1.md), we learned that the Gateway is the central point for all interactions with OpenFaaS. In [Chapter 2: Configuration](./Chapter_2.md), we saw how the Gateway gets its settings, like where to find the Provider.

Now, let's talk about how you, or other systems, actually *communicate* with the Gateway. How does the Gateway understand your request? Do you just type random things at it? No! The Gateway has a very specific set of commands and addresses it understands. This structured way of communicating is called the **API Definition**.

## What Problem Does API Definition Solve?

Imagine walking up to the Gateway (our helpful receptionist) and saying, "Do something with `hello-world`!". The receptionist would look confused. Do you want to *run* `hello-world`? Do you want to know its status? Do you want to delete it?

The Gateway needs a clear, standardized way to understand your intent. This is exactly what the **API Definition** provides.

It's like the receptionist having a detailed manual or a menu that says:
*   "If someone says 'Run function' and points to the `/function/` door followed by a name, send their request there."
*   "If someone says 'Tell me about functions' and asks at the `/system/functions` counter with a 'GET' form, give them the list."

**Central Use Case:** Our goal is still to call the `hello-world` function. The API Definition tells us exactly *how* to do this using a standard web request (HTTP). It defines the specific URL path and the HTTP method you need to use to trigger a synchronous function invocation.

## What is API Definition?

The **API Definition** is the formal contract that describes all the ways you can interact with the OpenFaaS Gateway using HTTP requests. It's like the Gateway's public instruction manual.

It specifies:

1.  **Endpoints (Paths):** The specific URL paths you can send requests to (e.g., `/function/hello-world`, `/system/functions`).
2.  **HTTP Methods:** Which action you want to perform on that path (e.g., `GET` to read info, `POST` to create/send data, `DELETE` to remove something, `PUT` to update something).
3.  **Parameters:** Any extra information needed in the URL itself (like the function name in `/function/{name}`).
4.  **Request Body:** If you're sending data (like the function's input or a function deployment spec), what format it should be in.
5.  **Responses:** What kinds of responses you can expect back (e.g., status codes like 200 OK, 404 Not Found, and the format of the response data).

OpenFaaS uses a standard format called **OpenAPI** (also known as Swagger) to define its API. This is a widely recognized way to describe RESTful APIs. The OpenFaaS API definition is stored in a file, commonly named `spec.openapi.yml`.

## Key API Endpoints

The OpenFaaS Gateway API definition covers several categories of interactions:

| Category              | Purpose                                                    | Example Endpoints                                  | HTTP Methods (Common) |
| :-------------------- | :--------------------------------------------------------- | :------------------------------------------------- | :-------------------- |
| **Function Invocation** | Running your actual functions                              | `/function/{name}`, `/async-function/{name}`       | POST                  |
| **Function Management** | Deploying, updating, listing, deleting, scaling functions | `/system/functions`, `/system/scale-function/{name}` | GET, POST, PUT, DELETE |
| **System Information**  | Getting Gateway/Provider status, logs, secrets, namespaces | `/system/info`, `/system/logs`, `/system/secrets`  | GET, POST, PUT, DELETE |
| **Internal**          | Health checks, metrics (usually for monitoring systems)    | `/healthz`, `/metrics`, `/system/alert`            | GET, POST             |

For a beginner interacting with OpenFaaS, the **Function Invocation** and some **Function Management** endpoints are the most common.

## How API Definition Solves Our Use Case

To call the `hello-world` function, we need to know the correct API endpoint and method. Looking at the API Definition (or the OpenFaaS documentation which is generated *from* the definition), you'll find something like this:

*   **Endpoint:** `/function/{functionName}`
*   **Method:** `POST`
*   **Description:** Synchronously invoke a function. `{functionName}` is a part of the URL path that tells the Gateway *which* function to invoke.

This definition tells you exactly what to do: send a `POST` request to the Gateway's address, with the path `/function/hello-world`.

The Gateway, receiving this request, looks at the path (`/function/hello-world`) and the method (`POST`). It consults its internal setup (which implements the API definition) and knows, "Ah, this user wants to synchronously invoke the function named `hello-world`."

## Exploring the OpenFaaS API Definition

The formal API definition for the OpenFaaS Gateway is in a file called `spec.openapi.yml`. You can view this file directly, or use tools like the Swagger Editor to see it in a user-friendly format.

Let's look at a tiny snippet from the `spec.openapi.yml` file (the real file is much longer, describing *all* the endpoints):

```yaml
paths:
  "/function/{functionName}": # This is the endpoint path pattern
    post: # This defines the action for the HTTP POST method on this path
      operationId: InvokeFunction # An internal identifier for this operation
      description: Invoke a function in the default OpenFaaS namespace.
      summary: Synchronously invoke a function...
      tags:
        - function # This groups the endpoint under the "function" category
      parameters: # Defines pieces of information extracted from the request
      - name: functionName # The name of the parameter
        in: path # Where to find the parameter (in the URL path)
        description: Function name # What it represents
        required: true # Is this part of the path mandatory? Yes.
        schema:
          type: string # What type of data is it? A string.
      requestBody: # Describes the data sent with the request (the function input)
        description: "(Optional) data to pass to function"
        content:
          "*/*": # Can accept any content type
            schema:
              type: string
              format: binary # Expects arbitrary binary data
              example: '{"hello": "world"}'
        required: false # The request body is optional
      responses: # Describes the possible responses from the Gateway
        '200':
          description: Value returned from function # What a 200 OK response means
        '404':
          description: Not Found # What a 404 response means (function not found)
        '500':
          description: Internal server error # What a 500 response means
```

**Explanation:**

*   This YAML snippet defines *one* specific interaction: sending a `POST` request to a path starting with `/function/` followed by something that will be captured as `functionName`.
*   It clearly states the purpose (`Synchronously invoke a function`).
*   It explains that the `functionName` is a required part of the path.
*   It says you *can* send data in the request body (like JSON or text) and that the Gateway will pass it to the function.
*   It tells you that a successful response will likely have a `200` status code, and you might get `404` or `500` for errors.

This is the detailed instruction manual entry for synchronously invoking a function.

You can see the full API definition yourself:
1. Go to the [Swagger editor](http://editor.swagger.io/) in your web browser.
2. Click `File` -> `Import URL`.
3. Paste `https://raw.githubusercontent.com/openfaas/faas/master/api-docs/spec.openapi.yml` and click `OK`.

This will show you the complete, interactive documentation for all the Gateway's endpoints, generated directly from the `spec.openapi.yml` file.

## Gateway's Internal Implementation (Simplified)

How does the Gateway application *use* this API Definition? The Go code within the Gateway doesn't *read* the `spec.openapi.yml` file at runtime (that file is mainly for documentation and code generation). Instead, the code is written to **implement** the endpoints defined in that specification.

We saw this in [Chapter 1: Gateway Application](./Chapter_1.md) when looking at the `gorilla/mux` router setup in `main.go`. The lines of code that define the routes *are* the implementation of the API Definition within the Gateway application.

Let's look at those snippets again, now with the context of the API Definition:

```go
// main.go (simplified)
func main() {
	// ... config reading and initialization ...

	r := mux.NewRouter() // Create the router

	// Get the handler that knows how to proxy requests to functions
	var faasHandlers types.HandlerSet // Holds various handler functions
	// ... initialization of faasHandlers ...
	functionProxy := faasHandlers.Proxy // The handler for invoking functions

	// ... setup for async handling ...

	// Define routes that IMPLEMENT the API Definition for function invocation
	// These paths match the patterns like "/function/{functionName}" from the spec
	r.HandleFunc("/function/{name:["+NameExpression+"]+}", functionProxy) // Handles /function/my-func
	r.HandleFunc("/function/{name:["+NameExpression+"]+}/", functionProxy) // Handles /function/my-func/ (trailing slash)
	r.HandleFunc("/function/{name:["+NameExpression+"]+}/{params:.*}", functionProxy) // Handles /function/my-func/extra/path?q=1

	// Define routes that IMPLEMENT the API Definition for system management
	// These paths match patterns like "/system/functions" from the spec
	r.HandleFunc("/system/functions", faasHandlers.ListFunctions).Methods(http.MethodGet) // GET /system/functions -> ListFunctions handler
	r.HandleFunc("/system/functions", faasHandlers.DeployFunction).Methods(http.MethodPost) // POST /system/functions -> DeployFunction handler
	r.HandleFunc("/system/functions", faasHandlers.DeleteFunction).Methods(http.MethodDelete) // DELETE /system/functions -> DeleteFunction handler
	// ... many other system routes ...

	// ... start the HTTP server using the router ...
}
```

**Explanation:**

*   The `r.HandleFunc(...)` lines register specific URL path patterns (`/function/{name...}`, `/system/functions`) with specific pieces of code (`functionProxy`, `ListFunctions`, `DeployFunction`, etc.).
*   These path patterns and the associated HTTP methods (`.Methods(...)`) directly correspond to the endpoints and methods described in the `spec.openapi.yml` file.
*   When an incoming request arrives at the Gateway, the `mux.NewRouter()` (assigned to `r`) looks at the request's path and method and finds the *matching* `HandleFunc` definition.
*   It then calls the **handler function** (like `functionProxy` or `ListFunctions`) that was registered for that specific path and method.

So, the `spec.openapi.yml` file tells us *what* the API looks like, and the `gorilla/mux` router setup in the Gateway's code is *how* that API is brought to life, directing incoming requests to the correct internal logic.

## Conclusion

The API Definition is the essential contract that specifies how users and clients interact with the OpenFaaS Gateway. It lists all the available endpoints (paths), the actions you can perform on them (HTTP methods), and what data to send and receive. By defining endpoints like `/function/{name}` for invocation and `/system/functions` for management, the API provides a clear, standardized way to use OpenFaaS. The Gateway's internal code uses a router to implement this definition, ensuring that requests to specific paths are correctly directed to the right internal logic.

Now that we understand the Gateway's API contract, let's look at what happens to a request *immediately* after it arrives but *before* it gets to the specific handler function. This involves **Request Middleware**.

[Chapter 4: Request Middleware](./Chapter_4.md)

---

<sub><sup>Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge).</sup></sub> <sub><sup>**References**: [[1]](https://github.com/openfaas/faas/blob/7803ea1861f2a22adcbcfa8c79ed539bc6506d5b/api-docs/README.md), [[2]](https://github.com/openfaas/faas/blob/7803ea1861f2a22adcbcfa8c79ed539bc6506d5b/api-docs/spec.openapi.yml)</sup></sub>
