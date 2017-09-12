# Introduction of an abstraction around Amazon DynamoDB Expressions with the package expression

The new `expression` package of the
[AWS SDK for Go](https://aws.amazon.com/sdk-for-go/) provides the functionality
of creating formatted
[DynamoDB Expression](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.html)
strings for the users, abstracting away the idea of
[`ExpressionAttributeNames`](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.ExpressionAttributeNames.html)
and
[`ExpressionAttributeValues`](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.ExpressionAttributeValues.html).
With the `expression` package, users are able to specify the operands of the
DynamoDB Expressions and the relationships between the operands functionally
using a builder pattern. Users can then call getter methods that return the
correctly formatted DynamoDB Expressions as well as `ExpressionAttributeNames`
and `ExpressionAttributeValues` instead of having to create those structs by
hand. This blog post shows some examples of using the new package.

## Creating a `ScanInput` struct using the `expression` package

```go
filt := expression.Name("Artist").Equal(expression.Value("No One You Know"))
proj := expression.NamesList(expression.Name("SongTitle"), expression.Name("AlbumTitle"))
expr, err := expression.NewBuilder().WithFilter(filt).WithProjection(proj).Build()
if err != nil {
  fmt.Println(err)
}

input := &dynamodb.ScanInput{
  ExpressionAttributeNames:  expr.Names(),
  ExpressionAttributeValues: expr.Values(),
  FilterExpression:          expr.Filter(),
  ProjectionExpression:      expr.Projection(),
  TableName:                 aws.String("Music"),
}
```

The example above shows how a user can create builder structs representing the
DynamoDB Expression strings, see `filt` and `proj`, and how the user can use
those structs as inputs to a builder pattern to create an `Expression` struct.
Then, using getter methods on the `Expression` struct, the user can fill out the
member fields of the `dynamodb.ScanInput` struct. Note that if a user decides to
use the `expression` package, they must fill out the `ExpressionAttributeNames`
field and `ExpressionAttributeValues` field of the `dynamodb` input struct. This
is due to the fact that the `expression` package substitutes all item attribute
names and values that are specified when creating the builder structs
representing the DynamoDB Expression strings. If the `ExpressionAttributeNames`
field or `ExpressionAttributeValues` field is left unset while using the
`expression` package, logic errors may occur.

## Strong typing for functions

The `expression` package utilizes strong typing of Go to enforce syntax rules of
DynamoDB Expressions.

```go
// returns an error
keyCond := expression.Name("foo").Equal(expression.Value("bar"))
expr, err := expression.NewBuilder().WithKeyCondition(keyCond).Build()

// does not return an error
keyCond := expression.Key("foo").Equal(expression.Value("bar"))
expr, err := expression.NewBuilder().WithKeyCondition(keyCond).Build()
```

The example above shows an instance where the type safety employed in the
`expresssion` package returns an error. The error occurs since
`KeyConditionBuilder` is only created by `KeyBuilder`, not `NameBuilder`. Thus,
when the method `Equal()` is called on the `NameBuilder` created by `Name()`,
the method `Equal()` creates a `ConditionBuilder` instead of a
`KeyConditionBuilder`. Since the `WithKeyCondition()` method expects a
`KeyConditionBuilder`, the compiler will return an error. There are many other
DynamoDB syntax rules that are enforced using type safety. If some code is not
compiling, please check the
[AWS SDK for Go API Reference](https://docs.aws.amazon.com/sdk-for-go/api/service/dynamodb/expression)
and the
[Amazon DynamoDB Developer Guide](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.html).
