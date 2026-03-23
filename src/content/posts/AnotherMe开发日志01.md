---
title: AnotherMe开发日志01
published: 2026-03-23
description: AnotherMe立项第81天
image: ./images/AnotherMeC1.png
tags:
  - AnotherMe
  - Godot
category: '"Godot",AnotherMe"'
draft: false
lang: ""
---
klqn.top在两天前正式**复活**了，虽然上学很少有时间更新博客，但我还是尽量做到**一周五更**吧，至少写**AnotherMe**的时候更一次。
这个项目我在B站都没发，所以几乎没有人知道。关于这个独立游戏的世界观设定啥的我之后会搞一个Wiki出来（虽然我可能懒得搞（））
## 战斗系统的更改
首先今天在战斗系统里加了一个偷懒键，以后开发的时候就可以跳过战斗了（每次打一遍很拖时间）
这个东西一般开发都会加上的，而我已经都懒得偷懒了（）
```
if Input.is_action_just_pressed("GiveUp"):
		_over(1);
# 此处省略一些代码
func _over(state:bool):
	if state:	
		check_pass();
		var win : PackedScene = load("res://Scenes/Fight/fight_win_view.tscn");
		var win_scene = win.instantiate();
		Game.control(false);
		add_child(win_scene);
		create_tween().tween_property($"/root/FightScene/BGM","volume_db",-80,7);
	else:
		Level.change_scene_to("res://Scenes/Fight/fight_lose.tscn",0,0);
```
相信你也看出来了，`_over`方法的作用是**玩家失败或胜利时结束整个战斗**的方法。
	首先用一个`state`变量做判定，true就是胜利了，false就是失败了
		`check_pass()`做了一个简单的战斗评分系统，首先固定一个战斗时长，战斗超过这个时长的一定比例就会降低结算评分，最后会显示到`win_scene`中
			如果战斗失败，就会跳转到战斗失败场景
顺便贴一个跳转场景的脚本吧，可以参考一下，自带了黑屏渐入渐出的转场效果
```
extends CanvasLayer
var color_rect : ColorRect;
var scene_to_load : String;
var color_rect_tween : Tween;
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
	get_tree().call_deferred("change_scene_to_file",scene_to_load);

```
## BGMManager
由于每个场景都要创建一个AudioStreamPlayer2D节点很麻烦，而且DialogueManager不好调用，于是直接**自动加载**了一个脚本
```
extends AudioStreamPlayer2D


# Called when the node enters the scene tree for the first time.
func _ready() -> void:
	pass # Replace with function body.


# Called every frame. 'delta' is the elapsed time since the previous frame.
func _process(delta: float) -> void:
	pass
func play_bgm(path : String,fade_time : float = 0,volume : float = 1,pitch : float = 1):
	if stream == load(path):
		return;
	stream = load(path);
	create_tween().tween_property(self,"volume_db",volume,fade_time);
	self.pitch_scale = pitch;
	play();

```
    很简单，不用多说啥。
	说起来Godot做游戏真的比Unity轻松多了，节省了好多不必要的自己造轮子。
	以后想要在Dialogue里放BGM的时候直接：
	```
	do BgmManager.play_bgm("音频路径")
	```
	nice。
## 发现的一些小特性
由于转换到战斗场景的时候我还想保留原本主世界场景的状态，但是用Level的切换场景就直接被清除初始化了，在网上查了一圈后找到了这个方法：
```
origin_scene = get_tree().current_scene;
get_tree().root.remove_child(origin_scene);
# ...
# 此时要返回主世界
get_tree().root.add_child(origin_scene);
```
这样就能保存场景的状态了。（再夸一下Godot这个节点设计真的很牛逼，换成Unity可能就要更麻烦点)
但是我用原来的Level转换场景时，出问题了：
我一看远程，原来的主世界场景转换后居然还在！而且挡住了新场景。
因为Level转换的是current_scene，于是我打印了这个变量，结果是Null
这说明：通过add_child添加回主世界场景后，主世界并没有变成新的current_scene
要想解决这个问题也很简单：
```
get_tree().current_scene = origin_scene;
```
Solve。

那么这就是今天的工程了（说实话没做多少，主要是修bug和优化）
