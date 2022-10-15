+++
title = "Using Driver Information As A Lesson In Vulkan Structure Chains"
date = 2022-10-15

[taxonomies]
tags = ["c++", "vulkan"]
+++

I was working on Hotarubi's driver detection today when it hit me that I don't think I've ever seen a nice and simple explanation of what `pNext` and `sType` are in the Vulkan API. Sure, [it's documented very well by Khronos themselves](https://github.com/KhronosGroup/Vulkan-Guide/blob/master/chapters/pnext_and_stype.adoc) but there isn't much in the way of examples and the examples that do exist can be a little hard to find and interpret for the first time Vulkaneer. So I just want to provide a quick overview using a very simple real-word example.

> I'll be using Vulkan 1.3 for this example but all of the structures I'll be using are also available for earlier forms of the API as extensions. You will, however, need to query support for them and handle any driver's lack of support accordingly. The implementation details of that are outside the scope of this article.

Let's consider the following chain of Vulkan structures:

```cpp
// https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkPhysicalDeviceProperties2.html
VkPhysicalDeviceProperties2 device_props =
{
  // ..
};

// https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkPhysicalDeviceIDPropertiesKHR.html
VkPhysicalDeviceIDProperties id_props =
{
  // ..
};

// https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkPhysicalDeviceDriverProperties.html
VkPhysicalDeviceDriverProperties driver_props =
{
  // ..
};
```

Our goal is to call `vkGetPhysicalDeviceProperties2` with a pointer to `VkPhysicalDeviceProperties2` . So our top level structure will be `device_props`. The order of the other two structures, however, is entirely up to you. So long as they properly set the `pNext` and `sType` variables, it will work as expected.

Essentially, the driver is going to cast the `pNext` variable to `VkBaseOutStructure*` and then check the `sType` variable for a structure type it supports for this function call (it will ignore any other structure types):

```cpp
typedef struct VkBaseOutStructure {
    VkStructureType               sType;
    struct VkBaseOutStructure*    pNext;
} VkBaseOutStructure;
```

Then it will cast it to the original structure type based on the information in `sType` and fill it in accordingly. Finally, it will do the same for that structure's `pNext` variable repeating those steps until it reaches a `pNext` which is `VK_NULL_HANDLE`.

If this sounds like CS 101, that's because it's basically a singly linked-list with extra steps.

![](/linked-list-extra-steps.jpg)

This is also the reason that you need to boilerplate all your structures with an `sType`. Vulkan is a C API and as such it cannot know the actual type of the structure from a void pointer alone. It therefore needs the additional `sType` variable to know what type of structure it is working with and whether the driver supports that structure for a given function call.

The most obvious thing to do here is this:

```cpp
VkPhysicalDeviceProperties2 device_props = {};
device_props.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_PROPERTIES_2;

VkPhysicalDeviceIDProperties id_props = {};
id_props.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_ID_PROPERTIES;

VkPhysicalDeviceDriverProperties driver_props = {};
driver_props.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_DRIVER_PROPERTIES;

device_props.pNext = reinterpret_cast<void*>(&id_props);
id_props.pNext = reinterpret_cast<void*>(&driver_props);
// driver_props.pNext = VK_NULL_HANDLE; // implied by = {}
```

We can also go this route:

```cpp
VkPhysicalDeviceIDProperties id_props =
{
  .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_ID_PROPERTIES,
  .pNext = VK_NULL_HANDLE,
};

VkPhysicalDeviceDriverProperties driver_props =
{
  .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_DRIVER_PROPERTIES,
  .pNext = reinterpret_cast<void*>(&id_props),
};

VkPhysicalDeviceProperties2 device_props =
{
  .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_PROPERTIES_2,
  .pNext = reinterpret_cast<void*>(&driver_props),
};
```

And for all intents and purposes, these are both fine solutions.

It does, however, result in some occasionally awkward code. For example, if you want to define your structures without modifying `pNext` manually after the fact, you need to define them such that they are in the reverse order of their inclusion. Or, alternatively, if you do go setting the various `pNext` variables after the fact it can get hard to keep track of the chain itself. Both solutions can get even more hairy on earlier versions of the API where you need to query support for these structures before including them in the chain.

Instead, for a little extra runtime cost we can mimic the driver and do something like this:

```cpp
// the types aren't important unless you do some static checking
// feel free to drop the template and use two void*
template<typename B, typename T>
inline void ChainInsert(B* base, T* item)
{
  // start at the base
  auto head = reinterpret_cast<VkBaseOutStructure*>(base);

  // loop until we reach the end of the chain
  // you might consider runtime checking for a duplicate entry here
  while (head->pNext != VK_NULL_HANDLE)
    head = head->pNext; 

  // we broke the loop so pNext is VK_NULL_HANDLE
  // we can insert our item here
  head->pNext = reinterpret_cast<VkBaseOutStructure*>(item);
}
```

Then it's just a matter of calling `ChainInsert` with our base and the structures we want to include in the chain:

```cpp
// first define our structures with pNext as VK_NULL_HANDLE
VkPhysicalDeviceIDProperties id_props =
{
  .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_ID_PROPERTIES,
  .pNext = VK_NULL_HANDLE,
};

VkPhysicalDeviceDriverProperties driver_props =
{
  .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_DRIVER_PROPERTIES,
  .pNext = VK_NULL_HANDLE,
};

VkPhysicalDeviceProperties2 device_props =
{
  .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_PROPERTIES_2,
  .pNext = VK_NULL_HANDLE,
};

// using the top level struct as the base
ChainInsert(&device_props, &id_props);
ChainInsert(&device_props, &driver_props);

// using the previous item in the chain as the base
// if, for exmaple, you know it will be included unconditionally
// ChainInsert(&device_props, &id_props);
// ChainInsert(&id_props, &driver_props);
```

Regardless of how you decide to do it, the next step is to call `VkPhysicalDeviceProperties2` with our base structure and then we can use the other structs in the chain normally:

```cpp
const VkResult res = VkPhysicalDeviceProperties2(physical_device, &device_props);
if (res != VK_SUCCESS)
{
  // handle failure
}

// https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkDriverId.html
// maybe handle driver errata
switch(driver_props.driverID)
{
  case VK_DRIVER_ID_MESA_RADV:
   // ..
  case VK_DRIVER_ID_AMD_OPEN_SOURCE:
   // ..
}

// maybe log the driver name
printf("driver: %s\n", driver_props.driverName);
```

Hopefully that demystified a little of the Vulkan API for you. 


