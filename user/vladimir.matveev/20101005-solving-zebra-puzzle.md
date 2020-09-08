---
title: "Solving Zebra Puzzle"
categories: "f#,fp"
abstract: ""
identity: "1041,74622"
---
[Zebra puzzle](https://en.wikipedia.org/wiki/Zebra_Puzzle) (often referred as Einstein's problem) is a well-known class of logic puzzles where you need to reconstruct the missing parts from the set of known facts. In university I've solved similar puzzles with Prolog, but I've never tried to do it with F#... Time to act with justice.

First of all, I'd like to remind the rules (this is one of many versions but core principles remain the same):

 * There are 5 houses (along the street) in 5 different colors: blue, green, red, white and yellow.
 * In each house lives a person of a different nationality: Brit, Dane, German, Norwegian and Swede.
 * These 5 owners drink a certain beverage: beer, coffee, milk, tea and water, smoke a certain brand of cigar: Blue Master, Dunhill, Pall Mall, Prince and blend, and keep a certain pet:cat, bird, dog, fish and horse.
 * No owners have the same pet, smoke the same brand of cigar, or drink the same beverage.

Besides these facts there are several hints:

 * The Brit lives in a red house.
 * The Swede keeps dogs as pets.
 * The Dane drinks tea.
 * The green house is on the left of the white house (next to it).
 * The green house owner drinks coffee.
 * The person who smokes Pall Mall rears birds.
 * The owner of the yellow house smokes Dunhill.
 * The man living in the house right in the center drinks milk.
 * The Norwegian lives in the first house.
 * The man who smokes blend lives next to the one who keeps cats.
 * The man who keeps horses lives next to the man who smokes Dunhill.
 * The owner who smokes Blue Master drinks beer.
 * The German smokes Prince.
 * The Norwegian lives next to the blue house.
 * The man who smokes blend has a neighbor who drinks water.

Question: who owns the fish?

Well, first intention is usually: forget about complex solutions and use brute force. Very bad idea indeed, especially if we try to calculate amount of possible combinations: `5*5*5*5*5 = 3125` - this is total amount of all existing combinations of persons' attributes we need to check; then from this set we need to select permutations of size 5 (representing the street). [Article in Wiki](https://en.wikipedia.org/wiki/Permutation#In_combinatorics) gives us necessary formula to calculate total amount of permutations: `(n! / (n - k)!)`, after substitution of formal parameters with real values we get the result - `297 070 617 187 575 000`. Now it is clear that brute force is not the best thing we can invent...

Let's have look at hints. We can split then into two groups: first that uses street related information and second - that doesn't. Maybe if we convert hint groups to filters and apply them subsequently, second, then first - then on the second step source set will be smaller thus decreasing amount of combinations.

```fsharp
open System

type House       =  Blue      = 1| Green  = 2| Red     = 4| White    = 8| Yellow = 16 
type Nationality =  Brit      = 1| Dane   = 2| German  = 4| Norwegian= 8| Swede  = 16
type Beverage    =  Beer      = 1| Coffee = 2| Milk    = 4| Tea      = 8| Water  = 16
type Cigar       =  BlueMaster= 1| Dunhill= 2| PallMall= 4| Prince   = 8| Blend  = 16
type Pet         =  Cat       = 1| Bird   = 2| Dog     = 4| Fish     = 8| Horse  = 16

type Person = House * Nationality * Beverage * Cigar  * Pet

let values<'T when 'T : enum<int>> = Enum.GetValues(typeof<'T>) :?> 'T array

let house       v  ((h, _, _, _, _) : Person) = h = v
let nationality v  ((_, n, _, _, _) : Person) = n = v
let beverage    v  ((_, _, b, _, _) : Person) = b = v
let cigar       v  ((_, _, _, c, _) : Person) = c = v
let pet         v  ((_, _, _, _, p) : Person) = p = v

let (<&>) f1 f2 = fun p -> 
    let a = f1 p 
    let b = f2 p
    a = b

let hint1  = nationality Nationality.Brit   <&> house House.Red
let hint2  = nationality Nationality.Swede  <&> pet Pet.Dog
let hint3  = nationality Nationality.Dane   <&> beverage Beverage.Tea
let hint5  = house House.Green              <&> beverage Beverage.Coffee
let hint6  = cigar Cigar.PallMall           <&> pet Pet.Bird
let hint7  = house House.Yellow             <&> cigar Cigar.Dunhill
let hint12 = cigar Cigar.BlueMaster         <&> beverage Beverage.Beer
let hint13 = nationality Nationality.German <&> cigar Cigar.Prince

let (<+>) f1 f2 = fun p -> f1 p && f2 p

let tests = 
    [ for house         in values<House>        do
      for nationality   in values<Nationality>  do
      for beverage      in values<Beverage>     do
      for cigar         in values<Cigar>        do
      for pet           in values<Pet>          do
        yield (house, nationality, beverage, cigar, pet) ]

tests 
    |> List.filter(
        hint1 <+> 
        hint2 <+> 
        hint3 <+> 
        hint5 <+> 
        hint6 <+> 
        hint7 <+> 
        hint12 <+> 
        hint13
        )
    |> (List.length >> printfn "length %d")

// length: 78
```

Some notes:

 * `house`, `nationality`, `beverage`, `cigar`, `pet` are helper functions for creating hints.

 * The `<&>` operator applies two hints and then combines its results. Tricky moment here is that both arguments should be evaluated to make final decision. I.e. if hints are `(cigar PallMall) & (house Yellow)`, then for successful result person being tested should either satisfty both conditions or non-one: if person smokes Pallmall, then it should live in yellow house, and the opposite - if it lives in yellow house - it should smoke Pallmall. That's why we cannot use builtin `&` operator - it can reject potentially positive results.

After test execution is completed of our code we see that amount of source combinations is now 78. Nice! However, it is not the final improvement. When generating street structure we may notice that combination with duplicated values of some particular attribute is incorrect: two different persons cannot live in blue house at the same time. So during generation we can filter out knowingly incorrect options thus reducing number of combinations to be checked.

Straightforward way to denote the whole combination of attributes at once - bitmask. Value of type `int` shall be split into 5 groups, 5 bits in each group. During generation we will pass the mask with all involved attributes. On every step candidate value will be matched against mask and possibly rejected (if one or more property was already used on this generation stage).

```fsharp
open System

type House       =  Blue      = 1| Green  = 2| Red     = 4| White    = 8| Yellow = 16 
type Nationality =  Brit      = 1| Dane   = 2| German  = 4| Norwegian= 8| Swede  = 16
type Beverage    =  Beer      = 1| Coffee = 2| Milk    = 4| Tea      = 8| Water  = 16
type Cigar       =  BlueMaster= 1| Dunhill= 2| PallMall= 4| Prince   = 8| Blend  = 16
type Pet         =  Cat       = 1| Bird   = 2| Dog     = 4| Fish     = 8| Horse  = 16

type Person = House * Nationality * Beverage * Cigar  * Pet

let values<'T when 'T : enum<int>> = Enum.GetValues(typeof<'T>) :?> 'T array

let inline mask ((h, n, b, c, p) : Person) = 
    (int h)        ||| 
    (int n <<< 5)  ||| 
    (int b <<< 10) ||| 
    (int c <<< 15) ||| 
    (int p <<< 20)

let house       v  ((h, _, _, _, _) : Person) = h = v
let nationality v  ((_, n, _, _, _) : Person) = n = v
let beverage    v  ((_, _, b, _, _) : Person) = b = v
let cigar       v  ((_, _, _, c, _) : Person) = c = v
let pet         v  ((_, _, _, _, p) : Person) = p = v

let (<&>) f1 f2 = fun p -> 
    let a = f1 p 
    let b = f2 p
    a = b

let hint1  = nationality Nationality.Brit   <&> house House.Red
let hint2  = nationality Nationality.Swede  <&> pet Pet.Dog
let hint3  = nationality Nationality.Dane   <&> beverage Beverage.Tea
let hint5  = house House.Green              <&> beverage Beverage.Coffee
let hint6  = cigar Cigar.PallMall           <&> pet Pet.Bird
let hint7  = house House.Yellow             <&> cigar Cigar.Dunhill
let hint12 = cigar Cigar.BlueMaster         <&> beverage Beverage.Beer
let hint13 = nationality Nationality.German <&> cigar Cigar.Prince

let (<+>) f1 f2 = fun p -> f1 p && f2 p

let permutations k l =
    let rec impl n flags = seq { 
        if n = 0 then yield []
        else
            for value in l do
                if flags &&& (mask value) = 0 then
                    for suffix in impl (n - 1) (flags ||| mask value) do
                        yield value::suffix
        }
    impl k 0

let hint4 persons =
    let greenIndex = List.findIndex (house House.Green) persons
    let whiteIndex = List.findIndex (house House.White) persons
    (whiteIndex - greenIndex) = 1

let hint8 persons = 
    let milkIndex = List.findIndex (beverage Beverage.Milk) persons
    milkIndex = 2
let hint9 persons = 
    let norwegianIndex = List.findIndex (nationality Nationality.Norwegian) persons
    norwegianIndex = 0

let neighbour h1 h2 persons = 
    let index1 = List.findIndex h1 persons 
    let index2 = List.findIndex h2 persons 
    abs (index1 - index2) = 1

let hint10 = neighbour (cigar Cigar.Blend)   (pet Pet.Cat)
let hint11 = neighbour (cigar Cigar.Dunhill) (pet Pet.Horse)
let hint14 = neighbour (nationality Nationality.Norwegian) (house House.Blue)
let hint15 = neighbour (cigar Cigar.Blend) (beverage Beverage.Water)

let tests = 
    [ for house         in values<House>        do
      for nationality   in values<Nationality>  do
      for beverage      in values<Beverage>     do
      for cigar         in values<Cigar>        do
      for pet           in values<Pet>          do
        yield (house, nationality, beverage, cigar, pet) ]

let sw = System.Diagnostics.Stopwatch.StartNew()
tests 
    |> List.filter(
        hint1 <+> 
        hint2 <+> 
        hint3 <+> 
        hint5 <+> 
        hint6 <+> 
        hint7 <+> 
        hint12 <+> 
        hint13
        )
    |> permutations 5
    |> Seq.filter (
        hint4 <+> 
        hint8 <+> 
        hint9 <+> 
        hint10 <+> 
        hint11 <+> 
        hint14 <+> 
        hint15
        )
    |> Seq.toList
    |> (fun [x] -> printfn "%A" x)
sw.Stop()
printfn "completed in %A" sw.Elapsed
```

```
[(Yellow, Norwegian, Water, Dunhill, Cat); (Blue, Dane, Tea, Blend, Horse);
 (Red, Brit, Milk, PallMall, Bird); (Green, German, Coffee, Prince, Fish);
 (White, Swede, Beer, BlueMaster, Dog)]
completed in 00:00:00.5034056
```

Mystery is solved.
