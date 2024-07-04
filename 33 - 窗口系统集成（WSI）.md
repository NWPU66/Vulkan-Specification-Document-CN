# 33 - 窗口系统集成（WSI）

## WSI 交换链

交换链对象（又称交换链）可将渲染结果呈现在表面（surface）上。

交换链是与表面（surface）关联的可呈现图像数组的抽象。可呈现图像由平台创建的 VkImage 对象表示。一次只能显示一幅图像（多视图/立体-3D 表面可以是一个数组图像），但可以排队显示多幅图像。应用程序对图像进行渲染，然后排队将图像显示到曲面上。

一个本地（native）窗口不能同时与一个以上的非退役交换链关联。此外，无法为关联了非 Vulkan 图形 API 表面（surface）的本地窗口创建交换链。

> [!note]
> 呈现引擎是平台合成器或显示引擎的抽象。
> 对于应用程序和/或逻辑设备而言，呈现引擎可以是同步的，也可以是异步的。
> 某些实现可使用设备的图形队列或专用演示硬件来执行呈现。

交换链的可呈现图像归呈现引擎所有。应用程序可以从呈现引擎获取可呈现图像的使用权。对可呈现图像的使用必须在 vkAcquireNextImageKHR 返回图像之后、vkQueuePresentKHR 释放图像之前进行。 这包括图像布局和渲染命令的转换。

应用程序可通过 vkAcquireNextImageKHR 获取可呈现图像的使用。获取可呈现图像后，在对其进行修改前，应用程序必须使用同步原语，以确保呈现引擎已完成图像读取。然后，应用程序就可以转换图像的布局、对其执行队列呈现命令等。最后，应用程序通过 vkQueuePresentKHR 呈现图像，从而释放对图像的获取。如果设备不使用图像，应用程序还可以通过 vkReleaseSwapchainImagesEXT 释放图像采集，并跳过呈现操作。

呈现引擎控制着获取可呈现图像供应用程序使用的顺序。

> [!note]
> 这样，平台就能处理需要在展示后按顺序返回图像的情况。同时，它还允许应用程序在初始化时，而不是在主循环中，生成引用交换链中所有图像的命令缓冲区。

具体工作原理如下。

如果在创建交换链时将 presentMode 设置为 VK_PRESENT_MODE_SHARED_DEMAND_REFRESH_KHR 或 VK_PRESENT_MODE_SHARED_CONTINUOUS_REFRESH_KHR，那么就可以获取单个可呈现图像，称为共享可呈现图像。共享的可呈现图像可由应用程序和呈现引擎同时访问，而无需在最初呈现图像后转换图像的布局。

- 有了 VK_PRESENT_MODE_SHARED_DEMAND_REFESH_KHR，呈现引擎只需在呈现后更新共享可呈现图像的最新内容。应用程序必须调用 vkQueuePresentKHR 来保证更新。不过，呈现引擎可以随时从中进行更新。
- 使用 VK_PRESENT_MODE_SHARED_CONTINUOUS_REFRESH_KHR，呈现引擎会在每个刷新周期中自动呈现共享呈现图像的最新内容。应用程序只需对 vkQueuePresentKHR 进行一次初始调用，之后演示引擎就会从中进行更新，而无需再调用任何 Present 调用。应用程序可以通过调用 vkQueuePresentKHR 来显示图像内容已更新，但这并不能保证更新发生的时间。

呈现引擎可在共享的可呈现图像首次呈现后的任何时间访问该图像。为避免撕裂，应用程序应与呈现引擎协调访问。这就需要通过特定于平台的机制提供呈现引擎的定时信息，并确保在呈现引擎的刷新周期中，颜色附件的写入是可用的。

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
