# Accelerated compositing in WebKit: Now and in the future
Gwang Yoon Hwang, yoon@igalia.com
----


## Who am I?

- Gwang Yoon Hwang
- Hacker in Igalia, S. L.
- Working on WebKit Project, focused on rendering performance of WebKitGTK+ in embedded environment

![](images/igalia-cover.png)

----


# Before we begin
###What is the Compositing?

----

If we have a single backing store, we need to redraw almost everything for each frame

If green and red have their own backing stores, then nothing needs "re-rasterizing" while this example animates.
<!-- .element: class="fragment" -->

<iframe data-src="./examples/simple-animation.html" width="100%" height="800px" clss="stretch"></iframe>

----

## Compositing
### The use of multiple backing stores to cache and group chunks of the render tree.
<small>From Shawn Singh's talk: https://docs.google.com/presentation/d/1dDE5u76ZBIKmsqkWi2apx3BqV8HOcNf4xxBdyNywZR8</small>

----

## Steps for Rendering
- Parsing: Nodes to DOM Tree
- Constructing RenderObject Tree to RenderLayer Tree

----

### Parsing: Creates the DOM Tree from source
![Parsing and DOM constructing](./images/parsing-and-constructing-dom.png)

<small>From: https://developers.google.com/web/fundamentals/performance/critical-rendering-path/
<br>Under the Creative Commons Attribution 3.0 Licenses</small>

----

### Creates the Render Tree from the DOM and the CSSOM
![Render Tree Construction](./images/render-tree-construction.png)

<small>From: https://developers.google.com/web/fundamentals/performance/critical-rendering-path/
<br>Under the Creative Commons Attribution 3.0 Licenses</small>

----

### Creates the GraphicsLayer Tree and Composite it to the screen
![Render Tree to Compositor](./images/rendertree-to-compositor.svg)

----

## Accelerate Compositing
- Do not have to re-rasterize entire page for each animated frame
 - Rasterization is expensive operation

- Composite page layers on the GPU can achieve far better efficiency than the CPU
 - GPU is specialized to handle large number of pixels

- Provide more efficent/practical ways to support features
  - Scrolling, 3D CSS, opacity, filters, WebGL, hardware video decoding, etc.

----

## This is not the focus of this talk

----

## Unfortunately, we still have problems

- The main-thread is always busy (Parsing, Layout, JS ...)
- The main-thread can be blocked by VSync
- And we want awesome webpages which uses HTML5 features

----

## Example: If we have layout operations during a animation
<iframe data-src="./examples/animation-with-layout.html" width="100%" height="900px" clss="stretch"></iframe>

----

## Timeline of the previous example: Best Case
![Timeline of the previous example: Best case](./images/rendering-only-main-thread.png)

----

## Timeline of the previous example: Worse Case
![Timeline of the previous example: Worse case](./images/rendering-only-main-thread-bad.png)

----

## Timeline of the previous example: Even Worse Case
![Timeline of the previous example: Worst case](./images/Threaded_Compositor_Vsync2.png)

----

# Off-the-main-thread Compositing

----

![Concept of Off-the-main-thread compositing](./images/concept-of-off-the-main-thread-compositing.png)
<!-- .element: class="stretch" -->

----

## Compositing in the dedicated thread / or process

- The main-thread don't have to care about Vsync and compositing operations
- It shows more smooth CSS animations, zoom, and scale operations.

----

### Worst case
![Timeline of the previous example: Worst case](./images/Threaded_Compositor_Vsync2.png)
### Same case with dedicated compositing thread
![Timeline of the previous example: Threaded case](./images/Threaded_Compositor_Vsync1.png)

----

## What we are (going to) using now
### Coordinated Graphics with threaded or process model

- It implement a dedicated compositing thread in WebProcess or UIProcess.
- Depends on OpenGL[ES] only: Easy to port to other enviroment

----

<!-- .slide: data-state="no-title-footer" -->
![Coordinated Graphics: Model](./images/coordinated-graphics-model.svg)

----

### Sequence Diagram: Updating Animation 1
![Sequence Diagram: Updating Animation 1](./images/Threaded_Compositor_Update_Animation_1.png)

----

### Sequence Diagram: Updating Animation 2
![Sequence Diagram: Updating Animation 2](./images/Threaded_Compositor_Update_Animation_2.png)

----

### Sequence Diagram: Scrolling without compositing thread
![Scrolling without compositing thread](./images/Threaded_Compositor_Simplified_Normal_Scrolling.png)

----

### Sequence Diagram: Scrolling with compositing thread
![Scrolling with compositing thread](./images/Threaded_Compositor_Simplified_Threaded_Scrolling.png)

----

## So, Off-the-main-thread Compositing does

- Utilize multi-core CPUs and GPU
- Play CSS Animation off-the-main-thread
- Reduce latencies of scrolling and scaling operations

----

# WebGL, HTML5 2D Canvas and HTML5 Video

----

## Problem of Supporting WebGL

- WebGL executes OpenGLES commands at the main-thread in WebProcess.
 - Synchronization!
- Anti-aliasing consumes a lot of memory bandwidth

----

### Synchronization: UI SIDE COMPOSITING Case

![Cross Process Rendering: WebGL](./images/WebGL-crossprocess.svg)

----

### Cross Process Shareable GL Surface

- Texture is not shareable across processes
- X Windows System: Allocate memory in the X server via pixmap or offscreen window
- The rendered frame buffer should be copied to **a cross-process-sharable** GL surface
<!-- .element: class="fragment" -->
- Needs cross process synchronization between updates
 - No standards


----

### Synchronization: Threaded Case

![Cross Thread Rendering: WebGL](./images/WebGL-crossthread.svg)

----

### Synchronization: Threaded Case

- Uses normal texture from GraphicsContext3D
- Uses double buffer without coping textures
- Needs cross thread synchronization between updates
 - EGLSyncObject (Unstable)

----

## Problem of Supporting HTML5 2D Canvas

### Same with WebGL's case
- GPU Accelerated 2D vector graphics library executes OpenGLES commands at the main-thread in WebProcess.
 - Synchronization!
- Anti-aliasing consumes a lot of memory bandwidth

----

### Solutions are also almost same with WebGL, except:
We cannot avoid texture copy operations to support accumulate rendering

<iframe data-src="./examples/canvas.html" width="1200px" height="600px" clss="stretch"></iframe>

----

## Problem of Supporting HTML5 2D Video

### A little bit different with WebGL's case
- GPU Accelerated video decoder uses GPU's API at the decoder threads in WebProcess.
 - Synchronization!

----

### Solutions are also almost same with WebGL, except:
- Need to implement multimedia backend specific codes to avoid texture copies
- Need to implement a S/W fallback

----

![Compositing Platform layers](./images/Threaded_Compositor_Simplified_Platform_Layers.png)

----

# Current Status and Future

----

## Current Status of Coordinated Graphics : Threaded Mode

- You can build WebKitGTK+ with a --threaded-compositor flag
- However, It doesn't support WebGL, Canvas and Video yet in the upstream
- Most of codes to support those features are ready, however we need to test it seriously before using it as a default

----

## Current Status of Coordinated Graphics : Process Mode

- WebKitEFL uses it as a default
- Fully functional

----

## To Do

### Add a compositing thread to the Process Mode

### Rasterize contents to textures

### Unify codes with Apple's rendering system

----

# Thank You!
