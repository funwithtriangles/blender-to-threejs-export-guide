# Blender to three.js export guide
📋 _This is a living document. The landscape for this topic is constantly changing and so this document will need to evolve with it. If anyone has anything to add/edit/remove, please create a pull request._

## It's best to export to glTF
glTF is the open standard for 3D models on the web. It is the format recommended in the three.js documentation. The three loaders for other formats (e.g OBJ, FBX) are not well maintained and it is likely many of them will end up very buggy or completely broken. This guide only explains how to export to glTF.

## Test with the glTF viewer
After exporting, it's best to test with the [glTF Viewer](https://gltf-viewer.donmccurdy.com/) by Dom McCurdy. This is a quick and easy way to check your model. If you test with your own code, there's a chance that any problems you experience are from your code and not the model.

## Which Blender version?
2.8 is not currently considered stable, so the "safest" answer is use 2.79. However 2.8 does have some nice features that help with exporting to glTF. A quick overview:

### 2.79
- ✔️ Stable version of Blender
- ✔️ More compatible addons (i.e. [Apply modifier to object with shape keys](https://github.com/przemir/ApplyModifierForObjectWithShapeKeys))
- ❌ glTF exporter no longer maintained [glTF-Blender-Exporter](https://github.com/KhronosGroup/glTF-Blender-Exporter)
- ❌ Textures don't export when going via FBX to glTF convert route

### 2.8
- ✔️ glTF exporter actively maintained [glTF-Blender-IO](https://github.com/KhronosGroup/glTF-Blender-IO)
- ✔️ glTF exporter pre-installed
- ✔️ Good material export support
- ❌ Bones are very buggy
- ❌ Animations are buggy
- ❌ Many addons are not compatible

## Exporting a simple mesh (no bones or animation)
If all you are doing is exporting a simple mesh, your best bet is to export straight to glTF using [glTF-Blender-Exporter](https://github.com/KhronosGroup/glTF-Blender-Exporter) (Blender 2.79) or [glTF-Blender-IO](https://github.com/KhronosGroup/glTF-Blender-IO) (Blender 2.8)

## Exporting something complicated (bones / shape keys)
This is where exporting straight to glTF doesn't work so well. Feel free to try (as it may be fixed by the time you read this) but as of the time of writing, exporting a model to glTF with bones seems to cause many issues.

Instead, the best route here is to export to FBX and then convert to glTF using [FBX2glTF](https://github.com/facebookincubator/FBX2glTF). Blender 2.8 plays best here, with 2.79 you will loose all your textures. However, if you are using shape keys and modifiers, you may want to consider using 2.7 (see below).

### Armatures / Bones
- Don't try and export multiple armatures. Work with one armature per glTF file.
- Inverse Kinematics work, but animations will need to be baked in the export setting. Also, if you're using empties as targets, you'll need to make sure "Empties" is checked in the FBX export setting.

### Exporting animations
If you want separate animations, you'll need to save them as "actions" in Blender. This part of the software is quite unintuitive, especially when it comes to exporting. Some tips:
- Add new actions in the "action editor"
- When switching between actions, make sure you have the correct object selected in the 3D View, otherwise the wrong object will suddenly have the animation applied to it
- When you're happy with an action, make sure it's "pushed down" into the NLA editor.
- Actions in the NLA editor, on separate strips, is how you'll export them

### Shape keys / Morph Targets
Shape keys should convert to glTF "morph targets". However, if you're using modifiers such as "mirror" or "subdivision surface", you will not be able to apply these once you've created shape keys. This is a limitation in Blender, you can't apply modifiers to objects with shape keys. This is problematic, because this will need to happen on export (via an option). Note that armature modifiers seem to export fine without any extra effort. You have a few options here:
- Apply all your modifiers before adding shape keys. This is the simplest option, but is somewhat destructive.
- Keep your modifiers and do a [manual workaround](https://blender.stackexchange.com/questions/56795/shape-keys-and-applying-subdivision-surface-modifier) to apply modifiers. This could be quite time consuming.
- If you're using Blender 2.7, use the [Apply modifier to object with shape keys](https://github.com/przemir/ApplyModifierForObjectWithShapeKeys) addon. Note that using Blender 2.7 means you'll lose textures when exporting.
- Convert the above plugin to be compatible with Blender 2.8 (If you do this, please share on this repo via an issue or PR!!!)

## Important FBX export options
### Main 
- Version: FBX 7.4 binary
- Exported features: "Empties", "Armature", "Mesh"

### Geometries 
- Apply Modifiers

### Animation
- Baked Animation
- Key All Bones
- NLA Strips (this one works better instead of "All Actions")
- Force Start/End Keying

## Useful links
- [All exporters/converters and their features](https://github.com/KhronosGroup/glTF/issues/1271)
