title: IdentityServer4实现Token认证登录以及权限控制
author: 若邪
tags:
 - IdentityServer4
 - Token
 - 权限控制
categories:
 - 后端
date: 2017/11/5
---
### 相关知识点
不再对IdentityServer4做相关介绍，博客园上已经有人出了相关的系列文章，不了解的可以看一下：

蟋蟀大神的：[小菜学习编程-IdentityServer4](http://www.cnblogs.com/xishuai/tag/%5B34%5D%E5%B0%8F%E8%8F%9C%E5%AD%A6%E4%B9%A0%E7%BC%96%E7%A8%8B-IdentityServer4/)

晓晨Master：[IdentityServer4](http://www.cnblogs.com/stulzq/tag/IdentityServer4/)

以及Identity,Claim等相关知识：

Savorboard：[ ASP.NET Core 之 Identity 入门（一）](http://www.cnblogs.com/savorboard/p/aspnetcore-identity.html)，[ASP.NET Core 之 Identity 入门（二）](http://www.cnblogs.com/savorboard/p/aspnetcore-identity2.html)

### 创建IndentityServer4 服务
创建一个名为QuickstartIdentityServer的ASP.NET Core Web 空项目（asp.net core 2.0），端口5000


![](http://upload-images.jianshu.io/upload_images/2125695-cf107209ba258212.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![](http://upload-images.jianshu.io/upload_images/2125695-4805b07226cee1e4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

NuGet包：

![](http://upload-images.jianshu.io/upload_images/2125695-fc62950ab394b5fc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

修改Startup.cs 设置使用IdentityServer：

```cs
public class Startup
    {
        public void ConfigureServices(IServiceCollection services)
        {
            // configure identity server with in-memory stores, keys, clients and scopes
            services.AddIdentityServer()
                .AddDeveloperSigningCredential()
                .AddInMemoryIdentityResources(Config.GetIdentityResourceResources())
                .AddInMemoryApiResources(Config.GetApiResources())
                .AddInMemoryClients(Config.GetClients())
                .AddResourceOwnerValidator<ResourceOwnerPasswordValidator>()
                .AddProfileService<ProfileService>();
        }

        public void Configure(IApplicationBuilder app, IHostingEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseIdentityServer();
        }
    }
```
添加Config.cs配置IdentityResource，ApiResource以及Client：

```cs
 public class Config
    {
        public static IEnumerable<IdentityResource> GetIdentityResourceResources()
        {
            return new List<IdentityResource>
            {
                new IdentityResources.OpenId(), //必须要添加，否则报无效的scope错误
                new IdentityResources.Profile()
            };
        }
        // scopes define the API resources in your system
        public static IEnumerable<ApiResource> GetApiResources()
        {
            return new List<ApiResource>
            {
                new ApiResource("api1", "My API")
            };
        }

        // clients want to access resources (aka scopes)
        public static IEnumerable<Client> GetClients()
        {
            // client credentials client
            return new List<Client>
            {
                new Client
                {
                    ClientId = "client1",
                    AllowedGrantTypes = GrantTypes.ClientCredentials,

                    ClientSecrets =
                    {
                        new Secret("secret".Sha256())
                    },
                    AllowedScopes = { "api1",IdentityServerConstants.StandardScopes.OpenId, //必须要添加，否则报forbidden错误
                  IdentityServerConstants.StandardScopes.Profile},

                },

                // resource owner password grant client
                new Client
                {
                    ClientId = "client2",
                    AllowedGrantTypes = GrantTypes.ResourceOwnerPassword,

                    ClientSecrets =
                    {
                        new Secret("secret".Sha256())
                    },
                    AllowedScopes = { "api1",IdentityServerConstants.StandardScopes.OpenId, //必须要添加，否则报forbidden错误
                  IdentityServerConstants.StandardScopes.Profile }
                }
            };
        }
    }
```
因为要使用登录的时候要使用数据中保存的用户进行验证，要实IResourceOwnerPasswordValidator接口：
```cs
public class ResourceOwnerPasswordValidator : IResourceOwnerPasswordValidator
    {
        public ResourceOwnerPasswordValidator()
        {

        }

        public async Task ValidateAsync(ResourceOwnerPasswordValidationContext context)
        {
            //根据context.UserName和context.Password与数据库的数据做校验，判断是否合法
            if (context.UserName=="wjk"&&context.Password=="123")
            {
                context.Result = new GrantValidationResult(
                 subject: context.UserName,
                 authenticationMethod: "custom",
                 claims: GetUserClaims());
            }
            else
            {

                 //验证失败
                 context.Result = new GrantValidationResult(TokenRequestErrors.InvalidGrant, "invalid custom credential");
            }
        }
        //可以根据需要设置相应的Claim
        private Claim[] GetUserClaims()
        {
            return new Claim[]
            {
            new Claim("UserId", 1.ToString()),
            new Claim(JwtClaimTypes.Name,"wjk"),
            new Claim(JwtClaimTypes.GivenName, "jaycewu"),
            new Claim(JwtClaimTypes.FamilyName, "yyy"),
            new Claim(JwtClaimTypes.Email, "977865769@qq.com"),
            new Claim(JwtClaimTypes.Role,"admin")
            };
        }
    }
```
IdentityServer提供了接口访问用户信息，但是默认返回的数据只有sub，就是上面设置的subject: context.UserName，要返回更多的信息，需要实现IProfileService接口：
```cs
public class ProfileService : IProfileService
    {
        public async Task GetProfileDataAsync(ProfileDataRequestContext context)
        {
            try
            {
                //depending on the scope accessing the user data.
                var claims = context.Subject.Claims.ToList();

                //set issued claims to return
                context.IssuedClaims = claims.ToList();
            }
            catch (Exception ex)
            {
                //log your error
            }
        }

        public async Task IsActiveAsync(IsActiveContext context)
        {
            context.IsActive = true;
        }
```
context.Subject.Claims就是之前实现IResourceOwnerPasswordValidator接口时claims: GetUserClaims()给到的数据。
另外，经过调试发现，显示执行ResourceOwnerPasswordValidator 里的ValidateAsync，然后执行ProfileService 里的IsActiveAsync，GetProfileDataAsync。

启动项目，使用postman进行请求就可以获取到token：


![](http://upload-images.jianshu.io/upload_images/2125695-cf65dadc447a35d4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

再用token获取相应的用户信息：


![](http://upload-images.jianshu.io/upload_images/2125695-eebb00f8f68d8915.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

token认证服务一般是与web程序分开的，上面创建的QuickstartIdentityServer项目就相当于服务端，我们需要写业务逻辑的web程序就相当于客户端。当用户请求web程序的时候，web程序拿着用户已经登录取得的token去IdentityServer服务端校验。

### 创建web应用

创建一个名为API的ASP.NET Core Web 空项目（asp.net core 2.0），端口5001。

NuGet包：


![](http://upload-images.jianshu.io/upload_images/2125695-5dafa321288df66e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


修改Startup.cs 设置使用IdentityServer进行校验：

```cs
public class Startup
    {
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddMvcCore(option=>
            {
                option.Filters.Add(new TestAuthorizationFilter());
            }).AddAuthorization()
                .AddJsonFormatters();

            services.AddAuthentication("Bearer")
                .AddIdentityServerAuthentication(options =>
                {
                    options.Authority = "http://localhost:5000";
                    options.RequireHttpsMetadata = false;

                    options.ApiName = "api1";
                });
        }

        public void Configure(IApplicationBuilder app)
        {
            app.UseAuthentication();

            app.UseMvc();
        }
    }
```
创建IdentityController：
```cs
[Route("[controller]")]
    public class IdentityController : ControllerBase
    {
        [HttpGet]
        [Authorize]
        public IActionResult Get()
        {
            return new JsonResult("Hello Word");
        }

    }
```
分别运行QuickstartIdentityServer，API项目。用生成的token访问API：

![](http://upload-images.jianshu.io/upload_images/2125695-38958488026e6a83.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过上述程序，已经可以做一个前后端分离的登录功能。

实际上，web应用程序中我们经常需要获取当前用户的相关信息进行操作，比如记录用户的一些操作日志等。之前说过IdentityServer提供了接口/connect/userinfo来获取用户的相关信息。之前我的想法也是web程序中拿着token去请求这个接口来获取用户信息，并且第一次获取后做相应的缓冲。但是感觉有点怪怪的，IdentityServer不可能没有想到这一点，正常的做法应该是校验通过会将用户的信息返回的web程序中。问题又来了，如果IdentityServer真的是这么做的，web程序该怎么获取到呢，查了官方文档也没有找到。然后就拿着"Claim"关键字查了一通（之前没了解过ASP.NET Identity），最后通过HttpContext.User.Claims取到了设置的用户信息：

修改IdentityController ：
```cs
[Route("[controller]")]
    public class IdentityController : ControllerBase
    {
        [HttpGet]
        [Authorize]
        public IActionResult Get()
        {
            return new JsonResult(from c in HttpContext.User.Claims select new { c.Type, c.Value });
        }

    }
```


![](http://upload-images.jianshu.io/upload_images/2125695-da0e6d31052b0cb5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 权限控制
IdentityServer4 也提供了权限管理的功能，大概看了一眼，没有达到我想要（没耐心去研究）。
我需要的是针对不同的模块，功能定义权限码（字符串），每个权限码对应相应的功能权限。当用户进行请求的时候，判断用户是否具备相应功能的权限（是否赋予了相应的权限字符串编码），来达到权限控制。

IdentityServer的校验是通过Authorize特性来判断相应的Controller或Action是否需要校验。这里也通过自定义特性来实现权限的校验，并且是在原有的Authorize特性上进行扩展。可行的方案继承AuthorizeAttribute，重写。可是在.net core中提示没有OnAuthorization方法可进行重写。最后参考的ABP的做法，过滤器和特性共同使用。

新建TestAuthorizationFilter.cs
```cs
public class TestAuthorizationFilter : IAuthorizationFilter
    {
        public void OnAuthorization(AuthorizationFilterContext context)
        {
            if (context.Filters.Any(item => item is IAllowAnonymousFilter))
            {
                return;
            }

            if (!(context.ActionDescriptor is ControllerActionDescriptor))
            {
                return;
            }
            var attributeList = new List<object>();
            attributeList.AddRange((context.ActionDescriptor as ControllerActionDescriptor).MethodInfo.GetCustomAttributes(true));
            attributeList.AddRange((context.ActionDescriptor as ControllerActionDescriptor).MethodInfo.DeclaringType.GetCustomAttributes(true));
            var authorizeAttributes = attributeList.OfType<TestAuthorizeAttribute>().ToList();
            var claims = context.HttpContext.User.Claims;
            // 从claims取出用户相关信息，到数据库中取得用户具备的权限码，与当前Controller或Action标识的权限码做比较
            var userPermissions = "User_Edit";
            if (!authorizeAttributes.Any(s => s.Permission.Equals(userPermissions)))
            {
                context.Result = new JsonResult("没有权限");
            }
            return;

        }
    }
```

新建TestAuthorizeAttribute
```cs
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method, AllowMultiple = true)]
    public class TestAuthorizeAttribute: AuthorizeAttribute
    {

        public string Permission { get; set; }

        public TestAuthorizeAttribute(string permission)
        {
            Permission = permission;
        }

    }
```
将IdentityController [Authorize]改为[TestAuthorize("User_Edit")]，再运行API项目。

通过修改权限码，验证是否起作用


![](http://upload-images.jianshu.io/upload_images/2125695-a99385ed5e940a0e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

除了使用过滤器和特性结合使用，貌似还有别的方法，有空再研究。

本文中的[源码](https://github.com/wjkang/IdentityServer4_ResourceOwnerPasswords)














