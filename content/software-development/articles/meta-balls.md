+++
date = '2026-04-29T20:10:07+01:00'
title = 'Meta Balls'
+++

At Lush I was asked to work on something that was coined Bubble Nav, it looks like this:

![Bubbles][1]

Obviously, as you can see, I managed to implement it. What you can't see that in image is that it is also fully animated. That view is dynamically generated at runtime, and is implemented using shaders. Well, actually, that's a bit of a leap, but I'll get into the details shortly. First, let me explain what is actually happening here at a conceptual level.

From a design perspective, each of the circles in the above image is a clickable "node" within a nav tree, and clicking it will take you to either a sub tree (if the node was the root of a sub-tree) or, it will deep link you to some content elsewhere in the app if the node was a leaf node. Furthermore, the tree is supplied as a structure from the back end in the form of JSON, and each node has an associated color and text property. The design intent was to have the circles "blend with one another" and to fill the screen space dynamically, so if there are more nodes, the tree needs to grow organically to fill the screen space, keeping each node as a circle arranged around a central circle concentrically, with some chaos thrown in so it doesn't look too symetrical.


The first thing I did was to do some research. After a few hours, I came across something called [Metaballs][2] which looked promising. After reading that article, and getting ideas far above my station, I started in earnest learning shaders, something I've never done b efore.

The plan then was to implement a fully custom shader using OpenGL ES 2.0 which is the OpenGL standard that Android supports. I made some pretty good headway with that, but ultimately, the fact I'd never written a shader in my life before so was effectively having to learn GPU programming from scratch, combined with deadline pressure meant I had to return to the drawing board. In any case, I learned all about shader pipelines, vertex and fragment, UVs, supplying data from the CPU to the GPU etc, so it wasn't totally time wasted.

So, back to the drawing board, a bit frustrated and feeling the burn of the impending deadline I was researching alternatives to my initial approach, and talked to colleague on the iOS team who had made more progress than me (side note, this seems to be a common theme, those iOS devs are another breed). The approach they had taken was compltely different to the shader approach, but the results were just as impressive. They had effectively taken a more "compositional approach" with doing post processing effects on a rasterised image. The steps were to draw colored cirlces to a bitmap, then blur them with a guassian blur with a quite large radius, then to pass the whole bitmap through a matrix transform applying threshold to "cut out" any pixels that were too transparent (over the threshold), then finally pumping up the resulting images alpha to fully opaque again to get the metaball effect.

So, I implemented that on Android. But, it was super slow... like way too slow to be acceptable, because remember, this whole view animates, when you click the nodes, they need to collapse in and re-expand to the new tree with a boingy effect, so all this bitmap transformations happening on the CPU, every frame, even on a background thread, was just unnnaceptably slow, so I needed a way to offload this work to the GPU. I managed to do that using [RenderNode][3] which allows us to setup render pipelines to run on the GPU, kind of like a halfway house between writing a shader and doing higher level code.

All in all then, this was some complicated shit, but I pulled it off! I think I could improve the look and feel of it further, but it is stable and runs at a silky smooth 60fps even on crappy older devices. I'm not sold on the actual UX of this view, but that's above my paygrade to be honest, I did protest the entire time about the fundamental UX of it but it didn't make any difference, the business wanted Bubbles, so Bubbles the business would have...

Here is the code that produces the wonderful Bubble view:

```
@RequiresApi(Build.VERSION_CODES.S)
@Composable
fun GooeyCanvas(
    modifier: Modifier = Modifier,
    renderTreeNodes: List<RenderTreeNode>,
    renderTreeState: TransitionState<RenderTreeState>,
    onTransitionComplete: (RenderTreeState) -> Unit
) {
    val context = LocalContext.current
    val dynamicTextSize = with(LocalDensity.current) {
        16f.sp.toPx()
    }
    val size = remember {
        mutableStateOf(Size.Zero)
    }
    val blurRadius = with(LocalDensity.current) {
        26.dp.toPx()
    }
    val filterThreshold = with(LocalDensity.current) {
        17.dp.toPx()
    }
    val renderNode = remember(size.value) {
        val blurRenderEffect = RenderEffect.createBlurEffect(
            blurRadius,
            blurRadius,
            Shader.TileMode.MIRROR
        )
        val colorFilterEffect = RenderEffect.createColorFilterEffect(
            ColorMatrixColorFilter(
                floatArrayOf(
                    1f, 0f, 0f, 0f, 0f,
                    0f, 1f, 0f, 0f, 0f,
                    0f, 0f, 1f, 0f, 0f,
                    0f, 0f, 0f, 200f, -255f * filterThreshold
                )
            )
        )
        val gooeyEffect = RenderEffect.createChainEffect(
            colorFilterEffect,
            blurRenderEffect
        )
        val newRenderNode = RenderNode("Gooey")
        newRenderNode.setPosition(0, 0, size.value.width.toInt(), size.value.height.toInt())
        newRenderNode.setRenderEffect(gooeyEffect)
        newRenderNode
    }

    val transitions = renderTreeNodes.associateWith { node ->
        val transition = rememberTransition(
            transitionState = renderTreeState,
            label = "Menu State"
        )
        val textAlpha by transition.animateInt(
            label = "Alpha",
            transitionSpec = {
                when (targetState) {
                    RenderTreeState.FIRST_SHOW -> snap()
                    RenderTreeState.ENTER -> snap()
                    RenderTreeState.EXPANDED -> tween(250)
                    RenderTreeState.EXIT -> tween(250)
                }
            }
        ) { state ->
            when (state) {
                RenderTreeState.ENTER, RenderTreeState.FIRST_SHOW -> when (node) {
                    is RenderTreeNode.Close -> 255
                    is RenderTreeNode.Other -> 0
                }
                RenderTreeState.EXPANDED -> 255
                RenderTreeState.EXIT -> when (node) {
                    is RenderTreeNode.Close -> 255
                    is RenderTreeNode.Other -> 0
                }
            }
        }
        val offset by transition.animateOffset(
            label = "Offset",
            transitionSpec = {
                when (targetState) {
                    RenderTreeState.FIRST_SHOW -> snap()
                    RenderTreeState.ENTER -> snap()
                    RenderTreeState.EXPANDED -> tween(250)
                    RenderTreeState.EXIT -> tween(250)
                }
            }
        ) { state ->
            when (state) {
                RenderTreeState.ENTER, RenderTreeState.FIRST_SHOW -> when (node) {
                    is RenderTreeNode.Close -> node.position
                    is RenderTreeNode.Other -> node.enterPosition
                }
                RenderTreeState.EXPANDED -> when (node) {
                    is RenderTreeNode.Close -> node.position
                    is RenderTreeNode.Other -> node.expandedPosition
                }
                RenderTreeState.EXIT -> when (node) {
                    is RenderTreeNode.Close -> node.position
                    is RenderTreeNode.Other -> node.exitPosition
                }
            }
        }

        val radius by transition.animateFloat(
            label = "Radius",
            transitionSpec = {
                when (targetState) {
                    RenderTreeState.FIRST_SHOW -> snap()
                    RenderTreeState.ENTER -> snap()
                    RenderTreeState.EXPANDED -> tween(250)
                    RenderTreeState.EXIT -> tween(250)
                }
            }
        ) { state ->
            when (state) {
                RenderTreeState.ENTER, RenderTreeState.FIRST_SHOW -> when (node) {
                    is RenderTreeNode.Close -> node.radius
                    is RenderTreeNode.Other -> node.enterRadius
                }
                RenderTreeState.EXPANDED -> when (node) {
                    is RenderTreeNode.Close -> node.radius
                    is RenderTreeNode.Other -> node.expandedRadius
                }
                RenderTreeState.EXIT -> when (node) {
                    is RenderTreeNode.Close -> node.radius
                    is RenderTreeNode.Other -> node.expandedRadius
                }
            }
        }
        TransitionData(
            transition = transition,
            textAlpha = textAlpha,
            offset = offset,
            radius = radius
        )
    }

    LaunchedEffect(
        key1 = renderTreeState.targetState,
        key2 = renderTreeState.currentState
    ) {
        val transitionsFinished = renderTreeState.currentState == renderTreeState.targetState
        if (transitionsFinished)
            onTransitionComplete(renderTreeState.currentState)
    }

    Canvas(
        modifier = modifier
            .onGloballyPositioned { layoutCoordinates ->
                size.value = layoutCoordinates.size.toSize()
            }
    ) {
        val canvas = renderNode.beginRecording()
        renderTreeNodes.forEach { node ->
            val radius = transitions[node]?.radius ?: 0f
            val offset = transitions[node]?.offset ?: Offset.Zero
            val blobColor = when (node) {
                is RenderTreeNode.Close -> Color.Black.toArgb()
                is RenderTreeNode.Other -> Color(node.color).let {
                    val hsl = floatArrayOf(0f, 0f, 0f)
                    ColorUtils.colorToHSL(it.toArgb(), hsl)
                    hsl[1] = hsl[1].coerceAtMost(0.8f)
                    hsl[2] = hsl[2].coerceAtLeast(0.6f)
                    ColorUtils.HSLToColor(hsl)
                }
            }
            canvas.drawCircle(
                offset.x,
                offset.y,
                radius,
                Paint().apply {
                    style = Paint.Style.FILL_AND_STROKE
                    color = blobColor
                    blendMode = BlendMode.LIGHTEN
                }
            )
        }
        renderNode.endRecording()
        drawIntoCanvas {
            it.nativeCanvas.drawRenderNode(renderNode)
            renderTreeNodes.forEach { node ->
                val transition = transitions[node]
                when (node) {
                    is RenderTreeNode.Close -> {
                        val drawable = ResourcesCompat.getDrawable(
                            context.resources, node.drawableId, null)
                        val iconSize = (transition?.radius?.toInt() ?: 0) / 2
                        val left = (transition?.offset?.x?.toInt() ?: 0) - iconSize / 2
                        val top = (transition?.offset?.y?.toInt() ?: 0) - iconSize / 2
                        drawable?.setBounds(
                            left,
                            top,
                            left + iconSize,
                            top + iconSize
                        )
                        drawable?.setTint(context.getColor(R.color.white))
                        drawable?.alpha = transition?.textAlpha ?: 0
                        drawable?.draw(it.nativeCanvas)
                    }
                    is RenderTreeNode.Other.Backward -> {
                        val drawable = ResourcesCompat.getDrawable(
                            context.resources, R.drawable.ic_back, null)
                        val iconSize = (transition?.radius?.toInt() ?: 0) / 2
                        val left = (transition?.offset?.x?.toInt() ?: 0) - iconSize / 2
                        val top = (transition?.offset?.y?.toInt() ?: 0) - iconSize
                        drawable?.setBounds(
                            left,
                            top,
                            left + iconSize,
                            top + iconSize
                        )
                        drawable?.setTint(context.getColor(R.color.black))
                        drawable?.alpha = transition?.textAlpha ?: 0
                        drawable?.draw(it.nativeCanvas)
                        val textPaint = TextPaint().apply {
                            val minRadius = node.enterRadius
                            val maxRadius = node.expandedRadius
                            val currentRadius = transition?.radius ?: minRadius
                            val normalisedRadius = if (minRadius == maxRadius) {
                                1f
                            } else {
                                (currentRadius - minRadius) / (maxRadius - minRadius) * 1f
                            }
                            textSize = dynamicTextSize
                            textScaleX = normalisedRadius
                            textAlign = Paint.Align.CENTER
                            typeface = context.resources.getFont(R.font.inter_variable)
                            color = android.graphics.Color.BLACK
                            alpha = transition?.textAlpha ?: 0
                        }
                        val textLayout = StaticLayout.Builder.obtain(
                            node.text,
                            0,
                            node.text.length,
                            textPaint,
                            (node.expandedRadius * 2f).toInt()
                        ).setBreakStrategy(LineBreaker.BREAK_STRATEGY_SIMPLE)
                            .setHyphenationFrequency(Layout.HYPHENATION_FREQUENCY_NORMAL)
                            .build()
                        transition?.offset?.also { offset ->
                            val yOffset = offset.y + iconSize / 2f - (textLayout.height / 2f)
                            it.nativeCanvas.save()
                            it.nativeCanvas.translate(offset.x, yOffset)
                            textLayout.draw(it.nativeCanvas)
                            it.nativeCanvas.restore()
                        }
                    }
                    is RenderTreeNode.Other -> {
                        val textPaint = TextPaint().apply {
                            val minRadius = node.enterRadius
                            val maxRadius = node.expandedRadius
                            val currentRadius = transition?.radius ?: minRadius
                            val normalisedRadius = if (minRadius == maxRadius) {
                                1f
                            } else {
                                (currentRadius - minRadius) / (maxRadius - minRadius) * 1f
                            }
                            textSize = dynamicTextSize
                            textScaleX = normalisedRadius
                            textAlign = Paint.Align.CENTER
                            typeface = context.resources.getFont(R.font.inter_variable)
                            color = android.graphics.Color.BLACK
                            alpha = transition?.textAlpha ?: 0
                        }
                        val textLayout = StaticLayout.Builder.obtain(
                            node.text,
                            0,
                            node.text.length,
                            textPaint,
                            (node.expandedRadius * 2f).toInt()
                        ).setBreakStrategy(LineBreaker.BREAK_STRATEGY_SIMPLE)
                            .setHyphenationFrequency(Layout.HYPHENATION_FREQUENCY_NORMAL)
                            .build()
                        transition?.offset?.also { offset ->
                            val yOffset = offset.y - (textLayout.height / 2f)
                            it.nativeCanvas.save()
                            it.nativeCanvas.translate(offset.x, yOffset)
                            textLayout.draw(it.nativeCanvas)
                            it.nativeCanvas.restore()
                        }
                    }
                }
            }
        }
    }
}
```

[1]: https://drive.google.com/file/d/1uzEErv_Rl_yY4aQ_7Z8lxW8HdL1IiRE9/view?usp=drive_link
[2]: https://matiaslavik.wordpress.com/computer-graphics/metaball-rendering/
[3]: https://developer.android.com/reference/android/graphics/RenderNode
