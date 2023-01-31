# Error handling and logging with Node Express web application framework

Error handling and logging, they go hand in hand in web application development. Whenever we encounter an unexpected situation in a web application, we probably want to log something out too. Also, not to forget users of the web app who would like to see sensible error messages generated for these errors.

Basically we would like to be able to generate custom errors and send them back to users. We would like to log these errors out too, and somehow to be able to connect the errors to the incoming requests.

Is there a simple way to handle this by using Express middleware? Let's find out.

## What is an Express middleware?

In Express, a middleware is a function that has access to the request and response objects, and also to the next middleware function in the web application's request - response cycle. Basically we are able to manipulate request and response objects in a middleware as we like. For example, we could set request date in a middleware to be used later in the cycle, in controllers or in middleware that get executed later in the stack.

`/middlewares/requestTime.js`:

```
const requestTime = (request, response, next) => {
    request.headers["X-Request-Time"] = Date.now();
    next();
});

export default requestTime;
```

When creating APIs, we generally use several middlewares like `express.json` (a built-in middleware) for parsing incoming JSON payloads or a third-party middleware like `cors` to manipulate response headers to enable cross-origin requests.

## Could we use a middleware for logging requests?

Indeed, we could. This is another common example of how we use middleware to add piece of functionality into our web applications.

`/middlewares/requestLogger.js`:

```
const requestLogger = (request, response, next) => {
    // We should have request time in headers now
    const requestTime = request.headers["X-Request-Time"];
    console.log(`[Request]: ${requestTime} ${request.method} ${request.url}`);
    next();
}

export default requestLogger;
```

Now our `app.js` would look something like this:

```
import express from "express";
import cors from "cors";
import router from "./api/index";
import requestLogger from "./middlewares/requestLogger";
import requestTime from "./middlewares/requestDate";

...

// Include request time in request headers
app.use(requestTime);

// Log requests
app.use(requestLogger);

// Define API routes
app.use("/api", router);

export default app;
```

## Logging errors?

So how about logging errors? There is another type of middleware called Error-handling middleware, the only difference being that these functions take four arguments instead of three we saw for the other middleware. The fourth argument would be the actual error object.

So in a similar manner our Error-handling middleware would be something like:

`/middlewares/errorHandler.js`:

```
const errorHandler = (error, request, response, next) => {
    const requestTime = request.headers["requestTime"];
    console.log(`[Error]: ${requestTime} ${error.stack || error.message}`);

    response
        .status(500)
        .send({ message: error.message || "Unknown error" });
}

export default errorHandler;
```

and in `app.js` now:

```
import errorHandler from "./middlewares/errorHandler";

...

// Define API routes
app.use("/api", router);

// Handle errors
app.use(errorHandler)

export default app;
```

That takes care of that. But how our error handler gets triggered then? Just pass your error object to the `next()` function and Express calls your first error handler in the stack.

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

So here we picked up the thrown error in catch clause and passed it to error handler using `next()` function.

## Returning errors to client

To return meaningful errors to client the response status should be set and reflect the generated error. Javascript Error class does not have status field but we can extend the Error class to have one.

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

And the error handler in `/middlewares/errorHandler.js` becomes now:

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

## Split further

Instead of stopping here, we could split logging and returning the error into separate handlers.

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

And `app.js` becomes now:

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

## Final words

We have now used Express middleware first to add request time to request headers where it becomes available further down the web application's request - response cycle. Similarly to request time, we could add request id which could be used to connect events together in different logs, for example.

We have also added middleware for logging requests, logging errors and returning errors back to client. For this we created HttpError class which expanded Javascript Error class to include field for http response status. Here we also could expand the error class further to fit whatever needs we have for the web application in hand (we might want to return validation metadata to the client, for example).

In general, we see that Express middleware is a handy method for adding functionality into the request processing stack.

## Acknowledgements

Sami Ruokamo is a software developer and works at Buutti.

## References
