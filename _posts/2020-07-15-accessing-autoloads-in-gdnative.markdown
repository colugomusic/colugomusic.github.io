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
 
Node* autoload(godot::Node* node, godot::NodePath path)
{
    return node->get_tree()->get_root()->get_node(path);
}
 
SampleManager* sample_manager(godot::Node* node)
{
    return godot::Object::cast_to<SampleManager>(autoload(node, "SampleManager"));
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

You can c-plus-plussify this further:

#### **`mess.h`**
```c++
namespace mess {
namespace get {

template <class T>
T* autoload(godot::Node* node, const godot::NodePath path)
{
	return godot::Object::cast_to<T>(autoload(node, path));
}

extern SampleManager* sample_manager(godot::Node* node);

// ...

}}
```
#### **`mess.cpp`**
```c++
namespace mess {
namespace get {

SampleManager* sample_manager(godot::Node* node)
{
    return autoload<SampleManager>(node, "SampleManager");
}

// ...

}}
```

Having the `path` argument is important because the autoload name might not necessarily be the same as the type name. Often it will be though so you can avoid the potential typo by utilitizing the static `___get_type_name` method which is defined by the `GODOT_CLASS` macro.

#### **`mess.h`**
```c++
namespace mess {
namespace get {

template <class T>
T* autoload(godot::Node* node, const godot::NodePath path)
{
	return godot::Object::cast_to<T>(autoload(node, path));
}

template <class T>
T* autoload(godot::Node* node)
{
	return godot::Object::cast_to<T>(autoload(node, T::___get_type_name()));
}

extern SampleManager* sample_manager(godot::Node* node);

// ...

}}
```
#### **`mess.cpp`**
```c++
namespace mess {
namespace get {

SampleManager* sample_manager(godot::Node* node)
{
	return autoload<SampleManager>(node);
}

// ...

}}
```