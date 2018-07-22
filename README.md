The architecture I used for Herzog Drei has served the game very well, so I thought to describe it quickly.

The goal is to be able to manage and update a complex, rich game state with a flat hierarchy,
where everything can interact with everything else in every possible way.

It's a practical implementation of the structure described by John Carmak in his
[Quakecon 2013 talk](https://www.youtube.com/watch?v=1PhArSujR_A).



The basic idea
==============

Every time you want to update the game state, everything that can affect the game
should produce a `Delta`:
```elm
type
    Delta
    -- change the game state
    = DeltaGame (GameState -> GameState)
      -- don't change anything
    | DeltaNone
      -- change more than one thing
      -- (useful for nesting and splitting your changes)
    | DeltaList (List Delta)
      -- do something that does NOT affect the game state such as playing sounds
    | DeltaSideEffect SideEffect


updateEverything : GameState -> ( GameState, List SideEffect )
updateEverything gameState =
    let
        victoryDelta : Delta
        victoryDelta =
            -- victoryConditionThink : GameState -> Delta
            victoryConditionThink gameState

        listOfAllUnits : List Unit
        listOfAllUnits =
            Dict.values gameState.unitsById

        unitDeltas : List Delta
        unitDeltas =
            -- unitDelta : GameState -> Unit -> Delta
            List.map (unitThink gameState) listOfAllUnits

        ...

        allChanges : List Delta
        allChanges =
            [ victoryDelta
            , DeltaList unitDeltas
            ...
            ]
    in
        applyGameDeltas allChanges gameState


applyGameDeltas : List Delta -> GameState -> ( GameState, List SideEffect )
applyGameDeltas deltas gameState =
    List.foldl applyDelta ( gameState, [] ) deltas


applyDelta : Delta -> ( GameState, List SideEffect ) -> ( GameState, List SideEffect )
applyDelta delta (gameState, sideEffects) =
    case delta of
        DeltaNone ->
            (gameState, sideEffects)

        DeltaList deltas ->
            List.foldl applyDelta (gameState, sideEffects) deltas

        DeltaGame updateGameState ->
            (updateGameState gameState, sideEffects)

        DeltaSideEffect sideEffect ->
            (gameState, sideEffects :: sideEffects)
```


The `DeltaGame` constructor is the one we will be using the most, but it is
also as generic as it gets, so it will be useful to have some helper functions
like:
```elm
deltaUnit : UnitId -> (Game -> Unit -> Unit) -> Delta
deltaUnit unitId updateFunction =
    DeltaGame (updateUnit unitId updateFunction)


updateUnit : UnitId -> (Game -> Unit -> Unit) -> GameState -> GameState =
updateUnit unitId updateFunction gameState =
    case Dict.get unitId gameState.unitsById of
        Nothing ->
            gameState

        Just unit ->
            let
                updatedUnit : Unit
                updatedUnit =
                    updateFunction gameState unit
            in
            { gameState | unitsById = Dict.insert unitId updatedUnit game.unitsById }
```



Ok, so how do I use this?
=========================
Let's say I want to describe what units can do: in this case, only move and
heal nearby unuits.

```elm
unitThink : GameState -> Unit -> Delta
unitThink gameState unit =
    let
        deltaMove =
            if hasDestination unit then
                deltaUnit unit.id unitMove
            else
                deltaNone

        deltaHeal =
            if isHealer unit then
                deltaUnitHealEveryone gameState unit
            else
                deltaNone
    in
    DeltaList
        [ deltaMove
        , deltaHeal
        ]


unitMove : GameState -> Unit -> Unit
unitMove gameState unit =
    { unit | position = Vec2.add unit.position + unit.speed }


deltaUnitHealEveryone : GameState -> Unit -> Delta
deltaUnitHealEveryone gameState unit =
    getAllUnitsInHealingRangeOf unit gameState
        |> List.map (deltaUnitHealOneUnit unit)
        |> DeltaList


deltaUnitHealOneUnit : Unit -> Unit -> Delta
deltaUnitHealOneUnit healer healed =
    if isOk healed then
        DeltaNone
    else
        DeltaList
            [ deltaUnit healed.id (healUnitBy healer.healingSpeed)
            , deltaSpawnHealingIcon healed.position
            ]


healUnitBy : Int -> GameState -> Unit -> Unit
healUnitBy amount gameState unit =
    { unit | life = min (unit.life + healAmount) gameState.maxUnitLife }
```



Use absolute times instead of relative ones
===========================================

Every single DeltaGame in the tree will force Elm to create a whole new copy
of the whole game state, so we want to avoid using it if we can.

Let's say a shooting unit has to wait for weapon cooldown before it can shoot
again.

My naive approach was to have a `cooldown` variable that stored the cooldown
time left and reduce it at every tick, so that when it reached zero the
unity would be ready to shoot again.
```elm
deltaUnitShoots : GameState -> Unit -> Unit -> Delta
deltaUnitShoots gameState unit target =
    DeltaList
        [ deltaSpawnProjectile unit target
        , deltaUnit unit.id (\gameState unit -> { unit | cooldown = gameState.attackCooldown })
        ]


unitThink : GameState -> Unit -> Delta
unitThink gameState unit =
    DeltaList
        [ ...
        , ...
        , deltaUnit unit.id (\gameState unit -> { unit | cooldown = max 0 (unit.cooldown - 1) })
        ]


unitCanFire : Unit -> Bool
unitCanFire unit =
    unit.cooldown == 0
```
This required a DeltaGame for each unit, for each tick, and it's easy to see
that it can grow fast...

A better approach is to just store the absolute time of when the unit will
be ready to fire again, and allow the unit to fire only past that time:
```elm
deltaUnitShoots : GameState -> Unit -> Unit -> Delta
deltaUnitShoots gameState unit target =
    DeltaList
        [ deltaSpawnProjectile unit target
        , deltaUnit unit.id (\gameState unit -> { unit | cooldownEnd = gameState.time + gameState.attackCooldown })
        ]


unitThink : GameState -> Unit -> Delta
unitThink gameState unit =
    DeltaList
        [ ...
        , ...
        -- we can remove the delta entirely!
        ]


unitCanFire : GameState -> Unit -> Bool
unitCanFire gameState unit =
    gameState.time > unit.cooldownEnd
```



Resolving conflicts: few units
==============================

Let's say healing power is limited, and every time a healer heals, it loses
a bit of power.
```elm
    DeltaList
        [ deltaUnit healed.id (healUnitBy healer.healingSpeed)
        , deltaSpawnHealingIcon healed.position
        -- we add this:
        , deltaRemovePower healer
        ]
```

If two healers try to heal a unit, one of them may waste some of its precious
healing power!

In this case, you should have a single update function, so that the healing
and the power loss become a single atomic operation:
```elm
...

    DeltaList
        [ DeltaGame (heal healer.id healed.id)
        , deltaSpawnHealingIcon healed.position
        ]

...

heal : UnitId -> UnitId -> GameState -> GameState
heal healerId healedId gameState =
    case Dict.get healerId gameState.unitsById (Dict.get healedId gameState.unitsById) of
        ( Just healer, Just healed ) ->
            if healer.power == 0 || healed.life == gameState.maxUnitLife then
                gameState
            else
                let
                    updatedHealer : Unit
                    updatedHealer =
                        { healer | healingPower = healer.healingPower - 1 }

                    updatedHealed : Unit
                    updatedHealed =
                        healUnitBy healer.healingSpeed gameState healed

                    updatedUnitsById : Dict UnitId Unit
                    updatedUnitsById =
                        gameState.unitsById
                            |> Dict.insert healerId updatedHealer
                            |> Dict.insert healedId updatedHealed
                in
                { gameState | unitsById = updatedUnitsById }

        _ ->
            gameState
```


Resolving conflicts: many units
===============================
I haven't tried this, but the idea is to record the *intention* of someone
trying to do something, e.g. "unit X wants to move to square Y", then, once
all deltas have been applied, process all the intentions in a single step,
resolving the conflicts, and update the game state accordingly.

You can even add a dedicated constructor to the `Delta` type if you want this
intention to be recorded outside of the game state.



Scheduling changes
==================

What if a unit wants something to happen, but *only after a certain time*?

We can add a new constructor to the `Delta` type, and store the scheduled
actions in the game state:
```elm
type Delta
    ...
    | DeltaDoLater TimeLength Delta


applyDelta : Delta -> ( GameState, List SideEffect ) -> ( GameState, List SideEffect )
applyDelta delta (gameState, sideEffects) =
    case delta of

        ...

        DeltaDoLater delay delta ->
            let
                updatedStuffToDoLater : List ( TimeLength, Delta )
                updatedStuffToDoLater =
                    (gameState.time + delay, delta) :: gameState.stuffToDoLater
            in
                ( { gameState | stuffToDoLater = updatedStuffToDoLater }, sideEffects )


updateEverything : GameState -> ( GameState, List SideEffect )
updateEverything gameState =
    let

        ...

        ( doNow, doLater ) : ( List ( TimeLength, Delta ), List ( TimeLength, Delta ) )
        ( doNow, doLater ) =
            gameState.stuffToDoLater
                |> List.partition (\( scheduledTime, delta ) -> scheduledTime < gameState.time)

        scheduledDeltas : List Delta
        scheduledDeltas =
            List.map Tuple.second doNow

        allChanges : List Delta
        allChanges =
            [ ...
            , DeltaList scheduledDeltas
            ]
    in
        applyGameDeltas allChanges { gameState | stuffToDoLater = doLater }
```

This approach has the drawback that functions will be stored in the game state,
which means you can't serialize it, you can't use `(==)` to compare game states
and you can't use `elm reactor`.

If you need any of the above, you will have to create a type to describe your
delayed actions instead of a normal `Delta`:
```elm
type DelayedDelta
    = DelayedHeal UnitId UnitId


type Delta
    ...
    | DeltaDoLater TimeLength DelayedDelta


updateEverything : GameState -> ( GameState, List SideEffect )
updateEverything gameState =
    let

        ...

        scheduledDeltas : List Delta
        scheduledDeltas =
            List.map (Tuple.second >> delayedDeltaToDelta) doNow

        ...


delayedDeltaToDelta : DelayedDelta -> Delta
delayedDeltaToDelta delayed =
    case delayed of
        DelayedHeal healerId healedId ->
            -- `heal` is defined in "Resolving conflicts"
            DeltaGame (heal healerId healedId)
```

This is a lot clunkier to use because for every effect you want you need to
define a new constructor and add it to the `case` statement above.


End note
========

With this architecture every effect and interaction can be described in full
without having to create and manage new type constructors and without modifying
modules other than the one where the effect is invoked, with the notable
exception of delayed changes, as noted above.

In my (limited) experience this allowed me to easily add and easily maintain
all the complexity I wanted, make experimens on the fly and prototype new
stuff quickly.

I used to be skeptical but now I think that immutable ML languages can be
very good for games.
