# GlobalExceptionHandler
Custom Middleware – Global Exception Handling In ASP.NET Core
In this section let’s create a Custom Global Exception Handling Middleware that gives even more control to the developer and makes the entire process much better.

Custom Global Exception Handling Middleware – Firstly, what is it? It’s a piece of code that can be configured as a middleware in the ASP.NET Core pipeline which contains our custom error handling logics. There are a variety of exceptions that can be caught by this pipeline.
We will also be creating Custom Exception classes that can essentially make your application throw more sensible exceptions that can be easily understood.
But before that, let’s build a Response class that I recommend to be a part of every project you build, at least the concept. So, the idea is to make your ASP.NET Core API send uniform responses no matter what kind of requests it gets hit with. This make the work easier for whoever is consuming your API. Additionally it gives a much experience while developing.

Create a new class ApiResponse and copy down the following  
```
public class ApiResponse<T>
{
    public T Data { get; set; }
    public bool Succeeded { get; set; }
    public string Message { get; set; }
    public static ApiResponse<T> Fail(string errorMessage)
    {
        return new ApiResponse<T> { Succeeded = false, Message = errorMessage };
    }
    public static ApiResponse<T> Success(T data)
    {
        return new ApiResponse<T> { Succeeded = true, Data = data };
    }
}
```
The ApiResponse class is of a generic type, meaning any kind of data can be passed along with it. Data property will hold the actual data returned from the server. Message contains any Exceptions or Info message in string type. And finally there is a boolean that denotes if the request is a success. You can add multiple other properties as well depending on your requirement.

We also have Fail and Success method that is built specifically for our Exception handling scenario. You can find how this is being used in the upcoming sections.

As mentioned earlier, let’s also create a custom exception. Create a new class and name it SomeException.cs or anything. Make sure that you inherit Exception as the base class. Here is how the custom exception looks like.
```
public class SomeException : Exception
{
    public SomeException() : base()
    {
    }
    public SomeException(string message) : base(message)
    {
    }
    public SomeException(string message, params object[] args) : base(String.Format(CultureInfo.CurrentCulture, message, args))
    {
    }
}
```
Here is how you would be using this Custom Exception class that we created now.

throw new SomeException("An error occurred...");


Get the idea, right? In this way you can actually differentiate between exceptions. To get even more clarity related to this scenario, let’s say we have other custom exceptions like ProductNotFoundException , StockExpiredException, CustomerInvalidException and so on. Just give some meaningful names so that you can easily identify. Now you can use these exception classes wherever the specific exception arises. This sends the related exception to the middleware, which has logics to handle it.

Now, let’s create the Global Exception Handling Middleware. Create a new class and name it ErrorHandlerMiddleware.cs
```
public class ErrorHandlerMiddleware
{
    private readonly RequestDelegate _next;
    public ErrorHandlerMiddleware(RequestDelegate next)
    {
        _next = next;
    }
    public async Task Invoke(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception error)
        {
            var response = context.Response;
            response.ContentType = "application/json";
            var responseModel = ApiResponse<string>.Fail(error.Message);
            switch (error)
            {
                case SomeException e:
                    // custom application error
                    response.StatusCode = (int)HttpStatusCode.BadRequest;
                    break;
                case KeyNotFoundException e:
                    // not found error
                    response.StatusCode = (int)HttpStatusCode.NotFound;
                    break;
                default:
                    // unhandled error
                    response.StatusCode = (int)HttpStatusCode.InternalServerError;
                    break;
            }
            var result = JsonSerializer.Serialize(responseModel);
            await response.WriteAsync(result);
        }
    }
}
```
Line 3 – RequestDelegate denotes a HTTP Request completion.
Line 10 – A simple try-catch block over the request delegate. It means that whenever there is an exception of any type in the pipeline for the current request, control goes to the catch block. In this middleware, Catch block has all the goodness.

Line 14 – Catches all the Exceptions. Remember, all our custom exceptions are derived from the Exception base class.
Line 18 – Creates an APIReponse Model out of the error message using the Fail method that we created earlier.
Line 21 – In case the caught exception is of type SomeException, the status code is set to BadRequest. You get the idea, yeah? The other exceptions are also handled in a similar fashion.
Line 34 – Finally, the created api-response model is serialized and send as a response.
Before running this implementation, make sure that you don’t miss adding this middleware to the application pipeline. Open up the Startup.cs / Configure method and add in the following line.
```
app.UseMiddleware<ErrorHandlerMiddleware>();
```
