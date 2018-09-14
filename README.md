# Mongo-Connector-Preview
This is a repo for an early preview of the Mongo Connector.
This is a very early prototype that is mainly intended to test a new concept we are introducing with the Mongo connector - embedded types. Normal relations between non-embedded types are not supported at this early stage, but will be added soon.

**How to use**

The functionality for the Mongo Connector is present in the Prisma Core starting in 1.17-beta. The CLI does not yet know about the Mongo Connector though, so some changes need to be done manually.

We are changing the preview branch to alpha. This way you will always have the most current version of the connector. I've adjusted the docker-compose.yml accordingly. If you are still running on the 1.17-beta pls restart prisma with the new alpha image. We'll merge new functionality regularly into the alpha and will adjust the list of known limitations accordingly.

- install the Prisma cli
- run `prisma init` - choose new local db with mysql
- switch out the auto-generated docker-compose.yml with the following one

```yml
    version: '3'
    services:
      prisma:
        image: prismagraphql/prisma:1.18-alpha
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
                migrations: false
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

**New Features**

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

**Known Limitations**

These are things that are currently not implemented yet, but we will be working on these in the coming weeks.Since this is an early prototype things might fail with `Not Implemented` exceptions. This is not intended for production use.

- No Upsert, neither on the top level nor nested
- No raw Mongo query execution.
- Unique Constraints are not enforced
- Pagination (first, last, skip, after, before) not implemented.
- Subscriptions are not working
- The schema for embedded types will still contain connect / disconnect even though these do not work on embedded types
- Relations are only implemented for embedded types 
- Cascading delete not implemented, but since all relations are embedded they automatically behave as if cascade is set from top to bottom.
- No Import / Export implemented yet
- A lot of deploy functionality not implemented (renaming does not work / Deleting a type does not delete it's collection)
- If required fields are not in the db, there will be an error and the whole object will return null
- Filters and pagination on nested Documents are ignored

**Reporting Issues**

Please report issues in this repo if something that is not listed as a known limitation breaks or you detect any other unexpected bugs. If you have general feedback or ideas for improvements let us know, we're always happy for suggestions and at this early stage it is easier to incorporate them.
