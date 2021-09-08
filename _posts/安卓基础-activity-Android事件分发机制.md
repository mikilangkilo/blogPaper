---
title: 安卓基础-Android事件分发机制
date: 2018-01-03 23:10:07
tag : "activity"
category : "安卓基础"
description : "本文主要是对Android渠道包打包取舍以及打包流程分析"
---

假如一个button
```
	button.setOnClickListener(new OnClickListener(){
		@Override
		public void onClick(View v){
			Log.d("TAG","onClick execute");
		}
	})
```

同时设置ontouch事件

```
	button.setOnTouchListener(new OnTouchListener(){
		@Override
		public boolean onTouch(View v, MotionEvent event){
			Log.d("TAG","onTouch execute, action "+ event.getAction());
			return false;
		}
	})
```

点击按钮log如下
```
onTouch execute, action 0
onTouch execute, action 1
onClick execute
```

onTouch是优先于onClick执行的，并且onTouch执行了2次，一次是ACTION_DOWN,一次是ACTION_UP。
顺序是先经过onTouch，在传递到onClick。

将ontouch返回值改为true，在运行一次，会发现onClick不再执行，这就是拦截了。

# dispatchTouchEvent

任何view被触摸，一定会执行dispatchTouchEvent这个方法。

```
public boolean dispatchTouchEvent(MotionEvent event) {  
    if (mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED &&  
            mOnTouchListener.onTouch(this, event)) {  
        return true;  
    }  
    return onTouchEvent(event);  
}  
```

这段代码很明显的指出了，首先会执行的是set的onTouchListener的onTouch事件，之后才会执行自身的onTouchEvent事件。

```
public boolean onTouchEvent(MotionEvent event) {  
    final int viewFlags = mViewFlags;  
    if ((viewFlags & ENABLED_MASK) == DISABLED) {  
        // A disabled view that is clickable still consumes the touch  
        // events, it just doesn't respond to them.  
        return (((viewFlags & CLICKABLE) == CLICKABLE ||  
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));  
    }  
    if (mTouchDelegate != null) {  
        if (mTouchDelegate.onTouchEvent(event)) {  
            return true;  
        }  
    }  
    if (((viewFlags & CLICKABLE) == CLICKABLE ||  
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {  
        switch (event.getAction()) {  
            case MotionEvent.ACTION_UP:  
                boolean prepressed = (mPrivateFlags & PREPRESSED) != 0;  
                if ((mPrivateFlags & PRESSED) != 0 || prepressed) {  
                    // take focus if we don't have it already and we should in  
                    // touch mode.  
                    boolean focusTaken = false;  
                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {  
                        focusTaken = requestFocus();  
                    }  
                    if (!mHasPerformedLongPress) {  
                        // This is a tap, so remove the longpress check  
                        removeLongPressCallback();  
                        // Only perform take click actions if we were in the pressed state  
                        if (!focusTaken) {  
                            // Use a Runnable and post this rather than calling  
                            // performClick directly. This lets other visual state  
                            // of the view update before click actions start.  
                            if (mPerformClick == null) {  
                                mPerformClick = new PerformClick();  
                            }  
                            if (!post(mPerformClick)) {  
                                performClick();  
                            }  
                        }  
                    }  
                    if (mUnsetPressedState == null) {  
                        mUnsetPressedState = new UnsetPressedState();  
                    }  
                    if (prepressed) {  
                        mPrivateFlags |= PRESSED;  
                        refreshDrawableState();  
                        postDelayed(mUnsetPressedState,  
                                ViewConfiguration.getPressedStateDuration());  
                    } else if (!post(mUnsetPressedState)) {  
                        // If the post failed, unpress right now  
                        mUnsetPressedState.run();  
                    }  
                    removeTapCallback();  
                }  
                break;  
            case MotionEvent.ACTION_DOWN:  
                if (mPendingCheckForTap == null) {  
                    mPendingCheckForTap = new CheckForTap();  
                }  
                mPrivateFlags |= PREPRESSED;  
                mHasPerformedLongPress = false;  
                postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());  
                break;  
            case MotionEvent.ACTION_CANCEL:  
                mPrivateFlags &= ~PRESSED;  
                refreshDrawableState();  
                removeTapCallback();  
                break;  
            case MotionEvent.ACTION_MOVE:  
                final int x = (int) event.getX();  
                final int y = (int) event.getY();  
                // Be lenient about moving outside of buttons  
                int slop = mTouchSlop;  
                if ((x < 0 - slop) || (x >= getWidth() + slop) ||  
                        (y < 0 - slop) || (y >= getHeight() + slop)) {  
                    // Outside button  
                    removeTapCallback();  
                    if ((mPrivateFlags & PRESSED) != 0) {  
                        // Remove any future long press/tap checks  
                        removeLongPressCallback();  
                        // Need to switch from pressed to not pressed  
                        mPrivateFlags &= ~PRESSED;  
                        refreshDrawableState();  
                    }  
                }  
                break;  
        }  
        return true;  
    }  
    return false;  
}  
```

当事件为MotionEvent.ACTION_UP时，如果该控件可点击，会执行performClick()

```
public boolean performClick() {  
    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);  
    if (mOnClickListener != null) {  
        playSoundEffect(SoundEffectConstants.CLICK);  
        mOnClickListener.onClick(this);  
        return true;  
    }  
    return false;  
}
```

从源码可以看出，基本上每一次事件，都会return 一个true，也就是消耗掉这次事件了。而之后的事件会重新进入，这也和之前的截断事件机制符合。

# onTouch和onTouchEvent有什么区别，又该如何使用？

从源码中可以看出，这两个方法都是在View的dispatchTouchEvent中调用的，onTouch优先于onTouchEvent执行。如果在onTouch方法中通过返回true将事件消费掉，onTouchEvent将不会再执行。
另外需要注意的是，onTouch能够得到执行需要两个前提条件，第一mOnTouchListener的值不能为空，第二当前点击的控件必须是enable的。因此如果你有一个控件是非enable的，那么给它注册onTouch事件将永远得不到执行。对于这一类控件，如果我们想要监听它的touch事件，就必须通过在该控件中重写onTouchEvent方法来实现。

# 为什么给ListView引入了一个滑动菜单的功能，ListView就不能滚动了？
比如说左右滑的时候，还是正常的左右滑。
但是上下滑的时候，会出现一个问题，就是listview无法上下滑动，这主要是事件被scrollview给消耗掉了，需要在触摸的时候，scrollview不进行截断操作，让事件能够传递下去。