本文主要记录了Launcher3拖动时的流程和代码记录，在桌面图标拖动时会引起图标的重排，拖动时受影响的图标在文中由item或cell来表示。
###图标点击效果和摇动效果
mTouchFeedbackView为点击按下时图标表面一层透明灰色动画效果，这个动画效果在Celllayout中，在构造函数中addView加入，BubbleTextView 中 setStayPressed方法调用这个动画效果，在点击图标时呈现，下图就是我把Wallpaper改变成纯白色，更改动画位置，这样效果很明显。

<img src="http://img.blog.csdn.net/20160501104528625" width="300" height="300" alt="点击图标动画">

```java
addView(mTouchFeedbackView, (int) (grid.cellWidthPx * 1.5), (int) (grid.cellHeightPx * 1.5));
addView(mShortcutsAndWidgets);
```

拖动时受影响的Item有一个摇动的(shake)动画，这个动画效果是通过ReorderPreviewAnimation和mShakeAnimators实现，受影响的item由ItemConfiguration管理，Celllayout-completeAndClearReorderPreviewAnimations方法播放动画，在适当时机调用。这个动画比较有意思，单纯的线性动画没有这种效果，看了代码才发现用到了贝塞尔曲线线性方程，这部分代码主要在Celllayout中。

###图标的占位
Celllayout的boolean[][] mOccupied存储桌面图标布局中是否有图标占位，存在则为true，该对象在CellLayout的addViewToCellLayout方法赋值，这个方法在Workspace-addInScreen方法中调用，而桌面初始化时的bind过程会一直调用到这个方法。

###图标拖动处理

####DragController和DragLayer
DragLayer是Launcher3中处理拖动图标事件时自定义View，DragController的注释为：发起一个在一个View内或在几个View之间拖动事件的类。从名字可以看出这个类是一个调节者或控制者。

在DragLayer-onInterceptTouchEvent方法将事件给DragController-onInterceptTouchEvent方法处理。拖动图标的事件开始处理时，会一直调用到DragController-startDrag方法，此时，startDrag里将mDragging置为true,而DragController-onInterceptTouchEvent返回mDragging,此时截断事件传递，当前View处理触摸事件。类似的，DragLayer的onTouchEent方法也将事件给DragController的onTouchEent方法处理。

handleMoveEvent方法为处理触摸事件的主要方法，在handleMoveEvent方法中调用findDropTarget取的当前的具体的DropTarget对象，DragController中有一个mDropTargets链表，这个链表中在桌面的加载和操作会包含进来具体DropTarget对象，在Launcher3中，实现DropTarget类的有Folder和Workspace等，这个interface抽象了拖动时落点对象。DropTarget-isDropEnabled方法，表示当前这个落点View是否可以Drop，当该方法返回false时，显然是不能将拖动的Object放到这里来的。checkTouchMove方法检查拖动时的状态，根据拖动时DropTarget之间的关系调用DropTarget声明的onDrop，onDragEnter,onDragExit,onDragOver方法进行视图的演示，最后调用checkScrollState方法，这个方法主要是对拖动时的翻页进行判断处理。

####DropTarget为Workspace的拖动处理
根据checkTouchMove方法，在Workspace中开始拖动图标时，图标开始时在Workspace中，此时DropTarget也为Workspace，所以刚开始时调用Workspace的onDragEnter方法，紧接着调用Workspace的onDragOver方法。而onDragExit方法的调用时机为拖动的Cell离开当前Target，当前Target变为mLastDropTarget，又没有进入一个有效的Target时。
Workspace的onDragEnter方法，这个方法比较简单，对Workspace的拖动时的涉及到的变量进行了初始化。

Workspace的onDragOver方法，在Workspace中有一个mDragViewVisualCenter变量，根据字面理解，这个变量是拖动对象的视觉中心位置，由getDragViewVisualCenter方法计算而来，在对拖动处理时多处使用，此外拖动的时候也要判断当前位置Layout的是处于翻动的页面还是Hotseat中，确定mDragTargetLayout。接着findNearestArea方法根据mDragViewVisualCenter先大致当前的落点，findNearestArea方法会调用到Celllayout的findNearestArea方法，根据不同情况在调用的过程中会分别考虑和不考虑当前页面的其他item的占用情况。确定了可能的落点后首先对Folder情况进行处理，分别为新建和加入已有的Folder两种，然后是落点已经被其他的Item占据的情况，这里调用performReorder方法处理，这个方法相当长，拖动过程中和结束拖动放下时的onDrag方法中都会调用这个方法，拖动过程中图标的重排效果就是在此方法中实现的。Workspace的onDragExit方法，根据上面onDragExit的调用时机，这里主要确定拖动图标离开有效DragTarget时最后的落点。

根据DragController的onTouchEent方法，触摸事件为MotionEvent.ACTION_UP时，拖动事件结束，这时有两种可能：当满足快速向上扔时，删除拖动的item，isFlingingToDelete方法判断条件是否满足。如果不满足扔的条件，调用drop方法，drop方法调用DropTarget的acceptDrop方法，判断该Drop事件是否发生，依此调用DropTarget的onDrop方法。

Workspace-onDrop()流程，方法参数为DropTarget-DragObject对象，首先计算拖动View的视觉中心mDragViewVisualCenter，dropTargetLayout为Drop的Celllayout对象，下一步判断当前是否在Hotseat上，求出相对于dropTargetLayout的视觉中心坐标。如果DragObject-dragSource！=Worspace，转而调用onDropExternal()，否则继续处理onDrop()的内容。接着调用findNearestArea方法球drop的xy的值，Workspace-findNearestArea()调用到CellLayout-findNearestArea(),该方法是一个重要的方法，作用是找到离落点最近距离cell，其中有一个boolean参数ignoreOccupied，当ignoreOccupied为false时，寻找cell时不考虑已经占据的区域，为true时考虑。

接下来开始处理落点和文件夹的关系,如果落点处可以合成一个Folder，调用Workspace-createUserFolderIfNecessary()方法，如果拖动的图标可以加进一个文件夹，则调用Workspace-addToExistingFolderIfNecessary()方法。如果不满足文件夹的条件，则调用CellLayout-performReorder方法，这个方法就是处理拖动图标时，如果当前落点被占据时，挤开当前图标的效果。

AppWidget可能在拖动时发生缩小，因此会调用AppWidgetResizeFrame-updateWidgetSizeRanges方法，拖动时可能落点在别的页面，所以还会有页面滑动的效果。如果满足则更新位置，保存新的位置信息到数据库中，播放动画效果，否则弹回原来位置。

acceptDrop方法的调用过程和逻辑和onDrop方法差不多，需要注意的是调用performReorder方法时mode参数不同。

除了DropTarget，WorkSpace还implements了DragController.DragListener，DragCtroller中存在一个mListeners链表，在桌面加载时会将相应的对象加进来，比如Workspace在Launcher的setupViews方法中加入。这个接口声明了onDragStart方法和onDragEnd方法,上面说过开始拖动时调用DragController的startDrag方法,startDrag方法中依次调用mListeners中对象的onDragStart方法，这里处理的是开始拖动时桌面受影响的View对象，比如刚开始拖动时Workspace有一些提示效果，搜索删除按钮(SearchDropTargetBar)会由搜索界面变为删除等，这些View对象都是继承了这个借口的。除此外，startDrag方法还显示了拖动item的DragView。Workspace的onDragStart方法处理了一些界面相关的操作，比如锁定屏幕方向，生成额外的空白页等。onDragEnd方法即为相反的操作。

Workspace还实现了一个拖动相关的接口为DragSource接口，这个接口定义了拖动可以从它本身开始的对象，就是可以从Workspace开始拖动一个图标，DragSorce声明了一系列拖动时判断条件或某种状态回调的方法。

####具体方法
findNearestArea方法，这个在拖动过程出镜频率相当高。
```java
    /**
     * Find a vacant area that will fit the given bounds nearest the requested
     * cell location. Uses Euclidean distance to score multiple vacant areas.
     */
    int[] findNearestArea(int pixelX, int pixelY, int minSpanX, int minSpanY, int spanX, int spanY,
            View ignoreView, boolean ignoreOccupied, int[] result, int[] resultSpan,
            boolean[][] occupied) {
        //初始化一个mCountX*mCountY的栈
        lazyInitTempRectStack();
        // mark space take by ignoreView as available (method checks if ignoreView is null)
        //此时occupied为mOccupied,这里将ignoreView的位置置为false,即忽略当前位置View
        markCellsAsUnoccupiedForView(ignoreView, occupied);
        // For items with a spanX / spanY > 1, the passed in point (pixelX, pixelY) corresponds
        // to the center of the item, but we are searching based on the top-left cell, so
        // we translate the point over to correspond to the top-left.
        // 即左上角cell的中心pixelX -= (mCellWidth*span + mWidthGap*(spanX-1))/2-mCellWidth/2
        pixelX -= (mCellWidth + mWidthGap) * (spanX - 1) / 2f;
        pixelY -= (mCellHeight + mHeightGap) * (spanY - 1) / 2f;

        // Keep track of best-scoring drop area
        final int[] bestXY = result != null ? result : new int[2];
        double bestDistance = Double.MAX_VALUE;
        final Rect bestRect = new Rect(-1, -1, -1, -1);
        final Stack<Rect> validRegions = new Stack<Rect>();
        final int countX = mCountX;
        final int countY = mCountY;

        if (minSpanX <= 0 || minSpanY <= 0 || spanX <= 0 || spanY <= 0 ||
                spanX < minSpanX || spanY < minSpanY) {
            return bestXY;
        }

        for (int y = 0; y < countY - (minSpanY - 1); y++) {
            inner:
            for (int x = 0; x < countX - (minSpanX - 1); x++) {
                int ySize = -1;
                int xSize = -1;
                
                //为true时代表要考虑已经被占据的位置
                //先寻找符合minSpanX和minSpanY的位置
                //然后向分x，y方向向外扩大，试探当前位置最大的可能性
				//findConfigurationNoShuffle方法调用时为true
                if (ignoreOccupied) {
                    // First, let's see if this thing fits anywhere
                    for (int i = 0; i < minSpanX; i++) {
                        for (int j = 0; j < minSpanY; j++) {
                            if (occupied[x + i][y + j]) {
                                continue inner;
                            }
                        }
                    }
                    xSize = minSpanX;
                    ySize = minSpanY;
                    
                    // We know that the item will fit at _some_ acceptable size, now let's see
                    // how big we can make it. We'll alternate between incrementing x and y spans
                    // until we hit a limit.
                    boolean incX = true;
                    //为true停止
                    boolean hitMaxX = xSize >= spanX;
                    boolean hitMaxY = ySize >= spanY;
                    while (!(hitMaxX && hitMaxY)) {
                        if (incX && !hitMaxX) {
                            for (int j = 0; j < ySize; j++) {
                                if (x + xSize > countX -1 || occupied[x + xSize][y + j]) {
                                    // We can't move out horizontally
                                    hitMaxX = true;
                                }
                            }
                                                // We know that the item will fit at _some_ acceptable size, now let's see
                    // how big we can make it. We'll alternate between incrementing x and y spans
                    // until we hit a limit.
                    boolean incX = true;
                    //为true停止
                    boolean hitMaxX = xSize >= spanX;
                    boolean hitMaxY = ySize >= spanY;
                    while (!(hitMaxX && hitMaxY)) {
                        if (incX && !hitMaxX) {
                            for (int j = 0; j < ySize; j++) {
                                if (x + xSize > countX -1 || occupied[x + xSize][y + j]) {
                                    // We can't move out horizontally
                                    hitMaxX = true;
                                }
                            }
                            if (!hitMaxX) {
                                xSize++;
                            }
                        } else if (!hitMaxY) {
                            for (int i = 0; i < xSize; i++) {
                                if (y + ySize > countY - 1 || occupied[x + i][y + ySize]) {
                                    // We can't move out vertically
                                    hitMaxY = true;
                                }
                            }
                            if (!hitMaxY) {
                                ySize++;
                            }
                        }
                        //为true或大于等于span
                        hitMaxX |= xSize >= spanX;
                        hitMaxY |= ySize >= spanY;
                        incX = !incX;
                    }
                    incX = true;
                    hitMaxX = xSize >= spanX;
                    hitMaxY = ySize >= spanY;
                }
                //求(x, y)的点的坐标
                final int[] cellXY = mTmpXY;
                cellToCenterPoint(x, y, cellXY);

                // We verify that the current rect is not a sub-rect of any of our previous
                // candidates. In this case, the current rect is disqualified in favour of the
                // containing rect.
                Rect currentRect = mTempRectStack.pop();
                currentRect.set(x, y, x + xSize, y + ySize);
                boolean contained = false;
                for (Rect r : validRegions) {
                    if (r.contains(currentRect)) {
                        contained = true;
                        break;
                    }
                }
                validRegions.push(currentRect);
                double distance = Math.sqrt(Math.pow(cellXY[0] - pixelX, 2)
                        + Math.pow(cellXY[1] - pixelY, 2));
                if ((distance <= bestDistance && !contained) ||
                        currentRect.contains(bestRect)) {
                    bestDistance = distance;
                    bestXY[0] = x;
                    bestXY[1] = y;
                    if (resultSpan != null) {
                        resultSpan[0] = xSize;
                        resultSpan[1] = ySize;
                    }
                    bestRect.set(currentRect);
                }
            }
        }
        // re-mark space taken by ignoreView as occupied
        // 恢复occupied的值
        markCellsAsOccupiedForView(ignoreView, occupied);

        // Return -1, -1 if no suitable location found
        if (bestDistance == Double.MAX_VALUE) {
            bestXY[0] = -1;
            bestXY[1] = -1;
        }
        recycleTempRects(validRegions);
        return bestXY;
    }
```
在后面拖动时具体执行桌面重排时会调用到findNearestArea方法的重载方法，还会权衡建议的方向向量。

rearrangementExists方法：
参数solution为ItemConfiguration对象，ItemConfiguration保存了当前桌面各个item的情况。首先确定拖动View的Rect，使用Rect.intersects方法，得出soulution的intersectingViews列表，这里是拖动时影响到的item。接下来实现具体的方案，根据注释，首先找出符合推力的方案attemptPushInDirection方法，拖动时，一个被推出当前位置item也会挤开别的item，如果失败，addViewsToTempLocation方法，尝试受影响的iew整块移动，如果还是不想，遍历mIntersectingViews，addViewToTempLocation方法分别尝试放置到新位置。

attemptPushInDirection方法：
若xy方向上均有作用力，先pushViewsToTempLocation()移动x轴，再移动y轴，若正方向上移动失败，换方向，若只有一个方向有作用力，以以下顺序推动，正方向，反方向，xy轴互换，xy轴互换反方向。确定方向后，调用pushViewsToTempLocation方法。

pushViewsToTempLocation方法：
这个方法里初始化了ViewCluster对象,这个类有四个数组(leftEdge，rightEdge，topEdge和bottomEdge)记录了上下左右边上当前影响的Views的边界位置，用来精确的表示一串view的边界，以便于在拖动时操作View的情况。

![ViewCluster](http://img.blog.csdn.net/20160501161039568)

例如上图4*4的Workspace，如果拖动一个item从右靠近时钟widget时，时钟widget就为受影响的View，此时ViewCluster对象只有一个View，它的四个边界数组分别为{1,1,-1,-1},{2,2,-1,-1},{-1,0,0,-1}和{-1,1,1,-1}，初始化时为-1，代表该位置没有受影响的View。

首先根据推力向量的方向计算当前受影响的边界和被推开的view的移动距离pushDistance，ViewCluster的sortConfigurationForEdgePush方法根据当前边界将拖动时受影响的View判断先后受影响的次序排序。推动时，碰到all app按钮，即canReorder为false，推动失败。推动时遍历以排好序的列表，调用isViewTouchingEdge方法判断当前View是否接触到当前受影响View集，true则加进来，一次推动一个单位距离，一直推到目标位置，拖动时影响的view就这样一层层扩大“感染”。拖动结束后，判断结果是否超过边界，不合格则restore返回前状态。

addViewsToTempLocation方法
boundingRect为所有相关view所占的rect使用union方法联合的Rect，再用一个boolean二维数组表示其精确的占据情况，再调用上面所说的findNearestArea重载方法求出落下区域。addViewToTempLocation方法类似。

其他DropTarget对象和Workspace大同小异，根据不同的情景功能可能细节上会有不同。

###数学基础
- 数量积 isFlingingToDelete方法，findNearestArea方法，这里用于判断方向是否符合。
维基百科：https://zh.wikipedia.org/wiki/%E6%95%B0%E9%87%8F%E7%A7%AF
- 三角函数 computeDirectionVector方法，这里计算具体的推动方向


