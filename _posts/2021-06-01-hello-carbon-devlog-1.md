---
layout: post
title: Hello Carbon (Devlog &#35;1)
subtitle: An adventure into Vulkan
gh-repo: DraperDanMan/DraperDanMan.github.io
gh-badge: [star, follow]
tags: [vulkan, renderer, cpp]
---

Last November in my spare time over a holiday break I started **another** side project currently named Carbon. It started out as an interest in learning the rendering API Vulkan and as a way to refresh my C++ skills. *(I've been working in Unity with C# and very rarely needed C++ code for the last 5 or 6 years)*

Honestly, Until a little over two years ago, I hadn't really thought much about the low level rendering of game engines. I guess I never really found it relevant to what I was doing, after all Unity/Unreal Engines handled it all right? Well... That certainly changed when we needed to get Samurai Punk's [Feather](https://samuraipunk.com/feather) to render on a Nintendo Switch. Overcoming that was its own challenge and perhaps one day I'll write about how we overcame that hurdle. For now I'll focus on Carbon, for the sake of avoiding more detours than the TV hit LOST.

## The Goal

Carbon started out with the simple goal of rendering a cube, with perspective and some colour, maybe a little deferred rendering setup. The next goal would be learning how to do a shadow pass, then some cascades, and finally digging into some of the newer RTX features that Vulkan has extensions ready to go. However, after a very light digging online, people were prepping me for a tough learning curve ahead. The gist of all the reading I was *you have to set up everything, nothing is inferred*. They weren't kidding either, it gives you almost full control over the entire process and that verbosity is intimidating. However, I think the key here is that you are in **control**. Something that is often sorely missing in modern day to day programming.

By now you're probably wondering, "So, How far did you make it?" Well, check it out.

![A animated gif of some vertex coloured donuts flying around in a void.](/img/CarbonVertexColors.gif)

It isn't quite where I imagined, but I'll be damned if I didn't learn more than I bargained when I set out with this goal. What follows is a summary of what I did get done and the roadblocks I hit and had to overcome.

### Naïve Forward Rendering

I say Naïve because I don't sort my draw calls, so I have a bunch of overdraw. Once you have the ability to just draw a mesh its an easy jump to renderer multiple meshes. No instancing yet. I would like to tell you getting to this point is pretty straight forward *(pun certainly intended)*, however all the building blocks to actually renderer your first triangle is a fair chunk of work and learning how Vulkan needs things set up ready for the GPU.

### Importers for OBJ & FBX

The OBJ format was kind of a necessity to get a mesh rendering on screen short of manually writing all the mesh data by hand... The OBJ format was super nice to handle (I've also done it before), the file is in ascii and lists out, verts, indices, normal and texture uv's. You iterate through the lines of the file and fill a couple of arrays with all the data. Done. While I was initially working with meshes, this was all I needed, but the renderer doesn't have a system for handling Textures yet, so all the colours of the meshes were just their normal or UV data. Which looks pretty ugly when you're rendering more than one of them and lighting doesn't look soo good either. (you can see what the uv coloured mesh looks like below in the camera section)

While showing my progress to a work colleague, they pointed out that vertex colours is a quick way to get colour onto a mesh without having to use the normals or support a texture. Which is true, but I'm fully aware that the OBJ format doesn't support vertex colours... Hence, we now arrive at supporting the FBX. I was under the impression that sooner or later I would need to support the FBX format, due to it pretty much being the standard in many 3D industries today. The files are able to hold, mesh data, animation data, rigging data, material data as well as custom meta-data. A powerful format indeed. Luckily there are a few people who have already done the hard yards on making an open source FBX parser to import and FBX file into a dictionary of key,value pairs.  There were a couple of projects that I stumbled upon, but most were abandoned or not fully featured. The best one I found and was fairly straight forward to use was [OpenFBX](https://github.com/nem0/OpenFBX). With a little jiggling I was able to get meshes importing from both formats without too much pain. 

### A Basic Camera

It's possible to render an image without 'camera' to the screen, though it's renderer as is the camera is at the origin, and so will all your geometry. So unless you place all of your geometry away from the origin you often wont see it when it comes time to render the image. The solution is to set up the concept of a camera, which is actually just a 4x4 matrix. 

![A animated gif of a virtual camera moving around a uv coloured donut](/img/vulkanFixedCam.gif)

This stage alone also added up to be quite a bit of work. It required matrices, quaternions and vectors. All of which I didn't have yet. I considered using an already established math library. However, I wanted to test the concept of using [Rotors](https://en.wikipedia.org/wiki/Rotor_(mathematics)) instead of quaternions. So I started building my own little math library (for now). The idea is that rotors are meant to be more intuitive to understand than quaternions making development easier in future. I couldn't find a good implementation online so I mashed together some code based on [this youtube video](https://www.youtube.com/watch?v=Idlv83CxP-8) and my quaternion code that I had already set up. I'll share my findings another time. For now I'll leave with 'it didn't work out'.

### Runtime Shader Compilation

One of my favourite features from modern game engines and 3D rendering software is runtime shader compilation. When I first started I wrote a basic vertex and fragment shader to render the initial triangle and thoroughly disliked manually compiling it from the command line. So I had a poke around to see if anyone was compiling GLSL to SPIR-V (Vulkan only accepts spir-v bytecode) and people were doing it with some crazy setups including big shader compiling libraries. All of which looked like a big hassle. The Vulkan SDK already comes with Google's [shaderc](https://github.com/google/shaderc/) library. which has a nice convenient `glsl_to_spirv()` function. Though it does require a little bit of set up to use the pre-processor. So I spent an extra day setting up compiling the shaders every time the renderer boots up. This could be made smarter by calculating a hash of the shader file to see if it needs to be re-compiled rather than compiling it every time it boots up. But currently that is a non issue as everything uses the same few basic shaders for now and the compile time is a millisecond at most.

### The Boring Bits

While there is a few other small things that have gone into it. Nothing that deserves a paragraph of its own. I have added some basic file utilities for finding and reading files for the models and shaders. Some basic logging utilities so I can actually log the status of a function or when errors messages happen. Also some basic profiling macros to time how long certain functions take (loading models, compiling shaders or building command buffers). There was certainly a non-zero cost to setting up 3rd Party code, Currently, I'm using GLFW for the window creation and managing windows input and OpenFBX. Honestly, a lot of time has been spent fighting with Visual Studio to manage the C++ project in general. That's a rant for another day though. 

## Wrapping Up

Now, I'm fully aware this isn't the most well written blog you've ever read, so I appreciate the you read this far. I have no idea if this project is interesting. But I think this a good way to have a record of my progress, process and reference, because future me is sure to forget half the things I have written after a week. 

Not sure what the future holds, but I've been enjoying working on this project a great deal and I'm excited for all the places I can go from here. I guess just expect more LOST references.