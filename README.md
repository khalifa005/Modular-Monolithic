# Modular-Monolithic

What is Modular Monolithic Architecture?
Modular Monolith Architecture is a software design in which a monolith is made better and modular with importance to re-use components/modules. It’s often quite easy to transition from a Monolith Architecture to Modular Monolith.

Here is an illustration:

![image](https://user-images.githubusercontent.com/29863643/230741640-0b293081-f503-4b37-b19d-3d239fe557ed.png)


The Need to Build Better Monoliths
Transitioning to Microservices is a real painful process given that you educate the entire team of all the required basics at the practical level. Sticking to a well-built monolith is a viable solution for most of the products and code bases. When your product’s user base explodes, that is when you would actually need to transition to a Microservices Approach. But until then, design your application in such a way that it mimics Microservices yet keeps the simplicity and goodness of Monoliths. Makes sense, yeah?


The main idea here is to build a better Monolith Solution.

API / Host – A very thin Rest API / Host Application that is responsible for registering the controllers/services of other modules into the service container.
Modules – A logical block of the business unit. For example, Sales. Everything that is related to Sales can be found here. We will walk through the definition of a module in the next section.
Shared Infrastructure – Application-Specific Interfaces and implementations are found here for other modules to consume. This includes Middlewares, Data Access providers, and so on.
Finally a Database. Note that you have the flexibility to use multiple databases, i.e one database per module.But it ultimately depends on how you would want to design your application.


You can see that there is not much deviation from a standard Monolith implementation. The basic recipe is to Split your Application into multiple smaller applications/modules and make them follow clean architecture principles.

Definition of a Module
A module is a logical unit of the business requirement. In a Point of Sales application, sales, customers, and Inventory are a few examples of Modules.
Each module will have a DBContext and can access only the assigned table/entities.
One module should never depend on any other module. It can depend on Abstraction Interfaces that are present in Shared Application Projects.
Each module has to follow a domain-driven architecture
Every module will be further split into API, Core, and Infrastructure projects to enforce Clean Onion Architecture.
Cross Module communication can happen only via Interfaces/events/in-memory bus. Cross Module DB Writes should be kept minimal or avoided completely.
To get a better understanding, let’s see an actual module from the fluentpos project and examine its responsibility.


![image](https://user-images.githubusercontent.com/29863643/230742085-2094ba56-22bd-4b8a-bf91-3659f47682b1.png)

Modules.Catalog – Contains the API Controllers needed for the module.
Modules.Catalog.Core – Contains Entities, Abstractions, CQRS handlers, and everything needed for the module to function independently.
Modules.Catalog.Infrastructure – Consists of DbContexts and Migrations. This project depends on the Core for abstractions.
You will get more idea on this when to start to build an application later in this article.


Benefits of Modular Architecture in ASP.NET Core
Clear Separation of Concerns
Easily Scalable
Lower complexity compared to Microservices
Low operational / deployment costs.
Reusability
Organized Dependencies

Cons of Modular Architecture when compared to Microservices
Not Multi-technology compatible.
Horizontal Scaling can be a concern. But this can be managed via load balancers.
Since Interprocess Communication is used, messages may be lost during Application Termination. Microservices combat this issue by using external messaging brokers like Kafka, RabbitMQ. (You can still use message brokers in Monoliths, but let’s keep things simple)


Examining fluentpos Project Structure


![image](https://user-images.githubusercontent.com/29863643/230742279-2ecd3568-5cf4-4cf2-834a-4f55e83e1efe.png)

What we will build
We will be building a simple application that demonstrates the implementation of Modular Architecture in ASP.NET Core. We will not be building a full-blown application in this article, as it may require lots of explanation. I plan to build a framework / Nuget package later to help you generate Modular Solutions for your upcoming project (Any thoughts? Leave your comments below!). But for now, let’s build a very basic implementation. Here is what you can expect.

Controller registration from other Class libraries
CQRS using MediatR
MSSQL
Migrations
Catalog Module
Customer Module
Shared DTOs
Architectural Assumptions
To keep things simple, we will assume that Entity Framework Core will be our default DB Abstraction provider and will go strong for another 10+ years. In this way, we can avoid the Repository pattern that usually tends to make our codebase larger.

Getting Started
Let’s start by creating a new Blank Solution in Visual Studio. PS, I will be using Visual Studio 2019 Community for this demonstration.

![image](https://user-images.githubusercontent.com/29863643/230743949-9c8d2542-da4e-4880-9863-84a3bd9d3dff.png)


Project Structure
Within the newly created solution, let’s create a new folder Host and add in an ASP.NET Core 5.0 WebAPI Application. Remove all the boilerplate code that comes along with the WebAPI.

![image](https://user-images.githubusercontent.com/29863643/230743960-2061c563-8fee-4e1e-b1d8-5d8ddb121580.png)


With that done, let me add a few other C# library projects. You can follow the similar folder structure as demonstrated in the screenshot below.

![image](https://user-images.githubusercontent.com/29863643/230744000-96206a9a-9c4d-4b5c-b055-a17426e437ba.png)


As mentioned,

API will hold all the service / controller registration logics and nothing else.
Module.Catalog & Module.People will contain the API controllers only,which will be picked up by the API Project.
Module.Catalog.Core & Module.People.Core will contain the Entity models, interfaces specific to the module, Mediatr Handlers and so on.
Module.Catalog.Infrastructure & Module.People.Infrastructure will mainly hold the module specific DBContext,Migrations, SeedData and Service implementation if any.
Shared.Core will have MediatR Behaviors, Common Service Implementations / Interfaces and basically everything that has to be shared across the application.
Shared.Models is where you will have to add in the Request /Response classes.Note that this project can be used for any C# Client applications as well.
Finally, Shared.Infrastructure is where you would want your middlewares, utilities and specify which Database Provider to use for the entire application.
With the structure ready, let’s add in the required extensions and controllers.


Controller Registrations
The first challenge is if you place the controllers in Module.Catalog & Module.People projects, how will the API project recognize it and add the routing required? Thus we need a way to make the API project use controllers that are in separate projects but uses the standard naming conventions of API Controllers.

Before all that, you would want to add the following to the Shared.Infrastructure project file to ensure that we have access to the AspNetCore Framework references and classes.

  <ItemGroup>
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
  </ItemGroup>
  
  ![image](https://user-images.githubusercontent.com/29863643/230744091-426a15b8-8dc9-4a55-9701-4f85a5f79b66.png)


Next, create a Controllers folder under Shared.Infrastructure and add a new class InternalControllerFeatureProvider.

internal class InternalControllerFeatureProvider : ControllerFeatureProvider
{
    protected override bool IsController(TypeInfo typeInfo)
    {
        if (!typeInfo.IsClass)
        {
            return false;
        }
        if (typeInfo.IsAbstract)
        {
            return false;
        }
        if (typeInfo.ContainsGenericParameters)
        {
            return false;
        }
        if (typeInfo.IsDefined(typeof(NonControllerAttribute)))
        {
            return false;
        }
        return typeInfo.Name.EndsWith("Controller", StringComparison.OrdinalIgnoreCase) ||
                typeInfo.IsDefined(typeof(ControllerAttribute));
    }
}



Now, this class will be responsible for adding controllers that are in different projects. We will have to register this class into the service container of our host ASP.NET Core application.

Create a new folder under Shared.Infrastructure, Extensions and add a new class ServiceCollectionExtensions.cs.


public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddSharedInfrastructure(this IServiceCollection services, IConfiguration config)
    {
        services.AddControllers()
            .ConfigureApplicationPartManager(manager =>
            {
                manager.FeatureProviders.Add(new InternalControllerFeatureProvider());
            });
        return services;
    }
}


Now, let’s navigate to the API project / Startup / ConfigureServices method and add the following.


public void ConfigureServices(IServiceCollection services)
{
    services.AddSharedInfrastructure(Configuration);
    services.AddSwaggerGen(c =>
    {
        c.SwaggerDoc("v1", new OpenApiInfo { Title = "API", Version = "v1" });
    });            
}

Make sure that the API Project has reference to the Shared Infrastructure, Module.Catalog, Module.People projects too. (Important)

![image](https://user-images.githubusercontent.com/29863643/230744244-3003e642-4261-45db-a598-81720efd0a32.png)


Let’s add a controller to our Modules.Catalog. Create a new folder Controllers under the Modules.Catalog project and add in a new Controller, BrandsController. We are just adding this controller to ensure that the API project is able to detect the controller in the Module project.


[ApiController]
[Route("/api/catalog/[controller]")]
internal class BrandsController : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> GetAllAsync()
    {
        return Ok();
    }
}
  
  IMPORTANT – You might be seeing a lot of unresolved dependencies now. It’s important to add the following project references for each module going forward.

Module.Catalog.Core should have a referece to Shared.Core
Module.Catalog.Infrastructure should have a referece to Shared.Infrastructure & Module.Catalog.Core
Module.Catalog should have a referece to Module.Catalog.Core and Module.Catalog.Infrastructure
Shared.Infrastructure should have a reference to Shared.Core
Shared.Core should depend on Shared.Models
  
  Make sure to add similar dependencies for the People Module as well.


  PS, these are few crucial dependencies stated by Clean Architecture principles (Onion). So make sure that you get them all right
  
  ![image](https://user-images.githubusercontent.com/29863643/230744315-0601c01f-68a4-4273-9868-d4ea9e15ef2f.png)

  
  With that done, let’s run the project and check if the BrandController comes up in Swagger.


  
  ![image](https://user-images.githubusercontent.com/29863643/230744335-343d4bfb-c1ea-4ef8-b6f8-bac6f8073fd9.png)

  
  So, that’s done. Now let’s connect the application to a database, in a modular way.


  
  Persistence
As mentioned earlier, we will be using Entity Framework Core as the DB Abstraction in this project.

Let’s get started by adding the Brand Model Entity. Open up the Modules.Catalog.Core and add a new folder, Entities. Here create a new class and name it Brand.


  
  public class Brand
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Detail { get; set; }
}
  
  Since we decided to add separate DBContexts for each module, it makes sense to add in a common DBContext first, then inherit it as the base class, yeah?

First, Navigate to Shared.Infrastructure Project and add a new Folder, Persistence. Here let’s add a new class named ModuleDbContext. Remember, this will be the base of all the DBContext classes that you will be creating moving forward in each and every module.
  
  
  public abstract class ModuleDbContext : DbContext
{
    protected abstract string Schema { get; }
    protected ModuleDbContext(DbContextOptions options) : base(options)
    {
    }
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        if (!string.IsNullOrWhiteSpace(Schema))
        {
            modelBuilder.HasDefaultSchema(Schema);
        }
        base.OnModelCreating(modelBuilder);
        modelBuilder.ApplyConfigurationsFromAssembly(GetType().Assembly);
    }
    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        return (await base.SaveChangesAsync(true, cancellationToken));
    }
}
  
  It’s important to note that we are using Schemas to make a logical separation between the database tables as well. For example, tables associated with Catalog module will be named Catalog.Brand , Catalog.Products and so on. You get the idea, yeah?

Next, let’s add the DBContext specific to this Module. Remember, no other modules can have access to the Brand Table other than the Catalog Module. This is made sure by creating separate DbContexts for each Module.

Navigate to Module.Catalog.Core and create a new folder Abstractions. Here is where you will have to place the interfaces to achieve Dependency Inversion. This is the whole essence of Onion Architecture, yeah? In this folder, let’s add a new interface and name it ICatalogDbContext.
  
  public interface ICatalogDbContext
{
    public DbSet<Brand> Brands { get; set; }
    Task<int> SaveChangesAsync(CancellationToken cancellationToken);
}
  
  Next, Navigate to Modules.Catalog.Infrastructure and add a new folder, Persistence. Here, add in a new class, CatalogDbContext which inherits from the catalog DB context interface and the module Dbcontext base class.

Note that anything related to the data access of the Catalog module will have to be put here.
  
  public class CatalogDbContext : ModuleDbContext, ICatalogDbContext
{
    protected override string Schema => "Catalog";
    public CatalogDbContext(DbContextOptions<CatalogDbContext> options): base(options)
    {
    }
    public DbSet<Brand> Brands { get; set; }
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
    }
}
  
  Note that we specify the schema name as Catalog here. Also, make sure to inherit from our common ModuleDbContext and the interface specific to our current module.

Let’s install the required packages now.

For Shared.Infrastructure project, install the following packages.
  
  
  Install-Package Microsoft.EntityFrameworkCore
Install-Package Microsoft.EntityFrameworkCore.Relational
Install-Package Microsoft.EntityFrameworkCore.SqlServer
Install-Package Microsoft.EntityFrameworkCore.Tools
Install-Package MediatR
  
  For the API Project, install the following.


  Install-Package Microsoft.EntityFrameworkCore.Design
  
  Now comes the interesting part of adding Database Provider extensions. We know that we will be using MSSQL as the DB provider in this implementation. But let’s build a somewhat flexible system, where it can be switched to PostgreSQL or other providers easily. Ideally, this solution should be present in a common project for other modules to use easily. That’s right, navigate to Shared.Infrastructure project and open up Extensions/ServiceCollectionExtensions.cs file. Here, add in the following Extension methods.
  
  public static IServiceCollection AddDatabaseContext<T>(this IServiceCollection services, IConfiguration config) where T : DbContext
{
    var connectionString = config.GetConnectionString("Default");
    services.AddMSSQL<T>(connectionString);
    return services;
}
private static IServiceCollection AddMSSQL<T>(this IServiceCollection services, string connectionString) where T : DbContext
{
    services.AddDbContext<T>(m => m.UseSqlServer(connectionString, e => e.MigrationsAssembly(typeof(T).Assembly.FullName)));
    using var scope = services.BuildServiceProvider().CreateScope();
    var dbContext = scope.ServiceProvider.GetRequiredService<T>();
    dbContext.Database.Migrate();
    return services;
}
  
  Line 3 – Get the connection string defined in the appsettings.json of the API project. Note that we will be adding the connection string in the next section.
Line 4 – Calls the extension method specific for MSSQL. You could write a new extension for other DB Providers as well. You get the idea, yeah?
Line 9 – Adds the passed DbContext to the service container using the MSSQL package of EFCore. Make sure that you have already installed it.
Line 12 – Updates the database using the latest available Migrations.
  
  Open up the appsettings.json of the API Project and add in the following.
  
  "ConnectionStrings": {
  "Default": "Data Source=(localdb)\\mssqllocaldb;Initial Catalog=monolithSample;Integrated Security=True;MultipleActiveResultSets=True"
}
  
  Next, we need to make sure that each of the modules uses these extensions. Navigate to Module.Catalog.Infrastructure and add a new Folder, Extensions. Here add a new static class, ServiceCollectionExtensions


  
  public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddCatalogInfrastructure(this IServiceCollection services, IConfiguration config)
    {
        services
            .AddDatabaseContext<CatalogDbContext>(config)
            .AddScoped<ICatalogDbContext>(provider => provider.GetService<CatalogDbContext>());
        return services;
    }
}
  Next, in the Module.Catalog.Core project, add an Extensions folder and add ServiceCollectionExtensions.cs. We will need this while implementing the MediatR handlers.


  
  public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddCatalogCore(this IServiceCollection services)
    {
        services.AddMediatR(Assembly.GetExecutingAssembly());
        return services;
    }
}
  
    Next, we need an extension for each module, that can be read by the API Project for registering the required services. Navigate to Module.Catalog and add a new class, ModuleExtensions. Here we will be adding the other extensions like AddCatalogCore and AddCatalogInfrastructure.


  
  public static class ModuleExtensions
{
    public static IServiceCollection AddCatalogModule(this IServiceCollection services, IConfiguration configuration)
    {
        services
            .AddCatalogCore()
            .AddCatalogInfrastructure(configuration);
        return services;
    }
}
  
  Finally, go to the API Project / Startup / ConfigureServices method and add in the following. Make sure that you have added the reference to Module.Catalog as well.


  
  services.AddCatalogModule(Configuration);
  That’s everything you need to do to set up Database Access in a Modular fashion. As the last step, let’s add the required migrations and check if the tables are created as expected.

On Visual Studio, Right-click the Module.Catalog.Infrastructure and click on ‘Open in Terminal’. Run the following command.

dotnet ef migrations add "initial" --startup-project ../API -o Persistence/Migrations/ --context CatalogDbContext
  
  ![image](https://user-images.githubusercontent.com/29863643/230744942-b3919335-ceab-4dcc-b937-4296ee5735a9.png)

  
  
  This would create the Migrations in the following folder.


  ![image](https://user-images.githubusercontent.com/29863643/230744948-e237a14a-a6b3-4e41-8e6b-a29351bf7f3a.png)

  
  As per our code, as soon as the application runs, the required tables will be created. Let’s test.


  ![image](https://user-images.githubusercontent.com/29863643/230744978-a127bc8e-0434-462b-8df6-b17a33b6a5ea.png)

  
There you go, the tables are created exactly as we wanted.

Adding MediatR Handlers and Controllers
As the final part of this implementation, we will be adding the required MediatR handlers and API controllers. To keep the article minimal, we will just be adding the ‘GetAll’ and ‘Register’ endpoints for the Brands Entity.

Let’s start by adding 2 new folders under the Module.Catalog.Core project and name it Queries & Commands respectively.

Under the Queries folder, add in a new class and name it GetAllBrandsQuery.

namespace Module.Catalog.Core.Queries.GetAll
{
    public class GetAllBrandsQuery : IRequest<IEnumerable<Brand>>
    {
    }
    internal class BrandQueryHandler : IRequestHandler<GetAllBrandsQuery, IEnumerable<Brand>>
    {
        private readonly ICatalogDbContext _context;
        public BrandQueryHandler(ICatalogDbContext context)
        {
            _context = context;
        }
        public async Task<IEnumerable<Brand>> Handle(GetAllBrandsQuery request, CancellationToken cancellationToken)
        {
            var brands = await _context.Brands.OrderBy(x => x.Id).ToListAsync();
            if (brands == null) throw new Exception("Brands Not Found!");
            return brands;
        }
    }
}
  
  Similarly, add a new class, RegisterBrandCommand under the Register folder.

namespace Module.Catalog.Core.Commands.Register
{
    public class RegisterBrandCommand : IRequest<int>
    {
        public string Name { get; set; }
        public string Detail { get; set; }
    }
    internal class BrandCommandHandler : IRequestHandler<RegisterBrandCommand, int>
    {
        private readonly ICatalogDbContext _context;
        public BrandCommandHandler(ICatalogDbContext context)
        {
            _context = context;
        }
        public async Task<int> Handle(RegisterBrandCommand command, CancellationToken cancellationToken)
        {
            if (await _context.Brands.AnyAsync(c => c.Name == command.Name, cancellationToken))
            {
                throw new Exception("Brand with the same name already exists.");
            }
            var brand = new Brand { Detail = command.Detail, Name = command.Name };
            await _context.Brands.AddAsync(brand, cancellationToken);
            await _context.SaveChangesAsync(cancellationToken);
            return brand.Id;
        }
    }
}
  
  Now that we have taken care of the Handlers, let’s wire them up with our BrandsController. Open up the BrandsController and add in the following.

namespace Module.Catalog.Controllers
{
    [ApiController]
    [Route("/api/catalog/[controller]")]
    internal class BrandsController : ControllerBase
    {
        private readonly IMediator _mediator;
        public BrandsController(IMediator mediator)
        {
            _mediator = mediator;
        }
        [HttpGet]
        public async Task<IActionResult> GetAllAsync()
        {
            var brands = await _mediator.Send(new GetAllBrandsQuery());
            return Ok(brands);
        }
        [HttpPost]
        public async Task<IActionResult> RegisterAsync(RegisterBrandCommand command)
        {
            return Ok(await _mediator.Send(command));
        }
    }
}
  
  That’s it for the implementation. Let’s run the application and verify the swagger endpoints.

The POST method allows you to add in new brands.


  
  That’s it for the implementation. Let’s run the application and verify the swagger endpoints.

The POST method allows you to add in new brands.


  
  And using the Get method, we can fetch all the brands records from the DB.


  
  ![image](https://user-images.githubusercontent.com/29863643/230745120-692008ef-e818-47cb-bbaf-8e94cde6e084.png)

  That’s it for the article. Do you want me to write another article to build upon this same solution and add extra infrastructure like Middlewares, Logging, and so on? Do let me know in the comments section. Modularizing application is definitely the way to go for a cleaner and scalable project.



