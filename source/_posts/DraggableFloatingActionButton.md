---
title: 可拖拽的FloatingActionButton
date: 2018-08-22 18:52:48
header-img: "draggablefloatingactionbutton_cover.jpg"
author:     "spirytusz"
tags: 
- Android
---

## 前言
FloatingActionButton是Google力推的一个Material Design控件。最近做的一个项目，需要实现一个可以左右滑动，松手回到原位的FloatingActionButton。研究了大半天就写出来了，于是写写这片博客记录一下。

<!--more-->

## MotionEvent的事件
当我们点击一个view，或者拖动一个view到处滑动时，都会产生一个MotionEvent事件。
* 当我们的手指刚刚点击到屏幕时，会产生MotionEvent.ACTION_DOWN事件；
* 当我们的手指在屏幕上到处滑动时，会产生MotionEvent.ACTION_MOVE事件；
* 当我们的手指离开屏幕时，会产生MotionEvent.ACTION_UP事件。

那么这就好办了，当产生了MotionEvent.ACTION_MOVE事件时，只需要获得屏幕触点的坐标，把fab移动过去就行了

## 设置OnTouchListener
前面已经说明了，view会产生MotionEvent.ACTION_DOWN，MotionEvent.ACTION_MOVE 和 MotionEvent.ACTION_UP事件。问题是如何捕获这些事件。当我们的手指开始接触屏幕，直到离开屏幕前，View.onTouch()方法都会被调用，而且这个方法的参数中有MotionEvent参数，可以获取事件。因此我们需要实现该方法，并为fab设置OnTouchListener监听器。

{% codeblock lang:Java %}
mFab.setOnTouchListener(this);

@Override
public boolean onTouch(View view, MotionEvent motionEvent) {
	... ...
}
{% endcodeblock %}

接下来就可以通过motionEvent来获得事件，并根据事件来响应。实现的思路已经很明显了，接下来分事件来说明。

### MotionEvent.ACTION_DOWN
当手指开始接触屏幕时，会产生MotionEvent.ACTION_DOWN事件。
这时要做一件事，记录下『mFab初始坐标X值和触点坐标X值之差』，以供备用。
其中initRawX为mFab的初始坐标的X值。

{% codeblock lang:Java %}
case MotionEvent.ACTION_DOWN:
    dX = initRawX - motionEvent.getRawX();
    return true;
{% endcodeblock %}

### MotionEvent.ACTION_MOVE
当手指开始在屏幕上滑动时，会产生MotionEvent.ACTION_MOVE事件。
这时需要实现mFab跟随手指移动的功能。
要实现mFab移动的功能，我们必须计算出手指移动的距离deltaX，如下：

{% codeblock lang:Java %}
deltaX = (motionEvent.getRawX() - initRawX + dX) * damping;
{% endcodeblock %}

其中damping是阻尼，0 < damping <= 1.

计算出手指移动的距离deltaX后：
* 当移动的距离deltaX满足 -CLICK_DRAG_TOLERANCE < deltaX < CLICK_DRAG_TOLERANCE时，mFab不用移动，因为我们单击mFab时，是不需要mFab移动的。
* 当移动的距离deltaX满足 deltaX > border 或 deltaX < -border 时，mFab则停留在border或-border的位置就可以了。

所以便有：
{% codeblock lang:Java %}
if (deltaX < -border) {
    deltaX = -border;
} else if (deltaX < -CLICK_DRAG_TOLERANCE) {
    setImageResource(R.drawable.ic_skip_previous_white_48dp);
} else if (deltaX > CLICK_DRAG_TOLERANCE && deltaX <= border) {
    setImageResource(R.drawable.ic_skip_next_white_48dp);
} else if (deltaX > border) {
    deltaX = border;
}
if (Math.abs(deltaX) >= CLICK_DRAG_TOLERANCE) {
    // 此时deltaX已经大于mFab移动的阈值CLICK_DRAG_TOLERANCE，mFab移动deltaX个单位。
    view.animate()
        .x(initRawX + deltaX)
        .setDuration(RESPONSE_ACTION_MOVE_DELAY)
        // RESPONSE_ACTION_MOVE_DELAY == 0，立即移动
        .start();
}
return true;
{% endcodeblock %}
其中CLICK_DRAG_TOLERANCE是mFab能移动距离的阈值。

### MotionEvent.ACTION_UP
当手指离开屏幕时，会产生MotionEvent.ACTION_UP事件。
这个时候就可以处理单击、左滑和右滑的事件。
{% codeblock lang:Java %}
if (onDraggableFABEventListener != null) {
    if (Math.abs(deltaX) < CLICK_DRAG_TOLERANCE) {
        onDraggableFABEventListener.onClick();
    } else {
        if (deltaX == border) {
            onDraggableFABEventListener.onDraggedRight();
        } else if (deltaX == -border) {
            onDraggableFABEventListener.onDraggedLeft();
        }
    }
}
{% endcodeblock %}

如果deltaX的绝对值小于CLICK_DRAG_TOLERANCE，则表明是单击事件；
否则如果deltaX == border，则表明是右滑事件；
否则如果deltaX == -border，则表明是左滑事件。

处理完事件后，mFab要回到原位，因此：
{% codeblock lang:Java %}
view.animate()
        .x(initRawX)
        .setDuration(RESET_ANIMATOR_DURATION)
        // RESET_ANIMATOR_DURATION > 0
        .start();
return true;
{% endcodeblock %}

至此，整个onTouch方法就完成了！以下是完整代码:
{% codeblock lang:Java %}
@Override
public boolean onTouch(View view, MotionEvent motionEvent) {
    int action = motionEvent.getAction();
    switch (action) {
        case MotionEvent.ACTION_DOWN:
            dX = initRawX - motionEvent.getRawX();
            return true;
        case MotionEvent.ACTION_MOVE:
            deltaX = (motionEvent.getRawX() - initRawX + dX) * damping;
            if (deltaX < -border) {
                deltaX = -border;
            } else if (deltaX < -CLICK_DRAG_TOLERANCE) {
                setImageResource(R.drawable.ic_skip_previous_white_48dp);
            } else if (deltaX > CLICK_DRAG_TOLERANCE && deltaX <= border) {
                setImageResource(R.drawable.ic_skip_next_white_48dp);
            } else if (deltaX > border) {
                deltaX = border;
            }
            if (Math.abs(deltaX) >= CLICK_DRAG_TOLERANCE) {
                view.animate()
                        .x(initRawX + deltaX)
                        .setDuration(RESPONSE_ACTION_MOVE_DELAY)
                        .start();
            }
            return true;
        case MotionEvent.ACTION_UP:
            int resId = MediaPlayController.getInstance().isPlaying() ? R.drawable.ic_pause_white_48dp : R.drawable.ic_play_arrow_white_48dp;
            setImageResource(resId);
            if (onDraggableFABEventListener != null) {
                if (Math.abs(deltaX) < CLICK_DRAG_TOLERANCE) {
                    onDraggableFABEventListener.onClick();
                } else {
                    if (deltaX == border) {
                        onDraggableFABEventListener.onDraggedRight();
                    } else if (deltaX == -border) {
                        onDraggableFABEventListener.onDraggedLeft();
                    }
                }
            }
            view.animate()
                    .x(initRawX)
                    .setDuration(RESET_ANIMATOR_DURATION)
                    .start();
            return true;
    }
    return super.onTouchEvent(motionEvent);
}
{% endcodeblock %}


## mFab的初始位置
前面的onTouch方法需要initRawX的值，我们如何获得这个值？
`this.getX()`是不可行的，因为当执行`this.getX()`的时候，不能保证mFab已经被绘制出来了；而如果mFab没有绘制出来，`this.getX()`将会返回0。因此我们需要在适当的实际调用`this.getX()`来获取initRawX。
onWindowFocusChanged()方法是在Activity的onResume()后被调用，当Activity的onResume()方法被调用后，Activity是可见，可与用户交互的，说明view都已经绘制完毕，所以我们可以在onWindowFocusChanged()方法中获取mFab的初始坐标。

{% codeblock lang:Java %}
@Override
public void onWindowFocusChanged(boolean hasWindowFocus) {
    super.onWindowFocusChanged(hasWindowFocus);
    // get mFab initial location X
    initRawX = getX();
}
{% endcodeblock %}

## 总结
其实可拖拽的FloatingActionButton的实现原理很简单，只需要为mFab设置OnTouchListener，在监听器中捕获MotionEvent.ACTION_DOWN、MotionEvent.ACTION_MOVE、MotionEvent.ACTION_UP 事件，再分别处理事件即可。完整代码可以看[这里](https://github.com/zkw012300/DraggableFloatingActionButton/blob/master/mylibrary/src/main/java/com/zspirytus/mylibrary/DraggableFloatingActionButton.java)。

## 感想
之前看Android中的View事件的分发机制，第一次接触到了ACTION_DOWN、ACTION_MOVE、ACTION_UP事件，看了一遍云里雾里的，但是多看几遍后就开始理解。这次的可拖拽的FloatingActionButton是我在写的项目中的一个小控件，有了之前的基础，写起来不太吃力。所以我认为这三个事件还是蛮重要的。
