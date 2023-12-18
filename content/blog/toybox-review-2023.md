+++
title = "Toybox Development 2023 Recap"
date = 2023-12-18
+++

As 2023 comes to a close I've decided that I need to take some time to reflect on my progress developing the Toybox game engine. I always have a hard time keeping it in my brain that I've actually every gotten anything done. But 2023 was definitely a year in which I got many things done so join me as we take a peek at what I did on my personal game engine Toybox.

## January
I was still struggling with trying to get shadow maps to behave correctly and as a distraction I spent a bit of time cleaning up CI and the build system. I began dropping some of the more complex aspects of mobile support (a trend that would continue) and began shoring up things like asset cooking for sample projects. I also ditched the concept of EulerAngles from the engine entirely as they were more trouble than they were worth; Quaternions all the way baybeee!

## Febuary
This month began with a bunch of work to get the ocean system playing nicely with the `boat2` scene which has been my testbed for boat related features that I want for my sailing game concept. Other than that it was a fairly slow month mostly filled with little fixes and work to make the code base more consistent.

## March
This was a month where I started off strong by committing to the Vulkan 1.3 dynamic rendering API. This ultimately gates the project from any platform that targets Vulkan 1.1 or 1.2 but I believe this rendering API is far superior to the traditional `VkRenderPass` based API. For desktop platforms there are pretty much no downsides and I wasn't able to measure any major performance delta so I think it's a win. 

This effectively makes Android and MoltenVk (iOS, macOS) unsupported by the current renderer. I'm not super thrilled about this so I've committed to keep CI building for iOS, macOS and Android in the hopes that some future work will enable Vulkan 1.3 on these platforms. The roadmap for Toybox is still very long at this point so I think looking to the future is appropriate.

## April 
March also saw a lot of other smaller fixes and work in a lot of misc areas that really burnt me out so April was particularly slow. Not much in particular to report.

## May
One feature I really wanted to implement was Screen Space Ambient Occlusion (SSAO) and May was the month where I took a stab at that. This involved a bunch of scaffolding work to support Compute work in the rendering API so that the engine would be able to blur the SSAO target with a compute shader. It was also at this point that I gave up on a Reverse Depth Buffer for now; it made shadows in particular a bit hard to wrap my head around when debugging. It's just controlled by a define so I can (and probably will) revive it.

## June
As the scaffolding for Compute was done I decided to lean even further into Compute by adding Luminance Histogram based automatic exposure for the HDR tonemapping. HDR and auto exposure is a huge topic and I'm sure there's a million ways I'm doing it wrong but getting this working resulted in a much nicer looking image. This month also saw a lot of tweaks to my math library and just a lot of changes for rendering correctness.

## July
For some reason July was the month that I had to implement audio playback. I'm making agressive use of `SDL2` as it is so `SDL_mixer` was an obvious and easy dependency to bring in. This work will probably see a lot of refactoring in the future, especially during the inevitable move to `SDL3`, but having ambient ocean sounds and the ability to play music in the background really adds to a scene. 

This month also saw the introduction of the `viewer` application. This is an application in the toybox repo that lives outside of the concept of a "sample". It's more of a tool for me to be able to throw whatever scene I want at the engine and make sure it works without having to modify an existing sample.

## August
Another slower month where I was mostly making changes to my homebrewed entity component system (ECS). I expanded the ticking interface so that systems could expose multiple tick functions. Other than that it was just a lot of misc work; fixing/upgrading dependencies, correcting renderering behavior. 

## September
Here I finally hit a breaking point with my custom ECS. It was a good exercise while it lasted but when really comparing its capabilites to other ECS libraries I realized I was mostly just re-inventing the wheel, and not particularly well. All things considered my ECS wasn't *that* bad; I think the interface wasn't great but it wasn't awful either. But after realizing that I was more interested in building a renderer than an entire ECS I began moving to `flecs`. This was really a great move. Flecs has a wonderful C api and the docs aren't half bad. It was a bit of an uphill battle at first as I struggled to grok the API but after reading the docs a few times I got everything working. The end result is a faster ECS with a cleaner API resulting in less code for me to maintain. And as the saying goes: `red code best code`.

## October
The other motivation to use Flecs was that I wanted to bring in a physics library to really get things moving. The problem was that most physics APIs suitable for game engines are written in C++, not C. I wasn't really interested in maintianing, say, an entire C binding to PhysX. Nor was I thrilled at the concept of having to integrate something like PhysX into my build system. Thankfully the [Jolt](https://github.com/jrouwe/JoltPhysics) physics library had by back. After some upstream work to get the `joltphysics` package on vcpkg behaving nicely I was able to build a C++ Flecs system that can act as my physics implementation and abstraction. 

This month also brought an end to my SSAO implementation. It exhibited a lot of AO bleeding and after spending a lot of time debugging it I decided to put it on ice and just get rid of it. It's also expensive so it really needed work to be made scalable anyway. I'd like to go back and try to re-implement SSAO based on Intel's [Adaptive SSAO](https://www.intel.com/content/www/us/en/developer/articles/technical/adaptive-screen-space-ambient-occlusion.html) technique but that remains a future plan.

## November
November was a month of tweaks and fixes. Adding better deadzone handling for controller input, implementing a third person character controller for fun etc. But this also saw a commitment that started in October. Previously my math library was using compiler extensions supported by both GCC and clang. However I've decided that I want to commit the kind of crimes that only LLVM will let me. The math library now has a first-class `float4` type that supports swizzling. Effectively making it syntactically compatible with HLSL's `float4`. 

Since this locked me in to clang exclusively I figured I'd also go whole hog and rely on some other LLVM exclusive features including blocks. Blocks *kind of* solve the problem of missing lambdas in C. Unfortunately they rely on some runtime support but thankfully Apple's block runtime is not only open source but actually quite small so now I have blocks on every platform. Was this a good idea? Kind of! Combined with `__auto_type` (defined to `tb_auto`) I can do things like this: 
```
tb_auto create_world =
      ^(TbWorld *world, TbRenderThread *thread, SDL_Window *window) {
        tb_create_default_world(world, thread, window);
        tb_register_viewer_sys(world);
      };
tb_auto world = tb_create_world(std_alloc, tmp_alloc, create_world,
                                  render_thread, window);
```
Really this isn't much different than using a function pointer to specify a kind of "on created" callback but I think there's a lot of value in having the block being in-line next to where it is used. At least it makes sense in my brain.

## December
Hey that's now! The month isn't out but I've still gotten quite a bit done. I decided to go all-out on a "bindless" architecture and I've gotten the Bistro test scene all drawn in 1 draw call. Or at least all the opaque objects, transparent objects need their own draw call. This means I was able to get rid of multiple shaders that were effectively just permutations of the same shader but accepting different vertex input layouts. I went a bit further than that though and not only are all textures also stored in a global descriptor array but so is all mesh data, including the indices. Instead of using `vkCmdDrawIndexedIndirect` and binding one big index buffer I instead draw with `vkCmdDrawIndirect`. I supply some per-draw info which is looked up with the `DrawIndex` built-in and then look up the mesh from there. The index of each "vertex" of the vert count supplied in the draw command is used as an index into the mesh's index buffer and then *that* value is used to index the mesh's vertex buffers. This seems like it might be slow but so far it's turned out to be a totally reasonable cost. At least for now whatever slowdown this may incur seems to be worth the fact that I have a *lot* less code to maintain. And the CPU time required to record these indirect draws is so much lower than before that the Bistro scene is now entirely GPU bound. Time will tell if this is shippable by any standard but I'm feeling good about it.

## Past
During all of this I also helped to ship the game Immortals of Aveum at my day job! The studio I work at was contracted by Ascendant and I was responsible for the Xbox Series X/S versions of the game. That could probably use a post of its own but I learned a lot about Lumen and Nanite in UE5. This was also my first shipped console game and I'll just be say I'm really happy with my work even if the game didn't quite hit the mark it was looking to. The day job obviously takes priority over toybox but I'm still really happy with what I was able to do in my free time.

## Future
December isn't out just yet but I am going to try to take at least some kind of a break. At least for a little bit. But I have lots that I still want to do. One of the few big costs on the CPU left in Bistro is in the render object system. All objects have their transform written to their descriptor every frame and that obviously can take a lot of time in a big scene. I'd like to add a concept of "mobility" so that only objects marked as movable/dynamic have to have their transforms updated; even then it should only be if the transform has actually changed. I also really want to get some sort of Anti Aliasing implemented. At least FXAA but Intel's CMAA2 also seems like a pretty compelling option. The Ocean also needs a lot more love. I'd like to try actually implementing an inverse FFT based wave simulation and I need to finally start using Jolt to move the boat around in the water. Plus I've had the itch to whip up a First Person controller and see what kind of levels I can whitebox. I guess I'll just have to see what 2024 has in store for me.