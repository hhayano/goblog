# Introducing Amazon DynamoDB Expression Builder in the AWS SDK for Go

The (version) release of the [AWS SDK for Go](https://aws.amazon.com/sdk-for-go/)
adds a new
[`expression`](https://docs.aws.amazon.com/sdk-for-go/api/service/dynamodb/expression)
package giving you the ability to create
[DynamoDB Expressions](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.html)
using statically typed builders. The `expression` package abstracts away the
low-level detail of using DynamoDB Expressions and simplifies the process of
using DynamoDB Expressions in
[DynamoDB Operations](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.html).
In this blog post, we explain how to use the `expression` package.

In the previous versions of the AWS SDK for Go, the member fields of the
DynamoDB Operation input structs, such as `QueryInput` and `UpdateItemInput`,
had to be declared explicitly. That meant that the syntax and rules of DynamoDB
Expressions were up to you to figure out. The goal of the `expression` package
is to create the formatted DynamoDB Expression strings under the hood thus
simplifying the process of using DynamoDB Expressions. The following example
will illustrate the verbosity of writing DynamoDB Expressions by hand.

```go
input := &dynamodb.ScanInput{
    ExpressionAttributeNames: map[string]*string{
        "#AT": aws.String("AlbumTitle"),
        "#ST": aws.String("SongTitle"),
    },
    ExpressionAttributeValues: map[string]*dynamodb.AttributeValue{
        ":a": {
            S: aws.String("No One You Know"),
        },
    },
    FilterExpression:     aws.String("Artist = :a"),
    ProjectionExpression: aws.String("#ST, #AT"),
    TableName:            aws.String("Music"),
}
```

## Representing DynamoDB Expressions

DynamoDB Expressions are represented by static builder types in the `expression`
package. These builders, like `ConditionBuilder` and `UpdateBuilder`, are
created in the package using a builder pattern. The static typing of the
builders allows compile-time checks on the syntax of the DynamoDB Expressions
being created. The following example shows how to create a builder that
represents a `FilterExpression` and a `ProjectionExpression`.

```go
filt := expression.Name("Artist").Equal(expression.Value("No One You Know"))
// let :a be an ExpressionAttributeValue representing the string "No One You Know"
// equivalent FilterExpression: "Artist = :a"

proj := expression.NamesList(expression.Name("SongTitle"), expression.Name("AlbumTitle"))
// equivalent ProjectionExpression: "SongTitle, AlbumTitle"
```

In the example above, the variable `filt` represents a `FilterExpression`. Note
that DynamoDB item attributes are represented using the function `Name()` and
DynamoDB item values are similarly represented using the function `Value()`. In
this context, the string `"Artist"` represents the name of the item attribute
that we want to evaluate and the string `"No One You Know"` represents the value
we want to evaluate the item attribute against. The relationship between the two
[operands](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.OperatorsAndFunctions.html#Expressions.OperatorsAndFunctions.Syntax)
are specified using the method `Equal()`.

Similarly, the variable `proj` represents a `ProjectionExpression`. The list of
item attribute names comprising the `ProjectionExpression` are specified as
arguments to the function `NamesList()`. The `expression` package utilizes the
type safety of Go and if an item value were to be used as an argument to the
function `NamesList()`, a compile time error is returned. The pattern of
representing DynamoDB Expressions by indicating relationships between `operands`
with functions is consistent throughout the whole `expression` package.

## Creating an `Expression`

The `Expression` type is the core of the `expression` package. An `Expression`
represents a collection of DynamoDB Expressions with getter methods such as
`Condition()` and `Projection()` to retrieve specific formatted DynamoDB
Expression strings. The following example shows how to create an `Expression`.

```go
filt := expression.Name("Artist").Equal(expression.Value("No One You Know"))
proj := expression.NamesList(expression.Name("SongTitle"), expression.Name("AlbumTitle"))

expr, err := expression.NewBuilder().WithFilter(filt).WithProjection(proj).Build()
if err != nil {
  fmt.Println(err)
}
```

In the example above, the variable `expr` is an instance of an `Expression`
type. An `Expression` is built using a builder pattern. First, a new `Builder`
is initialized by the `NewBuilder()` function. Then, types representing DynamoDB
Expressions are added to the `Builder` by methods `WithFilter()` and
`WithProjection()`. The `Build()` method returns an instance of an `Expression`
and an error. The error will be either an `InvalidParameterError` or an
`UnsetParameterError`.

There is no limit to the number of different kinds of DynamoDB Expressions that
can be added to the `Builder` but adding the same type of DynamoDB Expression
will overwrite the previous DynamoDB Expression. The following example will show
a specific instance of this problem.

```go
cond1 := expression.Name("foo").Equal(expression.Value(5))
cond2 := expression.Name("bar").Equal(expression.Value(6))
expr, err := expression.NewBuilder().WithCondition(cond1).WithCondition(cond2).Build()
if err != nil {
  fmt.Println(err)
}
```

In the example above, it highlights the fact that the second call of
`WithCondition()` overwrites the first call.

## Filling in the fields of a DynamoDB `Scan` API

The following example shows how to use an `Expression` to fill in the member
fields of DynamoDB Operation API

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

In the example above, the getter methods of the `Expression` type is used to
get the formatted DynamoDB Expression strings. The `ExpressionAttributeNames`
and `ExpressionAttributeValues` member field of the DynamoDB API must always be
assigned when using an `Expression` since all item attribute names and values
are aliased. That means that if the `ExpressionAttributeNames` and
`ExpressionAttributeValues` member is not assigned with the corresponding
`Names()` and `Values()` methods, the DynamoDB operation will run into a logic
error.

If you need a starting point, check out the
[working example](https://github.com/aws/aws-sdk-go/tree/master/example/service/dynamodb/expression/)
in the Go SDK. 

Overall, the `expression` package makes using the DynamoDB Expressions clean and
simple. The complicated syntax and rules of DynamoDB Expressions is abstracted
away you no longer have to worry about them!
