---
title: AnotherMe开发日志02
published: 2026-03-31
description: AnotherMe立项第89天
image: ./images/cover2.png
tags:
  - AnotherMe
  - Godot
category: ""
draft: false
lang: ""
---
这几天做的比较水，主要是更新了一下这个屎山场景转换器。
直接看源码吧：
```
extends CanvasLayer
var color_rect : ColorRect;
var scene_to_load : String;
var color_rect_tween : Tween;
var scene_loaded : bool = true;
signal on_scene_changed();
func _ready() -> void:
	layer = 9999;
	color_rect = ColorRect.new();
	color_rect.size = Vector2(1152,648);
	color_rect.color = Color(0,0,0,0);
	color_rect.z_index=4096;
	add_child(color_rect);
	visible = false;
func change_scene_to(scene_path : String,fade_in_time : float = 0.2,fade_out_time : float = 0.4) -> void:
	visible = true;
	print("INFO:正在切换场景到"+scene_path+"...");
	if color_rect_tween:
		color_rect_tween.kill();
	scene_to_load = scene_path;
	color_rect_tween = create_tween().set_trans(Tween.TRANS_SINE);
	print("INFO:切换场景：创建渐变动画...");
	color_rect_tween.tween_property(color_rect,"color:a",1,fade_in_time).connect("finished",_load_new_scene);
	print("INFO:成功切换场景到"+scene_path+"!");
	color_rect_tween.chain().tween_property(color_rect,"color:a",0.0,fade_out_time);
	await get_tree().create_timer(fade_in_time+fade_out_time).timeout;
	on_scene_changed.emit();
	visible = false;
func _load_new_scene() -> void:
	get_tree().root.remove_child(get_tree().current_scene);
	var scene = load(scene_to_load).instantiate();
	get_tree().root.add_child(scene);
	get_tree().current_scene = scene;
	
func scene_point(scene_path : String,to_pos : Vector2,fade_in_time : float = 0.2,fade_out_time : float = 0.4) -> void:
	visible = true;
	print("INFO:正在切换场景到"+scene_path+"...");
	scene_to_load = scene_path;
	scene_loaded = false;
	color_rect_tween = create_tween();
	print("INFO:切换场景：创建渐变动画...");
	color_rect_tween.tween_property(color_rect,"color:a",1,fade_in_time);
	color_rect_tween.chain().tween_property(color_rect,"color:a",0.0,fade_out_time);
	await get_tree().create_timer(fade_in_time).timeout;
	_load_new_scene()
	print("INFO:成功切换场景到"+scene_path+"!");
	Game.player.global_position = to_pos;
	await get_tree().create_timer(fade_out_time).timeout;
	color_rect_tween.kill();
	scene_loaded = true;
	on_scene_changed.emit();
	visible = false;

```
看出来啥了没
对，其实跟之前没太大区别。
主要是这段修改的经历比较有趣：因为场景切换后，如果玩家切换回来的话，之前场景的状态（比如剧情后隐藏的物体）都会显示出来，而且剧情也会重复触发一遍。
这是因为godot的场景切换会把要切换的场景初始化一遍，于是就没有之前的进度了。
于是我就想到用数组来存储每一个加载过的场景，在加载新的场景时，先：
```
# 伪代码，大概是这样的
loaded_scene[current_scene] = get_tree().current_scene;
```
但是这样一是费内存，大量场景被缓存难免会造成性能问题；二是这样会产生很多bug，比如经常会存储一个空节点，或者读取一个空节点之类的。
而且这样也没必要，我们只需要存储**剧情的进度**和**物体的状态**就可以了
于是我又加了一个自动加载：
```
extends Node
var bool_data : Dictionary;
var int_data : Dictionary;
var float_data : Dictionary;
var string_data : Dictionary;

```
专门来存各种场景数据，这样一来就方便多了。
至于对话进度，我直接用了一个变量存储当前最新的剧情进度（用剧情index表示），如果触发剧情的脚本检测到自己的剧情index比较旧就不会触发剧情。
自动加载Dialogue：
```
extends Node
var current_dialogue : int = -1;
# ...
func update_current_dialogue(dialogue : int):
	current_dialogue = dialogue;

```
触发剧情dialogue_trigger:
```
extends Area2D

@export var dialogue : DialogueResource;
@export var dialogue_index : int = -1;
@export var dialogue_title : String = "";
@export var press_key : bool = false;
@export var delay_time : float = 0.0;
@export var once : bool = false;
var player_entered : bool = false;
# Called when the node enters the scene tree for the first time.
func _ready() -> void:
	pass # Replace with function body.


# Called every frame. 'delta' is the elapsed time since the previous frame.
func _process(delta: float) -> void:
	# 检测场景是否已经加载完毕，玩家是否进入判定区域，是否要按键才能触发剧情，自己index是否较旧
	if Level.scene_loaded and player_entered and (press_key or Dialogue.current_dialogue <dialogue_index):
		if press_key: 
			if Input.is_action_pressed("enter_press"):
				start_dialogue();
		else:
			start_dialogue();
func _on_body_entered(body: Node2D) -> void:
	if body.is_in_group("player"):
		player_entered = true;
	
func _on_body_exited(body: Node2D) -> void:
	if body.is_in_group("player"):
		player_entered = false;
func start_dialogue():
	await get_tree().create_timer(delay_time).timeout;
	Dialogue.update_current_dialogue(dialogue_index);
	DialogueManager.show_example_dialogue_balloon(dialogue,dialogue_title);
	if once:
		queue_free();

```
这样就基本完成了，感觉这个代码有点屎，看以后优化吧。