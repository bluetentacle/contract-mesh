- [The Contract Mesh: manageable microservices from the start](#the-contract-mesh-manageable-microservices-from-the-start)
- [Components](#components)
  - [Service contracts](#service-contracts)
  - [Service catalog](#service-catalog)
  - [Contract validator](#contract-validator)
  - [Contract editor (nice to have)](#contract-editor-nice-to-have)
- [Use cases](#use-cases)
  - [Developer portal](#developer-portal)
  - [Code generation](#code-generation)
  - [Serve as a design document](#serve-as-a-design-document)
  - [Help deploy an entire environment from scratch](#help-deploy-an-entire-environment-from-scratch)
  - [API gateway configuration](#api-gateway-configuration)
  - [Configure service mesh sidecars or resiliency behavior in code](#configure-service-mesh-sidecars-or-resiliency-behavior-in-code)

# The Contract Mesh: manageable microservices from the very start

The microservice pattern has many well-known benefits but also brings its own changes. As the number of microservices increase in a company, the following manageability issues surface:

- **Discoverability**: It's difficult for human actors to discover what services are already available, the capabilities of these services, and the different versions of these services.
- **Documentation**: There's no centralized documentation for these services that can act as a developer portal.
- **Dependency management**: It's difficult to determine the dependencies of each service, and versions of these dependencies.
- **Deployment**: It's difficult to deploy an entire environment from scratch, each service having its own unique and complex deployment procedures, and the order in which services must be deployed is hard to determine.
- **Testing**: due to the lack of a dependency graph, it's difficult to deploy and test any subset of services in a clean environment.

Many patterns have been invented to make microservices more manageable, such as service discovery, distributed tracing, API gateway, service mesh, etc. But by and large, these patterns solve problems encountered *post-deployment*, at *runtime*.

The reality is though, that the complexity of a microservice ecosystem has long started before anything is deployed, and that it's just as challenging to make sense of the source code and artifacts of microservices as deployed instances. No pattern currently exists to make the pre-deployment life cycle of microservices more manageable. The Contract Mesh is intended to fill this void.

# Components

The Contract Mesh consists of the following components:

## Service contracts

A service contract is a YAML document with a standard structure, which declares the features offered by a service, as well as the dependencies that it has on other services and resources.

Each microservice shall declare a contract in this standard format, residing in a standard location in the service's source code repository.

The contract is authored by the developer, *prior* to writing service code.

Specifically, the document contains:

- Basic metadata about the service, such as name, version, SCM tag, artifact location, etc. (The last three pieces of information can be filled by by the CI pipeline and shouldn't be specified by the author.)
- **Features** that the service provides to the outside world, including:
  - Schemas of business entities owned by the service and referenced by REST API, events, and other features
  - REST API definition, provided in the standard OpenAPI format, embedded inside the contract document
  - Event definition, including topics and body schemas
  - Metrics published by the service to a centralized telemetry platform
  - Log record schemas published by the service into a central logging platform
- **Dependencies** on other services. Each dependency specifies:
  - Required version range of the service
  - Specific abilities of the service that are utilized, such as REST API endpoints and events
  - Resiliency requirements for the dependency, such as the timeout, number of retries, circuit-breaker thresholds, etc.

A contract looks like the following:

```yaml
contractFormat: 1
info:
  category: dom
  name: orderService
  version: 1.2.3.456
  source: http://github.acme.com/business/orders-service/tree/v1.2.3.456
  artifacts:
    docker: docker.acme.com/orders-service:v1.2.3.456
features:
  entities:
    order:
      v1:
        required:
          - id
          - product-name
          - price
        properties:
          id:
            type: int,
            description: The unique identifier of this order
          productName:
            type: string,
            description: The name of the project being ordered
          price:
            type: float
            description: The price of this order
  apis:
    v1:
      openapi: 3.0.0
      paths:
        /orders/{orderId}:
          get:
            operationId: getOrderById
            x-api-gateway: true
            parameters:
              - name: orderId
                in: path
                schema:
                  type: integer
                  format: int64
            responses:
              200:
                content:
                  application/json
                    schema:
                      $ref: '#/entities/order/v1'
        /orders:
          post:
            operationId: postOrder
            requestBody:
              content:
                application/json
                  schema:
                    $ref: '#entities/order/v1'
            responses:
              200:
                content:
                  application/json:
                    schema:
                      properties:
                        id:
                          type: int
    v2: ...
  events:
    order.created
      v1:
        schema:
          $ref: '#/entities/order/v1'
  metrics:
    orderCount:
      type: counter
      dimensions:
        - state
        - result
      resolution: 10s
  logRecords:
    orderFailed:
      schema:
        productName:
          type: string
        userName:
          type: string
dependencies:
  productService:
    version: 1.3
    apis:
      getProductById:
        resiliency:
          timeout: 10s
          retries: 3
          circuitBreaker:
            threshold: 10
  pricingService: ...
```

## Service catalog

The service catalog is a microservice that aggregates all contracts from all services available in an organization.

The catalog is populated by continuous integration. Whenever a service is built, the CI pipeline publishes the artifacts (Docker images, binaries, Nuget packages, etc), as well as the service's contract into the catalog.

The catalog supports versioning. When CI publishes the contract to the catalog, it assigns the same version to the contract as the associated artifacts.

The catalog supports querying services in flexible ways, including:

- Returns a complete dependency graph of all available microservices, as wells 
- Given a service of a particular vesion, return a subset of dependency graph of the service and all of its direct and indirect dependencies
- Return a subset of a contract, such as only the REST API

The catalog itself comes with a service contract; we eat our own dog food!

## Contract validator

The contract validator is a CLI application that validates the correctness of the contract. The validator can be run by developers manually when authoring the contract. It could also be invoked by the CI pipeline prior to publishing, to ensure that no invalid schemas are . The validator ensures that:

- The contract syntax is correct, and required information is provided
- Breaking changes to the contract are not made without versioning
- Dependencies are correctly named and actually exist in the service catalog
- No circular depedencies are formed
- The contract conforms to company standards

## Contract editor (nice to have)

The contract editor is a GUI application that makes it easy to write correct contracts. Alternatively, the contract can be written with any text editor with the help of the validator.

# Use cases

The Contract Mesh is a concept that kills many birds with one stone. It enables many powerful possibilities. They include:

## Developer portal

The Service Catalog provides complete metadata on all available services, such as REST APIs, event schemas, log records, and metrics. A Developer portal can be written that exposes this information in an highly human-readable format. Such a portal would not need any manual mantenance.

## Code generation

The contract can be used to automatically generate server-side code that implements the REST APIs, event schemas, log records, and metrics described in the contract. A reusable services framework can help ensure that the contract is strictly obeyed by code.

Using the dependencies described in the contract, and by querying the service catalog for details about the dependencies, *client-side* code can also be generated that calls other services' APIs and consume other services' events.

## Serve as a design document

The contract, produced before any code is written, can serve as a wonderful of a service design.

## Help deploy an entire environment from scratch

The contract mesh fully describe the dependencies between services and this allows us to easily create an environment from scratch. It's possible to both deploy the entire microservice ecosystem, or a subset of services.

Fixed testing environments will be a thing of the past. Instead of dedicated DEV, QA, INT environments that are constantly running, the contract mesh allows us to create a test environment for the duration of a test run, and tear it down when we are done. These environments can be used for both manual testing and automated testing.

## API gateway configuration

API gateways is an essential part of any microservice architecture. However, the gateway's configuration is usually separate from the deployment of services, thus adding to deployment complexity.

The service contract can be used to specify the API gateway configuration for REST endpoints provided by a service. By adding a custom field to the OpenAPI document such as `x-api-gateway`, the contract author can decide which endpoints to expose to the gateway:

```yaml
...
/orders/{orderId}:
  get:
    operationId: getOrderById
    x-api-gateway: true
...
```

When the service is deployed to an environment, the API gateway in that environment can use service discovery to obtain a list of all services and retrieve their contracts. It looks for those endpoints with gateway exposure enabled, and configure itself accordingly.

## Configure service mesh sidecars or resiliency behavior in code

Sidecars are a definitive feature of a service mesh that provides invocation resiliency between services. Just like the API Gateway, naive implementations of service mesh requires it to be configured separately from services themselves, adding to deployment complexity and management challenges.

The service contract is a natural location for sidecar configuration. During deployment, when the service is being deployed, the deployment pipeline can query the service's contract and configure the sidecar accordingly:

```yaml
...
dependencies:
  productService:
    version: 1.3
    apis:
      getProductById:
        resiliency:
          timeout: 10s
          retries: 3
          circuitBreaker:
            threshold: 10
...
```