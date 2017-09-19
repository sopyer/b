---
title: "Vulkan Shader Sample"
date: 2017-09-19T20:57:38+03:00
---

In previous blog post I presented simple Vulkan program, which was mostly initialization/deinitialization sequences. I replaced rendering with simple clear. In this blog post I'll explain how to do actual rendering. I thought that simple triangle will be too boring, so instead I decided to illustrate shaders with [primitive raytracing example from ShaderToy](https://www.shadertoy.com/view/Xds3zN). 

Also first I want to list few interesting Vulkan articles, which I encountered since last time:

- Quick intro to Vulkan: https://renderdoc.org/vulkan-in-30-minutes.html
- Vulkan objects diagram explaining their relations: http://gpuopen.com/understanding-vulkan-objects/
- GPUOpen blog post about synchronization in Vulkan: http://gpuopen.com/performance-tweets-series-barriers-fences-synchronization/
- A single-header library with a simplified interface for Vulkan synchronization https://github.com/Tobski/simple_vulkan_synchronization
- Collection of links about Vulkan: https://github.com/vinjn/awesome-vulkan
- Sascha Willems blog post about new Vulkan device simulation layer:  https://www.saschawillems.de/?p=2485

I think they can be useful to those who is making first steps in mastering Vulkan, people like me.

## Renderpass and Clear

Lets start with simple preparations and learn a new way to clear images. Entire code can be found in [this commit](https://github.com/sopyer/Vulkan/commit/2408b81ff479ffd64e1af768c21062c4334e7371).

In Vulkan in order to issue draw calls, they should be encapsulated  within render passes.  So lets start with creating renderpass and framebuffer objects. Framebuffer object are somewhat similar OpenGL ones - set of rendertargets which can be rendered into and also read as input attachments within different subpasses of renderpass. But renderpass object is a new concept. On GPUOpen blog there is [introduction to it](http://gpuopen.com/vulkan-renderpasses/). I like to think about renderpass as a graph, which encodes data operations on individual rendertargets within framebuffer. In this graph nodes are rendertargets and edges are per pixel read/write operations. These information is then used in driver in order to optimize rendering, especially in [GPUs with tiled rendering](http://c0de517e.blogspot.com/2017/08/tiled-hardware-speculations.html). People have taken this idea even further, exposing entire frame as a graph:

 - [Render graphs and Vulkan — a deep dive](http://themaister.net/blog/2017/08/15/render-graphs-and-vulkan-a-deep-dive/)
 - [FrameGraph: Extensible Rendering Architecture in Frostbite](https://www.slideshare.net/DICEStudio/framegraph-extensible-rendering-architecture-in-frostbite)

Lets move to renderpass creation code:

```c
VkRenderPass renderPass;

int createRenderPass()
{
    VkRenderPassCreateInfo renderPassCreateInfo = {
        .sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO,
        .attachmentCount = 1,
        .pAttachments = &(VkAttachmentDescription) {
            .format = surfaceFormat.format,
            .samples = VK_SAMPLE_COUNT_1_BIT,
            .loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR,
            .storeOp = VK_ATTACHMENT_STORE_OP_STORE,
            .stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE,
            .stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE,
            .initialLayout = VK_IMAGE_LAYOUT_UNDEFINED,
            .finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR,
        },
        .subpassCount = 1,
        .pSubpasses = &(VkSubpassDescription) {
            .pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS,
            .colorAttachmentCount = 1,
            .pColorAttachments = &(VkAttachmentReference) {
                .attachment = 0,
                .layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL,
            },
        },
        .dependencyCount = 1,
        .pDependencies = &(VkSubpassDependency) {
            .srcSubpass = VK_SUBPASS_EXTERNAL,
            .dstSubpass = 0,
            .srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT,
            .dstStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT,
            .srcAccessMask = 0,
            .dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_READ_BIT | VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT,
            .dependencyFlags = 0,
        },
    };

    vkCreateRenderPass(device, &renderPassCreateInfo, 0, &renderPass);

    return 1;
}

void destroyRenderPass()
{
    vkDestroyRenderPass(device, renderPass, NULL);
}
``` 

In my example frame structure is quite simple - we have only one attachment, which is cleared at start `.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR` and at the end we is stored to memory `.storeOp = VK_ATTACHMENT_STORE_OP_STORE` with layout `.finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`. We also have only one dependency - on completion of pipeline writes to attachment before we start our own writes.

Next we need to create framebuffers, one for every swapchain image:

```c
VkImageView swapchainImageViews[MAX_SWAPCHAIN_IMAGES];
VkFramebuffer framebuffers[MAX_SWAPCHAIN_IMAGES];

int createFramebuffers()
{
    for (uint32_t i = 0; i < swapchainImageCount; ++i)
    {
        VkImageViewCreateInfo createInfo = {
            .sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO,
            .image = swapchainImages[i],
            .viewType = VK_IMAGE_VIEW_TYPE_2D,
            .format = surfaceFormat.format,
            .subresourceRange = {
                .aspectMask = VK_IMAGE_ASPECT_COLOR_BIT,
                .baseMipLevel = 0,
                .levelCount = 1,
                .baseArrayLayer = 0,
                .layerCount = 1,
            },
        };
        vkCreateImageView(device, &createInfo, 0, &swapchainImageViews[i]);

        VkFramebufferCreateInfo framebufferCreateInfo = {
            .sType = VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO,
            .renderPass = renderPass,
            .attachmentCount = 1,
            .pAttachments = &swapchainImageViews[i],
            .width = swapchainExtent.width,
            .height = swapchainExtent.height,
            .layers = 1,
        };
        vkCreateFramebuffer(device, &framebufferCreateInfo, 0, &framebuffers[i]);
    }

    return 1;
}

void destroyFramebuffers()
{
    for (uint32_t i = 0; i < swapchainImageCount; ++i)
    {
        vkDestroyFramebuffer(device, framebuffers[i], NULL);
        vkDestroyImageView(device, swapchainImageViews[i], NULL);
    }
}
```

When everything is in place we can now clear our rendertarget:

```c
    vkCmdBeginRenderPass(commandBuffers[index],
        &(VkRenderPassBeginInfo) {
        .sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO,
        .renderPass = renderPass,
        .framebuffer = framebuffers[imageIndex],
        .clearValueCount = 1,
        .pClearValues = &(VkClearValue) { 0.0f, 0.1f, 0.2f, 1.0f },
        .renderArea.offset = (VkOffset2D) { .x = 0,.y = 0 },
        .renderArea.extent = swapchainExtent,
        },
        VK_SUBPASS_CONTENTS_INLINE
    );
    vkCmdEndRenderPass(commandBuffers[index]);
```

Please note images do not require `VK_IMAGE_USAGE_TRANSFER_DST_BIT` flag with new clear approach.

## Simple Triangle

Next step is to create Vulkan pipeline object and use it to draw triangle. My example is the same as in [this Vulkan tutorial](https://vulkan-tutorial.com/). Entire code can be viewed in [this commit](https://github.com/sopyer/Vulkan/commit/a70d9029ad9b267cc4da7dbbdfa3136215dd38d0). 

Lets start with pipeline creation. And this step requires to fill `VkGraphicsPipelineCreateInfo`, which provides describes entire pipeline. There are quite a few structures to be filled and objects to be created, and it can be quite tedious and intimidating at times. Few notes about my case. I do not use any vertex inputs so `VkPipelineVertexInputStateCreateInfo` is empty. Vertices are generated on fly in vertex shader using `gl_VertexIndex` built-in shader  input variable, so program does not need vertex data. Program does not use any uniforms,  so it is created with empty pipeline layout. Another point is program uses dynamic states in order to avoid pipeline recreation, when viewport or scissor rectangle change.

Here is code:

```C
VkShaderModule createShaderModule(const char* shaderFile)
{
    VkShaderModule shaderModule = VK_NULL_HANDLE;

    HANDLE hFile = CreateFile(shaderFile, GENERIC_READ, FILE_SHARE_READ, NULL, OPEN_EXISTING, 0, NULL);
    if (hFile == INVALID_HANDLE_VALUE) return VK_NULL_HANDLE;

    LARGE_INTEGER size;
    GetFileSizeEx(hFile, &size);

    HANDLE hMapping = CreateFileMapping(hFile, NULL, PAGE_READONLY, 0, 0, NULL);
    CloseHandle(hFile);
    if (!hMapping) return VK_NULL_HANDLE;

    const uint32_t* data = (const uint32_t*)MapViewOfFile(hMapping, FILE_MAP_READ, 0, 0, 0);

    VkShaderModuleCreateInfo shaderModuleCreateInfo = {
        .sType = VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO,
        .codeSize = size.LowPart,
        .pCode = data,
    };
    vkCreateShaderModule(device, &shaderModuleCreateInfo, 0, &shaderModule);

    UnmapViewOfFile(data);
    CloseHandle(hMapping);

    return shaderModule;
}

VkPipelineLayout pipelineLayout; 
VkPipeline pipeline;

int createPipeline()
{
    VkShaderModule vertexShader = createShaderModule("shaders\\shader.vert.spv");
    VkShaderModule fragmentShader = createShaderModule("shaders\\shader.frag.spv");

    const VkPipelineShaderStageCreateInfo stages[] = {
        {
            .sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO,
            .stage = VK_SHADER_STAGE_VERTEX_BIT,
            .module = vertexShader,
            .pName = "main",
        },
        {
            .sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO,
            .stage = VK_SHADER_STAGE_FRAGMENT_BIT,
            .module = fragmentShader,
            .pName = "main",
        },
    };
    VkPipelineVertexInputStateCreateInfo vertexInputState = {
        .sType = VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO,
    };
    VkPipelineInputAssemblyStateCreateInfo inputAssemblyState = {
        .sType = VK_STRUCTURE_TYPE_PIPELINE_INPUT_ASSEMBLY_STATE_CREATE_INFO,
        .topology = VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST,
        .primitiveRestartEnable = VK_FALSE,
    };
    VkPipelineRasterizationStateCreateInfo rasterizationState = {
        .sType = VK_STRUCTURE_TYPE_PIPELINE_RASTERIZATION_STATE_CREATE_INFO,
        .depthClampEnable = VK_FALSE,
        .rasterizerDiscardEnable = VK_FALSE,
        .polygonMode = VK_POLYGON_MODE_FILL,
        .cullMode = VK_CULL_MODE_BACK_BIT,
        .frontFace = VK_FRONT_FACE_CLOCKWISE,
        .depthBiasEnable = VK_FALSE,
        .depthBiasConstantFactor = 0.0f,
        .depthBiasClamp = 0.0f,
        .depthBiasSlopeFactor = 0.0f,
        .lineWidth = 1.0f,
    };
    VkPipelineViewportStateCreateInfo viewportState = {
        .sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO,
        .viewportCount = 1,
        .pViewports = 0,
        .scissorCount = 1,
        .pScissors = 0,
    };
    VkPipelineMultisampleStateCreateInfo multisampleState = {
        .sType = VK_STRUCTURE_TYPE_PIPELINE_MULTISAMPLE_STATE_CREATE_INFO,
        .rasterizationSamples = VK_SAMPLE_COUNT_1_BIT,
        .sampleShadingEnable = VK_FALSE,
        .minSampleShading = 1.0f,
        .pSampleMask = 0,
        .alphaToCoverageEnable = VK_FALSE,
        .alphaToOneEnable = VK_FALSE,
    };
    VkPipelineColorBlendStateCreateInfo colorBlendState = {
        .sType = VK_STRUCTURE_TYPE_PIPELINE_COLOR_BLEND_STATE_CREATE_INFO,
        .logicOpEnable = VK_FALSE,
        .attachmentCount = 1,
        .pAttachments = (VkPipelineColorBlendAttachmentState[]) {
            {
                .blendEnable = VK_FALSE,
                .srcColorBlendFactor = VK_BLEND_FACTOR_ONE,
                .dstColorBlendFactor = VK_BLEND_FACTOR_ZERO,
                .colorBlendOp = VK_BLEND_OP_ADD,
                .srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE,
                .dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO,
                .alphaBlendOp = VK_BLEND_OP_ADD,
                .colorWriteMask = VK_COLOR_COMPONENT_R_BIT | VK_COLOR_COMPONENT_G_BIT | VK_COLOR_COMPONENT_B_BIT | VK_COLOR_COMPONENT_A_BIT,
            },
        },
    };
    VkPipelineDynamicStateCreateInfo dynamicState = {
        .sType = VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO,
        .dynamicStateCount = 2,
        .pDynamicStates = (VkDynamicState[]) { VK_DYNAMIC_STATE_VIEWPORT, VK_DYNAMIC_STATE_SCISSOR },
    };

    VkPipelineLayoutCreateInfo pipelineLayoutCreateInfo = {
        .sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO,
    };
    vkCreatePipelineLayout(device, &pipelineLayoutCreateInfo, 0, &pipelineLayout);

    VkGraphicsPipelineCreateInfo pipelineCreateInfo = {
        .sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO,
        .stageCount = 2,
        .pStages = stages, 
        .pVertexInputState = &vertexInputState,
        .pInputAssemblyState = &inputAssemblyState,
        .pViewportState = &viewportState,
        .pRasterizationState = &rasterizationState,
        .pMultisampleState = &multisampleState ,
        .pColorBlendState = &colorBlendState,
        .pDynamicState = &dynamicState,
        .layout = pipelineLayout,
        .renderPass = renderPass,
    };

    vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &pipelineCreateInfo, 0, &pipeline);

    vkDestroyShaderModule(device, vertexShader, 0);
    vkDestroyShaderModule(device, fragmentShader, 0);

    return pipeline != 0;
}
```

Next step is to create and compile shaders. Compilation is done with glslangValidator utility from Vulkan SDK. Also please note shaders should be compiled before first run, otherwise program will crash due to some missing error handling.

Vertex shader:

```glsl
#version 450
#extension GL_ARB_separate_shader_objects : enable

out gl_PerVertex
{
    vec4 gl_Position;
};

layout(location = 0) out vec3 fragColor;

vec2 positions[3] = vec2[](
    vec2(0.0, -0.5),
    vec2(0.5, 0.5),
    vec2(-0.5, 0.5)
);

vec3 colors[3] = vec3[](
    vec3(1.0, 0.0, 0.0),
    vec3(0.0, 1.0, 0.0),
    vec3(0.0, 0.0, 1.0)
);

void main()
{
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
    fragColor = colors[gl_VertexIndex];
}
```

Fragment shader:

```glsl
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(location = 0) in vec3 fragColor;

layout(location = 0) out vec4 outColor;

void main()
{
    outColor = vec4(fragColor, 1.0);
}
```

Batch file for shader compilations:

```shell
for /r %%f in (*.vert;*.frag) do %VULKAN_SDK%\Bin32\glslangValidator.exe -V %%f -o %%f.spv
pause
```

And that is all we need to see a triangle.

## Fullscreen Triangle

Next step in our tutorial is to cover entire screen with single triangle. For this purpose I created 2 new shaders fullscreentri.vert and rtprimitives.frag. Both shaders are the same as in previous example, except for positions array:

```glsl
vec2 positions[3] = vec2[](
    vec2(-1.0,  1.0),
    vec2(-1.0, -3.0),
    vec2( 3.0,  1.0)
);
```

This technique was quite widespread some time ago as it allows to avoid running fragment shader twice along diagonal, which is a case when screen is covered with 2 triangles. Fragment shader is not invoked for regions outside of viewport. In modern engines I believe most or a lot of fullscreen passes are done in compute.

Code can be found in [this commit](https://github.com/sopyer/Vulkan/commit/b8ef2a4622c1cb69604afa4406f1ef59d88052d3)

## Compiling Shaders with Ninja

Batch file has one problem - it always rebuild all shaders. This can become very annoying very quickly. Especially if number of shaders increases. So in order to improve this behavior I [replaced batch file with ninja build file](https://github.com/sopyer/Vulkan/commit/82f176afd7c7439ae5d66164c3b6a219edc9447d). [Ninja](https://ninja-build.org/) is a build system written by Google. It is low level and similar to make, but faster.

So here is our simple ninja file:

```Ninja
builddir = ../build/temp/shaders

compile_glsl_vertex = cmd /c %VULKAN_SDK%\Bin32\glslangValidator -S vert
compile_glsl_fragment = cmd /c %VULKAN_SDK%\Bin32\glslangValidator -S frag

rule compile_glsl_vs
    command = $compile_glsl_vertex -V $in -o $out

rule compile_glsl_fs
    command = $compile_glsl_fragment -V $in -o $out

build shader.spv-vs: compile_glsl_vs shader.glsl-vs
build fullscreentri.spv-vs: compile_glsl_vs fullscreentri.glsl-vs
build shader.spv-fs: compile_glsl_fs shader.glsl-fs
build rtprimitives.spv-fs: compile_glsl_fs rtprimitives.glsl-fs

```

## Distance Field Raytracing

After all setup work we approached our main goal - to implement raytracing shader. As I said I decided to pick a [shader](https://www.shadertoy.com/view/Xds3zN), written by [Inigo Quilez](http://www.iquilezles.org/index.html), which demonstrates how to raytrace basic primitives and results of some basic operations on them. You can find more details in [this article](http://www.iquilezles.org/www/articles/distfunctions/distfunctions.htm).

Source code for this part of tutorial can be found in [this commit](https://github.com/sopyer/Vulkan/commit/952e65c362bcb3f349b9bfe190a01425a62a5e64). I consider it funny that all changes are limited to shaders and all C++ code stays the same.
Please note that I hardcoded all parameters like this in order to simplify implementation:

```
struct FSConst {
    vec2 resolution;
    vec2 mouse;
    float time;
};

const FSConst u_input = {
    vec2(1280.0, 720.0),
    vec2(0.0, 0.0),
    0.0
};
```

## Push Constants

And now lets talk about cleaning up some technical debt related hardcoded constants in shaders. The simplest way(and also the fastest way) in Vulkan to pass some information to shaders is to use [push constants](https://github.com/KhronosGroup/Vulkan-Docs/blob/cd4de492bf04b4a7542a5029ae4b998f117a7ac8/doc/specs/misc/GL_KHR_vulkan_glsl.txt#L130). Lets see what changes are required.

First we need to modify pipeline layout
```C
     VkPipelineLayoutCreateInfo pipelineLayoutCreateInfo = {
          .sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO,
          .pushConstantRangeCount = 1,
          .pPushConstantRanges = &(VkPushConstantRange) {
              .stageFlags = VK_SHADER_STAGE_FRAGMENT_BIT,
              .offset = 0,
              .size = 20,
          },
      };
      vkCreatePipelineLayout(device, &pipelineLayoutCreateInfo, 0, &pipelineLayout);
```

Next data needs to be uploaded via push constants
```
vkCmdPushConstants(commandBuffers[index], pipelineLayout, VK_SHADER_STAGE_FRAGMENT_BIT, 0, sizeof(fragmentConstants), fragmentConstants);
```

And also fragment shader should be modified accordingly
```C
layout(push_constant) uniform FSConst {
    vec2 resolution;
    vec2 mouse;
    float time;
} u_input;
```

In the end we have a dynamic version of raytracing tutorial. I also added some support for time and mouse - similar to original shader on ShaderToy.

Hope you enjoyed this little ride and it was not very boring. Complete code of final version is available in [this commit](https://github.com/sopyer/Vulkan/commit/906317d8f0d31aab2e208ba0f28de244b81f76c4)
