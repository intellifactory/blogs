---
title: "F# query expressions and composability"
categories: "query,composability,quotations,linq,f#"
abstract: "F# 3.x Query Expressions are a nice piece of syntactic sugar, but they are notably difficult to compose. This article discusses the auto-quoting mechanism of query expressions and how it can be leveraged to implement not-too-clunky composition."
identity: "3508,76794"
---
Anyone who has written non-trivial applications that need to access a SQL database can attest it: being able to compose SQL queries, insert parts of query that depend on some runtime criterion, is tremendously helpful. Not having this capability often means having to copy and paste queries just to be able to add a "where" or a "join" clause, and I know few better ways to end up with buggy, hard-to-maintain code.

This article discusses the auto-quoting mechanism of F# 3.x query expressions and how it can be leveraged to implement not-too-clunky composition.

## PowerPack.Linq

In F# 2.0, the standard / usual way to access SQL databases was PowerPack.Linq, which featured a `query` function capable of translating a quotation containing sequence operations into a LINQ query. Therefore, composing sub-queries basically meant composing sequence operations, and it was not too difficult: quotation splicing did the trick.

Let's take the classic example of a web forum, with a `Message` table containing a field `UserId` that keys into a `User` table through its `Id` field. Let's write the following function:

```fsharp
val GetLatestMessages : num: int -> username: string option -> Message[]
```

If `username` is `None`, we return the `num` latest messages posted by anyone; else, only those posted by the given user. So depending on `username`, we will need two different SQL queries: one with a join into the users table, and one without.

With PowerPack.Linq, it will look like this:

```fsharp
let GetLatestMessages num usernameOpt =
  use db = (* retrieve LINQ-to-SQL context *)
  let baseQuery =
    <@ db.Messages |> Seq.sortBy (fun m -> m.PostedDate) @>
  let filteredQuery =
    match usernameOpt with
    | None -> baseQuery
    | Some username ->
      <@ seq { for m in %baseQuery do
               for u in db.Users do
               if u.Username = username && m.UserId = u.Id then
                 yield m } @>
  query <@ %filteredQuery |> Seq.take num @>
  |> Array.ofSeq
```

Note the use of the percent operator. It performs quotation splicing, which means that its argument, itself a quotation, is put as-is into the quotation being defined. In the end, the quotation compiled by `query` depends on the value of `usernameOpt`. If it is `None`, then the final quotation is:

```fsharp
<@ db.Messages
   |> Seq.sortBy (fun m -> m.PostedDate)
   |> Seq.take num @>
```

If it is `Some username`, then the final quotation is:

```fsharp
<@ seq { for m in (db.Messages |> Seq.sortBy (fun m -> m.PostedDate)) do
         for u in db.Users do
         if u.Username = username && m.UserId = u.Id then
           yield m }
   |> Seq.take num @>
```

The function `query` is then able to take whichever of these quotations, compile it into a valid SQL query, and run it. We do have the functionality we need, with the totality of the query being performed by the database.

## Query Expressions: The Problem

F# 3.0 introduced query expressions, with the intent of creating a syntax similar to C#'s own LINQ syntax, itself very close to plain SQL. Using query expressions, the two above variants would be written as follows:

```fsharp
query { for m in db.Messages do
        sortBy m.PostedDate
        select m
        take num }

query { for m in db.Messages do
        sortBy m.PostedDate
        join u in db.Users on (m.UserId = u.Id)
        where (u.Username = username)
        select m
        take num }
```

But how can we choose between these two queries without having to duplicate the common parts, just like we did with `PowerPack.Linq`? As a first attempt, we can try to use the result of the previous fragment in the `for` part of the next one.

```fsharp
let GetLatestMessages num usernameOpt =
  use db = (* retrieve LINQ-to-SQL context *)
  let baseQuery =
    query { for m in db.Messages do
            sortBy m.PostedDate
            select m }
  let filteredQuery =
    match usernameOpt with
    | None -> baseQuery
    | Some username ->
      query { for m in baseQuery do
              join u in db.Users on (m.UserId = u.Id)
              where (u.Username = username)
              select m }
  query { for m in filteredQuery do
          take num }
  |> Array.ofSeq
```

However, at runtime, we get this cryptic exception (using a LINQ to Entities context; the message will probably be different if you use LINQ to SQL):

```
This method supports the LINQ to Entities infrastructure
and is not intended to be used directly from your code.
```

What is going on here? Well, the problem here is that the `query` builder actually performs the translation to LINQ. So `baseQuery` is actually an `IQueryable<Message>` representing the output of the query. It is therefore too late to combine into a more complex query!

## Query Expressions: The Solution

In order to progress further, we need to understand how query expressions work internally. Like any F# computation expression, a query expression is mostly syntactic sugar on top of method calls on the builder object used to create the expression, in this case the `query` object. For example, `baseQuery` above corresponds to the following expression:

```fsharp
  query.Select(
    query.SortBy(
      query.Source(db.Messages),
      fun m -> m.PostedDate),
    fun m -> m)
```

(I am actually omitting a couple extra method calls that would make this less readable, but the principle stays the same.)

But actually... Not quite. If this was all there was to it, then it would be trivially composable, just like normal `seq { ... }` expressions are composable. But in order to be able to compile such an expression to LINQ, it needs to be able to inspect it; and to do so, without telling you, it uses... *(drumrolls...)* quotations.

In fact, the query expression syntax does more than put sugar on top of some method calls; when some conditions are met (ie. when the builder object has certain specific methods), it also implicitly puts the whole body in a quotation, and calls the `Run` method on it. The translation of `baseQuery` is therefore closer to this:

```fsharp
query.Run <@
  query.Select(
    query.SortBy(
      query.Source(db.Messages),
      fun m -> m.PostedDate),
    fun m -> m)
@>
```

`query.Run : Expr<Linq.QuerySource<'T, _>> -> IQueryable<'T>` is responsible for taking this quotation and turning it into the final LINQ expression that maps directly to SQL.

Now if only we could have direct access to the quotation, we would be golden. Well, it turns out we can do exactly that. We are going to play around with quoting and unquoting pieces of queries, which is something I've first seen done by Tomas Petricek [here](http://stackoverflow.com/questions/13826749/how-do-you-compose-query-expressions-in-f). In fact, this SO response is the original inspiration for this whole post.

All we have to do here is to create a subclass of `Linq.QueryBuilder` (the class of `query`) that implements `Run` differently.

```fsharp
type PartialQueryBuilder() =
  inherit Linq.QueryBuilder()

  member this.Run(e: Expr<Linq.QuerySource<'T, IQueryable>>) = e

let pquery = PartialQueryBuilder()
```

We now have a query expression constructor, `pquery`, that we can invoke exactly like `query`, except that it returns the unevaluated query quotation.

```fsharp
let baseQuery =
  pquery{ for m in db.Messages do
          sortBy m.PostedDate
          select m }

// val baseQuery : Expr<Linq.QuerySource<'T, _>>
```

What we want to do next is to be able to pass this unevaluated query to `for` in another query. As I've shown above, the sequence given to `for` is passed by the desugaring to a method `query.Source : IQueryable<'T> -> Linq.QuerySource<'T, _>`. But here, pquery directly gives us a `Linq.QuerySource<'T, _>`. No problem, we can use a trick similar to the above `Run` trick, except using an extension method on `query` itself:

```fsharp
type Linq.QueryBuilder with
  [<ReflectedDefinition>]
  member this.Source(qs: Linq.QuerySource<'T, _>) = qs
```

Okay, now all we have to do is pass our partial query to `for`. But wait, isn't it an `Expr<Linq.QuerySource<'T, _>>`, ie. a quoted version of what the `Source` we just wrote expects? Well, remember when I said that the body of query expressions gets wrapped in a quotation? This is not just a figure, it really is exactly what happens. So you can even use the `%` splicing operator in query expressions! For our example, the following code is valid:

```fsharp
query { for m in %baseQuery do
        join u in db.Users on (m.UserId = u.Id)
        where (u.Username = username)
        select m }
```

And there we go! We just wrote a partial query and composed it into a final query, and readability-wise all it takes is an extra `p` in the former and a `%` in the latter. Not bad!

Here is the whole code for reference:

```fsharp
// Utility code, write it once

type PartialQueryBuilder() =
  inherit Linq.QueryBuilder()

  member this.Run(e: Expr<Linq.QuerySource<'T, IQueryable>>) = e

let pquery = PartialQueryBuilder()

type Linq.QueryBuilder with
  [<ReflectedDefinition>]
  member this.Source(qs: Linq.QuerySource<'T, _>) = qs

// Example usage

let GetLatestMessages num usernameOpt =
  use db = (* retrieve LINQ-to-SQL context *)
  let baseQuery =
    pquery{ for m in ctx.Messages do
            sortBy m.PostedDate
            select m }
  let filteredQuery =
    match usernameOpt with
    | None -> baseQuery
    | Some username ->
      pquery{ for m in %baseQuery do
              join u in db.Users on (m.UserId = u.Id)
              where (u.Username = username)
              select m }
  query { for m in %filteredQuery do
          take num }
  |> Array.ofSeq
```

Note how we can just as easily compose a partial query into another partial query. And finally, if you need to directly run a partial query (as opposed to splicing it into a normal query), you can just pass it to the standard `query.Run`.
