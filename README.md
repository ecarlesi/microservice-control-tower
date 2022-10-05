# Microservice Control Tower (MCT)

This project starts from the need to make architectures based on microservices more dynamic.

Below I illustrate some of the constraints that this project wants to solve:
- Clients don't need to know the endpoints of the services they need to consume, as long as they know the name.
- I can install multiple instances of the same microservice to improve the performance of my solution or to ensure a better level of service in terms of uptime.
- The permissions to access a microservice must be modifiable at runtime and their modification must not require deploy or reboot.

### Definitions
**Farm**: represents a set of microservices managed by a **MCT** installation. 

**Instance**: it is a microservice implementation. Multiple instances of the same implementation can be present in parallel in a **farm**, this to allow clients to have alternatives in case of problems in using an instance. In the event of an error, clients may decide to use another instance. Having multiple instances of the same implementation also allows you to implement charge balancing logic without having to act on the infrastructure.

**Environment**: the environment identifies an instance that differentiates itself from other instances that are the same at the implementation level but different from the point of view of the configuration. For example, we can have the same service distributed in different instances with different configurations useful for defining the development, testing and production environments. Another possible scenario is that in which a solution manages different tenants that perhaps isolate the data at the database level and therefore only have different database configuration to use.

**Identity provider**: it is a service enabled to delivery a JWT token trusted by the **MCT** and usable by clients and microservice instances to authorize the comunication. When an instance registers with a **MCT**, it agrees to use the same identity provider used by the **MCT**.

## Farm example
<img width="1158" alt="image" src="https://user-images.githubusercontent.com/195652/190275949-db83f2c9-e46d-4037-8474-80afa6ca4241.png">

## Operation diagram
- The instance starts and need to register itself in **MCT**. Af first it invokes the **Identities** resource to find out which are the trusted identity providers for the **MCT**. In response, it gets the list of endpoints that can issue a valid JWT token for the **MCT**.
- The instance invokes one of the identity provider it received from the **Identities** call made to the **MCT** previously and obtains a valid JWT token to invoke the **MCT** to register itself.
- The instance, using the JWT token, invoke the resource **Register** to register itself. In the response the instance obtains its configuration (including the authorization required to access the service). After this call this instance will be available in the farm. 
- The client needs to invoke the "calculator" service to perform a calculation. The client does not know which microservices implement this functionality or where they are located, but it knows the URL to access the **MCT**. First, it makes a call to the **Identities** resource to find out which are the trusted identity providers for the **MCT**. In response, it gets the list of endpoints that can issue a valid JWT token for the **MCT**.
- The client invokes one of the identity provider it received from the **Identities** call made to the **MCT** previously and obtains a valid JWT token to invoke the **MCT** to resolve the instance name. The response from **MCT** will contains the list of instance URL that implement the service "calculator".
- The client now knows how to invoke the microservice it need.

## MCT goals
1. The service should be as light as possible. Only the functionality that is really needed will need to be implemented.
1. The persistence layer will need to be configurable so that new persistence layers can be easily developed to use different databases or storage in general.


## Resources
### Identities
This resource returns the list of service providers trusted by the **MCT** service. In order to register, instances will need to authenticate with a token issued by one of these services.

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
    "environment": "production",
    "service":
    {
        "name": "calculator",
        "version": "1.0"
    },
    "url": "https://192.168.1.100:5033"
}
```

**Response payload**
```json
{
    "settings": 
    [
        {
            "key": "a",
            "value": "value a"
        },
        {
            "key": "b",
            "value": "value b"
        }
    ],
    "authorizations": 
    [
        {
            "role": "admins",
            "actions": 
            [
                {
                    "action": "read",
                    "object": "database-x"
                },
                {
                    "action": "read",
                    "object": "database-y"
                },
                {
                    "action": "write",
                    "object": "database-y"
                }
            ]
        }
    ]
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

**Response payload**
```json
{
    "settings": 
    [
        {
            "key": "a",
            "value": "value a"
        },
        {
            "key": "b",
            "value": "value b"
        }
    ],
    "authorizations": 
    [
        {
            "role": "admins",
            "actions": 
            [
                {
                    "action": "read",
                    "object": "database-x"
                },
                {
                    "action": "read",
                    "object": "database-y"
                },
                {
                    "action": "write",
                    "object": "database-y"
                }
            ]
        },
        {
            "role": "sys-users",
            "actions": 
            [
                {
                    "action": "read",
                    "object": "database-x"
                },
                {
                    "action": "read",
                    "object": "database-y"
                }
            ]
        }
    ]
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
        "environment": "production",
        "service":
        {
            "name": "calculator",
            "version": "1.0"
        }
    }
}
```

**Response payload**
```json
{
    "query":
    {
        "environment": "production",
        "service":
        {
            "name": "calculator",
            "version": "1.0"
        }
    },
    "endpoints":
    [
        {
            "instance": "AS43-3343",
            "url": "https://192.168.1.100:5033"
        },
        {
            "instance": "AS43-3663",
            "url": "https://192.168.1.105:1099"
        },
        {
            "instance": "AS43-3663",
            "url": "https://192.168.1.178:4533"
        }
    ]
}
```

### Do you want to help this project?
We need help from developers, documentation writers and graphic designers. If you want to collaborate send me a message :)
