---
title: "IntelliLogo"
categories: "f#,websharper,parsing,lexing,logo,fsadvent"
abstract: ""
identity: "-1,-1"
---

(Happy Holidays everyone and sorry for the delay in getting this article out to you! Just want to say a huge thanks to Sergey Tihon for organizing F# Advent and to everyone who participated!)

I wrote an article nearly 20 years ago about [implementing a Logo interpreter in F# with 400 LOC](https://intellifactory.com/user/granicz/20061124-a-logo-interpreter-under-400-lines), and a fun fact is that it was the first article on this very blog. Back then, one of the coolest things about it was that it used partial active patterns to parse Logo programs and executed them in a desktop client.

In this article, I will give a quick overview of a more sophisticated rewrite, by implementing IntelliLogo, an abstract Logo "virtual machine" with a range of built-in functions and features, and adding a simple web UI around it using WebSharper. The [resulting app](https://logo.intellifactorylabs.com) looks like this in action (click for a larger image):

| | | |
|--|--|--|
| [![](../assets/intellilogo-app.png)](../assets/intellilogo-app.png) | [![](../assets/intellilogo-warning.png)](../assets/intellilogo-warning.png) | [![](../assets/intellilogo-plant.png)](../assets/intellilogo-plant.png) |


> Currently, WebSharper 8 is in beta, and it is most readily available from the WebSharper GitHub/developer feed. Check [here](https://docs.websharper.com/basics/nuget/) about configuring your NuGet/Paket client to use these releases, you won't be able to compile the [project](https://github.com/granicz/IntelliLogo) otherwise!
> ### Try it!
> (You will need the latest `dotnet` CLI and `npx` installed on your system.)
>
> 1. Clone the repo: `git clone https://github.com/granicz/IntelliLogo`
>
> 2. cd into the project: `cd IntelliLogo`
>
> 3. Build and run it: `dotnet build`
>
> 4. Run the client: `npx vite`
>
> 5. Navigate to `https://localhost:xxxx`, where `xxxx` is the port the app is started on (as reported by vite)


## Project structure

```
- IntelliLogo
   - AST.fs
   - Lexer.fsl
   - Parser.fsy
   - [Parser.fs]  -> generated
   - [Lexer.fs]   -> generated
   - Eval.fs
- Client
   - [AST.fs]     -> from IntelliLogo
   - [Parser.fs]  -> from IntelliLogo
   - [Lexer.fs]   -> from IntelliLogo
   - [Eval.fs]    -> from IntelliLogo
   - Builtins.fs
   - App.fs
```

## The AST

The types needed for the IntelliLogo VM are in `AST.fs`, the AST itself is pretty straightforward, with a constructor for each notable feature:

```fsharp
module AST =
    type id = string * pos

    type Program = Statement list

    and Commands = Statement list
    
    and Statement =
        | FunDef of id * id list * Commands * pos
        | Repeat of Expr * Commands * pos
        | For of id * Expr * Expr * Expr option * Commands * pos
        | If of Expr * Commands * pos
        | IfElse of Expr * Commands * Commands * pos
        // make "i 1
        | Make of id * Expr * pos
        // makelocal "i 1
        | MakeLocal of id * Expr * pos
        // local "i
        | Local of id * pos
        | FunCall of id * Expr list * pos
        // Expressions can be returned with "output"
        | Output of Expr * pos

    and Expr =
        | RepCount of pos
        | Var of string * pos
        | Number of float * pos
        | Word of string * pos
        | Text of string * pos
        | FunCall of id * Expr list * pos
        | List of Expr list * pos
        | Add of Expr * Expr * pos
        | Sub of Expr * Expr * pos
        | Mul of Expr * Expr * pos
        | Div of Expr * Expr * pos
        | Gt of Expr * Expr * pos
        | Lt of Expr * Expr * pos
        | Ge of Expr * Expr * pos
        | Le of Expr * Expr * pos
        | Eq of Expr * Expr * pos
        | NotEq of Expr * Expr * pos
```

For `pos`, we use a record type that wraps FsLexYacc's native `Lexing.Position` type and adds its own helpers, this way a consuming library doesn't have to know about lexing internals.

## The parser

For IntelliLogo, I decided to use a lexer/parser generator instead of the active pattern-based approach to benefit from a more formal way to describe IntelliLogo's syntax, although in the process I opted for a small but notable deviation from the mainstream LOGO syntax: function calls with zero or multiple arguments use parentheses to surround those arguments.

For instance, the following:

```logo
LT 10 PENUP PRINT 1 2 3
```

need to be written as:

```logo
LT 10 PENUP () PRINT (1 2 3)
```

Functions are defined the same way as in standard logo, no extra parentheses needed there.

An IntelliLogo program consists of a list of statements:

```
program:
    | statements EOF {
        fst $1
    }
```

Here, `program` is a so-called non-terminal symbol, as opposed to terminal symbols which are fed from the lexer. We can write `statements`, and list of anything in general, as follows:

```
statements:
    | statement {
        ...
    }
    | statement statements {
        ...
    }
```

Another useful grammar abstraction is expressing optionality. For statements, a "possibly empty list of statements" is written as:

```
statements_opt:
    | {
        []
    }
    | statements {
        (fst $1)
    }
```

Note how the first rule is "empty". The order of cases within a rule is not relevant, and the generated parser will always use the pattern that matches the longest input string.

Many statements use blocks of code (further statements) as part of their syntax, and put these bits inside brackets, for instance, conditionals or REPEAT blocks:

```
REPEAT 10 [LT 10 FD 10]
```

To make parsing these inner blocks easier, we can introduce another rule for them:

```
commands:
    | LBRACK statements_opt RBRACK {
        $2, pos.union $1 $3
    }
```

At this point, we can start specifying what statements we accept - function definitions, REPEAT and FOR statements, etc:

```
statement:
    | TO ID vars_opt statements END {
        ...
    }
    | REPEAT expr commands {
        ...
    }
    | FOR LBRACK ID expr expr RBRACK commands {
        ...
    }
    ...
```

### Handling source positions

An essential components of parsing IntelliLogo programs is keeping track of source regions that correspond to each parsed symbol. This involves aggregating the positions of the inner symbols within a production rule, and returning this combined source position as well. Consider a couple actual snippets from parsing expressions:

```
expr:
    ...
    | ID LPAREN exprs RPAREN {
        let pos = pos.union (snd $1) $4
        FunCall ($1, fst $3, pos), pos
    }
    // A list is an expression, not to be confused with statement blocks.
    | LBRACK exprs_opt RBRACK {
        let pos = pos.union $1 $3
        List ($2, pos), pos
    }
    | expr PLUS expr {
        let pos = pos.union (snd $1) (snd $3)
        Add (fst $1, fst $3, pos), pos
    }
```

Here, some symbols (`ID`, `expr`) carry a value and a source position as a tuple, and we extract these to combine with other positions with `pos.union`, which simply creates a position value that spans its two input positions.


## The evaluator

The evaluator computes a `Value` for various AST forms. These values can be of a range of shapes:

```fsharp
type Value =
    | Nothing
    | Number of float
    | Boolean of bool
    | Text of string
    | List of Value list
    | Explicit of Value

...

let evalCommands ... = ...
and evalStatements ... = ...
and evalExpr ... = ...
```

Statements eventually evaluate to `Nothing` (but may change the execution context), after all, these constructs don't return values like functions and expressions. `Explicit` marks values returned using the `OUTPUT` function - this shape is not strictly necessary, but I decided to keep it explicit in case I later wanted to add extra semantic checks around user-defined functions.

Each helper in the evaluator receives an `Environment`, the evaluation context, containing the global and local variables in scope, and the user-defined and built-in functions.

```fsharp
type Environment =
    {
        GlobalVariables: Map<string, Value>
        LocalVariables: Map<string, Value option>
        Functions: Map<string, (string list * Commands)>
        BuiltIns: Map<string, (Environment -> pos -> Value list -> ((Environment * Value) * bool))>
    }
```

When calling user-defined functions, the evaluator manages the environment, but built-in functions explicitly take and return an `Environment` value: this allows certain built-in functions (such as `debug_environment`) to examine/print it.

The evaluator helpers return a new environment and the computed value, along with a Boolean that signals whether the evaluation has been stopped using `STOP ()` - this function terminates the currently executing user-defined function and returns control to the caller (without a return value). (Under the cover, this shortcut logic is implemented in `List.foldUntil` - a custom extension method.)

```fsharp
let rec evalCommands (env: Environment) commands : ((Environment * Value) * bool)=
    commands
    |> List.foldUntil (fun (env, v) stmt ->
        evalStatement env stmt
    ) (env, Value.Nothing)

// Evaluates a list of commands, assuming that they can only
// return a value via OUTPUT.
and evalStatementCommands env body =
    let (env, v), isStopped = evalCommands env body
    match v with
    | Value.Explicit v ->
        (env, v), isStopped
    | _ ->
        (env, Value.Nothing), isStopped

and evalStatement (env: Environment) stmt : (Environment * Value) * bool =
    match stmt with
    ...
    | Statement.IfElse(cond, yes, no, pos) ->
        let env, v = evalExpr env cond
        match v with
        | Value.Boolean b ->
            if b then
                evalStatementCommands env yes
            else
                evalStatementCommands env no
        | _ ->
            raise (BooleanExpected (v, pos))
    | ...
...
// Expressions can't contain STOP
and evalExpr env = function
    | RepCount pos ->
        evalExpr env (Var ("repcount", pos))
    | Var (v, pos) ->
        match Map.tryFind (v.ToLower()) env.GlobalVariables with
        | Some value ->
            env, value
        | None ->
            match Map.tryFind (v.ToLower()) env.LocalVariables with
            | Some (Some value) ->
                env, value
            | Some None ->
                raise (UninitializedLocalVariable(v, pos))
            | None ->
                raise (UndefinedVariable(v, pos))
    | Expr.Number (n, _) ->
        env, Value.Number n
    | Word (w, _) ->
        env, Value.Text w
    | Expr.Text (txt, _) ->
        env, Value.Text txt
    | FunCall ((f, fpos), args, pos) ->
        let (env, v), _ = evalFunCall env ((f, fpos), args, pos)
        env, v
    | Expr.List (es, pos) ->
        es
        |> List.fold (fun (env, lst) e ->
            let env, e = evalExpr env e
            env, e :: lst) (env, [])
        |> fun (env, lst) ->
            env, Value.List (List.rev lst)
    | Expr.Add (e1, e2, pos) ->
        let env, v1 = evalExpr env e1
        let env, v2 = evalExpr env e2
        match v1, v2 with
        | Value.Number n1, Value.Number n2 ->
            env, Value.Number (n1+n2)
        | _ ->
            raise (ArgumentTypeMismatch ("(+)", v1, v2, pos))
    ...
```

### Error handling and semantic checking

Since we evaluate the AST directly, without using another internal representation (IR) more suited to execution, we have to do some sanity checks "on the fly". These are handled via raising exceptions, which are then caught on the top-level (=the UI) and dealt with accordingly.

This also applies to the built-in functions, for instance, consider `random` (which takes the upper limit as an argument) and `stop` (which expects no arguments):

```fsharp
let Core = [
    "random", (fun env pos args ->
        match args with
        | [Number n] ->
            (env, Value.Number(rnd.Next(int n))), false
        | _ ->
            raise (UnexpectedArgument ("random", args, pos))
    )
    "stop", (fun env pos args ->
        match args with
        | [] ->
            // Stop the execution of the current user-defined function
            (env, Value.Nothing), true
        | _ ->
            raise (UnexpectedArgument ("stop", args, pos))
    )
    ...
```

## The client UI

Now with the parsing and evalution bits in their own project, it's time to build a UI around the VM. This is contained in the `Client` project. This project references some of the same files from the `IntelliLogo` VM, as shown earlier in the project structure section.

Recompiling the same source files in a different project provides an important advantage: by making the `Client` project a WebSharper SPA, for instance, all code within gets compiled with WebSharper, and apart from code marked specifically for server-side use (RPCs, etc.), it is retargeted to JavaScript.

### Introducing WebSharper.FsLexYacc

Here is where another important piece comes in the play: WebSharper.FsLexYacc. This package adds a WebSharper proxy for `FsLexYacc.Runtime`, the library that is used by FsLexYacc-generated lexers and parsers, including the ones used in IntelliLogo.

This proxy allows WebSharper to compile the generated code to JavaScript, essentially, making it run in the browser.

### Client-specific built-in functions

The IntelliLogo evaluator can receive additional built-in functions through the `Environment` context, in addition to the few stdlib functions implemented along with the VM. This enables us to plug in client-specific built-ins as well. These are defined in `Client\Builtins.fs`, and use an additional execution context, encapsulated in the `ClientBuiltins.LogoState` type:

```fsharp
type LogoState =
    {
        mutable CanvasX : float
        mutable CanvasY : float
        mutable CurrentX : float
        mutable CurrentY : float
        mutable Direction : float
        mutable PenDown : bool
        mutable Ctx : CanvasRenderingContext2D
        mutable Console : Var<string> option
    }

let State =
    {
        CanvasX = 1000.
        CanvasY = 1000.
        CurrentX = 500.
        CurrentY = 500.
        Direction = 90.
        PenDown = true
        Ctx = null
        Console = None
    }
```

`State` is then made available in the main SPA page. The `Ctx` field, which is an HTML5 `CanvasRenderingContext2D` backed by a `CANVAS` HTML element, and `Console`, a WebSharper.UI reactive variable bound to a UI element (the `TEXTAREA` that corresponds to the Console/Output widget in the app) are initialized when the application boots up.

Armed with UI-specific state/model, more stdlib functions can be injected as we initialize the main application - for instance, consider `home` (which clears the Logo drawing canvas and resets the turtle to the center), `rt`/`right` (which turn the turtle to the right), and `print` (which prints values to the console):

```fsharp
let Core = [
    ["home"], (fun env _ args ->
        State.CurrentX <- 500
        State.CurrentY <- 500
        (env, Value.Nothing), false
    )
    ["rt"; "right"], (fun env pos args ->
        match args with
        | [Value.Number n] ->
            TurtleOperations.RT n
        | _ ->
            raise (NumberExpectedButGot(args, pos))
        (env, Value.Nothing), false
    )
    ["print"; "pr"], (fun env pos args ->
        args |> List.iter (fun arg ->
            let s = NonListValueToString pos arg
            if State.Console.IsSome then
                Var.Concat(State.Console.Value, s + " ")
        )
        Var.Concat(State.Console.Value, "\n")
        (env, Value.Nothing), false
    )
    ...
]
```

### The actual UI

For the main IntelliLogo UI, I used ChatGPT to generate a Tailwind-based HTML page that has a toolbar and footer, and a resizeable main content panel that has a left-hand side code editor and a fill-rest preview panel with a bottom Console textarea.

In the toolbar, I threw in a center dropdown to contain a selection of built-in IntelliLogo snippets, and a handful of buttons for wiping the editor, zooming in/out of the preview canvas, running the code in the editor, and a light/dark theme selector.

Once I was finished with the HTML skeleton, I sprinkled in WebSharper attributes for each notable placeholder (`ws-replace`, `ws-hole`) and event handler (`ws-onclick`, etc.)

Then I brought the entire UI into scope using the WebSharper templating type provider:

```fsharp
[<JavaScript>]
module Client =
    type MainTemplate = Template<"index.html", ClientLoad.FromDocument, ServerLoad.WhenChanged>
```

... and started to populate the placeholders and implement the necessary event handlers:

```fsharp
[<SPAEntryPoint>]
let Main =
    ... // Initialization - omitted
    MainTemplate()
        .Editor(editor)
        // We can put global initialization code here...
        .OAR_WithModel(...)
        .EditorOnKeyUp(fun e -> UpdateCaretPosition e.Target)
        .EditorOnKeyDown(...)
        .EditorCol(EditorCaretPosition.View.Map(snd >> string))
        .EditorRow(EditorCaretPosition.View.Map(fst >> string))
        .ExampleOptions(...)
        ...
```

I paid attention to several smaller details to make the app much more pleasant to use: showing the caret's position to make it easier to track down syntax errors, keeping track of changes to selected snippets, warning on losing changes, saving the code in the editor so it survives page refreshes, remembering the selected theme, etc - you are welcome to take a look in the repo and play with adding more refinements.

Since I was planning to host the app in a static web container such as GitHub Pages, I thought it would be nice to add some dynamic functionality as well, namely, to initialize the Examples selector/dropdown from a configuration file.

For this, I used a `config.json` file in the examples folder in the repo, and read its contents as:

```fsharp
type Example =
    {
        [<Name "name">] Name: string
        [<Name "filename">] FileName: string
    }

type Examples =
    {
        [<Name "examples">] Examples : Example array
    }

let Examples : Var<Example array> = Var.Create [||]
let CurrentExampleSelected : Var<string option> = Var.Create None

let InitializeExampleSelector() =
    Communication.fetch<Examples> false "/examples/config.json"
        <| fun exs ->
            Examples := exs.Examples
        <| fun error ->
            printfn "Error: %A" error
```

Here, `Communications.fetch` is a simple helper to do an HTTP request:

```fsharp
[<JavaScript>]
module Communication =
    let fetch<'T> (isRaw: bool) (url: string) (onSuccess: 'T -> unit) (onError: string -> unit) =
        let request = new XMLHttpRequest()
        request.Open("GET", url, true)
        request.OnReadyStateChange <- fun _ ->
            if request.ReadyState = XMLHttpRequest.DONE then
                if request.Status = 200 then
                    try
                        if isRaw then
                            box request.ResponseText :?> 'T
                        else
                            Json.Parse(request.ResponseText) :?> 'T
                        |> onSuccess
                    with
                        | ex -> onError $"Parsing error: {ex.Message}"
                else
                    onError $"HTTP error: {request.StatusText}"
        request.Send()
```

Once I call `InitializeExampleSelector` in the app's boot-up sequence, I can read `Examples` and bind it to the UI dropdown that contains the built-in examples, next to the few other chores I need to wire up for the other UI widgets.

Finally, adding the event handler for the Run button feels like a breeze:

```fsharp
MainTemplate()
    // ...
    .OnRun(fun e ->
        ClientBuiltins.State.Ctx <- ..
        ClientBuiltins.State.Console <- ...
        let input = editor.Value
        match IntelliLogo.Parse.ProgramFromString input with
        | Ok program ->
            // ...
            let env = IntelliLogo.Evaluator.Environment.Create()
            // Inject the specific client builtins
            let env = ...
            try
                let (_, v), isStopped = IntelliLogo.Evaluator.evalCommands env program
                if isStopped then
                    Var.Concat(console, sprintf "Program stopped.\n")
                Var.Concat(console, sprintf "Result=%A\n" v)
            with
            // Report errors
            | IntelliLogo.Evaluator.ArgumentTypeMismatch(f, v1, v2, pos) -> ...
            | ...
```

## Conclusion

In this article, I walked through some of the implementation details of IntelliLogo: a Logo-like programming language with a web IDE. The main points of interest could be how easy it is to use lexer/parser generators to implement programming languages, or how much fun WebSharper made it to compile it into a web app using a ChatGPT-generated UI.

Extending the standard library with other Logo primitives can vastly enhance the value proposition. With a few more of these, here is a one-liner to showcase how to output text (a multiplication table) instead of turtle graphics:

```logo
for [i 1 10] [repeat 10 [type form (repcount * :i 4 0)] pr()]
```

... which yields:

```
   1   2   3   4   5   6   7   8   9  10
   2   4   6   8  10  12  14  16  18  20
   3   6   9  12  15  18  21  24  27  30
   4   8  12  16  20  24  28  32  36  40
   5  10  15  20  25  30  35  40  45  50
   6  12  18  24  30  36  42  48  54  60
   7  14  21  28  35  42  49  56  63  70
   8  16  24  32  40  48  56  64  72  80
   9  18  27  36  45  54  63  72  81  90
  10  20  30  40  50  60  70  80  90 100
```

As always, there is room for improvement: implementing Logo's higher-order primitives, more stdlib functions, adding the ability to halt execution, etc. I also thought of spinning off a 3D version, which would need 3D turtle primitives and could use an accelerated 3D graphics library such as ThreeJS. If you want to join me in creating these, don't hesitate to get in touch.

Happy holidays and happy coding!
