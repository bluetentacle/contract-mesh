# The Contract Mesh

The microservice pattern has many well-known benefits but also brings its own changes. As the number of microservices increase in a company, the following manageability issues surface:

- **Discoverability**: It's difficult for human actors to discover what services are available, and the capabilities of these services.
- **Dependency management**: It's difficult to determine the dependencies of each service, and versions of these dependencies.
- **Deployment**: It's difficult to deploy an entire environment from scratch, each service having its own unique and complex deployment procedures, and the order in which services must be deployed is hard to determine.
- **Testing**: due to the lack of a dependency graph, it's difficult to deploy and test any subset of services in a clean environment.
- ****

Many patterns have been invented to manage microservices, but by and large, these patterns address the problems of microservices 
