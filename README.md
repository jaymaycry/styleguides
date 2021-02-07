# Graphql API Styleguide - Apollo, Nexus, Prisma

## Structure

Graphql-Apis should ideally be structured in three layers. The Resolver-Layer, the Service-Layer and the Model-Layer, as shown in the picture below.

![alt text](/images/api.png "Logo Title Text 1")

This can be achieved by a folder structure like this:

```
api/
  modules/
    user/
      service/
        userService.ts        // User service
        userService.spec.ts
      query.ts                // query resolvers and object type definitions
      mutation.ts             // mutation resolvers and input type definitions
      validation.ts           // validation schemas (Joi)
  permissions/
    query.ts                  // query permissions
    mutation.ts               // mutation permissions
  utils/
    validation.ts             // validation helper methods
```

## Services

Every entity in the database that can be accessed through the api has its own service, f.e. the `UserService`. There might be additional services like an `AuthService` or services to communicate with 3rd party apis. Services export themselfs as a singleton and communicate with other services through depdency injection.

```js
export class UserService {
  private prisma: PrismaClient;

  constructor({ prisma }) {
    this.prisma = prisma;
    // add additional services here
  }

  private getHashedPassword(password: string) {
    return hash(password, 10);
  }


  async create(data: Prisma.UserCreateInput) {
    // enter business logic here
    const { password, ...rest } = data;
    const hashedPassword = await this.getHashedPassword(password);

    return this.prisma.user.create({
      data: {
        ...rest,
        password: hashedPassword,
      },
    });
  }

  findAll(params: {
    skip?: number;
    take?: number;
    cursor?: Prisma.UserWhereUniqueInput;
    where?: Prisma.UserWhereInput;
    orderBy?: Prisma.UserOrderByInput;
  }) {
    const { skip, take, cursor, where, orderBy } = params;
    return this.prisma.user.findMany(params);
  }

  findOne(where: Prisma.UserWhereUniqueInput) {
    return this.prisma.game.findUnique({
      where,
    });
  }

  update(params: {
    where: Prisma.UserWhereUniqueInput;
    data: Prisma.UserUpdateInput;
  }) {
    // enter business logic here
    const { where, data } = params;
    const newPassword = data.password
      ? await this.getHashedPassword(data.password as string)
      : undefined;

    return this.prisma.user.update({
      data: {
        ...data,
        password: newPassword,
      },
      where,
    });
  }

  remove(where: Prisma.UserWhereUniqueInput) {
    // enter business logic here
    return this.prisma.game.delete({
      where,
    });
  }
}

export default new UserService({ prisma });
```

This is a base class for the user entity that handels creation and update of users. The prisma service gets injected as a dependency. Business logic in this case is the handling of hashing passwords before inserting it into the database. There could be other things like triggering an emails, creation of other entities (and therefore communicating with other injected services as well) or cleanup on deletion. This can be tested in an easy way by mocking the injected services and testing all the public methods.

## Queries

### Nodes

A node is every entity which has an id field. It should be an interface and implemented by every type in the app. It's a way of defining common fields like id, createdAt updatedAt or maybe an `active` flag in case one wants to use soft delete.

```js
// nexusjs approach
const Node = interfaceType({
  name: "Node",
  definition(t) {
    t.id("id", { description: "GUID for a resource" });
    t.string("createdAt");
    t.string("updatedAt");
    t.boolean("active"); // soft delete
  },
});

const User = objectType({
  name: "User",
  definition(t) {
    t.implements("Node");
    t.model.email();
    t.model.firstName();
    t.model.lastName();
    // ...
  },
});
```

### Me / Viewer

The logged in user is part of the root query object and should be queriable as "me" or "viewer". If the viewer has specific fields that relates to his active session it's recommended to create a `Viewer` type just for him.

```js
// nexusjs approach
const Profile = interfaceType({
  name: "Profile",
  definition(t) {
    t.string("email");
    t.string("name");
  },
});
const User = objectType({
  name: "User",
  definition(t) {
    t.implements("Node");
    t.implements("Profile");
  },
});

const Viewer = objectType({
  name: "Viewer",
  definition(t) {
    t.implements("Node");
    t.implements("Profile");
    // additional fields like stuff only the user can query etc.
    t.field("projects", {
      resolver: () {
        // custom resolver that loads ones projects
        // should only be accessible by the viewer
        // ...
      }
    });
  },
});
```

### Root Query Object

The root query object should only contain stuff that is accessible by every user. Like a search or

## Mutations

### Results

Mutations should return objects containing the entities they edited. That way the cache can easily be updated when something changes on the backend. Also if you return an object rather than the entity itself you can later easily extend the result with further nodes that might interest the api user.

## Input validation

## Security

### Authentication

Authentication should be implemented by using a jwt token that comes with each request and can be used to load a user into the `context` of a request.

```js
const { ApolloServer } = require("apollo-server");

const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: ({ req }) => {
    // get the user token from the headers
    const token = req.headers.authorization || "";

    // try to retrieve a user with the token
    const user = getUser(token);

    // add the user to the context
    return { user };
  },
});
```

### Authorization

### Protection from malicious Queries

```gql
query maliciousQuery {
  thread(id: "some-id") {
    messages(first: 99999) {
      thread {
        messages(first: 99999) {
          thread {
            messages(first: 99999) {
              thread {
                # ...repeat times 10000...
              }
            }
          }
        }
      }
    }
  }
}
```

Restrict the query depth by using https://github.com/stems/graphql-depth-limit

```js
app.use(
  "/api",
  graphqlServer({
    validationRules: [depthLimit(10)],
  })
);
```

## Resources
