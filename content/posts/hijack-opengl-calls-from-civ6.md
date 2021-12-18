---
title: "Hijack Opengl Calls From Civilization 6"
date: 2021-08-31T14:35:23+02:00
draft: true
---

Some people consume League of Legends on a regular basis. Others prefer spending time playing Counter Strike. Each people may have a specific addiction which he finds difficult to part with (and this addiction may not be a video game). During some part of my third year at uni (during covid) my addiction was clearly Civilization 6. I do not have to introduce it. A 4X game playable with friends during hours and hours. Available on Windows, Console and Linux, but unfortunately the Linux version is somewhat crappy... (Not to blame the developers, I'm sure they did their best with what they had). So one day, I decided to dive into the deep world of reverse engineering. Tadaaaaa!

## What was the goal?

The raw computing part of Civilization 6 is directly linked to the artificial intelligence, something I wasn't interested in, to be honest. Moreover, it wasn't the main source of performance problems (according to absolutely no benchmark but my own intuition). So I decided to begin writing a little OpenGL library that would intercept all OpenGL calls from glX context creation, glX swapbuffers, ... to gl compile shader, gl create buffer, ... With some linker flags, it's trivial. I began by writing a rust library exporting all glX functions.

```rust
#[no_mangle]
pub extern "C" fn glXGetClientString(
    display: *const c_void,
    name: c_int
) -> *const c_char { /* */ }

#[no_mangle]
pub extern "C" fn glXMakeCurrent(
    display: *const c_void,
    drawable: *const c_void,
    context: *const c_void,
) -> bool { /* */ }

// blah blah
```

Read https://github.com/vinhig/probriquegl/blob/master/src/glx.rs for reference. When you want to create an OpenGL context (think about a link between your application and the driver), you use a bunch of glX calls to specify the desired version, the linked window, when you want to swap buffers, etc. The first step of my little library was to allow my game to run on my ooold computer: Intel HD 3000 powered *beast*. Sadly, it's an OpenGL 3.3 max GPU. Obviously, it doesn't normally run civ6. Normally? Remember, I have the control over the glX context [glX context creation procedure](https://github.com/vinhig/probriquegl/blob/90dc4289262f9c24f06f12b9240d0337444ba6d2/src/glx.rs#L402)... So why not changing some attributes value? Wow! It works!

![Civilization 6 on Intel HD 3000](/images/civ6-on-intel-hd-3000.png)

Can we do more things? By modifying  the [glX get proc address](https://github.com/vinhig/probriquegl/blob/90dc4289262f9c24f06f12b9240d0337444ba6d2/src/glx.rs#L374) function, we can redirect all OpenGL calls. Before OpenGL 4.6 and the advent of SPIR-V, all shaders are give to the driver in text format. That means it can be easily read and *modified*. After some days, I managed to create a basic shader reloader `probriquegl`:

1. A the start of the game, each shader passed to [glShaderSource](https://github.com/vinhig/probriquegl/blob/90dc4289262f9c24f06f12b9240d0337444ba6d2/src/gl.rs#L75) is identified by a unique ID and saved on disk.
2. A interactive terminal appears, allowing the user to [reload a shader by its identifier](https://github.com/vinhig/probriquegl/blob/90dc4289262f9c24f06f12b9240d0337444ba6d2/src/lib.rs#L213).


## What's not working?

When you choose to reload a shader after applying modifications, you sometimes choose to not use some uniforms. That's ok, I won't be mad... but the driver, **yes**. Previously retrieved uniform locations requested by the game are all invalid now :/ It would require a lot of work, but a possible solution would be:

1. For each compiled program, create a table of all uniforms (register each time the game requests its location). Each entry contains the name, the original uniform location and an optional alternative uniform location.
2. For each shader recompilation, modify table of all uniforms for this program:
    1. If a uniform disappear (optimized out, for example), register is as *unused*.
    2. If a uniform disappear, request all uniforms location by their name and update the alternative uniform location.
3. Each time the game is using a uniform location, optionally replace it by the alternative uniform location for the currently bound program. If the uniform disappeared, do not accept the call.

A solution requiring some level of indirection for each shader-related call.

## Conclusion

So the building blocks of a bigger project are ready. To be honest, before stopping working on it, I thought it would be a great idea to create a "minified" version of Civilization 6 playable on your little laptop without having it consuming your whole battery. Modifying meshes, textures would require detecting the pattern of resource creation to correctly identify which object is currently being uploaded. A though work I'm not even sure someone would actually use.
