# Mixin Configuration Reference

## Overview
This document provides a quick reference for the Mixin configurations used in this mod.

## Mixin Configuration File
**Location**: `src/main/resources/mixins.etheriaperfix.json`

```json
{
  "required": true,
  "minVersion": "0.8",
  "package": "com.konek101.etheriaperfix.mixin",
  "compatibilityLevel": "JAVA_17",
  "refmap": "etheriaperfix.refmap.json",
  "mixins": [
    "ItemsLootModifierMixin",
    "EtheriaCapabilityMixin"
  ],
  "injectors": {
    "defaultRequire": 1
  }
}
```

## Mixin Classes

### 1. ItemsLootModifierMixin

**Target Class**: `it.mralxart.etheria.loot.ItemsLootModifier`

**Annotations**:
```java
@Pseudo  // Makes mixin optional if target class not found
@Mixin(targets = "it.mralxart.etheria.loot.ItemsLootModifier", remap = false)
```

**Target Method**:
```java
@Overwrite
protected ObjectArrayList<ItemStack> doApply(
    ObjectArrayList<ItemStack> generatedLoot, 
    LootContext context
)
```

**Key Features**:
- `@Pseudo`: Won't crash if Etheria mod is not installed
- `remap = false`: Prevents obfuscation remapping (mod classes use SRG names)
- `@Overwrite`: Completely replaces the target method

**Static Fields** (not present in original):
```java
private static final ConcurrentHashMap<String, Pattern> PATTERN_CACHE
private static volatile List<Item> cachedLootDataItems
private static volatile List<Item> cachedSpellScrollItems
private static final Random SHARED_RANDOM
```

### 2. EtheriaCapabilityMixin

**Target Class**: `it.mralxart.etheria.capability.EtheriaCapability`

**Annotations**:
```java
@Pseudo
@Mixin(targets = "it.mralxart.etheria.capability.EtheriaCapability", remap = false)
```

**Shadow Fields** (from original class):
```java
@Shadow private List<?> starsList;
@Shadow private List<?> skillsList;
```

**Shadow Methods** (from original class):
```java
@Shadow protected abstract CompoundTag serializeStar(Object star);
@Shadow protected abstract CompoundTag serializeSkill(Object skill);
@Shadow protected abstract String getStarId(Object star);
@Shadow protected abstract String getSkillId(Object skill);
```

**Target Methods**:
```java
@Overwrite
public ListTag serializeStarsList()

@Overwrite
public ListTag serializeSkillsList()
```

## Mixin Loading Process

### 1. Mod Initialization
```
ForgeModLoader loads mod
    ↓
Reads META-INF/mods.toml
    ↓
Finds MixinConfigs entry in JAR manifest
    ↓
Loads mixins.etheriaperfix.json
```

### 2. Mixin Application
```
MixinBootstrap initializes
    ↓
Loads mixin configuration
    ↓
Scans for target classes (Etheria mod)
    ↓
If @Pseudo and target not found → Skip mixin
    ↓
If target found → Apply transformations
    ↓
Generate refmap (etheriaperfix.refmap.json)
```

### 3. Runtime Behavior
```
Block break event occurs
    ↓
Loot table query → ItemsLootModifierMixin.doApply()
    ↓
Capability sync → EtheriaCapabilityMixin.serializeStarsList()
    ↓
Optimized code executes
```

## Build-Time Mixin Processing

### Mixin Annotation Processor
Configured in `build.gradle`:
```gradle
dependencies {
    annotationProcessor 'org.spongepowered:mixin:0.8.5:processor'
}

mixin {
    add sourceSets.main, 'etheriaperfix.refmap.json'
    config 'mixins.etheriaperfix.json'
}
```

### What the Processor Does:
1. **Validates** mixin classes against targets
2. **Generates** refmap for obfuscation compatibility
3. **Checks** @Shadow declarations match target class
4. **Verifies** method signatures are correct

### Refmap Generation
The refmap (`etheriaperfix.refmap.json`) maps:
- Development names → Production names
- Obfuscated names → Deobfuscated names
- Ensures mixins work in both dev and production environments

Example refmap entry:
```json
{
  "mixins": {
    "ItemsLootModifierMixin": {
      "doApply": "doApply(Lit/unimi/dsi/fastutil/objects/ObjectArrayList;Lnet/minecraft/world/level/storage/loot/LootContext;)Lit/unimi/dsi/fastutil/objects/ObjectArrayList;"
    }
  }
}
```

## JAR Manifest Configuration

Configured in `build.gradle`:
```gradle
jar {
    manifest {
        attributes([
            "MixinConfigs": "mixins.etheriaperfix.json"
        ])
    }
}
```

This tells Forge where to find the mixin configuration.

## Debugging Mixins

### Enable Mixin Debug Output
Add to JVM arguments:
```
-Dmixin.debug=true
-Dmixin.debug.export=true
```

### Export Transformed Classes
```
-Dmixin.debug.export.decompile=true
-Dmixin.debug.export.path=.mixin.out/
```

This exports the transformed class files for inspection.

### Common Issues

#### 1. Mixin Not Applying
**Symptom**: Original code still runs
**Check**:
- Is Etheria mod installed?
- Is mixin listed in mixins.etheriaperfix.json?
- Check logs for mixin errors

#### 2. ClassNotFoundException
**Symptom**: Crash at startup
**Cause**: @Pseudo missing and target class not found
**Solution**: Add `@Pseudo` annotation

#### 3. Method Not Found
**Symptom**: MixinError about missing method
**Cause**: Method signature doesn't match target
**Solution**: Verify method signature matches exactly

#### 4. Refmap Missing
**Symptom**: Mixin fails in production environment
**Cause**: Refmap not generated or not included in JAR
**Solution**: Verify mixin gradle plugin is configured

## Testing Mixin Application

### Verify in Development
```java
// Add debug logging in mixin
@Overwrite
protected ObjectArrayList<ItemStack> doApply(...) {
    System.out.println("MIXIN APPLIED: ItemsLootModifierMixin");
    // ... rest of code
}
```

### Verify at Runtime
Check logs for:
```
[INFO] Etheria Performance Fix initialized - applying performance optimizations
[DEBUG] Mixin: ItemsLootModifierMixin applied to it.mralxart.etheria.loot.ItemsLootModifier
[DEBUG] Mixin: EtheriaCapabilityMixin applied to it.mralxart.etheria.capability.EtheriaCapability
```

## Compatibility Notes

### Why `remap = false`?
Etheria is a mod, not Minecraft itself:
- Mod classes use SRG (Searge) names, not obfuscated names
- Remapping would try to deobfuscate names that aren't obfuscated
- `remap = false` treats class names as-is

### Why `targets = "..."` instead of class reference?
```java
// This would require Etheria to be a compile dependency:
@Mixin(ItemsLootModifier.class)

// This uses string target, making it optional:
@Mixin(targets = "it.mralxart.etheria.loot.ItemsLootModifier", remap = false)
```

Benefits:
- Mod compiles without Etheria on classpath
- Mixin applies at runtime when Etheria is present
- @Pseudo prevents crash if Etheria is missing

## Performance Monitoring

### Before/After Comparison
Use Spark profiler:
```
/spark profiler start
<perform actions>
/spark profiler stop

Compare:
- it.mralxart.etheria.loot.ItemsLootModifier.doApply
- it.mralxart.etheria.capability.EtheriaCapability.serializeStarsList
- it.mralxart.etheria.capability.EtheriaCapability.serializeSkillsList
```

Expected improvements documented in TECHNICAL.md.

## References

- **Mixin Wiki**: https://github.com/SpongePowered/Mixin/wiki
- **ForgeGradle Docs**: https://docs.minecraftforge.net/
- **Mixin Javadocs**: https://jenkins.liteloader.com/view/Other/job/Mixin/javadoc/
