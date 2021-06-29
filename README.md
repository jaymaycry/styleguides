# Graphql API Styleguide - Apollo, Nexus, Prisma

## Structure

Graphql-Apis should be structured in three layers. The resolver-layer, the service-layer and the model-layer, as shown in the picture below.

![alt text](/images/api.png "Logo Title Text 1")

This can be achieved by a folder structure like this:

```
api/
  services/
    userService.ts            // User service
  graphql/
    modules/
      user/
        index.ts
        query.ts              // query resolvers and object type definitions
        mutation.ts           // mutation resolvers and input type definitions
    permissions/
      index.ts
      query.ts                // query permissions
      mutation.ts             // mutation permissions
```

## Services

Every entity in the database that can be accessed through the api has its own service, f.e. the `UserService`. There might be additional services like an `AuthService` or services to communicate with 3rd party apis. Services export themselves as a singleton and communicate with other services through dependency injection.

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
    // business logic here
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
    // business logic here
    return this.prisma.game.delete({
      where,
    });
  }
}

export default new UserService({ prisma });
```

This is a base class for the user entity that handles crud operations of users. The prisma service gets injected as a dependency. Business logic in this case is the handling of hashing passwords before inserting it into the database. There could be other things like triggering an e-mail, creation of other entities (and therefore communicating with other injected services as well) or cleanup on deletion. This can be tested in an easy way by mocking the injected services and testing all the public methods.

## Queries

### Abstract types

Graphql offers the ability to specify abstract types to share a set of fields between object types. It requires some time though to think of possible use cases but in general it's good practice to use them for larger applications. F.e. a github `User` type implements the following abstract types (besides others):

| Type         | Description                                                                     | Also implemented by               |
| ------------ | ------------------------------------------------------------------------------- | --------------------------------- |
| Node         | Things that all nodes share in common (see chapter below)                       | All objects                       |
| Actor        | avatarUrl, username, resourcePath (profile url)                                 | EnterpriseUser, Organization, Bot |
| ProjectOwner | a projects list, a boolean which says if the viewer/me can create projects here | Organization, Repository          |

### Nodes

A node is every entity which has an id field. It should be an interface and implemented by every type in the app.

```gql
type Node {
  id: ID!
}

type User implements Node {
  email: String!
  firstName: String
  lastName: String
}
```

As Nexus Example:

```js
// nexusjs approach
const Node = interfaceType({
  name: "Node",
  definition(t) {
    t.id("id", { description: "GUID for a resource" });
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

The logged in user is part of the root query object and should be queryable as "me" or "viewer". It's in general of type `User`, but can be created as a separate type if it has additional fields that only the active user can query.

```gql
type User implements Node & ProfileOwner {
}
```

```js
// nexusjs approach
const ProfileOwner = interfaceType({
  name: "ProfileOwner",
  definition(t) {
    t.string("email");
    t.string("name");
  },
});
const User = objectType({
  name: "User",
  definition(t) {
    t.implements("Node");
    t.implements("ProfileOwner");
  },
});

export const Query = extendType({
  type: "Query",
  definition(t) {
    t.field("me", {
      nullable: true,
      type: User,
      resolve: (root, args, context) => {
        return context.user;
      },
    });
  },
});
```

### Root Query Type

The root query object should only contain queries that are usable by every user. The resolvers there should be written in a way that it hides information from the user that he's not allowed to see, either by implementing queries accordingly or by filtering stuff that's not supposed to be found through authorization. In case there's something that only administrators can do, it should be prefixed with `admin` to mark it clearly for the api user.

### Pagination

TODO

## Mutations

### User Related Mutations

Some mutations may be related to the user, that is calling them. For better overview it's important to name them accordingly, a bit like the "me" field on Query.

For example having those two mutations:

```gql
type Mutation {
  editUser(userId: string): UserUpdateResult
  assignTicket(userId: string, ticketId: string): AssignTicketResult
}
```

If the usecase is that a User only needs to update himself or only assign tickets to himself it could look like this:

```gql
type Mutation {
  editMe(): UserUpdateResult
  assignTicketToMe(tickedId: string): AssignTicketResult
}
```

And the user is resolved from the context of the query. No need to check permissions (can the user calling this do that?).

### Results

Mutations should return objects containing the entities they edited. That way the cache can easily be updated when something changes on the backend. Also if you return an object rather than the entity itself you can later easily extend the result with further nodes that might interest the api user.

```graphql
type LoginResult {
  user: User!
  token: String!
}
```

## Input validation

Input validation is part of the resolver logic as it depends on the input args. You could provide input validation in the service but the resolver should then map those errors to the input args.

## Security

### Authentication

Authentication should be implemented by using a token (f.e. JWT) that comes with each request and can be used to load a user into the `context` of a request.

```js
const { ApolloServer } = require("apollo-server");

const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: ({ req }) => {
    // get the user token from the headers
    const token = ctx.req.headers.authorization || ctx.req.cookies?.token;

    // try to retrieve a user with the token
    const user = getUser(token);

    // add the user to the context
    return { user };
  },
});
```

### Authorization

Authorization can be implemented globally, per type or per field. Even for input args there's a way to apply permission checks. It's therefore necessary to differentiate between query permissions and mutation permissions.

https://docs.gitlab.com/ee/development/api_graphql_styleguide.html#authorization

#### Field Authorization

Ideally you specify permissions for each field of each Type. That way you can prevent data leaks and have full control over the output.

```js
{
  Query: {
    "*": isAdmin,                       // check for admin role
    me: allow,                          // allow rule for any user (incl. unauthenticated)
    client: isAuthenticated,            // logged in users
    building: isAuthenticated,
    partner: race(isAdmin, isPartner),  // check for admin or partner role
  },
  LoginResult: {
    "*": allow,
  },
  User: {
    id: race(isAdmin, isPartner, isMe), // check if admin or partner or if it's my user
    firstname: race(isAdmin, isPartner, isMe),
    lastname: race(isAdmin, isPartner, isMe)
    ...
  }
}
```

#### Type Authorization

Another way would be to define authorization rules on types. This saves some lines of code but may leave the door open to unintended data access when adding new resolvers to a type.

```js
{
  User: isMe, // check if it's my user
}
```

#### Mutation Authorization

Mutation authorization depends on the context and input args. For some cases it's enough to check if the user is authenticated or if he has a certain role. But sometimes you need to check the input arguments, f.e. when referencing another entity. Then you may need to check this reference if the user is allowed to use it.

```js
{
  Mutation: {
    "*": isAdmin,
    login: allow,
    logout: allow,
    updateRoom: isRoomOwner, // depends on user and referenced roomId in the args
  }
}
```

### Protection from malicious queries

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

https://graphql.org/learn/best-practices
https://docs.gitlab.com/ee/development/api_graphql_styleguide.html
https://docs.gitlab.com/ee/development/api_graphql_styleguide.html#authorization

## Example APIS

https://gitlab.com/-/graphql-explorer
https://docs.github.com/en/graphql/overview/explorer

## NEXTJS

Pass on cookies for initial render

```js
setContext(() => {
  const headers: {
    "accept-language"?: string;
    authorization?: string;
  } = {
    ["accept-language"]: i18n.language,
  };

  // add auth header for ssr executed
  // queries to pass the cookie token
  const token = (ctx?.req as any)?.cookies?.token;
  if (!process.browser && token) {
    headers.authorization = token;
  }

  return {
    headers,
  };
}) as any,
```
