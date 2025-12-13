# BYOND Performance Review - Critical Optimization Opportunities

## Executive Summary

After thorough analysis of the codebase, I've identified several critical performance bottlenecks centered around **hard deletions**, **`get_limb_icon`**, and **general mob processing**. This document provides detailed findings and prioritized optimization recommendations.

---

## 1. Hard Deletions (Critical Priority)

### Location: `code/controllers/subsystem/garbage.dm`

### Current Behavior
The garbage collector uses a two-phase approach:
1. **GC_QUEUE_CHECK** (2 minute timeout) - Attempts soft deletion via BYOND's garbage collector
2. **GC_QUEUE_HARDDELETE** (10 second timeout) - Falls back to `del()` if soft delete fails

### Key Issues

#### 1.1 Reference Retention Causing Hard Deletes
When `Destroy()` is called but objects still have references, they enter the hard delete queue. The profiling data shows this is a major CPU drain because:
- `del()` is extremely expensive in BYOND - it walks the entire reference tree
- Each hard delete can cause tick overruns (seen in lines 257-259 logging >1 second deletes)

#### 1.2 Common Reference Leak Patterns Found

**Status Effects not cleaning up:**
```dm
// code/datums/status_effects/status_effect.dm
// Status effects may retain references to their owners
```

**Bodypart references:**
```dm
// code/modules/surgery/bodyparts/_bodyparts.dm:158-166
/obj/item/bodypart/Destroy()
    if(owner)
        owner.bodyparts -= src
        owner = null
    if(bandage)
        QDEL_NULL(bandage)
    for(var/datum/wound/wound as anything in wounds)
        qdel(wound)  // Good - but wounds may reference the bodypart back
    return ..()
```

**Component signals not unregistering:**
```dm
// code/datums/datum.dm:104-134
// Signal cleanup is done but comp_lookup may retain references
```

### Optimization Recommendations

#### High Priority
1. **Add QDEL_HINT_IWILLGC** to frequently deleted types that properly clean themselves:
```dm
/obj/item/bodypart/Destroy()
    // ... cleanup code ...
    return QDEL_HINT_IWILLGC  // Skip queue if we're confident
```

2. **Profile and fix the top hard-delete offenders** by enabling `TESTING` compile flag and checking the qdel log at shutdown (line 86-106 in garbage.dm)

3. **Use weak references** for non-essential backreferences:
```dm
var/datum/weakref/owner_ref  // Instead of var/mob/owner
```

---

## 2. get_limb_icon (Critical Priority)

### Location: `code/modules/surgery/bodyparts/_bodyparts.dm:617-733`

### Current Behavior
Called when updating bodypart visual representation. Creates multiple `mutable_appearance` objects each invocation.

### Key Issues

#### 2.1 No Caching of Appearances
Every call to `get_limb_icon()` creates new mutable_appearances:

```dm
/obj/item/bodypart/proc/get_limb_icon(dropped, hideaux = FALSE)
    // Line 636-639: Creates new image objects every call
    var/image/limb = image(layer = -BODYPARTS_LAYER, dir = image_dir)
    var/image/aux
    . = list()
    . += limb
```

#### 2.2 Called Excessively
`get_limb_icon` is called from `update_body_parts()` which is triggered by:
- `update_hair()` 
- `update_inv_wear_mask()`
- `update_inv_shirt()` 
- `update_inv_armor()`
- And many more cascading updates

#### 2.3 Organ Iteration Each Call
```dm
// Lines 718-724
if(!skeletonized && draw_organ_features)
    for(var/obj/item/organ/organ as anything in get_organs())
        if(!organ.is_visible())
            continue
        var/mutable_appearance/organ_appearance = organ.get_bodypart_overlay(src)
```

### Optimization Recommendations

#### High Priority
1. **Implement limb appearance caching** using the existing `limb_icon_cache`:
```dm
/obj/item/bodypart/proc/get_limb_icon(dropped, hideaux = FALSE)
    var/cache_key = generate_limb_cache_key(dropped, hideaux)
    if(limb_appearance_cache[cache_key])
        return limb_appearance_cache[cache_key].Copy()
    
    // ... existing generation code ...
    
    limb_appearance_cache[cache_key] = .
    return .
```

2. **Cache organ overlays** at the bodypart level:
```dm
var/list/cached_organ_overlays
var/organ_overlay_dirty = TRUE

/obj/item/bodypart/proc/get_organ_overlays()
    if(!organ_overlay_dirty && cached_organ_overlays)
        return cached_organ_overlays
    // Generate and cache
    organ_overlay_dirty = FALSE
```

3. **Batch appearance updates** - defer `get_limb_icon` calls to a subsystem like `SSdamoverlays` does

---

## 3. General Mob Processing (High Priority)

### Locations:
- `code/controllers/subsystem/mobs.dm`
- `code/modules/mob/living/life.dm`
- `code/modules/mob/living/carbon/life.dm`
- `code/modules/mob/living/carbon/human/life.dm`

### Current Behavior
Every living mob has `Life()` called each tick (configurable wait, currently 1 tick). The human Life() chain calls:
- `handle_organs()` - iterates internal_organs
- `handle_wounds()` - iterates all wounds
- `handle_embedded_objects()` - nested iteration over bodyparts and embedded objects
- `handle_bodyparts()` - iterates bodyparts
- `update_stress()` (every 3rd tick)
- Various other handlers

### Key Issues

#### 3.1 Excessive Bodypart Iteration
Found **85 instances** of `for(var/... in bodyparts)` and **30 instances** of `for(var/... in internal_organs)` across the codebase.

Each Life() tick for a human can iterate bodyparts 3-5+ times:
```dm
// carbon/life.dm:186-195
/mob/living/carbon/proc/handle_bodyparts()
    for(var/I in bodyparts)  // Iteration #1
        var/obj/item/bodypart/BP = I
        if(BP.needs_processing)
            . |= BP.on_life(stam_regen)

// carbon/life.dm:207-220 
/mob/living/carbon/handle_embedded_objects()
    for(var/obj/item/bodypart/bodypart as anything in bodyparts)  // Iteration #2
        for(var/obj/item/embedded as anything in bodypart.embedded_objects)  // Nested!
```

#### 3.2 List Copy Operations
The `for(var/X in list)` pattern in BYOND creates a copy of the list. With 6 bodyparts per human and many humans, this adds up.

#### 3.3 Cascading Icon Updates
```dm
// human/life.dm:95
name = get_visible_name()  // Can trigger icon updates

// From update_inv_* procs triggering each other
update_inv_armor() -> update_hair() -> update_body_parts() -> update_damage_overlays()
```

### Optimization Recommendations

#### High Priority
1. **Consolidate bodypart iterations** into a single pass:
```dm
/mob/living/carbon/proc/process_bodyparts_unified(stam_regen)
    for(var/obj/item/bodypart/BP as anything in bodyparts)
        // Handle all bodypart processing in one loop
        if(BP.needs_processing)
            . |= BP.on_life(stam_regen)
        // Handle embedded objects in same loop
        for(var/obj/item/embedded as anything in BP.embedded_objects)
            handle_single_embedded(embedded, BP)
```

2. **Use `as anything` type hints** to skip type checking overhead:
```dm
// Instead of:
for(var/I in bodyparts)
    var/obj/item/bodypart/BP = I

// Use:
for(var/obj/item/bodypart/BP as anything in bodyparts)
```

3. **Flag-based processing** - only process what's needed:
```dm
var/bodypart_processing_flags = NONE

/mob/living/carbon/proc/handle_bodyparts()
    if(!(bodypart_processing_flags & BP_PROCESS_DAMAGE))
        return
```

---

## 4. Icon Update System (High Priority)

### Location: `code/modules/mob/living/carbon/human/update_icons.dm`

### Key Issues

#### 4.1 Massive mutable_appearance Creation
`update_damage_overlays_real()` (lines 106-274) creates **up to 18 mutable_appearances per bodypart**:
- 3 damage layers × 2 damage types × potentially multiple wound overlays

```dm
// Lines 155-167 - This pattern repeats many times
if(BP.brutestate)
    var/mutable_appearance/damage_overlay = mutable_appearance(limb_icon, "[BP.body_zone]_[BP.brutestate]0", -DAMAGE_LAYER)
    damage_overlays += damage_overlay
    var/mutable_appearance/legdam_overlay = mutable_appearance(limb_icon, "legdam_[BP.body_zone]_[BP.brutestate]0", -LEG_DAMAGE_LAYER)
    legdam_overlays += legdam_overlay
    var/mutable_appearance/armdam_overlay = mutable_appearance(limb_icon, "armdam_[BP.body_zone]_[BP.brutestate]0", -ARM_DAMAGE_LAYER)
    armdam_overlays += armdam_overlay
```

#### 4.2 Cascading Update Calls
Many update_inv_* procs call each other:
```dm
update_inv_neck() -> update_hair()  // line 404
update_inv_head() -> update_hair()  // line 724
update_inv_shirt() -> update_hair() + update_inv_wrists()  // lines 1279-1280
update_inv_armor() -> update_hair() + update_inv_shirt()  // lines 1348-1349
```

### Optimization Recommendations

#### High Priority
1. **Use static appearance caches** for damage overlays:
```dm
var/static/list/damage_overlay_cache = list()

/proc/get_cached_damage_overlay(icon, state, layer)
    var/key = "[icon]-[state]-[layer]"
    if(!damage_overlay_cache[key])
        damage_overlay_cache[key] = mutable_appearance(icon, state, -layer)
    return damage_overlay_cache[key]
```

2. **Defer icon updates** to batch processing:
```dm
var/icon_update_queued = FALSE

/mob/living/carbon/human/proc/queue_icon_update(update_type)
    pending_icon_updates |= update_type
    if(!icon_update_queued)
        icon_update_queued = TRUE
        addtimer(CALLBACK(src, PROC_REF(process_queued_updates)), 0)
```

3. **Break update cascades** with dirty flags:
```dm
var/hair_dirty = FALSE

/mob/living/carbon/human/update_hair()
    if(!hair_dirty)
        return
    hair_dirty = FALSE
    // actual update code
```

---

## 5. NPC/Simple Animal Processing (Medium Priority)

### Locations:
- `code/controllers/subsystem/npcpool.dm`
- `code/modules/mob/living/simple_animal/simple_animal.dm`

### Current Behavior
Active NPCs are processed every tick with:
- `handle_automated_action()`
- `handle_automated_movement()`
- `handle_automated_speech()`

### Key Issues

#### 5.1 Three Separate Handler Calls
```dm
// npcpool.dm:26-32
if(SA.stat != DEAD)
    SA.handle_automated_action()
if(SA.stat != DEAD)
    SA.handle_automated_movement()
if(SA.stat != DEAD)
    SA.handle_automated_speech()
```

#### 5.2 Grid/Cell Recalculation on Every Move
```dm
// simple_animal.dm:1021-1038
/mob/living/simple_animal/Moved()
    . = ..()
    update_grid()  // Called EVERY move

/mob/living/simple_animal/proc/update_grid()
    var/list/cell_collections = our_cells.recalculate_cells(our_turf)
    // Signal registration/unregistration
```

### Optimization Recommendations

1. **Consolidate NPC handlers** into a single method
2. **Rate-limit grid updates** - don't recalculate every move:
```dm
var/next_grid_update = 0

/mob/living/simple_animal/proc/update_grid()
    if(world.time < next_grid_update)
        return
    next_grid_update = world.time + 10  // Every second
    // actual update
```

---

## 6. Additional Performance Patterns Found

### 6.1 Unnecessary Type Checking
```dm
// Bad - checks type then casts
for(var/X in bodyparts)
    var/obj/item/bodypart/BP = X
    
// Better - cast with as anything
for(var/obj/item/bodypart/BP as anything in bodyparts)
```

### 6.2 Repeated get_turf() Calls
```dm
// Seen in multiple places - cache the turf
var/turf/T = get_turf(src)
// ... use T multiple times instead of calling get_turf() again
```

### 6.3 String Concatenation in Loops
```dm
// Bad - creates new strings each iteration
for(var/thing in list)
    output += "[thing], "
    
// Better - use list join
output = list_to_concat.Join(", ")
```

---

## Priority Summary

| Issue | Impact | Effort | Priority |
|-------|--------|--------|----------|
| Hard Deletions - Fix top offenders | Critical | Medium | 1 |
| get_limb_icon caching | Critical | Medium | 2 |
| Consolidate bodypart iterations | High | Low | 3 |
| Damage overlay caching | High | Low | 4 |
| Break update cascades | High | Medium | 5 |
| NPC grid update throttling | Medium | Low | 6 |

---

## Implementation Notes

### Testing Performance Changes
1. Enable `TESTING` define to get qdel statistics
2. Use BYOND's built-in profiler: `?debug` → Profiler
3. Monitor `SSgarbage` stat panel for GC ratio (goal: >95% soft deletes)
4. Check `SSoverlays` queue length for overlay system health

### Safe Refactoring Patterns
1. Always test appearance changes with multiple species/genders
2. Verify dirty flag systems trigger updates correctly
3. Cache invalidation is critical - when in doubt, invalidate

### BYOND-Specific Considerations
- Lists use red-black trees internally - O(log n) access
- `for(var/x in list)` copies the list - use `as anything` when safe
- `istype()` is relatively expensive - avoid in hot paths
- Appearances are reference-counted - reuse when possible

---

## 7. Concrete Optimization Examples

### 7.1 Optimized update_body_parts (carbon/update_icons.dm)

**Current code (lines 399-427):**
```dm
/mob/living/carbon/proc/update_body_parts()
    var/oldkey = icon_render_key
    icon_render_key = generate_icon_render_key()
    if(oldkey == icon_render_key)
        return

    remove_overlay(BODYPARTS_LAYER)

    for(var/X in bodyparts)  // First iteration
        var/obj/item/bodypart/BP = X
        BP.update_limb()

    if(limb_icon_cache[icon_render_key])
        load_limb_from_cache()
        return

    var/list/new_limbs = list()
    for(var/X in bodyparts)  // Second iteration - redundant!
        var/obj/item/bodypart/BP = X
        new_limbs += BP.get_limb_icon()
```

**Optimized version:**
```dm
/mob/living/carbon/proc/update_body_parts()
    var/oldkey = icon_render_key
    icon_render_key = generate_icon_render_key()
    if(oldkey == icon_render_key)
        return

    remove_overlay(BODYPARTS_LAYER)

    // Check cache FIRST before updating limbs
    if(limb_icon_cache[icon_render_key])
        load_limb_from_cache()
        return

    // Single iteration for both update_limb and get_limb_icon
    var/list/new_limbs = list()
    for(var/obj/item/bodypart/BP as anything in bodyparts)
        BP.update_limb()
        new_limbs += BP.get_limb_icon()
    
    if(length(new_limbs))
        overlays_standing[BODYPARTS_LAYER] = new_limbs
        limb_icon_cache[icon_render_key] = new_limbs

    apply_overlay(BODYPARTS_LAYER)
    update_damage_overlays()
```

### 7.2 Optimized handle_bodyparts (carbon/life.dm)

**Current code (lines 186-195):**
```dm
/mob/living/carbon/proc/handle_bodyparts()
    var/stam_regen = FALSE
    if(stam_regen_start_time <= world.time)
        stam_regen = TRUE
        if(stam_paralyzed)
            . |= BODYPART_LIFE_UPDATE_HEALTH
    for(var/I in bodyparts)
        var/obj/item/bodypart/BP = I
        if(BP.needs_processing)
            . |= BP.on_life(stam_regen)
```

**Optimized version:**
```dm
/mob/living/carbon/proc/handle_bodyparts()
    var/stam_regen = stam_regen_start_time <= world.time
    if(stam_regen && stam_paralyzed)
        . |= BODYPART_LIFE_UPDATE_HEALTH
    
    // Use 'as anything' to skip type checking - we know the list contents
    for(var/obj/item/bodypart/BP as anything in bodyparts)
        if(BP.needs_processing)
            . |= BP.on_life(stam_regen)
```

### 7.3 Cached Damage Overlay System

**Current problem:** `update_damage_overlays_real()` creates 3-18 new mutable_appearances per bodypart every call.

**Solution - Static cached appearances:**
```dm
// Add to a global file or __DEFINES
GLOBAL_LIST_EMPTY(damage_overlay_cache)

/proc/get_damage_overlay(icon, state, layer)
    var/key = "[icon]|[state]|[layer]"
    var/mutable_appearance/cached = GLOB.damage_overlay_cache[key]
    if(!cached)
        cached = mutable_appearance(icon, state, -layer)
        GLOB.damage_overlay_cache[key] = cached
    return cached

// Then in update_damage_overlays_real(), replace:
// var/mutable_appearance/damage_overlay = mutable_appearance(limb_icon, "[BP.body_zone]_[BP.brutestate]0", -DAMAGE_LAYER)
// With:
// var/mutable_appearance/damage_overlay = get_damage_overlay(limb_icon, "[BP.body_zone]_[BP.brutestate]0", DAMAGE_LAYER)
```

### 7.4 Batched Icon Update System

**Problem:** Update cascades cause redundant work (update_armor → update_hair → update_body_parts → update_damage_overlays)

**Solution:**
```dm
// Add to mob/living/carbon/human
var/pending_icon_updates = NONE
var/icon_update_scheduled = FALSE

#define ICON_UPDATE_HAIR      (1<<0)
#define ICON_UPDATE_BODY      (1<<1)
#define ICON_UPDATE_DAMAGE    (1<<2)
#define ICON_UPDATE_CLOTHING  (1<<3)

/mob/living/carbon/human/proc/schedule_icon_update(update_flags)
    pending_icon_updates |= update_flags
    if(!icon_update_scheduled)
        icon_update_scheduled = TRUE
        // Process at end of current tick
        addtimer(CALLBACK(src, PROC_REF(process_icon_updates)), 0, TIMER_UNIQUE)

/mob/living/carbon/human/proc/process_icon_updates()
    icon_update_scheduled = FALSE
    var/flags = pending_icon_updates
    pending_icon_updates = NONE
    
    // Process in dependency order
    if(flags & ICON_UPDATE_BODY)
        real_update_body()
    if(flags & ICON_UPDATE_HAIR)
        real_update_hair()
    if(flags & ICON_UPDATE_CLOTHING)
        real_update_clothing()
    if(flags & ICON_UPDATE_DAMAGE)
        real_update_damage_overlays()
```

### 7.5 Optimized Wound Iteration

**Current code scattered across multiple files:**
```dm
for(var/datum/wound/wound as anything in BP.wounds)
    if(isnull(wound) || isnull(wound.mob_overlay))
        continue
    // process wound
```

**Problem:** Null checks in hot path suggest data integrity issues.

**Solution:** Ensure wounds list is always clean:
```dm
// In bodypart, ensure wounds are never null
/obj/item/bodypart/proc/add_wound(datum/wound/W)
    if(!W)
        return
    wounds += W

/obj/item/bodypart/proc/remove_wound(datum/wound/W)
    wounds -= W
    // No need for null check in iteration anymore
```

### 7.6 Simple Animal Grid Update Throttling

**Current code (simple_animal.dm lines 1021-1038):**
```dm
/mob/living/simple_animal/Moved()
    . = ..()
    update_grid()  // Called every single move!
```

**Optimized version:**
```dm
var/next_grid_update_time = 0

/mob/living/simple_animal/Moved()
    . = ..()
    // Only update grid every 5 ticks (0.5 seconds)
    if(world.time >= next_grid_update_time)
        next_grid_update_time = world.time + 5
        update_grid()
```

---

## 8. Quick Wins (Minimal Effort, Good Impact)

1. **Replace `for(var/X in list)` with `for(var/type/X as anything in list)`** throughout hot paths
   - Files: carbon/life.dm, carbon/update_icons.dm, human/update_icons.dm
   - Impact: ~10-15% reduction in list iteration overhead

2. **Add `QDEL_HINT_IWILLGC` to simple datums** that clean up properly
   - Files: status_effects, wounds, components
   - Impact: Reduces GC queue processing

3. **Cache `get_turf(src)` results** in procs that call it multiple times
   - Common pattern: `var/turf/T = get_turf(src)` at start of proc

4. **Use `length()` proc instead of `.len`** for lists that might be null
   - `length(list)` is null-safe, `.len` will runtime on null

5. **Remove testing() calls** from production code
   - Search for: `testing("` - these are debug calls that should be compiled out

---

## 9. Monitoring and Validation

### Key Metrics to Track
1. **GC Ratio**: `SSgarbage` stat panel shows soft delete success rate (target: >95%)
2. **Overlay Queue**: `SSoverlays` queue length (should stay low, spikes indicate problems)
3. **Mob Processing Time**: Profile `SSmobs.fire()` duration
4. **Hard Delete Time**: Check qdel log at round end for expensive deletions

### Profiling Commands
```dm
// Enable in-game profiler
?debug  // Then click Profiler

// Check GC health
debug_gc()  // Custom verb if available

// Monitor specific subsystem
SSmobs.stat_entry()  // Returns processing stats
```

### Compile-Time Diagnostics
Enable these defines for debugging:
```dm
#define TESTING           // Enables qdel logging
#define REFERENCE_TRACKING // Helps find GC failures (expensive!)
```
