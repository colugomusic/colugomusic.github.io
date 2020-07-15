---
layout: post
title:  "Accessing autoloads in GDNative"
---
Here’s an example of how you do it:

```c++
class Sample : public godot::Node
{
    GODOT_CLASS(Sample, godot::Node);
     
    SampleManager* sample_manager_;
 
    // ...
 
public:
 
    void _ready()
    {
        sample_manager_ =
            godot::Object::cast_to<SampleManager>(
                get_tree()->get_root()->get_node("SampleManager"));
    }
     
    // ...
 
};
```

I don’t think there’s any way around this kind of piss when dealing with GDScript/C++ interoperability. Personally I encapsulate it all into a namespace called “mess”:

#### **`mess.cpp`**
```c++
namespace mess {
namespace get {
 
Node* autoload(Node* node, NodePath path)
{
    return node->get_tree()->get_root()->get_node(path);
}
 
SampleManager* sample_manager(godot::Node* node)
{
    return godot::Object::cast_to<SampleManager>(
        mess::get::autoload(node, "SampleManager"));
}
 
// ...
 
}}
```
#### **`sample.cpp`**
```c++
void Sample::_ready()
{
    sample_manager_ = mess::get::sample_manager(this);
}
```