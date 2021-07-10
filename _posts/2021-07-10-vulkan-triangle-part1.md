---
layout: post
title: A Vulkan Triangle (part 1/3)
subtitle: Getting a window and creating a surface
gh-repo: DraperDanMan/DraperDanMan.github.io
gh-badge: [star, follow]
tags: [vulkan, renderer, cpp]
---

When you start looking into getting started with Vulkan, there isn't an abundance of information like there is for graphics API's like OpenGL or DirectX. (Though DirectX 12 is sort of in a similar boat to Vulkan now with its verbosity) Most posts/people or websites direct you to two places, the Vulkan [specification](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html) and the [vulkan-tutorial](https://vulkan-tutorial.com) website. *I'm fully aware I just did the same thing, but I have some alternatives for you!*

### The Lightweight Process

I want to outline in a light weight approach to get from a basic Window to a rendered triangle in respecting your time and knowing you wouldn't be here without an understanding of programming. The code is just a simple single C++ file with the use of the external library GLFW for managing the Window. Many of the examples out there have you building abstractions right away and I'm not going to lie, I found it confusing trying to figure out exactly where the Vulkan API ends and the user built abstractions had hidden some of the complexity.

Things you're going to need if you want to follow along:

- [The Vulkan SDK](https://www.lunarg.com/vulkan-sdk/)
- [GLFW](https://github.com/glfw/glfw) (Add it to your source somewhere and add the headers to your includes path) 
- A Dev Environment (Currently, I'm using Rider for Unreal & Visual Studio)

Once you have these things you're ready. I should also mention this is part 1 of 3 posts, so if you're following along and part 3 isn't out yet. It might be worth waiting if you want to smash the whole thing out.

At the end of this we should have something like this:
![A Triangle Rendered in Vulkan](/img/VulkanTriangle_goal.png)

#### Step 1: Creating the window

First thing we need to do is create a window where we want to output our rendered image. I guess this is optional if you just want to render into an actual image file. which is totally valid. So I'm going to make use of GLFW to easily create a window, start the framework for our render loop. 

```cpp
#include <assert.h>
#include <GLFW/glfw3.h>
#include <GLFW/glfw3native.h> //Our includes at the top of the file

int main()
{
    printf("Renderer Starting Up\n");

    int returnCode = glfwInit(); //initialize the GLFW api
    assert(returnCode);

    //Create a Window, with the hint that we're going to use our own API to render
    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    GLFWwindow* window = glfwCreateWindow(1024, 720, "TriangleRenderer", 0, 0);
    assert(window);

	//Start the loop until close is requested (X button on the window)
    while (!glfwWindowShouldClose(window))
    {
        glfwPollEvents(); //handle the windows events

        //Do our frames rendering

        glfwWaitEvents();//wait until the window recieves a new event
    }

    glfwDestroyWindow(window); //cleanup
}
```

We now have a window that we can open and close. 

![Picture of our newly created window](/img/VulkanTriangle_blankWindow.png)

To go from this blank window to a cleared screen or a basic triangle there is quite a few things we're going to need to set up. (It's going to take a while)

- Connect to the GPU (either discrete or integrated)
- Create a surface and link that with our Window
- Create a swap chain for swapping framebuffers to then present to the surface.
- Create the framebuffers for the swap chain
- Create a render pass to outline what image we're drawing to
- Import and setup 2 shader programs for handling vertices and fragments
- Create a pipeline layout to indicate where data will be stored throughout the pipeline
- Create a graphics pipeline with all of the configurations we need to draw
- Create a couple of semaphores to manage synchronization
- Allocate some space for a command pool
- Allocate a command buffer from that pool
- Finally bind all the state we need each frame and call draw & present

#### Step 2: Connecting with Vulkan and picking our device

There isn't going to be a many pretty images for a while as we start setting up and configuring everything we need to be able to use our GPU to render something. 
*Normally, I would be programming this all into main and then splitting it out into functions when I need to condense or re-use something. So for the sake of your mental health and mine, I'm going to use the power of future me to already split some of the code into functions. This is still all in the same file and is mostly just splitting off code to simplify the main() function to be easier to follow.*

##### VkInstance

This is our connection to the Vulkan driver. During its creation we can attach some meta information about the program like the app name, engine name, app version. But the there is 3 things that have a meaningful effect here: API version, API layers and API extensions.  Each of these have an effect on what features we need.

The API version is pretty straight forward, you want to set it to whatever your target platform/s supports. From what I could find scattered online its mostly older android devices that don't support the latest driver.  So for this code we're using **API Version 1.2** which is the latest version as of writing.

The API Layers and extensions are interesting. Layers are explicitly run before your code hits the driver, so they can do things like modify requests or produce errors or warnings before the code hits and crashes the driver. Where as extensions add actual implementation details to either you application code, a layer or the driver itself. The most common extensions are things like enabling platform integration, for example support for rendering to a **Windows** or **Linux** window.

Alright lets actually create the Instance.

```cpp
#include <vulkan/vulkan.h> //At the top of the file (C Header)

VkInstance CreateVulkanInstance()
{
    VkInstanceCreateInfo createInfo = { VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO };
    
    VkApplicationInfo appInfo = { VK_STRUCTURE_TYPE_APPLICATION_INFO };
    appInfo.apiVersion = VK_API_VERSION_1_2;

    createInfo.pApplicationInfo = &appInfo;

#ifdef _DEBUG //you can Only enable the validation layer for debug builds if you want.
    const char* debugVkLayers[] =
    {
        "VK_LAYER_KHRONOS_validation",
    };
    
    createInfo.ppEnabledLayerNames = debugVkLayers;
    createInfo.enabledLayerCount = sizeof(debugVkLayers) / sizeof(debugVkLayers[0]);
#endif

    //There is a handy enum with all the available extensions
    const char* extensions[] = 
    {
        VK_KHR_SURFACE_EXTENSION_NAME,
//(If you're making a multiplatfom renderer, then you can define what platforms need)
//#ifdef Win32 
        VK_KHR_WIN32_SURFACE_EXTENSION_NAME,
//#endif
        VK_EXT_DEBUG_REPORT_EXTENSION_NAME, //we're going to use this soon
    };

    createInfo.ppEnabledExtensionNames = extensions;
    createInfo.enabledExtensionCount = sizeof(extensions) / sizeof(extensions[0]);

    //Create the Vulkan Instance
    VkInstance instance = nullptr;
    //the 0 in this function is for where we can pass in a custom memory allocator!
    VkResult result = vkCreateInstance(&createInfo, 0, &instance));
	assert(result == VK_SUCCESS);
    
    return instance;
}
```

Okay our first bit of Vulkan code. Hopefully this is pretty straight forward. Note here at the top I'm using the pure C header, rather than the cpp header. This is mainly because all the docs refer to the c headers not the cpp headers. Also get used to the way we create these structures fill them with info and then pass them into the Vulkan API. In almost all places that I use a `0` in a vk function, that is where you would pass in a custom memory allocator, which is amazing as an option. if you want to test it out without your own allocator there is the open source memory allocator called [Vulkan Memory Allocator](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator)

I'm not super keen on typing VkResult and asserting it **every** time I call a vk function so I'm going to make a quick macro at the top of the file after includes, to simplify the check step. (This is all one line in case your browser wraps this, you can split it over multiple lines if you like with a backslash \ )

```cpp
#define VK_CHECK(call) do { VkResult result = call;assert(result == VK_SUCCESS);} while(0)
```

Now when we call most vk functions we can just wrap it in `VK_CHECK(vkCreateInstance(&createInfo, 0, &instance));`

So in our main function we can call our `CreateVulkanInstance` function, we should also clean up after we're done at the end too.

```cpp
    //... near the top of main
	int returnCode = glfwInit(); //initialize the GLFW api
    assert(returnCode);

	//Create the Vulkan Instance
    VkInstance vulkanInstance = CreateVulkanInstance();
    assert(vulkanInstance);

    //Create a Window, with the hint that we're going to use our own API to render
    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);

```

```cpp
	//... end of main
    vkDestroyInstance(vulkanInstance, 0); //clean up VulkanInstance
    
    glfwDestroyWindow(window); //cleanup
```



##### VkDevice

Now that we can talk to Vulkan properly we need to find the physical device we want to use and then create a logical representation (a VkDevice). This can be done with a lot of assumptions, but really, before you try to use the device you should check that it can do what you want. It is entirely possible to have a device that doesn't have the capability to render graphics. It's also entirely possible to accidently select an integrated device instead of your discrete GPU. So we're going to at least do some checking to make sure we pick the right things like being able to render and also that its the discrete GPU.

###### finding our device

```cpp
VkPhysicalDevice FindPhysicalDevice(VkInstance instance, VkPhysicalDevice* physicalDevices)
{
    //grab all the physical devices that support vulkan (max 16 should be plenty)
    //physicalDeviceCount will be overriden with the actual amount if its less
    uint32_t physicalDeviceCount = 16;
    VK_CHECK(vkEnumeratePhysicalDevices(instance, &physicalDeviceCount, physicalDevices));

    for (uint32_t i = 0; i < physicalDeviceCount; ++i)
    {
        //get information about the device, name, type, etc.
        VkPhysicalDeviceProperties props;
        vkGetPhysicalDeviceProperties(physicalDevices[i], &props);

        if (props.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU)
        {
            printf("Found discrete GPU: %s\n", props.deviceName);
            return physicalDevices[i];
        }
    }

    //if we didn't find a discrete GPU then fallback to the first item
    //in all cases I tested slot 0 was always the integrated GPU. 
	//but that could just be coincidence.
    if (physicalDeviceCount > 0)
    {
        VkPhysicalDeviceProperties props;
        vkGetPhysicalDeviceProperties(physicalDevices[0], &props);
        printf("No discrete GPU found, falling back to GPU: %s\n", props.deviceName);
        return physicalDevices[0];
    }

    printf("NO GPU SUPPORTING VULKAN FOUND");
    return 0;
}
```

This physical device is going to be the one we use to create our logical device. However, the logical device is going to want to know what queues we're going to be using on that device. There are 3 main queues that you might be touching in your app, those are:

- **TRANSFER** - used for sending and retrieving GPU only memory.
- **GRAPHICS** - used for running shading programs and rendering.
- **COMPUTE** - used for running general purpose programs.

For now we're just going to be using the GRAPHICS queue. Though I'll highly likely talk about the other two queues in the future. **note: it is entirely possible that the device doesn't have a dedicated queue for transfer or compute.** 

###### setting up our logical device

```cpp
uint32_t GetGraphicsQueueFamily(VkPhysicalDevice device)
{
    //we'll create some empty structs for the function to fill. 
    VkQueueFamilyProperties queues[16];
    uint32_t queuesCount = ARRAYSIZE(queues);
    vkGetPhysicalDeviceQueueFamilyProperties(device, &queuesCount, queues);
    for (uint32_t i = 0; i < queuesCount; i++)
    {
        //check if this queue is a graphics queue
        if (queues[i].queueFlags & VK_QUEUE_GRAPHICS_BIT)
            return i;
    }

    printf("No Graphics Queues found, is this a Compute only device?");
    return VK_QUEUE_FAMILY_IGNORED;
}
```

Finally we can create our logical device that will be used from this point forward. An interesting point here is that when you create a device there is extensions that can be enabled here too. I imagine the choice here was to limit the overhead if you're enabling a device for a very specific purpose. eg, running a compute program without any need for graphics. Because we plan on using graphics and a swap chain we have to specifically enable the swap chain extension. Here we also specify what device queues have priority over the hardware, though we'll only be using the graphics queue so that will be the only queue with any priority.

```cpp
VkDevice CreateDevice(VkInstance instance, VkPhysicalDevice physicalDevice, uint32_t* familyIndex)
{
    // setup the priorities for the queues we're going to use (just our selected queue)
    float queuePriorties[] = { 1.0f };
    VkDeviceQueueCreateInfo deviceQueueCreateInfo = { VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO };
    deviceQueueCreateInfo.queueFamilyIndex = *familyIndex;
    deviceQueueCreateInfo.queueCount = 1;
    deviceQueueCreateInfo.pQueuePriorities = queuePriorties;

    //extensions to enable for this device
    const char* extensions[] =
    {
        VK_KHR_SWAPCHAIN_EXTENSION_NAME,
    };

    //all we need is out queue priority and our extensions
    VkDeviceCreateInfo deviceCreateInfo = { VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO };
    deviceCreateInfo.queueCreateInfoCount = 1;
    deviceCreateInfo.pQueueCreateInfos = &deviceQueueCreateInfo;
    deviceCreateInfo.ppEnabledExtensionNames = extensions;
    deviceCreateInfo.enabledExtensionCount = ARRAYSIZE(extensions);

    VkDevice device;
    VK_CHECK(vkCreateDevice(physicalDevice, &deviceCreateInfo, 0, &device));

    return device;
}
```

We can now hook this all together in our main function now. Not that we're close to seeing anything on screen yet.

###### setting it up in main

```cpp
 //... main
	//Create the Vulkan Instance
    VkInstance vulkanInstance = CreateVulkanInstance();
    assert(vulkanInstance);

	VkPhysicalDevice physicalDevices[16];

    //Fetch the physical GPU we want to use
    VkPhysicalDevice physicalDevice = PickPhysicalDevice(vulkanInstance, physicalDevices);
    assert(physicalDevice);

	//Create our logical device with out graphics queue
	uint32_t deviceFamilyIndex = GetGraphicsQueueFamily(physicalDevice);
    VkDevice vulkanDevice = CreateDevice(vulkanInstance, physicalDevice, &deviceFamilyIndex);
    assert(vulkanDevice);
```

```cpp
//...before we clean up the instance.
	vkDestroyDevice(vulkanDevice, 0); //clean up the logical device
	
    vkDestroyInstance(vulkanInstance, 0); //clean up VulkanInstance
```

You can run the program now to test that you are indeed finding a device and setting it up. If there is any issues our asserts should catch them. This does however lead into what is an invaluable tool during Vulkan development. The validation layer. The astute reader may have remembered that we included the validation layer in the layers during the VkInstance creation. The problem is that it requires an extra step to log those messages out to the console, for which we included the Debug Report extension. So lets make a quick detour to setup our debug report callback so if we hit any errors from this point on we can see the validation layer tell us about them and give us useful links to where we can learn more about why that error is occurring. 

##### Debug Callback

Strangely, the vk function for setting up a callback isn't readily available for us we actually need to dynamically link the function with `vkGetInstanceProcAddr` so that we can call it. We'll need to keep a reference to our callback so that we can destroy it when shutting down our app.

###### setting up our callback

```cpp
//The function that gets called by vulkan/validation layer each time it tries to log
VkBool32 VulkanDebugReportCallback(VkDebugReportFlagsEXT flags, VkDebugReportObjectTypeEXT objectType, uint64_t object, size_t location, int32_t messageCode, const char* pLayerPrefix, const char* pMessage, void* pUserData)
{
    //Lets prefix the message with its type for easy viewing
    const char* type =
        (flags & VK_DEBUG_REPORT_ERROR_BIT_EXT) ? "ERROR" :
        (flags & (VK_DEBUG_REPORT_WARNING_BIT_EXT | VK_DEBUG_REPORT_PERFORMANCE_WARNING_BIT_EXT)) ? "WARNING" :
        "INFO"; //last case

    char message[4096];
    snprintf(message, ARRAYSIZE(message),"%s: %s\n", type, pMessage);
    
    printf("%s", message);

#ifdef _WIN32
    OutputDebugStringA(message);
#endif

    //If its an error lets assert.
    if (flags & VK_DEBUG_REPORT_ERROR_BIT_EXT)
        assert(!"Validation Error encounted");
    return VK_FALSE;
}
```

```cpp
VkDebugReportCallbackEXT CreateVulkanDebugCallback(VkInstance instance)
{
    //when we create the callback we can pass it what messages we want to filter.
    //lets grab almost all of them
    VkDebugReportCallbackCreateInfoEXT createInfo = { VK_STRUCTURE_TYPE_DEBUG_REPORT_CALLBACK_CREATE_INFO_EXT };
    createInfo.flags = VK_DEBUG_REPORT_WARNING_BIT_EXT | VK_DEBUG_REPORT_PERFORMANCE_WARNING_BIT_EXT | VK_DEBUG_REPORT_ERROR_BIT_EXT | VK_DEBUG_REPORT_DEBUG_BIT_EXT;
    //point to the function that the callback will run. (The function we created above)
    createInfo.pfnCallback = VulkanDebugReportCallback;

    //To use an extension we need to dynamically link to it 
    PFN_vkCreateDebugReportCallbackEXT vkCreateDebugReportCallbackEXT = (PFN_vkCreateDebugReportCallbackEXT)vkGetInstanceProcAddr(instance, "vkCreateDebugReportCallbackEXT");

    VkDebugReportCallbackEXT callback = 0;
    VK_CHECK(vkCreateDebugReportCallbackEXT(instance, &createInfo, 0, &callback));
    return callback;
}
```

Now we can create the debug callback in our main function. I like to create it right after I enable the instance. So that I can catch any potential issues with the creation of the logical device. 

```cpp
//Create the Vulkan Instance
VkInstance vulkanInstance = CreateVulkanInstance();
assert(vulkanInstance);

//Create our debug callback right after our instance.
VkDebugReportCallbackEXT vulkanDebugCallback = CreateVulkanDebugCallback(vulkanInstance);
```

```cpp
//... near the end of main

//to destroy the callback we also need to dynamically link to the destroy function too.
PFN_vkDestroyDebugReportCallbackEXT vkDestroyDebugReportCallbackEXT = (PFN_vkDestroyDebugReportCallbackEXT)vkGetInstanceProcAddr(vulkanInstance, "vkDestroyDebugReportCallbackEXT");
    vkDestroyDebugReportCallbackEXT(vulkanInstance, vulkanDebugCallback, 0);

//before we destroy the instance.
vkDestroyInstance(vulkanInstance, 0);
```

Now if we run the program we should get a bunch of INFO logs about all the available extensions and that validation has been turned on. We might even get a couple of warnings about our device creation that we should be requesting and checking more information before device creation. (we'll ignore it though, haha)

#### Step 3 : Creating a Surface and connecting it to the window

Now that we have a solid connection to Vulkan, we can start preparing our presentation surface. The surface is essentially the middle layer between the Vulkan device and the operating system. this layer is set up differently based on the platform you're going to be outputting too. If your renderer is Vulkan only, this is probably the only place in your code that you will need platform defines to create the surface for each type of device, the rest of the API is platform agnostic. This example is being built on windows so We're using the windows surface extension and creating the surface that links with a HWND window handle.

This part is really straight forward, but We'll create a couple of other functions we're going to need later that use the surface.

```cpp
VkSurfaceKHR CreateSurface(VkInstance instance, GLFWwindow* window)
{
//#ifdef Win32 //I'm using a win32 define here for windows, totally optional
    VkWin32SurfaceCreateInfoKHR surfaceCreateInfo = { VK_STRUCTURE_TYPE_WIN32_SURFACE_CREATE_INFO_KHR };
    surfaceCreateInfo.hinstance = GetModuleHandle(0); //GLFWnative get program instance
    surfaceCreateInfo.hwnd = glfwGetWin32Window(window); //GLFW get window handle
    VkSurfaceKHR surface = 0;
    VK_CHECK(vkCreateWin32SurfaeKHR(instance, &surfaceCreateInfo, 0, &surface));
    return surface;
//#else
//#error Unsupported Platform
//#endif
}
```

Done. That will create us a surface and link it with our window. Before we wrap up this part of the series of posts. We're going to create some functions we're going to use later. We will need to check that the surface we created actually supports presenting to screen. Something that is not guaranteed. That said, I've yet to find a device that doesn't support presenting in 2021... We'll also grab the first supported color format too.

```cpp
VkSurfaceFormatKHR GetSurfaceFormat(VkPhysicalDevice physicalDevice, VkSurfaceKHR surface)
{
    uint32_t surfaceFormatCount = 8;
    VkSurfaceFormatKHR surfaceFormats[8];
    VK_CHECK(vkGetPhysicalDeviceSurfaceFormatsKHR(physicalDevice, surface, &surfaceFormatCount, surfaceFormats));

    for (uint32_t i = 0; i < surfaceFormatCount; ++i)
    {
        printf("Suported Format: %i with color space: %i \n", surfaceFormats[i].format, surfaceFormats[i].colorSpace);
    }
    return surfaceFormats[0];
}

VkSurfaceCapabilitiesKHR GetSurfaceCapabilities(VkPhysicalDevice physicalDevice, VkSurfaceKHR surface)
{
    VkSurfaceCapabilitiesKHR caps;
    VK_CHECK(vkGetPhysicalDeviceSurfaceCapabilitiesKHR(physicalDevice, surface, &caps));

    return caps;
}
```

Alright we can now call these functions in our main program. We'll call our create surface function, because we actually need the surface before we can check if it supports presenting to the screen. The color format will come in handy when we start creating out images for rendering to, while our surface capabilities will let us know if there is a functionality we don't support. 

```cpp
//Create a Windows window..
//after we assert(window);

//Create a surface and attach it to the window
VkSurfaceKHR vulkanSurface = CreateSurface(vulkanInstance, window);
assert(vulkanSurface);

//Does the physical device support actually writing to the surface.
VkBool32 surfaceSupported;
VK_CHECK(vkGetPhysicalDeviceSurfaceSupportKHR(physicalDevice, deviceFamilyIndex, vulkanSurface, &surfaceSupported));

printf("Suported Surface: %s \n", surfaceSupported ? "True" : "Bool");

VkSurfaceFormatKHR surfaceFormat = GetSurfaceFormat(physicalDevice, vulkanSurface);

VkSurfaceCapabilitiesKHR surfaceCapabilities = GetSurfaceCapabilities(physicalDevice, vulkanSurface); //we'll use these shortly

//... rest of main

//... before we call vkDestroyDevice(vulkanDevice, 0);
vkDestroySurfaceKHR(vulkanInstance, vulkanSurface, 0);

Yay, we have created our surface, we have something we can now present into! 

#### Part 1 Wrap Up

Originally this was going to be just a huge post, but to be honest this has taken long enough to write in of itself. So I'm going to split this tutorial of sorts into 3 posts because we're about a third of the way to rendering a triangle in Vulkan. While we're already significantly more lines than Open GL or DirectX (except dx12), this step is probably the least malleable compared with the rest of the API, there are still things here that we have full control over. eg. the physical devices we use, the number of queues we make use of on the device, as well as how we connect the surface up with our end point. 

For example, the optimal way to use data on a GPU is to use the dedicated transfer queue to push or pull data to and from the GPU in between graphics programs, or to use all available compute queues to run several programs interleaved with our main graphics processing.  
We also have control over creating logical devices for each attached GPU available to the application. There hasn't been many applications that mix these two devices. You could theoretically use the integrated GPU to run compute programs while you run graphics programs on a discrete GPU device.

Next we'll be creating out swap chain as well as our two required shader programs to render a triangle. If I can fit it in, we'll also start getting a taste of the basic semaphore pattern for synchronising our rendering into  the correct order. 

Catch you in the next post! 