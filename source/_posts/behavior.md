---
title: Android Behavior

date: 2017-12-15 10:40:10

tags:
	- Android Behavior
---

# （一）简单使用

## 1、简介
实现CoordinatorLayout中的**直接子View**的相互交互行为，这些交互行为可能包括拖动，滑动，抛掷或任何其他手势！[CoordinatorLayout.Behavior官方文档](https://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html)

## 2、自定义Behavior

**第一步**：必须要继承**CoordinatorLayout.Behavior**，且重写构造方法**public Behavior(Context context, AttributeSet attrs)**

	    public TopicBehavior(Context context, AttributeSet set) {
	        super(context, set);
	    }
另外，我们也可以自定义参数，通过AttributeSet获取已定义参数，并在以后的工作中使用！

**第二步**：重写相应的方法

**依赖处理**

	layoutDependsOn(CoordinatorLayout parent, V child, View dependency)

	onDependentViewChanged(CoordinatorLayout parent, V child, View dependency)

layoutDependsOn：确定提供的子视图是否具有另一个特定的兄弟视图作为布局依赖关系，[链接](https://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html#layoutDependsOn(android.support.design.widget.CoordinatorLayout, V, android.view.View))

onDependentViewChanged：回应一个孩子的依赖观点的变化，只要依赖视图在标准布局流程外改变大小或位置，就会调用此方法，[链接](https://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html#onDependentViewChanged(android.support.design.widget.CoordinatorLayout, V, android.view.View))

参数解析：

第一个参数，CoordinatorLayout，作为父布局；

第二个参数，V，目标View；

第三个参数，View，目标View所依赖的视图View

**滑动处理**

	onStartNestedScroll(CoordinatorLayout coordinatorLayout, V child, View directTargetChild, View target, int axes, int type)
	
	onNestedScroll(CoordinatorLayout coordinatorLayout, V child, View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, int type)
	
	onNestedFling(CoordinatorLayout coordinatorLayout, V child, View target, float velocityX, float velocityY, boolean consumed)
	
onStartNestedScroll：当CoordinatorLayout的后代尝试启动嵌套滚动时调用，[链接](https://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html#onStartNestedScroll(android.support.design.widget.CoordinatorLayout, V, android.view.View, android.view.View, int, int))

onNestedScroll：当正在进行嵌套的滚动更新并且目标已经滚动或试图滚动时调用，[链接](https://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html#onNestedScroll(android.support.design.widget.CoordinatorLayout, V, android.view.View, int, int, int, int, int))

onNestedFling：当一个嵌套滚动的孩子开始一个投掷或一个行动，这将是一个投掷时调用，[链接](https://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html#onNestedFling(android.support.design.widget.CoordinatorLayout, V, android.view.View, float, float, boolean))

**第三步**：

在布局中添加：

	app:layout_behavior="自定会behavior对应的完整路径">
	
比如：

	app:layout_behavior="com.uwinltd.period.ui.club.TopicBehavior">
	
通过注解添加：自定义View类上添加@DefaultBehavior(你的Behavior.class)

比如：

	@CoordinatorLayout.DefaultBehavior(AppBarLayout.Behavior.class)
	public class AppBarLayout extends LinearLayout {
		...
	}
	
	
**自定义Behavior如何添加回调**

首先，提供获取behavior对象的方法，比如：

	    public static <V extends View> TopicBehavior<V> from(V view) {
        ViewGroup.LayoutParams params = view.getLayoutParams();
        if (!(params instanceof CoordinatorLayout.LayoutParams)) {
            throw new IllegalArgumentException("The view is not a child of CoordinatorLayout");
        }
        CoordinatorLayout.Behavior behavior = ((CoordinatorLayout.LayoutParams) params)
                .getBehavior();
        if (!(behavior instanceof TopicBehavior)) {
            throw new IllegalArgumentException(
                    "The view is not associated with TopicBehavior");
        }
        return (TopicBehavior<V>) behavior;
    }
    
然后，设置回调，比如：

	TopicBehavior behavior = TopicBehavior.from(mFlFollow);
        mBehavior.setOnStateChangeListener(new TopicBehavior.OnStateChangeListener() {
            @Override
            public void onScrollUp() {
                
            }

            @Override
            public void onScrollDown() {

            }

            @Override
            public void onHeadExpand() {

            }

            @Override
            public void onHeadCollapse() {

            }
        });
        

#（二）原理分析