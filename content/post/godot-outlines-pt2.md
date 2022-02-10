---
title: "3D Outlines in Godot, Part 2"
date: 2022-02-10T13:02:00+11:00
draft: false
---

My [previous post](https://jazzfool.github.io/post/godot-outlines/) explained how to implement 3D outlines in Godot using post-processing on top of a custom depth buffer. This works, but is an ugly solution which doesn't look good with large outline widths, and was locked to a single outline color and width globally. It also lacked anti-aliasing as it was a post-process pass presumably running after FXAA.

This new method solves these issues, and is also more performant.

![Godot outlines screenshot](/static/godot-outline-2.png)

## Step 1: Stencil

This new version is based on [QuickOutline](https://github.com/chrisnolet/QuickOutline), which is a hybrid method of screen-space masking and world-space vertex inflation. We first draw a second version of the mesh ignoring depth, inflate it to act as an outline, and mask out the inside of the mesh with a stencil buffer. This means that we can draw the outline differently for each object, and only need 1 texture sample to mask out the outline.

This solution is also far less invasive in the scene, and works with visual layers to cull out objects we don't want to stencil.

This is the top-level scene tree you need to set up:

![Godot scene tree](/static/godot-outline-2-scene-tree.png)

Where `Node3D` is your main scene.

The `Node2D` has the following script:

```python
# MainScene.gd
class_name MainScene
extends Node2D

@onready var game_camera: Camera3D = $Node3D/Camera3D
@onready var game_scene: GameScene = $Node3D
@onready var stencil_viewport: SubViewport = $StencilViewport
@onready var stencil_camera: Camera3D = $StencilViewport/Camera3D

func _process(delta):
	game_scene.main_scene = self

	var viewport := get_viewport()

	if stencil_viewport.size != viewport.size:
		stencil_viewport.size = viewport.size

	stencil_camera.fov = game_camera.fov
	stencil_camera.global_transform = game_camera.global_transform
```

This just syncs up the cameras and re-exports the viewport.

On the `StencilViewport`, set `Rendering > Debug Draw` to `Wireframe`. Basically, we need to render any non-zero color to the viewport, and wireframe will do that.

## Step 2: Outline

This version is also programatically a lot simpler. We can make it so that outlining a mesh simply involves adding/removing a child node. That child node can just be an empty `Node3D` scene with the following script attached to the root:

```python
# Outline.gd
extends Node3D

@export var width: float = 5.0
@export var color: Color = Color.RED

const shader = preload("res://outline.gdshader")

func _enter_tree():
	var mesh: MeshInstance3D = get_parent()
	mesh.set_layer_mask_value(2, true)

	var surf = SurfaceTool.new()
	surf.begin(Mesh.PRIMITIVE_TRIANGLES)
	var seen = {}
	var i = 0
	for face in mesh.mesh.get_faces():
		if !seen.has(face):
			var index = i
			i += 1
			seen[face] = index
			surf.add_vertex(face)
			surf.add_index(index)
		else:
			surf.add_index(seen[face])

	surf.generate_normals()

	var material = ShaderMaterial.new()
	material.shader = shader
	material.set_shader_param("stencil", get_tree().current_scene.stencil_viewport.get_texture())
	material.set_shader_param("width", width)
	material.set_shader_param("color", color)

	var outline := MeshInstance3D.new()
	outline.mesh = surf.commit()
	outline.material_override = material
	add_child(outline)

func _exit_tree():
	var mesh: MeshInstance3D = get_parent()
	mesh.set_layer_mask_value(2, false)
```

The main interesting bit here is the smooth normal generation. Since we're offsetting the faces by the normals, if they're disjoint (i.e. flat shaded), the outline will have gaps, so we just merge the vertices and output a new mesh with the outline material.

Now the outline shader itself:

```glsl
// outline.gdshader
shader_type spatial;
render_mode unshaded, cull_disabled, depth_draw_never, depth_test_disabled, skip_vertex_transform;

uniform sampler2D stencil;
uniform float width = 20.0;
uniform vec4 color : hint_color = vec4(0.0, 1.0, 0.0, 1.0);

varying vec2 uv_offset;

const float UV_BIAS = 0.00087;

void vertex() {
	// neutralize scaling from the transform matrix.
	// this ensures the normals are scaled uniformally, even for stretched meshes
	mat4 noscale = MODELVIEW_MATRIX;
	noscale[0] = normalize(noscale[0]);
	noscale[1] = normalize(noscale[1]);
	noscale[2] = normalize(noscale[2]);

	VERTEX = (MODELVIEW_MATRIX * vec4(VERTEX, 1.0)).xyz;
	NORMAL = normalize((noscale * vec4(NORMAL, 0.0)).xyz);
	vec4 a = vec4(VERTEX, 1.0);

	// inflate vertex by normal * width (adjusted for viewing distance)
	VERTEX += NORMAL * -VERTEX.z * width / 1000.0;
	vec4 b = vec4(VERTEX, 1.0);

	a *= PROJECTION_MATRIX;
	b *= PROJECTION_MATRIX;
	a /= a.w;
	b /= b.w;

	uv_offset = normalize(b.xy - a.xy);
}

void fragment() {
	ALBEDO = color.rgb;
	// stencil out the inside
	ALPHA = float(!(texture(stencil, SCREEN_UV + uv_offset * UV_BIAS).r > 0.0));
}
```

That's all. Adding an outline just means instancing the `Outline` scene as a child of whatever mesh you wish to outline. If you add this directly in the editor, you'll need to change `Outline.gd` so that it waits after `MainScene._ready` is called.

## Shortcomings

In terms of functionality, there's not much lacking. But once again, Godot's neglect towards advanced 3D rendering means that this solution is wasteful for no good reason. The viewport "stencil pass" is just a RGBA8 full color render pass with wireframe mode, where non-zero values are considered stenciled out. Depth testing is still taking place in this pass, for no reason whatsoever. As a result, both GPU memory and GPU time is being wasted.
