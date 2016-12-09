# Technical Screening Test Questions

## Server Side Questions (C#)

### Service Side Toubleshooting
 
** You have an application that you want to debug, it is not a web site and could be a lot of tasks and threads. What are the various way of debugging the application?**
 
*Ways of debugging the application.* This is a wide subject with many different circumstances, strategies and toolsets. I will assume we are on Windows OS and we have the more difficult situation of not been able to reproduce the problem in Dev. We therefore need to debug the app in **production** at least until we can understand it well enough to reproduce it. Furthermore, we will assume the application is monolithic with a database and native procedure calls as opposed to a distributed app with some rpc method and distributed logs as this requires different approach and toolset.
 
*Some General Strategies*
   - Get feedback of your peers. Until you are able to reproduce a problem, debugging is a more creative process that benefits greatly from their input.  
   - Debugging using StackOverflow is like a depending on a GPS, it can be quick and useful, but tends to reduce our understanding of where we are and how we get to our destination. And in my experience, a bug is like a rotten apple in a basket of apples, it is best to check for others.  
   - Make reliable observations and then listen carefully to what they are telling you -- because too often, we *only* see what we already believe.  
   - If you have alot of tasks and threads, use a tool that supports flame graphs, [see example](https://randomascii.wordpress.com/2016/09/05/etw-flame-graphs-made-easy/)  
   - Visual Studio has great debugging tools which can be used to explore code or nail the reproduced problem, but even the remote debugging tools are not normally for use in production. On the other hand ETW can be used in production. 
   - MS SQL Server has great debugging analysis tools as well.
   - Use some iterative process like an OODA loop, for example:
      1. Discuss the problem to your peers and brainstorm for what and where to make observations.
      2. Make observations using tools like ETW (Event Tracing Windows) related toolsets like xperf, UIforETW, Dynatrace and New Relic. Other tools like intellitrace and procdump/windbg can help when you can reproduce the problem in production. 
      3. If the bug still can not be understood or reproduced in Dev, and assuming that we are able to deploy easily to production, I would consider instrumenting and experimenting the code. This intrumentation would use ETW or other low friction mechanism.
      4. If the bug can now be reproduced with test cases, solve it using Visual Studio and deep inspection (breakpoints, watches, etc.), otherwise goto step 1.

** For windows services if you want to stop/start a service what are the ways to do it?**

There are many ways to start and stop windows service. The code based methods include Powershell (start-service/stop-service), windows C++ code (ServiceStart/ControlService), in managed code (System.ServiceProcess.ServiceController.Start/.Stop), or in a batch file (sc.exe). Perhaps best of all of these might be using [Powershell Desired State Configuration Service resource](https://msdn.microsoft.com/en-us/powershell/dsc/serviceresource). Less recommendable in production is to use the Service Manager user interface.

System orchestration and configuration tooling (e.g. System Center, DCOS) should be used when available to control and track changes. However, for complex distributed systems such as MS SQL, SharePoint, and IIS, you normally do not manage the production services directly, but use the system's provided management tools which in turn manage the services.

** What is the quickest way of seeing if you are having a memory leak?**

On windows, use [Performance Manager](https://msdn.microsoft.com/en-us/library/windows/hardware/ff541886(v=vs.85).aspx), confirm that you have a memory leak and where (user or kernel side) .
If you have a managed code memory leak you can use a combination of procdump.exe and Visual Studio (Actions - Debug Managed Memory).
Otherwise, use ETW tooling to determine where in your program (user side) or os drivers (kernel side) the memory leak is.

### Anonymous Functions/Delegates

** Assuming you have the following class, please write a function the computes the circumference of circle (2πr)**

```cs
   public sealed class Circle {
      private double radius;
      public double Calculate(Func<double, double> op)   {
         return op(radius);
      }
   }
```

Answer:

```cs
  var circle = new Circle();
  var circumference = circle.Calculate((r) => 2 * Math.PI * r);
```

This returns the circumference of a circle with given radius of zero. In order to make this more useful, you would want to refactor the class to have a getter and setter for radius, or declare it as public.
You can use [reflection](http://stackoverflow.com/questions/934930/can-i-change-a-private-readonly-field-in-c-sharp-using-reflection) to change the private field, but this is really bad practice that can undermine code trustworthiness.
Another unattractive alternative would be to use a extension method that used its own radius, but this also is less desirable because the object's field in now not used:

```cs
public static class CircleExtension
{
       public static double SetRadiusAndCalculate(this Circle c, int radius)
        {
            return c.Calculate(y => { return 2 * Math.PI * radius; });
        }
}
Circle x = new Circle();
double perimeter = x.SetRadiusAndCalculate(7.5);
```

### Understanding of Threads

** Explain Thread deadlock? Assuming you have MVC application what will the following do**

```cs
   Public void button1Click(object sender, EventArgs e){
      Task<string> s = LoadStringAsync();
      textBox1.Text = s.Result;
   }
```

The above code is likely to hang due to a deadlock on the UI thread. Thread deadlock is caused by the code as written because the click handler (outer function) blocks the UI thread waiting for the result, while the inner function that provides the result is likely to use the event bus to ask the blocked UI thread to execute the code. This particular deadly embrace is cool because the code is concise and looks so innocent.

This code can be fixed using the `async...await` language construct where the click handler is declared as `async`, and the `s.Result` is replaced with `await s`. At the point of the implicit callback, s is no longer the task, but the task's string return value.

** What is wrong with the following code under load**

```cs
public static SQLConnection Conn{return "DataSource=.;Initial Catalog=NorthWind;Integrated Security=True";}
public void ShowFirstTenUsers(){
   using (SQLCommand command = new SQLCommand(Conn)){
      command.CommandText = "Select * from Users limit 10";
      SqlDataReader reader = command.ExecuteReader();
      try {
         while (reader.Read()) {
            Console.WriteLine(String.Format("{0}, {1}",reader[0], reader[1]));
         }
      }
   }
}
```

There seems to be a typo in the declaration of `Conn` where it does not actually create the declared type (SQLConnection). Assuming that is corrected, there is still a problem under load. The declared static connection is probably intended as a performance optimization, but this is not thread safe, and the performance optimization is already accomplished with connection pooling of the underlying SQL driver. If this code is called multiple times in web server backend, for instance, it will likely cause an exception.

**1.How can you improve performance?**

```cs
private static void ShowFirstTenUsers(string connectionString)
{
    string queryString = 
        "Select * from Users limit 10;";
    using (SqlConnection connection = new SqlConnection(
               connectionString))
    {
        SqlCommand command = new SqlCommand(
            queryString, connection);
        connection.Open();
        SqlDataReader reader = command.ExecuteReader();
        try
        {
            while (reader.Read())
            {
                Console.WriteLine(String.Format("{0}, {1}",
                    reader[0], reader[1]));
            }
        }
        finally
        {
            // Close reader when done
            reader.Close();
        }
    }
}
```

**2.How can you minimize schema change failures**

Instead of using '*' as the field list, use the specific field names that are intended for display.

**3.How can you improve exception handling?**

Depending on the contract with the callers, you can add catch blocks for common exceptions like connection initiation and timeouts.

### Understanding of Design/Architecture approach

** You are reading files from a directory and then storing them to a data storage, describe** 

**1.How would you design it? White boarding is fine.**

In order to allow for many or large file transfers, the function should have two features
- break large files into multiple parts which limits the impact of network failure and allows for parallelism  
- upload in parallel to increases performance
Each file could have three steps, initiation, parallel transfer, finalization. Single-part files can be transferred in parallel as well. The initation would divide up the file into parts, initate the transfer and receive an id for the transfer. This id can be used for further control or completion of the transfer. The parallel transfers would be done numbering each and receiving a confirmation number for each part's completion. The finalization request is made by submitting the list of parts and confirmation id's and then the parts are reassembled and completion confirmed. An abort is also possible and will allow the receiving end to clean up incomplete parts. Incomplete transfers can be cleaned up after a set exiry date.

**2.What potential IO or Memory bound problems you could encounter.**
The multi-part strategy limits the effects of network timeouts and other failures.
This strategy also moderates IO and memory resource use.
There is a trade-off between the number of parallel threads and various physical resource limits. 
Assuming normal scale characteristics, I would start with 10-12 parallel uploads.

Building RPC's protocols like this can be very nuanced and requires much testing. It should only be done where characteristics of existing protocols can not be used.  
Some alternatives that I use are:  
1. Use Http/2. Http 1.1 file upload will do both mime-multipart uploads and Http 2.0 will do so with native binary and parallel multiplexing and provides much better results. 
2. Use Cloud vendor tools. Organize your files and folders, then use cloud vendor tooling for filesystem to cloud-filesystem transfer (and syncronization). Then use a well scaled cloud-local processing to move it to storage. For example, syncronizing a directory with the cloud file system can be as simple as `aws s3 sync . s3://mystoragebucket --recursive`. Then you are left with transferring it to other storage using the cloud providers low latency high bandwidth network and any choice of compute sizing to match the task. This is a good place to use "serverless" technology like AWS Lambda or Azure Functions that can be triggered by file transfer completion.
3. Use Cloud vendor tooling api. Same as #2, but using your favourite language and an SDK to have deeper integration and control of the process.

**3.How would you solve the add/delete problem of the same folder.**

Assuming that you want a one way sycronization from file system to storage, including propagating deletes, you can do a backward pass which goes through the target storage and verifies the matching item's existence in the authoritative source directory. If it does not exist, then remove the item from the target. 

## Web API Related Questions

** What is the difference between Get/POST and what implications does it have for SSL**

1. Data. A get requests has a uri, query string, cookies, but no request body. A post includes a request body of arbitrary length, whereas the get request method has a limited length subject to limits (around 1-2Kb). Data on the url query string although encrypted by transport layer security, may be recorded in various ways (e.g. history, plugins) on the client side and on the encryption termination side (device or server), whereas the request body data normally is not. 
2. Security. The common way to protect against cross site request forgeries is to use a hidden form with a CSRF token which has a limited lifespan. Although it can be done with either get or post, it has implications for the payload size (see Data above), and re-submission (see Caching and Idempotency below). 
3. Caching. A get response can be cached by the client or server, whereas a post can not be. A network device (e.g. F5 BigIP) or reverse proxy (e.g. nginx) can also cache if it is used for encryption termination or otherwise knows the session encryption keys (like a man-in-the-middle trusted proxy).
4. Idempotency. A get request can be assumed to be idempotent, whereas with a post, it can not be assumed.


** Why use JSON over XML? JSON isn’t that much smaller.**

JSON can be preferred over XML in cases that you do not need type a tight schema, or message validation is simple or not required, and your environment components support json. JSON is very easy to use in Browsers and NodeJS environments where is represents a native objects. JSON is easier to read than XML.   

** Can you write a JSON representation of person object**

```JSON
{"name":"Don Gillis", "address":"638 Main", "telephone":"514-887-8100"}
```

** Why does cookie based authentication doesn’t work for an API**

Cookies are a browser technology, so this would limit callers to browsers.
Composed web applications can have api calls to multiple domains and browsers do not share cookies between domains.

** How does Google/Facebook authentication work and why is it secure?**

Google and Facebook use OIdC (OpenID Connect) which in turn uses Oauth 2.0. This is to allow access to "resources" (like facebook) from web clients, native clients (like mobile apps), and headless services like background tasks. It is secure because the "Client" which is working on your behalf does not need to know your credentials and you do not know the "client" resource access credentials. Yet the "client" can provide a fresh token that indicates that you are or were present (depending on the circumstances). The token is generated for a web app in a multistep process where the "client" redirects you to authenticate with the resource with an auth code is given on the URL. The "resource" then verifies or prompts for consent and then redirects you back to a "client" url with the new token as payload. This "client" is usually registered with the resource provider and for web purposes includes a callback URL which further secures usage. Your consent to specific data access is also kept in this registry. All these communications are channel encrypted using TLS.

## JavaScript Questions
** What is the difference and why would you choose one vs. the other.**

```javascript   
   var sum = function (a, b){ return a+b; }
```

vs.

```javascript
   var sum = function (a,b, callback) { 
      console.log(a+b);
      callback;
   };
```

Answer: The difference (after the fix below is applied), is that one is a syncronous call and the other is asyncronous. The first is simpler to follow, but the second avoids blocking the javascript thread. So if the operation is possibly long running prefer the second, otherwise prefer the simpler code. Although not directly applicable here, an interesting alternative way to avoid not blocking the javascript UI thread is to use [web workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers).

** What is wrong with this function**

The line `callback;` should read `callback(a+b)`.


** Answer the following questions with the snippet**

```javascript
   var myObject = {
     foo: "bar",
     func: function() {
       var self = this;
       console.log("outer func:  this.foo = " + this.foo);
       console.log("outer func:  self.foo = " + self.foo);
       (function() {
         console.log("inner func:  this.foo = " + this.foo);
         console.log("inner func:  self.foo = " + self.foo)
       });
     }
   };
   myObject.func();
```

**1.What will this print?**

It is probably intended to demonstrate that the scope of the inner function "this" does not include "foo", whereas using the var self allows access to the outer vars. However it will only execute the outer function and will print:

```
         outer func: this.foo = bar

         outer func: self.foo = bar 
```

**2.What bugs do you see?**

It was probably intended to immediately invoke the inner function, therefore it should be writen as to cause immediate invocation as follows:

```javascript
      (function() {
         console.log("inner func:  this.foo = " + this.foo);
         console.log("inner func:  self.foo = " + self.foo)
       })();
       // note the added () in the line above could have been added within the final parenthesis   
```

Assuming that foo in not defined in the global/window scope, this will give us the expected result of:


```
         outer func: this.foo = bar

         outer func: self.foo = bar 

         inner func: this.foo = undefined

         inner func: self.foo = bar 

```

## Hands-on test

**Write an Angular Review management application which allows you to write review, paginate them and sort them.** 

**You must use:**

**- Services**

**- Some sort of persistence (Local Storage is acceptable)**

**- Write test scenarios using BDD (given/when/test)** 

ITH
