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




