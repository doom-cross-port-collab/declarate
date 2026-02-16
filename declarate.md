# DECLARATE v0.5.0

Author of the original [document](https://github.com/Blzut3/Declarate/blob/master/docs/language.md) is Braden "Blzut3" Obrzut.

## 1 - Introduction

The DECLARATE language is a declarative language for specifying static actor behavior and defining sprite animation sequences. It assumes that the engine runs at a constant tic rate and that actions are performed by the actor on animation state transitions.

An actor is an object in the playsim of the game engine which performs actions based on an animation sequence of states.

### 1.1 - Limitations

The DECLARATE language is based on the DECORATE language that originated from the ZDoom source port. DECORATE was developed with a lax lexer based on the one used for parsing scripts in Hexen. Over time, the limitations of this lexing mode have become obvious, as it heavily restricts the format. This document defines a dialect that uses a stricter, more C-style lexer, which allows more generic tools to parse the language.

In general, existing DECORATE scripts follow a consistent style that could be parsed using the language defined here; however, they are not 100% compatible in either direction. For example, ZDoom would parse the following, although it would be considered invalid by any sane person.

    actor~:Medikit"replaces""Zombieman""600" {
        inventory "." "amount" "1"
        +"ACTOR" "." SHOOTABLE
        states {
            "Death" ".": :
                \::\["RANDOM""(""1"",""2"")"
                loop
        }
    }

In our case, we would consider the above ill-formed and it will not compile. Likewise, since ZDoom's lax lexer can't distinguish between different types of tokens, the following could be unambiguously parsed using this dialect, but not with ZDoom.

    actor Example {
        states {
            Spawn:
                TNT1 A 1
                    A_Look loop
        }
    }

In this case, the style isn't necessarily poor, but it would be considered unusual for DECORATE code.

### 1.2 - Lump

The name of the lump is `DECLARE`. `DECLARE` lumps are cumulative, meaning that several can be loaded at once without overwriting each other in memory.

## 2 - Grammar

### 2.1 - Tokens

The following base tokens are defined.

>identifier := [A-Za-z_][A-Za-z0-9_]\*  
>integer := [+-]?[1-9][0-9]\* | 0[0-9]+ | 0x[0-9A-Fa-f]+  
>float := [+-]?[0-9]+'.'[0-9]\*([eE][+-]?[0-9]+)?  
>string := "([^"\\]\*(\\.[^"\\]\*)\*)"

### 2.2 - Keywords

The following identifiers are reserved for language use.

>thing weapon ammo item goto loop native replaces states stop wait

Keywords are case insensitive.

### 2.2.1 - `#include` directive

The #include directive includes the contents of the specified lump file. The included file is parsed immediately.

>#include *path*

>*path*:
>>*string*

### 2.2.2 - `version` directive

>version *string*

Must be the fist record in any DECLARATE file. Expected to take the format of `".."`. The regex string `^(\d+)\.(\d+)\.(\d+)$` should thus result in an exact match and three groups with the provided value.

### 2.3 - Actor Definition

>*actor-definition*:
>>*actor-header* { *actor-bodyₒₚₜ* }

>*actor-header*:
>>*actor-class* *identifier* *inheritance-specifierₒₚₜ* *replaces-specifierₒₚₜ* *editor-numberₒₚₜ* nativeₒₚₜ

>*actor-class*:
>>thing  
>>weapon  
>>ammo  
>>item

>*inheritance-specifier*:
>>: *identifier*

>*replaces-specifier*:
>>replaces *identifier*

>*editor-number*:
>>*integer*

>*actor-body*:
>>*actor-body-statement*  
>>*actor-body* *actor-body-statement*

>*actor-body-statement*:
>>*property-assignment*  
>>*flag-combo*  
>>*flag-assignment*  
>>*states-decl*

An *actor-definition* declares a new actor. The name of the actor is given in the *actor-header* and shall be unique.

An actor with an *inheritance-specifier* copies the properties, flags, and states from the specified actor and makes the specified actor its parent. The *actor-class* should match the *actor-class* of the parent. The parent actor must be previously defined and shall not inherit from itself.

The *replaces-specifier* allows an actor to wholly replace another actor. The new actor will assume the *editor-number* of the replacee and intercept any reference to the replacee when spawning. The replacee shall not be native.

The *editor-number* specifies a number which is used to reference this actor for static spawning in game levels. It shall be valid as a signed 16-bit integer and must be positive.

The *native* keyword marks an actor as native, meaning it exists in the engine's original code and should not be modified.

### 2.3.1 - Properties

>*property-assignment*:
>>*property* *property-args*

>*property*:
>>*identifier*

>*property-args:*
>>*property-arg*  
>>*property-arg*, *property-args*

>*property-arg*:
>>*string*  
>>*integer*  
>>*float*

A *property-assignment* directly assigns a constant value to the actor structure. The type and number of arguments for a property depends on the property being assigned.

The list of properties that are available is dependent on the *actor-class* and defined in section 4.

In the case of duplicated assignments, the last assignment is used.

**Note:** Due to the zero initialization of actor structures, integer/float properties default to 0 if not specified in any parent. String properties default to NULL.

### 2.3.2 - Flags

>*flag-assignment*:
>>\+ *flag*  
>>\- *flag*

>*flag*:
>>*identifier*

>*flag-combo*:
>>MONSTER  
>>PROJECTILE

A *flag-assignment* sets or removes a flag from an actor. If the assignment begins with the `+` token then the specified flag is to be set. If the assignment begins with the `-` token then the flag is to be removed.

The resolution of a *flag* is determined by the same rules as specified in the properties section.

If there are multiple assignments for the same flag, then the last assignment is used.

A *flag-combo* sets a defined set of flags. These act just as if the flags were set manually and specific flags in the combo can be unset.

`MONSTER` sets `SHOOTABLE`, `SOLID`, and `COUNTKILL`.
`PROJECTILE` sets `NOBLOCKMAP`, `MISSILE`, `DROPOFF`, and `NOGRAVITY`.

### 2.3.3 - States

>*states-decl*:
>>states { *state-seqₒₚₜ* }

>*states-seq*:
>>*state*  
>>*state* *states-seq*

>*state*:
>>*state-labelsₒₚₜ* *sprite-name* *frame-sequence* *duration* *frame-propertiesₒₚₜ* *codepointerₒₚₜ* *state-flow-specifierₒₚₜ*  

>*state-labels*:
>>*state-label* :  
>>*state-label* : *state-labels*

>*state-label*:
>>*state-label-name*

>*state-label-name*:
>>*identifier*

>*sprite-name*:
>>*identifier*  
>>*string*

>*frame-sequence*:
>>*identifier*  
>>*string*

>*duration*:
>>*integer*

>*frame-properties*:
>>bright  
>>fast  
>>offset ( *integer* , *integer* )

>*state-flow-specifier*:
>>goto *state-label* *\+ offsetₒₚₜ*  
>>loop  
>>stop  
>>wait  

The *states-decl* allows for the animation sequences to be defined for the actor. All states from the parent are accessible from the new actor.

There shall only be one *states-decl* in an actor definition.

The *sprite-name* should be exactly four characters long.

The *frame-sequence* may be of any length, but may only contain the characters A-Z or a-z. The characters `[`, `\`, or `]` may also be used if the string form use utilized.

If the *frame-sequence* contains more than one character, then the *state* is to be expanded into a sequence of states with all properties identical except for the frame which will be in the order given. The *state-labels* will refer to only the first state in the sequence. The *state-flow-specifier* modifies the flow of only the last state in the sequence.

The *bright* property will make the sprite render at full brightness, ignoring sector lighting. The *fast* property will make the frame take half as long on fast skill levels (used in Doom by the demons). The *offset* property will apply a render offset to the sprite with the given (X, Y) values (used in Doom for weapons).

The *duration* indicates the number of play sim tics that the state is to be held before going to the next state. A duration of -1 indicates infinite, and a duration of 0 indicates that only the codepointer should be executed and the following state should be used immediately. Should there be a 0 duration loop, the result is undefined.

Without the *state-flow-specifier*, the state should continue on to the next state in the sequence. If the end of the states block is reached and the state does not have a duration of -1 the result is undefined. If the *state-flow-specifier* is `goto`, then the next state will be set to the state referred to by the label. If the specifier is `wait` then the state will be re-entered after the duration. If the specifier is `loop` then the most recently declared *state-label* will be the next frame. If the specifier is `stop` then the actor is to be removed from the play sim.

Specifying a *state* without a previously declared *state-label* is ill-formed.

Goto resolution is to be performed after all states are parsed in order to allow state-labels defined after the goto to be referenced.

#### 2.3.3.1 - Special Sprites

The *sprite-name* `TNT1` indicates that no sprite should be rendered.

### 2.3.4 - Codepointer

>*codepointer*:
>>*identifier*  
>>*identifier* ( )  
>>*identifier* ( *arg-list* )

>*arg-list*:
>>*arg*  
>>*arg*, *arg-list*

>*arg*:
>>*identifier*  
>>*integer*  
>>*float*  
>>*string*  
>>*flag-list*

>*flag-list*:
>>*flag*  
>>*flag* \+ *flag-list*  
>>*flag* \| *flag-list*

>*flag*:
>>*identifier*

The *codepointer* is to be executed on the tic that the state is transitioned to. The name of a *codepointer* shall be at least 5 characters in length to distinguish it from a sprite name. Some codepointers, especially in MBF and later formats, can be optionally parameterized.

## 3 - Base Actor Hierarchy (WIP)

The basic actor hierarchy consists of native actors which provide various functions within the language. The hierarchy only needs to exist insofar as needed for performing inheritance resolution and do not need to exist outside of the DECLARATE language.

### 3.1 Doom v1.9 actors

* `thing` class
* * DoomPlayer
* * ZombieMan
* * ShotgunGuy
* * Archvile
* * ArchvileFire
* * Revenant
* * RevenantTracer
* * RevenantTracerSmoke
* * Fatso
* * FatShot
* * ChaingunGuy
* * DoomImp
* * Demon
* * Spectre
* * Cacodemon
* * BaronOfHell
* * BaronBall
* * HellKnight
* * LostSoul
* * SpiderMastermind
* * Arachnotron
* * Cyberdemon
* * PainElemental
* * WolfensteinSS
* * CommanderKeen
* * BossBrain
* * BossEye
* * BossTarget
* * SpawnShot
* * SpawnFire
* * ExplosiveBarrel
* * DoomImpBall
* * CacodemonBall
* * Rocket
* * PlasmaBall
* * BFGBall
* * ArachnotronPlasma
* * BulletPuff
* * Blood
* * TeleportFog
* * ItemFog
* * TeleportDest
* * BFGExtra
* * TechLamp
* * TechLamp2
* * Column
* * TallGreenColumn
* * ShortGreenColumn
* * TallRedColumn
* * ShortRedColumn
* * SkullColumn
* * HeartColumn
* * EvilEye
* * FloatingSkull
* * TorchTree
* * BlueTorch
* * GreenTorch
* * RedTorch
* * ShortBlueTorch
* * ShortGreenTorch
* * ShortRedTorch
* * Stalagtite
* * TechPillar
* * CandleStick
* * Candelabra
* * BloodyTwitch
* * Meat2
* * Meat3
* * Meat4
* * Meat5
* * NonsolidMeat2
* * NonsolidMeat4
* * NonsolidMeat3
* * NonsolidMeat5
* * NonsolidTwitch
* * DeadCacodemon
* * DeadMarine
* * DeadZombieMan
* * DeadDemon
* * DeadLostSoul
* * DeadDoomImp
* * DeadShotgunGuy
* * GibbedMarine
* * GibbedMarineExtra
* * HeadsOnAStick
* * Gibs
* * HeadOnAStick
* * HeadCandles
* * DeadStick
* * LiveStick
* * BigTree
* * BurningBarrel
* * HangNoGuts
* * HangBNoBrain
* * HangTLookingDown
* * HangTSkull
* * HangTLookingUp
* * HangTNoBrain
* * ColonGibs
* * SmallBloodPool
* * BrainStem

* `item` class
* * GreenArmor
* * BlueArmor
* * HealthBonus
* * ArmorBonus
* * BlueCard
* * RedCard
* * YellowCard
* * YellowSkull
* * RedSkull
* * BlueSkull
* * Stimpack
* * Medikit
* * Soulsphere
* * InvulnerabilitySphere
* * Berserk
* * BlurSphere
* * RadSuit
* * Allmap
* * Infrared
* * Megasphere
* * Clip
* * ClipBox
* * RocketAmmo
* * RocketBox
* * Cell
* * CellPack
* * Shell
* * ShellBox
* * Backpack
* * BFG9000
* * Chaingun
* * Chainsaw
* * RocketLauncher
* * PlasmaRifle
* * Shotgun
* * SuperShotgun

* `weapon` class
* * Fist
* * Pistol
* * Chainsaw
* * Shotgun
* * SuperShotgun
* * Chaingun
* * RocketLauncher
* * PlasmaRifle
* * BFG9000

## 3.2 Boom actors

* `thing` class
* * PointPusher
* * PointPuller

## 3.3 MBF actors

* `thing` class
* * PlasmaBall1
* * PlasmaBall2
* * MBFHelperDog

* `item` class
* * EvilSceptre
* * UnholyBible

## 3.4 Additional actors

* `thing` class
* * MusicChanger

## 4 - Properties & Flags

## 4.1 - Properties

### 4.1.1 - Doom v1.9 Thing Properties

* Health *integer*
* SeeSound *sound*
* ReactionTime *integer*
* AttackSound *sound*
* PainChance *integer*
* PainSound *sound*
* DeathSound *sound*
* Speed *integer*
* Radius *float*
* Height *float*
* Mass *integer*
* Damage *integer*
* ActiveSound *sound*

**Note:** The *sound* type indicates that the property takes a string refering to a SNDINFO logical sound. A SNDINFO implementation is not required to implement DECLARATE as a fixed lookup table can be used in its place.

**Note:** Health is the only property which doesn't match with vanilla Doom's internal name. This sets the `spawnhealth` field.

**Note:** The `speed` property is an integer, but will be converted to a fixed point value if the `MISSILE` flag is set.

### 4.1.2 - MBF21 Properties

* InfightingGroup *group*
* ProjectileGroup *group*
* SplashGroup *group*
* RipSound *string*
* AltSpeed *integer*
* MeleeRange *float*

### 4.1.3 - Additional properties

* DropItem *item*
* RenderStyle *style*
* Translation *integer*
* Obituary *string*
* HitObituary *string*
* SelfObituary *string*

**Note:** The *style* type indicates that the property takes a string "fuzzy" or "none".

### 4.1.4 - Recognized Thing States

* Spawn
* See
* Pain
* Melee
* Missile
* Death
* XDeath
* Raise

### 4.1.5 - Recognized Weapon States

* Select
* Deselect
* Ready
* Fire
* Flash

**Note:** The select state is known internally as `upstate`, the deselect as `downstate`.

## 4.2 - Flags

### 4.2.1 - Doom v1.9 Flags

* SPECIAL
* SOLID
* SHOOTABLE
* NOSECTOR
* NOBLOCKMAP
* AMBUSH
* JUSTHIT
* JUSTATTACKED
* SPAWNCEILING
* NOGRAVITY
* DROPOFF
* PICKUP
* NOCLIP
* SLIDE
* FLOAT
* TELEPORT
* MISSILE
* DROPPED
* SHADOW
* NOBLOOD
* CORPSE
* INFLOAT
* COUNTKILL
* COUNTITEM
* SKULLFLY
* NOTDMATCH
* TRANSLATION1
* TRANSLATION2

### 4.2.1 - Boom and MBF flags

* TOUCHY
* BOUNCES
* FRIEND
* TRANSLUCENT

### 4.2.2 - MBF21 Flags

* LOGRAV
* SHORTMRANGE
* DMGIGNORED
* NORADIUSDMG
* FORCERADIUSDMG
* HIGHERMPROB
* RANGEHALF
* NOTHRESHOLD
* LONGMELEE
* BOSS
* MAP07BOSS1
* MAP07BOSS2
* E1M8BOSS
* E2M8BOSS
* E3M8BOSS
* E4M6BOSS
* E4M8BOSS
* RIP
* FULLVOLSOUNDS

# 5. - Codepointers

### 5.1 - Doom v1.9 Weapon Codepointers

* A_Light0
* A_WeaponReady
* A_Lower
* A_Raise
* A_Punch
* A_ReFire
* A_FirePistol
* A_Light1
* A_FireShotgun
* A_Light2
* A_FireShotgun2
* A_CheckReload
* A_OpenShotgun2
* A_LoadShotgun2
* A_CloseShotgun2
* A_FireCGun
* A_GunFlash
* A_FireMissile
* A_Saw
* A_FirePlasma
* A_BFGsound
* A_FireBFG
* A_BFGSpray

### 5.2 - Doom v1.9 Thing Codepointers

* A_BFGSpray
* A_Explode
* A_Pain
* A_PlayerScream
* A_Fall
* A_XScream
* A_Look
* A_Chase
* A_FaceTarget
* A_PosAttack
* A_Scream
* A_SPosAttack
* A_VileChase
* A_VileStart
* A_VileTarget
* A_VileAttack
* A_StartFire
* A_Fire
* A_FireCrackle
* A_Tracer
* A_SkelWhoosh
* A_SkelFist
* A_SkelMissile
* A_FatRaise
* A_FatAttack1
* A_FatAttack2
* A_FatAttack3
* A_BossDeath
* A_CPosAttack
* A_CPosRefire
* A_TroopAttack
* A_SargAttack
* A_HeadAttack
* A_BruisAttack
* A_SkullAttack
* A_Metal
* A_SpidRefire
* A_BabyMetal
* A_BspiAttack
* A_Hoof
* A_CyberAttack
* A_PainAttack
* A_PainDie
* A_KeenDie
* A_BrainPain
* A_BrainScream
* A_BrainDie
* A_BrainAwake
* A_BrainSpit
* A_SpawnSound
* A_SpawnFly
* A_BrainExplode

### 5.3 - MBF Thing Codepointers

* A_Detonate
* A_Mushroom(*float* angle, *float* speed)
* A_Die
* A_Spawn(*identifier* item, *float* z_pos)
* A_Turn(*integer* degrees)
* A_Face(*integer* degrees)
* A_Scratch(*integer* damage, *sound* sound)
* A_PlaySound(*sound* sound, *boolean* fullvolume)
* A_RandomJump(*identifier* state, *integer* probability)
* A_LineEffect(*integer* special, *integer* tag)
* A_BetaSkullAttack
* A_Stop

### 5.4 - MBF Weapon Codepointers

* A_FireOldBFG

### 5.5 - MBF21 Thing Codepointers

* A_SpawnObject(*identifier* thing, *float* angle, *float* x_ofs, *float* y_ofs, *float* z_ofs, *float* x_vel, *float* y_vel, *float* z_vel)
* A_MonsterProjectile(*identifier* thing, *float* angle, *float* pitch, *float* hoffset, *float* voffset)
* A_MonsterBulletAttack(*float* hspread, *float* vspread, *integer* numbullets, *integer* damagebase, *integer* damagedice)
* A_MonsterMeleeAttack(*integer* damagebase, *integer* damagedice, *sound* sound, *float* range)
* A_RadiusDamage(*integer* damage, *integer* radius)
* A_NoiseAlert
* A_HealChase(*identifier* state, *sound* sound)
* A_SeekTracer(*float* threshold, *float* maxturnangle)
* A_FindTracer(*float* fov, *integer* rangeblocks)
* A_ClearTracer
* A_JumpIfHealthBelow(*identifier* state, *integer* health)
* A_JumpIfTargetInSight(*identifier* state, *float* fov)
* A_JumpIfTargetCloser(*identifier* state, *float* distance)
* A_JumpIfTracerInSight(*identifier* state, *float* fov)
* A_JumpIfTracerCloser(*identifier* state, *float* distance)
* A_JumpIfFlagsSet(*identifier* state, *flag-list* flags)
* A_AddFlags(*flag-list* flags)
* A_RemoveFlags(*flag-list* flags)

### 5.6 - MBF21 Weapon Codepointers (WIP)

* A_WeaponProjectile
* A_WeaponBulletAttack
* A_WeaponMeleeAttack
* A_WeaponSound
* A_WeaponAlert
* A_WeaponJump
* A_ConsumeAmmo
* A_CheckAmmo
* A_RefireTo
* A_GunFlashTo
