# Blender to three.js export guide
📋 _This is a living document. The landscape for this topic is constantly changing and so this document will need to evolve with it. If anyone has anything to add/edit/remove, please create a pull request or start a discussion with an issue._

## It's best to export to glTF
glTF is the open standard for 3D models on the web. It is the format recommended in the three.js documentation. The three loaders for other formats (e.g OBJ, FBX) are not well maintained and it is likely many of them will end up very buggy or completely broken. This guide only explains how to export to glTF.

## Test with the glTF viewer
After exporting, it's best to test with the [glTF Viewer](https://gltf-viewer.donmccurdy.com/) by Dom McCurdy. This is a quick and easy way to check your model without worrying if your own code is broken. It's also worth testing in the [Babylon Sandbox](https://sandbox.babylonjs.com/) if you're having problems with the glTF viewer. One known issue is with [skinned meshes scaling strangely](https://github.com/donmccurdy/three-gltf-viewer/issues/147).

## Which Blender version?
For the best results, use the latest version of Blender (2.92 as of writing). Find it on the [daily builds](https://builder.blender.org/download/) page of Blender

## PBR Materials
Exporting PBR from Blender can be done, but there are a few caveats. A lot of it is covered in [this guide](https://docs.blender.org/manual/en/2.80/addons/io_scene_gltf2.html), so definitely read that. Also:

- All texture maps (color, roughness, metalness) must be an image texture. If you're using nodes then you'll want to bake these first! 
- If you have a metalness map texture or roughness map texture, you *must* then have both set. These two textures are combined into one upon export and you'll come across issues if only one is set. If you're just setting both of these to numeric values, then you're good to go.
- Keep your file sizes low by choosing JPEG format in the export settings (under Geometry > Images).

### Armatures / Bones
- If you're working with Mixamo models, I highly recommend [this guide](https://www.donmccurdy.com/2017/11/06/creating-animated-gltf-characters-with-mixamo-and-blender/)!
- Don't try and export multiple armatures. Work with one armature per glTF file.
- Inverse Kinematics work, but animations will need to be "sampled" in export settings. Also, if you're using empties as targets, you'll need to make sure these are also exporting.
- Don't use bendy bones. These don't export for any of the standardised formats. One alternative could be to use normal bones and [Spline IK](https://docs.blender.org/manual/en/dev/rigging/constraints/tracking/spline_ik.html).
- Parenting meshes to a bone (rather than skinning) works. Select the object, shift-select the bone in pose mode and press `ctrl + p` > `to bone`. Do not use the "child of" constraint, this won't export.

### Shape keys / Morph Targets
Shape keys should convert to glTF "morph targets". However, if you're using modifiers such as "mirror" or "subdivision surface", you will not be able to apply these once you've created shape keys. This is a limitation in Blender, you can't apply modifiers to objects with shape keys. This is problematic, because this will need to happen on export (via an option). Note that armature modifiers seem to export fine without any extra effort. You have a few options here:
- Try the [Apply modifier to object with shape keys](https://github.com/przemir/ApplyModifierForObjectWithShapeKeys) addon
- Apply all your modifiers before adding shape keys. This is the simplest option, but is somewhat destructive.
- Keep your modifiers and do a [manual workaround](https://blender.stackexchange.com/questions/56795/shape-keys-and-applying-subdivision-surface-modifier) to apply modifiers. This could be quite time consuming.

### Exporting animations 
If you want separate animations, you'll need to save them as "actions" in Blender and then make sure they are in the NLA editor. 

- Use the dope sheet / action editor to make sure all of your animations are saved as individual actions (it's also good idea to name them something simple)
- Make sure the actions are "pushed down" into the NLA editor (this can be done from the action editor)
- If you want to check which actions will be exported, see if they're in the NLA editor. They should have their own strip (not just "stashed" with dotted outline)
- "Group by NLA Track" should be checked in the export settings

### A note on mixing actions once in three.js
Because of the way actions are exported, blending animations doesn't work too well for anything other than transitions. Let's say you have a figure running, and also a figure standing and waving. In an ideal world, you'd be able to blend these two in your application and have a figure running and waving. Unfortunately, instead you'll have a figure halfway between the two states (e.g. walking and half waving). This is because Blender exports all static information as well as important stuff.

This even applies to animations with multiple meshes. For instance, if you had a character with a hat as a seperate object and you wanted a separate hat wobble animation, you won't be able to blend that animation with something else. One option here is export each mesh seperately and combine them in three.js.

This also applies to shape keys in animations. While they blend perfectly well as separate morph targets, when shape keys are part of an animation they blend in a similar way as explained above.

#### This mixing problem can be fixed with code!

Thanks to a [code snippet from CasualKyle](https://discourse.threejs.org/t/creating-a-dynamic-run-walk-animation/442/23) on the three.js forum, it is possible to strip out redundant animation data from each clip, so they blend properly (e.g running and waving). Unfortunately this won't work for two animations blending morph target information, but does work for bones and other objects.

```javascript
const loader = new THREE.GLTFLoader()

loader.load(url, (gltf) => {
  const scene = gltf.scene || gltf.scenes[ 0 ]
  const clips = gltf.animations || []

  const getInc = (trackName, scene) => {
    let inc

    const nameParts = trackName.split('.')
    const name = nameParts[ 0 ]
    const type = nameParts[ 1 ]

    switch (type) {
      case 'morphTargetInfluences':
        const mesh = scene.getObjectByName(name)
        inc = mesh.morphTargetInfluences.length
        break

      case 'quaternion':
        inc = 4
        break

      default:
        inc = 3
    }

    return inc
  }

  clips.forEach(function (clip) {
    for (let t = clip.tracks.length - 1; t >= 0; t--) {
      const track = clip.tracks[ t ]
      let isStatic = true

      const inc = getInc(track.name, scene)

      for (let i = 0; i < track.values.length - inc; i += inc) {
        for (let j = 0; j < inc; j++) {
          if (Math.abs(track.values[ i + j ] - track.values[ i + j + inc ]) > 0.000001) {
            isStatic = false
            break
          }
        }

        if (!isStatic) { break }
      }

      if (isStatic) {
        clip.tracks.splice(t, 1)
      }
    }
  })
})
```

_Please note this code hasn't been well tested._

## Important export options
### Geometry
- Apply Modifiers
- Images: JPEG

### Animation
- Group by NLA Track

## Troubleshooting

### Parts of my mesh are missing when viewing in the browser. glTF error message "Accessor element at index 0 is NaN or Infinity."
This may be because you've added new parts to a mesh that is not associated with your armature. Try weight painting some of this mesh, or use the "weights > assign autmoatic from bones" in weight painting mode, with bone(s) selected.

### My object/bone position/rotation/scale is always exporting as 0
For some reason, if you don't have any values changing for an object from frame to frame, it will be set to 0. A simple fix for this is to make sure that there is some non-zero value change for the position/rotation/scale in your animation. Not sure why this is happening or at what point during the export process.

### My armature is scaled differently from the mesh. If I try to apply scale to the armature, it breaks the animation
This is an issue when importing from Mixamo. Below is a Blender python script that might help. Select your armature and then run the script.

```python
import bpy
for a in bpy.data.actions:
    for fc in a.fcurves:
        if "location" in fc.data_path:
            for kp in fc.keyframe_points:
                kp.co.y *= 0.01

ob = bpy.context.object
for pb in ob.pose.bones:
    pb.location = (0, 0, 0)

bpy.ops.object.transform_apply(scale=True)
```

The above script was modified from a [stack exchange answe](
https://blender.stackexchange.com/questions/143196/apply-scale-to-armature-works-on-rest-position-but-breaks-poses). More info there!

## Useful links
- [All exporters/converters and their features](https://github.com/KhronosGroup/glTF/issues/1271)
- [Creating Animated glTF Characters with Mixamo and Blender](https://www.donmccurdy.com/2017/11/06/creating-animated-gltf-characters-with-mixamo-and-blender/)
