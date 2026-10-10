# GMODIFY Guide for CoffeeMud 5.11.0.1

Oct 10, 2026 · Headsarm

GMODIFY finds every object that matches some conditions and can change a field on all of them at once. It saves those changes straight to the database. Everything here was checked against `com/planet_ink/coffee_mud/Commands/GModify.java`, not just the in-game help, and the source disagrees with the help in a few places (flagged below).

## What it is and what it touches

One command does two jobs:

- **Search:** without `CHANGE=`, it only lists matches (`Matched on <name> from <room id>`). Nothing is saved.
- **Search and change:** with `CHANGE=`, it edits every match and saves it.

What it looks at, in each room in scope:

- the **area** and the **room** themselves
- **items** on the floor, including items inside containers
- **mobs** saved with the room (NPCs, not players or their followers)
- **items** carried or worn by those mobs
- **shopkeeper stock**

What it does **not** look at:

- **Player inventories and worn gear.** Players aren't saved with rooms, so a ROOM, AREA or WORLD run skips them and everything they carry. The PLAYERS scope edits only the player's own fields, never their items.
- Exits.

Every change is written to `mud.log` under the `GMODIFY` tag, together with the full command you typed. That log is your only undo record, so keep it.

## Command shape

```
gmodify [SCOPE] [WHEN=]conditions [CHANGE=changes]
```

- **SCOPE** says where to look. It's optional, but always type it.
- **conditions** decide which objects match. `WHEN=` in front is optional.
- **changes** say what to set on each match. Leave them out to preview.
- Conditions and changes are joined with `&&`. Spaces in values are fine and need no quotes. Don't add quotes: they end up in the value.

| Scope | What it searches | Permission needed |
| --- | --- | --- |
| *(none)* | **Only the room you're standing in** | GMODIFY in that room |
| `room` | The room you're standing in | GMODIFY in that room |
| `area` | Every room in your current area | GMODIFY in this room, then checked per room |
| `world` | Every room in every area you're allowed to edit | GMODIFY everywhere |
| `players` | Every player in the database, offline ones too (loaded, checked, saved) | GMODIFY and CMDPLAYERS everywhere |
| `#tagname` | Only objects carrying that tag, in rooms that hold tagged objects | GMODIFY in each of those rooms |

**Help text vs source:** the help says leaving out the scope searches the whole world. In this build it searches **only your current room**. Always type the scope you mean.

Two more commands help:

- `gmodify ?` lists every valid field name on your server, including your factions.
- Turning on the `GMODIFY` debug flag makes the command print every comparison it makes, which helps when a condition won't match. It's very noisy on a big scope.

## Safe workflow

Preview first, then change the smallest scope, then go wider. A change run saves immediately, and there's no undo.

1. **Back up.** Copy the database, or at least note the time so you can find the run in `mud.log`.
2. **Find the field name.** Run `gmodify ?`, or look at the object with `stat` or `modify` to see what the field holds.
3. **Preview in one room.** Stand in a room that has a target and run the command **without** `CHANGE=`:

   ```
   gmodify room classtype=armor&&secret=[ss]cybernetic
   ```
4. **Preview the real scope.** Run the same thing with `area` or `world`. Read the `Matched on ...` list. Anything you didn't expect would be changed too.
5. **Narrow it** until the list is exactly right. Add a `CLASSTYPE` or `CLASS` condition so areas and rooms can't match by accident.
6. **Change one room first.** Add `CHANGE=` and run it in `room` scope. Check the result with `stat`.
7. **Run the full scope.** Use the identical conditions you previewed.

What happens to rooms during a run:

- **Preview** reads each room's saved copy from the database, so it shows what's saved, not live edits you haven't saved yet.
- **Change** resets each room in scope from the database before editing it. Anything unsaved there is lost, and mobs and items go back to their saved state.
- Each area is briefly frozen while its rooms are processed.

## Conditions

Each condition is `FIELD` + operator + `VALUE`. Field names are case-insensitive.

| Operator | Meaning | Example |
| --- | --- | --- |
| `=` | Equals | `name=a rusty sword` |
| `!=` | Doesn't equal | `class!=GenMob` |
| `>` `<` `>=` `<=` | Compares as numbers when both sides are whole numbers, otherwise alphabetically | `level>=20` |
| `$` | Matches a regular expression | `name$[ss]sword\|blade` |

Put these modifiers right after the operator. They also combine, as in `[ss][cs]`:

| Modifier | Effect | Example |
| --- | --- | --- |
| *(none)* | Whole value, case ignored | `name=a rusty sword` |
| `[ss]` | Substring: the value only has to appear somewhere in the field | `name=[ss]rusty` |
| `[cs]` | Case-sensitive | `class=[cs]GenMob` |

Things the help doesn't tell you:

- **`$` without `[ss]` is case-sensitive and must match the whole field,** even though the help says case is ignored. Write `$[ss]` to search anywhere with case ignored.
- **`!=` also matches objects that don't have that field at all,** because a missing field counts as empty. `secret!=cybernetic` alone matches every room and area. Pair it with `CLASSTYPE`.
- **Numbers are compared as whole numbers.** Decimals are cut off.

### Joining conditions

- `&&` means AND: `classtype=armor&&level>10`.
- **`||` (OR) doesn't work in this build.** The parser looks for `|&` instead of `||`. So `name=bob||name=joe` becomes one condition whose value is the literal text `bob||name=joe`, and it matches nothing.
- For OR on one field, use a regex: `name$[ss]^(bob|joe)$`. For OR across different fields, run the command twice.
- Conditions are read left to right, with no grouping or brackets.

## Changes

Write `CHANGE=FIELD=VALUE`. Join several with `&&`: `change=level=10&&rejuv=500`. Only `=` means anything here, and a field is skipped if it already holds the new value.

**Without conditions, a change hits everything.** `gmodify area change=level=5` edits the area, every room, mob and item in it. Always give conditions, or use a `#tag` scope.

### Replacing only part of a value

You can swap just the matched piece instead of the whole value. This only works when the same field has `[ss]` in **both** the condition and the change:

```
gmodify world class=GenMob&&name=[ss]city change=name=[ss]town
```

"the city guard" becomes "the town guard". The help's example `change=name=[ss]` (an empty value) deletes the matched text. A regex works too (`secret$[ss]cyber\w*`): the part the regex matched is what gets replaced.

### Special change fields

| Field | What it does | Example |
| --- | --- | --- |
| `DESTROY` | `true` deletes the object. In `players` scope it **deletes the player**. On a room it deletes the room | `change=destroy=true` |
| `CLASS` | Switches the object to another Java class. Level, ability and saved data carry over | `change=class=GenWeapon` |
| `ADDAFFECT` / `DELAFFECT` | Adds or removes a permanent effect/property. Parameters go in brackets | `change=addaffect=Prop_HaveZapper(-LEVEL +>10)` |
| `ADDBEHAVIOR` / `DELBEHAVIOR` | Adds or removes a behavior | `change=addbehavior=Scriptable(LOAD=scripts/cyber.script)` |
| `ADDABILITY` / `DELABILITY` | Adds or removes a skill (mobs) | `change=addability=Skill_Dodge` |
| `DELFACTION` | Removes a faction from a mob | `change=delfaction=myfaction.ini` |
| *faction id* | Sets a faction value the mob already has | `change=myfaction.ini=500` |
| `REJUV` | Respawn ticks. `0` turns rejuv off | `change=rejuv=0` |
| `RESOURCE` | Exact material, such as STEEL or OAK | `change=resource=STEEL` |
| `MATERIALTYPE` | Material family, such as METAL or WOODEN | `change=materialtype=METAL` |
| `GENDER` | `M`, `F`, `N`, or `?` for random M/F | `change=gender=?` |
| `REBALANCE` | Rebuilds an item's stats for its level. A number rebalances as if it were that level | `change=rebalance=20` |

**Use `REBALANCE` on items only.** On mobs, the code casts the mob to an item, so the run errors out.

The `Scriptable(LOAD=...)` value is a placeholder showing the format. Put your real script path there.

## Worked examples

Each pair is the preview first, then the change. Run the preview, read the list, and only then run the change.

**Make every cyberware secret exactly "cybernetic".** This fixes old items like "cybernetic arm". It won't touch copies players already own (see What it touches). Repeat with `classtype=item` and `classtype=weapon` if some cyberware isn't armor.

```
gmodify world classtype=armor&&secret=[ss]cybernetic
gmodify world classtype=armor&&secret=[ss]cybernetic change=secret=cybernetic
```

**Rename part of mob names in one area.** Only the matched text changes.

```
gmodify area class=GenMob&&name=[ss]cityguard
gmodify area class=GenMob&&name=[ss]cityguard change=name=[ss]town guard
```

**Find items that carry a given effect or behavior.** `AFFBEHAV` holds an item's effects and behaviors as text, so a substring search works.

```
gmodify world classtype=armor&&affbehav=[ss]Prop_WearZapper
```

**Raise low-level mobs in an area.**

```
gmodify area classtype=mob&&level<5
gmodify area classtype=mob&&level<5 change=level=5
```

**Delete every copy of a junk item.**

```
gmodify world class=GenItem&&name=a rusty sword
gmodify world class=GenItem&&name=a rusty sword change=destroy=true
```

**Match two names (the OR workaround).**

```
gmodify area classtype=mob&&name$[ss]^(the guard|the captain)$
```

**Find players by class.** Preview only. This loads every saved player, so it's slow on a large database.

```
gmodify players charclass=Fighter
```

## Field reference

These are the common fields, taken from the source. Weapons, armor, containers and so on add more of their own. `gmodify ?` prints the full list for your server.

| Applies to | Fields |
| --- | --- |
| Anything | `CLASS` (Java class, e.g. GenItem, GenMob), `CLASSTYPE` (see below), `NAME`, `DISPLAY`, `DESCRIPTION`, `LEVEL`, `ABILITY`, `REJUV`, `DISPOSITION`, `SENSES`, `WEIGHT`, `HEIGHT`, `ARMOR`, `DAMAGE`, `ATTACK`, `AFFBEHAV`, `IMG`, `TAGS` |
| Generic items | `SECRET` (secret identity), `USES`, `PROPERWORN`, `WORNAND`, `BASEGOLD`, `MATERIAL`, `RESOURCE`, `MATERIALTYPE`, `ISREADABLE`, `ISGETTABLE`, `ISDROPPABLE`, `ISREMOVABLE`, `READABLETEXT` |
| Generic mobs | `RACE`, `MONEY`, `VARMONEY`, `ALIGNMENT`, `SPEED`, `ABLES`, `INVENTORY`, `TATTS`, `EXPS`, `FACTIONS`, `GENDER`, any faction id |
| Players only | `CHARCLASS`, character stats like `STRENGTH`, and state fields like `HITPOINTS` |

`CLASSTYPE` can be any of these: `MOB`, `ITEM`, `WEAPON`, `ARMOR`, `MISCMAGIC`, `CLANITEM`, `TECH`, `COMPTECH`, `SOFTWARE`, `LOCALE` (rooms), `AREA`.

Note that `ITEM` covers only plain items: weapons are `WEAPON` and armor is `ARMOR`. To be sure, check a known object with `stat` first, or preview with `classtype=...` alone.

## Gotchas and troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Matches nothing, though you know targets exist | No scope given, so it only searched your current room | Add `area` or `world` |
| `'X' is an unknown field name` | Typo, or a field that doesn't exist on this server | Check `gmodify ?` |
| `does not contain an equation divider` | A condition or change is missing its `=` | Every piece needs `FIELD=VALUE`, including inside `CHANGE=` |
| An OR condition never matches | `\|\|` isn't supported in this build | Regex alternation, or run twice |
| A regex matches in a tester but not here | `$` without `[ss]` is case-sensitive and must match the whole field | Use `$[ss]` |
| Matches rooms and areas you didn't want | `!=`, or a field that rooms/areas also have | Add `classtype=...` |
| Player gear wasn't changed | Player inventories are never scanned | Edit those players by hand, or have a script fix items on wear |
| Unsaved builder work in a room disappeared | Change runs reset every room in scope first | Save rooms before a run |
| The run errors out on mobs | `REBALANCE` only works on items | Add `classtype=armor` (or another item type) to the conditions |
| A value has stray quote marks | Quotes are kept as part of the value | Type values with spaces and no quotes |

To stop a long run, disconnect your session. The command checks between objects and stops cleanly, but anything it already changed stays changed.

If a run went wrong, search `mud.log` for `GMODIFY`. Each line reads *The X field on Y in room Z was changed from A to B*. Use those lines to put the values back.
