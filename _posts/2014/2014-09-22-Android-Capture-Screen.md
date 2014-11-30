---
layout: post
title: "Android Capture Screen"
categories:
- android
tags:
- graphics
- screen capture
- surfaceflinger


---

当同时按下POWER键和音量下键的时候，安卓就会产生截屏操作。触发的场景不限于此，按HOME或BACK键退出app，或是发生横竖屏切换的时候，都会触发截屏，只不过此时没有截屏动画。也可以使用以下命令进行截屏。

~~~
adb shell screencap -p /sdcard/tmp.png
~~~

只有系统的图层合成者（composer）才知道当前系统中各个图层（layer）是什么样子，所以截屏工作的主要承担者就是安卓的Composer——SurfaceFlinger（SF）。SF中维护了一个系统的图层集合（layer list)，考虑到某些图层可能是半透明的，所以会将图层按Z序从下到上排好顺序，方便后续合成。SF会把当前所有可见的图层，都通过OpenGL ES合成进一个buffer中。因为是离屏渲染，所以这一步一般是通过framebuffer object(FBO)来实现的。具体的实现很简单，对每个图层来说都画一个矩形，将图层作为一张纹理，即可贴到FBO的目标buffer中。合成完以后，可以用合成的buffer来创建一个新的layer，用来显示截屏动画或是旋转动画，或是简单地保存成一个图片文件。

因为OpenGL是系统无关的，所以Android提供的native buffer不能直接作为render target被GLES使用。幸好，EGL已经提供了办法。利用EGL的API可以将一个本地窗口系统提供的native buffer转换成一个EGLImage，这是EGL用来实现在多种context之间共享资源的途径。利用它，native buffer就可以光明正大的attach到一个FBO上，作为截屏时候的渲染目标。

截屏还有个特殊性。一般情况下，Android BufferQueue的Bp端在APP（生产者），Bn端在SF（消费者）。在截屏场景下却是相反的，因为SF成了生产者。截屏时，screenshot进程或WMS，会发送一个IGraphicBufferProducer的匿名Binder给SF，SF进程正是使用这个Binder，来dequeue一块buffer -> 合成 -> queueBuffer的。具体的细节可参考下一篇——[Android Binder环](/2014/09/android-binder环/)。

大致的代码过程如下：

```C
result = native_window_dequeue_buffer_and_wait(window,  &buffer);
...

EGLImageKHR image = eglCreateImageKHR(mEGLDisplay, EGL_NO_CONTEXT,
    EGL_NATIVE_BUFFER_ANDROID, buffer, NULL);
...

    glGenTextures(1, &tname);
    glBindTexture(GL_TEXTURE_2D, tname);
    glEGLImageTargetTexture2DOES(GL_TEXTURE_2D, (GLeglImageOES)image);

    // create a Framebuffer Object to render into
    glGenFramebuffers(1, &name);
    glBindFramebuffer(GL_FRAMEBUFFER, name);
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, tname, 0);
...

        for (size_t i=0 ; i<count ; ++i) {
            const sp<Layer>& layer(layers[i]);
            const Layer::State& state(layer->getDrawingState());
            if (state.layerStack == hw->getLayerStack()) {
                if (state.z >= minLayerZ && state.z <= maxLayerZ) {
                    if (layer->isVisible()) {
                        if (filtering) layer->setFiltering(true);
                        layer->draw(hw);
                        if (filtering) layer->setFiltering(false);
                    }
                }
            }
        }
...

EGLSyncKHR sync = eglCreateSyncKHR(mEGLDisplay, EGL_SYNC_FENCE_KHR, NULL);
if (sync != EGL_NO_SYNC_KHR) {
    EGLint result = eglClientWaitSyncKHR(mEGLDisplay, sync, //等待合成完成
        EGL_SYNC_FLUSH_COMMANDS_BIT_KHR, 2000000000 /*2 sec*/);
...

window->queueBuffer(window, buffer, -1);
```

