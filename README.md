# Minecraft 26.x Modding Notes

Things that cost me a day each while porting mods to **Minecraft 26.1 / 26.2**
(NeoForge). Every entry has the evidence — the actual line from decompiled
Minecraft, or a measurement from a running server. Not "I think", not "somebody
on Discord said".

Minecraft's calendar versions renamed a lot. If you are porting from 1.21.x and
something silently does nothing, it is probably in here.

> **Verified on NeoForge 26.2.0.88, Minecraft 26.2** (`DataVersion` 4903,
> `resource_major` 88, `data_major` 107) — and only there. I did not test 26.1
> or 26.3. Most of these are renames that landed with the calendar versioning,
> so they likely apply, but the numbers above certainly differ. Check
> `version.json` inside your `client.jar` for yours.

---

## The one that cost the most: `useWithoutItem` is not called by default

**Symptom:** Right-clicking your block does nothing. No menu, no message, no
error. Placing the block works, commands work, everything else works.

**Cause:** In `ServerPlayerGameMode.useItemOn`:

```java
InteractionResult itemUse = state.useItemOn(player.getItemInHand(hand), level, player, hand, hitResult);
if (itemUse.consumesAction()) {
    return itemUse;
}
if (itemUse instanceof InteractionResult.TryEmptyHandInteraction && hand == InteractionHand.MAIN_HAND) {
    InteractionResult use = state.useWithoutItem(level, player, hitResult);
```

`useWithoutItem` is reached **only** when `useItemOn` returned
`InteractionResult.TRY_WITH_EMPTY_HAND` first. Returning `PASS` — the obvious
"I don't handle this" value — silently ends the chain.

```java
// WRONG: useWithoutItem is never reached
return InteractionResult.PASS;

// RIGHT
return InteractionResult.TRY_WITH_EMPTY_HAND;
```

**Why it hides so well:** `setPlacedBy`, commands, block entities, saved data —
all of that keeps working. Only the single most-used interaction in the game is
dead, and nothing logs a word about it.

---

## Renamed: things you will not guess

| 1.21.x | 26.x | Note |
|---|---|---|
| `ClickType` | `ContainerInput` | `PICKUP` (button 0/1 = left/right), `QUICK_MOVE` = shift |
| `entity.getTags()` | `entity.entityTags()` | |
| `EntityType.ARMOR_STAND` | `EntityTypes.ARMOR_STAND` | **two different classes** — `EntityType` only holds `CODEC`/`STREAM_CODEC` now |
| `Items.GRAY_STAINED_GLASS_PANE` | `Items.STAINED_GLASS_PANE.gray()` | dyed variants are a `ColorCollection` |
| `level.random` | `level.getRandom()` | field is `protected` |
| `ResourceLocation` | `Identifier` | |
| `ResourceKey.location()` | `ResourceKey.identifier()` | |
| `level.getSharedSpawnPos()` | `level.getRespawnData().pos()` | |
| `displayClientMessage(...)` | `sendSystemMessage(Component, boolean)` | |
| `saveAdditional(CompoundTag)` | `saveAdditional(ValueOutput)` | same for `loadAdditional(ValueInput)` |

`ChunkPos.x` and `.z` are **private** now, and there are no getters. Work with
plain ints or `getMinBlockX() >> 4`.

Game rules are snake_case: `keep_inventory`, `log_admin_commands`,
`send_command_feedback`. `logAdminCommands` returns *Incorrect argument for
command*.

**How to check instead of guessing:**

```bash
javap -cp <recompiled-minecraft>.jar net.minecraft.world.item.Items | grep -i glass
```

The decompiled source sits in the ModDevGradle cache under
`caches/neoformruntime/intermediate_results/decompile_*_output.jar`.

---

## `SavedData`: add fields only with `optionalFieldOf`

Adding a required field to an existing codec makes reading fail — and the
**whole list** is gone, silently.

```java
// safe: existing saves keep working
Codec.STRING.optionalFieldOf("symbol", "").forGetter(Punkt::symbol)
```

🔴 And that reasoning is not a test. Copy the real `.dat` from the live server
onto a test server and start it. Five entries in, five entries out — that is
the test.

The file is **not** under `world/data/`. It lives in
`world/dimensions/minecraft/overworld/data/<modid>/`.

---

## A freshly spawned entity is not findable in the same tick

`addFreshEntity` returns, and `getEntitiesOfClass` right afterwards finds
**nothing**. The entity only enters the chunk index next tick.

This bites when you check "is there already one here?" before spawning: the
check misses your own entity and you create a second one. If that code runs
per keystroke — for example in `AnvilMenu.setItemName`, which is called for
**every character typed** — you get one entity per letter.

Two fixes, both needed:

1. Do such work when the window **closes**, not while typing.
2. Keep a small map of just-created entities and look there first.

---

## A mod jar does not contain what it uses

BlueMap's jar (6.7 MB) contains **only** `de/bluecolored`. `Vector3d` appears
in every marker signature but is not in there — it is loaded at runtime from a
jar-in-jar.

So you need the dependency separately at compile time, and whether it is
*visible* to your mod at runtime is a different question. Don't hope — probe it
at startup:

```java
var probe = new com.flowpowered.math.vector.Vector3d(0, 0, 0);
LOGGER.info("helper classes reachable ({})", probe);
```

The log then answers it, and tells you why:

```
mods/bluemap-5.27-neoforge.jar > flow-math-1.0.3.jar
```

NeoForge puts embedded jars on the shared module path, so it worked. That was
luck, not design.

**Pattern for optional mod dependencies:** put every foreign type behind a
separate nested class, and only touch that class after `Class.forName(...)`
confirmed the mod exists. A foreign type in the *outer* class's signature takes
the server down when the class loads.

---

## Access transformers, and how to verify them

`ArmorStand.setMarker` is not public, so a nametag hologram gets a hitbox and
blocks the spot. The supported fix is a file NeoForge finds on its own:

```
src/main/resources/META-INF/accesstransformer.cfg
---
public net.minecraft.world.entity.decoration.ArmorStand setMarker(Z)V
```

🔴 **A typo in that line does not fail the build.** It is silently ignored, and
you find out in-game. Check the result on the spawned entity:

```bash
rcon-cli "data get entity @e[tag=my_tag,limit=1]"
# → Marker: 1b, Invisible: 1b, CustomNameVisible: 1b
```

---

## Resource and data packs: three numbers, two kinds

`pack_format` alone is no longer enough:

```
Pack declares support for version newer than 81, but is missing
mandatory fields min_format and max_format
```

```json
{ "pack": { "pack_format": 88, "min_format": 88, "max_format": 88 } }
```

The number differs per kind, and it is in `version.json` inside `client.jar`:

| Kind | Field | 26.2 |
|---|---|---|
| Resource pack | `pack_version.resource_major` | **88** |
| Data pack | `pack_version.data_major` | **107** |

Data pack function folder is `function` — singular, it used to be `functions`.

There is **no emissive flag** for block models in 26.2. Searched every vanilla
block model: the only such switch is `force_translucent`, 163 times, all glass.

---

## Several blocks share one model

`infested_stone` has no model of its own — its blockstate points at
`minecraft:block/stone`:

```
infested_stone -> ['minecraft:block/stone', 'minecraft:block/stone_mirrored']
```

Rewriting models by a block list therefore rewrites more than you listed. One
entry made *all stone in the world* visible in an X-ray pack, and nothing
complained: every file valid, pack built fine.

Build the list that must win **first**, lock those model names, and **report**
collisions instead of overwriting silently. Then open one of the generated
files and look — don't read your own script's summary.

---

## Entities you place in someone's world

Anything you spawn is saved and stays. Three rules that emerged the hard way:

1. **Tag everything you create** so it is identifiable after a restart.
2. **Make it self-healing**, not one-shot. The chunk gets loaded long after the
   block was placed; a one-time spawn never happens.
3. **Give yourself a cleanup command** that finds orphans across all loaded
   entities — the periodic pass is too expensive for that, a command is not.

Chunk force-loading is worse: **the world remembers it, not your mod.** Set it
and remove the mod, and it runs forever. Keep a ledger of every lock you set,
release it when the reason goes away, and offer a manual release.

---

## Cross-dimension teleporting takes the passengers along

`Entity.teleport(TeleportTransition)` moves passengers and re-seats them
afterwards — for same-dimension and cross-dimension alike:

```java
private Entity teleportCrossDimension(...) {
    List<Entity> oldPassengers = this.getPassengers();
    ...
    for (Entity newPassenger : newPassengers) {
        newPassenger.startRiding(newEntity, true, false);
    }
```

So to move a mounted player, teleport the **vehicle**, not the player.
Teleporting the player leaves the horse behind.

🔴 Crossing dimensions creates a **new** entity. Re-attach leashes to the
**return value** of `teleport`, not to the old object.

---

## Testing when you cannot start a client

Most of a mod's logic is testable from a running server. I ended up with a
`/mymod selftest` command that runs ~60 assertions over RCON — no player
needed, working from world spawn.

It found two real bugs on the **live** server that the test server had passed,
and it answers from a terminal whether the logic is sound.

Three rules it taught me:

* **Check the test before the code.** A red result means one of the two is
  wrong, and it is often the test. Mine asserted that a point does not find
  itself as a duplicate — while both test points had the same name, so finding
  the other one was correct.
* **A test that leaves litter is worse than no test.** Mine placed marker
  entities at world spawn — which on that server is the middle of someone's
  village. Six of them piled up before anyone noticed. It now asserts its own
  cleanup.
* **The workbench must not be someone's living room.** World spawn is
  convenient because it is reachable without a player. That somebody lives
  there was in none of my assertions.

What it cannot cover: anything a player has to click. That is exactly where the
`TRY_WITH_EMPTY_HAND` bug sat for days, under green test runs.

---

## Server-side mods cannot use translation keys

If your mod runs on the server only — no client install, using vanilla
`GENERIC_9x6` containers instead of a custom screen — then `Component.translatable`
is useless. **The client resolves translation keys against its own language
files, and it does not have your mod.** Players see the raw key:

```
bestenliste.category.mined
```

That is the price of "no client mod required", and it is easy to miss because
it works perfectly in a development environment where the mod *is* on both
sides.

The way out is that the server knows the player's language setting:

```java
public java.lang.String getLanguage();          // on ServerPlayer, e.g. "de_de"
public ClientInformation clientInformation();
```

So build the text on the server and pick the language yourself:

```java
String lang = player.getLanguage();             // "de_de", "en_us", ...
String text = TEXTS.getOrDefault(lang.substring(0, 2), ENGLISH).get(key);
player.sendSystemMessage(Component.literal(text));
```

Slightly more work than a language file, and you carry the strings in code —
but it is the only way a server-side mod speaks more than one language.

## Renamed and moved in 26.2 — the ones that stop a build

Found while writing a mod that draws its own HUD. Every one of these shows up
as `cannot find symbol`, and none of them is in a wiki yet.

| Before | In 26.2 |
|---|---|
| `BlockEvent.BreakEvent` | `event.level.block.BreakBlockEvent` |
| `PickaxeItem`, `SwordItem` | **gone** — tools are data-driven now |
| `GuiGraphics` in `RenderGuiLayerEvent` | `GuiGraphicsExtractor` (it *is* the drawing surface) |
| `GuiGraphics.drawString(...)` | `.text(...)`, plus `.centeredText(...)` |
| `FMLEnvironment.dist` | `FMLEnvironment.getDist()` |
| `entity.projectile.AbstractArrow` | `entity.projectile.arrow.AbstractArrow` |
| `CommandSourceStack.hasPermission(2)` | `Commands.hasPermission(new PermissionCheck.Require(Permissions.COMMANDS_GAMEMASTER))` |
| `AbstractArrow.getBaseDamage()` | gone — only the setter is left |

**`ShovelItem`, `AxeItem` and `BowItem` still exist** — only pickaxe and sword
disappeared. Code that mixes the survivors with a workaround for the missing
two breaks again at the next change. Use item tags instead:
`ItemTags.PICKAXES`, `SHOVELS`, `AXES`, `SWORDS`. They catch tools from other
mods for free. There is no bow tag; `BOW_ENCHANTABLE` is something else and
includes the crossbow.

`GuiGraphicsExtractor` is a Minecraft class, not a NeoForge one
(`net.minecraft.client.gui`). The permission pattern comes straight from
vanilla's `GameModeCommand`.

## A client-only handler must never be named on the server

```java
PayloadRegistrar registrar = event.registrar("1");
if (FMLEnvironment.getDist().isClient()) {
    ClientNetwork.register(registrar);         // with a handler
} else {
    registrar.playToClient(TYPE, CODEC);       // without one
}
```

Writing `ClientDisplay::receive` directly in the registration makes Java load
that class **when registering** — on a dedicated server too, which has no
screen classes. The result is a `NoClassDefFoundError` at startup, on exactly
the machine you never test from your IDE. The `playToClient` overload without a
handler exists for this.

What is verified here: a dedicated server starts cleanly with this split. The
failure itself was not reproduced — the split was built in from the start.

## Network channels are an entry ticket

From NeoForge's own `PayloadRegistrar.optional()`:

> If any non-optional payloads are missing during a connection attempt, the
> connection will fail.

Every `playToClient(...)` is mandatory by default. Put such a mod on a server
and every player without it is locked out. (Quoted from the source; not yet
observed with a real client.) Right for a mod that needs a client part
anyway; worth knowing before you drop the jar on a live server. Call
`registrar.optional()` first if the mod should tolerate vanilla clients.

## `Level.destroyBlock` drops as if no tool was used

An ability that breaks extra blocks — a vein, a tree — is tempting to write
with `level.destroyBlock(pos, true, player)`. It looks right and it drops
items. But from `Level.java`:

```java
Block.dropResources(blockState, this, pos, blockEntity, breaker, ItemStack.EMPTY);
```

The tool is **`ItemStack.EMPTY`**. Fortune and Silk Touch do nothing, and ores
give no experience. A vein ability built like this gives a Fortune III player
*fewer* diamonds than mining by hand. It also does not fire `BreakBlockEvent`,
and does not wear the tool down.

What a left click does is `player.gameMode.destroyBlock(pos)`: real drops with
the tool's enchantments, experience, durability, statistics, spawn protection
— and it **fires `BreakBlockEvent`** (`CommonHooks.fireBlockBreak` in
`ServerPlayerGameMode`). So if the ability itself listens to that event, it
starts again for every block it breaks. Guard it:

```java
private static final ThreadLocal<Boolean> ACTIVE = ThreadLocal.withInitial(() -> false);
```

A `ThreadLocal`, not a static flag: a server may tick dimensions on separate
threads, and a shared flag would let one player's ability silence another's.

Measured with a fake player (`FakePlayerFactory`) on a test server: a
27-block diamond vein broken through `gameMode.destroyBlock` with a Fortune III
pickaxe gave **57 diamonds** (expected mean 59; without Fortune it would be
27).

**Leaves cost durability too** when mined through the game mode:
`Item.mineBlock` damages the tool for every block whose hardness is not 0, and
leaves have 0.2. A tree-felling ability that mines leaves wears the axe down by
sixty points on a big oak. Breaking leaves with `level.destroyBlock` instead
lets them fall like natural decay — saplings and apples, no durability.

## Saved data lives per namespace

World data is at
`world/dimensions/minecraft/overworld/data/<modid>/<name>.dat`. Changing the
mod id means a new folder and the old data is simply not read any more. When
renaming, **stop** the server rather than restarting it: the old mod writes
its file on shutdown, so a file moved away while it runs comes straight back.

---

## Odds and ends

* **`forceload add` takes effect next tick.** Placing something right after it
  lands in an unloaded chunk.
* **"No entity was found" does not mean "gone".** It means "not in a loaded
  chunk". Deleting on that basis destroys data.
* **Statistics files are written on save**, not live. `save-all` first, then
  read.
* **`text_display` wants an NBT compound**, not a JSON string — pass a JSON
  string and you see the markup as text in-game.
* **No emoji in game text** unless you checked it survives the build, and none
  in PowerShell scripts — Windows PowerShell 5.1 reads `.ps1` without BOM as
  ANSI.
* **A `-sources.jar` is not a mod.** 572 `.java` files, zero classes — and it
  can be marked "primary" on the download page. It crashes the client.
* **Worlds only move forward.** There is no downgrade. Ever.

---

## Where this comes from

A private server for two people, running Minecraft 26.2 on NeoForge with
self-built mods. Everything here was hit in practice between 17 and 23
September 2026, and every claim was verified against decompiled source or a
running server before it was written down.

Corrections welcome — if something here is wrong, it cost somebody a day, and
that is worth fixing.

One of those mods is public: [Server Leaderboards](https://github.com/shinobiofgloom/server-leaderboards),
which is where the "server-side mods cannot use language files" entry above
came from.
