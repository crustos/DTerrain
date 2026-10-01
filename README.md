# DTerrain
Destructible terrain in Unity

Simple **destructible terrain** in Unity based on bitmaps and Quadtree. Pretty efficent and works with Unity Colliders. Very reusable and customizable for your own needs. This beginner friendly tool will take your game to the next level. If you want to use it, clone this repository and see example scene.

Also provides additional functionalities as **painting on customizable layers**.

![Demo](dterrainexample_new.gif)

## FAQ
### How destructible terrain works?
I use **Ranges**: [min;max]
Then I make a list of ranges called **Column**.
I fill those ranges using image (if color.alpha>threshold I fill it and use range to remember it effectively).
Now I have a list of **Columns** that make a single **Chunk**. **Layer** is made of chunks and when any changes are made to the **Layer** - it changes the certain chunk (or chunks). 

There are many ways you can handle destruction: if I have shape (list of **Columns**) I can manually remove each pixel, or use **ranges** of this shape to delete effectively bigger areas of terrain.

Each **Chunk** has their sub-texture made from original Texture. I found out that changing a few smaller Textures is much, much faster than changing one big texture. 

Whenever a change is made (destruction) each **Chunk** recalculates sub-textures based on their **Columns** and recreates BoxColliders2D using Quadtree to fit new terrain.

### Would it work in realtime game?
*Yes. Example scene rarely goes below ~80 FPS druing per frame destruction on my machine.*
### How I can make it work even faster?
*Slightly increase number of chunks or reduce number of operations done on world per frame.*
*Reduce unnecessary per frame changes to the terrain as it makes chunk to recalculate it's colliders.*
*Recalculating colliders takes about 90% of all computation for DTerrain, keep that in mind.*
*Also, chunks are based on SpriteRenderers meaning the more chunks camera sees the slower it will work. To reduce lag separate typical collision layer into two layers: visible (only sprite renderer - one big chunk) and logic (only collision - many chunks). See ``SampleScene2Optimized`` to see the idea behind it.*
### I can't access any of the components from this package.
*Make sure you add ```using DTerrain;``` at the begining of you scripts.*
### Will it work with my Unity version?
*It should as code is universal and doesn't use version specific tweaks in Unity (only BoxColliders2D, SpriteRenderers and SpriteMasks).*
### Can I use it for free?
*Yes. Now and forever. You don't have to credit me, but I'd really like to know if you built something meaningful with it.* 

## Special thanks:
- **/u/idbrii** for pointing out a fix that nearly doubled the FPS.

## Features in the future:
- Rewriting the whole code to be more expandable (DONE ✔️)
- Adding mesh optimization (DONE ✔️ - separate collision logic and visible layers: see ``SampleScene2``)
- Adding wiki on how to use it (IN PROGRESS 🔜) 

## Crust port: plan

This fork is being ported to C with [crust](https://github.com/brentharts/crust),
aimed at small, bounded games: few objects, every count known when the game is
packed. The pieces go where they fit:

* **Model** (`Range`, `Column`, `Shape`): plain data and loops, through crust's
  C# subset (CSRUST.md).
* **Chunks, layers, texture sources** (MonoBehaviours): through crust's
  unity_pack.
* **Colliders**: straight into [Box2D-Packed](https://github.com/crustos/box2d)
  shapes, instead of `BoxCollider2D` components.
* **Inspector code** (`HelpAttribute`) is not part of it: it stays behind
  `#if !CRUST` (crust defines `CRUST`), as it is already behind `UNITY_EDITOR`.

Every change keeps the source plain C# that Unity compiles as before.

### Status

8 of the 28 scripts translate through crust's C# subset on their own: `Range`,
`Shape`, `UnityExtensions` and the interfaces (`IChunk`, `IChunkCollider`,
`ILayer`, `ITextureSource`, `ITextureGenerator`). The rest stop at a named line
with a reason: the MonoBehaviours belong to unity_pack, and the remaining gaps
are listed under step 6.

### Where the time goes, and what changes

The README above says it: recalculating colliders is about 90% of the work.
Three things make it so, and each has a direct fix.

1. **The quadtree tests pixel by pixel.** `QuadTreeToRect` asks
   `chunk[i].isWithin(j)` -- a scan of the column's ranges -- for every pixel of
   every node, at every level. The data is already run-length: whether a
   rectangle is all ground, all air or mixed is a question per *column*
   (does one range cover rows y..y+h, does any range touch them), so a node
   costs columns x ranges, not pixels x ranges. Same rectangles, same result,
   in Unity too.
2. **Colliders are components, created and destroyed.** Every rebuild
   `AddComponent<BoxCollider2D>`s and `Destroy`s, and matches old to new with
   `colls.Find(lambda)` -- quadratic in the number of boxes. Packed, a chunk is
   one static Box2D body and its rectangles are box shapes
   (`b2MakeOffsetBox`): a rebuild destroys the chunk's shapes and creates the
   new ones, no component bookkeeping. Further, in order of payoff:
   * **merge rectangles** greedily along rows before making shapes -- far fewer
     shapes than quadtree leaves;
   * **outline instead of fill**: the terrain's boundary as `b2ChainShape`s,
     traced from the column ranges. A solid region becomes one chain however
     large it is, and chains are smooth for things rolling over them: no
     catching on the seams between boxes.
3. **Every span is an object.** `Range` is a class, so each `[min;max]` is a
   heap allocation, and destruction allocates them by the hundred. As a
   struct it is two ints in a list -- in C a plain array. One care point:
   `Range` overrides `Equals` with value equality, and `Column` removes ranges
   with `List.Remove`, which uses it; the port keeps that meaning (it must not
   become reference equality).

### Plan, in order

1. **Repository.** 210 MB of Unity's generated `Library/` (8,638 files) is
   committed: the root `.gitignore` says `/[Ll]ibrary/`, which only matches a
   Library at the repository root, and the project is in
   `Destructible Terrain/`. Ignore the project's `Library/`, `obj/`, `Temp/` and
   `Logs/` and remove them from the index; Unity regenerates them.
2. **Model as values.** `Range` a struct; `Column`'s ranges a list of them.
   crust: `List.Remove` / `IndexOf` / `Contains` through a user-defined
   `Equals` (today it refuses anything but primitives and enums).
3. **Column-wise quadtree** (fix 1 above) -- a change in plain C#, faster in
   Unity as well.
4. **Box2D shapes per chunk** (fix 2): one static body a chunk, rectangles
   merged, then chains. In crust's unity_pack this is a chunk-collider kind that
   the packed engine hands to Box2D-Packed, not a stream of AddComponent calls.
5. **Bounded counts.** The chunk grid follows from the texture size and the
   chunk count, both known when the game is packed: `[MaxInstances(N)]` on the
   chunk and layer classes, tables sized once. Per-chunk textures update in
   place: only the rectangle that changed is uploaded (`glTexSubImage2D` in the
   GLES hosts), not the whole chunk through `SetPixels` / `Apply`.
6. **C# subset gaps** the scripts hit, each small: `?.` (CollidableChunk,
   the example scripts), `base.Method()` / `base.Property` (ComplexChunk,
   BlankSingleTextureSource), string interpolation in log text (PaintableLayer:
   behind `#if !CRUST`, or a crust lowering), `new Color[n]`, `List.Find` with a
   lambda (the loop it stands for, after step 4 it is gone anyway), a generic
   base class (`PaintableLayer<T>`), and a field named like its enum
   (`PaintingMode PaintingMode`).
