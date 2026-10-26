# Prompt: Build the "R3ckxBot" Minecraft practice-bot plugin

You are an expert Minecraft plugin developer (Java 21, Paper API, Citizens API). Build a **complete, production-ready, compilable** Paper plugin from scratch. Write every file in full. Do not leave TODOs, stubs, placeholder methods or "implement this yourself" comments. Do not copy code or assets from any existing paid plugin; write everything originally.

---

## 1. Identity and branding (read carefully)

- The plugin's name in `plugin.yml` is **`R3ckxBot`**. That is the only place this name appears.
- Everywhere else, the code and every player-facing element use **`KittyFFA`**:
  - Base package: `dev.kittyffa`
  - Main class: `dev.kittyffa.KittyFFA`
  - All class names, config keys, permissions (`kittyffa.*`), messages, GUI titles, chat prefixes, bot nametags and the database file use KittyFFA / KittyFFAPVPBot.
  - The product is presented in-game as **"KittyFFAPVPBot"**.
- **Never show "R3ckxBot" to players.** Implement all of the following:
  - Set `prefix: KittyFFA` in `plugin.yml` so console log lines show `[KittyFFA]`.
  - Do **not** declare commands in `plugin.yml`. Register them at runtime with `Bukkit.getCommandMap().register("kittyffa", command)` so the namespaced form is `/kittyffa:bot`, never `/r3ckxbot:bot`.
  - Add a listener that hides the plugin from players without `kittyffa.admin`. It intercepts `/pl`, `/plugins`, `/bukkit:pl`, `/bukkit:plugins`, `/ver`, `/version`, `/about`, `/icanhasbukkit` and their `bukkit:` forms (case-insensitive). For `/plugins` it sends its own plugin list with R3ckxBot left out, formatted like Paper's. For `/version R3ckxBot` it replies as if the plugin doesn't exist.
  - Strip R3ckxBot from tab completions (`TabCompleteEvent` and `AsyncTabCompleteEvent`).
  - Strip `r3ckxbot:` namespaced commands from `PlayerCommandSendEvent`.

---

## 2. Target platform

- **Paper / Purpur 1.21 to 1.21.11**, Java 21. Folia is not required.
- `api-version: '1.21'`
- **Hard dependency on Citizens** (`depend: [Citizens]`). Bots are Citizens player NPCs (`EntityType.PLAYER`).
- Build with **Gradle Kotlin DSL** (`build.gradle.kts`, `settings.gradle.kts`):
  - `compileOnly("io.papermc.paper:paper-api:1.21.4-R0.1-SNAPSHOT")` from `https://repo.papermc.io/repository/maven-public/`
  - Citizens API from `https://maven.citizensnpcs.co/repo` (`net.citizensnpcs:citizens-main:<latest 2.0.x-SNAPSHOT>`, compileOnly, `exclude(group = "*", module = "*")`)
  - Output jar name: `R3ckxBot-<version>.jar`
- Also include a GitHub Actions workflow (`.github/workflows/build.yml`) that builds the jar and uploads it as an artifact.
- **Cross-version safety rules** (1.21.3 changed several enums into interfaces):
  - Look up attributes by key through `Registry.ATTRIBUTE`. Try `attack_speed` first, then fall back to `generic.attack_speed`. Do the same for `attack_damage`, `max_health` and `movement_speed`. Put this in a small `Attrs` helper.
  - Play sounds by string key, for example `world.playSound(loc, "minecraft:entity.player.attack.crit", 1f, 1f)`, not through the `Sound` enum.
  - Look up enchantments and potion effects by key through `Registry` where convenient.
  - Use only modern (1.20.5+) names for `Particle` and `PotionEffectType`.
- Use **Adventure / MiniMessage** (bundled with Paper) for all text. Don't use legacy `§` codes in code. Messages may use MiniMessage tags in `messages.yml`.
- Use the Bukkit scheduler. Run one global bot-tick task every tick, and keep all AI math cheap.

---

## 3. Bot runtime (Citizens)

- Create the NPC registry with `CitizensAPI.createInMemoryNPCRegistry("kittyffa")` so bots are never saved to Citizens' `saves.yml`.
- Spawn with `registry.createNPC(EntityType.PLAYER, name)`, then `npc.setProtected(false)` and `npc.spawn(location)`. Apply the skin with `SkinTrait` (player name or texture value/signature from the profile).
- Set equipment through the Citizens `Equipment` trait (`npc.getOrAddTrait(Equipment.class).set(Equipment.EquipmentSlot.X, item)`): helmet, chestplate, leggings, boots, main hand and off hand. Re-sync the trait whenever items change, for example after a totem pops.
- **Movement:**
  - Up close (within about 6 to 8 blocks, line of sight): control the bot with `entity.setVelocity` every tick. Sprint speed is about 0.28 blocks per tick and walk speed about 0.21. Jump with y = 0.42 when on the ground. Auto-jump one-block steps when the block ahead is solid and the block above it is passable.
  - Farther away or without line of sight: use `npc.getNavigator().setTarget(target, false)` with tuned `NavigatorParameters` (`speedModifier`, `range`, `updatePathRate`, `distanceMargin`, `useNewPathfinder(true)`, `straightLineTargetingDistance`).
  - **Never** override velocity while the bot is taking knockback. After being hit, skip horizontal control until the bot lands again, or for about 8 ticks, whichever comes first. In the air, allow only small air-strafe nudges, so knockback looks vanilla.
- **Looking:** use `npc.faceLocation(target.getEyeLocation())`, with configurable aim smoothing and jitter so the bot looks human.
- **Attacking:** use Paper's `LivingEntity#attack(Entity)` and `swingMainHand()` on the bot's Player entity, so vanilla handles knockback, sprint knockback, shield disabling with axes, sweeping, enchants and fire aspect.
  - Citizens NPCs may not tick their attack-strength meter correctly. To keep damage correct, record bot attacks and, in `EntityDamageByEntityEvent` (bot is the damager, cause `ENTITY_ATTACK`), set the base damage to the true full-charge value: the attack-damage attribute, plus Sharpness, plus the Strength effect, times 1.5 if the bot is critting.
  - The bot crits when it is falling (`!onGround`, velocity y < 0), not sprinting and not in water. Show crit particles and play the crit sound.
- **Reach:** configurable from **1.0 to 5.0** (0.1 steps). Measure from the bot's eye to the closest point on the target's `BoundingBox`. Attack only within reach, with a small random miss chance from a configurable "accuracy" value.
- **Attack timing:** respect weapon cooldown (20 / attack speed, in ticks). Add a configurable humanised CPS jitter.
- **Shield:** if the shield ability is on and the bot holds a shield in its off hand, raise it with `startUsingItem(EquipmentSlot.OFF_HAND)` when the target is close, facing it and about to swing, and lower it with `clearActiveItem()` to attack. Vanilla handles axe shield-disable. Check `hasCooldown(Material.SHIELD)`.
- **Smart totem system:**
  - Per-profile totem count (0 to 64, or infinite), swap delay in ticks, and an optional "panic" health threshold for swapping early.
  - On `EntityResurrectEvent` for a bot: if no totems are left, cancel the event so the bot dies. Otherwise decrement the count and clear the off hand in the trait. After the swap delay, put a new totem in the off hand.
  - After a pop, the bot plays a short recovery behaviour (backs off or strafes for a configurable number of ticks), then re-applies its profile potion effects, since totems clear effects.
- **Godmode:** the bot takes knockback and plays the hurt animation, but damage is set to 0.
- **Slow Falling** ability: give the bot infinite Slow Falling.
- **Potion effects:** strength, speed, fire resistance, regeneration, resistance, jump boost, absorption, haste and slowness, each with a level from 0 (off) to 5, applied with `PotionEffect.INFINITE_DURATION`.

### Movement modes
`STAND` (never moves or looks), `LOOK` (tracks the owner with its head only), `FOLLOW` (follows the owner but doesn't attack), `ATTACK` (full combat AI against its owner).

### Combat modes
Each mode is its own class implementing a `CombatBrain` interface (`tick(KittyBot bot, Player target)`):

1. **NORMAL:** sprint at the target, attack on cooldown, small random strafe.
2. **W_TAP:** releases sprint for 1 to 3 ticks right after each hit, then re-sprints, for extra knockback and combo control.
3. **CRIT_SPAM:** jumps on approach and times hits on the way down to always crit. Never sprint-hits.
4. **STRAFE:** circles the target, switching strafe direction every 10 to 30 ticks (random), and closes in to hit.
5. **CRYSTAL_PVP:** see section 4.
6. **NETHPOT:** see section 5.

---

## 4. CrystalPvP brain (configurable per profile)

- Places obsidian (if missing) and an End Crystal near the target's feet, then breaks it by calling `attack(crystal)` from the bot, so the explosion is credited to the bot. Crystals must actually be placed: real obsidian blocks and real `EnderCrystal` entities. Track every block and entity the bot creates and remove them on despawn.
- **Crystal spam** toggle, and place delay / break delay in ticks.
- **DTap:** place, break, place, break in a quick sequence, with a configurable chance.
- **Respawn anchor:** place an anchor, charge it with glowstone (set the charges in the block data) and detonate it with `world.createExplosion(loc, 5f, true, true, bot)`. Configurable cooldown, and the anchor block is tracked for cleanup.
- **Pearl reactions:** if the target is beyond the pearl distance (configurable), throw an ender pearl with `launchProjectile(EnderPearl.class)` after the pearl delay. On land, teleport the NPC with `npc.teleport(loc, TeleportCause.ENDER_PEARL)`.
- **Totem recovery:** after a pop, back off and re-totem before re-engaging.
- **Close-range / hole fighting:** if the target is in a 1×1 obsidian or bedrock hole, place crystals at head level, or use the anchor instead.
- **Obsidian defense:** at low HP, place obsidian between the bot and the target for cover.
- **Reaction & opening weights:** a weighted random pick between openings (crystal rush, anchor open, pearl in, melee open), plus a "reaction time" in ticks.
- Respect a config world blacklist and never break pre-existing blocks: only replace air.

## 5. Nethpot brain (configurable per profile)

- Uses a netherite sword. Strafes and crits like CRIT_SPAM.
- Below the configurable HP threshold, **disengages** (sprints away for N ticks), turns around (turn delay), looks down (pitch 90) and throws Instant Health II splash potions at its feet with `launchProjectile(ThrownPotion.class)` with the item set. Settings: **pot count**, **pot delay**, and a **double-pot** toggle that throws 2 in quick succession.
- **Gapple timing:** below a second threshold, "eats" a golden apple over 32 ticks (hold it, play the eat sound and particles), then gives Absorption II and Regeneration II like vanilla. The bot can't attack while eating.
- After healing, re-engages after a recovery window.
- Totem delay setting, same as section 3.
- Pot and gapple counts are restocked when the bot is reset.

---

## 6. Profiles, presets and sharing

- A `BotProfile` holds:
  - name, skin, armor (4 ItemStacks incl. trims and enchants), main hand, off hand
  - reach, accuracy, CPS jitter, aim smoothing
  - combat mode, movement mode, speed multiplier
  - totem count and delay
  - potion levels map
  - abilities (shield, slow falling, godmode, auto-respawn)
  - `CrystalSettings` and `NethpotSettings` sub-objects
- **Multiple profiles per player** (limit by permission `kittyffa.profiles.<n>`, default 3). Switch instantly: switching respawns the live bot with the new profile in place.
- **Sharing:** `/bot share` makes an 8-character share code (unambiguous alphabet, no 0/O/1/I) for the active profile. `/bot import <code>` copies it as a new profile. Codes are stored in the DB and expire after a configurable number of days.
- **Admin presets:** admins save any profile as a named preset (for ranked practice, events or standard kits). All players can load presets from the Presets GUI. Ship 4 default presets: "Sword Duel", "Axe & Shield", "Crystal", "Nethpot".
- **Storage:** SQLite by default (`plugins/R3ckxBot/kittyffa.db`, using the SQLite driver bundled with Paper) or MySQL (config, also using the bundled driver). Use a tiny connection wrapper and run every DB call off the main thread (`CompletableFuture` and async scheduler), with results applied back on the main thread.
  - Tables: `kittyffa_profiles(id, owner_uuid, name, active, data TEXT)`, `kittyffa_presets(name, data TEXT, created_by)`, `kittyffa_shares(code, data TEXT, created_at)`.
  - Serialise profiles with `YamlConfiguration.saveToString()`, since ItemStacks serialise natively.

---

## 7. Commands (`/bot`, aliases `/kittyffa`, `/kbot`)

Every subcommand needs full tab completion and permission checks:

- `/bot` (no args): opens the Main Editor GUI.
- `/bot spawn | despawn | tp | reset | edit`
- `/bot mode <combat mode>`, `/bot move <movement mode>`, `/bot reach <1.0-5.0>`
- `/bot profile list | create <name> | switch <name> | delete <name> | rename <old> <new>`
- `/bot share`, `/bot import <code>`
- `/bot preset list | load <name>`, and for admins `/bot preset save <name> | delete <name>`
- Admin: `/bot admin list` (opens the Admin GUI), `/bot admin remove <player>`, `/bot admin edit <player>` (Admin Edit Mode: opens that player's editor and saves into their profile), `/bot admin removeall`, `/bot reload`

Permissions: `kittyffa.use` (default true), `kittyffa.mode.<mode>`, `kittyffa.crystal`, `kittyffa.nethpot`, `kittyffa.godmode`, `kittyffa.profiles.<n>`, `kittyffa.admin` (op).

---

## 8. Fully custom GUIs

Build a small menu framework:

- An abstract `Menu` that implements `InventoryHolder`, with button-click lambdas.
- One `MenuListener` that cancels all clicks and drags in KittyFFA menus, including shift-click and number-key clicks.
- Paginated menu support.
- A chat-input prompt utility (`AsyncChatEvent`, with a "type `cancel`" option and a timeout) for names, codes and numbers.

**Visual style** (original, not copied from any other plugin):

- Theme: "Kitty" pink/purple. MiniMessage gradient titles such as `<gradient:#ff7eb9:#a46bff>✦ KittyFFA ✦</gradient>`.
- Pink and magenta stained-glass borders, with a filler pane with a blank name.
- Each button has a bold coloured name, short grey lore, a `▸ Left-click` / `▸ Right-click` hint line, and current values shown as `● ON` / `○ OFF` in green or red.
- Number settings: left-click +, right-click −, shift-click ±10×. Show a progress bar in the lore, e.g. `▰▰▰▰▱▱▱▱ 3.0`.
- Every sub-menu has a back button (arrow) in the bottom-left and a close button (barrier) in the bottom-centre.
- Click sounds by string key, and an error sound on denied actions.

**Menus:**

1. **Main Editor (54 slots):** armor, weapon, enchants, reach, combat mode, movement mode, totems, potions, abilities, profile management, Crystal settings, Nethpot settings, spawn/despawn/teleport/reset quick buttons, and a bot skin head showing the active profile name.
2. **Armor menu:** per piece, cycle the material (leather → chainmail → iron → gold → diamond → netherite → none), plus trim pattern and trim material pickers (`ArmorMeta.setTrim`, from the registry).
3. **Weapon menu:** swords and axes (all tiers), mace, and off hand (totem, shield, none).
4. **Enchant menu:** pick an item slot, then toggle and set levels for sharpness, protection (each piece), unbreaking, fire aspect, knockback, sweeping edge, thorns and density/breach (mace). Cap at vanilla max, or at the config max for admins.
5. **Combat Mode menu:** one icon per mode, locked icon and lore if missing permission.
6. **Potions menu**, **Abilities menu**, **Totem menu** (count, delay, panic HP).
7. **CrystalPvP settings menu** and **Nethpot settings menu**: every field from sections 4 and 5.
8. **Profiles menu:** list profiles (active one glowing). Left-click switches, right-click renames, shift-right-click deletes (with a confirm menu). Also create, share and import buttons.
9. **Presets menu** (paginated). Admins see save/delete buttons.
10. **Admin Bots menu** (paginated): every live bot as its skin head, with owner, mode, HP and world in the lore. Left-click teleports to it, right-click removes it, shift-click opens Admin Edit Mode.

**In-combat HUD:** an action bar shown to the owner: `❤ Bot 14.5 ⛨ Totems 2 ⚔ Combo 3 ⌖ 2.87m` (the last value is the distance to the bot), plus an optional BossBar of the bot's health with a pink colour. Both can be toggled in config.

---

## 9. Bot lifecycle and cleanup

- One bot per player (admins: configurable). The nametag shows `<pink>KittyFFAPVPBot</pink>` or a configurable format with the owner's name.
- **World blacklist** (`config.yml`): refuse spawning there, and despawn if the owner enters one.
- **Auto cleanup**, each toggleable: owner quits, owner dies, owner teleports across worlds or more than X blocks, distance from owner > `max-distance`, idle for N minutes. Clean up on plugin disable too: despawn all NPCs and remove tracked crystals, obsidian and anchors.
- **Bot death:** message the owner (win), show stats (hits landed/taken, longest combo, totems popped, fight time), and auto-respawn after 3 seconds if enabled.
- **Owner death:** the bot wins a message, and the bot resets (full heal, restock totems/pots/gapples).
- Per-player fight stats stay in memory for the session.

---

## 10. Config files (fully commented)

- **`config.yml`:** storage (sqlite/mysql + credentials), world-blacklist, cleanup toggles and values, max-bots-per-player, default profile values, reach min/max, AI tuning (tick-rate divisor for far bots, pathfinding update rate, aim smoothing defaults), HUD toggles, share-code expiry, hide-plugin toggle (default true).
- **`messages.yml`:** every message in MiniMessage with a `<prefix>` placeholder (default `<gradient:#ff7eb9:#a46bff><b>KittyFFA</b></gradient> <dark_gray>»</dark_gray>`).
- **`presets-default.yml`:** the 4 starter presets, imported into the DB on first run.
- `/bot reload` reloads all of the above without restarting.

---

## 11. Quality bar

- Clean architecture:
  - `KittyFFA` (bootstrap)
  - `bot/` (`KittyBot`, `BotManager`, `BotProfile`, settings classes, enums)
  - `bot/brain/` (one class per combat mode)
  - `storage/`, `gui/` (framework + `menus/`), `command/`, `listener/` (`BotListener`, `CleanupListener`, `HidePluginListener`), `util/` (`Attrs`, `ItemBuilder`, `Text`, `ChatPrompt`)
- Null-safe and exception-safe. A bot error must never crash the tick loop: catch errors, log them and despawn that bot.
- Efficient: avoid per-tick allocations where easy, and skip AI for bots whose owner is offline or in another world.
- Include a `README.md` covering features, install (Paper 1.21.x + Citizens), build steps (`gradle build`), commands, permissions and config reference. The README may say "R3ckxBot" only as the jar/plugin name for server owners. All player-facing text says KittyFFA.

**Output:** print the complete project file-by-file (path heading, then the full file contents), starting with `build.gradle.kts`, `settings.gradle.kts`, `plugin.yml`, and the configs. Then print every Java class. Make sure the project compiles with no missing imports or methods.
