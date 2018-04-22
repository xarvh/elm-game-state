# How to manage rich, scalable game state in Elm

In this article I want to present a practical implementation to manage the state of a rich simulation game in a purely functional, strictly typed language.

TL;DR: Describe your game state changes as:
```
type Change
   = ChangeNone
   | ChangeList (List Change)
   | ChangeEntity EntityId (WholeState -> Entity -> Entity)
   | ChangeGame (Game -> Game)
   | ChangeAddSideEffect SideEffect
```
Go through the whole tree of Changes, grouping together Changes that affect the same thing, then execute them in batches to minimize the amount of state copies needed.



## Intro

Strictly typed, purely functional programming languages are great in many ways, but when it comes to writing games they lag behind imperative languages.

Why is that?

The problem is immutability or, in the immortal words of John Carmack: “How do you shoot somebody if you can’t affect them?”
[https://youtu.be/1PhArSujR_A?t=19m23s]

In games, and especially rich simulation games, everything can affect everything else, sometimes even in ways that do NOT involve shooting.
This flat dynamic cannot be easily described with the same patterns that we use for Web apps, which are a lot more hierarchical in nature.

More in general, if we keep creating a whole copy of the game state every time we change the smallest thing we will quickly run into two problems:
1. Performance. Not even lazy copying can save us here.
2. Describing changes in our game state becomes unwieldy and clumsy, forcing major rewrites every time we need to add the smallest feature.

This is my attempt at implementing the solution that John mentions, which I used for my game Herzog Drei (https://xarvh.github.io/herzog-drei/)
I found that it scales very well and allows me to add a lot of complexity and interactions without having to modify ten different files every time.

The general ideas are:

- Use functions to describe state changes
This gives us maximum flexibility.

- Group the changes by the entity they affect before applying them
(This allows us to apply all changes without having to copy the whole state at every change)

Let’s see them in practice.



## Describing State Changes


### Think Functions


We need a function that takes the whole state of the game and a specific actor and returns the changes that the actor wants to produce.
Let’s call this the “think” function:
```
actorThink : WholeState -> actor -> Change
```

For example, to make actor A heal actor B after John Carmack shot it, calling `actorThink theWholeWorld unitA` must return a change that somehow describes “heal unitB”.


---> Think functions produce “CHANGES”


Herzog Drei has several different types of actors, so it needs a Think function for each:
```
playerThink : WholeState -> Player -> Change
unitThink : WholeState -> Unit -> Change
baseThink : WholeState -> Base -> Change
projectileThink : WholeState -> Projectile -> Change
```

If instead we are using the Entity-Component-System pattern, we need just one:
```
entityThink : WholeState -> Entity -> Change
```


### Update Functions

At its most basic, a “Change” should include a reference to what needs to be changed, and an “update” function (WholeState -> actor -> actor) that actually makes the change:
```
type Change
   = ChangePlayer PlayerId (WholeState -> Player -> Player)
   | ChangeUnit UnitId (WholeState -> Unit -> Unit)
   | ChangeBase BaseId (WholeState -> Base -> Base)
```

Or with ECS:
```
type Change
  = ChangeEntity EntityId (WholeState -> Entity -> Entity)
```


In our healing example, the Change and Update function could be:
```
updateHealUnit : Int -> WholeState -> Unit -> Unit
updateHealUnit healAmount state unit =
  { unit | health = unit.health + healAmount }
```

Using partial function application, we can express our Change as:
```
changeToHealUnit : Int -> Unit -> Change
changeToHealUnit amountToHeal unit =
  ChangeUnit unit.id (updateHealUnit amountToHeal)
```


---> Update functions mutate state; they use partial application


### Zero or More Changes


Often times Think functions will need to produce no changes, or more than one change, so we can expand the union type with a constructor for no changes and one for nesting several:
```
type Change
   = ChangeNone
   | ChangeList (List Change)
   | ChangePlayer PlayerId (WholeState -> Player -> Player)
   | ChangeUnit UnitId (WholeState -> Unit -> Unit)
   | ChangeBase BaseId (WholeState -> Base -> Base)
```

As the game grows in complexity and we break the Think functions into smaller functions, being able to nest multiple changes into one will become very handy:
```
thinkUnit : WholeState -> Unit -> Change
thinkUnit state unit =
  ChangeList
    [ thinkMove state unit
    , thinkAttack state unit
    , thinkHeal state unit
    ]

thinkHeal : WholeState -> Unit -> Change
thinkHeal state unitDoingTheHealing =
  let
    updateHeal = updateHealUnit unitDoingTheHealing.healingStrength
  in
  getUnitsThatNeedHealing state
    |> List.map (\healedUnit -> ChangeUnit healedUnit.id updateHeal)
    |> ChangeList
```



### Adding, Removing

Adding and removing actors is in practice a messy business that usually involves different other actors, so having an "escape hatch" Change constructor to deal with the complexity is very useful:
```
type Change
  = ChangeNone
  ...
  | ChangeWholeState (WholeState -> WholeState)
```
This allows a Think function to affect the game state in any way possible, including adding, removing and making changes that affect several different actors.
Of course, every Change of this type requires to make a copy of the whole thing, which is what we wanted to avoid.
However, as long as we use this sparingly, it's perfectly fine, and adding and removing a few entities per second won't make much of a difference.

Things change if we need to add and remove entities very very often, (for example, projectiles).
In this case, it might be more efficient to have some dedicated Change constructors:
```
type Change
  = ChangeNone
  ...
  | ChangeWholeState (WholeState -> WholeState)
  | ChangeAddProjectile ProjectileComponents
  | ChangeRemoveProjectile ProjectileId
```

Before we tackle other issues with Changes, let's see how to apply them.


## Applying All Changes

TODO







## Conflict & Collaboration

TODO


## Pitfalls

TODO

