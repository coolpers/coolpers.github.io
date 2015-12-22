---
layout: post
title:  "Android 下拉刷新控件原理解析"
date:   2015-12-12
categories: android pulltorefresh
tags: android pulltorefresh 
---

<link rel="stylesheet" href="http://yandex.st/highlightjs/6.2/styles/googlecode.min.css">
  
<script src="http://code.jquery.com/jquery-1.7.2.min.js"></script>
<script src="http://yandex.st/highlightjs/6.2/highlight.min.js"></script>
  
<script>hljs.initHighlightingOnLoad();</script>
<script type="text/javascript">
 $(document).ready(function(){
      $("h1,h2,h3,h4,h5,h6").each(function(i,item){
        var tag = $(item).get(0).localName;
        $(item).attr("id","wow"+i);
        $("#category").append('<a class="new'+tag+'" href="#wow'+i+'">'+$(this).text()+'</a></br>');
		$(".newh1").css("margin-left",0);
        $(".newh2").css("margin-left",20);
        $(".newh3").css("margin-left",40);
        $(".newh4").css("margin-left",60);
        $(".newh5").css("margin-left",80);
        $(".newh6").css("margin-left",100);
      });
 });
</script>
<div id="category"></div>

#一、下拉刷新控件#

&nbsp;&nbsp;&nbsp;&nbsp;下拉刷新效果由Atebits公司创始人Loren Brichter发明并申请[“下拉刷新”](http://patft.uspto.gov/netacgi/nph-Parser?Sect1=PTO1&Sect2=HITOFF&d=PALL&p=1&u=%2Fnetahtml%2FPTO%2Fsrchnum.htm&r=1&f=G&l=50&s1=8,448,084.PN.&OS=PN/8,448,084&RS=PN/8,448,084)专利，Twitter在2010年收购Atebits公司并获得此专利，同时Twitter签订[“创新者专利协议”](https://blog.twitter.com/2013/brewing-our-first-innovator%E2%80%99s-patent-agreement-patent-0)此协议是约束Twitter只能使用此专利用于防御目的，所以在项目中使用此效果完全不用考虑侵权的问题。

&nbsp;&nbsp;&nbsp;&nbsp;目前手机百度、微信、FaceBook、新浪微博等大量应用都有此效果，最早的开源项目是由johannilsson在2011年1月9日发布的[android-pulltorefresh](https://github.com/johannilsson/android-pulltorefresh)不过此开源项目已经不维护，且作者推荐使用v4 support library中的[SwipeRefreshLayout](https://developer.android.com/reference/android/support/v4/widget/SwipeRefreshLayout.html)。另外由chrisbanes发布的开源项目[Android-PullToRefresh](https://github.com/chrisbanes/Android-PullToRefresh) 不仅支持ListView下拉刷新，还支持GrdiView等其他控件支持下拉刷新效果。本篇分析下这两个框架的使用与原理。

<br><br>

#二、下拉刷新交互#
下图来自johannilsson GitHub首页：
![](/assets/posts/2015-12-22-pull-to-refresh/johannilsson-pull-to-refresh.png)

下拉刷新主要流程：      
![](/assets/posts/2015-12-22-pull-to-refresh/ux_pull-to-refresh.png)


&nbsp;&nbsp;&nbsp;&nbsp;上面介绍的交互都是最常见的流程，如果对交互细节与提升感兴趣可以看下[有趣的下拉刷新](http://isux.tencent.com/pull-down-to-reflesh.html)与[快来QQ空间玩小鸟！~](http://djt.qq.com/article/view/911)这两篇文章讨论如何把下拉刷新更趣味。      

&nbsp;&nbsp;&nbsp;&nbsp;此控件最早是在IOS平台，当移植到Andorid平台是并不是一片赞誉，感兴趣可以看下Cyril Mottier的评论["Pull-to-refresh": An Anti UI Pattern on Android](http://cyrilmottier.com/2012/03/28/the-pull-to-refresh-an-anti-ui-pattern-on-android/) ，还有来自著名博客Android UI Patterns不一样的观点[Pull-to-refresh, or not?](http://www.androiduipatterns.com/2012/03/pull-to-refresh-or-not.html)与[Google's first pull-to-refresh - a good first try](http://www.androiduipatterns.com/2013/06/googles-first-pull-to-refresh-good.html)。
<br><br>
#三、chrisbanes Android-PullToRefresh开源项目#

项目地址： [Android-PullToRefresh](https://github.com/chrisbanes/Android-PullToRefresh)      


##1.控件特点##

* 支持顶部下拉、底部上拉、左侧向右滑动、右侧向左滑动刷新  
* 在2.3以上手机支持Over Scroll  
* 支持控件  
&nbsp;&nbsp;&nbsp;&nbsp;ListView、ExpandableListView、GridView、WebView、ScrollView、HorizontalScrollView、ViewPager、ListFragment  
* 提供滚动到列表底部的监听  
* 大量定制选项  
<br><br>
##2.控件使用##

* 布局文件      
      
		<com.handmark.pulltorefresh.library.PullToRefreshListView      
		    android:id="@+id/pull_refresh_list"      
		    android:layout_width="fill_parent"      
		    android:layout_height="fill_parent"      
		    android:cacheColorHint="#00000000"      
		    android:divider="#19000000"      
		    android:dividerHeight="4dp"      
		    android:fadingEdge="none"      
		    android:fastScrollEnabled="false"      
		    android:footerDividersEnabled="false"      
		    android:headerDividersEnabled="false"      
		    android:smoothScrollbar="true" />      

* 使用      

		mPullRefreshListView = (PullToRefreshListView) findViewById(R.id.pull_refresh_list);

		// Set a listener to be invoked when the list should be refreshed.
		mPullRefreshListView.setOnRefreshListener(new OnRefreshListener<ListView>() {
			@Override
			public void onRefresh(PullToRefreshBase<ListView> refreshView) {
				String label = DateUtils.formatDateTime(getApplicationContext(), System.currentTimeMillis(),
						DateUtils.FORMAT_SHOW_TIME | DateUtils.FORMAT_SHOW_DATE | DateUtils.FORMAT_ABBREV_ALL);
				
				// Update the LastUpdatedLabel
				refreshView.getLoadingLayoutProxy().setLastUpdatedLabel(label);

				// Do work to refresh the list here.
				new GetDataTask().execute();
			}
		});
<br><br>
##3. 源码分析##


类图      
![](/assets/posts/2015-12-22-pull-to-refresh/class_diagrams.png)

&nbsp;&nbsp;&nbsp;&nbsp;由类图可以看出此项目的主要功能类结构，此处仅分析最核心的PullToRefreshBase，通过ListView下拉刷新的一种情形分析整个流程。分析主要分为4块，布局、下拉手势判断、视图随手指移动与松手后自动回滚。  
<br><br>
###1 布局###

	private void init(Context context, AttributeSet attrs) {
		// 因为是LinearLayout的子类，可以根据需要设置横纵向展示
		switch (getPullToRefreshScrollDirection()) {
			case HORIZONTAL:
				setOrientation(LinearLayout.HORIZONTAL);
				break;
			case VERTICAL:
			default:
				setOrientation(LinearLayout.VERTICAL);
				break;
		}

		// ..... 读取一些参数

		// 创建与添加需要下拉刷新的视图
		mRefreshableView = createRefreshableView(context, attrs);
		addRefreshableView(context, mRefreshableView);

	    // 仅创建顶部与底部视图，并未添加视图到视图树中
		mHeaderLayout = createLoadingLayout(context, Mode.PULL_FROM_START, a);
		mFooterLayout = createLoadingLayout(context, Mode.PULL_FROM_END, a);

		// ..... 读取一些参数
		
		// 刷新UI
		updateUIForMode();
	}

	protected void updateUIForMode() {
		// We need to use the correct LayoutParam values, based on scroll
		// direction
		final LinearLayout.LayoutParams lp = getLoadingLayoutLayoutParams();
		
		// 添加Header View到顶部
		if (this == mHeaderLayout.getParent()) {
			removeView(mHeaderLayout);
		}
		if (mMode.showHeaderLoadingLayout()) {
			addViewInternal(mHeaderLayout, 0, lp);
		}

		// 添加Header View到底部
		if (this == mFooterLayout.getParent()) {
			removeView(mFooterLayout);
		}
		if (mMode.showFooterLoadingLayout()) {
			addViewInternal(mFooterLayout, lp);
		}

		// Hide Loading Views
		refreshLoadingViewsSize();

		// If we're not using Mode.BOTH, set mCurrentMode to mMode, otherwise
		// set it to pull down
		mCurrentMode = (mMode != Mode.BOTH) ? mMode : Mode.PULL_FROM_START;
	}


	protected final void refreshLoadingViewsSize() {
		final int maximumPullScroll = (int) (getMaximumPullScroll() * 1.2f);

		int pLeft = getPaddingLeft();
		int pTop = getPaddingTop();
		int pRight = getPaddingRight();
		int pBottom = getPaddingBottom();

		switch (getPullToRefreshScrollDirection()) {
			case HORIZONTAL:
				if (mMode.showHeaderLoadingLayout()) {
					mHeaderLayout.setWidth(maximumPullScroll);
					pLeft = -maximumPullScroll;
				} else {
					pLeft = 0;
				}

				if (mMode.showFooterLoadingLayout()) {
					mFooterLayout.setWidth(maximumPullScroll);
					pRight = -maximumPullScroll;
				} else {
					pRight = 0;
				}
				break;

			case VERTICAL:
				if (mMode.showHeaderLoadingLayout()) {
					mHeaderLayout.setHeight(maximumPullScroll);
					pTop = -maximumPullScroll;
				} else {
					pTop = 0;
				}

				if (mMode.showFooterLoadingLayout()) {
					mFooterLayout.setHeight(maximumPullScroll);
					pBottom = -maximumPullScroll;
				} else {
					pBottom = 0;
				}
				break;
		}

		if (DEBUG) {
			Log.d(LOG_TAG, String.format("Setting Padding. L: %d, T: %d, R: %d, B: %d", pLeft, pTop, pRight, pBottom));
		}
		
		// 通过把相应padding设置被负数，隐藏Header与Footer
		setPadding(pLeft, pTop, pRight, pBottom);
	}

&nbsp;&nbsp;&nbsp;&nbsp;PullToRefreshBase继承自LinearLayout，好处在于此效果视图都是横向或者纵向依次排布，完全可以复用LinearLayout排布视图的逻辑，不用自己再覆写onMeasure, onLayout去测量与排布视图，只需要设置Orientation属性并依次添加Header、RefreshableView、FooterView3个视图。      

&nbsp;&nbsp;&nbsp;&nbsp;HeaderView在refreshLoadingViewsSize函数中通过设置-paddingTop达到此默认状态不展示顶部视图的效果。布局已经完成接下来看下第2块，控件是如果进行手势判断的。
<br><br><br><br>
###2 下拉手势判断###

	@Override
	public final boolean onInterceptTouchEvent(MotionEvent event) {
		if (!isPullToRefreshEnabled()) {
			return false;
		}

		final int action = event.getAction();

		// 手指抬起与取消操作，不把事件拦截给onTouchEvent
		if (action == MotionEvent.ACTION_CANCEL || action == MotionEvent.ACTION_UP) {
			mIsBeingDragged = false;
			return false;
		}

		// 已经符合下拉刷新条件，果断拦截
		if (action != MotionEvent.ACTION_DOWN && mIsBeingDragged) {
			return true;
		}

		switch (action) {
			case MotionEvent.ACTION_MOVE: {
				// If we're refreshing, and the flag is set. Eat all MOVE events
				if (!mScrollingWhileRefreshingEnabled && isRefreshing()) {
					return true;
				}

				if (isReadyForPull()) {
					final float y = event.getY(), x = event.getX();
					final float diff, oppositeDiff, absDiff;

					// We need to use the correct values, based on scroll
					// direction
					switch (getPullToRefreshScrollDirection()) {
						case HORIZONTAL:
							diff = x - mLastMotionX;
							oppositeDiff = y - mLastMotionY;
							break;
						case VERTICAL:
						default:
							diff = y - mLastMotionY;
							oppositeDiff = x - mLastMotionX;
							break;
					}
					// 滑动的绝对值，仅用于获取移动长度，后续有单独的方向判断
					absDiff = Math.abs(diff);

					// 手指在屏幕上的移动距离已经满足滚动条件，但是移动距离小于TouchSlop时ListView ItemView会处于tap按下状态，此时并不拦截
					// 需要纵轴的移动距离大于横轴的移动距离，目的是斜着在屏幕上滑动时不会触发。
					if (absDiff > mTouchSlop && (!mFilterTouchEvents || absDiff > Math.abs(oppositeDiff))) {
						// diff >= 1f 方向判断，说明是向下或者向右
						if (mMode.showHeaderLoadingLayout() && diff >= 1f && isReadyForPullStart()) {
							mLastMotionY = y;
							mLastMotionX = x;
							mIsBeingDragged = true;
							if (mMode == Mode.BOTH) {
								mCurrentMode = Mode.PULL_FROM_START;
							}
						} else if (mMode.showFooterLoadingLayout() && diff <= -1f && isReadyForPullEnd()) {
							mLastMotionY = y;
							mLastMotionX = x;
							mIsBeingDragged = true;
							if (mMode == Mode.BOTH) {
								mCurrentMode = Mode.PULL_FROM_END;
							}
						}
					}
				}
				break;
			}
			case MotionEvent.ACTION_DOWN: {
				if (isReadyForPull()) {
					mLastMotionY = mInitialMotionY = event.getY();
					mLastMotionX = mInitialMotionX = event.getX();
					mIsBeingDragged = false;
				}
				break;
			}
		}

		// 返回true说明已经触发拖住刷新
		return mIsBeingDragged;
	}

    // 来自PullToRefreshAdapterViewBase，用于判断ListView与GridView是否满足下拉条件
	protected boolean isReadyForPullStart() {
		return isFirstItemVisible();
	}

	private boolean isFirstItemVisible() {
		final Adapter adapter = mRefreshableView.getAdapter();

		if (null == adapter || adapter.isEmpty()) {
			if (DEBUG) {
				Log.d(LOG_TAG, "isFirstItemVisible. Empty View.");
			}
			return true;

		} else {

			/**
			 * This check should really just be:
			 * mRefreshableView.getFirstVisiblePosition() == 0, but PtRListView
			 * internally use a HeaderView which messes the positions up. For
			 * now we'll just add one to account for it and rely on the inner
			 * condition which checks getTop().
			 */
			if (mRefreshableView.getFirstVisiblePosition() <= 1) {
				final View firstVisibleChild = mRefreshableView.getChildAt(0);
				if (firstVisibleChild != null) {
					return firstVisibleChild.getTop() >= mRefreshableView.getTop();
				}
			}
		}

		return false;
	}
<br>
&nbsp;&nbsp;&nbsp;&nbsp;此控件在刷新视图外添加一层LinearLayout，然后通过onInterceptTouchEvent函数判断如何满足下拉刷新条件进行拦截手势处理，不继续派发给刷新视图，因为这两个条件此控件可以支持任意视图，例如ListView、Gridview，仅需要这些视图然后告知PullToRefreshBase何时满足下拉刷新条件即可（ListView下拉刷新是告知已到达ListView顶部）。

<br><br><br><br>
###Android 事件传递流程###

![](/assets/posts/2015-12-22-pull-to-refresh/touch.png)      

* 1）传递流程      
传递: ViewGroup/View.dispatchTouchEvent(MotionEvent)      
拦截: ViewGroup.onInterceptTouchEvent(MotionEvent)      
处理: ViewGroup/View.onTouchEvent(MotionEvent)      

&nbsp;&nbsp;&nbsp;&nbsp;Android事件每隔几毫秒派发一次，在View层传递主要涉及以上3个函数，由上图的ViewRoot向下逐层传递，每层仅有一个视图满足传递条件，通过调用满足条件子视图的dispatchTouchEvent向下传递事件，如果有视图消耗此事件再向上返回true，表示此次事件已经被处理。


* 2）单次传递规律      
向下传递:      

		ViewGroup.dispatchTouchEvent()      
			通过当前所有子视图添加顺序（addView）的反序遍历，是否满足以下条件      
			判断子视图是否显示，如果不显示肯定不需要向此子视图派发         
			检查位置是否在当前子视图内。     
			以上是最常见判断，如果满足调用此视图dispatchTouchEvent传递事件            
		      
		ViewGroup.onInterceptTouchEvent()      
			如果传递到当前视图，通过覆写此函数返回true，拦截此事件并派发给当前视图onTouchEvent函数    
			通用用于手势冲突            
		
		View.onTouchEvent()      
			通用用于手势处理，例如控制视图移动，处理视图点击行为等。
 			此函数中会调用的回调：
			setOnClickListener         
			setOnLongClickListener       
			setOnTouchListener      
			setOnItemClickListener    

向上传递：      

		有视图处理，onTouchEvent return true（消耗），View.dispatchTouchEvent()逐层返回true。            
		最底层子视图未处理，会返回上层，父视图是否处理。      


* 4）基础知识
MotionEvent      

		getX(),getY() 获取的是当前视图针对当前父视图的x，y轴距离      
		getRawX(),getRawY() 获取的是针对屏幕左上角的距离。      
	    时间、历史记录,多点            
		事件类型ACTION_DOWN, ACTION_UP, ACTION_MOVE, ACTION_CANCEL, ACTION_POINTER_DOWN, ACTION_POINTER_UP            
	

* 5）手势识别：      
单点：GestureDetector      
多点缩放：ScaleGestureDetector


&nbsp;&nbsp;&nbsp;&nbsp;以上比较简单的总结Touch事件，详细可查看文档[Mastering the Android Touch System](http://wugengxin.cn/download/pdf/android/PRE_andevcon_mastering-the-android-touch-system.pdf)
<br><br><br><br>
###3 视图随手指移动###

	@Override
	public final boolean onTouchEvent(MotionEvent event) {
	    
		if (!isPullToRefreshEnabled()) {
			return false;
		}

		// If we're refreshing, and the flag is set. Eat the event
		if (!mScrollingWhileRefreshingEnabled && isRefreshing()) {
			return true;
		}

		// 按下的时候已经到当前视图边界，已经出范围所以不是拖拽刷新
		if (event.getAction() == MotionEvent.ACTION_DOWN && event.getEdgeFlags() != 0) {
			return false;
		}

		switch (event.getAction()) {
			case MotionEvent.ACTION_MOVE: {
				if (mIsBeingDragged) {
					mLastMotionY = event.getY();
					mLastMotionX = event.getX();
					// 处理视图拖拽操作
					pullEvent();
					return true;
				}
				break;
			}

			case MotionEvent.ACTION_DOWN: {
			    // 如果手指触及的当前类的子视图未处理onTouch，此时当前onInterceptTouchEvent函数
			    // 还未满足判断是否为mIsBeingDragged的条件，所以此处需要判断是否满足滚动前的边界条件
				if (isReadyForPull()) {
					mLastMotionY = mInitialMotionY = event.getY();
					mLastMotionX = mInitialMotionX = event.getX();
					return true;
				}
				break;
			}

			case MotionEvent.ACTION_CANCEL:
			case MotionEvent.ACTION_UP: {
				if (mIsBeingDragged) {
					mIsBeingDragged = false;

					if (mState == State.RELEASE_TO_REFRESH
							&& (null != mOnRefreshListener || null != mOnRefreshListener2)) {
						setState(State.REFRESHING, true);
						return true;
					}

					// If we're already refreshing, just scroll back to the top
					if (isRefreshing()) {
						smoothScrollTo(0);
						return true;
					}

					// If we haven't returned by here, then we're not in a state
					// to pull, so just reset
					setState(State.RESET);

					return true;
				}
				break;
			}
		}

		return false;
	}

onTouchEvent函数中通过pullEvent处理视图跟随手指移动，通过smoothScrollTo处理视图自动滚动。

	private void pullEvent() {
		final int newScrollValue;
		final int itemDimension;
		final float initialMotionValue, lastMotionValue;

		switch (getPullToRefreshScrollDirection()) {
			case HORIZONTAL:
				initialMotionValue = mInitialMotionX;
				lastMotionValue = mLastMotionX;
				break;
			case VERTICAL:
			default:
				initialMotionValue = mInitialMotionY;
				lastMotionValue = mLastMotionY;
				break;
		}
		
		switch (mCurrentMode) {
			case PULL_FROM_END:
			    // 向上或者向左拖拽不能为正值
				newScrollValue = Math.round(Math.max(initialMotionValue - lastMotionValue, 0) / FRICTION);
				itemDimension = getFooterSize();
				break;
			case PULL_FROM_START:
			default:
			    // 向下或者向右如果满足拖拽条件，移动的值肯定是大于0。
			    // 移动的值/2是摩察系数，最明显的就是屏幕顶部向下滚动到底部，但是其中被拖拽视图仅向下移动屏幕的一半
			    // 如果不添加Math.round，手指在屏幕上一个像素的速度移动，此处算出的float值会永远不变，视图也不会移动
				newScrollValue = Math.round(Math.min(initialMotionValue - lastMotionValue, 0) / FRICTION);
				itemDimension = getHeaderSize();
				break;
		}

		// 视图移动使用scrollTo，所以传入的是手指在屏幕上移动的距离
		// 如果减去mTouchSlop，刚开始滚动的时候就不会有一个跳动的感觉
		setHeaderScroll(newScrollValue);

		if (newScrollValue != 0 && !isRefreshing()) {
			float scale = Math.abs(newScrollValue) / (float) itemDimension;
			switch (mCurrentMode) {
				case PULL_FROM_END:
					mFooterLayout.onPull(scale);
					break;
				case PULL_FROM_START:
				default:
				    // 通知顶部视图刷新
					mHeaderLayout.onPull(scale);
					break;
			}

			if (mState != State.PULL_TO_REFRESH && itemDimension >= Math.abs(newScrollValue)) {
				// 向下滚动的距离已经超出顶部视图高度，认为是下拉刷新状态
				setState(State.PULL_TO_REFRESH);
			} else if (mState == State.PULL_TO_REFRESH && itemDimension < Math.abs(newScrollValue)) {
			    // 向下滑动距离大于顶部视图高度，现在如果松开手已经满足刷新数据的条件
				setState(State.RELEASE_TO_REFRESH);
			}
		}
	}

	protected final void setHeaderScroll(int value) {
		if (DEBUG) {
			Log.d(LOG_TAG, "setHeaderScroll: " + value);
		}

		// -max ~ max
		// Clamp value to with pull scroll range
		final int maximumPullScroll = getMaximumPullScroll();
		value = Math.min(maximumPullScroll, Math.max(-maximumPullScroll, value));

		if (mLayoutVisibilityChangesEnabled) {
		    // 移动方向正确，且有移动距离才展示
			if (value < 0) {
				mHeaderLayout.setVisibility(View.VISIBLE);
			} else if (value > 0) {
				mFooterLayout.setVisibility(View.VISIBLE);
			} else {
				mHeaderLayout.setVisibility(View.INVISIBLE);
				mFooterLayout.setVisibility(View.INVISIBLE);
			}
		}

		if (USE_HW_LAYERS) {
			/**
			 * Use a Hardware Layer on the Refreshable View if we've scrolled at
			 * all. We don't use them on the Header/Footer Views as they change
			 * often, which would negate any HW layer performance boost.
			 */
			ViewCompat.setLayerType(mRefreshableViewWrapper, value != 0 ? View.LAYER_TYPE_HARDWARE
					: View.LAYER_TYPE_NONE);
		}

		// 移动到指定位置
		switch (getPullToRefreshScrollDirection()) {
			case VERTICAL:
				scrollTo(0, value);
				break;
			case HORIZONTAL:
				scrollTo(value, 0);
				break;
		}
	}
<br><br><br><br>
###视图移动方法###

&nbsp;&nbsp;&nbsp;&nbsp;此控件通过scrollTo函数来移动视图，目前已知有4种实现视图移动的方法：      

方法|修改值|效果|
----|-----|-------|
1|mScrollX,mScrollY|view.scrollTo(int x, int y)、scrollBy(int x, int y)，视图大小与位置（x, y）都为发未改变，移动过程中不会触发onMeasure,onLayout函数，仅触发onDraw。      
||在视图上调用此函数，当前视图不会移动而是移动其所有子视图。      
2|x,y|修改left,top,right,bottom移动视图,通过view.layout或者view.offsetTopAndBottom、view.offsetLeftAndRight函数达到效果，ListView控制Item移动使用的是后者。      
3|padding|最早的johannilsson实现的下拉刷新就是基于这种，不过需要每次都重新measure、layout才能生效。      
4|margin|从来没见过哪个开源控件使用此种方式实现，不过也是一种使视图位置改变的一种办法。      
<br><br><br><br>
###4 视图自动滚动 ###

	final void setState(State state, final boolean... params) {
		mState = state;
		if (DEBUG) {
			Log.d(LOG_TAG, "State: " + mState.name());
		}

		switch (mState) {
			case RESET:
				// 列表滚动到顶部，顶部视图也重置为默认状态
				onReset();
				break;
			case PULL_TO_REFRESH:
				// 通知顶部视图下拉中
				onPullToRefresh();
				break;
			case RELEASE_TO_REFRESH:
				// 通知顶部视图手指释放刷新中
				onReleaseToRefresh();
				break;
			case REFRESHING:
			case MANUAL_REFRESHING:
				// 自动滚动到漏出顶部视图区域
				onRefreshing(params[0]); 
				break;
			case OVERSCROLLING:
				// NO-OP
				break;
		}

		// Call OnPullEventListener
		if (null != mOnPullEventListener) {
			mOnPullEventListener.onPullEvent(this, mState, mCurrentMode);
		}
	}
	
	protected void onRefreshing(final boolean doScroll) {
        if (mMode.showHeaderLoadingLayout()) {
            mHeaderLayout.refreshing();
        }
        if (mMode.showFooterLoadingLayout()) {
            mFooterLayout.refreshing();
        }
 
        if (doScroll) {
            if (mShowViewWhileRefreshing) {
 
                // Call Refresh Listener when the Scroll has finished
                OnSmoothScrollFinishedListener listener = new OnSmoothScrollFinishedListener() {
                    @Override
                    public void onSmoothScrollFinished() {
                        callRefreshListener();
                    }
                };
 
                switch (mCurrentMode) {
                    case MANUAL_REFRESH_ONLY:
                    case PULL_FROM_END:
                        smoothScrollTo(getFooterSize(), listener);
                        break;
                    default:
                    case PULL_FROM_START:
						// 注意是负值，scrollY向下是负数，向上相反
						// 向上滚动到HeaderSize高度的位置
                        smoothScrollTo(-getHeaderSize(), listener);
                        break;
                }
            } else {
				// 回滚到初始状态
                smoothScrollTo(0);
            }
        } else {
            // We're not scrolling, so just call Refresh Listener now
            callRefreshListener();
        }
    }

	private final void smoothScrollTo(int newScrollValue, long duration, long delayMillis,
			OnSmoothScrollFinishedListener listener) {
		// 停止自动滚动动画
		if (null != mCurrentSmoothScrollRunnable) {
			mCurrentSmoothScrollRunnable.stop();
		}

		// 当前位置
		final int oldScrollValue;
		switch (getPullToRefreshScrollDirection()) {
			case HORIZONTAL:
				oldScrollValue = getScrollX();
				break;
			case VERTICAL:
			default:
				oldScrollValue = getScrollY();
				break;
		}

		if (oldScrollValue != newScrollValue) {
			if (null == mScrollAnimationInterpolator) {
				// Default interpolator is a Decelerate Interpolator
				mScrollAnimationInterpolator = new DecelerateInterpolator();
			}
			mCurrentSmoothScrollRunnable = new SmoothScrollRunnable(oldScrollValue, newScrollValue, duration, listener);

			// 执行子线程
			if (delayMillis > 0) {
				postDelayed(mCurrentSmoothScrollRunnable, delayMillis);
			} else {
				post(mCurrentSmoothScrollRunnable);
			}
		}
	}
	
	
	final class SmoothScrollRunnable implements Runnable {

		@Override
		public void run() {

			/**
			 * Only set mStartTime if this is the first time we're starting,
			 * else actually calculate the Y delta
			 */
			if (mStartTime == -1) {
				mStartTime = System.currentTimeMillis();
			} else {

				/**
				 * We do do all calculations in long to reduce software float
				 * calculations. We use 1000 as it gives us good accuracy and
				 * small rounding errors
				 */
				// 时间消耗比例
				long normalizedTime = (1000 * (System.currentTimeMillis() - mStartTime)) / mDuration;
				normalizedTime = Math.max(Math.min(normalizedTime, 1000), 0);
				
				final int deltaY = Math.round((mScrollFromY - mScrollToY)
						* mInterpolator.getInterpolation(normalizedTime / 1000f));
				mCurrentY = mScrollFromY - deltaY;
				setHeaderScroll(mCurrentY);
			}

			// If we're not at the target Y, keep going...
			if (mContinueRunning && mScrollToY != mCurrentY) {
				ViewCompat.postOnAnimation(PullToRefreshBase.this, this);
			} else {
				if (null != mListener) {
					mListener.onSmoothScrollFinished();
				}
			}
		}

		public void stop() {
			mContinueRunning = false;
			removeCallbacks(this);
		}
	}

	public class ViewCompat {

		public static void postOnAnimation(View view, Runnable runnable) {
			if (VERSION.SDK_INT >= VERSION_CODES.JELLY_BEAN) {
				SDK16.postOnAnimation(view, runnable);
			} else {
				view.postDelayed(runnable, 16);
			}
		}
	}
<br><br><br><br>
###视图自动滚动方法###
* 自动滚动的循环方式：
<br><br>
* 1.使用Handler      

		class ScrollRunnable implements Runnable {
		    @Override
		    public void run() {
		        if (currentY < toY) {
		            offsetTopAndBottom(offsetY);
					// ViewCompat.postOnAnimation
		            mHandler().postDelayed(this, 16);	            
		        }
		    }
		};

&nbsp;&nbsp;&nbsp;&nbsp;Handler发出一个消息，执行此消息时如果满足判断，改变位置再发出一个Handler消息。
<br><br>
* 2.利用系统机制

		@Override
		public void computeScroll() {
		    
		    if (currentY < toY) {
		        offsetTopAndBottom(offsetY);
		        invalidate();
		    }
		}
    
&nbsp;&nbsp;&nbsp;&nbsp;调用invalidate()函数后，最终会执行onDraw，onDraw中会调用computeScroll()函数。如果未到指定位置，再次出发刷新，达到循环的效果。
<br><br>
* 3.使用动画

		ObjectAnimator yAnimator = ObjectAnimator.ofFloat(view, "translationY", fromY, toY);


&nbsp;&nbsp;&nbsp;&nbsp;这种可以实现效果，在Android 3.0以下需要使用nineoldanimation.jar开源库，框架通过修改视图的Matix达到在Android 3.0以下视图视觉上发生移动，但是视图的位置并未发生改变导致点击视图并不一定触发视图的点击事件。
<br><br>
* 自动滚动循环过程中获取当前位置
	
	    // 需要执行自动滚动处调用      
	    mScroller.startScroll(startX, startY, dx, dy, duration);
	
	    private class ScrollerRunnable implements Runnable {  
		    @Override  
		    public void run() {  
		        final Scroller scroller = mScroller;  
				
		        if (scroller.computeScrollOffset()) {  
					final int currentY = scroller.getCurrY();

					......

		            offsetTopAndBottom(offsetY);
					
		            invalidate();  
		            mHandler.postDelayed(this, DELAY_MILLIS);  
		        }  
		    }  
	    }  
<br><br>
&nbsp;&nbsp;&nbsp;&nbsp;Scroller本身并不控制视图的移动，仅仅是提供数值。通过当前消耗时间占总时间的比例乘以总长度，算出当前移动的距离。
如果希望减速、加速滚动等可以使用Interpolator 插值器，详见：[android动画（一）Interpolator](http://my.oschina.net/banxi/blog/135633)
<br><br><br><br>
#四、Android support v4 SwipeRefreshLayout#
&nbsp;&nbsp;&nbsp;&nbsp;Android V4 在19.1与20分别提供两种样式的下拉刷新效果

&nbsp;&nbsp;&nbsp;&nbsp;Android support v4 19.1的效果如下，下拉时ListView可以跟随手指移动，但是加载视图并不是在ListView的上面，而是叠在ListView顶部。      
![](/assets/posts/2015-12-22-pull-to-refresh/SwipeRefreshLayout_19.png)      
<br><br><br><br>
&nbsp;&nbsp;&nbsp;&nbsp;Android support v4 20的效果如下，下拉时ListView不会跟随手指移动。
![](/assets/posts/2015-12-22-pull-to-refresh/SwipeRefreshLayout_20.png)      