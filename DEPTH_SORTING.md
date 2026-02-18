# Y-Sort Layering and Object Depth Sorting in Stardew Valley

## Overview

Stardew Valley uses a **Y-sort depth sorting system** to create the illusion of depth in its 2D world. This ensures that objects and characters render in the correct order relative to their position in the game world - objects that are "further back" (higher Y-coordinates) appear behind objects that are "closer to the camera" (lower Y-coordinates).

## Core Concept

The fundamental principle is simple: **objects with higher Y-positions on the screen should be drawn first** (appear behind), while **objects with lower Y-positions should be drawn later** (appear in front). This creates the isometric-style depth effect seen in top-down 2D games.

## The layerDepth Formula

### Basic Formula

The core Y-sort formula used throughout Stardew Valley is:

```csharp
layerDepth = (float)standingY / 10000f
```

This converts a Y-position (in pixels) into a normalized depth value between 0.0 and 1.0. The division by 10,000 ensures that the depth value stays within the valid range for sprite rendering.

### Extended Formula for Objects

For objects and temporary animated sprites, a more sophisticated formula is used:

```csharp
layerDepth = ((float)tileLocationY + 1.1f) * 64f / 10000f
```

Where:
- `tileLocationY` is the tile Y-coordinate (world position / 64)
- `1.1f` is an offset to ensure proper layering relative to the tile
- `64f` converts tile coordinates back to pixel coordinates
- `10000f` normalizes to the 0.0-1.0 range

This formula appears in `GameLocation.cs` when adding temporary animated sprites like torches:

```csharp
// From GameLocation.cs, line 3785
layerDepth = ((float)tileLocationY + 1.1f) * 64f / 10000f
```

## Key Components

### 1. Character.getStandingY()

The `getStandingY()` method in the `Character` class calculates the Y-position at the center of a character's bounding box:

```csharp
// From Character.cs, line 1193
public int getStandingY()
{
    Microsoft.Xna.Framework.Rectangle box = GetBoundingBox();
    return box.Y + box.Height / 2;
}
```

This standing position is used as the basis for depth calculations. It represents where the character is "standing" in the world, which determines their render order.

### 2. Farmer.getDrawLayer()

The `Farmer` class (player character) has a more sophisticated `getDrawLayer()` method that handles special cases:

```csharp
// From Farmer.cs, line 6032
public float getDrawLayer()
{
    if (onBridge.Value)
    {
        return (float)getStandingY() / 10000f + drawLayerDisambiguator + 0.0256f;
    }
    if (IsSitting() && mapChairSitPosition.Value.X != -1f && mapChairSitPosition.Value.Y != -1f)
    {
        Vector2 sit_position = mapChairSitPosition.Value;
        return (sit_position.Y + 1f) * 64f / 10000f;
    }
    return (float)getStandingY() / 10000f + drawLayerDisambiguator;
}
```

Key features:
- **Bridge offset**: When on a bridge, adds `0.0256f` to render above water/ground
- **Chair positioning**: Uses the chair's tile position when sitting
- **Draw layer disambiguator**: A small offset to prevent z-fighting when multiple farmers occupy similar positions

### 3. Draw Layer Disambiguator

To prevent "z-fighting" (flickering) when multiple farmers stand very close to each other, the game uses a disambiguation system:

```csharp
// From GameLocation.cs, lines 12432-12449
farmer2.drawLayerDisambiguator = 0f;
// ...
if (!other_farmer.IsSitting() && Math.Abs(farmer.getDrawLayer() - other_farmer.getDrawLayer()) < disambiguator_amount 
    && Math.Abs(farmer.position.X - other_farmer.position.X) < 64f)
{
    other_farmer.drawLayerDisambiguator += farmer.getDrawLayer() - disambiguator_amount - other_farmer.getDrawLayer();
}
```

This system adds tiny offsets to farmers' draw layers when they're very close together, ensuring stable, consistent rendering order.

## Rendering Pipeline

### GameLocation Rendering Layers

The `GameLocation` class orchestrates rendering in specific layers:

1. **Background layers** - `drawBackground()`
2. **Floor decorations** - `drawFloorDecorations()`
3. **Water and water tiles** - `drawWater()`, `drawWaterTile()`
4. **Characters** - `drawCharacters()` (NPCs, monsters)
5. **Farmers** - `drawFarmers()` (player characters)
6. **Above front layer** - `drawAboveFrontLayer()` (trees, buildings)
7. **Above always front layer** - `drawAboveAlwaysFrontLayer()` (UI elements, effects)
8. **Light glows** - `drawLightGlows()`

Each layer uses depth values to sort elements within that layer.

### Character Drawing

Characters (NPCs, monsters, player) all use their `getStandingY()` position to calculate depth:

```csharp
// Typical depth usage in draw calls
b.Draw(texture, position, sourceRect, color, 0f, origin, scale, effects, 
       (float)getStandingY() / 10000f);  // layerDepth parameter
```

## Fine-Grained Depth Control

### FarmerRenderer Micro-Offsets

The `FarmerRenderer` class adds extremely small offsets to render different parts of the player sprite in the correct order:

```csharp
// From FarmerRenderer.cs, line 792
int sort_direction = (!Game1.isUsingBackToFrontSorting) ? 1 : (-1);
// Hair is rendered with a tiny offset from the base layer
b.Draw(hair_texture, position, hairstyleSourceRect, who.hairstyleColor, 0f, Vector2.Zero, scale, 
       flip ? SpriteEffects.FlipHorizontally : SpriteEffects.None, 
       layerDepth + 1.1E-07f * (float)sort_direction);  // Micro-offset for hair
```

These micro-offsets (`1.1E-07f` = 0.00000011) ensure that:
- Hair renders in front of the base body sprite
- Accessories render in the correct order
- Held items appear in front of or behind the body depending on facing direction

## Back-to-Front vs Front-to-Back Sorting

### The isUsingBackToFrontSorting Flag

Stardew Valley supports two rendering modes:

```csharp
// From Game1.cs, line 219
public static bool isUsingBackToFrontSorting = false;
```

When `isUsingBackToFrontSorting` is true:
- Objects are drawn from **back to front** (far to near)
- Micro-offsets are **negative** to maintain correct ordering
- Used in specific mini-games like the mine cart game

When false (default):
- Objects are drawn in **standard order**
- Micro-offsets are **positive**

The mine cart mini-game toggles this flag:

```csharp
// From Minigames/MineCart.cs
Game1.isUsingBackToFrontSorting = true;  // On start
Game1.isUsingBackToFrontSorting = false; // On end
```

## Practical Examples

### Example 1: Torch Placement

When placing a torch in the game world:

```csharp
// From GameLocation.cs, line 3785
temporarySprites.Add(new TemporaryAnimatedSprite(
    // ... texture and animation parameters ...
    layerDepth = ((float)tileLocationY + 1.1f) * 64f / 10000f
));
```

If the torch is placed at tile Y=10:
- `layerDepth = (10 + 1.1) * 64 / 10000 = 0.07104`

### Example 2: Character Emote

When a character displays an emote:

```csharp
// From Character.cs, line 1139
b.Draw(Game1.emoteSpriteSheet, emotePosition, sourceRect, Color.White * alpha, 
       0f, Vector2.Zero, 4f, SpriteEffects.None, 
       (float)getStandingY() / 10000f);
```

If the character is standing at pixel Y=640:
- `layerDepth = 640 / 10000 = 0.064`

The emote will be drawn at the same depth as the character, ensuring it moves with them in the rendering order.

### Example 3: Player Shadow

Player shadows are rendered slightly behind the player:

```csharp
// From Game1.cs, line 17639
float draw_layer = Math.Max(0.0001f, f4.getDrawLayer() + 0.00011f) - 0.0001f;
```

This calculates a depth slightly less than the player's draw layer, causing the shadow to render just behind the player.

## Summary

Stardew Valley's depth sorting system is elegant and effective:

1. **Y-position determines depth**: Objects further "back" (higher Y) have smaller depth values
2. **Normalized depth values**: All depths are in the 0.0-1.0 range via division by 10,000
3. **Layered rendering**: The game renders in distinct layers (background, characters, foreground)
4. **Micro-offsets prevent z-fighting**: Tiny depth adjustments (0.00000011) ensure stable ordering of overlapping sprites
5. **Special case handling**: Bridges, sitting, and farmer collisions all have custom depth logic
6. **Bidirectional support**: Can render back-to-front or front-to-back as needed

This system creates the classic 2D RPG depth effect where characters can walk "behind" objects when they're higher on screen, and "in front of" them when they're lower - a fundamental aspect of Stardew Valley's visual presentation.

## Key Files

- **GameLocation.cs** - Main rendering orchestration and depth calculations
- **Character.cs** - Base character positioning and depth methods
- **Farmer.cs** - Player-specific depth handling including bridges and sitting
- **FarmerRenderer.cs** - Fine-grained depth control for player sprite parts
- **Game1.cs** - Global rendering flags and shadow rendering
- **TemporaryAnimatedSprite.cs** - Sprite objects with configurable layer depth
- **TilePositionComparer.cs** - Tile position equality comparer for depth-based collections

## Additional Notes

### Why 10,000?

The division by 10,000 is chosen because:
- Most game areas are smaller than 10,000 pixels in height
- It provides 4 decimal places of precision (0.0001 units per pixel)
- It keeps all depth values safely within the 0.0-1.0 range required by the rendering system
- It allows for micro-offsets (like `1.1E-07f`) to fine-tune rendering order without overflow

### Z-Fighting Prevention

Without the disambiguator system, two farmers standing at exactly the same Y-position would have identical depth values, causing flickering as the renderer arbitrarily chooses which to draw first. The disambiguator adds a tiny consistent offset based on player ID or position, ensuring stable rendering.

### Performance Considerations

The depth sorting system is computationally efficient:
- Simple arithmetic calculations (division, addition)
- No complex sorting algorithms needed during render
- The rendering API (SpriteBatch) handles depth sorting internally
- Micro-offsets are precomputed constants
