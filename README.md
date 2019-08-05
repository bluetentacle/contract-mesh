# The Contract Mesh

The microservice pattern has many well-known benefits but also brings its own changes. As the number of microservices increase in a company, the following manageability issues surface:

- **Discoverability**: It's difficult for human actors to discover what services are already available, the capabilities of these services, and the different versions of these services.
- **Documentation**: There's no centralized documentation for these services that can act as a developer portal.
- **Dependency management**: It's difficult to determine the dependencies of each service, and versions of these dependencies.
- **Deployment**: It's difficult to deploy an entire environment from scratch, each service having its own unique and complex deployment procedures, and the order in which services must be deployed is hard to determine.
- **Testing**: due to the lack of a dependency graph, it's difficult to deploy and test any subset of services in a clean environment.

Many patterns have been invented to make microservices more manageable, such as service discovery, distributed tracing, service mesh, etc. But by and large, these patterns solved problems encountered *post-deployment*, at *runtime*.

No pattern currently exists to make the web of microservices more manageable prior to deployment. The Contract Mesh is intended to fill this void.

# Components

The Contract Mesh consists of the following components:

## Service contracts

A service contract is a YAML document with a standard structure, which declares the capabilities offers by a service, as well as the dependencies that it has other other services and resources.

Each microservice shall have a contract in this standard format, residing in a standard location in the source code repository.

The contract is authored by the developer, *prior* to writing service code.

The contract looks like the following:

```yaml
info:
  ...
apis:
  ...
events:
  ...
metrics:
  ...
log-records:
  ...
dependencies:
  service-b:
    version: 1.3
    apis:
      ...
  service-c:
    version: 4.2
    events:
      ...
```

Specifically, the document contains:

- Basic metadata about the service, such as name, version, source code location, artifact location, etc. A lot of this information can be automatically 
- Abilities that the service provides to the outside world, including:
  - REST API definition, provided in the standard OpenAPI format.
  - Event definition, including topics and body schemas
  - Metrics published by the service to a centralized telemetry platform
  - Log record schemas published by the service into a central logging platform
- Dependencies on other services. Each dependency specifies
  - Required version range of the service 
  - Specific abilities of the service that are utilized, such as REST API endpoints and events
  - Resiliency requirements for the dependency, such as the timeout, number of retries, circuit-breaker thresholds, etc.

## Service catalog

The service catalog is a microservice that aggregates all contracts from all services available in an organizatoin.

The catalog is populated by continuous integration. Whenever a service is built, the CI pipeline publishes the artifacts (Docker images, binaries, Nuget packages, etc), as well as the service's contract into the catalog.

The catalog supports versioning. When CI publishes the contract to the catalog, it assigns the same version to the contract as the associated artifacts.

The catalog supports querying services in flexible ways, including:

- Return all available services and available versions of each
- Given a service of a particular vesion, return all of its direct and indirect dependencies, along with their versions

## Contract validator

The contract validator is a CLI application that validates the correctness of the contract. The validator is run both by developers manually, and automatically by the CI pipeline prior to publishing. The validator ensures that:

- The contract syntax is correct, and required information is provided
- Breaking changes to the contract are not made without versioning
- Dependencies are correctly named and actually exist in the service catalog
- No circular depedencies are formed

## Contract editor (nice to have)

The contract editor is a GUI application that makes it easy to write correct contracts. Alternatively, the contract can be written with any text editor with the help of the validator.

# Use cases

The Contract Mesh enables many powerful possibilities. They include:

## Developer portal

The Service Catalog provides complete metadata on all available services, such as REST APIs, event schemas, log records, and metrics. A Developer portal can be written that exposes this information in an highly human-readable format. Such a portal would not need any manual mantenance.

## Code generation

The contract can be used to generate server-code that implements the REST APIs, event schemas, log records, and metrics described in the contract. A reusable services framework can help ensure that the contract is strictly obeyed by code.

Using the dependencies described in the contract, *client-side* code can also be generated that calls other services' APIs and consume other services' events.

## Serve as a design document

## Help deploy an entire environment from scratch

The contract mesh fully describe the dependencies between services and this allows us to easily create an environment from scratch. It's possible to both deploy the entire microservice ecosystem, or a subset of services.

Fixed testing environments will be a thing of the past. Instead of dedicated DEV, QA, INT environments that are constantly running, the contract mesh allows us to create a test environment for the duration of a test run, and tear it down when we are done. These environments can be used both for manual testing and for automated testing.

## API gateway configuration

## Configure service mesh sidecars or resiliency behavior in code
