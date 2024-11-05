---
title: 13`Staging buffer
date: 2024-11-02 12:00:00 +0800
categories: [Vulkan]
tags: [Vulkan]     # TAG names should always be lowercase
math: true
---
# VulkanTutorial（Staging buffer）

## Staging buffer

我们从CPU访问它的内存类型可能不是GPU读取的最佳内存类型，最佳内存具有VK_MEMORY_PROPERTY_DEVICE_BIT标志
我们将创建两个顶点缓冲区。CPU可访问内存中的一个暂存缓冲区，用于将顶点数组中的数据上传到其中，最后一个顶点缓冲区位于设备本地内存中。然后，我们将使用buffer copy命令将数据从暂存缓冲区移动到实际的顶点缓冲区。

## Transfer queue

缓冲区复制命令需要一个支持传输操作的queue Family,使用VK_QUEUE_TRANSFER_BIT表示,
具有VK_QUEUE_GRAPHICS_BIT或VK_QUEUE_COMPUTE_BIT的任何queue Family,已经隐式支持VK_QUEUE_TRANSFER_BIT操作

## Abstracting buffer creation

因为我们将在本章中创建多个缓冲区，所以最好将缓冲区创建移到单独的函数中
将原本createVertexBuffer除了MapMemory部分，全部移动到新函数中，不要忘记使用函数参数,而非原有的固定值，并在createVertexBuffer调用createBuffer
运行程序以确保顶点缓冲区仍然正常工作

## Using a staging buffer

现在使用一个主机可见缓冲区作为临时缓冲区，并使用一个设备本地缓冲区作为实际的顶点缓冲区
使用stagingBuffer缓冲区，和原本的vertexBuffer缓冲区
对于缓冲区的usage，有如下两种

* VK_BUFFER_USAGE_TRANSFER_SRC_BIT缓冲区可作为源用于内存传输操作
* VK_BUFFER_USAGE_TRANSFER_DST_BIT缓冲区可作为传输操作中的目的地使用

对于vertexBuffer我们使用第二种，因此意味着我们不能使用vkMapMemory，但是，我们可以将数据从stagingBuffer复制到vertexBuffer
vkAllocateCommandBuffers创建临时的commandBuffer
vkBeginCommandBuffer->vkCmdCopyBuffer传输缓冲区的内容，它将源缓冲区和目标缓冲区作为参数->vkEndCommandBuffer->vkQueueSubmit->vkFreeCommandBuffers清理用于传输操作的命令缓冲区
其中BeginCommandBuffer的flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT只使用一次命令缓冲区，并等待复制操作完成执行
这次我们不需要等待任何事件。我们只想立即在缓冲区上执行传输，
应该等待传输完成，可以通过之前的fence并使用vkWaitForFences进行等待，也可以vkQueueWaitIdle简单地等待传输队列空闲，fence允许您同时安排多个传输，并等待所有传输完成，而不是一次执行一个

# Index buffer

vulkan支持的基本图元包括点，线，三角形，当绘制非基本图元时通常会在多个三角形之间共享顶点，也就是我们将定义大量的重复顶点，这会需要更多的内存
为了节省内存需要Index buffer，它本质上是指向顶点缓冲区的指针数组，它允许您重新排序顶点数据，并为多个顶点重用现有数据

## Index buffer creation

这次我们要绘制四边形，四个Vertex和六个index
新增一个createIndexBuffer函数，和createVertexBuffer的差异在，bufferSize和VK_BUFFER_USAGE_INDEX_BUFFER_BIT除此之外，过程完全相同
也不要忘记vkDestroyBuffer和vkFreeMemory

## Using an index buffer

我们应该对recordCommandBuffer进行更改
就像顶点缓冲区首先要vkCmdBindIndexBuffer绑定，
还需要更改vkCmdDrawIndexed绘图命令以告知Vulkan使用索引缓冲区
![image](assets/img/blog/vulkan/rectangle.png)

# repository

对于已经commit到本地仓库的如何撤销

* git log 查看commit记录
* git fsck --lost-found可以查看已经删除的commit
* git reset <commit_id> 恢复到某id版本，此id为保留的id非删除的id

对于已经push到远程仓库的如何撤销

* git reset --mixed  <commit_id>（千万不要用--hard，否则本地的也没了＞﹏＜，幸好恢复成功……）

<script src="https://utteranc.es/client.js"
        repo="sengyongan/sengyongan.github.io"
        issue-term="pathname"
        label="Comment"
        theme="dark-blue"
        crossorigin="anonymous"
        async>
</script>