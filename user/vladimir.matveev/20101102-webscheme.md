---
title: "WebScheme"
categories: "websharper,f#,fp"
abstract: "Scheme is remarkable language; it combines syntactic simplicity and elegance with exceptional expressiveness and power. Implementations of Scheme exist in many languages, and though I’m not the first who used F# for this solving task (shame on me) I still can be the first man who have made it with WebSharper."
identity: "1040,74621"
---
Generally speaking if you abstract away from the visual part, WebSharper applications are not so different from usual F# libraries. Knowing this fact we can make our life much easier, and we can get away with simply: creating and testing basic components as a part of console application and later converting them to a WebSharper library. So, shall we begin?

Our little Scheme (an actual subset of real Scheme) interpreter will consist of three parts:

 * **Lexer** - transforms source text (sequence of chars) into a sequence of tokens
 * **Parser** - produces language expressions based on a given token stream
 * **Runtime** – generic name for the remaining part, that includes an execution engine and a standard library (in our case few builtin functions)

As I already mentioned, all these pieces are UI agnostic so they can be put into separate assembly that will be referenced from both the test console host and the front end WebSharper application.

Lexer comes first. Step 1: define `Token` type and implement lexical analyzer:

```fsharp
type Token = 
    | Identifier of string
    | Quote      of Token
    | Number     of int
    | Boolean    of bool
    | String     of string
    | List       of Token list
module Lexer = 
     let private toString (chars : char list) = 
        chars |> List.fold (fun acc s -> string s + acc) ""

    let isDigit c = c >= '0' && c <= '9' 
    let isWhiteSpace c = c = ' ' || c = '\t' || c = '\r' || c = '\n'

    let getTokens (source : string) = 
        let rec getToken inList = function
            | [] -> 
                if inList 
                then failwith "syntax error (missing close bracket)" 
                else None, []
            | ')'::rest -> 
                if inList 
                then None, rest 
                else failwith "syntax error (missing open bracket)" 
            | '('::rest ->
                let l, rest = getList true rest
                Some (List l), rest
            | '\''::rest ->
                let t, rest = getToken inList rest
                match t with
                | Some t -> Some (Quote t), rest
                | None -> failwith "syntax error (invalid quotation)"
            | c::rest when isWhiteSpace c -> getToken inList rest
            | '-'::((c::_) as rest) when isDigit c ->
                let num, rest = getNumber rest
                Some (Number -num), rest
            | (c::_) as rest when isDigit c ->
                let num, rest = getNumber rest
                Some (Number num), rest
            | '#'::'t'::rest ->
                Some (Boolean true), rest                
            | '#'::'f'::rest ->
                Some (Boolean false), rest                
            | '"'::rest ->
                let s, rest = getString rest
                Some (String s), rest
            | rest ->
                let identifier, rest = getIdentifier rest
                Some (Identifier identifier), rest
        
        and getList counter l = 
            let rec impl acc (rest : char list) l = 
                match getToken counter l with
                | Some t, rest -> impl (t::acc) [] rest
                | None, rest -> List.rev acc, rest
            impl [] [] l
        
        and getNumber l = 
            let rec impl acc = function
                | c::rest when isDigit c ->
                    let value = int c - int '0'
                    impl (acc * 10 + value) rest
                | rest -> acc, rest
            impl 0 l
        
        and getString l = 
            let rec impl acc = function
                | [] -> failwith "syntax error (malformed string)"
                | '\\'::'"'::rest -> impl ('"'::acc) rest
                | '"'::rest -> (toString acc, rest)
                | x::rest -> impl (x::acc) rest
            impl [] l

        and getIdentifier l = 
            let rec impl acc = function
                | [] -> toString acc, []
                | (('('::_) as rest)
                | ((')'::_) as rest) -> toString acc, rest
                | c::rest when isWhiteSpace c -> toString acc, rest
                | c::rest -> impl (c::acc) rest
            impl [] l
        
        let rec impl l = 
            [
                match getToken false l with
                | Some t, r -> 
                    yield t
                    match r with
                    | [] -> ()
                    | rest -> yield! impl rest
                | None, _ -> ()
            ]
        source 
            |> fun s -> s.ToCharArray()
            |> List.ofSeq
            |> impl
```

 * For simplicity our implementation will support only integers (floats are out of the boat for now)
 * Lexer results are slightly more than just conversion from chars to tokens, final sequence is already folded on brackets (lists are represented not as sequence of tokens like `LEFT_BRACKET`, `NUMBER`...`RIGHT_BRACKET`, but as one token **List** containing nested tokens with brackets omitted).
 * `getTokens` internally consists of several smaller functions: `getList` repeatadly reads a sequence of tokens containing within '(' and ')' and `getToken` tries to read next the token. If this attempt is successful - it returns the token and the remaining part of the stream.

Watch the doors, next stop: **parser**. With pattern matching its creation becomes a straightforward task:

```fsharp
type Expr = 
    | Bool      of bool
    | Number    of int
    | String    of string
    | Atom      of string
    | List      of Expr list
    | Var       of string
    | Apply     of Expr * Expr list
    | Lambda    of string list * Expr
    | If        of Expr * Expr * Expr option
    | Assign    of string * Expr
    | Cond      of (Expr * Expr) list * Expr option
    | Let       of (string * Expr) list * Expr
    | LetRec    of (string * Expr) list * Expr
    | Sequence  of Expr list
    | CallCC    of Expr

module Parser = 
    let rec private parse tokens = 
        match tokens with
        | Token.Identifier s -> Expr.Var s
        | Token.Number n -> Expr.Number n
        | Token.Boolean b -> Expr.Bool b 
        | Token.String s -> Expr.String s
        | Token.Quote t -> parseQuote t
        | Token.List(tokens) -> parseList tokens

    and private parseQuote = function
        | Token.Number n -> Expr.Number n
        | Token.Boolean b -> Expr.Bool b
        | Token.String s -> Expr.String s
        | Token.Quote _ -> failwith "quoted quotes not supported yet"
        | Token.Identifier id -> Expr.Atom id
        | Token.List l -> l |> List.map parseQuote |> Expr.List
    
    and private parseCond tokens = 
        let rec impl acc = function
            | [] -> List.rev acc, None 
            | [Token.List([Token.Identifier "else"; body])] -> 
                List.rev acc, Some(parse body)
            | Token.List([cond; body])::rest -> 
                let cond = parse cond
                let body = parse body
                impl ((cond, body)::acc) rest
            | x -> failwith "invalid 'cond' syntax"
        impl [] tokens
    
    and private parseList = function
        | Token.Identifier "begin"::rest -> 
            rest 
            |> List.map parse 
            |> Sequence        
        | Token.Identifier "cond"::body -> 
            parseCond body 
            |> Expr.Cond
        | [Token.Identifier "callcc"; body] ->
            parse body
            |> Expr.CallCC
        | Token.Identifier "lambda"::Token.List(formals)::body -> 
            let formals = parseFormals formals
            Lambda(formals, Sequence(parseMain body))
        | Token.Identifier "let"::Token.List(bindings)::body -> 
            parseLet bindings body Let
        | Token.Identifier "letrec"::Token.List(bindings)::body -> 
            parseLet bindings body LetRec
        | [Token.Identifier "if"; cond; ifTrue] -> 
            let cond = parse cond
            let ifTrue = parse ifTrue
            Expr.If(cond, ifTrue, None)
        | [Token.Identifier "if"; cond; ifTrue; ifFalse] ->
            let cond = parse cond
            let ifTrue = parse ifTrue
            let ifFalse = parse ifFalse
            Expr.If(cond, ifTrue, Some ifFalse)        
        | [Token.Identifier "set!"; Token.Identifier(var); value] -> 
            Expr.Assign(var, parse value)
        | proc::args -> 
            Expr.Apply(parse proc, List.map parse args)
        | x -> 
            failwith "unrecognized token sequence"
    and private parseFormals formals = 
        formals 
            |> List.map parse 
            |> List.map(function 
                | Expr.Var s -> s 
                | x -> failwith "formal parameter should be var"
                )
    and private parseLet bindings body f =
        let bindings = 
            bindings
            |> List.map(function 
                | Token.List([Token.Identifier name; value]) -> name, parse value 
                | x -> failwith "invalid binding structure"
                )
        let body = parseMain body
        f(bindings, Sequence body)
     
    and parseMain tokens = 
        [
            match tokens with
            | Token.List
                (
                [Token.Identifier "define"; Token.Identifier var; body]
                )::xs ->
                yield Assign(var, parse body)
                yield! parseMain xs
            | Token.List
                (
                Token.Identifier "define"
                ::Token.List(Token.Identifier var::formals)
                ::body
                )::xs ->
                let f = parseFormals formals
                let b = parseMain body |> Sequence
                yield Assign(var ,Lambda(f, b)
                yield! parseMain xs
            | x::xs -> 
                yield parse x
                yield! parseMain xs
            | [] -> ()
        ]
```

 * `Bool`, `Number` and `String` corresponds to literals found in source text
 * `List` contains values wrapped in a quoted list in the source such as `'(...)`
 * `Apply` denotes function application. First parameter should be a function, second - list of arguments
 * `If` - simple if-then expression with optional 'else' part
 * `Cond` - similar to if-elseif-elseif-else expression where 'else' can be omitted
 * `Assign` - corresponds to a 'setq' expression: it evaluates the value and stores it in the environment under the given name
 * `Var` - references a variable within the environment
 * `Let` and `LetRec` contains list of pairs (bindings from name to value) and `body`. Bindings are evaluated and stored in a newly created environment. Afterwards this environment is used to evaluate the `body`. Difference between `let` and `letrec` lies in fact that `letrec` bindings can be recursively defined (refer to themselves in their definition).
 * `Sequence` - just a list of expressions
 * `CallCC` - Special higher-order magic: argument should be procedure with one parameter. Runtime will evaluate this procedure with a current continuation as a parameter value


Final step: runtime. It is basically the heart of our implementation and thus I'd like to devote slightly more attention to this stage.

The execution environment will be represented as list of Frames, every frame stores its own set of values. Name lookup is performed in a hierarchical manner: if name is not found in a child frame -> try search in parent.

```fsharp
    type EnvFrame = Dictionary<string, obj> // System.Collections.Generic...
    type Env = EnvFrame list

    let private tryFindFrame name (d : EnvFrame) =
        if d.ContainsKey name 
        then Some (d, d.[name])
        else None
    
    let makeEnv prevEnv content  =
        let d = new Dictionary<_, _>()
        for (name, value) in content do
            d.Add(name, value)
        d::prevEnv 

    let private get name (env : Env) = 
        match List.tryPick (tryFindFrame name) env with
        | Some(_, value) -> value
        | None -> failwith ("name '" + name + "' not found")

    let private set name value (env : Env) = 
        match List.tryPick (tryFindFrame name) env with
        | Some (frame, _) -> frame.[name] <- value
        | None ->
            match env with
            | h::_ -> 
                h.[name] <- value
            | [] -> failwith "empty env."
```

Programming in Scheme heavily relies on recursion, but unfortunately our target platform (JavaScript) doesn't support optimization of tail calls. As a solution we can utilize 'trampolines' - a technique when every step returns delayed next step, so recursion is automagically converted to using loops. Main characters:

 * `Step` – a function that returns the next Step as its evaluation result.
 * `Cont` - represents the computation process at given point
 * `Function` - function representation, accepts argument values and continuation to proceed with execution.


```fsharp
    type Step = Step of (unit -> Step) option // None - stop 
    type Cont = obj -> Step
    type Function = obj list -> Cont -> Step
```

This is top level evaluation loop. Here we start the evaluation of some specified expression with respect to the given environment. `finalCont` will receive final result of computation and store it in a local variable.

```fsharp
    let evaluate env expr = 
        let result = ref null
        let ok = ref true
        let finalCont = fun o -> result := o; Step None
        
        let step = ref (evaluateExpression expr env finalCont)
        
        while !ok do
            let (Step s) = !step
            match s with
            | Some f -> // has next step
                step := f()
            | None ->   // completed!
                ok := false
        !result
```

Let's examine the computation process on the sample of a simple `(+)` function. In an ideal world the expression `x + y` (where `x` and `y` can be complex expressions themselves) will be transformed to the following sequence of continuations:

```fsharp
let rec eval (e : Expression) (k : Continuation) = ...
// finalCont - continuation that will accept the result
eval a (fun valueOfA -> 
    eval b (fun valueOfB -> 
        finalCont (valueOfA + valueOfB)))
```

This will work in F# but not in JavaScript (due to lack of tail call optimization). So instead of running subsequent computations in continuations we'll delay evaluation by wrapping them in thunks.

```fsharp
let thunk f = ... something that wraps f with fun () -(fun () -> f())

eval a (fun valueOfA -> // k1
    thunk(fun () -> eval b (fun valueOfB -> // k2 
        thunk(fun () -> finalCont (valueOfA + valueOfB)))))
```

Now result of every continuation will not be the value itself but rather a function that returns value. This is how the trampoline works: first you jump (by calling the eval a -> forward to continuation k1) then you land (because k1 returns delayed computation). The evaluation loop makes one more iteration and you jump and land again (running eval b -> continuation k2 -> delayed computation). We repeat this process until evaluation is completed. Everything works nicely but explicit managing of continuation looks awful from the internal sense of constructive laziness – too much manual work. Can we do something to make continuation passing ambient? Surely yes! Computation expressions to the rescue.

```fsharp
module Scheme = 
    open System.Collections.Generic

    type Step = Step of (unit -> Step) option
    type Cont = obj -> Step
    type Function = obj list -> Cont -> Step
    
    type EnvFrame = Dictionary<string, obj>
    type Env = EnvFrame list

    let private tryFindFrame name (d : EnvFrame) =
        if d.ContainsKey name 
        then Some (d, d.[name])
        else None
    
    let makeEnv prevEnv content  =
        let d = new Dictionary<_, _>()
        for (name, value) in content do
            d.Add(name, value)
        d::prevEnv 

    let private get name (env : Env) = 
        match List.tryPick (tryFindFrame name) env with
        | Some(_, value) -> value
        | None -> failwith ("name '" + name + "' not found")

    let private set name value (env : Env) = 
        match List.tryPick (tryFindFrame name) env with
        | Some (frame, _) -> frame.[name] <- value
        | None ->
            match env with
            | h::_ -> 
                h.[name] <- value
            | [] -> failwith "empty env."
    
    let thunk f = f |> Some |> Step
    
    type ContBuilder () = 

        member inline this.Return (v) = fun (k : Cont) -> thunk (fun() -> k v)

        member inline this.ReturnFrom (v : Cont -> Step ) = v
        
        member this.Bind(c : Cont -> Step, f : obj -> Cont -> Step)  = 
            fun (rk : Cont) ->
                let c1 (v : obj) = 
                    let c2 = f v
                    thunk (fun() -> c2 rk)
                thunk(fun() -> c c1)

    let K = ContBuilder()
    let private callCC (f : Function) : Cont -> Step =
        fun (extK : Cont) ->
            let exit : Function = fun [arg] _ -> K.Return arg extK
            f [exit] extK
    
    let rec private evaluateExpression expr (env : Env) = 
        match expr with
            | Expr.List(values) -> 
                let rec impl acc l = 
                    match l with
                    | [] ->  K { return List.rev acc }
                    | v::rest -> K { 
                        let! res = evaluateExpression v env
                        return! impl (res::acc) rest 
                        }
                impl [] values
            | Expr.Atom atom -> 
                K { return atom }
            | Expr.Bool v -> 
                K { return v }
            | Expr.Number n -> 
                K { return n }
            | Expr.String s -> 
                K { return s }
            | Expr.CallCC proc ->
                K { 
                    let! func = evaluateExpression proc env
                    match func with
                    | :? Function as f -> return! callCC f
                    | x -> return! failwith "callCC argument should be function"
                }
            | Expr.Assign (name, v) -> 
                K {
                    let! value = evaluateExpression v env
                    set name value env
                    return null
                }
            | Expr.Cond (clauses, elseCase) ->
                let rec evalCond = function
                    | (cond, body)::xs, _ -> 
                        K { 
                            let! v = evaluateExpression cond env
                            if unbox v 
                            then return! evaluateExpression body env
                            else return! evalCond(xs, elseCase)
                        }
                    | [], Some(c) -> K { return! evaluateExpression c env}
                    | _ -> K { return null }
                K { return! evalCond (clauses, elseCase)}
            | Expr.Apply(f, args) ->
                K { 
                    let! boxedFunc = evaluateExpression f env
                    match boxedFunc with
                    | :? Function as f ->
                        let rec evalArgs acc = function
                            | [] -> K { return! f (List.rev acc) }
                            | x::xs ->  K { 
                                let! value = evaluateExpression x env
                                return! evalArgs (value::acc) xs
                                }
                        return! evalArgs [] args
                    | _ -> return! failwith "function expected"    
                }
            | Expr.If (cond, ifTrue, ifFalse) ->
                K { 
                    let! v = evaluateExpression cond env
                    if unbox v 
                    then return! evaluateExpression ifTrue env
                    else 
                        match ifFalse with
                        | Some f -> return! evaluateExpression f env
                        | None -> return null
                }
            | Expr.Lambda(formals, body) ->
                let apply args= 
                    let newEnv = 
                        List.zip formals args
                        |> makeEnv env
                    K { return! evaluateExpression body newEnv}
                K { return apply }
            | Expr.Let(bindings, body) ->
                let rec evalBindings acc = function 
                    | [] -> 
                        let newEnv = 
                            acc 
                            |> List.rev
                            |> makeEnv env
                        K { return! evaluateExpression body newEnv }
                    | (name, x)::xs -> K { 
                        let! v = evaluateExpression x env
                        return! evalBindings ((name, v)::acc) xs
                        }
                K { return! evalBindings [] bindings }
            | Expr.LetRec(bindings, body) ->
                let newEnv = 
                    bindings
                    |> List.map (fun (name, _) -> name, null)
                    |> makeEnv env
                let rec evalBindings acc = function 
                    | [] -> K { return! evaluateExpression body newEnv }
                    | (n, x)::xs -> K { 
                        let! v = evaluateExpression x newEnv
                        set n v newEnv
                        return! evalBindings (v::acc) xs
                        }
                K { return! evalBindings [] bindings }
            | Expr.Sequence(statements) ->
                let rec impl s = 
                    match s with
                    | [] -> K { return null } 
                    | [x] -> K { return! evaluateExpression x env }
                    | x::xs -> K { 
                        let! _ = evaluateExpression x env
                        return! impl xs
                        }
                K { return! impl statements } 
            | Expr.Var(name) ->
                K { return get name env }
    
    let evaluate env expr = 
        let result = ref null
        let ok = ref true
        let finalCont = fun o -> result := o; Step None
    
        let step = ref (evaluateExpression expr env finalCont)
        
        while !ok do
            let (Step s) = !step
            match s with
            | Some f ->
                step := f()
            | None ->
                ok := false
        !result
```

To make our description complete - builtin functions:

```fsharp
module Functions = 
    let K = Scheme.K

    let private pack (f : Scheme.Function) = box f
    
    let private makeListFunction f = 
        (function | [v : obj] -> f (v :?> obj list)) |> pack

    let private makeBinary op = 
        fun args -> K { 
            return args 
                |> List.map unbox
                |> List.reduce op
        } |> pack   

    let builtins () = 
        [ 
          "+", makeBinary (+)
          "-", makeBinary (-)
          "*", makeBinary (*)
          "/", makeBinary (/)
          "or", makeBinary (||)
          "<", (fun [a; b] -> K { return unbox a < unbox b}) |> pack
          "=", (function 
                    | [] | [_]-> failwith "invalid args"
                    | [x; y] -> K { return x.Equals(y) }
                    | [x; y; z] -> K { return (x.Equals(y) && y.Equals(z)) } 
                    | _ -> failwith "not supported")
               |> pack
          "zero?", (fun [x] -> K { return unbox x = 0} ) |> pack
          "null?", (function
                    | [] -> K { return true}
                    | _  -> K { return false} ) |> makeListFunction
          "car", (function | x::_ -> K { return x } ) |> makeListFunction
          "cdr", (function | _::xs -> K { return xs } ) |> makeListFunction
        ] 
        |> Scheme.makeEnv []
```

That's all folks, now time to test our creation:

![](/assets/scheme-cmd.png)

Now we have a working implementation, what should be done with it to make it accessible in a WebSharper web application:

 * Add a reference to the `IntelliFactory.WebSharper` assembly

 * Annotate assembly with the `WebSharper` attribute
    ```fsharp
    open System.Runtime.CompilerServices

    [<assembly:IntelliFactory.WebSharper.WebSharper>]
    do()
    ```

 * Put `[<IntelliFactory.WebSharper.JavaScriptAttribute>]` and `[<IntelliFactory.WebSharper.JavaScriptTypeAttribute>]` to all functions and types in our library

 * Make some coffee, everything else is taken care of.


Main part is done but we still need a fancy front end: syntax highlighting and so on. The CodeMirror library already includes syntax highlighter for Scheme, we just need to plug it in. So add a WebSharper project to our solution, remove basic WebSharper sample code and perform a few magical passes:

```fsharp
open IntelliFactory.WebSharper

type CodeMirrorResources() = 
    interface IResource with
        member this.Render(resolver, writer) = 
            Resource.RenderJavaScript(resolver.ResolveUrl("js/codemirror.js")) writer
            Resource.RenderJavaScript(resolver.ResolveUrl("js/mirrorframe.js")) writer
            Resource.RenderCss(resolver.ResolveUrl("js/docs.css")) writer

[<Require(typeof<CodeMirrorResources>)>]    
module CodeMirror = 
    
    type Editor = 
        [<Inline "$this.getCode()">]
        member this.Code : string = failwith "no dynamic invokation"

    type EditorOptions [<Inline "{}">]() =
        [<DefaultValue; Name "height">] 
        val mutable Height : string

        [<DefaultValue; Name "content">] 
        val mutable Content : string

        [<DefaultValue; Name "stylesheet">] 
        val mutable StyleSheet : string

        [<DefaultValue; Name "path">] 
        val mutable Path : string

        [<DefaultValue; Name "parserfile">] 
        val mutable ParserFile : string array

        [<DefaultValue; Name "autoMatchParens">] 
        val mutable AutoMatchParens : bool

        [<DefaultValue; Name "disableSpellcheck">] 
        val mutable DisableSpellcheck : bool

        [<DefaultValue; Name "lineNumbers">] 
        val mutable LineNumbers : bool
    
    [<Inline("new CodeMirror(CodeMirror.replace($node), $options)")>]
    let create (node : Dom.Element) (options : EditorOptions) : Editor = 
        failwith "no dynamic invokation"
```

Here we have created typed wrappers for CodeMirror editor and configuration options. Remaining part is little compared to the work we've already done. Main frontend code + formatting the evaluation result:

```fsharp
open IntelliFactory.WebSharper
open IntelliFactory.WebSharper.Html

open WebScheme

[<JavaScriptType>]
type WebSchemeUI() = 
    inherit Web.Control()

    [<JavaScript>]
    let rec toString (v : obj) = 
        match JOperators.TypeOf(v) with
        | JType.Boolean 
        | JType.Number
        | JType.String -> v.ToString()
        | JType.Function -> "<function>"
        | JType.Undefined -> "<null>"
        | JType.Object ->
            let result = 
                match v :?> obj list with
                | [] -> ""
                | l -> 
                    l 
                    |> Seq.map toString 
                    |> Seq.reduce (fun acc v -> acc + "," + v) 
            "[" + result + "]"

    [<JavaScript>]
    override this.Body = 
        let env = WebScheme.Functions.builtins () |> ref
        
        let options = 
            CodeMirror.EditorOptions(
                Height = "300px",
                StyleSheet = "js/schemecolors.css",
                ParserFile = [|"tokenizescheme.js"; "parsescheme.js"|],
                AutoMatchParens = true,
                DisableSpellcheck = true,
                LineNumbers = true,
                Path = "js/"
            )
        let text = TextArea[]
        let editor = ref None
        text |> OnAfterRender (fun n -> 
            editor := Some (CodeMirror.create n.Dom options)
            )

        let result = Div [Width "100%"; Class "output"]
        let appendResult (v : obj) cls = 
            let value = Div [ Span [Class (cls + " message")] -< [Text (toString v)]]
            result.Append(value)
            result.JQuery.ScrollTop(result.JQuery.Height()).Ignore

        let evaluate = 
            Button [Text "Evaluate"]
            |>! OnClick (fun _ _ ->
                let text = editor.Value.Value.Code
                try
                    let tokens = Lexer.getTokens text
                    let expressions = Parser.parseMain tokens
                    for e in expressions do
                        match Scheme.evaluate !env e  with
                        | null -> ()
                        | result -> appendResult result "success"
                with
                    e -> appendResult (e.ToString()) "error"
                )

        let clear = 
            Button [Text "Clear"]
            |>! OnClick (fun _ _ ->
                env := Functions.builtins()
                result.Clear()
            )
        Table [Width "100%"; Border "0.8"] -< 
        [ 
            TR [Height "300px"] -< [  TD [ text ] ]
            TR [TD [evaluate; clear] ]
            TR [Height "200px"] -< [TD [result] ]
        ]
```

Enjoy the results:

![](/assets/scheme-ui.png)

Notice that despite the large input parameter, `sumall` completed successfully. If we modify `ContBuilder` to make all evaluation eager - then the result will match our expectation:

![](/assets/scheme-ui-2.png)

Source code for this sample is available [here](https://bitbucket.org/desco/webscheme/overview). Enjoy :)
