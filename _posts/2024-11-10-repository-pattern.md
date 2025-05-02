---
title: "Design Patterns: Repository in Node.js with TypeScript"
description: >-
  The repository pattern is one of the most used design patterns related to databases and storage in general, it’s a good pattern because it provides an abstraction layer between the application’s data access logic and the underlying data source. <br />
  In this post, let's dive into this pattern implementation with a few Typescript and Node.js code examples.

author: carlos
date: 2024-08-18
categories: [Design Patterns]
tags: [node.js, typescript, backend]
pin: true
image: /assets/img/typescript.jpg
---

## Main Benefits Of the Repository Pattern

### Separation of Concerns
The Repository Pattern separates the access data logic from the business logic, which makes the code more modular and easier to maintain. With this separation, we can focus on the business logic without worrying about data access details.

### Testability
By abstracting data access behind repositories, it is easy to write tests for the business logic, because we can create a mock implementation for the data access, allowing you to test the business logic independently.

### Reusability
Repositories promote the reuse of data access code across different application parts. Since the data access operations are condensed in repository classes, consistency in how data is manipulated and retrieved is ensured.

## TypeScript Implementation
I created a simple client registration Node.js REST API using some Clean Architecture concepts, dividing the application into layers so we can have a separation of concerns, keeping our application easier to understand.

### Create Client Use Case (`create-client.ts`)
```typescript
import IClientRepository from "../db/repositories/client-repository";
import Client from "../entities/client";

type Input = {
    name: string;
    email: string;
    address: string;
    phone: string;
};

export default class CreateClient {
    constructor(readonly client_repository: IClientRepository) {}

    async execute(input: Input): Promise<boolean> {
        const client = new Client(null, input.name, input.email, input.address, input.phone);  
        await this.client_repository.save(client);
        return true;
    }
}
```
As we can see, there is a use case class called `CreateClient` which is only responsible for handling the create client logic. In this case, we are instantiating an Entity called `Client` and inserting this created object into our repository.

### Client Entity (`client.ts`)
```typescript
import crypto from 'crypto';

export default class Client {
    readonly id: string;
    readonly name: string;
    readonly email: string;
    readonly address: string;
    readonly phone: string;

    constructor(id: string | null, name: string, email: string, address: string, phone: string) {
        this.id = id ? id : crypto.randomUUID();
        if (name.length < 1) {
            throw new Error('Invalid name');
        }
        if (!email.includes('@') || !email.includes('.')) {
            throw new Error('Invalid email');
        }       
        if (phone.length < 8 || phone.length > 9) {
            throw new Error('Invalid phone');
        }
        this.email = email;        
        this.address = address;
        this.name = name;
        this.phone = phone;
    }
}
```
This class also generates an ID if it's instantiated with a null argument. This is a good practice because it decouples our application from a specific database for ID generation.

## Repository Implementation
### Interface (`client-repository.d.ts`)
```typescript
export interface IClientRepository {
    save(client: Client): Promise<void>;
    findByEmail(email: string): Promise<Client | null>;
}
```

### PostgreSQL Implementation (`client-repository-postgres.ts`)
```typescript
export default class ClientRepository implements IClientRepository {   
    async save(client: Client): Promise<void> {
        // ORM or SQL query to save client
        throw new Error('Method not implemented.');
    }
    async findByEmail(email: string): Promise<Client | null> {
        // ORM or SQL query to find client by email
        throw new Error('Method not implemented.');
    }
}
```

### In-Memory Implementation (`client-repository-memory.ts`)
```typescript
export default class ClientRepositoryMemory implements IClientRepository {
    private clients: Client[];

    constructor() {
        this.clients = [];
    }

    async save(client: Client): Promise<void> {
        this.clients.push(client);
    }

    async findByEmail(email: string): Promise<Client | null> {
        const client = this.clients.find(client => client.email === email);
        return client ? client : null;
    }
}
```
To apply the repository pattern we must create an interface so we can have different repository implementations.

The `ClientRepositoryMemory` class stores all the data in local memory. This is useful for testing because we can inject this implementation into our use case, avoiding complex mocks and external dependencies.

The `ClientRepository` class is the actual database repository implementation, handling all necessary queries. The `save` method ensures that only valid `Client` instances are passed, while the `findByEmail` method guarantees that a valid client object is always returned.

## Conclusion
The Repository Pattern is really good because we can keep our application cohesive and decoupled. However, if your business doesn’t require this level of decoupling due to a simple business logic, this pattern might not be the best option due to its complexity. In big applications with complex business logic, though, it might be a great fit.

**GitHub Repository with the Code Example:** [repository-pattern-example](https://github.com/carlosfmvaz/repository-pattern-example)

