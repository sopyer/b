---
title: "Minimal Vulkan Sample: Clearing of Backbuffer"
date: 2017-08-12T12:00:00+03:00
---

For a long time I wanted to start learning new low-level API like DirectX and Vulkan. And I procrastinated, as my current job is barely related to rendering and there is little interest in high performance code and high performance rendering specifically. Recently I decided instead of just writing code myself to document my learning exercises in hope that they will be useful to other people. As a side effect I hope it will help with my motivation.

When I started writing my first Vulkan program and it was overwhelming. I took me a week, 1-2 hours everyday before I got my first triangle on screen. Most of time I spent reading about specific functions in documentation. Amount of knowledge required is just extremely big. So I have my doubts when Stephanie Hurlburt(@sehurlburt) asked about beginner friendly tutorials for Vulkan. My first answer was this is impossible. But after I completed my first triangle and got some experience I decided to write minimal Vulkan sample. Currently code is less then 500 lines. I cheated a bit - instead of rendering triangle I decided to just clear a screen :).

## Introduction

Lets begin with tools and libraries required for writing simple Vulkan program. I wrote my example using C in [Visual Studio Community edition](https://www.visualstudio.com/vs/community/). To install 3rd party libraries I used [vcpkg](https://github.com/Microsoft/vcpkg), which greatly simplifies this task for Windows. Surprising fact for me was that to install x64 version of library `:x64-windows` suffix is required. E.g to install both 32 and 64 bit sdl2 library following command `vcpkg intsall sdl2 sdl2:x64-windows` should be executed. I  also downloaded and installed [Vulkan SDK](https://www.lunarg.com/vulkan-sdk/) from LunarG, as at the moment of writing it was not possible to install Vulkan via vcpkg. For platform specific window creation I decided to use [SDL](https://www.libsdl.org/download-2.0.php) as I have found very few code example with Vulkan and SDL. Most know [Vulkan tutorial](https://vulkan-tutorial.com/) uses GLFW as it simplifies Vulkan initialization. In the end code it should be possible to compile my code on Linux as well as there is nothing Windows specific in it.
I'll omit steps to setup project and will go directly to coding. Lets start with sample structure first, full code can be found [here](https://github.com/sopyer/Vulkan/tree/562e653fbbd1f7a83ec050676b744dd082b2ebed):

```c
int main(int argc, char *argv[])
{
    SDL_Init(SDL_INIT_VIDEO | SDL_INIT_EVENTS);

    int run = init_vulkan() && init_window() && init_device() && init_swapchain() && init_render();
    while (run)
    {
        SDL_Event evt;
        while (SDL_PollEvent(&evt))
        {
            if (evt.type == SDL_QUIT)
            {
                run = 0;
            }
        }

        draw_frame();
    }

    fini_render();
    fini_swapchain();
    fini_device();
    fini_window();
    fini_vulkan();

    SDL_Quit();

    return 0;
}
```

Program is quite simple. First we initialize everything we need. Then run main loop. And cleanup after ourselves. As we can see from code above before we start using Vulkan for rendering first we need to do some preparations:

 - create vulkan instance(`init_vulkan`)
 - create window and vulkan surface(`init_window`)
 - create device(`init_device`)
 - create swapchain(`init_swapchain`)
 - create vulkan object required for rendering(`init_render`)

When we have everything in place we can at last render something(`draw_frame`), in our case we will just clear swapchain surfaces.

## Vulkan instance creation

In order to create instance, I decided to cheat a bit again. Most other tutorials do a lot of stuff for this step: request all available extensions, check what is supported etc. I decided to go another way I just define 2 initialization structures - one for release and one for debug with validation enabled. First I try to initialize debug Vulkan instance, if it is not available then try to create release one. This is done for case if we need to run build with validation enabled on system without SDK. If Vulkan instance was initialized with validation then I setup output callback using `vkCreateDebugReportCallbackEXT` function.

```c
const VkApplicationInfo appInfo = {
    .sType = VK_STRUCTURE_TYPE_APPLICATION_INFO,
    .pApplicationName = "Vulkan SDL tutorial",
    .applicationVersion = VK_MAKE_VERSION(1, 0, 0),
    .pEngineName = "No Engine",
    .engineVersion = VK_MAKE_VERSION(1, 0, 0),
    .apiVersion = VK_API_VERSION_1_0,
};

const VkInstanceCreateInfo createInfo = {
    .sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO,
    .pApplicationInfo = &appInfo,
    .enabledExtensionCount = 2,
    .ppEnabledExtensionNames = (const char* const[]) { VK_KHR_SURFACE_EXTENSION_NAME, VK_KHR_WIN32_SURFACE_EXTENSION_NAME },
};

#ifdef VULKAN_ENABLE_LUNARG_VALIDATION
const VkInstanceCreateInfo createInfoLunarGValidation = {
    .sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO,
    .pApplicationInfo = &appInfo,
    .enabledLayerCount = 1,
    .ppEnabledLayerNames = (const char* const[]) { "VK_LAYER_LUNARG_standard_validation" },
    .enabledExtensionCount = 3,
    .ppEnabledExtensionNames = (const char* const[]) { VK_KHR_SURFACE_EXTENSION_NAME, VK_KHR_WIN32_SURFACE_EXTENSION_NAME, VK_EXT_DEBUG_REPORT_EXTENSION_NAME },
};

PFN_vkCreateDebugReportCallbackEXT vkpfn_CreateDebugReportCallbackEXT = 0;
PFN_vkDestroyDebugReportCallbackEXT vkpfn_DestroyDebugReportCallbackEXT = 0;

VkDebugReportCallbackEXT debugCallback = VK_NULL_HANDLE;

static VKAPI_ATTR VkBool32 VKAPI_CALL VulkanReportFunc(
    VkDebugReportFlagsEXT flags,
    VkDebugReportObjectTypeEXT objType,
    uint64_t obj,
    size_t location,
    int32_t code,
    const char* layerPrefix,
    const char* msg,
    void* userData)
{
    printf("VULKAN VALIDATION: %s\n", msg);
    return VK_FALSE;
}

VkDebugReportCallbackCreateInfoEXT debugCallbackCreateInfo = {
    .sType = VK_STRUCTURE_TYPE_DEBUG_REPORT_CALLBACK_CREATE_INFO_EXT,
    .flags = VK_DEBUG_REPORT_ERROR_BIT_EXT | VK_DEBUG_REPORT_WARNING_BIT_EXT,
    .pfnCallback = VulkanReportFunc,
};
#endif

VkInstance instance = VK_NULL_HANDLE;

int init_vulkan()
{
    VkResult result = VK_ERROR_INITIALIZATION_FAILED;
#ifdef VULKAN_ENABLE_LUNARG_VALIDATION
    result = vkCreateInstance(&createInfoLunarGValidation, 0, &instance);
    if (result == VK_SUCCESS)
    {
        vkpfn_CreateDebugReportCallbackEXT =
            (PFN_vkCreateDebugReportCallbackEXT)vkGetInstanceProcAddr(instance, "vkCreateDebugReportCallbackEXT");
        vkpfn_DestroyDebugReportCallbackEXT =
            (PFN_vkDestroyDebugReportCallbackEXT)vkGetInstanceProcAddr(instance, "vkDestroyDebugReportCallbackEXT");
        if (vkpfn_CreateDebugReportCallbackEXT && vkpfn_DestroyDebugReportCallbackEXT)
        {
            vkpfn_CreateDebugReportCallbackEXT(instance, &debugCallbackCreateInfo, 0, &debugCallback);
        }
    }
#endif
    if (result != VK_SUCCESS)
    {
        result = vkCreateInstance(&createInfo, 0, &instance);
    }

    return result == VK_SUCCESS;
}

void fini_vulkan()
{
#ifdef VULKAN_ENABLE_LUNARG_VALIDATION
    if (vkpfn_DestroyDebugReportCallbackEXT && debugCallback)
    {
        vkpfn_DestroyDebugReportCallbackEXT(instance, debugCallback, 0);
    }
#endif
    vkDestroyInstance(instance, 0);
}
```

## Window and surface creation

This is easy part, we create SDL window and Vulkan surface using platform specific code. Also window is not resizable and its dimensions are fixed.

```c
SDL_Window* window = 0;
VkWin32SurfaceCreateInfoKHR surfaceCreateInfo = { VK_STRUCTURE_TYPE_WIN32_SURFACE_CREATE_INFO_KHR };
VkSurfaceKHR surface = VK_NULL_HANDLE;
uint32_t width = 1280;
uint32_t height = 720;

int init_window()
{
    SDL_Window* window = SDL_CreateWindow(
        "Vulkan Sample",
        SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED,
        width, height,
        0
    );

    SDL_SysWMinfo info;
    SDL_VERSION(&info.version);
    SDL_GetWindowWMInfo(window, &info);

    surfaceCreateInfo.hinstance = GetModuleHandle(0);
    surfaceCreateInfo.hwnd = info.info.win.window;

    VkResult result = vkCreateWin32SurfaceKHR(instance, &surfaceCreateInfo, NULL, &surface);

    return window != 0 && result == VK_SUCCESS;
}

void fini_window()
{
    vkDestroySurfaceKHR(instance, surface, 0);
    SDL_DestroyWindow(window);
}
```

## Picking physical device

In order to create logical device we need first evaluate all physical devices in system and pick one that suits our needs. This is done using `vkEnumeratePhysicalDevices` functions, which returns handles for physical devices. If is called with argument `pPhysicalDevices` set to `NULL` then it returns number of available devices in system. In order to avoid dynamic allocations and simplify code I decide to limit number of devices we check to `MAX_DEVICE_COUNT`. Very rarely user will have more then that probably most common count will be 2 devices: 1 integrated and 1 discrete GPU. My notebook report only 1 discrete GPU.
After device handles are retrieved then we need to find device that suits our needs. Also we need to check which queue families are present on device. We later will use queue family index for logical device creation. In my case I just looked for a device with graphics queue which supports present as well. Please note that present queue and graphics queue can be different, but at the moment I just ignore this fact. In order for program to work graphics queue **must** support present.
Better approach to device selection will be to define profiles in a way similar to DirectX 11 and select device which matches highest profile. We can also always prefer discrete GPU over integrated one. But I think it will be always better if game expose supported GPU list in
 menu and allow gamer to choose preferred one. As vendor id and device id are exposed we can always initialize previously selected GPU or try to search for new default if something goes wrong.
Please note I limited queue count to `MAX_QUEUE_COUNT` which is 4. I suspect that currently there can be no more then 4 queue types: transfer, graphics, compute, graphics+compute. Also in [Vulkan hardware database](http://vulkan.gpuinfo.org/) I saw at most 3 different queue families. Judging from unique flags in `VkQueueFlagBits` enumeration there can be no more then 16 unique queue families. But actual count will be less as graphics and/or compute queues always support transfer queue capabilities. Also I suspect there are not or will not be queue families differentiated by sparsity bit within same gpu. Another fact we should pay attention to is: queue family count is different from total queue count. It is possible to have multiple queues of the same family. This is probably is obvious but better to mention it explicitly.

```c
int init_device()
{ 
    uint32_t physicalDeviceCount;
    VkPhysicalDevice deviceHandles[MAX_DEVICE_COUNT];
    VkQueueFamilyProperties queueFamilyProperties[MAX_QUEUE_COUNT];

    vkEnumeratePhysicalDevices(instance, &physicalDeviceCount, 0);
    physicalDeviceCount = physicalDeviceCount > MAX_DEVICE_COUNT ? MAX_DEVICE_COUNT : physicalDeviceCount;
    vkEnumeratePhysicalDevices(instance, &physicalDeviceCount, deviceHandles);

    for (uint32_t i = 0; i < physicalDeviceCount; ++i)
    {
        uint32_t queueFamilyCount = 0;
        vkGetPhysicalDeviceQueueFamilyProperties(deviceHandles[i], &queueFamilyCount, NULL);
        queueFamilyCount = queueFamilyCount > MAX_QUEUE_COUNT ? MAX_QUEUE_COUNT : queueFamilyCount;
        vkGetPhysicalDeviceQueueFamilyProperties(deviceHandles[i], &queueFamilyCount, queueFamilyProperties);

        for (uint32_t j = 0; j < queueFamilyCount; ++j) {

            VkBool32 supportsPresent = VK_FALSE;
            vkGetPhysicalDeviceSurfaceSupportKHR(deviceHandles[i], j, surface, &supportsPresent);

            if (supportsPresent && (queueFamilyProperties[j].queueFlags & VK_QUEUE_GRAPHICS_BIT))
            {
                queueFamilyIndex = j;
                physicalDevice = deviceHandles[i];
                break;
            }
        }

        if (physicalDevice)
        {
            break;
        }
    }
	
	...
}
```

## Logical device creation

Logical device creation is trivial part, just use found physical device handle and queue index for logical device creation. Then get all requested queues from device, which we will later use for command submissions. Please note that we require `VK_KHR_SWAPCHAIN` extension to be supported without checking if this is a case. This simplifies life quite a bit.

```c
VkPhysicalDevice physicalDevice = VK_NULL_HANDLE;
uint32_t queueFamilyIndex;
VkQueue queue;
VkDevice device = VK_NULL_HANDLE;

VkDeviceCreateInfo deviceCreateInfo = {
    .sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO,
    .queueCreateInfoCount = 1,
    .pQueueCreateInfos = 0,
    .enabledLayerCount = 0,
    .ppEnabledLayerNames = 0,
    .enabledExtensionCount = 1,
    .ppEnabledExtensionNames = (const char* const[]) { VK_KHR_SWAPCHAIN_EXTENSION_NAME },
    .pEnabledFeatures = 0,
};

int init_device()
{
	...

    deviceCreateInfo.pQueueCreateInfos = (VkDeviceQueueCreateInfo[]) {
        {
            .sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO,
            .queueFamilyIndex = queueFamilyIndex,
            .queueCount = 1,
            .pQueuePriorities = (const float[]) { 1.0f }
        }
    };
    VkResult result = vkCreateDevice(physicalDevice, &deviceCreateInfo, NULL, &device);

    vkGetDeviceQueue(device, queueFamilyIndex, 0, &queue);

    return result == VK_SUCCESS;
}

void fini_device()
{
    vkDestroyDevice(device, 0);
}
```

## Swapchain creation

Swapchain creation in my example is also a bit simplified. In order to simplify implementation first available format. If format is `VK_FORMAT_UNDEFINED `, then we can use any supported format. I use `VK_FORMAT_B8G8R8A8_UNORM` as default.

```c
    //Use first available format
    uint32_t formatCount = 1;
    vkGetPhysicalDeviceSurfaceFormatsKHR(physicalDevice, surface, &formatCount, 0); // suppress validation layer
    vkGetPhysicalDeviceSurfaceFormatsKHR(physicalDevice, surface, &formatCount, &surfaceFormat);
    surfaceFormat.format = surfaceFormat.format == VK_FORMAT_UNDEFINED ? VK_FORMAT_B8G8R8A8_UNORM : surfaceFormat.format;
```

Next step is to select presentation mode. Mode `VK_PRESENT_MODE_FIFO_KHR` is always supported, I use it as double buffering. But I decided to support `VK_PRESENT_MODE_MAILBOX_KHR`, which is equivalent to triple buffering. Both modes use vertical synchronization. Another moment to note: specification now support 6 presentation modes at most, so I query no more then `MAX_PRESENT_MODE_COUNT` at most in order to avoid dynamic allocations. If you want to learn more details about presentation modes [Intel Vulkan swapchain tutorial](https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-part-2#_Toc445674479) has excellent visualizations.

```c
    uint32_t presentModeCount = 0;
    vkGetPhysicalDeviceSurfacePresentModesKHR(physicalDevice, surface, &presentModeCount, NULL);
    VkPresentModeKHR presentModes[MAX_PRESENT_MODE_COUNT];
    presentModeCount = presentModeCount > MAX_PRESENT_MODE_COUNT ? MAX_PRESENT_MODE_COUNT : presentModeCount;
    vkGetPhysicalDeviceSurfacePresentModesKHR(physicalDevice, surface, &presentModeCount, presentModes);    

	VkPresentModeKHR presentMode = VK_PRESENT_MODE_FIFO_KHR;   // always supported.
    for (uint32_t i = 0; i < presentModeCount; ++i)
    {
        if (presentModes[i] == VK_PRESENT_MODE_MAILBOX_KHR)
        {
            presentMode = VK_PRESENT_MODE_MAILBOX_KHR;
            break;
        }
    }
    swapchainImageCount = presentMode == VK_PRESENT_MODE_MAILBOX_KHR ? PRESENT_MODE_MAILBOX_IMAGE_COUNT : PRESENT_MODE_DEFAULT_IMAGE_COUNT;
```

In order to create swapchain one more step si required to define swapchain image dimensions. Surface capabilities for device can be queried with `vkGetPhysicalDeviceSurfaceCapabilitiesKHR` function call. Later its information will be used in swapchain creation. Compositing manager of OS in theory can support different resolutions then original window size. This is communicated via assigning `UINT32_MAX` to  `currentExtent.width` and `currentExtent.height` fields of `VkSurfaceCapabilitiesKHR` structure. In this case window dimensions are clamped to minimum and maximum supported dimensions of swapchain.

```c
    VkSurfaceCapabilitiesKHR surfaceCapabilities;
    vkGetPhysicalDeviceSurfaceCapabilitiesKHR(physicalDevice, surface, &surfaceCapabilities);

    swapchainExtent = surfaceCapabilities.currentExtent;
    if (swapchainExtent.width == UINT32_MAX)
    {
        swapchainExtent.width = clamp_u32(width, surfaceCapabilities.minImageExtent.width, surfaceCapabilities.maxImageExtent.width);
        swapchainExtent.height = clamp_u32(height, surfaceCapabilities.minImageExtent.height, surfaceCapabilities.maxImageExtent.height);
    }
```

Final step is swapchain creation. Two thing are of interest here: fields `imageUsage` and `imageSharingMode`. Please note that only `VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT` is supported by default for swapchain images. But `VK_IMAGE_USAGE_TRANSFER_DST_BIT` is required for clearing image.  Also we use `VK_SHARING_MODE_EXCLUSIVE` this means that our images can not be used concurrently by 2 queues at the same time and application is responsible for ownership transfer between queues. As there is only 1 queue then there is no reason to use swapchain images in different queues.
Rest of code is pretty straightforward:

```c
VkSwapchainKHR swapchain;
uint32_t swapchainImageCount;
VkImage swapchainImages[MAX_SWAPCHAIN_IMAGES];
VkExtent2D swapchainExtent;
VkSurfaceFormatKHR surfaceFormat;

int init_swapchain()
{
	...

    VkSwapchainCreateInfoKHR swapChainCreateInfo = {
        .sType = VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR,
        .surface = surface,
        .minImageCount = swapchainImageCount,
        .imageFormat = surfaceFormat.format,
        .imageColorSpace = surfaceFormat.colorSpace,
        .imageExtent = swapchainExtent,
        .imageArrayLayers = 1, // 2 for stereo
        .imageUsage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | VK_IMAGE_USAGE_TRANSFER_DST_BIT,
        .imageSharingMode = VK_SHARING_MODE_EXCLUSIVE,
        .preTransform = surfaceCapabilities.currentTransform,
        .compositeAlpha = VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR,
        .presentMode = presentMode,
        .clipped = VK_TRUE,
    };

    VkResult result = vkCreateSwapchainKHR(device, &swapChainCreateInfo, 0, &swapchain);
    if (result != VK_SUCCESS)
    {
        return 0;
    }

    vkGetSwapchainImagesKHR(device, swapchain, &swapchainImageCount, NULL);
    vkGetSwapchainImagesKHR(device, swapchain, &swapchainImageCount, swapchainImages);

    return 1;
}

void fini_swapchain()
{
    vkDestroySwapchainKHR(device, swapchain, 0);
}
```

## Simple rendering: clear backbuffer

Last but not least is rendering code. Rendering in Vulkan is done recording commands to command buffers and submitting command buffers to device queues.
This is simple part. Much more complex part is synchronization between different concurrent part of graphics pipeline. Some basic tutorials take easy route and record static command buffers for each swapchain image. But games usually have dynamic content and command buffers are recorder every frame. And we need to synchronize recording of those buffers. Also we need to synchronize presentation engine and GPU rendering. There is very elegant and simple idea best described in [Adam Sawicki blog post](http://asawicki.info/news_1647_vulkan_bits_and_pieces_synchronization_in_a_frame.html),
 which allows achieve both. I developed similar technique for uploading dynamic, and used it in my other projects. 
Idea is simple: we allow at most N frames it flight and wait for oldest frame before rendering new one. So we reduce our synchronization problem to frame synchronization problem. As long as every frame uses its own set of resources we can rely on Vulkan implicit synchronization guaranties for those resources. We still need to synchronize graphics pipeline for resource usage within frame but that is a different story.
Here is rendering code:

```c
uint32_t frameIndex = 0;
VkCommandPool commandPool;
VkCommandBuffer commandBuffers[FRAME_COUNT];
VkFence frameFences[FRAME_COUNT]; // Create with VK_FENCE_CREATE_SIGNALED_BIT.
VkSemaphore imageAvailableSemaphores[FRAME_COUNT];
VkSemaphore renderFinishedSemaphores[FRAME_COUNT];

int init_render()
{
    VkCommandPoolCreateInfo commandPoolCreateInfo = {
        .sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO,
        .flags = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT,
        .queueFamilyIndex = queueFamilyIndex,
    };

    vkCreateCommandPool(device, &commandPoolCreateInfo, 0, &commandPool);

    VkCommandBufferAllocateInfo commandBufferAllocInfo = {
        .sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO,
        .commandPool = commandPool,
        .level = VK_COMMAND_BUFFER_LEVEL_PRIMARY,
        .commandBufferCount = FRAME_COUNT,
    };

    vkAllocateCommandBuffers(device, &commandBufferAllocInfo, commandBuffers);

    VkSemaphoreCreateInfo semaphoreCreateInfo = { VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO };

    vkCreateSemaphore(device, &semaphoreCreateInfo, 0, &imageAvailableSemaphores[0]);
    vkCreateSemaphore(device, &semaphoreCreateInfo, 0, &imageAvailableSemaphores[1]);
    vkCreateSemaphore(device, &semaphoreCreateInfo, 0, &renderFinishedSemaphores[0]);
    vkCreateSemaphore(device, &semaphoreCreateInfo, 0, &renderFinishedSemaphores[1]);

    VkFenceCreateInfo fenceCreateInfo = {
        .sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO,
        .flags = VK_FENCE_CREATE_SIGNALED_BIT
    };

    vkCreateFence(device, &fenceCreateInfo, 0, &frameFences[0]);
    vkCreateFence(device, &fenceCreateInfo, 0, &frameFences[1]);

    return 1;
}

void fini_render()
{
    vkDeviceWaitIdle(device);
    vkDestroyFence(device, frameFences[0], 0);
    vkDestroyFence(device, frameFences[1], 0);
    vkDestroySemaphore(device, renderFinishedSemaphores[0], 0);
    vkDestroySemaphore(device, renderFinishedSemaphores[1], 0);
    vkDestroySemaphore(device, imageAvailableSemaphores[0], 0);
    vkDestroySemaphore(device, imageAvailableSemaphores[1], 0);
    vkDestroyCommandPool(device, commandPool, 0);
}

void draw_frame()
{
    uint32_t index = (frameIndex++) % FRAME_COUNT;
    vkWaitForFences(device, 1, &frameFences[index], VK_TRUE, UINT64_MAX);
    vkResetFences(device, 1, &frameFences[index]);

    uint32_t imageIndex;
    vkAcquireNextImageKHR(device, swapchain, UINT64_MAX, imageAvailableSemaphores[index], VK_NULL_HANDLE, &imageIndex);

    VkCommandBufferBeginInfo beginInfo = {
        .sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO,
        .flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT
    };
    vkBeginCommandBuffer(commandBuffers[index], &beginInfo);

	...

    vkEndCommandBuffer(commandBuffers[index]);

    VkSubmitInfo submitInfo = {
        .sType = VK_STRUCTURE_TYPE_SUBMIT_INFO,
        .waitSemaphoreCount = 1,
        .pWaitSemaphores = &imageAvailableSemaphores[index],
        .pWaitDstStageMask = (VkPipelineStageFlags[]) { VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT },
        .commandBufferCount = 1,
        .pCommandBuffers = &commandBuffers[index],
        .signalSemaphoreCount = 1,
        .pSignalSemaphores = &renderFinishedSemaphores[index],
    };
    vkQueueSubmit(queue, 1, &submitInfo, frameFences[index]);

    VkPresentInfoKHR presentInfo = {
        .sType = VK_STRUCTURE_TYPE_PRESENT_INFO_KHR,
        .waitSemaphoreCount = 1,
        .pWaitSemaphores = &renderFinishedSemaphores[index],
        .swapchainCount = 1,
        .pSwapchains = &swapchain,
        .pImageIndices = &imageIndex,
    };
    vkQueuePresentKHR(queue, &presentInfo);
}
```

Please note `vkDeviceWaitIdle` in `fini_render`. This is very simple synchronization mechanism. It wait for device to complete all work and then resources can be accessed without race conditions. This function simplifies global resource changes like creation or destruction quite a lot.

Rendering code is just 3 calls. But to understand it is required to understand barriers. Barriers can be used in different scenarios. One of usages is to change memory layout of graphics resources. One another usage is transfer of ownership between queues. So code can be extended to support different present queue. But I decided to simplify it and just to mention this possibility. What code does it basically changes underlying memory layout first to `VK_ACCESS_TRANSFER_WRITE_BIT`, which is required for clear operation, and later to `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`, which is required for present. In Vulkan every resource access operation requires specific layout and it is up to programmer to change those layouts, before operation is performed. More detailed explanation can be found in [Vulkan specification](https://www.khronos.org/registry/vulkan/), [GPUOpen post](http://gpuopen.com/vulkan-barriers-explained/) and [Khronos Vulkan barrier examples](https://github.com/KhronosGroup/Vulkan-Docs/wiki/Synchronization-Examples).
Code for clearing:

```c
    VkImageSubresourceRange subResourceRange = {
        .aspectMask = VK_IMAGE_ASPECT_COLOR_BIT,
        .baseMipLevel = 0,
        .levelCount = VK_REMAINING_MIP_LEVELS,
        .baseArrayLayer = 0,
        .layerCount = VK_REMAINING_ARRAY_LAYERS,
    };

    // Change layout of image to be optimal for clearing
    vkCmdPipelineBarrier(commandBuffers[index],
        VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_TRANSFER_BIT,
        0, 0, NULL, 0, NULL, 1,
        &(VkImageMemoryBarrier) {
        .sType = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER,
            .srcAccessMask = 0,
            .dstAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT,
            .oldLayout = VK_IMAGE_LAYOUT_UNDEFINED,
            .newLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
            .srcQueueFamilyIndex = queueFamilyIndex,
            .dstQueueFamilyIndex = queueFamilyIndex,
            .image = swapchainImages[imageIndex],
            .subresourceRange = subResourceRange,
    }
    );
    vkCmdClearColorImage(commandBuffers[index],
        swapchainImages[imageIndex], VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
        &(VkClearColorValue){ {1.0f, 0, 0, 1.0f} }, 1, &subResourceRange
    );
    // Change layout of image to be optimal for presenting
    vkCmdPipelineBarrier(commandBuffers[index],
        VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_BOTTOM_OF_PIPE_BIT,
        0, 0, NULL, 0, NULL, 1,
        &(VkImageMemoryBarrier){
        .sType = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER,
            .srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT,
            .dstAccessMask = VK_ACCESS_MEMORY_READ_BIT,
            .oldLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
            .newLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR,
            .srcQueueFamilyIndex = queueFamilyIndex,
            .dstQueueFamilyIndex = queueFamilyIndex,
            .image = swapchainImages[imageIndex],
            .subresourceRange = subResourceRange,
    }
    );
```

### Links

1. Blog post code. https://github.com/sopyer/Vulkan/tree/562e653fbbd1f7a83ec050676b744dd082b2ebed
2. Khronos Vulkan Registry. https://www.khronos.org/registry/vulkan/
3. Vulkan Tutorial. https://vulkan-tutorial.com/
4. Vulkan barriers explained. http://gpuopen.com/vulkan-barriers-explained/
5. Vulkan Synchronization Examples. https://github.com/KhronosGroup/Vulkan-Docs/wiki/Synchronization-Examples
6. API without Secrets: Introduction to Vulkan. Part 2: Swap Chain#Selecting Presentation Mode. https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-part-2#_Toc445674479
7. Vulkan Bits and Pieces: Synchronization in a Frame. http://asawicki.info/news_1647_vulkan_bits_and_pieces_synchronization_in_a_frame.html
8. VC++ Packaging Tool. https://github.com/Microsoft/vcpkg
9. SDL2. https://www.libsdl.org/download-2.0.php
10. LunarG Vulkan SDK. https://www.lunarg.com/vulkan-sdk/
11. Vulkan Hardware Database. http://vulkan.gpuinfo.org/
12. Visual Studio Community edition. https://www.visualstudio.com/vs/community/

