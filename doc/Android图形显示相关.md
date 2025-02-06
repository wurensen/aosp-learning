[TOC]

# Android图形显示相关

先临时记录，后续再重新整理

## 图形显示系统核心：SurfaceFlinger



## 开启硬件加速下的绘制流程

`ViewRootImpl.draw`函数：

```java
ViewRootImpl
private boolean draw(boolean fullRedrawNeeded) {
    ...
    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
        if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
            ...
			// 调用硬件加速Renderer的draw函数
            mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
        } else {
            ...
        }
    }

    ...
    return useAsyncReport;
}


ThreadedRenderer
/**
 * Draws the specified view.
 *
 * @param view The view to draw.
 * @param attachInfo AttachInfo tied to the specified view.
 */
void draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks) {
    final Choreographer choreographer = attachInfo.mViewRootImpl.mChoreographer;
    choreographer.mFrameInfo.markDrawStart();

    updateRootDisplayList(view, callbacks);

    ...
    // 调用android_view_ThreadedRenderer_syncAndDrawFrame函数，把RenderNode树同步到RenderThread线程执行请求绘制
    int syncResult = nSyncAndDrawFrame(mNativeProxy, frameInfo, frameInfo.length);
    ...
}

private void updateRootDisplayList(View view, DrawCallbacks callbacks) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Record View#draw()");
    updateViewTreeDisplayList(view);

    ...
}

private void updateViewTreeDisplayList(View view) {
    view.mPrivateFlags |= View.PFLAG_DRAWN;
    view.mRecreateDisplayList = (view.mPrivateFlags & View.PFLAG_INVALIDATED)
            == View.PFLAG_INVALIDATED;
    view.mPrivateFlags &= ~View.PFLAG_INVALIDATED;
    // 调用View（对于Activity，view即为DecorView）的updateDisplayListIfDirty，
    view.updateDisplayListIfDirty();
    view.mRecreateDisplayList = false;
}

View
/**
 * Gets the RenderNode for the view, and updates its DisplayList (if needed and supported)
 * @hide
 */
@NonNull
@UnsupportedAppUsage
public RenderNode updateDisplayListIfDirty() {
    ...

    if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0
                || !renderNode.isValid()
                || (mRecreateDisplayList))) {
        ...

        // 通过每个View自己对应的RenderNode获取canvas
        final DisplayListCanvas canvas = renderNode.start(width, height);

        try {
            // 当前View如果是软件绘制
            if (layerType == LAYER_TYPE_SOFTWARE) {
                buildDrawingCache(true);
                Bitmap cache = getDrawingCache(true);
                if (cache != null) {
                    canvas.drawBitmap(cache, 0, 0, mLayerPaint);
                }
            } else {
                computeScroll();

                canvas.translate(-mScrollX, -mScrollY);
                mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;

                // Fast path for layouts with no backgrounds
                if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                    // 设置了跳过绘制自身标记，直接走dispatchDraw调度child绘制
                    dispatchDraw(canvas);
                    drawAutofilledHighlight(canvas);
                    if (mOverlay != null && !mOverlay.isEmpty()) {
                        mOverlay.getOverlayView().draw(canvas);
                    }
                    if (isShowingLayoutBounds()) {
                        debugDrawFocus(canvas);
                    }
                } else {
                    // 调用draw，进入View/ViewGroup的分发流程
                    draw(canvas);
                }
            }
        } finally {
            // 结束绘制指令的录制
            renderNode.end(canvas);
            setDisplayListProperties(renderNode);
        }
    } else {
        mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
        mPrivateFlags &= ~PFLAG_DIRTY_MASK;
    }
    return renderNode;
}
```

`Canvas`的相关绘制接口可以理解为是一个个的绘制指令，调用会把绘制操作进行录制，最终都是去更新底层`RenderNode`的`DisplayList`，最终通过`ThreadedRenderer.nSyncAndDrawFrame()`请求底层的`RenderThread`线程进行绘制。

## 疑问：TextureView为何支持缩放旋转？

`Java`代码：

```java
android.view.TextureView
/**
 * Subclasses of TextureView cannot do their own rendering
 * with the {@link Canvas} object.
 *
 * @param canvas The Canvas to which the View is rendered.
 */
@Override
public final void draw(Canvas canvas) {
    // NOTE: Maintain this carefully (see View#draw)
    mPrivateFlags = (mPrivateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

    /* Simplify drawing to guarantee the layer is the only thing drawn - so e.g. no background,
    scrolling, or fading edges. This guarantees all drawing is in the layer, so drawing
    properties (alpha, layer paint) affect all of the content of a TextureView. */
	// 需要开启硬件加速才执行
    if (canvas.isHardwareAccelerated()) {
        DisplayListCanvas displayListCanvas = (DisplayListCanvas) canvas;
		// 如果未创建mLayer，就创建
        TextureLayer layer = getTextureLayer();
        if (layer != null) {
            // 应用更新
            applyUpdate();
            // 应用变化矩阵
            applyTransformMatrix();

            mLayer.setLayerPaint(mLayerPaint); // ensure layer paint is up to date
            // 把layer的内容绘制到canvas（底层最终会把该layer通过Drawable的形式绘制到canvas上）
            displayListCanvas.drawTextureLayer(layer);
        }
    }
}
```

`Native`代码：

```c++
frameworks/base/core/jni/android_view_DisplayListCanvas.cpp
static void android_view_DisplayListCanvas_drawTextureLayer(CRITICAL_JNI_PARAMS_COMMA jlong canvasPtr, jlong layerPtr) {
    Canvas* canvas = reinterpret_cast<Canvas*>(canvasPtr);
    DeferredLayerUpdater* layer = reinterpret_cast<DeferredLayerUpdater*>(layerPtr);
    canvas->drawLayer(layer);
}

frameworks/base/libs/hwui/pipeline/skia/SkiaRecordingCanvas.cpp
void SkiaRecordingCanvas::drawLayer(uirenderer::DeferredLayerUpdater* layerUpdater) {
#ifdef __ANDROID__ // Layoutlib does not support Layers
    if (layerUpdater != nullptr) {
        // Create a ref-counted drawable, which is kept alive by sk_sp in SkLiteDL.
        sk_sp<SkDrawable> drawable(new LayerDrawable(layerUpdater));
        drawDrawable(drawable.get());
    }
#endif
}
```

首先`TextureView`的`draw`函数，其实是把当前`SurfaceTexture`的`layer`绘制到当前`canvas`上，底层内部的实现本质上是调用`SkiaRecordingCanvas.drawLayer()->drawDrawable()`接口，最终会调用`SkCanvas.drawDrawable()`；本质上是绘制到当前`View`树上，所以也能支持缩放、透明度等操作。