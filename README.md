# Microservice Instance Locator

This project wants to solve a problem that I have faced several times during the development of microservices-based solutions: how to avoid having the service endpoints written in the code of the client components? The idea behind this project is that a client does not need to know where a service they need is located, knowing the name of the service they can get the details of how to contact it at runtime.

<img width="1158" alt="image" src="https://user-images.githubusercontent.com/195652/190275949-db83f2c9-e46d-4037-8474-80afa6ca4241.png">

### Definitions
**Farm**: represents a set of microservices managed by a Microservice Instance Locator installation. 

**Instance**: it is a microservice implementation. multiple instances of the same implementation can be present in parallel in a **farm**, this to allow clients to have alternatives in case of problems in using an instance. In the event of an error, clients may decide to use another instance. Having multiple instances of the same implementation also allows you to implement charge balancing logic without having to act on the infrastructure.

**Environment**: the environment identifies an instance that differentiates itself from other instances that are the same at the implementation level but different from the point of view of the configuration. For example, we can have the same service distributed in different instances with different configurations useful for defining the development, testing and production environments. Another possible scenario is that in which a solution manages different tenants that perhaps isolate the data at the database level and therefore only have different database configuration to use.

## Project goals
1. The service should be as light as possible. Only the functionality that is really needed will need to be implemented.
1. The persistence layer will need to be configurable so that new persistence layers can be easily developed for different databases or storage in general.


## Resources
### Register
This resource allows a microservice instance to register itself and become resolvable by clients.

instance: instance name, it must be unique in the farm. it should be automatically generated when the service is started on the host
environment: reference environment of the instance
service.name: service name, this information is known to the client who uses it to query and get the endpoints where the service instances are available
service.version: service version
**Request payload**
```json
{
    "instance": "AS43-3343",
    "environment": "dev",
    "service":
    {
        "name": "calculator",
        "version": "alpha 3"
    },
    "url": "https://192.168.1.12:5033"
}
```

### Confirm
This resource allows an instance to confirm its availability. When invoked updates a timestamp useful to the **Resolve** resource to select and sort the endpoints that are returned in the response. If a service has a confirm time too far in time, it is put at the bottom of the list or can be excluded from it if it exceeds a certain limit.
**Request payload**
```json
{
    "instance": "AS43-3343"
}
```

### Resolve
This resource allows clients to resolve a service name by getting a list of endpoints that implement the service.
The request contains the name of the service and context information useful for the server to select the appropriate services to return to the client. The response contains the list of endpoints that implement the service.
For each endpoint, useful information is provided to be able to invoke it. **TODO define authentication and authorization**
**Request payload**
```json
{
    "context": 
    {
        "device": 
        {
            "name": "computer 1"
        },
        "user":
        {
            "username": "eca",
            "domain": "local",
            "roles": 
            [
                "admin",
                "backup"
            ]
        }
    },
    "query":
    {
        "environment": "dev",
        "service":
        {
            "name": "calculator",
            "version": "alpha 3"
        }
    }
}
```

**Response payload**
```json
{
    "query":
    {
        "environment": "dev",
        "service":
        {
            "name": "calculator",
            "version": "alpha 3"
        }
    },
    "endpoints":
    [
        {
            "instance": "AS43-3343",
            "url": "https://192.168.1.12:5033"
        },
        {
            "instance": "AS43-3663",
            "url": "https://192.168.1.200:1099"
        }
    ]
}
```

### Do you want to help this project?
We need help from developers, documentation writers and graphic designers. If you want to collaborate send me a message :)
