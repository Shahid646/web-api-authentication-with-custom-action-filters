# Web API Authentication With Custom Action Filters

Web API includes filters to add extra logic before or after action method executes. Filters can be used to provide cross-cutting features such as logging, exception handling, performance measurement, authentication and authorization.

For this demo we will be creating 3 custom classes which will be derieved from  ActionFilterAttribute and they are as follows,

- AuthenticateWithApiKey.cs
- AuthenticateWithJwt.cs
- AuthenticateWithLdap.cs

We will override OnActionExecuting method in above classes to write our custom logic. Once that's done we will use our custom filters in our controller **(EmployeeController)** as below.

```
using System.Web.Http;
using ActionFilter.WebAPI.AuthenticationFilter;

namespace ActionFilter.WebAPI.Controllers
{
    [RoutePrefix("Employee")]
    public class EmployeeController : ApiController
    {
        [Route("AuthenticateWithApiKey")]
        [AuthenticateWithApiKey]
        [HttpGet]
        public string AuthenticateWithApiKey()
        {
            return "API Key Authentication Successful.";
        }

        [Route("AuthenticateWithJwt")]
        [AuthenticateWithJwt]
        [HttpGet]
        public string AuthenticateWithJwt()
        {
            return "JWT Authentication Successful.";
        }

        [Route("AuthenticateWithLdap")]
        [AuthenticateWithLdap]
        [HttpGet]
        public string AuthenticateWithLdap()
        {
            return "LDAP Authentication Successful.";
        }


    }

    public class Employee
    {
        public int EmployeeId { get; set; }
        public string EmployeeName { get; set; }

    }
}
```

Above you can see we have decorated our fuctions with custom action filters

# Deep dive

**AuthenticateWithApiKey**

Authentication with api keys is very useful. You can subscribe users, generate api keys and give them so that they can use it to authenticate themselves.

You can also use database here to validate the api keys. For the demo purpose we have hardcoded the api key.

If you look at below code, you can see we are passing **"apikey"** as a parameter in headers and checking the value. If value matches with **123456789** then the user is validated else we will send an un-authorized code back. 


```
public override void OnActionExecuting(HttpActionContext actionContext)
{
	bool validKey = false;
	actionContext.Request.Headers.TryGetValues("apikey", out var requestHeaders);

	if (requestHeaders != null)
	{
		if (requestHeaders.FirstOrDefault() == "123456789")
		{
			validKey = true;
		}
	}

	if (!validKey)
	{
		actionContext.Response = new HttpResponseMessage(HttpStatusCode.Unauthorized);
	}
}
```

If you call **AuthenticateWithApiKey** method in Employee Controller with out passing apikey parameter you will get un-authorized error as seen below.


| ![images/apikey-error-no-apikey.PNG](images/apikey-error-no-apikey.PNG) |
| ------------------------------------------------------------------- |
 

Sending correct api key will give you success result as seen below.


| ![images/apikey-success.PNG](images/apikey-success.PNG) |
| ------------------------------------------------------------------- |

**AuthenticateWithLdap**

Another way to authenticate users is with Ldap. Here we have again hardcoded the user and password for demo purpose but you can use Ldap validation as per your requirement.

Here again we have overridden OnActionExecuting method to write our custom logic.


```
public override void OnActionExecuting(HttpActionContext actionContext)
{
if (actionContext.Request.Headers.Authorization == null)
{
	actionContext.Response = new HttpResponseMessage(HttpStatusCode.Unauthorized);
}
else
{
	var authenticationToken = actionContext.Request.Headers.Authorization.Parameter;
	var decodedToken = Encoding.UTF8.GetString(Convert.FromBase64String(authenticationToken));
	var userName = decodedToken.Substring(0, decodedToken.IndexOf(":", StringComparison.Ordinal));
	var userPassword = decodedToken.Substring(decodedToken.IndexOf(":", StringComparison.Ordinal) + 1);

	//You can use ldap here to verify user
	if (userName == "ldap_user" && userPassword == "ldap_password")
	{
		//Authorized
	}
	else
	{
		actionContext.Response = new HttpResponseMessage
		{
			StatusCode = HttpStatusCode.Unauthorized,
			Content = new StringContent("You are unauthorized to access this resource")
		};
	}
}
}
```

Let's call **AuthenticateWithLdap** without passing userName and userPassword. You will see an un-authorized error as below.


| ![images/ldap-error-no-user-password.PNG](images/ldap-error-no-user-password.PNG) |
| ------------------------------------------------------------------- |


Now we provide the necessory parameter but with wrong password. Make sure you choose **"Basic Auth"** under Authorization. Still you get error as below.


| ![images/ldap-error-wrong-user-password.PNG](images/ldap-error-wrong-user-password.PNG) |
| ------------------------------------------------------------------- |


Lastly we will send correct information. Make sure you choose **"Basic Auth"** under Authorization. The result is as below.


| ![images/ldap-success.PNG](images/ldap-success.PNG) |
| ------------------------------------------------------------------- |


**AuthenticateWithJwt**

There is no denying that JWT is a cool breeze. A JWT token consists of three simple parts: a header describing the token, a payload that's the actual token, and a cryptographically secured signature, ensuring the token was created by a trusted source. All three components are base 64 encoded, separated by a ".", concatenated, and normally provided as a Bearer token in the Authorization HTTP header of your HTTP REST invocations — dead simple in fact.

Below is a typical example of a JWT token.

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

The reason why this is secure is because some sort of "secret" has been used when creating the signature, which is the last part of the token. Without the secret, you might as well try to brute force the unified theory of science. The thing is solid as a rock! Yet as simple as a cup of coffee on a Sunday morning.

Let's look at the overridden method below

```
public override void OnActionExecuting(System.Web.Http.Controllers.HttpActionContext actionContext)
{
	string secret = "db3OIsj+BXE9NZDy0t8W3TcNekrF+2d/1sFnWG4HnV8TZY30iTOdtVWJG8abWvB1GlOgJuQZdcF2Luqm/hccMw==";
	var request = actionContext.Request;
	if (!WithoutVerifyToken(request.RequestUri.ToString()))
	{
		if (request.Headers.Authorization == null || request.Headers.Authorization.Scheme != "Bearer")
		{
			actionContext.Response = new HttpResponseMessage(HttpStatusCode.Unauthorized);
		}
		else
		{
			var jwtObject = JWT.Decode<Dictionary<string, object>>(
				request.Headers.Authorization.Parameter,
				Encoding.UTF8.GetBytes(secret),
				JwsAlgorithm.HS512);

			if (IsTokenExpired(jwtObject["Exp"].ToString()))
			{
				actionContext.Response = new HttpResponseMessage(HttpStatusCode.Unauthorized);
			}
		}
	}

	base.OnActionExecuting(actionContext);
}
```

Here the secret is very important as i mensioned above. Here we validate the token sent from the request. But wait how do we get the token which needs to be send and verified. 

To accomplish this we need to make another api to **TokenController**. We need to pass Account which is user name and a password. I have hardcoded the values. But you can again use Ldap validation here if you want or database tables which holds some users and password. It's totally up to you how you want to acheive this.


Let's call **AuthenticateWithJwt** under **EmployeeController** without passing baerer token. The result will look like below.


| ![images/jwt-error-no-token.PNG](images/jwt-error-no-token.PNG) |
| ------------------------------------------------------------------- |


Now let's get the token first by passing valid Account(user name) and password as below.


| ![images/get-token-success.PNG](images/get-token-success.PNG) |
| ------------------------------------------------------------------- |


You can see we got a token back. Now let's pass this baerer token to the **AuthenticateWithJwt**.

| ![images/jwt-success.PNG](images/jwt-success.PNG) |
| ------------------------------------------------------------------- |


Hurray!!! We are validated with the jwt token successfully.
