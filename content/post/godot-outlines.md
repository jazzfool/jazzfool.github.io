---
title: "3D Outlines in Godot"
date: 2022-02-09T13:13:22+11:00
draft: false
---

_This guide was written for Godot 4.0 (Alpha 1 at the time of writing), but does not use any 4.0-specific features_

Outlines are a handy visual tool in games to show focus on objects. Unfortunately for Godot, I haven't seen much content on how to make nice looking outlines for 3D objects.

This guide will show you how to make outlines that look like this:

![Godot outlines screenshot](/static/godot-outline.png)

# Step 1: Isolating depth

Firstly, we need to isolate the depth for the objects we want to outline. To do this, we need to get our hands dirty with `SubViewport` (`Viewport` pre-4.0).

This is the scene tree you need to set up:

![Godot scene tree](/static/godot-outline-scene-tree.png)

_(Note: `OutlineViewport` and `GameViewport` are of type `SubViewport`, and `GameViewportContainer` is of type `SubViewportContainer`)_

The `Node3D` under `GameViewport` is where your entire game scene will go. Objects that we want to outline will be duplicated under `OutlineViewport/Node3D` as siblings of `Camera3D`. Finally, the script attached to `Node2D` will be managing everything.

Viewports do not have a built-in way to render depth-only, so the only way to do this currently is to put a depth-visualising material on all the objects in `OutlineViewport`.

Here is the shader code for that material:

```glsl
// depth_only.gdshader
shader_type spatial;
render_mode unshaded, skip_vertex_transform;

varying float depth;

void vertex() {
	VERTEX = (MODELVIEW_MATRIX * vec4(VERTEX, 1.0)).xyz;
	NORMAL = (MODELVIEW_MATRIX * vec4(VERTEX, 0.0)).xyz;

	vec4 ndc = PROJECTION_MATRIX * vec4(VERTEX, 1.0);
	ndc.xyz /= ndc.w;
	depth = ndc.z;
}

void fragment() {
	ALBEDO = vec3(depth);
}
```

Here is the script we'll attach to objects inside `OutlineViewport`:

```python
# OutlineCopy.gd
class_name OutlineCopy
extends MeshInstance3D

var original: Node3D = null;

const depth_only = preload("res://depth_only.gdshader")

func _ready():
	var material := ShaderMaterial.new()
	material.shader = depth_only
	material_override = material

func _process(delta):
	if is_instance_valid(original):
		global_transform = original.global_transform
```

The above will only replicate the global transform, but you may want to replicate other properties, for example the mesh if it changes at runtime.

Finally, this is the script we'll attach to the root `Node2D`:

```python
# Root.gd
extends Node2D

@onready var game_viewport_container: SubViewportContainer = $GameViewportContainer
@onready var game_viewport: SubViewport = $GameViewportContainer/GameViewport
@onready var game_camera: Camera3D = $GameViewportContainer/GameViewport/Node3D/Camera3D
@onready var outline_viewport: SubViewport = $OutlineViewport
@onready var outline_camera: Camera3D = $OutlineViewport/Node3D/Camera3D
@onready var outline_scene: Node3D = $OutlineViewport/Node3D

const OutlineCopy := preload("res://OutlineCopy.gd")

func _ready():
	game_viewport_container.material.set_shader_param("outline_viewport_texture", outline_viewport.get_texture())

    # call add_outline on the objects you want here, or anywhere really

func _process(delta):
    # synchronise viewport size, camera transform, and camera FOV

	var viewport := get_viewport()

	if game_viewport.size != viewport.size:
		game_viewport.size = viewport.size

	if outline_viewport.size != game_viewport.size:
		outline_viewport.size = game_viewport.size

	outline_camera.fov = game_camera.fov
	outline_camera.global_transform = game_camera.global_transform

func add_outline(node: MeshInstance3D):
	if is_instance_valid(node.get_meta("outline_object")):
		# object already has an outline
		return
	# 0 disables all duplicate flags - we only want to duplicate the visuals
	var copy = node.duplicate(0)
	copy.set_script(OutlineCopy)
	copy.original = node
	outline_scene.add_child(copy)
	node.set_meta("outline_object", copy)

func remove_outline(node: MeshInstance3D):
	var outline: MeshInstance3D = node.get_meta("outline_object")
	if is_instance_valid(outline):
		outline.queue_free()
```

You don't have to add objects to `OutlineViewport` through code as it can also be done manually, but just make sure the material is `depth_only.gdshader` and the script attached is `OutlineCopy.gd`.

We should now have a viewport that renders the depth buffer only for objects that we tell it to.

# Step 2: Outline

Now that we have a depth buffer isolated to objects that are outlined, we can just apply a post-process outline. We run dead-simple edge detection on said depth buffer, compare, and if the "edge value" falls above some threshold then we output a fixed color instead of the scene color.

Just add a shader material to `GameViewportContainer` with the following shader:

```glsl
// post_outline.gdshader
shader_type canvas_item;

uniform sampler2D outline_viewport_texture;
uniform vec4 outline_color : hint_color = vec4(1, 0, 0, 1);
uniform float outline_thickness = 2.0;

float tex(vec2 uv, vec2 ts, float x, float y) {
	return texture(outline_viewport_texture, uv + vec2(x, y) * ts).r;
}

float outline(vec2 uv, float thickness) {
	vec2 ts = thickness / vec2(textureSize(outline_viewport_texture, 0));
    // only 4 texture samples!
	return (abs(tex(uv, ts, -1, -1) - tex(uv, ts, 1, 1)) + abs(tex(uv, ts, 1, -1) - tex(uv, ts, -1, 1)));
}

void fragment() {
	vec3 color = texture(TEXTURE, UV).xyz;
	float outline = outline(UV, outline_thickness);
	COLOR = mix(vec4(color, 1), outline_color, float(outline > 0.05));
}
```

_Credit for this shader goes to http://blog.dalton.gd/quick-outline-ue4/ ._

That's it!

# Shortcomings

-   Wasteful on memory. Since `SubViewport` does not have a depth-only render mode (ideally this would be in the form of a depth buffer Debug Draw mode - I may open a PR for this in the future), there are basically two depth buffers, one of them in the format R8G8B8A8 (which is even more wasteful for a depth target!).
-   Wasteful on GPU usage. GPUs have hardware depth testing and writing. We _should be_ just utilising that in order to render our custom depth buffer and in fact it is, but only in a depth attachment we can't access, so we need to make our own depth-writing shader for no good reason. Again, this is just Godot's fault for not exposing the depth buffer of a viewport.
-   Large thickness values don't look great. This is because the edge detection only looks at the 4 adjacent texels to determine an outline and simply just scales the UV outwards to simulate thickness. "Real" thickness-based edge detection would need to look at `pi * thickness * thickness` texels.
