---
title: "Dealing with geometry in Vulkan"
date: 2017-10-10T20:15:12+03:00
---

In this blog post I decided to deal with rendering geometry in Vulkan. Depending on usage usually people talk about dynamic and static geometry. Dynamic is geometry which changes every frame or alternatively is uploaded every frame, static - respectively is constant across large number of frames. My goal is to learn how to deal with both in Vulkan. Entire source code can be found in these 2 commits: [dynamic geometry](https://github.com/sopyer/Vulkan/commit/563ac8d0bc0551feb9664e214dc32a5e257fbacd) and static [geometry handling](https://github.com/sopyer/Vulkan/commit/4ef4a09b99c4b1f94996ebab7b2b5d786c7b3352). Base for this 2 commits was commit which added ninja build for shaders.

As manual shaders build can be error prone I decided to automate process. I added separate project to Visual Studio solution which builds shaders. In order for everything to work just ensure that path to ninja executable is available in PATH environment variable.

## Memory in Vulkan

Vulkan memory is quite complex topic. Vulkan specification explains that entire memory is split into heaps. And within every heap we can allocate memory types with different properties. Great explanation about device memory types can be found [in this article](https://gpuopen.com/vulkan-device-memory/) on GPUOpen blog. As memory allocation count can be limited(I heard there can be limit at 4096 allocation for example) the need for suballocations arise. AMD developed [Vulkan Memory Allocator](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator) for this purpose. I have created my own limited adaptation of TLSF allocator for cases of external memory handling: [etlsf.h](https://github.com/sopyer/Shadowgrounds/blob/master/storm/storm3dv2/etlsf.h) [etlsf.c](https://github.com/sopyer/Shadowgrounds/blob/master/storm/storm3dv2/etlsf.c). But in this blog post I'll avoid dealing with suballocations for simplicity and clarity.

Now I am going to explain which changes are required to successfully allocate memory and create buffers.

At first lets define some constants

``` C
enum {
    Kb = (1 << 10),
    Mb = (1 << 20),
    ...
    UPLOAD_REGION_SIZE = 64 * Kb,
    UPLOAD_BUFFER_SIZE = FRAME_COUNT * UPLOAD_REGION_SIZE,
};

enum {
    VULKAN_MEM_DEVICE_READBACK,
    VULKAN_MEM_DEVICE_UPLOAD,
    VULKAN_MEM_DEVICE_LOCAL,
    VULKAN_MEM_COUNT
};
```
 
First enumeration defines limits for dynamic buffer size and some helper values.  Second enumeration purpose is to define 3 memory classes suitable for typical operations in graphics applications:

* readback from device operations: screenshots, occlusion depth buffer readback, etc
* uploading data to device: dynamic geometry, uploading static geometry, same for textures, uniforms, etc.
* storage for bulk of GPU data: rendertargets, textures, buffers, etc.

Next lets fill bitmasks which has bits set to 1 for each index of suitable memory types. There is bitmask for each memory class. 

```C
VkPhysicalDeviceMemoryProperties deviceMemProperties;
uint32_t compatibleMemTypes[VULKAN_MEM_COUNT];

uint32_t vkutFindCompatibleMemoryType(VkPhysicalDeviceMemoryProperties* memProperties, VkMemoryPropertyFlags flags)
{
    const uint32_t count = memProperties->memoryTypeCount;
    uint32_t compatibleMemoryTypes = 0;
    for (uint32_t i = 0; i < count; i++)
    {
        int isCompatible = ( memProperties->memoryTypes[i].propertyFlags & flags) == flags;
        compatibleMemoryTypes |= (isCompatible << i);
    }

    return compatibleMemoryTypes;
}

... 
    vkGetPhysicalDeviceMemoryProperties(physicalDevice, &deviceMemProperties);
    compatibleMemTypes[VULKAN_MEM_DEVICE_READBACK]
        = vkutFindCompatibleMemoryType(&deviceMemProperties, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT
                                                            | VK_MEMORY_PROPERTY_HOST_CACHED_BIT);
    compatibleMemTypes[VULKAN_MEM_DEVICE_UPLOAD]
        = vkutFindCompatibleMemoryType(&deviceMemProperties, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT
                                                            | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT);
    compatibleMemTypes[VULKAN_MEM_DEVICE_LOCAL]
        = vkutFindCompatibleMemoryType(&deviceMemProperties, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT);
```

Next is to create actual buffers, upload geometry and render triangle using it.

## Dealing with dynamic geometry

First lets modify shaders in order to read vertex data from buffer.

Vertex shader **vertex_color.glsl-vs**:

```GLSL
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(location = 0) in vec2 aPos;
layout(location = 1) in vec4 aColor;

out gl_PerVertex
{
    vec4 gl_Position;
};
layout(location = 0) out vec4 vColor;

void main()
{
    gl_Position = vec4(aPos, 0.0, 1.0);
    vColor = aColor;
}
```

Fragment shader **vertex_color.glsl-fs**:

```GLSL
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(location = 0) in vec4 vColor;

layout(location = 0) out vec4 rt0;

void main()
{
    rt0 = vColor;
}
```


Next lets add compilation rules to ninja build files:

```
...
build vertex_color.spv-vs: compile_glsl_vs vertex_color.glsl-vs
build vertex_color.spv-fs: compile_glsl_fs vertex_color.glsl-fs
...
```

Now we need to load modified shaders into our program

```C
...
    VkShaderModule vertexShader = createShaderModule("shaders\\vertex_color.spv-vs");
    VkShaderModule fragmentShader = createShaderModule("shaders\\vertex_color.spv-fs");
...
``` 

Another step is to modify `VkPipelineVertexInputStateCreateInfo` in order to reflect changes to shaders

```C
     VkPipelineVertexInputStateCreateInfo vertexInputState = {
         .sType = VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO,
+        .vertexBindingDescriptionCount = 1,
+        .pVertexBindingDescriptions = (VkVertexInputBindingDescription[]) {
+            {.binding = 0, .stride = sizeof(VertexP2C), .inputRate = VK_VERTEX_INPUT_RATE_VERTEX}
+        },
+        .vertexAttributeDescriptionCount = 2,
+        .pVertexAttributeDescriptions = (VkVertexInputAttributeDescription[]) {
+            {.location = 0, .binding = 0, .format = VK_FORMAT_R32G32_SFLOAT,  .offset = offsetof(VertexP2C, x)},
+            {.location = 1, .binding = 0, .format = VK_FORMAT_R8G8B8A8_UNORM, .offset = offsetof(VertexP2C, rgba)}
+        },
     };
```

Next is buffer creation. It is separated into 2 parts - buffer creation and allocating memory for that buffer. Please pay attention to selection of memory index. `bit_ffs32` function returns index of first set bit(counting from least significant bit which is at 0). This way we can automatically and fast find out what memory to use for our allocation. As this buffers is host visible - so we can permanently map it into host memory and use that pointer as destination for memcpy in order to upload our data. Another potentially interesting point is that this buffer can be used not only for vertices but also for indices, uniform data, as storage buffer, as source for memory transfer operations.

```C
VkBuffer uploadBuffer;
VkDeviceMemory uploadBufferMemory;
void* uploadBufferPtr;

int createUploadBuffer()
{
    VkBufferCreateInfo bufferCreateInfo = {
        .sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO,
        .size = UPLOAD_BUFFER_SIZE,
        .usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT | VK_BUFFER_USAGE_INDEX_BUFFER_BIT
                | VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT | VK_BUFFER_USAGE_STORAGE_BUFFER_BIT
                | VK_BUFFER_USAGE_TRANSFER_SRC_BIT,
        .sharingMode = VK_SHARING_MODE_EXCLUSIVE,
        .queueFamilyIndexCount = 1,
        .pQueueFamilyIndices = &queueFamilyIndex,
    };
    vkCreateBuffer(device, &bufferCreateInfo, NULL, &uploadBuffer);
    VkMemoryRequirements memoryRequirements;
    vkGetBufferMemoryRequirements(device, uploadBuffer, &memoryRequirements);

    uint32_t memoryIndex = bit_ffs32(compatibleMemTypes[VULKAN_MEM_DEVICE_UPLOAD] & memoryRequirements.memoryTypeBits);
    VkMemoryAllocateInfo allocInfo = {
        .sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO,
        .allocationSize = memoryRequirements.size,
        .memoryTypeIndex = memoryIndex,
    };
    vkAllocateMemory(device, &allocInfo, NULL, &uploadBufferMemory);
    vkBindBufferMemory(device, uploadBuffer, uploadBufferMemory, 0);
    vkMapMemory(device, uploadBufferMemory, 0, VK_WHOLE_SIZE, 0, &uploadBufferPtr);

    return TRUE;
}

void destroyUploadBuffer()
{
    vkFreeMemory(device, uploadBufferMemory, NULL);
    vkDestroyBuffer(device, uploadBuffer, NULL);
}
```

Last missing piece of puzzle is rendering. We copy data, bind buffer and everything else is still the same. Please note how we select target region for our data - based on frame index. In our firs tutorial we introduced frame synchronization mechanism. And in this tutorial we finally can benefit from it. We always know that there is only `MAX_FRAME_COUNT-1` executing on the device, so we can reuse region of unused frame for data storage.

```C
typedef struct tagVertexP2C
{
    float x, y;
    uint32_t rgba;
} VertexP2C;

...

void draw_frame()
{
    uint32_t index = (frameIndex++) % FRAME_COUNT;
    vkWaitForFences(device, 1, &frameFences[index], VK_TRUE, UINT64_MAX);
    vkResetFences(device, 1, &frameFences[index]);
    
    VkDeviceSize uploadFrameOffset = index * UPLOAD_REGION_SIZE;
    VertexP2C vertices[] = {
        {  0.0f, -0.5f, 0xFF0000FF },
        {  0.5f,  0.5f, 0xFF00FF00 },
        { -0.5f,  0.5f, 0xFFFF0000 }
    };
    memcpy((uint8_t*)uploadBufferPtr + uploadFrameOffset, vertices, sizeof(vertices));

...

    vkCmdBeginRenderPass(commandBuffers[index], ...)
    
    vkCmdBindPipeline(commandBuffers[index], VK_PIPELINE_BIND_POINT_GRAPHICS, pipeline);

...

    vkCmdBindVertexBuffers(commandBuffers[index], 0, 1, &uploadBuffer, (VkDeviceSize[]) { 0 });
    vkCmdDraw(commandBuffers[index], 3, 1, 0, 0);

    vkCmdEndRenderPass(commandBuffers[index]);

...

}
```

And one more thing. In Vulkan we always synchronize memory operations explicitly. And we have host write operation in our code. I had this question in mind: do I need to put pipeline barier to synchronize host writes and vertex input read operations? As it turned out answer is no. Some functions in Vulkan do this implicitly and one of this functions is `vkSubmitQueue`. But in some cases it is still needed for example if we write memory after `vkSubmitQueue` and still expect it to be used in submitted frame. As example I can provide this [tweet](https://twitter.com/nlguillemot/status/889281958961958912) where such situation arise.

## Dealing with static geometry

Next is rendering static geometry. 

First lets define constant for static buffer size

```C
enum {
...
    STATIC_BUFFER_SIZE = 64 * Kb,
};
```

Next is static buffer creation - similar to dynamic buffer creation, but this time we use device local memory for buffer. Another difference we replaced `VK_BUFFER_USAGE_TRANSFER_SRC_BIT` flag with `VK_BUFFER_USAGE_TRANSFER_DST_BIT` flag, as our buffer will be used as destination for copy operation.

```C
...

VkBuffer staticBuffer;
VkDeviceMemory staticBufferMemory;
 
int createUploadBuffer()
{
    VkBufferCreateInfo bufferCreateInfo;
    VkMemoryAllocateInfo allocInfo;
    VkMemoryRequirements memoryRequirements;
    uint32_t memoryIndex;

...
 
    bufferCreateInfo = (VkBufferCreateInfo){
        .sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO,
        .size = STATIC_BUFFER_SIZE,
        .usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT | VK_BUFFER_USAGE_INDEX_BUFFER_BIT
        | VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT | VK_BUFFER_USAGE_STORAGE_BUFFER_BIT
        | VK_BUFFER_USAGE_TRANSFER_DST_BIT,
        .sharingMode = VK_SHARING_MODE_EXCLUSIVE,
        .queueFamilyIndexCount = 1,
        .pQueueFamilyIndices = &queueFamilyIndex,
    };
    vkCreateBuffer(device, &bufferCreateInfo, NULL, &staticBuffer);
    vkGetBufferMemoryRequirements(device, staticBuffer, &memoryRequirements);

    memoryIndex = bit_ffs32(compatibleMemTypes[VULKAN_MEM_DEVICE_LOCAL] & memoryRequirements.memoryTypeBits);
    allocInfo = (VkMemoryAllocateInfo){
        .sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO,
        .allocationSize = memoryRequirements.size,
        .memoryTypeIndex = memoryIndex,
    };
    vkAllocateMemory(device, &allocInfo, NULL, &staticBufferMemory);
    vkBindBufferMemory(device, staticBuffer, staticBufferMemory, 0);

    return TRUE;
}
 
void destroyUploadBuffer()
{
    vkFreeMemory(device, staticBufferMemory, NULL);
    vkDestroyBuffer(device, staticBuffer, NULL);
...
}
```

Now we can start implementing rendering with new buffer, but first lets modify rendering code a bit:

```C
void draw_frame()
{
    uint32_t index = frameIndex % FRAME_COUNT;
    vkWaitForFences(device, 1, &frameFences[index], VK_TRUE, UINT64_MAX);
    vkResetFences(device, 1, &frameFences[index]);
 
    size_t uploadOffset = index * UPLOAD_REGION_SIZE;
    size_t uploadLimit = uploadOffset + UPLOAD_REGION_SIZE;
    uint8_t* uploadPtr = (uint8_t*)uploadBufferPtr;

    uint32_t mask = (SDL_GetTicks() >> 3) & 0x1FF;
    mask = mask > 0xFF ? 0x1FF - mask : mask;
    mask = (mask << 16) | (mask << 8) | mask;
    VertexP2C dynamicVertices[] = {
        { -0.5f, -1.0f, 0xFF0000FF|mask },
        {  0.0f,  0.0f, 0xFF00FF00|mask },
        { -1.0f,  0.0f, 0xFFFF0000|mask }
     };
    memcpy(uploadPtr+uploadOffset, dynamicVertices, sizeof(dynamicVertices));
    uploadOffset += sizeof(dynamicVertices);
    
    ...
```

First change - our rendering is now dynamic - triangle changes color with time, also we moved it to the left and up. Second we can now track memory usage in dynamic buffer with `uploadOffset` and `uploadLimit`.

Next lets upload static geometry to dynamic buffer:

```C 
    VkBufferCopy bufferCopyInfo;
    if (frameIndex == 0)
    {
        VertexP2C staticVertices[] = {
            { 0.5f, 0.0f, 0xFF0000FF },
            { 1.0f, 1.0f, 0xFF00FF00 },
            { 0.0f, 1.0f, 0xFFFF0000 }
        };
        memcpy(uploadPtr + uploadOffset, staticVertices, sizeof(staticVertices));
        bufferCopyInfo = (VkBufferCopy){
            .srcOffset = uploadOffset,
            .dstOffset = 0,
            .size = sizeof(staticVertices),
        };
    }

...
```

We also filled in VkBufferCopy structure in order to use it for copy operation from dynamic buffer to static buffer with `vkCmdCopyBuffer`. After copy we place memory barrier - to make result of copy visible to graphics pipeline.

```C
...

    if (frameIndex == 0)
    {
        vkCmdCopyBuffer(commandBuffers[index], uploadBuffer, staticBuffer, 1, &bufferCopyInfo);
        vkCmdPipelineBarrier(commandBuffers[index],
            VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_VERTEX_INPUT_BIT, 0, 
            1, &(VkMemoryBarrier){
                .sType = VK_STRUCTURE_TYPE_MEMORY_BARRIER,
                .srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT,
                .dstAccessMask = VK_ACCESS_VERTEX_ATTRIBUTE_READ_BIT
            },
            0, NULL, 0, NULL
        );
    }
...
```

When everything is ready lets complete our rendering with 2 triangles - one dynamic and one static:

```C++
...

    vkCmdBeginRenderPass(commandBuffers[index], ...)

    vkCmdBindPipeline(commandBuffers[index], VK_PIPELINE_BIND_POINT_GRAPHICS, pipeline);

...

    vkCmdBindVertexBuffers(commandBuffers[index], 0, 1, &uploadBuffer, (VkDeviceSize[]) { 0 });
    vkCmdDraw(commandBuffers[index], 3, 1, 0, 0);
 
    vkCmdBindVertexBuffers(commandBuffers[index], 0, 1, &staticBuffer, (VkDeviceSize[]) { 0 });
    vkCmdDraw(commandBuffers[index], 3, 1, 0, 0);

    vkCmdEndRenderPass(commandBuffers[index]);
    
... 

   ++frameIndex;
   
```

And that's it. Final result is not as attractive as in previous tutorial but nonetheless is very important {{< youtube suAetbhPq4Y >}}

