# DECLARATE v0.6.0

Original document by Braden "Blzut3" Obrzut: [language.md](https://github.com/Blzut3/Declarate/blob/master/docs/language.md)

## 1 - Introduction

The DECLARATE language is a **declarative language** for specifying static actor behavior and defining sprite animation sequences. It assumes that the engine runs at a constant tic rate and that actions are performed by the actor on animation state transitions.

An **actor** is an object in the playsim of the game engine which performs actions based on an animation sequence of states.

### 1.1 - Limitations

The DECLARATE language is based on the DECORATE language that originated from the ZDoom source port. DECORATE was developed with a lax lexer based on the one used for parsing scripts in Hexen. Over time, the limitations of this lexing mode have become obvious, as it heavily restricts the format. This document defines a dialect that uses a stricter, more C-style lexer, which allows more generic tools to parse the language.

In general, existing DECORATE scripts follow a consistent style that could be parsed using the language defined here; however, they are not 100% compatible in either direction. For example, ZDoom would parse the following, although it would be considered invalid by any sane person:

```decorate
actor~:Medikit"replaces""Zombieman""600" {
    inventory "." "amount" "1"
    +"ACTOR" "." SHOOTABLE
    states {
        "Death" ".": :
            \::\["RANDOM""(""1"",""2"")"
            loop
    }
}
```

In our case, the above is considered ill-formed and will not compile. Likewise, since ZDoom's lax lexer can't distinguish between different types of tokens, the following could be unambiguously parsed using this dialect, but not with ZDoom:

```decorate
actor Example {
    states {
        Spawn:
            TNT1 A 1
                A_Look loop
    }
}
```

In this case, the style isn't necessarily poor, but it would be considered unusual for DECORATE code.

### 1.2 - Lump

The name of the lump is `DECLARE`. `DECLARE` lumps are **cumulative**, meaning that several can be loaded at once without overwriting each other in memory.

---

## 2 - Grammar

### 2.1 - Tokens

The following base tokens are defined:

| Token       | Pattern                                           |
|-------------|---------------------------------------------------|
| identifier  | `[A-Za-z_][A-Za-z0-9_]*`                          |
| integer     | `[+-]?[1-9][0-9]* \| 0[0-9]+ \| 0x[0-9A-Fa-f]+`   |
| float       | `[+-]?[0-9]+'.'[0-9]*([eE][+-]?[0-9]+)?`          |
| string      | `"([^"\\]*(\\.[^"\\]*)*)"`                        |

### 2.2 - Keywords

The following identifiers are reserved for language use:

> `thing` `weapon` `ammo` `item` `sound` `ambient` `goto` `loop` `native` `replaces` `states` `stop` `wait` `metadata`

**Note:** Keywords are case-insensitive.

### 2.2.1 - `metadata` Block

```ebnf
metadata-definition = 'metadata' , '{' , metadata-properties , '}' ;
metadata-properties = metadata-property , { metadata-property } ;
metadata-property   = 'type' , string | 'version' , string ;
```

The `metadata` block **must be the first definition** in any DECLARATE file.

The **type** string value that should be `"declarate"`.
The **version** string expected to take the format of `"major.minor.patch"`. The regex `^(\d+)\.(\d+)\.(\d+)$` should result in an exact match with three capture groups.

### 2.2.2 - `#include` Directive

The `#include` directive includes the contents of the specified lump file. The included file is parsed immediately.

```ebnf
#include = string ;
```

### 2.3 - Actor Definition

```ebnf
actor-definition      = actor-header , '{' , [ actor-body ] , '}' ;
actor-header          = actor-class , identifier , [ inheritance-specifier ] , [ replaces-specifier ] , [ editor-number ] , [ 'native' ] ;
actor-class           = 'thing' | 'weapon' | 'ammo' | 'item' ;
inheritance-specifier = ':' , identifier ;
replaces-specifier    = 'replaces' , identifier ;
editor-number         = integer ;
actor-body            = actor-body-statement | actor-body , actor-body-statement ;
actor-body-statement  = property-assignment | flag-combo | flag-assignment | states-decl ;
```

An **actor-definition** declares a new actor. The name of the actor is given in the **actor-header** and shall be unique.

An actor with an **inheritance-specifier** copies the properties, flags, and states from the specified actor and makes the specified actor its parent. The **actor-class** should match the **actor-class** of the parent. The parent actor must be previously defined and shall not inherit from itself.

The **replaces-specifier** allows an actor to wholly replace another actor. The new actor will assume the **editor-number** of the replacee and intercept any reference to the replacee when spawning. The replacee shall not be native.

The **editor-number** specifies a number which is used to reference this actor for static spawning in game levels. It shall be valid as a signed 16-bit integer and must be positive.

The **native** keyword marks an actor as native, meaning it exists in the engine's original code and should not be modified.

#### 2.3.1 - Properties

```ebnf
property-assignment = property , property-args ;
property            = identifier ;
property-args       = property-arg | property-arg , ',' , property-args ;
property-arg        = string | integer | float ;
```

A **property-assignment** directly assigns a constant value to the actor structure. The type and number of arguments for a property depends on the property being assigned.

The list of properties that are available is dependent on the **actor-class** and defined in [Section 4](#4---properties--flags).

In the case of duplicated assignments, the last assignment is used.

**Note:** Due to the zero initialization of actor structures, integer/float properties default to 0 if not specified in any parent. String properties default to NULL.

#### 2.3.2 - Flags

```ebnf
flag-assignment = '+' , flag | '-' , flag ;
flag            = identifier ;
flag-combo      = 'MONSTER' | 'PROJECTILE' ;
```

A **flag-assignment** sets or removes a flag from an actor. If the assignment begins with the `+` token then the specified flag is to be set. If the assignment begins with the `-` token then the flag is to be removed.

The resolution of a **flag** is determined by the same rules as specified in the properties section.

If there are multiple assignments for the same flag, then the last assignment is used.

A **flag-combo** sets a defined set of flags. These act just as if the flags were set manually and specific flags in the combo can be unset.

| Combo     | Flags Set                                      |
|-----------|------------------------------------------------|
| MONSTER   | `SHOOTABLE`, `SOLID`, `COUNTKILL`              |
| PROJECTILE| `NOBLOCKMAP`, `MISSILE`, `DROPOFF`, `NOGRAVITY`|

#### 2.3.3 - States

```ebnf
states-decl         = 'states' , '{' , [ state-seq ] , '}' ;
state-seq           = state | state , state-seq ;
state               = [ state-labels ] , sprite-name , frame-sequence , duration , [ frame-properties ] , [ codepointer ] , [ state-flow-specifier ] ;
state-labels        = state-label , ':' | state-label , ':' , state-labels ;
state-label         = state-label-name ;
state-label-name    = identifier ;
sprite-name         = identifier | string ;
frame-sequence      = identifier | string ;
duration            = integer ;
frame-properties    = 'bright' | 'fast' | 'offset' , '(' , integer , ',' , integer , ')' ;
state-flow-specifier = 'goto' , state-label , [ '+' , offset ] | 'loop' | 'stop' | 'wait' ;
```

The **states-decl** allows for the animation sequences to be defined for the actor. All states from the parent are accessible from the new actor.

There shall only be one **states-decl** in an actor definition.

The **sprite-name** should be exactly four characters long.

The **frame-sequence** may be of any length, but may only contain the characters A-Z or a-z. The characters `[`, `\`, or `]` may also be used if the string form is utilized.

If the **frame-sequence** contains more than one character, then the **state** is to be expanded into a sequence of states with all properties identical except for the frame which will be in the order given.

The **state-labels** will refer to only the first state in the sequence. The **state-flow-specifier** modifies the flow of only the last state in the sequence.

**Frame Properties:**

| Property | Description |
|----------|-------------|
| bright   | Renders the sprite at full brightness, ignoring sector lighting |
| fast     | Frame takes half as long on fast skill levels (used in Doom by the demons) |
| offset   | Applies a render offset to the sprite with the given (X, Y) values (used in Doom for weapons) |

The **duration** indicates the number of play sim tics that the state is to be held before going to the next state. A duration of `-1` indicates infinite, and a duration of `0` indicates that only the codepointer should be executed and the following state should be used immediately. Should there be a 0 duration loop, the result is undefined.

Without the **state-flow-specifier**, the state should continue on to the next state in the sequence. If the end of the states block is reached and the state does not have a duration of -1 the result is undefined.

| Specifier | Behavior |
|-----------|----------|
| `goto`    | Next state will be set to the state referred to by the label |
| `wait`    | The state will be re-entered after the duration |
| `loop`    | The most recently declared **state-label** will be the next frame |
| `stop`    | The actor is to be removed from the play sim |

**Note:** Specifying a **state** without a previously declared **state-label** is ill-formed.

**Note:** Goto resolution is to be performed after all states are parsed in order to allow state-labels defined after the goto to be referenced.

#### 2.3.3.1 - Special Sprites

The **sprite-name** `TNT1` indicates that no sprite should be rendered.

### 2.3.4 - Codepointer

```ebnf
codepointer = identifier | identifier , '(' , ')' | identifier , '(' , arg-list , ')' ;
arg-list    = arg | arg , ',' , arg-list ;
arg         = identifier | integer | float | string | flag-list ;
flag-list   = flag | flag , '+' , flag-list | flag , '|' , flag-list ;
flag        = identifier ;
```

The **codepointer** is to be executed on the tic that the state is transitioned to. The name of a **codepointer** shall be at least 5 characters in length to distinguish it from a sprite name. Some codepointers, especially in MBF and later formats, can be optionally parameterized.

### 2.4 - Sound Definition

```ebnf
sound-definition = 'sound' , identifier , '{' , sound-properties , '}' ;
sound-properties = sound-property | sound-property , sound-properties ;
sound-property   = identifier | identifier , sound-args ;
sound-args       = sound-arg | sound-arg , ',' , sound-args ;
sound-arg        = integer | float | string | identifier ;
```

**Sound Properties:**

| Property | Type      | Description |
|----------|-----------|-------------|
| prefix   | none      |Enables checking lump name with "ds" (Doom SFX format) or "dp" (PC Speaker format) prefixes |
| lump     | lump-list |One or more lump names. If multiple strings are defined, a random sound will be chosen every time |

```ebnf
lump-list = string | string , ',' , lump-list ;
```

### 2.5 - Ambient Definition

```ebnf
ambient-definition = 'ambient' , '{' , ambient-properties , '}' ;
ambient-properties = ambient-property | ambient-property , ambient-properties ;
ambient-property   = identifier | identifier , ambient-arg ;
ambient-arg        = integer | float | string | identifier ;
```

**Ambient Properties:**

| Property     | Type      | Description |
|--------------|-----------|-------------|
| index        | integer   | Index that corresponds to DoomEd number, from 1-64 (14101-14164)  |
| sound        | identifier| Reference to a defined sound |
| attenuation  | float     | Distance attenuation factor |
| type         | identifier| Sound type: `Continuous`, `Random` or `Periodic` |
| volume       | float     | Volume level (0.0 to 1.0) |
| period       | float     | Period between plays (for `Periodic` type) |
| minperiod    | float     | Minimum period between plays (for `Random` type) |
| maxperiod    | float     | Maximum period between plays (for `Random` type) |

---

## 3 - Base Actor Hierarchy (WIP)

The basic actor hierarchy consists of native actors which provide various functions within the language. The hierarchy only needs to exist insofar as needed for performing inheritance resolution and do not need to exist outside of the DECLARATE language.

### 3.1 - Doom v1.9 Actors

#### `thing` Class

- DoomPlayer
- ZombieMan
- ShotgunGuy
- Archvile
- ArchvileFire
- Revenant
- RevenantTracer
- RevenantTracerSmoke
- Fatso
- FatShot
- ChaingunGuy
- DoomImp
- Demon
- Spectre
- Cacodemon
- BaronOfHell
- BaronBall
- HellKnight
- LostSoul
- SpiderMastermind
- Arachnotron
- Cyberdemon
- PainElemental
- WolfensteinSS
- CommanderKeen
- BossBrain
- BossEye
- BossTarget
- SpawnShot
- SpawnFire
- ExplosiveBarrel
- DoomImpBall
- CacodemonBall
- Rocket
- PlasmaBall
- BFGBall
- ArachnotronPlasma
- BulletPuff
- Blood
- TeleportFog
- ItemFog
- TeleportDest
- BFGExtra
- TechLamp
- TechLamp2
- Column
- TallGreenColumn
- ShortGreenColumn
- TallRedColumn
- ShortRedColumn
- SkullColumn
- HeartColumn
- EvilEye
- FloatingSkull
- TorchTree
- BlueTorch
- GreenTorch
- RedTorch
- ShortBlueTorch
- ShortGreenTorch
- ShortRedTorch
- Stalagtite
- TechPillar
- CandleStick
- Candelabra
- BloodyTwitch
- Meat2
- Meat3
- Meat4
- Meat5
- NonsolidMeat2
- NonsolidMeat4
- NonsolidMeat3
- NonsolidMeat5
- NonsolidTwitch
- DeadCacodemon
- DeadMarine
- DeadZombieMan
- DeadDemon
- DeadLostSoul
- DeadDoomImp
- DeadShotgunGuy
- GibbedMarine
- GibbedMarineExtra
- HeadsOnAStick
- Gibs
- HeadOnAStick
- HeadCandles
- DeadStick
- LiveStick
- BigTree
- BurningBarrel
- HangNoGuts
- HangBNoBrain
- HangTLookingDown
- HangTSkull
- HangTLookingUp
- HangTNoBrain
- ColonGibs
- SmallBloodPool
- BrainStem

#### `item` Class

- GreenArmor
- BlueArmor
- HealthBonus
- ArmorBonus
- BlueCard
- RedCard
- YellowCard
- YellowSkull
- RedSkull
- BlueSkull
- Stimpack
- Medikit
- Soulsphere
- InvulnerabilitySphere
- Berserk
- BlurSphere
- RadSuit
- Allmap
- Infrared
- Megasphere
- Clip
- ClipBox
- RocketAmmo
- RocketBox
- Cell
- CellPack
- Shell
- ShellBox
- Backpack
- BFG9000
- Chaingun
- Chainsaw
- RocketLauncher
- PlasmaRifle
- Shotgun
- SuperShotgun

#### `weapon` Class

- Fist
- Pistol
- Chainsaw
- Shotgun
- SuperShotgun
- Chaingun
- RocketLauncher
- PlasmaRifle
- BFG9000

### 3.2 - Boom Actors

#### `thing` Class

- PointPusher
- PointPuller

### 3.3 - MBF Actors

#### `thing` Class

- PlasmaBall1
- PlasmaBall2
- MBFHelperDog

#### `item` Class

- EvilSceptre
- UnholyBible

### 3.4 - Additional Actors

#### `thing` Class

- MusicChanger

---

## 4 - Properties & Flags

### 4.1 - Properties

#### 4.1.1 - Doom v1.9 Thing Properties

| Property       | Type     | Description |
|----------------|----------|-------------|
| Health         | integer  | Sets the spawn health |
| SeeSound       | sound    | Sound played when the actor sees the player |
| ReactionTime   | integer  | Tics before the actor can react |
| AttackSound    | sound    | Sound played when attacking |
| PainChance     | integer  | Probability of entering pain state |
| PainSound      | sound    | Sound played when hurt |
| DeathSound     | sound    | Sound played on death |
| Speed          | integer  | Movement speed (converted to fixed point if `MISSILE` flag is set) |
| Radius         | float    | Collision radius |
| Height         | float    | Actor height |
| Mass           | integer  | Mass for physics calculations |
| Damage         | integer  | Damage dealt by attacks |
| ActiveSound    | sound    | Ambient sound played while active |

**Note:** The `sound` type indicates that the property takes a *identifier* referring to a logical sound.

**Note:** `Health` is the only property which doesn't match with vanilla Doom's internal name. This sets the `spawnhealth` field.

**Note:** The `speed` property is an integer, but will be converted to a fixed point value if the `MISSILE` flag is set.

#### 4.1.2 - MBF21 Properties

| Property        | Type     | Description |
|-----------------|----------|-------------|
| InfightingGroup | group    | Infighting group identifier |
| ProjectileGroup | group    | Projectile group identifier |
| SplashGroup     | group    | Splash damage group identifier |
| RipSound        | string   | Sound played when projectile rips |
| AltSpeed        | integer  | Alternate movement speed |
| MeleeRange      | float    | Maximum melee attack range |

#### 4.1.3 - Additional Properties

| Property     | Type     | Description |
|--------------|----------|-------------|
| DropItem     | item     | Item dropped when killed |
| RenderStyle  | style    | Render style: `"fuzzy"` or `"none"` |
| Translation  | integer  | Color translation index |
| Obituary     | string   | Death message for victims |
| HitObituary  | string   | Death message when killed by melee |
| SelfObituary | string   | Death message for self-inflicted damage |

**Note:** The `style` type indicates that the property takes a string `"fuzzy"` or `"none"`.

### 4.2 - Flags

#### 4.2.1 - Doom v1.9 Flags

| Flag           | Description |
|----------------|-------------|
| SPECIAL        | Special activation |
| SOLID          | Solid collision |
| SHOOTABLE      | Can be shot |
| NOSECTOR       | No sector clipping |
| NOBLOCKMAP     | Not on blockmap |
| AMBUSH         | Ambush behavior |
| JUSTHIT        | Just hit state |
| JUSTATTACKED   | Just attacked state |
| SPAWNCEILING   | Spawn from ceiling |
| NOGRAVITY      | No gravity applied |
| DROPOFF        | Can drop off ledges |
| PICKUP         | Can be picked up |
| NOCLIP         | No collision clipping |
| SLIDE          | Sliding movement |
| FLOAT          | Floating movement |
| TELEPORT       | Teleported this tic |
| MISSILE        | Is a missile/projectile |
| DROPPED        | Dropped item flag |
| SHADOW         | Shadow rendering |
| NOBLOOD        | No blood splatter |
| CORPSE         | Is a corpse |
| INFLOAT        | Floating in place |
| COUNTKILL      | Counts toward kill total |
| COUNTITEM      | Counts toward item total |
| SKULLFLY       | Skull fly attack mode |
| NOTDMATCH      | No deathmatch spawn |
| TRANSLATION1   | Color translation 1 |
| TRANSLATION2   | Color translation 2 |

#### 4.2.2 - Boom and MBF Flags

| Flag          | Description |
|---------------|-------------|
| TOUCHY        | Dies on contact with solid objects |
| BOUNCES       | Bounces off walls/floors |
| FRIEND        | Friendly to player |
| TRANSLUCENT   | Translucent rendering |

#### 4.2.3 - MBF21 Flags

| Flag              | Description |
|-------------------|-------------|
| LOGRAV            | Low gravity |
| SHORTMRANGE       | Short melee range |
| DMGIGNORED        | Damage ignored |
| NORADIUSDMG       | No radius damage |
| FORCERADIUSDMG    | Force radius damage |
| HIGHERMPROB       | Higher missile probability |
| RANGEHALF         | Range at half |
| NOTHRESHOLD       | No damage threshold |
| LONGMELEE         | Long melee reach |
| BOSS              | Boss behavior |
| MAP07BOSS1        | MAP07 boss 1 |
| MAP07BOSS2        | MAP07 boss 2 |
| E1M8BOSS          | E1M8 boss |
| E2M8BOSS          | E2M8 boss |
| E3M8BOSS          | E3M8 boss |
| E4M6BOSS          | E4M6 boss |
| E4M8BOSS          | E4M8 boss |
| RIP               | Projectile rips through actors |
| FULLVOLSOUNDS     | Full volume sounds |

### 4.3 - Sound Properties

| Property | Type       | Description |
|----------|------------|-------------|
| prefix   | —          | Enables checking lump name with "ds" (Doom SFX format) or "dp" (PC Speaker format) prefixes |
| lump     | lump-list  | One or more lump names. If multiple strings are defined, a random sound will be chosen every time |

```ebnf
lump-list = string | string , ',' , lump-list ;
```

### 4.4 - Recognized States

#### 4.4.1 - Thing States

| State   | Description |
|---------|-------------|
| Spawn   | Initial spawning state |
| See     | Chasing the player |
| Pain    | Reacting to damage |
| Melee   | Melee attack sequence |
| Missile | Ranged attack sequence |
| Death   | Normal death sequence |
| XDeath  | Violent/extreme death sequence |
| Raise   | Resurrection sequence |

#### 4.4.2 - Weapon States

| State    | Description |
|----------|-------------|
| Select   | Weapon raise (internally `upstate`) |
| Deselect | Weapon lower (internally `downstate`) |
| Ready    | Idle ready state |
| Fire     | Firing sequence |
| Flash    | Muzzle flash effect |

---

## 5 - Codepointers

### 5.1 - Doom v1.9 Weapon Codepointers

**A_Light0**  
Turn off dynamic light

**A_WeaponReady**  
Check weapon ready state

**A_Lower**  
Lower weapon

**A_Raise**  
Raise weapon

**A_Punch**  
Fist punch attack

**A_ReFire**  
Check for refire

**A_FirePistol**  
Fire pistol

**A_Light1**  
Dynamic light level 1

**A_FireShotgun**  
Fire shotgun

**A_Light2**  
Dynamic light level 2

**A_FireShotgun2**  
Fire super shotgun

**A_CheckReload**  
Check reload status

**A_OpenShotgun2**  
Open super shotgun

**A_LoadShotgun2**  
Load super shotgun

**A_CloseShotgun2**  
Close super shotgun

**A_FireCGun**  
Fire chaingun

**A_GunFlash**  
Muzzle flash

**A_FireMissile**  
Fire missile

**A_Saw**  
Chainsaw attack

**A_FirePlasma**  
Fire plasma rifle

**A_BFGsound**  
Play BFG sound

**A_FireBFG**  
Fire BFG

**A_BFGSpray**  
BFG spray damage

### 5.2 - Doom v1.9 Thing Codepointers

**A_BFGSpray**  
BFG spray damage

**A_Explode**  
Explosion damage

**A_Pain**  
Pain reaction

**A_PlayerScream**  
Player death scream

**A_Fall**  
Apply gravity/fall

**A_XScream**  
Violent death scream

**A_Look**  
Look for target

**A_Chase**  
Chase target

**A_FaceTarget**  
Face current target

**A_PosAttack**  
Possessed attack

**A_Scream**  
Death scream

**A_SPosAttack**  
Shotgun guy attack

**A_VileChase**  
Arch-vile chase

**A_VileStart**  
Arch-vile attack start

**A_VileTarget**  
Arch-vile target selection

**A_VileAttack**  
Arch-vile fire attack

**A_StartFire**  
Start fire effect

**A_Fire**  
Fire damage

**A_FireCrackle**  
Fire crackle effect

**A_Tracer**  
Homing tracer

**A_SkelWhoosh**  
Revenant whoosh sound

**A_SkelFist**  
Revenant fist attack

**A_SkelMissile**  
Revenant missile attack

**A_FatRaise**  
Fatso raise arms

**A_FatAttack1**  
Fatso attack 1

**A_FatAttack2**  
Fatso attack 2

**A_FatAttack3**  
Fatso attack 3

**A_BossDeath**  
Boss death behavior

**A_CPosAttack**  
Chaingun guy attack

**A_CPosRefire**  
Chaingun guy refire

**A_TroopAttack**  
Imp troop attack

**A_SargAttack**  
Demon sergeant attack

**A_HeadAttack**  
Cacodemon head attack

**A_BruisAttack**  
Baron bruise attack

**A_SkullAttack**  
Lost soul attack

**A_Metal**  
Metal impact sound

**A_SpidRefire**  
Spider Mastermind refire

**A_BabyMetal**  
Arachnotron metal sound

**A_BspiAttack**  
Arachnotron plasma attack

**A_Hoof**  
Cyberdemon hoof sound

**A_CyberAttack**  
Cyberdemon attack

**A_PainAttack**  
Pain elemental attack

**A_PainDie**  
Pain elemental death

**A_KeenDie**  
Commander Keen death

**A_BrainPain**  
Boss brain pain

**A_BrainScream**  
Boss brain scream

**A_BrainDie**  
Boss brain death

**A_BrainAwake**  
Boss brain awake

**A_BrainSpit**  
Boss brain spit

**A_SpawnSound**  
Spawn sound effect

**A_SpawnFly**  
Spawn fly effect

**A_BrainExplode**  
Boss brain explode

### 5.3 - MBF Thing Codepointers

**A_Detonate**  
Detonate explosive

**A_Mushroom**(*float* angle, *float* speed)  
Mushroom explosion effect

**A_Die**  
Instant death

**A_Spawn**(*identifier* item, *float* z_pos)  
Spawn item

**A_Turn**(*integer* degrees)  
Turn actor

**A_Face**(*integer* degrees)  
Face direction

**A_Scratch**(*integer* damage, *sound* sound)  
Scratch attack

**A_PlaySound**(*sound* sound, *boolean* fullvolume)  
Play sound

**A_RandomJump**(*identifier* state, *integer* probability)  
Random state jump

**A_LineEffect**(*integer* special, *integer* tag)  
Trigger line effect

**A_BetaSkullAttack**  
Beta skull attack

**A_Stop**  
Stop actor

### 5.4 - MBF Weapon Codepointers

**A_FireOldBFG**  
Fire beta BFG

### 5.5 - MBF21 Thing Codepointers

**A_SpawnObject**(*identifier* thing, *float* angle, *float* x_ofs, *float* y_ofs, *float* z_ofs, *float* x_vel, *float* y_vel, *float* z_vel)  
Spawn object with velocity

**A_MonsterProjectile**(*identifier* thing, *float* angle, *float* pitch, *float* hoffset, *float* voffset)  
Fire projectile

**A_MonsterBulletAttack**(*float* hspread, *float* vspread, *integer* numbullets, *integer* damagebase, *integer* damagedice)  
Bullet attack

**A_MonsterMeleeAttack**(*integer* damagebase, *integer* damagedice, *sound* sound, *float* range)  
Melee attack

**A_RadiusDamage**(*integer* damage, *integer* radius)  
Radius damage

**A_NoiseAlert**  
Alert nearby monsters

**A_HealChase**(*identifier* state, *sound* sound)  
Heal and chase

**A_SeekTracer**(*float* threshold, *float* maxturnangle)  
Seek tracer target

**A_FindTracer**(*float* fov, *integer* rangeblocks)  
Find tracer in FOV

**A_ClearTracer**  
Clear tracer target

**A_JumpIfHealthBelow**(*identifier* state, *integer* health)  
Jump if health below threshold

**A_JumpIfTargetInSight**(*identifier* state, *float* fov)  
Jump if target visible

**A_JumpIfTargetCloser**(*identifier* state, *float* distance)  
Jump if target closer than distance

**A_JumpIfTracerInSight**(*identifier* state, *float* fov)  
Jump if tracer visible

**A_JumpIfTracerCloser**(*identifier* state, *float* distance)  
Jump if tracer closer than distance

**A_JumpIfFlagsSet**(*identifier* state, *flag-list* flags)  
Jump if flags are set

**A_AddFlags**(*flag-list* flags)  
Add flags to actor

**A_RemoveFlags**(*flag-list* flags)  
Remove flags from actor

### 5.6 - MBF21 Weapon Codepointers (WIP)

**A_WeaponProjectile**  
Fire projectile from weapon

**A_WeaponBulletAttack**  
Bullet attack from weapon

**A_WeaponMeleeAttack**  
Melee attack from weapon

**A_WeaponSound**  
Play weapon sound

**A_WeaponAlert**  
Alert monsters

**A_WeaponJump**  
Conditional state jump

**A_ConsumeAmmo**  
Consume ammunition

**A_CheckAmmo**  
Check ammunition count

**A_RefireTo**  
Refire to state

**A_GunFlashTo**  
Muzzle flash to state
