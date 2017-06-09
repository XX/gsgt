Title: Graphics by Squares: a Gfx-rs Tutorial

I'm making a little toy to understand gfx-rs better. But as a side product, I also write this little tutorial to help other people to learn gfx-rs too.

## Getting started

Let's write something that compiles and runs.

```
$ cargo init sqtoy
```

Add this to `Cargo.toml`:

```toml
[dependencies]
gfx = "0.16"
gfx_window_glutin = "0.16"
glutin = "0.8"
```

And put this into `main.rs`:

```rust
#[macro_use] extern crate gfx;

extern crate gfx_window_glutin;
extern crate glutin;

use gfx::traits::FactoryExt;
use gfx::Device;
use gfx_window_glutin as gfx_glutin;

pub type ColorFormat = gfx::format::Rgba8;
pub type DepthFormat = gfx::format::DepthStencil;

const BLACK: [f32; 4] = [0.0, 0.0, 0.0, 1.0];

pub fn main() {
    let events_loop = glutin::EventsLoop::new();
    let builder = glutin::WindowBuilder::new()
        .with_title("Square Toy".to_string())
        .with_dimensions(800, 800)
        .with_vsync();
    let (window, mut device, mut factory, main_color, mut main_depth) =
        gfx_glutin::init::<ColorFormat, DepthFormat>(builder, &events_loop);

    let mut encoder: gfx::Encoder<_, _> = factory.create_command_buffer().into();

    let mut running = true;
    while running {
        events_loop.poll_events(|glutin::Event::WindowEvent{window_id: _, event}| {
            use glutin::WindowEvent::*;
            match event {
                KeyboardInput(_, _, Some(glutin::VirtualKeyCode::Escape), _)
                | Closed => running = false,
                Resized(_, _) => {
                    gfx_glutin::update_views(&window, &mut main_color, &mut main_depth);
                },
                _ => (),
            }
        });

        encoder.clear(&main_color, BLACK);
        encoder.flush(&mut device);
        window.swap_buffers().unwrap();
        device.cleanup();
    }
}
```

As you can see, we use gfx with glutin and OpenGL. Shortly what the code does:

1. Creates an event loop and prepares to create a window with title “Square Toy”
2. Runs `gfx_window_glutin::init()` to get `glutin::Window`, `gfx_device_gl::Device` and a bunch or other things
3. Uses the `factory` to create an `Encoder` that allows you to avoid calling raw OpenGL procedures.
4. Each frame:
	1. Check whether it's the time to exit
	2. Fill the screen with the color you want (it's `BLACK`)
	3. Actually do it.
	4. Since our buffering is at least double, switch buffers
	5. Cleanup
	
Great! Whatever you use, it is always simple to draw a black screen full of nothing. Unfortunately, drawing something else is usually a little bit more complicated. In gfx-rs it requires a pipeline, vertices, shaders...

## Overview of gfx-rs architecture

![](https://i.imgur.com/Dgj7PX8.jpg)

Gfx-rs is a library that abstracts over four low-level graphics APIs: OpenGL (ordinary and ES), DirectX, Metal and Vulkan. Because of that, it cannot provide a direct API to do things. Neither it should though, as graphics APIs (especially older one like OpenGL) are extremely verbose, imperative and stateful. Also they are neither safe nor easy to use.

In gfx-rs, everything is built around three core types: `Factory`, `Encoder` and `Device`. The first is used to create things, the second is a buffer that stores graphics commands to be executed by the `Device`, and the `Device` translates commands into low-level API calls.

Also, like current-get API like DX12 and Vulcan but unlike OpenGL, the pipeline state is incapsulated in pipeline state objects (PSO). You can have a lot of PSOs and switch between them.  But to create a PSO, first you have to define a pipeline and specify vertex atributes and uniforms.

There's a [great post](https://gfx-rs.github.io/2016/09/14/programming-model.html) in Gfx-rs blog describing Gfx-rs architecture in much more detail.

## Drawing a square

We need a pipeline to draw anything on the screen.

```rust
gfx_defines! {
    vertex Vertex {
        pos: [f32; 2] = "a_Pos",
        color: [f32; 3] = "a_Color",
    }

    pipeline pipe {
        vbuf: gfx::VertexBuffer<Vertex> = (),
        out: gfx::RenderTarget<ColorFormat> = "Target0",
    }
}
```

In graphics programming, everything is made of triangles and triangles are defined by their vertices. Vertices can carry additional information beside the coordinates, ours have only 2D position `a_Pos` and color `a_Color`. The pipeline has only the vertex buffer and the render target, no textures, no transformations, nothing fancy.

The GPU doesn't know what *exactly* to do with the vertices and what color pixels should have. To define the behaviour *shaders* are used. There're two kinds of shaders vextex shaders and fragment shaders (let's ignore geometric shaders we don't use). Both are executed in parallel on the GPU. A vertex shader runs on each vertex and transfroms it in a some way. A fragment shader runs on each fragment (usually pixel) and determinates what the color the fragment will have.

Our vertex shader is very, very simple:

```glsl
// shaders/rect_150.glslv
#version 150 core

in vec2 a_Pos;
in vec3 a_Color;
out vec4 v_Color;

void main() {
    v_Color = vec4(a_Color, 1.0);
    gl_Position = vec4(a_Pos, 0.0, 1.0);
}
```

OpenGL uses `(x, y, z, w)` [homogeneous coordinates](http://www.tomdalling.com/blog/modern-opengl/explaining-homogenous-coordinates-and-projective-geometry/) and RGBA colors. The shader just translates `a_Pos` and `a_Color` into OpenGL position and color.

The fragment shader is even more simple:

```glsl
// shaders/rect_150.glslf
#version 150 core

in vec4 v_Color;
out vec4 Target0;

void main() {
    Target0 = v_Color;
}
```

It just sets the pixel color to `v_Color` value [interpolated](http://www.geeks3d.com/20130514/opengl-interpolation-qualifiers-glsl-tutorial/) from vertices `v_Color` values by the GPU.

Let's define our vertices:

```rust
const WHITE: [f32; 3] = [1.0, 1.0, 1.0];

const SQUARE: [Vertex; 3] = [
    Vertex { pos: [0.5, -0.5], color: WHITE },
    Vertex { pos: [-0.5, -0.5], color: WHITE },
    Vertex { pos: [-0.5, 0.5], color: WHITE }
];
```

And initalize everything we need for drawing:

```rust
let mut encoder: gfx::Encoder<_, _> = factory.create_command_buffer().into();
let pso = factory.create_pipeline_simple(
    include_bytes!(concat!(env!("CARGO_MANIFEST_DIR"), "/shaders/rect_150.glslv")),
    include_bytes!(concat!(env!("CARGO_MANIFEST_DIR"), "/shaders/rect_150.glslf")),
    pipe::new()
).unwrap();
let (vertex_buffer, slice) = factory.create_vertex_buffer_with_slice(&SQUARE, ());
let mut data = pipe::Data {
    vbuf: vertex_buffer,
    out: main_color
};
```

Since `main_color` is moved into `data`, we need to replace `&main_color` with `&data.out` everywhere. And then, in the event loop, we draw:

```rust
encoder.clear(&data.out, BLACK);
encoder.draw(&slice, &pso, &data);
encoder.flush(&mut device);
```

And the program was run.

![](https://i.imgur.com/7u3ol88.png){width=600px height=600px}

This is not a square. The reason why it is not a square is simple: it has three vertices, so it must be a triange. Also OpenGL doesn't know anything about squares, it can only draw triangles.

You could just add three more vertices to draw a square by two triangles. Like this:

```rust
const SQUARE: [Vertex; 6] = [
    Vertex { pos: [0.5, -0.5], color: WHITE },
    Vertex { pos: [-0.5, -0.5], color: WHITE },
    Vertex { pos: [-0.5, 0.5], color: WHITE },
    Vertex { pos: [-0.5, 0.5], color: WHITE },
    Vertex { pos: [0.5, 0.5], color: WHITE },
    Vertex { pos: [0.5, -0.5], color: WHITE },
];
```

But instead we can define just 4 vertices and reuse them with Element Buffer Objects.

![](http://www.opengl-tutorial.org/assets/images/tuto-9-vbo-indexing/indexing1.png)

So let's define vertices and indices:

```rust
const SQUARE: &[Vertex] = &[
    Vertex { pos: [0.5, -0.5], color: WHITE },
    Vertex { pos: [-0.5, -0.5], color: WHITE },
    Vertex { pos: [-0.5, 0.5], color: WHITE },
    Vertex { pos: [0.5, 0.5], color: WHITE },
];

const INDICIES: &[u16] = &[0, 1, 2, 2, 3, 0];
```

And use them:

```rust
let (vertex_buffer, slice) =
    factory.create_vertex_buffer_with_slice(SQUARE, INDICIES);
```

Compile the program and run.

![](https://i.imgur.com/5JfHmm6.png){width=600px height=600px}

Finally, a square.
