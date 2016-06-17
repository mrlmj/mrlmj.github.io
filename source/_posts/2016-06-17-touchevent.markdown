---
layout: post
title: Android事件分发机制
date: 2016-06-17 11:31:09 +0800
comments: true
categories:
---

之前看过几遍Android的事件分发流程，每次都以为看的差不多了，可过了一段时间印象又淡了，说到底还是知识掌握的不扎实，不够系统，只了解了皮毛。所以，本文对Android的事件分发机制进行系统总结，避免遗忘。结合android源码，参考《Android开发艺术探索》一书和相关博文。

<!-- more -->

### 基础内容

在Android的事件分发过程中涉及到View的三个重要方法:

**public boolean dispatchTouchEvent(MotionEvent ev)**

进行事件的分发。如果是ViewGroup会在其中将事件分发给其子View进行事件处理 或者 在自己的onTouchEvent中处理（当拦截或者子View都不消费事件时），如果是View，则在其中调用onTouchEvent进行事件的处理,不会分发。

返回值表示是否消费了事件。受当前View和其子View的影响。

**public boolean onInterceptTouchEvent(MotionEvent ev)**

进行事件的拦截。 返回true表示拦截，如果拦截了某个事件，则对于整个事件序列的其他序列，都不再调用此方法进行拦截。

返回值表示是否拦截事件。

**public boolean onTouchEvent(MotionEvent ev)**

进行事件的处理，返回值表示是否消费事件，如果不消费，则事件序列中的其他事件不会再传递给它。

---

整个事件分发的过程可以用以下代码来表示：

```java

public boolean dispatchTouchEvent(MotionEvent ev){
  boolean consume = false;
  if(onInterceptTouchEvent(ev)){
      consume = onTouchEvent(ev);
  }else{
      consume = child.dispatchTouchEvent(ev);
  }
  return consume;
}

```

上述的过程是整个事件分发流程的一环，依赖子View、返回结果也影响父View。这里也只是一个简单的过程，在实际的过程中还有很多需要注意的点，后面会再细化其中的某些细节。

### 结合源码进行分析

1、事件分发的入口从Activity#dispatchTouchEvent开始：

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```
方法总调用了**getWindow().superDispatchTouchEvent(ev)** .如果返回true(消费了事件)则直接return true执行完毕.
否则调用自己的**onTouchEvent**进行处理（如果事件没有消费，最终交给Activity处理)。

2、getWindow()的实现为PhoneWindow类：

```java
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```
此处调用mDecor中的方法。

mDecor是Window中的根View，本身是一个FrameLayout，根据应用的Theme会有不同的布局，我们在**Activity#onCreate**中setContentView设置的布局，其实就是放在它的一个子View中(id为android.R.id.content)。

![](/images/articles/layout.png)

3、继续跟下去，到达ViewGroup中:

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
       boolean handled = false;
       if (onFilterTouchEventForSecurity(ev)) {
           final int action = ev.getAction();
           final int actionMasked = action & MotionEvent.ACTION_MASK;

           // Handle an initial down.
           if (actionMasked == MotionEvent.ACTION_DOWN) {
               // Throw away all previous state when starting a new touch gesture.
               // The framework may have dropped the up or cancel event for the previous gesture
               // due to an app switch, ANR, or some other state change.
               cancelAndClearTouchTargets(ev);
               resetTouchState();
           }

           // Check for interception.
           final boolean intercepted;
           if (actionMasked == MotionEvent.ACTION_DOWN
                   || mFirstTouchTarget != null) {
               final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
               if (!disallowIntercept) {
                   intercepted = onInterceptTouchEvent(ev);
                   ev.setAction(action); // restore action in case it was changed
               } else {
                   intercepted = false;
               }
           } else {
               // There are no touch targets and this action is not an initial down
               // so this view group continues to intercept touches.
               intercepted = true;
           }

           // Check for cancelation.
           final boolean canceled = resetCancelNextUpFlag(this)
                   || actionMasked == MotionEvent.ACTION_CANCEL;

           // Update list of touch targets for pointer down, if needed.
           final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
           TouchTarget newTouchTarget = null;
           boolean alreadyDispatchedToNewTouchTarget = false;
           if (!canceled && !intercepted) {
               if (actionMasked == MotionEvent.ACTION_DOWN
                       || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                       || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {

                   final int childrenCount = mChildrenCount;
                   if (newTouchTarget == null && childrenCount != 0) {
                       final float x = ev.getX(actionIndex);
                       final float y = ev.getY(actionIndex);
                       // Find a child that can receive the event.
                       // Scan children from front to back.
                       final View[] children = mChildren;

                       final boolean customOrder = isChildrenDrawingOrderEnabled();
                       for (int i = childrenCount - 1; i >= 0; i--) {
                           final int childIndex = customOrder ?
                                   getChildDrawingOrder(childrenCount, i) : i;
                           final View child = children[childIndex];
                           if (!canViewReceivePointerEvents(child)
                                   || !isTransformedTouchPointInView(x, y, child, null)) {
                               continue;
                           }

                           newTouchTarget = getTouchTarget(child);
                           if (newTouchTarget != null) {
                               // Child is already receiving touch within its bounds.
                               // Give it the new pointer in addition to the ones it is handling.
                               newTouchTarget.pointerIdBits |= idBitsToAssign;
                               break;
                           }

                           resetCancelNextUpFlag(child);
                           if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                               // Child wants to receive touch within its bounds.
                               mLastTouchDownTime = ev.getDownTime();
                               mLastTouchDownIndex = childIndex;
                               mLastTouchDownX = ev.getX();
                               mLastTouchDownY = ev.getY();
                               newTouchTarget = addTouchTarget(child, idBitsToAssign);
                               alreadyDispatchedToNewTouchTarget = true;
                               break;
                           }
                       }
                   }

                   if (newTouchTarget == null && mFirstTouchTarget != null) {
                       // Did not find a child to receive the event.
                       // Assign the pointer to the least recently added target.
                       newTouchTarget = mFirstTouchTarget;
                       while (newTouchTarget.next != null) {
                           newTouchTarget = newTouchTarget.next;
                       }
                       newTouchTarget.pointerIdBits |= idBitsToAssign;
                   }
               }
           }

           // Dispatch to touch targets.
           if (mFirstTouchTarget == null) {
               // No touch targets so treat this as an ordinary view.
               handled = dispatchTransformedTouchEvent(ev, canceled, null,
                       TouchTarget.ALL_POINTER_IDS);
           } else {
               // Dispatch to touch targets, excluding the new touch target if we already
               // dispatched to it.  Cancel touch targets if necessary.
               TouchTarget predecessor = null;
               TouchTarget target = mFirstTouchTarget;
               while (target != null) {
                   final TouchTarget next = target.next;
                   if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                       handled = true;
                   } else {
                       final boolean cancelChild = resetCancelNextUpFlag(target.child)
                               || intercepted;
                       if (dispatchTransformedTouchEvent(ev, cancelChild,
                               target.child, target.pointerIdBits)) {
                           handled = true;
                       }
                       if (cancelChild) {
                           if (predecessor == null) {
                               mFirstTouchTarget = next;
                           } else {
                               predecessor.next = next;
                           }
                           target.recycle();
                           target = next;
                           continue;
                       }
                   }
                   predecessor = target;
                   target = next;
               }
           }

           // Update list of touch targets for pointer up or cancel, if needed.
           if (canceled
                   || actionMasked == MotionEvent.ACTION_UP
                   || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
               resetTouchState();
           } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
               final int actionIndex = ev.getActionIndex();
               final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
               removePointersFromTouchTargets(idBitsToRemove);
           }
       }

       if (!handled && mInputEventConsistencyVerifier != null) {
           mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
       }
       return handled;
   }
```

ViewGroup中dispatch方法内容比较多，这里删除了一部分无关的，所有的点基本都可以从这个方法中找到。

在ViewGroup#dispatchTouchEvent中第18行有如下判断,简化：
```java
if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
    intercept = onInterceptTouchEvent(ev);
}else{
    intercept = true;
}
```
可以看出在DOWN的时候，会进入if方法体，调用三大方法之一 **onInterceptTouchEvent** 此方法默认返回false,也就是不拦截事件，intercept = false;

mFirstTouchTarget 非常重要，之后的所有判断几乎都跟这个变量有联系。顾名思义，这个变量代表的是ViewGroup的事件分发目标，是ViewGroup的一个Child。有三种情况会导致mFirstTouchTarget=null（也就是没有事件分发的目标了），一是onInterceptTouchEvent返回了true拦截了DOWN事件；二是此ViewGroup没有子View；三是所有的子View都不消费事件。二和三的处理在41~84行，主要是遍历了能接收到点击事件的子View然后将事件分发过去，在子View分发事件返回了true消费了事件，才将mFirstTouchTarget指向它，其他情况则为null。三种情况下都会导致mFirstTouchTarget不能够被赋值。

如果满足上述三种情况导致mFirstTouchTarget=null，则对于事件序列中DOWN之后的事件，都会进入else体，intercept=true,拦截事件自己处理。在98~102行自己进行处理。

```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
           View child, int desiredPointerIdBits) {
       final boolean handled;

       // Canceling motions is a special case.  We don't need to perform any transformations
       // or filtering.  The important part is the action, not the contents.
       final int oldAction = event.getAction();
       if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
           event.setAction(MotionEvent.ACTION_CANCEL);
           if (child == null) {
               handled = super.dispatchTouchEvent(event);
           } else {
               handled = child.dispatchTouchEvent(event);
           }
           event.setAction(oldAction);
           return handled;
       }
}
```
可以看出如果child==null，则会调用父类的dispatchTouchEvent()方法进行处理，ViewGroup的父类也就是View,从此，分发事件的逻辑进入View中，后面再讲。

这里可以得出结论：

#### 一：如果一个View决定拦截，那么整个事件序列只能有它来处理，并且它的onInterceptTouchEvent方法不会再调用。
#### 二：如果一个View不消耗传递给它的ACTION_DOWN事件，则之后的所有事件都不会再交给它处理，事件会将重新交给父元素去处理。

---

![](/images/articles/touchevent.png)

由此，如果此时在上图的View处点击了一下，则事件的分发过程为:**Activity#dispatchTouchEvent -> ViewGroup#onInterceptTouchEvent -> ViewGroup#dispatchTouchEvent -> ... -> View#dispatchTouchEvent  -> View#onTouchEvent**

考虑一下特殊情况：

情况① 如果在ViewGroup的onInterceptTouchEvent中对MOVE事件return true进行拦截，会有什么情况?

按道理来讲，如果ViewGroup中对MOVE事件进行了拦截，那么事件就不应该分发给View了，做一下实验:

这里写了两个自定义的View

```java
public class MyViewGroup extends LinearLayout{

	public MyViewGroup(Context context) {
		super(context);
	}

	public MyViewGroup(Context context, AttributeSet attrs) {
		super(context, attrs);
	}

	public MyViewGroup(Context context, AttributeSet attrs, int defStyleAttr) {
		super(context, attrs, defStyleAttr);
	}

	@Override
	public boolean onInterceptTouchEvent(MotionEvent ev) {
		if(ev.getAction() == MotionEvent.ACTION_MOVE){
			return true;
		}
		return super.onInterceptTouchEvent(ev);
	}

	@Override
	public boolean onTouchEvent(MotionEvent event) {
		Log.v("seewo","ViewGroup#onTouchEvent:"+event.getAction());
		return super.onTouchEvent(event);
	}
}
```

```java
public class MyView extends Button{
	public MyView(Context context) {
		super(context);
	}

	public MyView(Context context, AttributeSet attrs) {
		super(context, attrs);
	}

	public MyView(Context context, AttributeSet attrs, int defStyleAttr) {
		super(context, attrs, defStyleAttr);
	}

	@Override
	public boolean onTouchEvent(MotionEvent event) {
		Log.v("monkey","onTouchEvent:"+event.getAction());
		return super.onTouchEvent(event);
	}
}
```

```java
public class MainActivity extends AppCompatActivity {

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
	}

	@Override
	public boolean dispatchTouchEvent(MotionEvent ev) {
		return super.dispatchTouchEvent(ev);
	}

	@Override
	public boolean onTouchEvent(MotionEvent event) {
		Log.v("monkey","Activity#onTouchEvent:"+event.getAction());
		return super.onTouchEvent(event);
	}
}
```

布局如前图所示：

```xml
<?xml version="1.0" encoding="utf-8"?>
<com.monkeyliu.test1.MyViewGroup
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <com.monkeyliu.test1.MyView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="button" />

</com.monkeyliu.test1.MyViewGroup>
```

在ViewGroup中对于MOVE事件进行了拦截，然后打了部分Log。

此时，如果在MyView上点击一下，Log为：

```java
06-17 10:58:53.954 23314-23314/com.monkeyliu.test1 V/monkey: onTouchEvent:0
06-17 10:58:54.065 23314-23314/com.monkeyliu.test1 V/monkey: onTouchEvent:1
```
View处理了一个DOWN和一个UP。

如果在MyView上DOWN-MOVE-MOVE-...-MOVE-UP，Log为：

```java
06-17 11:06:06.081 23314-23314/com.monkeyliu.test1 V/monkey: onTouchEvent:0
06-17 11:06:06.185 23314-23314/com.monkeyliu.test1 V/monkey: onTouchEvent:3
06-17 11:06:06.200 23314-23314/com.monkeyliu.test1 V/monkey: ViewGroup#onTouchEvent:2
06-17 11:06:06.200 23314-23314/com.monkeyliu.test1 V/monkey: Activity#onTouchEvent:2
06-17 11:06:06.291 23314-23314/com.monkeyliu.test1 V/monkey: ViewGroup#onTouchEvent:2
06-17 11:06:06.291 23314-23314/com.monkeyliu.test1 V/monkey: Activity#onTouchEvent:2
06-17 11:06:06.316 23314-23314/com.monkeyliu.test1 V/monkey: ViewGroup#onTouchEvent:2
06-17 11:06:06.316 23314-23314/com.monkeyliu.test1 V/monkey: Activity#onTouchEvent:2
```
View处理了一个DOWN和一个CANCEL，之后的事件向外传递给父类处理。可以看出，在拦截MOVE事件之后，向View分发了一个CANCEL事件，之后事件便不再分发给View.
这个处理的过程在98~134行,如下。

```java
if (mFirstTouchTarget == null) {
		// No touch targets so treat this as an ordinary view.
		handled = dispatchTransformedTouchEvent(ev, canceled, null,
				TouchTarget.ALL_POINTER_IDS);
	} else {
		// Dispatch to touch targets, excluding the new touch target if we already
		// dispatched to it.  Cancel touch targets if necessary.
		TouchTarget predecessor = null;
		TouchTarget target = mFirstTouchTarget;
		while (target != null) {
			final TouchTarget next = target.next;
			if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
				handled = true;
			} else {
				final boolean cancelChild = resetCancelNextUpFlag(target.child)
						|| intercepted;
				if (dispatchTransformedTouchEvent(ev, cancelChild,
						target.child, target.pointerIdBits)) {
					handled = true;
				}
				if (cancelChild) {
					if (predecessor == null) {
						mFirstTouchTarget = next;
					} else {
						predecessor.next = next;
					}
					target.recycle();
					target = next;
					continue;
				}
			}
			predecessor = target;
			target = next;
		}
	}
```

在else遍历了target(target其实是一个链表结构，表示了一系列可以接受事件的view,用于处理多点触摸的，这里对于单点只考虑一层循环)。其中if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) 是对于之前的DOWN事件进行处理，直接设置为handled=true;
重点是15~13行，因为我们之前拦截了MOVE事件，所以15行的cancelChild变量为true,而在dispatchTransformedTouchEvent中，此变量导致一个CANCEL事件的产生，之后21行，会将mFirstTouchTarget置为null.之后接受到的事件便由自己处理，View再接收不到事件序列之后的任何事件，事件给父View处理，证实了结论一。

情况②：如果在View的onTouchEvent中对于MOVE事件，return false不进行消费会有什么结果?

修改MyView.java

```java
@Override
public boolean onTouchEvent(MotionEvent event) {
  Log.v("monkey","onTouchEvent:"+event.getAction());
  if(event.getAction() == MotionEvent.ACTION_MOVE){
    return false;
  }
  return super.onTouchEvent(event);
}
```
修改MyViewGroup.java，不拦截事件
```java
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
  return super.onInterceptTouchEvent(ev);
}
```

在MyView上DOWN-MOVE-MOVE-...-MOVE-UP，Log为：

```java
06-17 11:28:09.265 13787-13787/com.monkeyliu.test1 V/monkey: onTouchEvent:0
06-17 11:28:09.399 13787-13787/com.monkeyliu.test1 V/monkey: onTouchEvent:2
06-17 11:28:09.399 13787-13787/com.monkeyliu.test1 V/monkey: Activity#onTouchEvent:2
06-17 11:28:09.418 13787-13787/com.monkeyliu.test1 V/monkey: onTouchEvent:2
06-17 11:28:09.418 13787-13787/com.monkeyliu.test1 V/monkey: Activity#onTouchEvent:2
06-17 11:28:09.449 13787-13787/com.monkeyliu.test1 V/monkey: onTouchEvent:2
06-17 11:28:09.449 13787-13787/com.monkeyliu.test1 V/monkey: Activity#onTouchEvent:2
06-17 11:28:09.467 13787-13787/com.monkeyliu.test1 V/monkey: onTouchEvent:2
06-17 11:28:09.468 13787-13787/com.monkeyliu.test1 V/monkey: Activity#onTouchEvent:2
06-17 11:28:09.699 13787-13787/com.monkeyliu.test1 V/monkey: onTouchEvent:2
06-17 11:28:09.699 13787-13787/com.monkeyliu.test1 V/monkey: Activity#onTouchEvent:2
06-17 11:28:09.715 13787-13787/com.monkeyliu.test1 V/monkey: onTouchEvent:2
06-17 11:28:09.716 13787-13787/com.monkeyliu.test1 V/monkey: Activity#onTouchEvent:2
06-17 11:28:09.882 13787-13787/com.monkeyliu.test1 V/monkey: onTouchEvent:2
06-17 11:28:09.883 13787-13787/com.monkeyliu.test1 V/monkey: Activity#onTouchEvent:2
06-17 11:28:10.083 13787-13787/com.monkeyliu.test1 V/monkey: onTouchEvent:2
06-17 11:28:10.083 13787-13787/com.monkeyliu.test1 V/monkey: Activity#onTouchEvent:2
06-17 11:28:10.400 13787-13787/com.monkeyliu.test1 V/monkey: onTouchEvent:1
```
很奇妙，View仍然接收到了每个事件，但是对于没有处理的MOVE事件，最后都交给了Activity处理，没有经过ViewGroup。

确实，从ViewGroup#dispatchTouchEvent中，在没有消费MOVE事件的情况下，没有找到将mFirstTouchTarget置为null,然后自己处理MOVE事件的代码。至于，最后将事件交给了Activity处理，我猜是在147~149行，对于!handled情况进行了处理。

对此可以得出一个结论：
#### 三：如果View不消耗除ACTION_DOWN之外的事件，仍然会收到事件序列的其他事件，父类的onTouchEvent不会调用，消失的事件会传给Activity处理（由此可见ACTION_DOWN事件才是主角，return true则一直接收事件，return false则不会再接收到后续事件）。

4、上述对事件分发进行了一个大概的梳理，最终事件会通过 View#dispatchTouchEvent -> View#onTouchEvent进行处理。

View类没有onInterceptTouchEvent方法。

View中事件的处理比较清晰简单，涉及到OnTouchListener和OnClickListener和onTouchEvent的执行顺序问题。
```java
public boolean dispatchTouchEvent(MotionEvent event) {
        if (onFilterTouchEventForSecurity(event)) {
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }

```

上述View#dispatchTouchEvent源码可以看出，会优先调用onTouchListener#onTouch方法，如果返回true，表示消耗了事件，不再进入onTouchEvent处理，否则会进入onTouchEvent逻辑中。

```java
public boolean onTouchEvent(MotionEvent event) {
       final float x = event.getX();
       final float y = event.getY();
       final int viewFlags = mViewFlags;
       final int action = event.getAction();

       if ((viewFlags & ENABLED_MASK) == DISABLED) {
           if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
               setPressed(false);
           }
           // A disabled view that is clickable still consumes the touch
           // events, it just doesn't respond to them.
           return (((viewFlags & CLICKABLE) == CLICKABLE
                   || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                   || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
       }

       if (mTouchDelegate != null) {
           if (mTouchDelegate.onTouchEvent(event)) {
               return true;
           }
       }

       if (((viewFlags & CLICKABLE) == CLICKABLE ||
               (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
               (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
           switch (action) {
               case MotionEvent.ACTION_UP:
                   boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                   if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                       // take focus if we don't have it already and we should in
                       // touch mode.
                       boolean focusTaken = false;
                       if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                           focusTaken = requestFocus();
                       }

                       if (prepressed) {
                           // The button is being released before we actually
                           // showed it as pressed.  Make it show the pressed
                           // state now (before scheduling the click) to ensure
                           // the user sees it.
                           setPressed(true, x, y);
                      }

                       if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
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
                           postDelayed(mUnsetPressedState,
                                   ViewConfiguration.getPressedStateDuration());
                       } else if (!post(mUnsetPressedState)) {
                           // If the post failed, unpress right now
                           mUnsetPressedState.run();
                       }

                       removeTapCallback();
                   }
                   mIgnoreNextUpEvent = false;
                   break;
```

上述为View#onTouchEvent源码，在ACTION_UP时，performClick调用了OnClickListener#onClick方法，由此可得出结论：

####四、事件的处理顺序为:onTouch -> onTouchEvent -> onClick
####五、事件的消费跟View的状态无关，即使View状态为DISABLED,只要满足CLICKABLE、LONG_CLICKABLE、CONTEXT_CLICKABLE中的一个就可以消费事件。


##总结

对于整个事件分发流程，抛开让人头脑混乱的代码不看，总结为：

事件分发是一个从外向内的过程。

如果一个View拦截了某个事件，则整个事件序列的后续事件都由此View进行处理。

如果一个View不消费ACTION_DOWN事件，则事件序列的后续事件都不会再传递给该View。


参考链接:

[谷哥的小弟博文](http://blog.csdn.net/lfdfhl/article/details/51603088)

[陈育的简书](http://www.jianshu.com/p/8236278676fe)

《Android开发艺术探索》
