---
layout: post
title: Adding Shadows (Carbon Devlog #2)
subtitle: Setting up another Vulkan pipeline
gh-repo: DraperDanMan/DraperDanMan.github.io
gh-badge: [star, follow]
tags: [vulkan, renderer, cpp]

---

It has been about a month since my last devlog. I think that's a good amount of time honestly, because I really only work on this renderer on Sunday, it ends up only being around 4 days of work between each update which is about a work week right? Here is a practical look at what I did to get it going, As well as how I got stuck for a solid several hours over a couple of days trying fix an issue.

#### What makes a shadow pass

I have never implemented a shadow pass before, I've understood the general idea, render a buffer of what each light can see as if it was a camera and then apply that some how in the normal render pass. That is about where my understanding ended. So off I went to learn a little more about it, surprisingly Wikipedia's [Shadow Mapping](https://en.wikipedia.org/wiki/Shadow_mapping) article really came through here with a great description of each step of the algorithm. 

The general idea ended up not being as scary as I have first thought.

1. Render the scene from the light source, perspective for spotlight and orthographic for directional.
2. Instead of writing out any colour, we only write out the Z position to an image buffer.
3. In our normal render pass, we pass in a sampler to that image buffer and also the lights matrix.
4. Inside our vertex shader we can multiply our lights matrix against our model-space position to get the light-space coordinate.
5. Inside our fragment shader we can use that coordinate to check our depth texture against to see if the the current z position is greater or less then the depth. 
6. Finally, multiply our output color by our shadow value.

Here is the scene we're going to be adding shadows to.

![Tree scene without shadows](/img/devlog2/carbonTrees.gif)

#### Stringing it together in Vulkan

The hurdles here were going to be the fact that I needed to set up another framebuffer, render pass and graphics pipeline which my current hacky renderer doesn't have nice functionality for. So did I fix that? not really. I ended up just creating a function to `CreateShadowPassFrameBuffer` to create my depth image, create a sampler for that image, create the render pass for the shadows and then finally the framebuffer to render the shadows into the image.

```cpp
void CreateShadowPassFrameBuffer(VkDevice device, const VkPhysicalDeviceMemoryProperties& memoryProps)
{
    //the size of the shadow image
    g_shadow_pass_frame_buffer.m_width  = 2048;
    g_shadow_pass_frame_buffer.m_height = 2048;
    //tell the image its only going to be used for writing depth and to be sampled
    VkImageUsageFlags flags             = VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT | VK_IMAGE_USAGE_SAMPLED_BIT;

    // For shadow mapping we only need a depth attachment
    CreateImage(g_shadow_pass_frame_buffer.m_depthImage, device, memoryProps, 2048, 2048, VK_FORMAT_D32_SFLOAT, flags);

    // Create sampler to sample from to depth attachment
    // Used to sample in the fragment shader for shadowed rendering
    VkFilter shadowFilterMode   = VK_FILTER_LINEAR; // : VK_FILTER_NEAREST;
    VkSamplerCreateInfo sampler = {VK_STRUCTURE_TYPE_SAMPLER_CREATE_INFO};
    sampler.magFilter           = shadowFilterMode;
    sampler.minFilter           = shadowFilterMode;
    sampler.mipmapMode          = VK_SAMPLER_MIPMAP_MODE_LINEAR;
    sampler.addressModeU        = VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE;
    sampler.addressModeV        = sampler.addressModeU;
    sampler.addressModeW        = sampler.addressModeU;
    sampler.mipLodBias          = 0.0f;
    sampler.maxAnisotropy       = 1.0f;
    sampler.minLod              = 0.0f;
    sampler.maxLod              = 1.0f;
    sampler.borderColor         = VK_BORDER_COLOR_FLOAT_OPAQUE_WHITE;
    VK_CHECK(vkCreateSampler(device, &sampler, nullptr, &g_shadow_pass_frame_buffer.m_depthSampler));

    // the descriptor we're going to push before we run the shadowpass
    g_shadow_pass_frame_buffer.m_descriptor = {};
    
    g_shadow_pass_frame_buffer.m_descriptor.sampler = g_shadow_pass_frame_buffer.m_depthSampler;
    g_shadow_pass_frame_buffer.m_descriptor.imageView = g_shadow_pass_frame_buffer.m_depthImage.m_imageView;
    g_shadow_pass_frame_buffer.m_descriptor.imageLayout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_READ_ONLY_OPTIMAL;

    //This function will be shown a little bit further down
    CreateShadowRenderPass(device);

    // Set up the framebuffer create info with the depth image and renderpass
    VkFramebufferCreateInfo fbufCreateInfo = {VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO};
    fbufCreateInfo.renderPass      = g_shadow_pass_frame_buffer.m_renderPass;
    fbufCreateInfo.attachmentCount = 1;
    fbufCreateInfo.pAttachments    = &g_shadow_pass_frame_buffer.m_depthImage.m_imageView;
    fbufCreateInfo.width  = g_shadow_pass_frame_buffer.m_width;
    fbufCreateInfo.height = g_shadow_pass_frame_buffer.m_height;
    fbufCreateInfo.layers = 1;

    //actually create the vulkan framebuffer.
    VK_CHECK(vkCreateFramebuffer(device, &fbufCreateInfo, nullptr, &g_shadow_pass_frame_buffer.m_frameBuffer));
}
```

Handballing all of this work made understanding the things required for the shadow pass pretty straight forward. There is two things in the above code that need a bit more information. `CreateShadowRenderPass` and the structure `g_shadow_pass_frame_buffer`. The global structure I just continued to add any fields to that I ended up needing while writing this code. it just made it easier to pass the data around my main.cpp file.

```cpp
//declared near the top of my file
struct ShadowPassFrameBuffer
{
    int32_t m_width; 
    int32_t m_height;
    VkFramebuffer m_frameBuffer;
    Image m_depthImage;
    VkRenderPass m_renderPass;
    VkSampler m_depthSampler;
    VkDescriptorImageInfo m_descriptor;
} g_shadow_pass_frame_buffer;
```

Creating the render pass was pretty easy. We only have 1 attachment and we can use the the subpass dependencies to force layout transitions. So we don't have to worry about creating barriers. I'll be honest I read that you can do this in some of the Sasha Willems [examples](https://github.com/SaschaWillems/Vulkan) and it's pretty handy and makes a lot of sense considering we have to specify a lot of this information anyway.

```cpp
void CreateShadowRenderPass(VkDevice device)
{
    VkAttachmentDescription attachmentDescription{};
    attachmentDescription.format  = VK_FORMAT_D32_SFLOAT;
    attachmentDescription.samples = VK_SAMPLE_COUNT_1_BIT;
    // Clear depth when we start the renderpass
    attachmentDescription.loadOp  = VK_ATTACHMENT_LOAD_OP_CLEAR;
    // We do want to store the results for use outside of this renderpass
    attachmentDescription.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
    //dont care about stencil or the initial layout 
    attachmentDescription.stencilLoadOp  = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    attachmentDescription.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    attachmentDescription.initialLayout  = VK_IMAGE_LAYOUT_UNDEFINED;
    // we do want to make sure that we leave the attachment ready to be read
    attachmentDescription.finalLayout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_READ_ONLY_OPTIMAL;

    // set the depth layout to be optimal for depth/stencil in slot 0
    VkAttachmentReference depthReference = {};
    depthReference.attachment            = 0;
    depthReference.layout                = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;

    //create out subpass and only give it out depth we don't have color
    VkSubpassDescription subpass    = {};
    subpass.pipelineBindPoint       = VK_PIPELINE_BIND_POINT_GRAPHICS;
    subpass.colorAttachmentCount    = 0; // No color attachments
    subpass.pDepthStencilAttachment = &depthReference;

    // Use subpass dependencies for layout transitions
    VkSubpassDependency dependencies[2] = {};

    dependencies[0].srcSubpass      = VK_SUBPASS_EXTERNAL;
    dependencies[0].dstSubpass      = 0;
    dependencies[0].srcStageMask    = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
    dependencies[0].dstStageMask    = VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT;
    dependencies[0].srcAccessMask   = VK_ACCESS_SHADER_READ_BIT;
    dependencies[0].dstAccessMask   = VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;
    dependencies[0].dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT;

    dependencies[1].srcSubpass      = 0;
    dependencies[1].dstSubpass      = VK_SUBPASS_EXTERNAL;
    dependencies[1].srcStageMask    = VK_PIPELINE_STAGE_LATE_FRAGMENT_TESTS_BIT;
    dependencies[1].dstStageMask    = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
    dependencies[1].srcAccessMask   = VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;
    dependencies[1].dstAccessMask   = VK_ACCESS_SHADER_READ_BIT;
    dependencies[1].dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT;

    VkRenderPassCreateInfo renderPassCreateInfo = {VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO};
    renderPassCreateInfo.attachmentCount = 1;
    renderPassCreateInfo.pAttachments    = &attachmentDescription;
    renderPassCreateInfo.subpassCount    = 1;
    renderPassCreateInfo.pSubpasses      = &subpass;
    renderPassCreateInfo.dependencyCount = static_cast<uint32_t>(ARRAYSIZE(dependencies));
    renderPassCreateInfo.pDependencies = dependencies;

    //finally create the actual renderpass object
    VK_CHECK(vkCreateRenderPass(device, &renderPassCreateInfo, nullptr, &g_shadow_pass_frame_buffer.m_renderPass));
}
```

Now that we have set up out frame buffer, image, render pass and sampler. all we need now is a the pipeline definition to tell Vulkan before we renderer our geometry. This is where we'll specify the data and shader programs we're going to be running the geometry through this render pass. This part I sort of attached to the end of my main `CreateGraphicsPipelines` and I added the new sampler layout in my binding 3 slot.

```cpp
	setBindings[2].binding         = 2;
    setBindings[2].descriptorType  = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    setBindings[2].descriptorCount = 1;
    setBindings[2].stageFlags      = VK_SHADER_STAGE_VERTEX_BIT;

	/* Binding 2 Uniform buffer
	struct ShadowUniformBufferVertex
    {
        Matrix4 m_depthMvp; //The Light Camera-like Matrix
    } g_shadow_ubo;
	*/

    setBindings[3].binding         = 3;
    setBindings[3].descriptorType  = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
    setBindings[3].descriptorCount = 1;
    setBindings[3].stageFlags      = VK_SHADER_STAGE_FRAGMENT_BIT;
	/* Binding 3 Shadow Depth Image sampler
	VkDescriptorImageInfo m_descriptor; (in g_shadow_pass_frame_buffer)
	*/
```

In my Graphics pipeline I kind of cheated a little. Instead of creating the whole create info again, (If you have ever created one before you know what I mean) I just changed the parts that needed to be different and then called create again. Saving writing out like 50 lines of the same explicit info. There might be optimization here that could be made, but to get this running I didn't need to change much.

```cpp
void CreateGraphicsPipeline(VkDevice device, VkRenderPass pass, VkPipelineCache pipelineCache, VkPipelineLayout layout)
{
    /*
    ... THE REST OF PIPELINE CREATE INFO ...
    //Create my normal render pipeline
    VK_CHECK(vkCreateGraphicsPipelines(device, pipelineCache, 1, &createInfo, nullptr, &g_pipelines.m_scenePipeline));*/

    //------- Create the ShadowPass Render Pipeline
    stages[0].module                = g_shaders.m_shadowVert; //shadow shader
    createInfo.stageCount           = 1; // only need to run custom vert stage
    colorBlendState.attachmentCount = 0; // no color attachment
    createInfo.renderPass = g_shadow_pass_frame_buffer.m_renderPass; //shadow renderpass
    //Create the new pipeline
    VK_CHECK(vkCreateGraphicsPipelines(device, pipelineCache, 1, &createInfo, nullptr, &g_pipelines.m_shadowPassPipeline));
}
```

The shader to actual write out the shadows is super simple, because all it needs to do is output the depth (z) position.

```glsl
// --- rest of my inputs and structures----

//I pass through my uniform mesh buffer because my meshes can rotate/move
layout(binding = 1) uniform UBOMesh
{
	mat4 projection;
	mat4 model;
	mat4 view;
	mat4 lightSpace; //I realized after that I could have just reused this matrix
	vec4 lightPos;
} uboMesh;

//lights matrix
layout(binding = 2) uniform UBO
{
	mat4 depthMVP;
} ubo;

void main()
{
	vec4 position = uboMesh.model * vec4(inPos.xyz, 1);
	gl_Position = ubo.depthMVP * position;//output the position from the lights POV
}
```

So I have set up the actual shadowmap shader, the next step is to read that in on my main pass fragment shader so I can sample the depth map. However, I also need to know the position of the current fragment in light space (the light casting the shadow). For this I added the lights matrix to my mesh uniform buffer (as you can see in the above code as well). So that in my vertex program, I can calculate the coordinates of the shadow map, and give those to the fragment shader to sample with.

```glsl
// ---inputs and buffers above---

layout(location=3) out vec4 outShadowCoord; //my output shadow coord

//This matrix is used to offset the coordinates to try and
const mat4 biasMat = mat4( 
	0.5, 0.0, 0.0, 0.0,
	0.0, 0.5, 0.0, 0.0,
	0.0, 0.0, 1.0, 0.0,
	0.5, 0.5, 0.0, 1.0 );

void main()
{
    //---rest of main vertex program---
    //here I just transform my model position by the lights matrix and the bias
	outShadowCoord = biasMat * ubo.lightSpace * position;
}
```

With that coordinate given to the fragment I could finally sample the shadows.

```glsl
//---rest of my inputs and outputs ----
layout (binding = 2) uniform sampler2D shadowMap;

#define ambient 0.1

float textureProjection(vec4 shadowCoord, vec2 off)
{
	float shadow = 1.0; //start with no shadow
    //if the coordinate is out of the bounds skip
	if ( shadowCoord.z > -1.0 && shadowCoord.z < 1.0 ) 
	{
		float dist = texture( shadowMap, shadowCoord.st + off ).r;
		if ( shadowCoord.w > 0.0 && dist < shadowCoord.z ) 
		{
            //if we're in shadow use the ambient light amount (0.1)
			shadow = ambient;
		}
	}
	return shadow;
}

void main()
{
		float shadow = textureProjection(inShadowCoord / inShadowCoord.w, vec2(0,0));
		vec3 light = vec3(lightDir.xyz);
		vec4 diffuse = color * max(dot(light,normal),ambient); //simple lighting
		outputColor = diffuse*shadow; // multiply by shadow value.
}
```

All that was left now was to render our scene first with this pipeline then continue with out normal rendering passing in our new shadow map texture. Then we'll be done right?

```cpp
VkClearValue clearValues[2] = {}; //re-used later on for the main pass
//Shadow clears
clearValues[0].depthStencil         = {1.0f, 0};
VkWriteDescriptorSet descriptors[1] = {};
//Render the shadow map for the scene
//Start the shadows
{
    VkRenderPassBeginInfo shadowRenderPassInfo = {VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO};
    shadowRenderPassInfo.renderPass = g_shadow_pass_frame_buffer.m_renderPass;
    shadowRenderPassInfo.framebuffer = g_shadow_pass_frame_buffer.m_frameBuffer;
    shadowRenderPassInfo.renderArea.extent.width = g_shadow_pass_frame_buffer.m_width;
    shadowRenderPassInfo.renderArea.extent.height = g_shadow_pass_frame_buffer.m_height;
    shadowRenderPassInfo.pClearValues = clearValues;
    shadowRenderPassInfo.clearValueCount = 1;

    vkCmdBeginRenderPass(commandBuffer, &shadowRenderPassInfo, VK_SUBPASS_CONTENTS_INLINE);

    //I have two dynamic pipeline attributes, viewport and scissor, so I set them
    VkViewport shadowViewport = {
        0, 0, static_cast<float>(g_shadow_pass_frame_buffer.m_width),
        static_cast<float>(g_shadow_pass_frame_buffer.m_height), 0, 1
    };
    vkCmdSetViewport(commandBuffer, 0, 1, &shadowViewport);

    VkRect2D shadowScissor = {{0, 0}, {g_shadow_pass_frame_buffer.m_width, g_shadow_pass_frame_buffer.m_height}};
    vkCmdSetScissor(commandBuffer, 0, 1, &shadowScissor);

    //bind the shadow pass ready for draw calls.
    vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS,
                              g_pipelines.m_shadowPassPipeline);

    //set up the lights matric buffer and push it to the gpu.
    VkDescriptorBufferInfo uniBufferInfo = {};
    uniBufferInfo.buffer = directionalLightBuffer.m_buffer;
    uniBufferInfo.offset = 0;
    uniBufferInfo.range = directionalLightBuffer.m_size;

    descriptors[0].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
    descriptors[0].dstBinding = 1;
    descriptors[0].descriptorCount = 1;
    descriptors[0].descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    descriptors[0].pBufferInfo = &uniBufferInfo;

    vkCmdPushDescriptorSet(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS,
                           g_pipeline_info.m_layout, 0,
                           ARRAYSIZE(descriptors),
                           descriptors);

    //Draws everything in my scene, a mask should probably be used in future.
    DrawRenderables(commandBuffer, &world, g_pipeline_info.m_layout, viewCamera);

    vkCmdEndRenderPass(commandBuffer); //finish up
}
```

That's my shadow pass, most of this is just setting up structures to pass to these functions. Something that may be abstracted away at some point. Finally, I use another push descriptor to push the shadow map buffer and call my normal render pass.

```cpp
//---a little later in my render loop---
//bind my normal pipeline
vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, g_pipelines.m_scenePipeline);

descriptors[0].sType           = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
descriptors[0].dstBinding      = 2;
descriptors[0].descriptorCount = 1;
descriptors[0].descriptorType  = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
descriptors[0].pImageInfo      = &g_shadow_pass_frame_buffer.m_descriptor;
descriptors[0].pBufferInfo     = nullptr;
//push the descriptor for the shadow map buffer.
vkCmdPushDescriptorSet(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, g_pipeline_info.m_layout, 0, ARRAYSIZE(descriptors), descriptors);

//Normal Draw calls
DrawRenderables(commandBuffer, &world, g_pipeline_info.m_layout, viewCamera);
```

and done. Ta-Da, I should now have shadows!

![Final shadows, nice shadowing from the trees onto the ground](/img/devlog2/CorrectShadows.gif)

#### The Roadblock

At least, that's what I was meant to have, but in reality, I bungled up something. **I will mention that the code in the blog so far has all been correct.** I hardly want to plaster non functioning code here. 

In the end, what I did have though looked like this:

![bungled shadows, that look like a thin strip across the scene](/img/devlog2/BorkedShadows1.gif)

*I should note, that I added some movement to the light to make it move around in a circle around its pivot in this gif, to try figure out if it was the level geometry or not.*

Something was definitely not right. My first thought was that maybe I had positioned my light in a weird place and the shadow was being cast from between two branches. It didn't really make sense why the shadow was going in both directions from the origin of the scene though because it is a spot light and shouldn't be looking face down. So maybe it was my matrix math somewhere? Well for this feature I did write a Matrix4 `LookAt` function that I haven't really tested before. So I started there. 

```cpp
inline Matrix4 LookAt(const Vector4& eye, const Vector4& center, const Vector4& up)
{
    Matrix4 matrix;
    Vector4 f = Normalize(center - eye);
    Vector4 s = Normalize(Cross(up, f));
    Vector4 u = Cross(f, s);

    f.w             = -Dot(f, eye);
    s.w             = -Dot(s, eye);
    u.w             = -Dot(u, eye);
    const Vector4 w = Vector4::wAxis();
    matrix.SetCol(0, s);
    matrix.SetCol(1, u);
    matrix.SetCol(2, f);
    matrix.SetCol(3, w);
    return matrix;
}
```

I step by step went through this on paper and didn't have any issues. As far as I could tell. However, just to double check I went to where I call this function and manually made the result matrix with my translation and rotation matrix utilities and to my surprise, I got a different result. It still wasn't the correct shadow result, though casted doubt on my `LookAt` function. So I added a breakpoint and inspected the result of `LookAt` compared with my manual matrix.  Sure enough something wasn't right.

```cpp
/*  RAW MATRIX
    0.0003,      -0,  0.3481,      -0,
   -0.3232,  0.1293,  0.0002,  0.0001,
   -0.3713, -0.9284,  0.0003, 26.9258,
         0,       0,       0,       1,

				VS
	LookAt MATRIX
	0.0003, -0.3232, -0.3713,       0,
        -0,  0.1293, -0.9284,       0,   
    0.3481,  0.0002,  0.0003,       0,
        -0,  0.0001, 26.9258,       1,
*/
```

Super subtle, but my matrix is mirrored on the diagonal. The result of me forgetting the I have a column based matrix internal and I had just done math that provides the results as rows...  Well, easy fix I have a `SetRow` method on Matrix4 so I can just replace the `SetCol` calls.

![slightly less bungled shadows](/img/devlog2/BorkedShadows2.gif)

Now it's no longer a strip but it's not quite mimicking the shapes of the trees or anything. So something else must be wrong. I'll be honest with you, I was looking for a good while trying to figure out where this was breaking down. The next day I was chatting with another programmer at work and talking about how my matrix was messed up, and it got me thinking about my matrices inside my shader code. What if it was because my matrices were messed up after being sent into the shader. Poking around that night, I still couldn't see what I was missing. Until, while I was simplifying some of the shader code and starte flying the camera around the scene that I noticed something strange. The dark 'shadow' area would move when I moved my view... That meant that my view was affecting some part of the shadow, which it should not be doing. Jumping into my shader code it was immediately obvious, I had been using the position in model-view space not just model space. I was grabbing the position before the projection was added in but it should have also been before the view was added. Separating that out as seen in the shader code provided earlier, not only made it a little clearer what values meant what, but also fixed the issue!

![Fixed final shadows!](/img/devlog2/CorrectShadows.gif)

#### Final Remarks

Well I sure did feel silly getting stuck for several hours on this, though I thought I would share this struggle as its interesting to know that you're not alone when you get stuck on small issues like this. Getting shadows going even at a rudimentary level like this is pretty cool step for my renderer and there is plenty more features I want to mess around with and learn how to do. So look forward to more devlogs. I'm still figuring out the exact nature/structure of these devlogs, I feel like maybe this post had a bit to much hands on code and probably should be more a summary of the things I did and challenges I faced. I didn't cull it though because hopefully this information is useful to you or myself in the future.

Anyway, see you around.
