# Authorization

In this guidance we'll examine two general approaches to authorization.

-	**Role-based authorization**. Authorize an action based on the roles assigned to a user. For example, some actions require an administrator role.
-	**Resource-based authorization**. Authorize an action based on a particular resource. For example, every resource has an owner. The owner can delete the resource; other users cannot.

A typical app will employ a mix of both. For example, to delete a resource, the user must be the resource owner _or_ an admin.

## Role-Based Authorization

In the Surveys application, we defined these roles:

- Administrator. Can perform all CRUD operations on any survey that belongs to that tenant.
- Creator. Can create new surveys
- Reader. Can read any surveys that belong to that tenant

Roles apply to _users_ of the application. In the Surveys application, a user is either an administrator, creator, or reader.

For a discussion of how to define and manage roles, see [Application roles](06-application-roles.md).

Regardless of how you manage the roles, your authorization code will look similar. ASP.NET 5 introduces an abstraction called _authorization policies_. With this feature, you define authorization policies in code, and then apply those policies to controller actions. The policy is decoupled from the controller.

To define a policy, first create a class that implements `IAuthorizationRequirement`. It's easiest to derive from `AuthorizationHandler`. In the `Handle` method, examine the relevant claim(s).

Here is an example from the Tailspin Surveys application:

```
public class SurveyCreatorRequirement : AuthorizationHandler<SurveyCreatorRequirement>, IAuthorizationRequirement
{
    protected override void Handle(AuthorizationContext context, SurveyCreatorRequirement requirement)
    {
        if (context.User.HasClaim(ClaimTypes.Role, Roles.SurveyAdmin) ||
            context.User.HasClaim(ClaimTypes.Role, Roles.SurveyCreator))
        {
            context.Succeed(requirement);
            return;
        }
        context.Fail();
    }
}
```

> See [SurveyCreatorRequirement.cs](https://github.com/mspnp/multitenant-saas-guidance/blob/master/src/Tailspin.Surveys.Security/Policy/SurveyCreatorRequirement.cs)

This class defines the requirement for a user to create a new survey. The user must be in the SurveyAdmin or SurveyCreator role.

In your startup class, define a named policy that includes one or more requirements. (If there are multiple requirements, the user must meet _each_ requirement to be authorized.) The following code defines two policies:

    services.AddAuthorization(options =>
    {
        options.AddPolicy(PolicyNames.RequireSurveyCreator,
            policy =>
            {
                policy.AddRequirements(new SurveyCreatorRequirement());
                policy.AddAuthenticationSchemes(CookieAuthenticationDefaults.AuthenticationScheme);
            });

        options.AddPolicy(PolicyNames.RequireSurveyAdmin,
            policy =>
            {
                policy.AddRequirements(new SurveyAdminRequirement());
                policy.AddAuthenticationSchemes(CookieAuthenticationDefaults.AuthenticationScheme);
            });
    });

> See [Startup.cs](https://github.com/mspnp/multitenant-saas-guidance/blob/master/src/Tailspin.Surveys.Web/Startup.cs)

Finally, to authorize an action in an MVC controller, set the policy in the Authorize attribute:

```
[Authorize(Policy = "SurveyCreatorRequirement")]
public IActionResult Create()
{
    // ...
}
```

In earlier versions of ASP.NET, you would set the **Roles** property on the attribute:

```
// old way
[Authorize(Roles = "SurveyCreator")]

```

This is still supported in ASP.NET 5, but it has some drawbacks compared with authorization policies:

-	It assumes a particular claim type. Policies can check for any claim type. Roles are just a type of claim.
-	The role name is hard-coded into the attribute. With policies, the authorization logic is all in one place, making it easier to update.
-	Policies enable more complex authorization decisions (e.g., age >= 21) that can't be expressed by simple role membership.

## Resource Based Authorization

_Resource based authorization_ occurs whenever the authorization depends on a specific resource that will be affected by an operation. In the Tailspin Surveys application, every survey has an owner and zero-to-many contributors.

-	The owner can read, update, delete, publish, and unpublish the survey.
-	The owner can assign contributors to the survey.
-	Contributors can read and update the survey.

Note that "owner" and "contributor" are not application roles; they are stored per survey, in the application database. To check whether a user can delete a survey, for example, the app checks whether the user is the owner for that survey.

In ASP.NET 5, you can implement resource-based authorization by creating a class that derives from **AuthorizationHandler** and overriding the **Handle** method.

```
public class SurveyAuthorizationHandler : AuthorizationHandler<OperationAuthorizationRequirement, Survey>
{
     protected override void Handle(AuthorizationContext context, OperationAuthorizationRequirement operation, Survey resource)
    {
    }
}
```

Notice that this class is strongly typed for Survey objects.  Register the class for DI on startup:

```
services.AddSingleton<IAuthorizationHandler>(factory =>
{
    return new SurveyAuthorizationHandler();
});
```

To perform authorization checks, use the **IAuthorizationService** interface, which you can inject into your controllers. The following code checks whether a user can read a survey:

```
if (await _authorizationService.AuthorizeAsync(User, survey, Operations.Read) == false)
{
    return new HttpStatusCodeResult(403);
}
```

Because we pass in a `Survey` object, this call will invoke the `SurveyAuthorizationHandler`.

In your authorization code, a good approach is to aggregate all of the user's role-based and resource-based permissions, then check the aggregate set against the desired operation.
Here is an example from the Surveys app. The application defines three permission types:

- Admin
- Contributor
- Creator
- Owner
- Reader

The application also defines a set of possible operations on surveys:

- Create
- Read
- Update
- Delete
- Publish
- Unpublsh

The following code creates a list of permissions, given a particular user and survey. Notice that this code looks at both the user's app roles, and the owner/contributor fields in the survey.

    protected override void Handle(AuthorizationContext context, OperationAuthorizationRequirement operation, Survey resource)
    {
        var permissions = new List<UserPermissionType>();
        string userTenantId = context.User.GetTenantIdValue();
        int userId = ClaimsPrincipalExtensions.GetUserKey(context.User);
        string user = context.User.GetUserName();

        if (resource.TenantId == userTenantId)
        {
            // Admin can do anything, as long as the resource belongs to the admin's tenant.
            if (context.User.HasClaim(ClaimTypes.Role, Roles.SurveyAdmin))
            {
                context.Succeed(operation);
                return;
            }

            if (context.User.HasClaim(ClaimTypes.Role, Roles.SurveyCreator))
            {
                permissions.Add(UserPermissionType.Creator);
            }
            else
            {
                permissions.Add(UserPermissionType.Reader);
            }

            if (resource.OwnerId == userId)
            {
                permissions.Add(UserPermissionType.Owner);
            }
        }
        if (resource.Contributors != null && resource.Contributors.Any(x => x.UserId == userId))
        {
            permissions.Add(UserPermissionType.Contributor);
        }
        if (ValidateUserPermissions[operation](permissions))
        {
            context.Succeed(operation);
        }
        else
        {
            context.Fail();
        }
    }


> See [Surveyauthorizationhandler.cs.](https://github.com/mspnp/multitenant-saas-guidance/blob/master/src/Tailspin.Surveys.Security/Policy/SurveyAuthorizationHandler.cs).

In a multi-tenant application, you must ensure that permissions don't "leak" to other tenant's data. In the Surveys app, the Contributor permission is allowed across tenants. (You can assign a user in another tenant as a contriubutor.) The other permission types are restricted to resources that belong to that user's tenant, so the code checks the tenant ID before granting those permission types. (The `TenantId` field as assigned when the survey is created.)

The next step is to check the operation (read, update, delete, etc) against the permissions. The Surveys app implements this step by using a lookup table of functions:

```
static readonly Dictionary<OperationAuthorizationRequirement, Func<List<UserPermissionType>, bool>> ValidateUserPermissions
    = new Dictionary<OperationAuthorizationRequirement, Func<List<UserPermissionType>, bool>>

    {
        { Operations.Create, x => x.Contains(UserPermissionType.Creator) },

        { Operations.Read, x => x.Contains(UserPermissionType.Creator) ||
                                x.Contains(UserPermissionType.Reader) ||
                                x.Contains(UserPermissionType.Contributor) ||
                                x.Contains(UserPermissionType.Owner) },

        { Operations.Update, x => x.Contains(UserPermissionType.Contributor) ||
                                x.Contains(UserPermissionType.Owner) },

        { Operations.Delete, x => x.Contains(UserPermissionType.Owner) },

        { Operations.Publish, x => x.Contains(UserPermissionType.Owner) },

        { Operations.UnPublish, x => x.Contains(UserPermissionType.Owner) }
    };
```

## Additional resources

- [Resource Based Authorization](https://docs.asp.net/en/latest/security/authorization/resourcebased.html)
- [Custom Policy-Based Authorization](https://docs.asp.net/en/latest/security/authorization/policies.html)
