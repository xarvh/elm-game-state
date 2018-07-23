The architecture I used for [Herzog Drei](https://xarvh.github.io/herzog-drei/)
has served the game very well, so I thought to describe it quickly.

The goal is to be able to manage and update a complex, rich game state with a flat hierarchy,
where everything can interact with everything else in every possible way.

It's a practical implementation of the structure described by John Carmak in his
[Quakecon 2013 talk](https://www.youtube.com/watch?v=1PhArSujR_A) and it makes
heavy use of partially applied functions.



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
            -- unitThink : GameState -> Unit -> Delta
            List.map (unitThink gameState) listOfAllUnits

        ...

        allChanges : List Delta
        allChanges =
            [ victoryDelta
            , DeltaList unitDeltas
            , ...
            , ...
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
Let's say I want units to be able to move and heal nearby units.
How would `unitThink` look like?

```elm
unitThink : GameState -> Unit -> Delta
unitThink gameState unit =
    let
        deltaMove : Delta
        deltaMove =
            if hasDestination unit then
                deltaUnit unit.id unitMove
            else
                DeltaNone

        deltaHeal : Delta
        deltaHeal =
            if isHealer unit then
                deltaUnitHealEveryone gameState unit
            else
                DeltaNone
    in
    DeltaList
        [ deltaMove
        , deltaHeal
        ]


unitMove : GameState -> Unit -> Unit
unitMove gameState unit =
    { unit | position = Vec2.add unit.position unit.speed }


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
            [ deltaSpawnHealingIcon healed.position
            -- we use partial application on `healUnitBy`
            , deltaUnit healed.id (healUnitBy healer.healingSpeed)
            ]


healUnitBy : Int -> GameState -> Unit -> Unit
healUnitBy amount gameState unit =
    { unit | life = min (unit.life + healAmount) gameState.maxUnitLife }
```

IMPORTANT: did you notice that the `unit` that appears in `unitThink` is not
the same `unit` used in `unitMove`?

The `unit` inside `unitThink` refers to the unit state *before* deltas are
applied to the game state.

The `unit` inside `unitMove` instead refers to the unit *during* the
application of deltas.

If more than one delta modify the same unit, the two `unit`s won't be the same.
Be careful to distinguish the two, otherwise you might end up undoing what the
previous delta did.



Spawning stuff
==============
Things that affect the larger game, or that do not refer to existing units,
can be always executed via `DeltaGame`:

```elm
deltaSpawnProjectile : Unit -> Unit -> Delta
deltaSpawnProjectile unit target =
    DeltaGame (addProjectile unit.position target.id)


addProjectile : Vec2 -> UnitId -> GameState -> GameState
addProjectile position targetId gameState =
    let
        projectile : Projectile
        projectile =
            { id = gameState.nextId
            , position = position
            , spawnTime = gameState.time
            , targetUnitId = targetId
            }

        updatedProjectiles : Dict ProjectileId Projectile
        updatedProjectiles =
            Dict.insert projectile.id projectile gameState.projectilesById
    in
        { gameState | nextId = projectile.id + 1, projectilesById = updatedProjectiles }
```

Given a function that modifies the game state, such as `addProjectile`, we
partially apply its arguments until only a `GameState -> GameState` function
is left, and the function becomes the argument to `DeltaGame`.



Use absolute times instead of relative ones
===========================================

Every single DeltaGame in the tree will force Elm to create a new copy (however
shallow) of the whole game state, so we want to avoid using it if we can.

Let's say a shooting unit has to wait for weapon cooldown before it can shoot
again.

My naive approach was to have a `cooldown` variable that stored the cooldown
time left and reduce it at every tick, so that when it reached zero the
unity would be ready to shoot:
```elm
deltaUnitShoots : GameState -> Unit -> Unit -> Delta
deltaUnitShoots gameState unit target =
    DeltaList
        [ deltaSpawnProjectile unit target
        , deltaUnit unit.id (\gameState u -> { u | cooldown = gameState.attackCooldown })
        ]


unitThink : GameState -> Unit -> Delta
unitThink gameState unit =
    DeltaList
        [ ...
        , ...
        , deltaUnit unit.id (\gameState u -> { u | cooldown = max 0 (unit.cooldown - 1) })
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
        , deltaUnit unit.id (\gameState u -> { u | cooldownEnd = gameState.time + gameState.attackCooldown })
        ]


unitThink : GameState -> Unit -> Delta
unitThink gameState unit =
    DeltaList
        [ ...
        , ...
        -- we can remove the deltaUnit call entirely!
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
    DeltaList
        [ DeltaGame (heal healer.id healed.id)
        , deltaSpawnHealingIcon healed.position
        ]
```

```elm
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
```

```elm
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

        -- ( List ( TimeLength, Delta ), List ( TimeLength, Delta ) )
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
```

```elm
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
Worse, you lose the composability and flexibility of the deltas.



Random Changes
==============

Sometimes changes should be random, and in an immutable world randomness needs
a way to update the random seed.

If we store the random seed together with the rest of the game state (as we
should), we can create a normal `GameState -> GameState` function that will
run `Random.step`, make the random changes and update the seed; we pass the
function to `DeltaGame` and we're done with it.

In my experience however, it's easier to express things in terms of deltas,
because they are more flexible and can be composed quickly.
Yet deltas cannot update the random seed before they are actually applied, so
how do we do this?

Well, we add another constructor:

```elm
type Delta
    ...
    | DeltaRandom (Random.Generator Delta)
```


```elm
applyDelta : Delta -> ( GameState, List SideEffect ) -> ( GameState, List SideEffect )
applyDelta delta ( gameState, sideEffects ) =
    case delta of

        ...

        DeltaRandom deltaGenerator ->
            let
                -- ( Delta, Random.Seed )
                ( generatedDelta, seed ) =
                    Random.step deltaGenerator gameState.seed
            in
            applyDelta generatedDelta ( { gameState | seed = seed }, outcomes )
```

And also some helpers:
```elm
deltaRandom : (a -> Delta) -> Random.Generator a -> Delta
deltaRandom function generator =
    DeltaRandom (Random.map function generator)


deltaRandom2 : (a -> b -> Delta) -> Random.Generator a -> Random.Generator b -> Delta
deltaRandom2 function generatorA generatorB =
    DeltaRandom (Random.map2 function generatorA generatorB)


deltaWithChance : Float -> Delta -> Delta
deltaWithChance chance delta =
    let
        rollToDelta : Float -> Delta
        rollToDelta roll =
            if roll > chance then
                DeltaNone
            else
                delta
    in
    deltaRandom rollToDelta (Random.float 0 1)
```


Now we can declare our deltas randomly:

```elm
deltaSpawnBubbleGfx : Delta
deltaSpawnBubbleGfx =
    deltaRandom2 (\x y -> deltaAddGfx x y GfxTypeBubble) (Random.int 0 5) (Random.int 0 6)


deltaSometimesSpawnBubbles : Delta
deltaSometimesSpawnBubbles =
    deltaWithChance 0.5 deltaSpawnBubbleGfx
```

The nice thing is that we can compose `DeltaRandom` with all the other
delta constructors, including `DeltaList`, `DeltaDoLater`, etc...

```elm
deltaSometimesSpawnManyBubbles : Delta
deltaSometimesSpawnManyBubbles =
    let
        spawnLater : Delta
        spawnLater =
            deltaRandom (\delay -> DeltaDoLater delay deltaSpawnBubbleGfx) (Random.float 0 2000)

        sometimesSpawnLater : Delta
        sometimesSpawnLater =
            deltaWithChance 0.3 spawnLater
    in
    deltaRandom (\n -> List.repeat n sometimesSpawnLater |> DeltaList) (Random.int 0 10)
```



End note
========

With this architecture every effect and interaction can be described in full
without having to create and manage new type constructors and without modifying
modules other than the one where the effect is invoked, with the notable
exception of delayed changes, as noted above.

In my (limited) experience this allowed me to easily add and easily maintain
all the complexity I wanted, make experiments on the fly and prototype new
stuff quickly.

I used to be skeptical but now I think that immutable ML languages can be
very good for games.
