# Blender to three.js export guide
📋 _This is a living document. The landscape for this topic is constantly changing and so this document will need to evolve with it. If anyone has anything to add/edit/remove, please create a pull request or start a discussion with an issue._

## It's best to export to glTF
glTF is the open standard for 3D models on the web. It is the format recommended in the three.js documentation. The three loaders for other formats (e.g OBJ, FBX) are not well maintained and it is likely many of them will end up very buggy or completely broken. This guide only explains how to export to glTF.

## Test with the glTF viewer
After exporting, it's best to test with the [glTF Viewer](https://gltf-viewer.donmccurdy.com/) by Dom McCurdy. This is a quick and easy way to check your model. If you test with your own code, there's a chance that any problems you experience are from your code and not the model.

## Which Blender version?
2.8 is not currently considered stable, so the "safest" answer is use 2.79. However 2.8 does have some nice features that help with exporting to glTF. A quick overview:

### 2.79
- ✔️ Stable version of Blender
- ✔️ More compatible addons (i.e. [Apply modifier to object with shape keys](https://github.com/przemir/ApplyModifierForObjectWithShapeKeys))
- ❌ glTF exporter no longer maintained ([glTF-Blender-Exporter](https://github.com/KhronosGroup/glTF-Blender-Exporter))
- ❌ Textures don't export when going via FBX to glTF convert route

### 2.8
- ✔️ glTF exporter actively maintained ([glTF-Blender-IO](https://github.com/KhronosGroup/glTF-Blender-IO))
- ✔️ glTF exporter pre-installed
- ✔️ Good material export support
- ❌ Many addons are not compatible

## Exporting a simple mesh (no bones or animation)
If all you are doing is exporting a simple mesh, your best bet is to export straight to glTF using [glTF-Blender-Exporter](https://github.com/KhronosGroup/glTF-Blender-Exporter) (Blender 2.79) or [glTF-Blender-IO](https://github.com/KhronosGroup/glTF-Blender-IO) (Blender 2.8)

## Exporting something complicated (bones / shape keys)
This is where exporting straight to glTF doesn't work so well. Feel free to try (as it may be fixed by the time you read this) but as of the time of writing, exporting a model to glTF with bones seems to cause many issues.

Instead, the best route here is to export to FBX and then convert to glTF using [FBX2glTF](https://github.com/facebookincubator/FBX2glTF). Blender 2.8 plays best here, with 2.79 you will loose all your textures. However, if you are using shape keys and modifiers, you may want to consider using 2.7 (see below).

### Armatures / Bones
- Don't try and export multiple armatures. Work with one armature per glTF file.
- Inverse Kinematics work, but animations will need to be baked in the export setting. Also, if you're using empties as targets, you'll need to make sure "Empties" is checked in the FBX export setting.
- Parenting objects outside of the mesh with a bone is buggy (e.g. weapons, hats). Feels like there might be a way to get this working (please share if you work this out).

### Shape keys / Morph Targets
Shape keys should convert to glTF "morph targets". However, if you're using modifiers such as "mirror" or "subdivision surface", you will not be able to apply these once you've created shape keys. This is a limitation in Blender, you can't apply modifiers to objects with shape keys. This is problematic, because this will need to happen on export (via an option). Note that armature modifiers seem to export fine without any extra effort. You have a few options here:
- Apply all your modifiers before adding shape keys. This is the simplest option, but is somewhat destructive.
- Keep your modifiers and do a [manual workaround](https://blender.stackexchange.com/questions/56795/shape-keys-and-applying-subdivision-surface-modifier) to apply modifiers. This could be quite time consuming.
- If you're using Blender 2.7, use the [Apply modifier to object with shape keys](https://github.com/przemir/ApplyModifierForObjectWithShapeKeys) addon. Note that using Blender 2.7 means you'll lose textures when exporting.
- Convert the above plugin to be compatible with Blender 2.8 (If you do this, please share on this repo via an issue or PR!!!)

### Exporting animations (NLA Editor tips)
If you want separate animations, you'll need to save them as "actions" in Blender and then make sure they are in the NLA editor. This part of the software is quite unintuitive, especially when it comes to exporting, it's highly advised to read through the Blender manual on the [NLA Editor](https://docs.blender.org/manual/en/latest/editors/nla/introduction.html). Some relevant tips below:
- Add new actions with the "action editor". You add keyframes as you would with the normal timeline. These apply to one object each.
- Shape keys can also be animated here, under "Shape key editor"
- When you're happy with an action, make sure it's "pushed down" into the NLA editor. Actions that are pushed down are no longer related to the actions you can choose from in the action editor. If you change something in the library of actions, it won't affect the strips in the NLA editor.
- If you want to edit an action that has been pushed down (a strip), select the strip and press tab. It goes green (this is known as tweak mode) and your edits will be saved to that strip if you press tab again.
- Actions on different places on the timeline will be exported as seperate animation clips. Actions above and below each other, layered on the same place on the timeline, will be combined and exported as a single animation clip.
- Make sure each action is set to "Nothing" under "Extrapolation" in the properties panel (sub group "Active Strip"). This prevents the last frame of one strip affecting strips further along the timeline. To see the properties panel, make sure you have a strip selected in the NLA editor and press "N". This prevents the strips from inteferring with each other further down the timeline.
- When animating an object, make sure its transforms are all set to zero. To do this, you need to "Apply" them. Press CTRL+A to do this when selecting the object in the 3D view. If you've done it correctly, you should see Location and Rotation set, scale set to 1 (view in the properties panel). This makes sure the object doesn't move to weird places when it is not being animated as part of a strip.
- When switching between actions, make sure you have the correct object selected in the 3D View, otherwise the wrong object will suddenly have the animation applied to it

### A note on mixing actions once in three.js
Because of the way actions are exported, blending animations doesn't work too well for anything other than transitions. Let's say you have a figure running, and also a figure standing and waving. In an ideal world, you'd be able to blend these two in your application and have a figure running and waving. Unfortunately, instead you'll have a figure halfway between the two states (e.g. walking and half waving). This is because Blender exports all static information as well as important stuff. 

This even applies to animations with multiple meshes. For instance, if you had a character with a hat as a seperate object and you wanted a separate hat wobble animation, you won't be able to blend that animation with something else. The best thing to do here is export each mesh seperately and combine them in three.js. One other option is to manually edit the GLTF JSON file to remove certain animation information, but this really isn't fun.

Note this also applies to shape keys in animations. While they blend perfectly well as separate morph targets, when shape keys are part of an animation they blend in a similar way as explained above.

## Important FBX export options
As mentioned above, for complex models, the best option is to export to FBX and then convert to glTF using [FBX2glTF](https://github.com/facebookincubator/FBX2glTF). Below are the important options for exporting to FBX from Blender.

### Main 
- Version: FBX 7.4 binary
- Exported features: "Empties", "Armature", "Mesh"

### Geometries 
- Apply Modifiers

### Animation
- Baked Animation
- NLA Strips (Not "All Actions". See NLA Editor tips above for explanation)
- Force Start/End Keying (Enable this if you have any single keyframe animations)

## Useful links
- [All exporters/converters and their features](https://github.com/KhronosGroup/glTF/issues/1271)
