# Effective Error Handling and Logging in Node Express Web Applications

Error handling and logging, they go hand in hand in web application development. Whenever we encounter an unexpected situation in a web application, we want to log out something. We also want to return some meaningful error messages to the users of the web app and preferably with some ID that can be used to connect the error messages to the error logs we have for the backend.

With the ID, we can easily find the request in the logs and see what happened.

To achieve this, we need to:

1. Generate request ID for each request
2. Log requests with request ID
3. Log errors with request ID
4. Return error message with request ID to the user of the web app

How to implement this in Node Express? Let's find out.

## Enter the Middleware?

In Express, a middleware is a function that has access to the request and response objects, and also to the next middleware function in the web application's request-response cycle. Basically we are able to manipulate request and response objects in a middleware as we like.

For example, we could set request date in a middleware to be used later in the cycle, in controllers or in the middleware that gets executed later in the stack.

`/middlewares/requestTime.js`:

```
const requestTime = (request, response, next) => {
    request.headers["x-request-time"] = Date.now();
    next();
});

export default requestTime;
```

When creating APIs, we typically use several middlewares like `express.json`, a built-in middleware for parsing incoming JSON payloads, or a third-party middleware like `cors` to manipulate response headers to enable cross-origin requests.

## Generating Request ID in Middleware

It appears that the request and response objects would be an appropriate location for transmitting request IDs throughout the pipeline. Let's create a separate middleware for this purpose.

First, create a unique string to be used as the request ID:

```
  const requestId = uuidv4();
```

Add the generated request ID to the request headers:

```
  request.headers["x-request-id"] = requestId;
```

Add the request ID in the response headers too (this way, we return the same ID to the client):

```
  response.set("x-request-id", requestId);
```

For CORS requests, we also need to expose the request ID in the response headers like this:

```
  response.set("Access-Control-Allow-Headers", "x-request-id");
  response.set("Access-Control-Expose-Headers", "x-request-id");
```

Finally, to make the request ID available in functions that are not part of the request-response cycle, we can set it in the `httpcontext` object like this:

```
  httpcontext.set("requestId", requestId);
```

The whole middleware would look something like this:

`/middlewares/requestId.js`:

```
import httpcontext from "express-http-context";
import { v4 as uuidv4 } from "uuid";

const requestId = (request, response, next) => {
  const requestId = uuidv4();

  // Set request id in request headers
  request.headers["x-request-id"] = requestId;

  // Set request id in response headers
  response.set("x-request-id", requestId);

  // Set CORS headers for allowing request id to be sent along response headers
  response.set("Access-Control-Allow-Headers", "x-request-id");
  response.set("Access-Control-Expose-Headers", "x-request-id");

  // Set request id in context
  httpcontext.set("requestId", requestId);

  next();
};

export default requestId;
```

The generated request ID can now be utilized not only in the middleware executed later in the stack, but also in controllers and other functions that are independent of the request-response cycle.

And as we have set request ID in the response headers, we can access it in the client side too. For example, we could log the request ID in the browser console like this:

```
  console.log(`Request ID: ${response.headers["x-request-id"]}`);
```

## Logging Requests

Let's log requests with the request id we have generated. We can create a middleware for this too.

`/middlewares/requestLogger.js`:

```
const requestLogger = (request, response, next) => {
    const requestTime = request.headers["X-Request-Time"];
    const requestId = httpcontext.get("requestId");

    console.log(
      `[Request]: ${requestTime} ${requestId} ${request.method} ${
        request.url
      } ${JSON.stringify(request.params)} ${JSON.stringify(request.body)}`
    );

    next();
}

export default requestLogger;
```

An example log would look like this:

```
[Request]: Wed, 01 Feb 2023 08:27:14 GMT eb2d9ba3-6d9c-4285-be0c-2a667e751853 GET /api/books {} {}
[Request]: Wed, 01 Feb 2023 08:28:09 GMT b0b25684-0fa2-40f6-b37b-4b5f53a26d10 PUT /api/books/65ddc15e-1db4-473e-bda2-7870eb06e370 {} {"id":"65ddc15e-1db4-473e-bda2-7870eb06e370","author":"Robert C. Martin","title":"Clean Code","description":"A Hand Book of Agile Software Craftmanship."}
[Request]: Wed, 01 Feb 2023 08:28:09 GMT d231bb75-2ebf-440d-b593-c82d59edeb71 GET /api/books {} {}
```

In real world applications, we would probably want to use a logging library like `winston` or `pino` to log requests, but for the sake of simplicity, we are using `console.log` here.

## Logging Errors

For error logging, there is a type of middleware called error-handling middleware, which differs from other middleware by taking four arguments instead of the three we saw previously. The fourth argument represents the error object itself.

We can use this middleware to log errors with the request ID we have generated:

`/middlewares/errorHandler.js`:

```
const errorHandler = (error, request, response, next) => {
    const requestTime = request.headers["requestTime"];
    const requestId = httpcontext.get("requestId");
    console.error(`[Error]: ${requestTime} ${requestId} ${error.stack || error.message}`);

    response
        .status(500)
        .send({ message: error.message || "Unknown error" });
}

export default errorHandler;
```

But how our error handler gets triggered in practice? Just pass your error object to the `next()` function and Express calls your first error handler in the stack:

In `/api/books.js`:

```
// GET /api/books/:id - Get book
router.get("/:id", async (request, response, next) => {
  try {
    const book = await books.get(request.params.id);
    if (!book) {
      throw new Error("Book not found");
    }

    response.send(book);
  } catch (error) {
    next(error);
  }
});
```

If we try to get a book that does not exist, next function is called with an error object. This triggers the error handler middleware and logs the error.

## Returning Errors to the Client

To return meaningful errors to client the response status should be set and reflect the generated error. Javascript Error class does not have status field but we can extend the Error class to have one:

In `/utils/types.js`:

```
export class HttpError extends Error {
    constructor(message = "Unknown error", status = 500) {
        super(message);
        this.name = "HttpError";
        this.status = status;
    }
}
```

With this we can write our endpoint using the new HttpError class.

In `/api/books.js`:

```
// GET /api/books/:id - Get book
router.get("/:id", async (request, response, next) => {
  try {
    const book = await books.get(request.params.id);
    if (!book) {
      throw new HttpError("Book not found", 404);
    }

    response.send(book);
  } catch (error) {
    next(error);
  }
});
```

The error handler in `/middlewares/errorHandler.js` becomes now:

```
const errorHandler = (error, request, response, next) => {
    const requestTime = request.headers["requestTime"];
    console.log(`[Error]: ${requestTime} ${error.status} ${error.stack || error.message}`);

    response
        .status(error.status)
        .send({ message: error.message });
}

export default errorHandler;
```

We log the error in our handler and send the error with status back to the client now.

## Split Further

In actual implementation, it is common to log the error in one middleware and handle the error response in another. This approach allows us to reuse the same middleware for various purposes.

We can split logging and returning the error into separate middlewares like this:

In `/middlewares/errorHandler.js`:

```
const logError = (error, request, response, next) => {
    const requestTime = request.headers["requestTime"];
    console.log(`[Error]: ${requestTime} ${error.status} ${error.stack || error.message}`);

    // Remember to pass the error for further processing
    next(error);
}

const returnError = (error, request, response, next) => {
    response
        .status(error.status)
        .send({ message: error.message || "Unknown error" });
}

export {
    logError,
    returnError,
}

```

And `app.js` with the whole middleware stack becomes now:

```
import express from "express";
import cors from "cors";
import router from "./api/index";
import requestLogger from "./middlewares/requestLogger";
import requestTime from "./middlewares/requestDate";
import { logError, returnError } from "./middlewares/errorHandler";

const port = 3001;
const app = express();

// Enable CORS
app.use(cors());

// Handle CORS preflight requests
app.options("*", cors());

// Parse incoming JSON into request body
app.use(express.json());

// Include request id in request headers
app.use(requestId);

// Include request time in request headers
app.use(requestTime);

// Log requests
app.use(requestLogger);

// Define API routes
app.use("/api", router);

// Log errors
app.use(logError);

// Return errors to client
app.use(returnError);

export default app;
```

Finally, an example how the user could see the error with the request ID that can be used to connect events together in different logs:

![Book not found](./images/book-not-found.png)

Again, in practice, we probably would like to show the request ID only for errors that are not 4xx errors (client errors).

## Conclusion

In this article, we have demonstrated how to utilize Express middleware to add functionality to the request-response cycle in a web application. First, we added the request time to the request headers, making it available throughout the pipeline. We then added the request ID, which can be used to connect events across different logs.

Additionally, we implemented middleware for logging requests, logging errors, and returning errors to the client. To achieve this, we created the HttpError class, which extends the built-in Error class in JavaScript and includes a field for the HTTP response status code. This can be further extended to accommodate specific needs for the web application, such as including validation metadata in the error response.

Overall, Express middleware provides a convenient and flexible means for adding functionality to the request processing stack.

## Author Information

Sami Ruokamo is a software developer and works at Buutti. He is interested in software development, especially in web development. He has been working with web application development for over 10 years. He is also interested in DevOps and cloud technologies.

To see the ideas presented in this article in action, check out https://github.com/samiru/node-express-react-template.

## References

[1] https://expressjs.com/en/guide/error-handling.html (Error handling)
[2] https://http.dev/x-request-id (X-Request-ID)
[3] https://javascript.info/custom-errors (Custom errors, extending Error)
[4] https://sematext.com/blog/node-js-error-handling/#what-is-error-handling-in-node-js (Node.js Error Handling Made Easy: Best Practices On Just About Everything You Need to Know)
