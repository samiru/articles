# Improving Web Application Error Diagnosis with Request IDs

Error handling and logging, they go hand in hand in web application development. Whenever we encounter unexpected situations in web applications, we want to log out something. We also want to return meaningful error messages to the users of the web application. Depending on the error we also might want to show users some ID that could be used to connect the error message to the error logs we have for the backend.

Additionally, if our APIs call microservices, it may be beneficial to pass along the request ID for logging purposes within the microservice. This way, we can easily trace back to the original request through the logs and understand what occurred with the help of the request ID.

Let's spend some time to understand how we can improve error diagnosis with request IDs.

## What happens when a browser makes a request to a web application?

Let's start with the basics. What happens when a browser makes a request to a web application? The web application receives the request, processes it, and returns a response to be displayed in the browser.

But how this exactly works, then?

When a browser makes a request to a web application, the request is sent to the web server as a HTTP request. A HTTP request contains information about the request, such as the request method, the URL, and the request headers. By using the request method and the URL, the web server can determine a web application to process the request.

When the web application has processed the request, it returns a HTTP response to the web server which then passes the response back to the browser. The response contains information about the response, such as the response status code, the response headers, and the response body. The response body contains the actual data that gets returned to the browser.

Both the request and the response contain headers. The request headers contain information about the browser, the operating system, and the user agent, while the response headers contain information about the web application, such as the server name, the server version, and the server type.

Thus, these headers are used to pass information between the browser and the web application. They can also contain arbitrary information about the request and the response.

We could pass a request ID in these headers too.

## Request ID to the Rescue

What is a Request ID?

A request ID is a unique identifier for each request. It is a string that is generated for each request and is passed along with the request and the response. The request ID is used to identify the request in the logs and to connect the request to the response.

A natural place to add the request ID to the headers is whenever our web application receives a request. This way, the request ID is available for the whole request-response cycle.

Could we do this automatically? Well, it depends on the web server and the platform you are using. For example, in Heroku, the request ID is automatically added to the request headers.

But even without support for request IDs in the platform, this is easy to implement by ourselves. The process is the same for all frameworks and platforms. We just need to generate a unique string for each request and add it to the request headers. If one likes to return the request ID to the user of the web application, it can be added to the response headers too.

## Example: How to Generate Request IDs in Node Express?

Let's see how we could implement request ID generation in Node Express.

In principle, the process would be the same in other web frameworks too, just generate a unique string for each request and add it to the request headers. If one likes to return the request ID to the user of the web application, it can be added to the response headers too.

In Express, we can use a middleware to generate a request ID for each request. To put simply, a middleware is a function that has access to the `request` and `response` objects, and also to the `next` middleware function in the web application's request-response cycle. Basically we are able to manipulate request and response objects in a middleware as we like.

A middleware that generates a request ID for each request could look like this:

`/middlewares/requestId.js`:

```javascript
import { v4 as uuidv4 } from "uuid";

const requestId = (request, response, next) => {
  const requestId = uuidv4();

  request.headers["x-request-id"] = requestId;
  response.set("x-request-id", requestId);

  next();
};

export default requestId;
```

The middleware uses the `uuid` package to generate a unique request ID. The request ID is then added to the request headers and the response headers. And finally, middleware calls the `next` middleware function in the request-response cycle.

Now the request ID is available later in the request-response cycle, for example in the error handling or logging middleware.

## How to Log Requests and Errors with Request IDs?

Typically, request logs and error logs are written to different files or locations. With the request ID, we can easily connect the request to the error log, and follow the request through the logs.

Let's see how we could implement request logging and error logging with request IDs in Node Express. Again, we can use middleware for this.

A request logging middleware could look like this:

`/middlewares/requestLogger.js`:

```javascript
const requestLogger = (request, response, next) => {
  const requestId = request.headers["requestId"];

  console.log(
    `[Request]: ${requestId} ${request.method} ${request.url} ${JSON.stringify(
      request.params
    )}`
  );

  next();
};

export default requestLogger;
```

And similarly, an error logging middleware might look like this:

`/middlewares/errorLogger.js`:

```javascript
const errorLogger = (error, request, response, next) => {
  const requestId = request.headers["requestId"];

  console.error(`[Error]: ${requestId} ${error.stack || error.message}`);

  next(error);
};

export default errorLogger;
```

Notice, how the first parameter of the error logging middleware is the error object. We want to pass this object to the next error handler in the stack, in this case we would like at least to return the error to the user of the web application.

## How to Return Request IDs to the Users of the Web Application?

As we already have set the request ID in the response headers in our request ID middleware, the request id is returned to the user of the web application automatically.

How this request ID would be used by the user of the web application depends on the application. For example, the user could send the request ID to the support team if they encounter an error. With the request ID, the support team could easily find the related request and the error log from the logs.

To determine if the users of the users of the web application should be able to see the request ID, we would probably examine the error status in the frontend. Typically, if error status is 5xx, we would show the request ID to the user.

This way the user could pass the request ID to the support team for closer examination.

## How to pass Request IDs to Microservices?

In our APIs, we might make calls also to external services. If these services are managed by us, we could add the request ID to the request headers of the calls to the services. This way, we could follow the request through the logs of these services too.

We might or might not have the logs aggregated in one place from all these services. In any case, having the request ID available will make it easier to resolve issues.

## Conclusion

In web application development, error handling and logging are essential for tracking unexpected situations and returning meaningful error messages to users. A request ID is a unique identifier for each request that can be used for easy identification of requests and errors in the logs.

We have seen here an example how to implement a simple Node Express middleware for generating a request ID for each request. Depending on the platform used, the principle is still the same, just add a unique string to the request headers where it will be available later in the request-response cycle.

We have also seen how to log requests and errors with request IDs, and discussed about how to return the request ID to the users of the web application or how to pass the request ID to other services, again to be used for logging purposes and for connecting dots when resolving issues.

All in all, request IDs are a simple but powerful tool for tracking requests and errors in web applications.

## Author Information

Sami Ruokamo is a software developer and works at Buutti. He is interested in software development, especially in web development. He has been working with web application development for over 10 years. He is also interested in DevOps and cloud technologies.

To see the ideas presented in this article in action, check out https://github.com/samiru/node-express-react-template.
