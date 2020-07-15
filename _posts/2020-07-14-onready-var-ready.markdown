---
layout: post
title:  "onready var ready = true"
---
Sometimes it’s necessary to check whether or not a node’s _ready() function has been called.

For example, if you connect a node’s resized() signal through the editor, the receiver method may be called before the node has been set up properly.

As far as I know, Godot’s standard library doesn’t provide any way to check this, but you can just add the boilerplate yourself:

```gdscript
var ready = false
 
func _ready():
    ready = true
 
func _on_ClipArea_resized():
    if not ready:
        return
     
    arrange_buttons()
```

This looks pretty stupid, but you can make it look slightly less stupid by utilizing GDScript’s onready keyword:

```gdscript
onready var ready = true
 
func _on_ClipArea_resized():
    if not ready:
        return
     
    arrange_buttons()
```

One thing to be aware of is that before _ready() is called, our ready variable isn’t a boolean yet, so it’s not false. It’s null. Even if you add a :bool type specifier. So make sure you’re not testing for the wrong thing.

```gdscript
onready var ready:bool = true
 
func _on_ClipArea_resized():
    # wrong
    if ready == false:
        return
 
    # good
    if ready != true:
        return
 
    # good
    if not ready:
        return
```