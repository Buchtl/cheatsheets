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

## References
[1] [graphql specification october 21](https://spec.graphql.org/October2021/#sec-ID)
