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

pub type ColorFormat = gfx::format::Srgba8;
pub type DepthFormat = gfx::format::DepthStencil;

const BLACK: [f32; 4] = [0.0, 0.0, 0.0, 1.0];

pub fn main() {
    let events_loop = glutin::EventsLoop::new();
    let builder = glutin::WindowBuilder::new()
        .with_title("Square Toy".to_string())
        .with_dimensions(800, 800)
        .with_vsync();
    let (window, mut device, mut factory, mut main_color, mut main_depth) =
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

![AbstractSingletonProxyFactoryBean](https://i.imgur.com/Dgj7PX8.jpg){title="A typical programmer experience. Fortunately, Gfx-rs is not like that."}

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

Since `main_color` is moved into `data`, we need to replace `&main_color` with `&data.out` everywhere and remove `mut` keyword for `main_color` in  the `let` statement. And then, in the event loop, we draw:

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

const INDICES: &[u16] = &[0, 1, 2, 2, 3, 0];
```

And use them:

```rust
let (vertex_buffer, slice) =
    factory.create_vertex_buffer_with_slice(SQUARE, INDICES);
```

Compile the program and run.

![](https://i.imgur.com/5JfHmm6.png){width=600px height=600px}

Finally, a square. The most basic thing is done, now we can do further.

## Going deeper

The first thing you should notice is that the square we drew is actually a rectangle: when you resize the window proportions change. That's because OpenGL uses *normalized* coordinates where both x and y are from –1 to 1. So we need to adjust the square vertices to the window ratio.

The second thing is: our vertices and indices are pre-defined and constant. We can't adjust them, we can't make new squares on the fly.

Let's fix both of these issues. Define a vertice generator:

```rust
#[derive(Debug, Clone, Copy)]
struct Square {
    pub pos: (f32, f32),
    pub size: f32,
    pub color: [f32; 3]
}

// A cube is a pile of infinitely (as continuum) many squares
// This data stucture is finite, so we call it “pseudo”
#[derive(Debug)]
struct Pseudocube {
    squares: Vec<Square>,
    ratio: f32,
}

impl Pseudocube {
    pub fn new() -> Self {
        Pseudocube {
            squares: vec![],
            ratio: 1.0,
        }
    }

    pub fn add_square(&mut self, x: f32, y: f32, size: f32, color: [f32; 3]) {
        let sq = Square {
            pos: (x, y),
            size, color
        };
        self.squares.push(sq);
    }

    pub fn get_vertices_indices(&self) -> (Vec<Vertex>, Vec<u16>) {
        let (mut vs, mut is) = (vec![], vec![]);
        for (i, sq) in self.squares.iter().enumerate() {
            let (pos, half) = (sq.pos, 0.5 * sq.size);
            let i = i as u16;

            let (hx, hy);
            if self.ratio > 1.0 {
                hx = half / self.ratio;
                hy = half;
            }
            else {
                hx = half;
                hy = half * self.ratio;
            }

            vs.extend(&[
                Vertex { pos: [pos.0 + hx, pos.1 - hy], color: sq.color },
                Vertex { pos: [pos.0 - hx, pos.1 - hy], color: sq.color },
                Vertex { pos: [pos.0 - hx, pos.1 + hy], color: sq.color },
                Vertex { pos: [pos.0 + hx, pos.1 + hy], color: sq.color },
            ]);
            is.extend(&[
                4*i, 4*i + 1, 4*i + 2, 4*i + 2, 4*i + 3, 4*i
            ]);
        }

        (vs, is)
    }

    pub fn update_ratio(&mut self, ratio: f32) {
        self.ratio = ratio
    }
}
```

And use it:

```rust
pub fn main() {
    let mut cube = Pseudocube::new();
    cube.add_square(0.0, 0.0, 1.0, WHITE);
    // ...
    let (vertices, indices) = cube.get_vertices_indices();
    let (vertex_buffer, mut slice) =
        factory.create_vertex_buffer_with_slice(&vertices, &*indices);
    // ...
    let mut running = true;
    let mut needs_update = false;
    while running {
        if needs_update {
            let (vs, is) = cube.get_vertices_indices();
            let (vbuf, sl) = factory.create_vertex_buffer_with_slice(&vs, &*is);

            data.vbuf = vbuf;
            slice = sl;

            needs_update = false
        }
        // ...
                Resized(w, h) => {
                    gfx_glutin::update_views(&window, &mut data.out, &mut main_depth);
                    cube.update_ratio(w as f32 / h as f32);
                    needs_update = true
                },
        // ...
    }
}
```

Great. Now our squares are always squares. Time to add some cursor:

```rust
#[derive(Debug, Clone, Copy)]
enum Cursor {
    Plain((f32, f32), [f32; 3]),
    Growing((f32, f32), f32, [f32; 3])
}

impl Cursor {
    fn to_square(self) -> Square {
        match self {
            Cursor::Plain(xy, color) => Square { pos: xy, size: 0.05, color },
            Cursor::Growing(xy, size, color) => Square { pos: xy, size, color },
        }
    }
}

// ...

impl Pseudocube {
// ...
    pub fn update_cursor_position(&mut self, x: f32, y: f32) {
        let x = 2.0*x - 1.0;
        let y = -2.0*y + 1.0;
        let cursor = match self.cursor {
            Cursor::Plain(_, color) => Cursor::Plain((x, y), color),
            Cursor::Growing(_, size, color) => Cursor::Growing((x, y), size, color),
        };
        self.cursor = cursor;
    }
}
// ...
    let mut running = true;
    let mut needs_update = false;
    let mut window_size = (800.0, 800.0);
    while running {
// ...
                Resized(w, h) => {
                    gfx_glutin::update_views(&window, &mut data.out, &mut main_depth);
                    cube.update_ratio(w as f32 / h as f32);
                    window_size = (w as f32, h as f32);
                    needs_update = true
                },
                MouseMoved(x, y) => {
                    cube.update_cursor_position(
                        x as f32 / window_size.0,
                        y as f32 / window_size.1
                    );
                    needs_update = true
                },
```


It's alive. **IT'S ALIVE!** Yeah, things always becomes more cool when you add a little bit of interactivity.

Let's grow squares:

```toml
[dependencies]
rand = "*"
```

```rust
impl Pseudocube {
// ...
    pub fn start_growing(&mut self) {
        if let Cursor::Plain(xy, color) = self.cursor {
            self.cursor = Cursor::Growing(xy, 0.05, color)
        }
    }

    pub fn stop_growing(&mut self) {
        if let Cursor::Growing(xy, size, color) = self.cursor {
            self.squares.push (Cursor::Growing(xy, size, color).to_square());
            self.cursor = Cursor::Plain(xy, rand::random())
        }
    }

    pub fn tick(&mut self) {
        if let Cursor::Growing(xy, size, color) = self.cursor {
            self.cursor = Cursor::Growing(xy, size + 0.01, color)
        }
    }
}
// ...
                MouseInput(ElementState::Pressed, MouseButton::Left) =>
                    cube.start_growing(),
                MouseInput(ElementState::Released, MouseButton::Left) =>
                    cube.stop_growing(),
                _ => (),
            }

            cube.tick();
```

And squares were grown:

![](https://i.imgur.com/rumV7tU.png){width=600px height=600px title="This is not a modern art."}

## Textures and uniforms

So, you can draw squares, you can move cursor around, what else do you need? Oh, I see. Graphics. Plain colors are boring, right? Let's add some [texturing](https://learnopengl.com/#!Getting-started/Textures):

```rust
gfx_defines! {
    vertex Vertex {
        pos: [f32; 2] = "a_Pos",
        uv: [f32; 2] = "a_Uv",
        color: [f32; 3] = "a_Color",
    }

    pipeline pipe {
        vbuf: gfx::VertexBuffer<Vertex> = (),
        awesome: gfx::TextureSampler<[f32; 4]> = "t_Awesome",
        out: gfx::RenderTarget<ColorFormat> = "Target0",
    }
}
```

There are two changes here. The first is: vertices have got a new data: `a_Uv`. And if you think this can mean only one thing, you're right: yes, GPU doesn't know how to *exactly* draw textures. And yes, we use a fragment shader to determinate the behavior. `a_Uv` are coordinates of a texture fragment.

The second change introduces `t_Awesome` texture in the pipeline. The texture is the same for all triangles drawn with this pipeline. But what if you want different squares to look different? Well, there're three ways. The first way is to switch textures for each square. This ways is slow because it requires a draw call for each square, you can't draw everything with one call. The second way is to put everything into one big texture (a texture atlas) and use uv coordinates to get a texture from the atlas. The third way is to use a texture array (if it's supported).

We'll use neither of these ways, so our squares will have the same simple texture:

![](https://i.imgur.com/40VzkBZ.jpg){title=":awesome:"}

So let's texture our squares. To do it, we need a crate to load images:

```toml
[dependencies]
image = "*"
```

And we need to modify our shaders a little bit:

```glsl
#version 150 core

in vec2 a_Pos;
in vec2 a_Uv;
in vec3 a_Color;
out vec4 v_Color;
out vec2 v_Uv;

void main() {
    v_Color = vec4(a_Color, 1.0);
    v_Uv = a_Uv;
    gl_Position = vec4(a_Pos, 0.0, 1.0);
}
```

```glsl
#version 150 core

uniform sampler2D t_Awesome;

in vec4 v_Color;
in vec2 v_Uv;
out vec4 Target0;

void main() {
    vec3 aw = texture(t_Awesome, v_Uv).rgb;

    if(aw == vec3(0.0, 0.0, 0.0)) {
        Target0 = 0.20 * v_Color;
    } else {
        Target0 = vec4(aw, 1.0);
    }
}
```

And copypaste a function from [an another tutorial](https://wiki.alopex.li/LearningGfx):

```rust
fn load_texture<F, R>(factory: &mut F, path: &str) -> gfx::handle::ShaderResourceView<R, [f32; 4]>
    where F: gfx::Factory<R>, R: gfx::Resources
{
    let img = image::open(path).unwrap().to_rgba();
    let (width, height) = img.dimensions();
    let kind = gfx::texture::Kind::D2(width as u16, height as u16, gfx::texture::AaMode::Single);
    let (_, view) = factory.create_texture_immutable_u8::<ColorFormat>(kind, &[&img]).unwrap();
    view
}
```

Add uv coordinates to vertices:

```rust
Vertex { pos: [pos.0 + hx, pos.1 - hy], uv: [1.0, 0.0], color: sq.color },
Vertex { pos: [pos.0 - hx, pos.1 - hy], uv: [0.0, 0.0], color: sq.color },
Vertex { pos: [pos.0 - hx, pos.1 + hy], uv: [0.0, 1.0], color: sq.color },
Vertex { pos: [pos.0 + hx, pos.1 + hy], uv: [1.0, 1.0], color: sq.color },
```

And load the texture:

```rust
    let texture = load_texture(&mut factory, "assets/awesome.png");
    let sampler = factory.create_sampler_linear();

    let mut data = pipe::Data {
        vbuf: vertex_buffer,
        awesome: (texture, sampler),
        out: main_color
    };
```

Ta-da:

![](https://i.imgur.com/jeKLvoc.png){width=600px height=600px}

Oh no. The black is still black and the image is upside down. Well, the first is the bug of the image itself (that's what you get for downloading JPEG from the Internet), but why it's upside down?

Well, the reason is simple. Image coordinates have y-axis up-down, while in OpenGL y axis is always down-up. So the most obvious solution is to flip the image. But there's a more simple way: we can flip uv coordinates instead.

```rust
Vertex { pos: [pos.0 + hx, pos.1 - hy], uv: [1.0, 1.0], color: sq.color },
Vertex { pos: [pos.0 - hx, pos.1 - hy], uv: [0.0, 1.0], color: sq.color },
Vertex { pos: [pos.0 - hx, pos.1 + hy], uv: [0.0, 0.0], color: sq.color },
Vertex { pos: [pos.0 + hx, pos.1 + hy], uv: [1.0, 0.0], color: sq.color },
```

And then...

![](https://i.imgur.com/8li9Csm.png){width=600px height=600px title="Really :awesome:"}

Great! But what if you actually like plain colors more? We need a switch. We need a uniform.

Uniforms are global constants of shaders. They are used to pass various information into shaders: transformation matices, mouse position or some kind of switch. There're two ways to create a uniform in gfx-rs, the first is to just declare a single value in the pipeline like this:

```rust
switch: gfx::Global<i32> = "i_Switch",
```

The second way is to define a group of constants like

```rust
constant Globals {
    mx_vp: [[f32; 4]; 4] = "u_ViewProj",
    num_lights: u32 = "u_NumLights",
}
```

and then create a constant buffer. We'll use the first way because it's the simplest one.

So let's change the shader a little bit:

```glsl
    if(i_Switch == 0) {
        if(aw == vec3(0.0, 0.0, 0.0)) {
            Target0 = 0.20 * v_Color;
        } else {
            Target0 = vec4(aw, 1.0);
        }
    } else {
        Target0 = v_Color;
    }
```

And add some code:

```rust
let mut data = pipe::Data {
    vbuf: vertex_buffer,
    awesome: (texture, sampler),
    switch: 0,
    out: main_color
};
```

```rust
    KeyboardInput(ElementState::Pressed, _, Some(VirtualKeyCode::Space), _) =>
        if data.switch == 0 {
            data.switch = 1
        } else {
            data.switch = 0
        },
```

And we are done.

## Conclusion

Programming is the art of data transformation. Graphics programming is a great example of this statement: GPU does no magic by itself. You have to feed it with vertex data and define the transformations (shaders) so it could transform this data, in a simple and direct way, into another kind of data: array of pixels to be shown on the screen.

Gfx-rs is a great library helping you with that. It provides a simple but clear and *rustic* way to interact with GPU. Even though the docs looks scary because of lack of enough documentation, the API itself is pretty straightforward and easy to use.

There's too few articels about gfx-rs and almost no tutorials. I hope this litte tutorial will help other people to get into graphics programming on Rust and makes the learning less steep.

## Resources

There're two great resouces about OpenGL: [Learn OpenGL](https://learnopengl.com/) and [opengl-tutorial](http://opengl-tutorial.org/). They explain basic graphics programming principles in much detail, providing great examples and illustrations.

I also strongly advice you the [gfx-rs gitter](https://gitter.im/gfx-rs/gfx). It is not only welcoming, but also very helpful, writing this tutorial would be much harder without their help.

[The Book of Shaders](https://thebookofshaders.com/) is an awesome book about fragment sharder. It is not only about the art of shader programming, but also about shader as *the art*. Shaders are not only about graphics in AAA games, they are an artist's tool as well. This book with a lot of beatiful examples will introduce you into the world of [creative coding](https://github.com/terkelg/awesome-creative-coding).

## Source code

The source code of sqtoy is available [on Github](https://github.com/suhr/sqtoy). And the source code of this tutorial is [available](https://github.com/suhr/gsgt) on Github too. Pull requests are welcome, especially pull requests to the tutorial: I'm neither a good writer nor a good English speaker, so there's probably tons things to improve.
