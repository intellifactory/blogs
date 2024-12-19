---
title: "Football with F# - Part 1"
categories: "f#,websharper,ui,datatable,fsadvent"
abstract: ""
identity: "-1,-1"
---

Two years ago, for the 2022 edition of FSharp Advent I built a little [application](https://intellifactory.com/user/jooseppi12/20221215-dart-in-fsharp-differently) that can visualize darts throw combinations, with giving us all the unique combinations. Now I find myself bridging the world of progamming and sports once again. You see, I have a problem. Is it a universal problem? No, absolutely not. My problem is my cousin is beating me in [FPL](https://fantasy.premierleague.com/). As each round passes, I'm getting farther and farther from him on the leaderboard and I feel like I have to put a stop to that. So I'm turning to my job to help me beat him

## Getting data from the FPL API

I won't go into details about how the API is structured, there are guides out there on the internet detailing the different endpoints.

We are going to rely on two of them:

- a summary endpoint
- a player data endpoint

The summary endpoint return us all the players and teams in the system with their basic information, like what is the current cost of the player and which team the given player plays for.

The player data endpoint will give us some performance stats for the given player, which we will use to calculate some metrics.

To gather this data, I'm going to use a little F# script, that I can call and cache the results locally to work from those, instead of calling to the API real time. This is fine, as we are not dealing with real time data.

The script writes all the data in the `data` folder, creating a `baseData.json` file in the root of that folder and writing all the player specific data into a file in the `player` subdirectory, naming them `$.json` where $ is the numerical id of the player.

## Some rules about how to construct a team

There are some limitations when building a team that you can use in the fantasy league. First, there is a limit on the budget - 100m. The API actually returns these as integers, that are multiplied by 10. So if a player costs 10m, the api will. 

Also there is a limitation on how you can choose players based on their position. You need to select 2 goalkeepers, 5 defenders, 5 midfielders and 3 attackers.

The final rule is that you cannot have more than 3 players from a single team.

## Building an application out of this

I'm using WebSharper's offline sitelet feature, where I can easily pregenerate the content of the html, combined with some logic to initialize some of the components on the client side. Currently I won't have any subpages, but setting up the project this way will let me expand the project in the future, like inspecting more specific player data on dedicated pages. But for now, I just need a quick way to sort through players with different criteria.

So on the server-side I'm just reading in the data from the files and deserializing into the following structure

```fsharp
type BaseData =
    {
        elements: Element []
        teams: Team []
    }

and Team =
    {
        id: int
        short_name: string
    }

and Element =
    {
        id: int
        web_name: string
        element_type: int
        now_cost: int
        team: int
    }

type Player =
    {
        history: History []
    }

and History =
    {
        total_points: float
    }
```

This gives me everything I need right now, so once I have these loaded, I can do some calculations and populate tables in the html.

In addition to the standard average calculation, I'm doing average calculations based on the last 5 and 10 rounds, so that I can look at tendencies, and also showing the variance, to get a little bit more insight on the performance of the given player. As my goal would be to pick someone with a lower variance on the same average. For this I'm using the history section from the json, which contains every rounds data that happened so far. To hold all this data together, I have created the `PlayerData` type:

```fsharp
type PlayerData =
    {
        id: int
        name: string
        value: int
        team: string
        points: float
        avg: float
        avg5: float
        avg10: float
        var: float
        element_type: int
    }

    member this.ToDoc() =
        Templates.MainTemplate.Players()
            .AVG(string this.avg |> Client.trimFloatString)
            .AVG5(string this.avg5 |> Client.trimFloatString)
            .AVG10(string this.avg10 |> Client.trimFloatString)
            .VAR(string this.var |> Client.trimFloatString)
            .PTS(string this.points)
            .Team(this.team)
            .Value(string this.value)
            .Name(this.name)
            .Type(string this.element_type)
            .ID(string this.id)
            .Doc()
```

Once we have the players stats calculated, we render them on the server-side into our markup.

```fsharp
Templates.MainTemplate.MainForm()
    .GKPlayers(
        goalkeepers
        |> List.map (fun gk ->
            gk.ToDoc()
        )
    )
```

## Initializing items on the client side

To add sorting/searching functionalities to the tables, I'm using [DataTables](https://datatables.net/) - and for this experiment, I'm just going to inline the calls to the library.

```fsharp
type DT =
    [<Inline "$this.on('click', $sel, $callback)">]
    member this.onClick (sel: string) (callback : Dom.Event -> unit) = X<unit>

[<Inline "new $global.DataTable($x)">]
let initializeTable (x: string) = X<DT>
```

From now on, I'm going to focus on just the goalkeepers section, but the other 3 roles are handled the same way.

In our markup, we have the goalkeepers table tagged with the `gk` id, so we are calling the above defined function with that

```fsharp
let dtgk = initializeTable "#gk"
```

The returned object can be used to attach event handlers through the library to the buttons in the table, which will be used to select the players for our team. So we are going to attach our event handler, the following way:

```fsharp
dtgk.onClick ".playerAdder" (fun ev -> onclickAddPlayer (ev.Target |> As<Dom.Element>) |> Async.Start)
```

If we take a look inside the onclickAddPlayer function, it looks like this:

```fsharp
let onclickAddPlayer (el: Dom.Element) =
    async {
        let tr = el.ParentElement.ParentElement
        let elementType = tr.GetAttribute("data-playertype") |> int
        let (var, max) = configuration.[elementType]
        if var.Value.Length < max then
            let playerName = tr.GetAttribute("data-playername")
            let playerValue = tr.GetAttribute("data-playervalue") |> int
            let playerTeam = tr.GetAttribute("data-playerteam")
            let playerId = tr.GetAttribute("data-playerid") |> int
            let playerAvg = tr.GetAttribute("data-playeravg") |> float
            let! cta = View.GetAsync currentTeamAllocation
            let! totalValue = View.GetAsync currentTotalValue
            let! playersInRole = View.GetAsync var.View
            if playersInRole |> List.exists (fun p -> playerId = p.Id) then
                "This player is already part of your team"
                |> JS.Alert
            elif cta |> List.contains playerTeam then
                sprintf "You already have the maximum number of players selected from %s" playerTeam
                |> JS.Alert
            elif totalValue + playerValue > MAX_VALUE then
                "The selected player would put you above the budget limit"
                |> JS.Alert
            else    
                var.Update (fun players -> List.append players [SelectedPlayer.Create playerName playerValue playerTeam playerId playerAvg])
        else
            getTypeString elementType
            |> sprintf "You already have the maximum number of %s selected"
            |> JS.Alert
    }
```

Let's break this down a little bit:

Each row in the table has attributes on them, these will be stored by us as we need to use these for validating the picks. The following attributes we have:

- id, to prevent selecting the same player multiple times
- team, to prevent selecting more than 3 players from the same team
- value, to prevent going over the set budget
- name, used for showing the name of the player in the selection
- avg, to show the average points scored by that player in the selection
- additionally we store the player's role, for helping with generalizing the logic for the onclick handler

So the `GetAttribute` functions are used to extract this data on the click event handler.

```fsharp
let elementType = tr.GetAttribute("data-playertype") |> int
let (var, max) = configuration.[elementType]
let! playersInRole = View.GetAsync var.View
if var.Value.Length < max then
```

The `configuration` is just a map that holds a tuple of the reactive `Var` and the `max` value for each role. The `max` value is the maximum number of players we can select from the given role, which we in turn use to check if we have already reached 

```fsharp
type SelectedPlayer = 
    {
        Name: string
        Value: int
        Team: string
        Id: int
        Avg: float
    }
```

Let's focus on these two lines next:

```fsharp
let! cta = View.GetAsync currentTeamAllocation
let! totalValue = View.GetAsync currentTotalValue
```

`currentTeamAllocation` and `currentTotalValue` are both views, that are built from the selected players list (which we store separately for each role). `currentTeamAllocation` returns a list of teams, from which we cannot pick anymore and `currentTotalValue` returns how much we have spent so far from our budget.

```fsharp
 // Cannot exceed 1000
 let currentTotalValue =
     View.Do {
         let! gks = gks.View
         let! defs = defs.View
         let! mids = mids.View
         let! atts = atts.View
         return (List.sumBy (fun x -> x.Value) (List.concat [gks;defs;mids;atts]))
     }

 // We can only have 3 players at maximum from each team
 let currentTeamAllocation =
     View.Do {
         let! gks = gks.View
         let! defs = defs.View
         let! mids = mids.View
         let! atts = atts.View
         return (List.groupBy (fun x -> x.Team) (List.concat [gks;defs;mids;atts]) |> List.choose (fun (t, items) -> if items.Length = 3 then Some t else None))
     }
```

So we use this to check if the player selection is still valid:

```fsharp
if playersInRole |> List.exists (fun p -> playerId = p.Id) then
    "This player is already part of your team"
    |> JS.Alert
elif cta |> List.contains playerTeam then
    sprintf "You already have the maximum number of players selected from %s" playerTeam
    |> JS.Alert
elif totalValue + playerValue > MAX_VALUE then
    "The selected player would put you above the budget limit"
    |> JS.Alert
```

If we passed every criteria, we are just updating our reactive var with the newly selected player:

```fsharp
var.Update (fun players -> List.append players [SelectedPlayer.Create playerName playerValue playerTeam playerId playerAvg])
```

Initializing the event handlers and the data tables is happening in the `Main` function, which we call on the server side like this:

```fsharp
Templates.MainTemplate.MainForm()
    .ClientInit(fun (el: JavaScript.Dom.Element) -> Client.Main())
```

```html
<div class="body" ws-template="MainForm">
    <div class="playerList" ws-onafterrender="ClientInit">
```

`ClientInit` here uses a special event handler, which is called `onafterrender`. This runs once the WS templating engine inserted the template into the DOM, so it's a perfect place to call things that are initialized once they are in the DOM.

Let's return to the selected players for a moment:

```fsharp
type SelectedPlayer = 
    {
        Name: string
        Value: int
        Team: string
        Id: int
        Avg: float
    }

    member this.ToDoc(v: Var<SelectedPlayer list>) =
        Templates.MainTemplate.PlayerData()
            .Name(this.Name)
            .Value(string this.Value)
            .Team(this.Team)
            .AVG(string this.Avg |> trimFloatString)
            .RemovePlayer(fun _ -> v.Update (fun players -> List.filter (fun x -> x.Id <> this.Id) players))
            .Doc()
```

This type also implements a `ToDoc` function, which we use to convert the reactive `Var` holding the selected players for a given role with the function below.

```fsharp
let SelectedPlayer (v: Var<SelectedPlayer list>) =
    v.View
    |> Doc.BindView(fun players ->
        match players with
        | [] -> text "None selected"
        | players -> players |> List.map(fun x -> x.ToDoc(v)) |> Doc.Concat
    )

let SelectedGKs () = SelectedPlayer gks
```

Then we use the `SelectedGKs` function to tell the generator, that the content for that part of the template will be generated by this function. We are using the `client` helper here, which bridges the context between client-side and server-side contexts.

```fsharp
Templates.MainTemplate.MainForm()
    .Goalkeepers(client <@Client.SelectedGKs()@>)
```

We have some other small bits here and there initialized so that we can switch between tabs instead of showing all 4 tables at the same, but I would like to return to these in a followup post soon.

The current state of the application can be found deployed [here](https://jooseppi12.github.io/fsadvent2024) and the source code [here](https://github.com/jooseppi12/fsadvent2024).

## Conclusion

Will this make me a better player? Hopefully. Will it make me feel better about my choices on the attempt to beat my cousin? I definitely think so. I'll come back with a 2nd part soon, with improving our application. Adding some functionalities like using localstorage or importing the current team I'm using in the league. Or maybe charting a given player's previous seasons, to look at form changes across the season. Also polishing the UI (and let's be honest, it needs it). :)

Thanks Sergey for organizing F# Advent every year. It's always a blast seeing what others are up to in the community. Happy holidays everyone!