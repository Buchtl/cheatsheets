# Graphql

## Misc
* `!`: non nullable field (e.g. `String!`)
    ```
    type User {
      id: ID!
      name: String
      age: Int
    }
    ```
* `ID`: Indicates a unique identifier that serializes as a string but doesn't have to be one. Uniqueness is not enforced, so it's somehow just a _hint_ free for interpretation.[[1]](#1)

## Introspection of the schema
* Query entire schema
``` json5
query {
  __schema {
    types {
      name
    }
  }
}
```
* Query specific type
``` json5
query {
  __type(name: "String") {
    name
    kind
    description
    fields {
      name
      type {
        name
      }
    }
  }
}
```
* Response for type "String":
``` json5
{
  "data": {
    "__type": {
      "name": "String",
      "kind": "SCALAR",
      "description": "Built-in String",
      "fields": null
    }
  }
}
```

## References
[1] [graphql specification october 21](https://spec.graphql.org/October2021/#sec-ID)

# Curl Mutation
To send a GraphQL mutation via `curl`, you need to include the GraphQL query or mutation as a JSON payload in the HTTP request. Here's how you can send the `addNewElement` mutation:

### Example Command

```bash
curl -X POST https://your-graphql-endpoint.com/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -d '{"query":"mutation { addNewElement(name: \"NEW1\") }"}'
```

### Explanation:
1. **`-X POST`**: Specifies a POST request since GraphQL typically uses POST for queries and mutations.
2. **`-H "Content-Type: application/json"`**: Sets the `Content-Type` header to indicate the payload format.
3. **`-H "Authorization: Bearer YOUR_ACCESS_TOKEN"`**: (Optional) Adds an Authorization header if your GraphQL API requires authentication.
4. **`-d '{"query":"mutation { addNewElement(name: \"NEW1\") }"}'`**:
   - Encodes the GraphQL mutation as a JSON string.
   - The `query` key contains the mutation in GraphQL syntax.

### Replaceable Parts:
- Replace `https://your-graphql-endpoint.com/graphql` with your GraphQL server's URL.
- If your server requires authentication, replace `YOUR_ACCESS_TOKEN` with your valid token or remove the header if not needed.

### Notes:
- Ensure you escape double quotes (`"`) in the JSON payload with a backslash (`\"`).
- If the mutation takes variables, you'll need to adjust the `curl` request to include a `variables` key. Let me know if this applies, and I can provide an example!
