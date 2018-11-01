# Mongo-Connector-Preview

This is a repo for an early preview of the Mongo Connector.
This is a very early prototype that is mainly intended to test a new concept we are introducing with the Mongo connector - embedded types. Normal relations have now been added.

**How to use**

The latest functionality for the Mongo Connector can always be found on the alpha branch. The CLI does not yet know about the Mongo Connector though, so some changes need to be done to the `docker-compose.yml` manually. We'll merge new functionality regularly into the alpha and will adjust the list of known limitations accordingly.

- install the beta version of Prisma cli via `npm install -g prisma@beta`
- run `prisma init` - choose new local db with mysql
- switch out the auto-generated docker-compose.yml with the following one

```yml
    version: '3'
    services:
      prisma:
        image: prismagraphql/prisma:1.21-alpha
        restart: always
        ports:
        - "4466:4466"
        environment:
          PRISMA_CONFIG: |
            port: 4466
            # uncomment the next line and provide the env var PRISMA_MANAGEMENT_API_SECRET=my-secret to activate cluster security
            # managementApiSecret: my-secret
            databases:
              default:
                connector: mongo
                migrations: true
                host: mongo
                port: 27017
                user: prisma
                password: prisma
      mongo:
        image: mongo:3.6
        restart: always
        environment:
          MONGO_INITDB_ROOT_USERNAME: prisma
          MONGO_INITDB_ROOT_PASSWORD: prisma
        ports:
          - "27017:27017"
    volumes:
          - mongo:/var/lib/mongo
    volumes:
      mongo:
```

- run `docker-compose up -d`
- run `prisma deploy` with your datamodel as usual
- have fun

Since we are regularly merging new functionality / fixes for this preview you should run `docker-compose pull` when restarting your prisma server to make sure it pulls the latest version of the preview.

**Embedded Types**

The Mongo connector introduces the concept of embedded types. These are stored nested within their parent types and do not have their own collections. They have no ids and no backrelations to their parents. We also do not generate toplevel queries or mutations for these. They can only be accessed through their respective parents. See more details and the reasoning behind them here: https://github.com/prisma/prisma/issues/2836

This is how an embedded type would be defined in the schema:

```graphql
type Parent {
  id: ID! @unique
  name: String!
  child: Child
 }
    
type Child @embedded {
  name: String!
}
```

**Join Relations**

These are the relations joining different collections. Relational filters on Join Relations do not yet work (_some, _none, _every on Relationfields). In order to specify a join relation all rules about relation directives from the SQL connectors apply. Additionally the model side that is supposed to store the related ids inline has to be decorated with `@mongoRelation(field: "fieldNameInDB")`.

```graphql
type Parent {
  id: ID! @unique
  name: String!
  child: Child @mongoRelation(field: "fieldNameInDB")
 }
    
type Child  {
  name: String!
  parent: Parent
}
```

**Relations Between Embedded Types and Top Level Types other Than Their Parent**

These allow an embedded type to link to other top level types. This relation will store the reference to the top level type on the embedded type. The relation can also only be traversed `Child` to `Friend` since there is no backrelation. This can be used to link to common shared data (country information in a shopping cart for example).

```graphql
type Parent{
    name: String
    child: Child
}

type Friend{
    name: String
}

type Child @embedded {
    name: String
    friend: Friend @mongoRelation(field: "friend")
}
```

**Nested DeleteMany / UpdateMany**

Since unique constraints on embedded documents are not enforced due to a Mongo bug https://jira.mongodb.org/browse/SERVER-1068  we are introducing these two new nested mutations to address related nodes without them having a unique constraint. You can find more information about how these work here: https://github.com/prisma/prisma/pull/3376 We will soon remove the possibility to set unique constraints on embedded types until the Mongo bug is fixed. 

**Performance**

The performance of the current state of the connector is not indicative of the final performance. There are several improvements we will make that should speed it up noticeably.

- only fetching selected fields / nested documents
- indexes on embedded types

**Known Limitations**

These are things that are currently not implemented yet, but we will be working on these in the coming weeks.Since this is an early prototype things might fail with `Not Implemented` exceptions. This is not yet intended for production use.

**Features Other Connectors Have That Will Be Implemented Later**
- Cascading delete
- Bulk Import / Export
- Raw Mongo query execution.

***API Differences (schema is not yet adjusted )

- Unique Constraints are not enforced within embedded documents due to a bug in Mongo https://jira.mongodb.org/browse/SERVER-1068 
- The schema for embedded types will still contain connect / disconnect even though these do not work on embedded types
- A lot of deploy functionality not implemented (renaming does not work / Deleting a type does not delete it's collection)
- Filters and pagination on nested Documents are ignored
- Relational Filters on Join Relations do not work

**Reporting Issues**

Please report issues in this repo if something that is not listed as a known limitation breaks or you detect any other unexpected bugs. If you have general feedback or ideas for improvements let us know, we're always happy for suggestions and at this early stage it is easier to incorporate them.


**Discussing the Development**

If you have questions or want to discuss feature requests come on over to the Beta channel in the Prisma Slack. 
