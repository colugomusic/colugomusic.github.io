---
layout: post
title:  "Passing loads of data to a shader"
---
![waveform](/assets/waveform_pitch_editing.png)

Godot's shading language is kind of like a stripped down version of GLSL, and at the time of writing there's no easy way to send large amounts of dynamically sized data to the GPU. You can manage it by packing it all into a `sampler2D` though.

```glsl
shader_type canvas_item;

uniform sampler2D data;
uniform int data_width;

float get_value(int idx)
{
    int x = idx % data_width;
    int y = idx / data_width;

    float u8 = texelFetch(data, ivec2(x, y), 0).r;
	
    return ((u8 * 2.0) - 1.0);
}

// ...
```
I'm going to upload a bunch of floating point values from -1.0 to 1.0. The values will be encoded as unsigned bytes (0-255) and stuffed into the red channel of a `godot::Image::FORMAT_R8` data texture. Obviously the best data format and encoding to use will depend on your project.

The `get_value` function figures out which row and column of the texture contains the indexed value and decodes it back to a float in the range -1.0 to 1.0.

Here's the C++. You can do the same thing with GDScript if you really want. The basic idea will be the same.

```c++
using namespace godot;

// ...

Ref<ShaderMaterial> material;
PoolByteArray buffer;

int buffer_size;
int data_width;
int data_height;

// ... initialize stuff ...

// ... calculate required buffer size ...

if (required_buffer_size > 0 && required_buffer_size > buffer_size)
{
    buffer_size = required_buffer_size;

    // here i'm calculating the dimensions of a texture large enough to
    // hold all the data. i think this is a better idea than just creating
    // a texture with one really long row or column since GPUs have
    // built-in limits on the width and height of textures
    //
    // a buffer size of 5, for example, will generate a texture like this:
    // ┏━━━┳━━━┓
    // ┃ 0 ┃ 1 ┃ the shader's get_value function will use the data_width
    // ┣━━━╋━━━┫ uniform to calculate which row and column of the texture
    // ┃ 2 ┃ 3 ┃ to read from for the given buffer index.
    // ┣━━━╋━━━┫
    // ┃ 4 ┃   ┃
    // ┗━━━┻━━━┛
    data_width = int(::floor(::sqrt(buffer_size)));
    data_height = int(::ceil(float(buffer_size) / data_width));

    material->set_shader_param("data_width", data_width);
    buffer.resize(data_width * data_height);
}

// fill buffer with data
for (int i = 0; i < buffer_size; i++)
{
    const float value = calculate_value(i);
    const uint8_t encoded = (uint8_t)(((value + 1.0f) / 2.0f) * 255.0f);

    buffer.set(i, encoded);
}

if (upload_needed)
{
    // upload the data
    auto render_data_image = Ref<Image>(Image::_new());
    auto render_data_texture = Ref<ImageTexture>(ImageTexture::_new());

    render_data_image->create_from_data(
        data_width,
        data_height,
        false,
        Image::FORMAT_R8,
        buffer);
    
    render_data_texture->create_from_image(render_data_image, 0);

    material->set_shader_param("data", render_data_texture);
}
```