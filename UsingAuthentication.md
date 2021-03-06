![Authentication](images/Ride_Authentication.jpeg)

# Using Authentication in your Ride Calls

* [Overview](#overview)
* [Rest-Assured Filters](#rest-assured-filters)
* [Passing a Filter to a Ride call](#passing-a-filter-to-a-ride-call)
* [The Authentication Sample Filter](#the-authentication-sample-filter)
* [Applying the Sample Filter](#applying-the-sample-filter)
* [Final Thoughts](#final-thoughts)

# Overview

Ride provides for the ability to utilize Rest-Assured Filters to augment calls into Ride framework as they are being made.  This filter workflow also allows for the myriad of ways for a system to authenticate a user and authorize calls.  This is the recommended way to add authentication to calls from your Ride extension.

## Rest-Assured Filters

Per its [documentation](https://github.com/rest-assured/rest-assured/wiki/usage#filters), Rest-Assured filters allow you "to inspect and alter a request before it's actually committed and also inspect and alter the response before it's returned to the expectations".  In the Ride core, you can send an arbitrary number of Filters to call any REST API, so adding a filter for authentication will not interfere (as long as your code is correct) with any other filters.

## Passing a Filter to a Ride call

In the RestApiController class in the ride core, one of the core calls is fireRestCall. Let's take a look at the signature of this method:

```
public static Response fireRestCall(String callingService, String restAPI,
      RequestSpecBuilder reqBuilder, ResponseSpecification expectedResponse, Method method,
      Filter... filters)
```

For the purposes of this document, we'll ignore the first 5 arguments and focus on the last one.  This is a variable length argument that allows you to pass in a single filter, or a list of comma delimited filters, or even pass in a Filter[] array object.  Ride is prepared to handle all of those cases.

## The Authentication Sample Filter

In the Ride samples there is a com.adobe.ride.sample.filters.AuthFilter in the sample-service-extension project.  This filter shows an example of how you can define how your system takes authentication data and authorizes calls to your API:

```
1  public class AuthFilter implements Filter {
2  @SuppressWarnings("unused")
3  private final String callingServiceName;
4
5   public AuthFilter(String callingServiceName) {
6    Validate.notNull(callingServiceName);
7    this.callingServiceName = callingServiceName;
8   }
9
10  public AuthFilter() {
11    this.callingServiceName = "";
12  }
13
14  public Response filter(FilterableRequestSpecification requestSpec,
15      FilterableResponseSpecification responseSpec, FilterContext ctx) {
16
17    if (!requestSpec.getHeaders().hasHeaderWithName("Authorization")) {
18
19      // Token retrieved from some code invoked here
20      String token = MyAuthenticationLibrary.authenticate(callingServiceName)
21      requestSpec.header(new Header("Authorization", token));
22    }
23
24    final Response response = ctx.next(requestSpec, responseSpec);
25
26    return response;
27  }
28}
```

There are two important things to look at here: (1) the constructor which takes an argument (line 5), and (2) the filter method (line 14).  

Let's look at the filter method first.  This class, in general, uses the standard workflow for using Rest-Assured filters (i.e. implementing the Rest-Assured Filter interface).  In this filter method (which must be implemented per the filter), you should place all the code you require in order to augment your call to have it properly authenticate.  You can import whatever external class and libraries you need, in order to add information to your call.  In this sample code, the filter checks to see if there is already an "Authorization" header in the RestAssured RequestSpec, and if there isn't, it calls a static method on an external class to retrieve a token for the header.  In your filter, you would leverage your own authentication flows.

The Constructor, which takes an argument, allows you to pass in service-specific data at runtime in order to be able to re-use this filter for multiple services, if you require that.

## Applying the Sample Filter

In order to apply this filter we simply need to instantiate it, and pass it to our Ride call as shown in the [Passing a Filter to a Ride Call](#passing-a-filter-to-a-ride-call) section above, but let's take a look at the sample code:

```
public class SampleServiceController extends RestApiController {
  
  public static final Filter filter = new AuthFilter("SampleService");
  
  ...
  
  public static Response callCore(ObjectType type, String objectPath, ModelObject object,
      ResponseSpecification expectedResponse, Method method, boolean addAuthorization) {
    
    ...
    
    Response response = null;
    if (addAuthorization) {
      response = fireRestCall(Service.SAMPLE_SERVICE.toString(), objectPath, reqBuilder,
          expectedResponse, method, filter);
    } else {
      response = fireRestCall(Service.SAMPLE_SERVICE.toString(), objectPath, reqBuilder,
          expectedResponse, method);
    }
    return response;
  } 
  
  ...
}
```

Notice how the filter is setup as soon as the class is loaded, and then when a method in the sample extension called "callCore" is called. The method determines whether the call is to be tried with or without Authorization and then adds the Filter, if the Authorization is called for.  Because addAuthorization is a variable argument, the method can handle the case when you do not specify a value there.

## Final Thoughts

Because Filters are so flexible, almost any possible authentication/authorization scheme should be accommodated.


