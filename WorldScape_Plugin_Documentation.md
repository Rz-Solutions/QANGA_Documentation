# WorldScape Plugin - Complete Documentation

> **Source:** [https://iolacorp-1.gitbook.io/worldscape-plugin](https://iolacorp-1.gitbook.io/worldscape-plugin)

---

## Table of Contents

- [WorldScape Documentation (Home)](#worldscape-documentation)
- [World Generation](#world-generation)
  - [Quick Start](#quick-start)
  - [Terrain Geometry](#terrain-geometry)
    - [Noise Generation](#noise-generation)
  - [Noise System](#noise-system)
    - [Noise Data Asset](#noise-data-asset)
    - [Apply your NoiseVolume](#apply-your-noisevolume)
    - [Custom Noise Class](#custom-noise-class)
  - [HeightMap System](#heightmap-system)
    - [HeightMap Data](#heightmap-data)
    - [Apply your Heightmap with Volume](#apply-your-heightmap-with-volume)
    - [Apply your Planetary Heightmap](#apply-your-planetary-heightmap)
  - [Hole Volume](#hole-volume)
  - [Collision Invoker Volume (Beta)](#collision-invoker-volume-beta)
  - [Procedural Foliage](#procedural-foliage)
    - [Placement Constraint Parameters for Foliages](#placement-constraint-parameters-for-foliages)
    - [Foliage Collection](#foliage-collection)
    - [Foliage Cluster](#foliage-cluster)
    - [Regular WorldScape Foliage](#regular-worldscape-foliage)
    - [Applying your Foliage Collection](#applying-your-foliage-collection)
    - [Foliage Layer Masking](#foliage-layer-masking)
    - [Foliage Volume](#foliage-volume)
    - [Foliage Baker (Beta)](#foliage-baker-beta)
  - [Terrain Material](#terrain-material)
    - [Terrain Material Legacy](#terrain-material-legacy)
  - [Setting up a World from Scratch](#setting-up-a-world-from-scratch)
  - [WS Tools](#ws-tools)
    - [How to Get Toolbar Running](#how-to-get-toolbar-running)
    - [Detailed Overview](#detailed-overview)
    - [Heightmap Editor (Beta)](#heightmap-editor-beta)
    - [Heightmap Tools Volume (Beta)](#heightmap-tools-volume-beta)
- [Grid Spawner](#grid-spawner)
  - [Planetary Grid Data](#planetary-grid-data)
  - [Grid Data Asset](#grid-data-asset)
- [Snap Actor](#snap-actor)
- [Volume Pooling](#volume-pooling)
- [Foliage Collision Pooling](#foliage-collision-pooling)
- [Gravity Value](#gravity-value)
- [SubSystem](#subsystem)
- [FAQ](#faq)
- [API Informations](#api-informations)
- [Unreal Engine 5 Setting Fixes](#unreal-engine-5-setting-fixes)
  - [Material Fix](#material-fix)
- [Blueprint API Infomations](#blueprint-api-infomations)
  - [Get Ground Height](#get-ground-height)
  - [Get Ground Height Normalize](#get-ground-height-normalize)
  - [Get Ground Noise](#get-ground-noise)
  - [Get Height](#get-height)
  - [Get Height Normalize](#get-height-normalize)
  - [Get Noise](#get-noise)
- [WorldScapePlayer](#worldscapeplayer)
  - [Custom Character Gravity Legacy](#custom-character-gravity-legacy)
- [Technical Support](#technical-support)
- [Play to Demo](#play-to-demo)

---

# WorldScape Documentation

WorldScape is a plugin for Unreal Engine 5. Create a planet and infinite flat landscape using geometry clipmaps.

**Welcome to documentation for WorldScape Plugin.**

Play on the surface in single player or multiplayer. Guide your imagination to quickly achieve fully procedural infinite playgrounds using noise customize generation through our heightmaps decal system importing 8-bit or 16-bit Heightmaps. The 64-bit precision is executed in runtime to define the surface mesh easily add foliage to your terrain using the procedural foliage system based on humidity and temperature of the surface. Work in a plugin designed for real time.

### Features

- Create an infinite world playable in single player and multiplayer flat and spherical.
- Geometry technology 64bits precision for build surface.
- Create infinitely large and tiny flat worlds.
- Create infinitely large and tiny planets.
- Instanced and multithreaded procedural foliages. (Spawn thanks to the temperature and humidity of the geometry of the ground.)
- Import HeightMaps in 8bit 16bits and apply them to the surface as decals. (They can be added and merged.)
- Editable terrain collision.
- Noise Biome customizable in C++.
- Realistic Lighting for planet.
- WorldShiping Fake 64bits Squared.
- Materials procedural for surface.
- Custom MovementComponent BP.
- Custom Gravity Behavior for planet.

For any kind of support, please visit the [Technical Support Page](#technical-support).

And if you're using Unreal Engine 5 please visit the [Unreal Engine 5 Setting Fixes](#unreal-engine-5-setting-fixes) to avoid issues.

For users still using engine versions below 5 some of the original docs have been saved.

**This documentation will be improved over time.**

---

# World Generation

## Quick Start

If you prefer to quickly see the technology in action, you can read this page.

### You can open demonstration levels

For good compatibility, enable these native plugins to UnrealEngine below 5.0:

- Enable the **Water plugin** (for ocean material)
- Enable the **Sun Position Calculator** plugin (for good sun setup)

Enable Plugin Content in your ContentBrowserSettings. You may need to enable the show engine content if Plugin is not in project files. A restart may be necessary.

And if you are experiencing surface disappearance or flickering use this console command:
```
r.UseVisibilityOctree 0
```

If you are using 5.0 and above use these options if you are experiencing any visual artifacts: open your project settings and click the all settings tab then in the search bar type in "velocity pass", be sure this is enabled for 5.X to avoid visual bugs.

You are about to create your world. But before that, you have to understand how WorldScape works! Install the plugin and open levels "Demonstration". Wait for the shaders to compile, and press Play on the selected levels shown below in the files.

Here you are in an infinite flat world and an explorable planet above you. The surface is generated in real time under your feet. Now it's up to you to choose what you want to do for your project. You can see in the WorldOutliner, the presence of two actors for the ground. WorldScapeFlat and WorldScapePlanet. Know that you can cumulate grounds. (Flat/planet world).

You can choose the seed in the NOISE menu of the WorldScape actor, a number from (1 to ∞) for a unique visual result.

When you are ready to create a level from scratch, go ahead and refer to the [Setting up a world from scratch](#setting-up-a-world-from-scratch) section for level creation with WorldScape.

### Making big world?

If your goal is to create a huge world, you will have to use World Origin Shifting. Use the "SetWorldOrigin" Function in blueprint and make the origin follow your player location.

---

## Terrain Geometry

WorldScape plugin brings clipMaps technology to the UnrealEngine to create endless landscapes and planets.

### Generation Parameter

You can drag and drop the **WorldScapeManager** BP actor into your level.

You can use the **C++ WorldScapeRoot** directly, the difference with the BP version is only demonstrates setup parameters and allows for easier customization. (If you need to make specific gameplay logic with your terrain you can create a BP actor from the WorldScapeRoot Actor.)

Now you have to choose the type of generation, flat and infinite or spherical with a size to define.

- Choose if your terrain is flat or if it is a planet.
- **Planet scale** allows to define the size of the planet by radius.

### Collision

You can manage terrain collisions in the Collision menu of the WorldScape actor in your world.

Collisions are generated around the player, 1 at the player position, and 8 around. They are generated in safe thread (on the main thread) and are actually very expensive. For this reason, collision resolution should stay low.

**Collision size** is: `Collision Resolution` × `Collision Triangle Size`.

#### Show Collision

Allow the display of the collisions, can only be enabled if collisions are enabled.

#### Collision Dependent Actors

Since generating collision is very expensive, we are limiting collision generation to only be around the player. But you will often need collision to be generated outside of player proximity. For this, **Collision Dependent Actor** is a list of actors that WorldScape will use to generate additional collisions.

### Level Of Detail (LOD)

You can define the definition of your field. The minimum recommended is **MaxLOD: 8** if you have low terrain height variation. But if your game requires altitude, then **MaxLOD: 12** is advised.

- **LOD Resolution:** Number of vertices generated on each axis. Higher resolution allows fine detail to be drawn farther but leads to more expensive terrain generation (slow generation), and can at very high values (500+) lead to lag spikes.
- **TriangleSize:** Distance between each vertex of the terrain when near the ground. If you want finer detail, this is the value to edit.
- **Height Anchor:** As you gain altitude, the detail you need on the ground is lower. This is the distance to ground at which the terrain generation will start doubling the triangle size.
- **Lod Angle min/Max:** This is used for planet type world. The ClipsMaps are projected to the planet, and to allow consistent grid as you move, the grid plan snaps at a set angle of the planet. This has no impact on performance, but low values can lead to more frequent complete generation of the terrain (where it usually only regenerates parts needing update). Should always be a divisor of 360.

### Ocean

The ocean works like the terrain. It uses ClipMaps. You can define a material. Here the material worldScape is a child of the material water from 4.26. (Floating works.)

As for the terrain you can define the resolution / LOD / triangle size. The ocean is deactivatable.

WorldScape is compatible with **Oceanology 5** if you use it on a flat world. For a planet, you have to change shaders & several things. (Tutorial soon.)

### Informations and Debug

The debug information allows you to visualize the distance between the player and the ground, know the altitude and the active list in the load.

---

### Noise Generation

Details about the noise generation and how to use it.

#### Temperature & Humidity System

Noise can output any number of parameters the developer needs.

By default the current system registers 3 parameters generated from the noise in the Mesh Vertex Colour:

- **Height** (Red Channel)
- **Temperature** (Green Channel)
- **Humidity** (Blue Channel)

These are used by the material to color the grass, put mountain and snow texture, or any creative way you can find. (Refer to [Terrain Material](#terrain-material).) If you ever want to change this, you'll have to edit the plugin source code.

#### Noise Seed

Change value in the Seed parameter to generate different terrain variations.

#### Editing the Noise in C++

WorldScape plugin is aimed to have the noise edited in C++. You have the possibility to modify the noise yourself and integrate your own Noise and parameters. There is a barebone example of the noise with comments to guide you.

#### FNoiseData

The noise data are stored in a FNoiseData Struct. You can edit it to fit your need. By default, WorldScape will use and need the following parameters:

- `double` **HeightNormalize** - The height normalized from 0 to 1, also registered in the Red channel of Vertex Color.
- `double` **Height** - The height in Unreal Unit that will displace the terrain either from the center of the planet or up depending on the generation type.
- `float` **Temperature** - Registered in the Green channel of Vertex color, also used for foliage masking.
- `float` **Humidity** - Registered in the Blue channel of Vertex colors, also used for foliage masking.
- `float` **WaterMask** - Used for foliage masking, also set to true for HeightMap Influencer when ocean is higher than terrain.
- `float` **FoliageMask** - Used for foliage Masking.

#### Noise Parameter

Noise Parameter is a struct used as an interface to influence the noise from the viewport. It is present in the WorldScapeRoot component to allow developers to interact with your noise. You can use it to simply change the scale/intensity of the noise, or allow to have completely different generation rules.

#### Required Functions

WorldScape will use 2 specific functions from the WorldScapeNoiseClass:

- `FNoiseData` **GetHeight** - responsible for terrain Noise.
- `FNoiseData` **GetOceanHeight** - responsible for Ocean Noise. It can be used for ocean, river and lake generation.

Rest of the class can be edited as you see fit. You are free to follow the already existing guideline and biome system or write your own system.

---

## Noise System

The Noise asset is very important for making unique looking landscapes. You can create a noise library folder to store your types of preset or custom made noise. Right click in the content browser and in the WorldScape Tab you will see the noise and for this instance call it "NoisePreset1".

Most of the settings you see are very straight forward and do what you expect; however, we are focusing on the "WorldScapeNoise". If you already created the "NoisePreset1" you can drag it in the "World Scape Noise" slot. When you open it, the settings are very straight forward. It is recommended that you experiment with the settings to get a feel of what each one does.

### Example Noise Screenshots

Various noise configurations can produce dramatically different terrains - from smooth rolling hills to jagged mountain ranges and crater-filled landscapes.

---

### Noise Data Asset

Noise Data is a way to save presets of built noise either you have made or using the default ones provided by the plugin.

Depending on what noise you are using, the parameters available will vary on the use of noise, but it is useful for loading in landscape types for your terrain.

---

### Apply your NoiseVolume

The Noise Volume can be used to set noise in a specific area. To create one, right click in the content browser, click "Blueprint Class", in all classes type "Noise Volume" and once created name it what you like.

Then drag it in the level and you're going to notice that in the details panel it looks very similar to the "BP_WorldScapeManager" noise tab. But next you need to make it where "BP_WorldScapeManager" will read the volume and in order to do that click it, scroll down to the Volumes Tab and click the Plus for the Noise Volume list and assign your NoiseVolumeBP.

**You're going to have to scale the NoiseVolumeBP up by a lot. Recommend at least a scale of 28000 or 5000000 on all axis for it to affect a large area. And like before experiment with the values that suits best for your needs.**

---

### Custom Noise Class

Video Tutorial for Custom Noise is available. (Refer to the original documentation page for the embedded video tutorial.)

---

## HeightMap System

Heightmap volumes are very useful for making specific areas look the way you want and work both with flatworld and planets. WorldScape Plugin already has some premade Heightmap Volumes which is shown below in **"Worldscape Content \ Blueprints"**.

Now if you want to create your own Blueprint you can do like you did with the Noise volume but instead type "HeightMapVolume".

#### Creating a HeightMapVolumeData

Now whichever method you did you can drag the HeightMapVolumBP into the level and when you click it you're going to be treated with some new things.

As you see there are some settings and like before go off by their names. But we now have the "HeightMapData". This allows you to choose what kind of heightmap you want to use.

WorldScape Plugin comes with one premade HeightMapData but we are going to make our own. Right click in the content browser and in the WorldScape tab click the HeightMap Volume Data and name it what you like (e.g., "Crater").

Now right click in an empty area of your content browser and click on the WorldScape panel, click on HeighMapVolumeData.

**Side note:** When you double click it, it may not show the settings immediately. If it doesn't, go to the Window tab and click on Details and it will show the settings.

Open it, and you can add a texture heightMap. If your texture is in 16bits check the highResolution parameter. You can export your heightmap from a landscape and convert it to 16bits. Import the texture and add it directly in heightmap texture.

Click to **GENERATE**.

Drag and Drop this in same level to your WorldScape terrain. Add your Heightmap Data to your WorldScape Manager BP.

Look this video: https://youtu.be/_Z3frULvurU

---

### HeightMap Data

HeightMap Data is the data asset that stores the heightmap texture and configuration. You can configure it with 8-bit or 16-bit textures, and toggle the high quality parameter for 16-bit textures to avoid stair-step artifacts.

---

### Apply your Heightmap with Volume

Video tutorial: https://youtu.be/_Z3frULvurU

Add a Height Map Volume in your level. Then add it to the HeightMap Influencer list on your WorldScapeRoot Actor.

Add your Volume in list to WorldScapeActor (panel detail).

You can define the minimum and maximal height for both land and ocean, Temperature and humidity.

---

### Apply your Planetary Heightmap

Planetary heightmaps allow you to apply heightmap data to the entire surface of a planet, wrapping the heightmap texture around the sphere. This follows a similar process to the volume-based heightmap but is applied at the planetary scale.

---

## Hole Volume

**Experimental (use with caution)**

Hole volume can be used to make holes. Right click the content browser and click blueprint class and type "hole".

Apply it under the volumes tab in your WorldScape actor.

### Showcase Example

You can use the foliage mask and overlap the same area of the hole volume to remove the foliage.

If you do not see any effects from the hole volume you may have to edit the master material for the WorldScape manager and multiply the alpha vertex color at the end of the opacity mask lerp.

---

## Collision Invoker Volume (Beta)

Part of the WS tools (beta).

The Collision Invoker Volume provides a way to generate isolated collision for WorldScape terrain. It lays out a collision actor localized under the volume that automatically attaches to the WorldScape actor.

Key settings include:

- **Max Resolution** - limits triangle count to avoid editor performance hits.
- **Quad Size Target** - controls collision mesh resolution based on volume size.
- **Quad Size Auto Correction** - adjusts the target resolution if the volume is too large to maintain reasonable performance at the defined max.
- **Triangle Size** - represents the triangle size in the worldscape mesh generation.

To use, simply place the Collision Invoker Volume over the desired area and click "Make Collision". Make sure it is assigned to the WorldScape root. Invoker actors will appear under the volume and connect to the WorldScape terrain under the collision tabs.

---

## Procedural Foliage

**Video Tutorial:** https://www.youtube.com/watch?v=bwTI7dplQck

WorldScape offers the tools to create foliage systems around your landscape automatically so that way you are able to bring out more life into your world you are creating. Refer down into the embedded pages to learn how they work and how you are able to use them.

---

### Placement Constraint Parameters for Foliages

The placement constraint parameters allow you to control where foliage can spawn based on terrain properties such as height, temperature, humidity, slope angle, and water mask values. These parameters give fine-grained control over biome distribution.

---

### Foliage Collection

Foliage Collection is a data asset that groups multiple foliage types together. You can create foliage collections to organize different vegetation types and assign them to your WorldScape terrain.

---

### Foliage Cluster

Foliage Clusters allow you to group foliage instances together in natural-looking arrangements rather than having uniformly distributed individual instances.

---

### Regular WorldScape Foliage

Regular WorldScape Foliage is the standard foliage spawning method that uses instanced static meshes for efficient rendering of vegetation across the terrain.

---

### Applying your Foliage Collection

To apply your foliage collection, add it to the foliage collection list in your WorldScape actor's foliage tab. The foliage will automatically spawn based on the terrain's temperature and humidity values combined with your placement constraint parameters.

---

### Foliage Layer Masking

Foliage Layer Masking allows you to exclude foliage spawn in specific areas. This is useful for creating clearings, paths, or areas where buildings will be placed.

---

### Foliage Volume

Foliage Volumes are area-based volumes that can override or modify foliage spawning within their boundaries. They can be used to create specific vegetation zones independent of the global foliage settings.

---

### Foliage Baker (Beta)

Foliage Baker is a beta feature that allows you to bake procedural foliage into static foliage instances for improved performance in areas that don't need dynamic foliage generation.

---

## Terrain Material

*Caution: The materials inside the project are placeholder. New materials, better suited for WorldScape, will be made and provided in the future. This article will explain the role of each parameter without going too far into details. Feel free to use the current materials as examples and create new ones or modify them to suit your needs.*

### Material Overview

The material for WorldScape 5.0 and above functions differently than the legacy to allow for more customization to your landscape allowing for both flexibility and realism. Most of the controls for the material are controlled in the blueprint actor of "BP_WorldScapeManager".

If you are using Unreal Engine 4.26 - 4.27, refer to [Terrain Material Legacy](#terrain-material-legacy).

### World Material Settings

When using WorldScape you may notice these options:

These new settings control the appearance of the landscape, and most of the values are straight forward. It should be noted that:

- **Bottom layer** - controls the coastline aspects of the landscape
- **Mid color layer 1** - controls the humid areas
- **Mid color layer 2** - controls the dry areas
- **Top layer** - controls the snow caps and overall snow color
- **Surface noise** - controls texture details seen from space

#### Mid Layer Color Information

**Mid color layer 1:**
- Colors 1-3 control the temperate grass color
- Color 4 controls the cold tundra colors closer around the colder parts near the edge of snow
- Color 5 controls the savanna colors around the landscape and more noticeable to the transition of desert

**Mid color layer 2:**
- Colors 1-2 control the sand color around the landscape
- Color 3 controls the top rock colors seen on mountains or dry cold areas
- Color 4 controls the tint of the sand
- Color 5 has no effect on anything

As for the **Layer tab**, that controls the height and sharpness transition of each of the primary layers (bottom, mid, and top).

Keep note that the foliage does not respond to the material. But given some work into it, you can make your own system or modify that foliage material to correspond to the colors associated.

---

### Terrain Material Legacy

The legacy terrain material system was used in Unreal Engine 4.26 - 4.27. It provides a triplanar material setup for the WorldScape terrain. For users on engine versions below 5.0, refer to this section for material configuration.

---

## Setting up a World from Scratch

So first off in the Place Actor tab type in "WorldScape" and you will see "WorldScapeRoot" and drag it into the scene, as well as a Directional Light. The same can be applied to the WorldScape Manager when following through this section on adding WorldScape terrain to empty worlds.

Now placed in the scene, for this instance for a planet, make sure you set the generation type to **"planet"** and have **Generate** and **Generate Ocean** checked.

You're immediately going to notice you don't see anything — that's because the planet's surface is way above. So you can just paste this coordinate for the transform of the planet:
```
(X=0.000000, Y=0.000000, Z=-637875790.000000)
```

Then scroll down a bit and you'll find where to put materials. You can copy over the materials from the planet example map or make your own.

Once that's done you can then scroll back up to where you see the Foliage tab and you can add foliage collection to that list.

And you can add other things such as atmosphere and skylight and you have your planet.

**As for flat world,** it's basically the same except that you choose "Flat World" and make sure the transform location is set to `(0, 0, 0)`.

---

## WS Tools

Worldscape (WS) Tools is a powerful and user-friendly plugin feature designed to enhance the planetary options within the landscape plugin. This toolset offers a dedicated toolbar that simplifies navigation and provides a seamless experience when working with planetary landscapes. WS Tools gives ease to users for the ability to explore, manipulate, and create stunning virtual worlds with ease.

**Please keep in mind** that some of these features are in beta and the following items may change or be removed and may be unstable. So some information may already be outdated by the time of release and changes to the docs will be made during full stable release of WS Tools section.

---

### How to Get Toolbar Running

Instructions for enabling and configuring the WS Tools toolbar in your Unreal Engine editor. The toolbar provides quick access to WorldScape-specific tools and navigation options.

---

### Detailed Overview

A comprehensive overview of all WS Tools features and their functionality within the editor environment.

---

### Heightmap Editor (Beta)

The Heightmap Editor provides tools for:

- **Edit Heightmap Data (Beta)** - Edit existing heightmap data directly within the editor
- **Create Landscape (Beta)** - Create new landscapes using the WS Tools interface
- **Landscape Export (Beta and Extras)** - Export landscapes and heightmaps

---

### Heightmap Tools Volume (Beta)

The Heightmap Tools Volume provides additional heightmap manipulation capabilities within a defined volume area.

---

# Grid Spawner

> **WARNING:** The documentation is not finished. Grid is Experimental.

Grid Spawner is an alternative spawn system that only spawns on planet surfaces.

The system allows:

- Spawning on both the client and server sides
- Maintaining a pre-defined location structure, with randomness control applied
- Spawning actors with parameters (serialization)
- Auto-snapping spawned elements onto the planet's surface while maintaining their overall structure

---

## Planetary Grid Data

Planetary Grid Data defines the grid structure used for spawning elements on planetary surfaces. It controls the distribution pattern and density of spawned objects.

---

## Grid Data Asset

Grid Data Asset stores the configuration for grid-based spawning, including spawn patterns, density settings, and actor references.

---

# Snap Actor

> **WARNING:** The documentation is not finished. Snap is Experimental.

Snap Actor is a derivative of Grid Spawner, except that this one is directly by placing an actor on the surface of the planet.

The system allows:

- Spawning elements (StaticMesh and actor, serialized or not) with a general structure maintained
- Snapping elements to the surface of the terrain
- Managing the spawn quantity directly with a manager

---

# Volume Pooling

> **WARNING:** The documentation is not finished. Volume Pooling is BETA.

Volume pooling is a method that allows you to keep the number of active volumes low in the field.

You can at any time in C++ or Blueprint request a volume from WorldScape root, which will be a volume that is part of the pooling list, then configure it and place it where you want.

---

# Foliage Collision Pooling

> **WARNING:** The documentation is not finished. Foliage Collision Pooling is BETA.

Foliage Collision Pooling is a new method for foliage collisions to address micro-freezing issues when spawning large amounts of foliage.

This allows all foliage to spawn without collision, then automatically moves the collision around the player, resulting in a significant performance gain when spawning foliage, especially at great distances.

This system only works on the client side.

---

# Gravity Value

> **WARNING:** The documentation is not finished. Gravity is BETA.

Gravity Value is a simple system that allows you to calculate the gravity at a given point on planets and flat worlds. This allows you to directly obtain a usable value to determine the gravity at a given location.

Currently, no force is applied to objects; you must obtain this value yourself from the planet or from the subsystem.

---

# SubSystem

> **WARNING:** The documentation is not finished. SubSystem is BETA.

The Subsystem now supports several basic routines:

- It allows you to update the terrain closest to the player, which allows you to transfer parameters from one terrain to another.
- It allows you to know the orientation from the center of the planets in real time.
- It automatically updates a Material Parameter Collection with planet and flat world parameters.
- It allows you to know when a planet is registered or when it is destroyed.

All systems have delegates that you can connect to your own system.

---

# FAQ

### I can walk on planet surface?

Yes.

### It's possible to use the material landscape provided for the original unreal landscape on clipmaps ground?

Unfortunately No, the Landscape material is designed to work with the LandscapeComponent. But the plugin offers an example triplanar material, you can improve it to make it as beautiful as that of your old landscape.

### Can I sculpt terrain in editor?

No, for now, you need to sculpt your terrain with heightmap software and export the heightmap. Use your heightmap in Influence to apply it to WorldScape terrain.

### Why my heightmap texture have artefact in terrain?

There are different types of artifact:

- If you have stairs on your heightmap, it's a precision issue of your texture. It is advised to use 16-bit textures and toggle the "High Quality" bool in the HeightMapInfluencer Data.
- If the artifacts are weirder (like random values) it is possible you are either: Using a non-power of 2 texture (resolution of 128×128, 256×256, 512×512, etc.) or using an 8-bit image as a 16-bit image, leading to texture reading issues.
- If you have any other issue, feel free to come on the [Discord](https://discord.gg/wzUNzPKuU2) and report the issue.

### Why are distant mountains suddenly appearing?

It's probably because your **MAX LOD** setting is too low. You should increase its value (advised value of 10-12).

### Why does the influencer Heightmap act weird to the terrain, yet I have correctly configured?

Sometimes it is necessary to restart your level or even your project to apply data. If it still doesn't work please join the [Discord Server](https://discord.gg/wzUNzPKuU2) for specific help.

### BlackScreen in Demonstration Level?

Enable the Volumetric Plugin or Disable CloudVolumetric in level.

### Fail Package?

**Fix:** Enable WaterPlugin.

---

# API Informations

## Functions

### `FNoiseData` GetGroundNoise(`FVector` Position, `bool` Water = false)

Get the Noise from ECEF Position (Planet Centered) projected to the ground.

- **Position** - Position to obtain the Noise From
- **Water** - Is the noise sampled for water or ground

### `FNoiseData` GetNoise(`FVector` Position, `bool` Water = false)

Get the Noise from ECEF Position.

- **Position** - Position to obtain the Noise From
- **Water** - Is the noise sampled for water or ground

### `FVector` GetPawnNormal(`FVector` PawnWorldPosition)

Get the UpVector from the WorldPosition.

- **PawnWorldPosition** - The position you want to get the up vector from.

### `FVector` GetPawnSnappedNormal(`FVector` PawnWorldPosition)

Get the UpVector from the WorldPosition (snap based on the angle parameters).

- **PawnWorldPosition** - The position you want to get the up vector from.

### `FVector` GetPawnTangent(`FVector` PawnWorldPosition)

Get the Tangent from the WorldPosition.

- **PawnWorldPosition** - The position you want to get the tangent vector from.

### `FVector` GetPawnBiTangent(`FVector` PawnWorldPosition)

Get the BiTangent from the WorldPosition.

- **PawnWorldPosition** - The position you want to get the bitangent vector from.

### `float` GetPawnAltitude(`FVector` PawnWorldPosition)

Return the altitude from a World Position.

- **PawnWorldPosition** - The position you want to get the altitude from.

### `float` GetPawnDistanceFromGround(`FVector` PawnWorldPosition)

Return the distance to the ground from a World Position.

- **PawnWorldPosition** - The position you want to get the distance from.

### `float` GetGroundHeight(`FVector` PawnWorldPosition)

Return the Ground altitude.

- **PawnWorldPosition** - The position you want to get the ground height from.

---

# Unreal Engine 5 Setting Fixes

For Unreal Engine 5.x, apply these settings to avoid visual issues:

1. Open your project settings and in the search bar type "velocity pass"
2. Make sure velocity pass is set to **"Write After Base Pass"** to avoid terrain visual artifacts
3. Include or put in console commands: `r.UseVisibilityOctree 0`

---

## Material Fix

Material-specific fixes for Unreal Engine 5 compatibility with WorldScape materials. Check the original documentation for specific material node adjustments needed.

---

# Blueprint API Infomations

WorldScape offers some blueprint functions. These can be found in the Info tab when you right click in the graph. Functions with descriptions are in separate pages below.

---

## Get Ground Height

Blueprint function to retrieve the ground height at a given position. Returns the altitude of the terrain surface.

---

## Get Ground Height Normalize

Blueprint function to retrieve the normalized ground height (0 to 1) at a given position.

---

## Get Ground Noise

Blueprint function to retrieve the full noise data projected to the ground at a given position.

---

## Get Height

Blueprint function to retrieve the height at a given position (not projected to ground).

---

## Get Height Normalize

Blueprint function to retrieve the normalized height at a given position.

---

## Get Noise

Blueprint function to retrieve the full noise data at a given position.

---

# WorldScapePlayer

In the plugin you are provided with a WorldScape player if you want to play test your planets if you don't have your own logic for the landscape yet.

The Blueprint can be found at the **PlayerLogic** folder under Blueprints in the WorldScape plugin.

---

## Custom Character Gravity Legacy

Legacy documentation for custom character gravity behavior. This covers the older method of implementing custom gravity for characters walking on planetary surfaces in engine versions prior to the current implementation.

---

# Technical Support

**WorldScape is constantly updated, so many issues can appear.**

In case of issue or problem linked to WorldScape, you should join the [Discord Server](https://discord.gg/wzUNzPKuU2) and send your bill to have access to the support channel.

---

# Play to Demo

Next months, a new version more in-depth will be published.

Please note that this demo was created quickly to meet the demand of the community. "Placeholder" objects and textures were used. Do not rely on graphics. This demo is for testing the possibilities of the plugin. The noise of the surface is very simple and the terrain is at medium resolution.

**PlanetSize: 6360km.**

Download link: [WorldScapePlay.zip (Google Drive)](https://drive.google.com/file/d/1sph3XtjhOU8kRgIK2AX0gctfyxJbWa87/view?usp=sharing)

**Help us to let us know:** Do not hesitate to notify WorldScape in your credits creation, it is very appreciated!

---

*This documentation was compiled from [https://iolacorp-1.gitbook.io/worldscape-plugin](https://iolacorp-1.gitbook.io/worldscape-plugin)*
