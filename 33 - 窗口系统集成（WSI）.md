# 33 - 窗口系统集成（WSI）

本章讨论 Vulkan API 与向用户显示渲染结果的各种形式之间的窗口系统集成（WSI）。 由于 Vulkan API 可以在不显示结果的情况下使用，因此 WSI 是通过使用可选的 Vulkan 扩展来提供的。
本章将概述 WSI。 有关每个 WSI 扩展的其他详细信息，包括必须启用哪些扩展才能使用本章介绍的每个功能，请参见附录。

## WSI 平台

平台是窗口系统、操作系统等的抽象概念。 例如 MS Windows、Android 和 Wayland。 Vulkan API 可以针对每个平台以独特的方式进行集成。

Vulkan API 没有定义任何类型的平台对象。 定义了特定平台的 WSI 扩展，每个扩展都包含使用 WSI 的特定平台功能。 这些扩展的使用受到预处理器符号的保护，如 "窗口系统专用头控制 "附录中所定义。

要在特定平台上编译应用程序以使用 WSI，应用程序必须

- 在包含 vulkan.h 头文件之前定义适当的预处理器符号
- 包含 vulkan_core.h 和任何原生平台头文件
- 然后是适当的平台特定头文件

窗口系统扩展和头文件表中定义了预处理器符号和特定平台头文件。

每个特定平台扩展都是一个实例扩展。 在使用实例扩展之前，应用程序必须使用 vkCreateInstance 启用它们。

## WSI Surface

本地平台的 Surface 或窗口对象由曲面对象抽象出来，而 Surface 对象则由 VkSurfaceKHR 句柄表示：

```cpp
// Provided by VK_KHR_surface
VK_DEFINE_NON_DISPATCHABLE_HANDLE(VkSurfaceKHR)
```

VK_KHR_surface 扩展声明了 VkSurfaceKHR 对象，并提供了一个用于销毁 VkSurfaceKHR 对象的函数。 针对不同平台的扩展为各自平台提供了创建 VkSurfaceKHR 对象的函数。 从应用程序的角度来看，这是一个不透明的句柄，就像其他 Vulkan 对象的句柄一样。

## 直接向显示设备展示

在某些环境中，应用程序还可以直接向显示设备呈现 Vulkan 渲染，而无需使用中间窗口系统。 这对于嵌入式应用或使用 Vulkan 实现窗口系统的呈现/显示后端非常有用。 VK_KHR_display 扩展提供了枚举显示设备和创建针对显示设备的 VkSurfaceKHR 对象所需的功能。

### 显示枚举值

### 显示控制

### 显示 Surface

### 向没有图形界面的系统呈现（Presenting to Headless Surfaces）

## 查询 WSI 支持

并非所有物理设备都支持 WSI。 在物理设备中，并非所有队列族都支持呈现。 WSI 支持和兼容性可以以平台中立的方式确定（确定对特定表面对象演示的支持），也可以以平台特定的方式确定（确定对指定物理设备演示的支持，但不保证对特定表面对象演示的支持）。

要确定物理设备的队列系列是否支持呈现到指定表面，请调用

```cpp
// Provided by VK_KHR_surface
VkResult vkGetPhysicalDeviceSurfaceSupportKHR(
    VkPhysicalDevice                            physicalDevice,
    uint32_t                                    queueFamilyIndex,
    VkSurfaceKHR                                surface,
    VkBool32*                                   pSupported);
```

- physicalDevice 是物理设备。
- queueFamilyIndex 是队列系列。
- surface 是表面。
- pSupported 是指向 VkBool32 的指针。 VK_TRUE 表示支持，VK_FALSE 表示不支持。

## Surface 查询

针对 Surface 的交换链功能是 WSI 平台、本地窗口或显示器以及物理设备功能的交叉点。 可以通过本节下面列出的查询获取所得到的功能。

> [!note]
> 除了通过以下表面查询获得的表面功能外，交换链图像还受 vkGetPhysicalDeviceImageFormatProperties 报告的普通图像创建限制的约束。 正如相应的 "有效使用 "章节所指示的那样，在创建交换链图像时，必须同时满足表面能力和图像创建限制。

### Surface 能力

### Surface 格式支持

要查询 Surface 所支持的显示模式，请调用

```cpp
// Provided by VK_KHR_surface
VkResult vkGetPhysicalDeviceSurfacePresentModesKHR(
    VkPhysicalDevice                            physicalDevice,
    VkSurfaceKHR                                surface,
    uint32_t*                                   pPresentModeCount,
    VkPresentModeKHR*                           pPresentModes);
```

- physicalDevice 是与要创建的交换链相关联的物理设备，如 vkCreateSwapchainKHR 所述。
- surface 是将与交换链关联的表面。
- pPresentModeCount 是指向一个整数的指针，该整数与可用或可查询的展示模式数量有关，如下所述。
- pPresentModes 既可以是 NULL，也可以是指向 VkPresentModeKHR 值数组的指针，表示支持的显示模式。

如果 pPresentModes 为 NULL，那么 pPresentModeCount 将返回给定表面所支持的显示模式的数量。 否则，pPresentModeCount 必须指向一个由应用程序设置为 pPresentModes 数组元素个数的变量，并在返回时用实际写入 pPresentModes 的值的个数覆盖该变量。 如果 pPresentModeCount 的值小于支持的显示模式数，则最多只会写入 pPresentModeCount 值，并返回 VK_INCOMPLETE，而不是 VK_SUCCESS，以表示没有返回所有可用的模式。

如果启用了 VK_GOOGLE_surfaceless_query 扩展且 surface 为 VK_NULL_HANDLE，则 pPresentModes 中返回的值将仅表示支持 VK_PRESENT_MODE_FIFO_KHR、VK_PRESENT_MODE_SHARED_DEMAND_REFRESH_KHR 和 VK_PRESENT_MODE_SHARED_CONTINUOUS_REFRESH_KHR。 若要查询对任何其他当前模式的支持，必须在 surface.SERVER 中提供一个有效句柄。

另外，如果要查询 Surface 支持的显示模式，并选择其他固定交换链创建参数，请调用：

```cpp
// Provided by VK_EXT_full_screen_exclusive
VkResult vkGetPhysicalDeviceSurfacePresentModes2EXT(
    VkPhysicalDevice                            physicalDevice,
    const VkPhysicalDeviceSurfaceInfo2KHR*      pSurfaceInfo,
    uint32_t*                                   pPresentModeCount,
    VkPresentModeKHR*                           pPresentModes);
```

- physicalDevice 是与要创建的交换链相关联的物理设备，如 vkCreateSwapchainKHR 所述。
- pSurfaceInfo 是指向 VkPhysicalDeviceSurfaceInfo2KHR 结构的指针，该结构描述了 vkCreateSwapchainKHR 将使用的表面和其他固定参数。
- pPresentModeCount 是一个整数指针，与可用或查询的显示模式数量有关，如下所述。
- pPresentModes 既可以是 NULL，也可以是指向 VkPresentModeKHR 值数组的指针，表示支持的呈现模式。

vkGetPhysicalDeviceSurfacePresentModes2EXT 的行为类似于 vkGetPhysicalDeviceSurfacePresentModesKHR，它可以通过链式输入结构指定扩展输入。

vkGetPhysicalDeviceSurfacePresentModesKHR::pPresentModes 数组元素的可能值表示曲面支持的显示模式：

```cpp
// Provided by VK_KHR_surface
typedef enum VkPresentModeKHR {
    VK_PRESENT_MODE_IMMEDIATE_KHR = 0,
    VK_PRESENT_MODE_MAILBOX_KHR = 1,
    VK_PRESENT_MODE_FIFO_KHR = 2,
    VK_PRESENT_MODE_FIFO_RELAXED_KHR = 3,
  // Provided by VK_KHR_shared_presentable_image
    VK_PRESENT_MODE_SHARED_DEMAND_REFRESH_KHR = 1000111000,
  // Provided by VK_KHR_shared_presentable_image
    VK_PRESENT_MODE_SHARED_CONTINUOUS_REFRESH_KHR = 1000111001,
  // Provided by VK_EXT_present_mode_fifo_latest_ready
    VK_PRESENT_MODE_FIFO_LATEST_READY_EXT = 1000361000,
} VkPresentModeKHR;
```

- VK_PRESENT_MODE_IMMEDIATE_KHR 指定呈现引擎不等待垂直消隐期（VBlank，一般要等待显示器发出 VSync 信号时交换双缓冲）更新当前图像，这意味着该模式可能会导致可见撕裂。 不需要对呈现请求进行内部排队，因为这些请求会被立即应用。
- VK_PRESENT_MODE_MAILBOX_KHR 指定呈现引擎等待下一个垂直消隐周期（VBlank）来更新当前图像。 不会有图像撕裂的问题。 内部单入口队列用于保存待处理的呈现请求。 如果收到新的呈现请求时队列已满，新请求将取代现有条目，与前一条目相关的任何图像都将可供应用程序重新使用。 在队列未空的每个垂直消隐期间，都会从队列中移除并处理一个请求。
- VK_PRESENT_MODE_FIFO_KHR 指定呈现引擎等待下一个垂直消隐周期（VBlank）更新当前图像。 不会有图像撕裂的问题。 内部队列用于保存待处理的呈现请求。 新的请求被添加到队列的末尾，在队列非空的每个垂直消隐周期内，一个请求从队列的起始位置被移除并得到处理。 这是唯一需要支持的 presentMode 值。
- VK_PRESENT_MODE_FIFO_RELAXED_KHR 指定呈现引擎通常会等待下一个垂直消隐周期来更新当前图像。 如果自上次更新当前图像后已经过了一个垂直消隐期，则呈现引擎不会等待下一个垂直消隐期进行更新，这意味着在这种情况下，该模式可能会导致可见撕裂。 这种模式适用于减少视觉卡顿的应用，因为这种应用大多数情况下会在下一个垂直消隐周期之前显示新图像，但偶尔也会延迟，在下一个垂直消隐周期之后才显示新图像。 内部队列用于保存待处理的呈现请求。 新的请求会被添加到队列的末尾，一个请求会从队列的起始位置移除，并在队列未空的每个垂直消隐周期期间或之后进行处理。
- VK_PRESENT_MODE_FIFO_LATEST_READY_EXT 指定呈现引擎等待下一个垂直消隐周期来更新当前图像。 无法观察到撕裂现象。 内部队列用于保存待处理的呈现请求。 新的请求会被添加到队列的末尾。 在每个垂直消隐周期，呈现引擎都会从队列的起始位置取消所有准备好呈现的连续请求。 如果使用 VK_GOOGLE_display_timing 提供目标呈现时间，呈现引擎将检查每幅图像的指定时间。 如果目标显示时间小于或等于当前时间，呈现引擎将重新排序图像并检查下一张图像。 最后一个重新排序请求的图像将被呈现。 其他取消排队的请求将被丢弃。
- VK_PRESENT_MODE_SHARED_DEMAND_REFRESH_KHR 规定呈现引擎和应用程序可同时访问单个图像，该图像被称为共享可呈现图像。 呈现引擎只有在收到新的呈现请求后才需要更新当前图像。 因此，只要需要更新，应用程序就必须发出呈现请求。 不过，呈现引擎可以在任何时候更新当前图像，这意味着该模式可能会导致可见撕裂。
- VK_PRESENT_MODE_SHARED_CONTINUOUS_REFRESH_KHR 指定呈现引擎和应用程序可同时访问单个图像，这被称为共享可呈现图像。 呈现引擎在其常规刷新周期中定期更新当前图像。 应用程序只需发出一个初始呈现请求，之后呈现引擎必须更新当前图像，而无需再发出任何呈现请求。 应用程序可通过提出呈现请求来显示图像内容已更新，但这并不能保证图像更新的时间。 如果图像的呈现时间不正确，这种模式可能会导致明显的撕裂。

为曲面创建的交换链的可显示图像的 VkImageUsageFlagBits 支持值可能因显示模式而异，可按下表确定：

![](pic/Pasted%20image%2020241204144451.png)

> [!note]
> 作为参考，VK_PRESENT_MODE_FIFO_KHR 表示的模式等同于交换间隔为 1 的 {wgl|glX|egl}SwapBuffers 的行为，而 VK_PRESENT_MODE_FIFO_RELAXED_KHR 表示的模式等同于交换间隔为 -1 的 {wgl|glX}SwapBuffers 的行为（来自 {WGL|GLX}_EXT_swap_control_tear 扩展）。

### Surface 呈现模式支持

## 全屏独家控制

## 设备组查询

## 显示定时查询

## 呈现等待

如果应用程序希望通过监控呈现过程的完成时间来控制应用程序的节奏，从而限制排队等待呈现的未显示图像的数量，则需要在呈现过程中获得信号的方法。

使用 VK_GOOGLE_display_timing 扩展，应用程序可以发现图像何时呈现，但只能是异步的。

提供一种机制允许应用程序阻塞，等待呈现过程的特定步骤完成，这样就可以控制未完成工作的数量（从而控制在响应用户输入或呈现环境变化时可能出现的延迟）。

VK_KHR_present_wait 扩展允许应用程序通过传递一个 VkPresentIdKHR 结构，在 vkQueuePresentKHR 调用中告诉演示引擎它计划等待呈现。 然后，该结构中传递的 presentId 可以传递给未来的 vkWaitForPresentKHR 调用，使应用程序阻塞，直到呈现结束。

## WSI 交换链

交换链对象（又称交换链）可将渲染结果呈现在表面（surface）上。

交换链是与 Surface 关联的可呈现图像数组的抽象。可呈现图像由平台创建的 VkImage 对象表示。一次只能显示一幅图像（多视图/立体-3D Surface 可以是一个数组图像），但可以排队显示多幅图像。应用程序对图像进行渲染，然后排队将图像显示到 Surface 上。

一个本地（native）窗口不能同时与一个以上的非退役交换链关联。此外，无法为关联了非 Vulkan 图形 API 表面（surface）的本地窗口创建交换链。

> [!note]
> 呈现引擎是平台合成器或显示引擎的抽象。
> 对于应用程序和/或逻辑设备而言，呈现引擎可以是同步的，也可以是异步的。
> 某些实现可使用设备的图形队列或专用演示硬件来执行呈现。

交换链的可呈现图像归呈现引擎所有。应用程序可以从呈现引擎获取可呈现图像的使用权。对可呈现图像的使用必须在 vkAcquireNextImageKHR 返回图像之后、vkQueuePresentKHR 释放图像之前进行。 这包括图像布局和渲染命令的转换。

应用程序可通过 vkAcquireNextImageKHR 获取可呈现图像的使用。获取可呈现图像后，在对其进行修改前，应用程序必须使用同步原语，以确保呈现引擎已完成图像读取。然后，应用程序就可以转换图像的布局、对其执行队列呈现命令等。最后，应用程序通过 vkQueuePresentKHR 呈现图像，从而释放对图像的获取。如果设备不使用图像，应用程序还可以通过 vkReleaseSwapchainImagesEXT 释放图像采集，并跳过呈现操作。

呈现引擎控制着获取可呈现图像供应用程序使用的顺序。

> [!note]
> 这种设计允许平台处理在图像呈现后需要不按顺序返回图像的情况。这种设计还允许应用程序在初始化时生成引用交换链中所有图像的命令缓冲区（我们为每个交换链图像创建一个 cmd Buffer），而不是在主循环中。

具体工作原理如下。

如果在创建交换链时将 presentMode 设置为 VK_PRESENT_MODE_SHARED_DEMAND_REFRESH_KHR 或 VK_PRESENT_MODE_SHARED_CONTINUOUS_REFRESH_KHR，那么就可以获取单个可呈现图像，称为共享可呈现图像。共享的可呈现图像可由应用程序和呈现引擎同时访问，而无需在最初呈现图像后转换图像的布局。

- 有了 VK_PRESENT_MODE_SHARED_DEMAND_REFESH_KHR，呈现引擎只需在呈现后更新共享可呈现图像的最新内容。应用程序必须调用 vkQueuePresentKHR 来保证更新。不过，呈现引擎可以随时从中进行更新。
- 使用 VK_PRESENT_MODE_SHARED_CONTINUOUS_REFRESH_KHR，呈现引擎会在每个刷新周期中自动呈现共享呈现图像的最新内容。应用程序只需对 vkQueuePresentKHR 进行一次初始调用，之后演示引擎就会从中进行更新，而无需再调用任何 Present 调用。应用程序可以通过调用 vkQueuePresentKHR 来显示图像内容已更新，但这并不能保证更新发生的时间。

呈现引擎可在共享的可呈现图像首次呈现后的任何时间访问该图像。为避免撕裂（两帧画面显示在同一个画面上），应用程序应与呈现引擎协调访问。这就需要通过特定于平台的机制提供呈现引擎的定时信息，并确保在呈现引擎的刷新周期中，颜色附件的写入是可用的。

> [!note]
> VK_KHR_shared_presentable_image 扩展不提供用于确定呈现引擎刷新周期时间的功能。

<details>
<summary>交换链API</summary>

要查询交换链在渲染为共享可呈现图像时的状态，请调用

```cpp
// Provided by VK_KHR_shared_presentable_image
VkResult vkGetSwapchainStatusKHR(
    VkDevice                                    device,
    VkSwapchainKHR                              swapchain);
```

vkGetSwapchainStatusKHR 的可能返回值应解释如下：

- VK_SUCCESS 表示呈现引擎正在按照交换链的 VkPresentModeKHR 呈现共享可呈现图像的内容。
- VK_SUBOPTIMAL_KHR 交换链不再与表面（surface）属性完全匹配，但呈现引擎正在按照交换链的 VkPresentModeKHR 呈现共享的可呈现图像的内容。
- VK_ERROR_OUT_OF_DATE_KHR 表面（surface）已发生变化，不再与交换链兼容。
- VK_ERROR_SURFACE_LOST_KHR 该表面（surface）已不可用。

> [!note]
> 具体的实现可能会缓存交换链状态，因此应用程序在使用 VkPresentModeKHR 设为 VK_PRESENT_MODE_SHARED_CONTINUOUS_REFRESH_KHR 的交换链时，应定期调用 vkGetSwapchainStatusKHR。

要创建交换链，请调用

```cpp
// Provided by VK_KHR_swapchain
VkResult vkCreateSwapchainKHR(
    VkDevice                                    device,
    const VkSwapchainCreateInfoKHR*             pCreateInfo,
    const VkAllocationCallbacks*                pAllocator,
    VkSwapchainKHR*                             pSwapchain);
```

- device 是要创建交换链的设备。
- pCreateInfo 是指向 VkSwapchainCreateInfoKHR 结构的指针，该结构指定了所创建 swapchain 的参数。
- pAllocator 是分配给 swapchain 对象的主机内存的分配器，如果没有更具体的分配器可用（请参阅内存分配）。
- pSwapchain 是指向 VkSwapchainKHR 句柄的指针，创建的交换链对象将在该句柄中返回。

如上所述，如果 vkCreateSwapchainKHR 成功创建，它将返回一个交换链句柄，该交换链包含一个至少由 pCreateInfo->minImageCount 呈现图像组成的数组。

在应用程序获取图像时，可呈现图像可以以任何等效的非可呈现图像的方式使用。可呈现图像等同于使用以下 VkImageCreateInfo 参数创建的非可呈现图像：
![](pic/Pasted%20image%2020240703092329.png)

pCreateInfo->surface 必须在 swapchain 销毁后才能销毁。

如果 oldSwapchain 是 VK_NULL_HANDLE，并且 pCreateInfo->surface 引用的本地窗口已经与 Vulkan swapchain 关联，则必须返回 VK_ERROR_NATIVE_WINDOW_IN_USE_KHR。

如果 pCreateInfo->surface 引用的本地窗口已与非 Vulkan 图形 API 表面关联，则返回 VK_ERROR_NATIVE_WINDOW_IN_USE_KHR。

在所有关联的 Vulkan swapchain 销毁之前，pCreateInfo->surface 引用的本地窗口不得与非 Vulkan 图形 API 表面关联。

如果逻辑设备丢失，vkCreateSwapchainKHR 将返回 VK_ERROR_DEVICE_LOST。VkSwapchainKHR 是设备的子设备，必须先于设备销毁。
然而，VkSurfaceKHR 不是任何 VkDevice 的子节点，因此不受丢失设备的影响。成功重新创建一个 VkDevice 后，可以使用相同的 VkSurfaceKHR 创建一个新的 VkSwapchainKHR，前提是之前的 VkSwapchainKHR 已被销毁。

如果 pCreateInfo 的 oldSwapchain 参数是一个有效的交换链，并且这个交换链拥有独占的全屏访问权限，那么 pCreateInfo->oldSwapchain 将释放该访问权限。如果命令成功执行，新创建的交换链将自动从 pCreateInfo->oldSwapchain 获得全屏独占访问权限。

> [!note]
> 这种隐式传输旨在避免退出和进入全屏独占模式，否则可能会对显示屏造成不必要的视觉更新。

在某些情况下，如果应用程序控制请求使用全屏独占模式，但由于某些特定的实现原因，所提供的特定参数组合无法使用全屏独占访问，则交换链创建可能会失败。如果出现这种情况，将返回 VK_ERROR_INITIALIZATION_FAILED。

> [!note]
> 特别是，如果 pCreateInfo 的 imageExtent 成员与监视器的外延不匹配，则会失败。其他失败原因可能包括应用程序未设置为高分辨率感知，或者物理设备和显示器在此模式下不兼容。

如果 VkSwapchainCreateInfoKHR 的 pNext 链包含一个 VkSwapchainPresentBarrierCreateInfoNV 结构，那么该结构就会包含与当前屏障（barrier ）相关的附加交换链创建参数。如果当前系统的状态限制了当前屏障（barrier ）功能 VkSurfaceCapabilitiesPresentBarrierNV 的使用，或者交换链本身不满足所有必要条件，交换链创建可能会失败。在这种情况下，将返回 VK_ERROR_INITIALIZATION_FAILED。

当 VkSwapchainCreateInfoKHR 中的 VkSurfaceKHR 是显示表面时，显示表面的 VkDisplaySurfaceCreateInfoKHR 中的 VkDisplayModeKHR 与特定的 VkDisplayKHR 相关联。 如果应用程序未获取该 VkDisplayKHR，交换链创建可能会失败。在这种情况下，将返回 VK_ERROR_INITIALIZATION_FAILED。

VkSwapchainCreateInfoKHR 结构定义如下：

```cpp
// Provided by VK_KHR_swapchain
typedef struct VkSwapchainCreateInfoKHR {
    VkStructureType                  sType;
    const void*                      pNext;
    VkSwapchainCreateFlagsKHR        flags;
    VkSurfaceKHR                     surface;
    uint32_t                         minImageCount;
    VkFormat                         imageFormat;
    VkColorSpaceKHR                  imageColorSpace;
    VkExtent2D                       imageExtent;
    uint32_t                         imageArrayLayers;
    VkImageUsageFlags                imageUsage;
    VkSharingMode                    imageSharingMode;
    uint32_t                         queueFamilyIndexCount;
    const uint32_t*                  pQueueFamilyIndices;
    VkSurfaceTransformFlagBitsKHR    preTransform;
    VkCompositeAlphaFlagBitsKHR      compositeAlpha;
    VkPresentModeKHR                 presentMode;
    VkBool32                         clipped;
    VkSwapchainKHR                   oldSwapchain;
} VkSwapchainCreateInfoKHR;
```

- sType 是标识此结构的 VkStructureType 值。
- pNext 是 NULL 或指向扩展此结构的结构的指针。
- flags 是 VkSwapchainCreateFlagBitsKHR 的位掩码，表示创建交换链的参数。
- surface 是交换链将显示图像的表面。如果创建成功，交换链将与表面关联。
- minImageCount 是应用程序所需的可呈现图像的最小数量。执行过程中，要么创建的交换链至少包含这么多图像，要么创建交换链失败。
- imageFormat 是一个 VkFormat 值，用于指定创建交换链图像的格式。
- imageColorSpace 是一个 VkColorSpaceKHR 值，指定交换链解释图像数据的方式。
- imageExtent 是交换链图像的大小（像素）。如果图像范围与 vkGetPhysicalDeviceSurfaceCapabilitiesKHR 返回的曲面当前范围不一致，则行为取决于平台。

> [!note]
> 在某些平台上，例如窗口最小化时，maxImageExtent 可能会变为 (0, 0)，这是正常现象。在这种情况下，由于有效使用要求，除非通过 VkSwapchainPresentScalingCreateInfoEXT（如果支持）选择缩放，否则无法创建交换链。

- imageArrayLayers 是多视图/立体表面中的视图数。对于非立体-3D 应用程序，该值为 1。
- imageUsage 是 VkImageUsageFlagBits 的位掩码，用于描述（获取的）交换链图像的预期用途。
- imageSharingMode 是交换链图像的共享模式。
- queueFamilyIndexCount 是当 imageSharingMode 为 VK_SHARING_MODE_CONCURRENT 时可访问交换链图像的队列族的数量。
- pQueueFamilyIndices 是指向队列族索引数组的指针，当 imageSharingMode 为 VK_SHARING_MODE_CONCURRENT 时，该队列族索引可访问交换链中的图像。
- preTransform 是一个 VkSurfaceTransformFlagBitsKHR 值，用于描述在呈现之前应用于图像内容的相对于呈现引擎自然方向的变换。如果它与 vkGetPhysicalDeviceSurfaceCapabilitiesKHR 返回的 currentTransform 值不匹配，呈现引擎将在呈现操作中转换图像内容。
- compositeAlpha 是一个 VkCompositeAlphaFlagBitsKHR 值，表示此表面（surface）在某些窗口系统上与其他曲面合成时使用的 alpha 合成模式。
- presentMode 是交换链将使用的呈现模式。交换链的呈现模式决定了内部处理传入的呈现请求和排队的方式。
- clipped 指定 Vulkan 实现是否允许放弃影响不可见表面（surface）区域的渲染操作。
  - 如果设置为 VK_TRUE，与交换链关联的可呈现图像可能不拥有其所有像素。如果可呈现图像中的像素对应的目标表面区域被桌面上的其他窗口遮挡，或受到其他剪切机制的影响，则在回读时会出现未定义的内容。片段着色器可能不会针对这些像素执行，因此不会产生任何副作用。设置 VK_TRUE 并不能保证会发生任何剪切，但可以在某些平台上使用更有效的表现方法。
  - 如果设置为 VK_FALSE，与交换链关联的可呈现图像将拥有其包含的所有像素。

> [!note]
> 如果应用程序不希望在显示可呈现图像之前或重新获取图像之后读回可呈现图像的内容，而且其片段着色器没有任何副作用要求对可呈现图像中的所有像素运行，则应将此值设置为 VK_TRUE。

- oldSwapchain 是 VK_NULL_HANDLE，或者是当前与 surface 关联的现有非退役交换链。提供一个有效的 oldSwapchain 有助于资源重用，还能让应用程序继续显示已从中获取的任何图像。

在调用 vkCreateSwapchainKHR 时，如果 oldSwapchain 不是 VK_NULL_HANDLE，则 oldSwapchain 将退役 - 即使新 swapchain 的创建失败。无论 oldSwapchain 是否为 VK_NULL_HANDLE，新的交换链都会在非退役状态下创建。

在使用非 VK_NULL_HANDLE 的 oldSwapchain 调用 vkCreateSwapchainKHR 时，应用程序可能会释放 oldSwapchain 中未被获取的任何映像，即使新 swapchain 的创建失败也可能发生这种情况。应用程序可以销毁 oldSwapchain，以释放与 oldSwapchain 相关的所有内存。

> [!note]
> 通过多次使用 oldSwapchain（使用次数多于对 vkDestroySwapchainKHR 的调用次数），可以将多个已退役的交换链与同一个 VkSurfaceKHR 关联起来。
>
> oldSwapchain 退役后，应用程序可将已从 oldSwapchain 获取的任何图像传递给 vkQueuePresentKHR。例如，应用程序可能会在新交换链中的图像准备就绪之前展示旧交换链中的图像。与往常一样，如果 oldSwapchain 已进入导致 VK_ERROR_OUT_OF_DATE_KHR 返回的状态，vkQueuePresentKHR 可能会失败。
>
> 只要未进入导致返回 VK_ERROR_OUT_OF_DATE_KHR 的状态，应用程序就可以继续使用从 oldSwapchain 获取的共享可呈现映像，直到从新 swapchain 获取可呈现映像为止。

可在 VkSwapchainCreateInfoKHR::flags 中设置的位有

```cpp
// Provided by VK_KHR_swapchain
typedef enum VkSwapchainCreateFlagBitsKHR {
  // Provided by VK_VERSION_1_1 with VK_KHR_swapchain, VK_KHR_device_group with VK_KHR_swapchain
    VK_SWAPCHAIN_CREATE_SPLIT_INSTANCE_BIND_REGIONS_BIT_KHR = 0x00000001,
  // Provided by VK_VERSION_1_1 with VK_KHR_swapchain
    VK_SWAPCHAIN_CREATE_PROTECTED_BIT_KHR = 0x00000002,
  // Provided by VK_KHR_swapchain_mutable_format
    VK_SWAPCHAIN_CREATE_MUTABLE_FORMAT_BIT_KHR = 0x00000004,
  // Provided by VK_EXT_swapchain_maintenance1
    VK_SWAPCHAIN_CREATE_DEFERRED_MEMORY_ALLOCATION_BIT_EXT = 0x00000008,
} VkSwapchainCreateFlagBitsKHR;
```

</details>

## HDR 元数据

## 滞后控制

## 呈现屏障
