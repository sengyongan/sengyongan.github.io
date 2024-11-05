---
title: 14`UniformBuffers
date: 2024-11-03 12:00:00 +0800
categories: [Vulkan]
tags: [Vulkan]     # TAG names should always be lowercase
math: true
---
# Uniform buffers

## Descriptor layout and buffer

我们将继续学习3D图形，这需要一个模型-视图-投影矩阵，因此我们要更改向vertex shader传输的数据，也就是通过vertex buffer
但是当实时渲染，每一帧这些数据都有可能变化，都需要更新顶点缓冲区，会造成内存浪费

uniform buffer是一块存储在GPU上的内存区域，它可以在多个着色器（甚至多个着色器阶段）之间共享一致的数据
在Vulkan中解决此问题的正确方法是使用资源描述符Descriptor，

描述符是一种数据结构，可以包括uniform buffer、采样器（Sampler）、图像（Image）等

描述符是着色器自由访问buffer的一种方式，描述符的使用包括三个部分：

* 在pipeline创建期间指定descriptorLayout（指定pipeline将要访问的资源类型，）
* 从descriptorPool分配descriptorSet（指定将绑定到描述符的实际缓冲区或图像资源）
* 在rendering期间bind descriptorSet（将uniform buffer数据传给shader）

## Vertex shader

修改vertex shader,让gl_Position每个顶点位置都应用这几个变化矩阵

```
layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;
```

## DescriptorSetLayout

在程序创建UniformBufferObject结构体

添加createDescriptorSetLayout();函数，包括VkDescriptorSetLayoutBinding，VkDescriptorSetLayoutCreateInfo，vkCreateDescriptorSetLayout一个VkDescriptorSetLayout，vkDestroyDescriptorSetLayout

前两个字段指定***着色器***中使用的绑定(binding = 0)和描述符的类型VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER，我们还需要指定描述符将被引用的着色器阶段VK_SHADER_STAGE_VERTEX_BIT,pImmutableSamplers字段只与图像采样相关的描述符(可选)

另外我们需要在管道创建期间指定DescriptorSetLayout

## Uniform buffer

我们需要首先创建Uniform buffer，我们将在每一帧中将新数据复制到统一缓冲区，我们应该设置为vector，因为有多个帧在并发执行

添加新的createUniformBuffers函数，我们在创建后立即使用vkMapMemory映射缓冲区，且不会vkUnmapMemory，在应用程序的整个生命周期中都保持映射到此指针，这会提高性能

## Updating uniform data

在drawFrame每帧渲染中，添加新函数updateUniformBuffer更新缓冲区的数据，以便计算transformations

glm/gtc/matrix_transform.hpp头部公开了可用于生成模型转换（如glm：：rotate）、视图转换（如glm：：lookAt）和投影转换（如glm：：perspective）的函数

还有有#define GLM_FORCE_RADIANS以确保像glm：：rotate这样的函数使用弧度作为参数，以避免任何可能的混淆

chrono标准库公开了执行精确**计时**的函数，让我们的数据随着时间变化

GLM最初是为OpenGL设计的，其中剪辑坐标的Y坐标是反转的。最简单的补偿方法是翻转投影矩阵中Y轴的缩放因子上的符号。如果你不这样做，那么图像将被颠倒渲染

依旧通过memcpy可以将临时的uniform buffer对象中的数据复制到当前uniformBuffer中

### memcpy

void * memcpy ( void * destination, const void * source, size_t num );

（指向要在其中复制内容的目标数组的指针，指向要复制的数据源的指针，要复制的字节数）

## Descriptor pool

我们将为每个VkBuffer资源创建一个descriptorSet，以将其绑定到Uniform buffer描述符

描述符集不能直接创建，必须从描述符池中分配，定义新函数createDescriptorPool

VkDescriptorPoolSize描述描述符集将包含哪些描述符类型以及它们的数量（为每个帧都分配），

VkDescriptorPoolCreateInfo，除了可用的单个描述符的最大数量外，我们还需要指定可以分配的描述符集的最大数量

vkCreateDescriptorPool，vkDestroyDescriptorPool

## Descriptor set

我们现在可以分配描述符集本身，添加新的createDescriptorSets函数

描述符集分配通过VkDescriptorSetAllocateInfo，需要指定要分配的描述符池、要分配的描述符集的数量（为每个帧都分配）以及它们所基于的描述符布局

vkAllocateDescriptorSets，不需要显式地清理描述符集，因为当描述符池被销毁时

描述符集现在已经分配完毕，但是仍然需要配置其中的描述符。我们现在将添加一个for循环来填充每个描述符

VkDescriptorBufferInfo，指定缓冲区以及其中包含描述符数据的区域

VkWriteDescriptorSet，

前两个字段指定要更新的描述符集和绑定，我们需要再次指定描述符的类型，descriptorCount可以一次更新数组中的多个描述符

pBufferInfo字段用于引用缓冲区数据的描述符，pImageInfo用于引用图像数据的描述符，pTexelBufferView用于引用缓冲区视图的描述符。

vkUpdateDescriptorSets

## Using descriptor sets

需要更新recordCommandBuffer函数，通过vkCmdBindDescriptorSets将描述符集绑定到渲染管线的相应阶段，这样着色器就可以通过描述符来访问uniform buffer中的数据了，就像vertex buffer那样

参数是描述符所基于的布局。接下来的三个参数指定第一个描述符集的索引、要绑定的集的数量以及要绑定的集的数组

如果你现在运行你的程序，什么都看不到，由于我们在投影矩阵中做了Y翻转，现在顶点是按逆时针顺序绘制的，而不是顺时针顺序。这将导致背面剔除

VkGraphicsPipeline函数并修改VkPipelineRasterizationStatesInfo中的frontFace以更正此错误

## Alignment requirements

vulkan要求结构中的数据以特定的方式在内存中对齐

* 标量必须按N（= 4字节给定32位浮点数）对齐。
* •vec2必须对齐2N（= 8字节）

* •vec3或vec4必须对齐4N（= 16字节）
* •嵌套结构必须按其成员的基本对齐方式对齐

* 四舍五入到16的倍数。
* •mat4矩阵必须与vec4具有相同的对齐方式。

比如在UniformBufferObject结构体加入glm::vec2 foo，它是8字节，而mat4是64字节

为了解决这个问题，我们可以使用C++11中引入的alignas说明符：例如alignas(16)表示偏移量为16的倍数

还有一种简单的方式，#define GLM_FORCE_DEFAULT_ALIGNED_GENTYPES，它会自动帮助我们对齐，（注意：对于嵌套结构不会正确计算，必须自己计算）

## Multiple descriptor sets

实际上可以同时绑定多个描述符集。在创建管道布局时，您需要为每个描述符集指定一个描述符布局

然后通过这样的形式

`layout(set = 0, binding = 0) uniform UniformBufferObject { ... }`

![image](/assets/img/blog/vulkan/gif.gif)

<script src="https://utteranc.es/client.js"
        repo="sengyongan/sengyongan.github.io"
        issue-term="pathname"
        label="Comment"
        theme="dark-blue"
        crossorigin="anonymous"
        async>
</script>