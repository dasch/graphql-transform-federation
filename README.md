# graphql-transform-federation

If you want to use
[GraphQL federation](https://www.apollographql.com/docs/apollo-server/federation/introduction/),
but you can't rebuild your current GraphQL schema, you can use this transform to
add GraphQL federation functionality to an existing schema.

This transform will add the resolvers and directives to conform to the
[federation specification](https://www.apollographql.com/docs/apollo-server/federation/federation-spec/#federation-schema-specification).
Much of the
[federation sourcecode](https://github.com/apollographql/apollo-server/tree/master/packages/apollo-federation)
could be reused.

## Usage

This example shows a configuration where the transformed schema extends an
existing schema. It already had a resolver `productById` which is used to relate
products between the two schemas. This example can be started using
[npm run example](#npm-run-example).

```typescript
import { transformSchemaFederation } from 'graphql-transform-federation';
import { delegateToSchema } from 'graphql-tools';

const schemaWithoutFederation = // your existing executable schema

const federationSchema = transformSchemaFederation(schemaWithoutFederation, {
  Query: {
    // Ensure the root queries of this schema show up the combined schema
    extend: true,
  },
  Product: {
    // extend Product {
    extend: true,
    // Product @key(fields: "id") {
    keyFields: ['id'],
    fields: {
      // id: Int! @external
      id: {
        external: true
      }
    },
    resolveReference({ id }, context, info) {
      return delegateToSchema({
        schema: info.schema,
        operation: 'query',
        fieldName: 'productById',
        args: {
          id,
        },
        context,
        info,
      });
    },
  },
});
```

To allow objects of an existing schema to be extended by other schemas it only
needs to get `@key(...)` directives.

```typescript
const federationSchema = transformSchemaFederation(schemaWithoutFederation, {
  Product: {
    // Product @key(fields: "id") {
    keyFields: ['id'],
  },
});
```

## API reference

```typescript
import { GraphQLSchema } from 'graphql';
import { GraphQLReferenceResolver } from '@apollo/federation/dist/types';

interface FederationFieldConfig {
  external?: boolean;
  provides?: string;
}

interface FederationFieldsConfig {
  [fieldName: string]: FederationFieldConfig;
}

interface FederationObjectConfig<TContext> {
  // An array so you can add multiple @key(...) directives
  keyFields?: string[];
  extend?: boolean;
  resolveReference?: GraphQLReferenceResolver<TContext>;
  fields?: FederationFieldsConfig;
}

interface FederationConfig<TContext> {
  [objectName: string]: FederationObjectConfig<TContext>;
}

function transformSchemaFederation<TContext>(
  schema: GraphQLSchema,
  federationConfig: FederationConfig<TContext>,
): GraphQLSchema;
```

## `npm run example`

Runs 2 GraphQL servers and a federation gateway to combine both schemas.
[Transformed-server](./example/transformed-server.ts) is a regular GraphQL
schema that is tranformed using this library. The
[federation-server](example/federation-server.ts) is a federation server which
is extended by a type defined by the `transformed-server`. The
[gateway](./example/gateway.ts) combines both schemas using the apollo gateway.

## `npm run example:watch`

Runs the example in watch mode for development.

## `npm run test`

Run the tests

## The `requires` directive

When extending an existing type you can resolve derived properties using
[the `requires` directive](https://www.apollographql.com/docs/apollo-server/federation/advanced-features/#computed-fields).
When you have an existing schema that you want to transform to a federated
schema I'm not sure what you would use if for. So unless I learn of a real world
usecase, I can't implement the `requires` directive.
