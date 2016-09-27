首先，介绍一些一些基本概念 

VSYNC 信号，VSYBC 是linux kernel 内核定时发送给 view 系统的关于输入处理，动画，绘制的事件。

Chorographer 是一个android应用View系统和底层渲染系统的一个纽带。

只有 Input Handle，Animation，Draw 三种事件。

linux 内核发送事件到ViewRootImpl,然后  ViewRootImpl 把事件发送到 Chorographer，Chorographer 在把事件发送到底层的 View 渲染系统。