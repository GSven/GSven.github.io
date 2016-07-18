---
layout: post
published: true
title: 『 Android 』 自定义switchButton
description: know more, do better 
---  

## 自定义switchButton ##

**good good study,day day up!**

![效果图](https://github.com/GSven/GSven.github.io/raw/master/gif/img_kaiguan.gif)

![示例图](https://github.com/GSven/GSven.github.io/raw/master/images/img_kai.png)

![示例图1](https://github.com/GSven/GSven.github.io/raw/master/images/img_kai1.png)

![示例图2](https://github.com/GSven/GSven.github.io/raw/master/images/img_kai2.png)

![示例图3](https://github.com/GSven/GSven.github.io/raw/master/images/img_kai3.png)

*分析*

* 将swichButton拆分为左边半圆，中间矩形，右边半圆，在加上一个移动的圆 。开和关状态的背景是一样的就是颜色不一样。（也可以用过path来绘制）
* 偷懒，设置长宽比为16：9。然后获取图片的布局之后的实际尺寸
* 绘制模型图
* 事件分发 ：
 1.	移动圆移动的最大距离就是中间矩形的长 0<moveX<centerRectFWidth
 2.	需要知道cavas.scale(scale,scale,x,y)以（x,y）点作为中心缩放点。 关闭背景图有个缩放动画
 3.	手指按下的时候记录下按下的坐标，在移动的过程中，就会产生moveX，根据这个变化的moveX不停的刷新view,而view的刷新需要调用onDraw(cavas)方法,所以需要不停的invalidate()；
 4.	手指操作的时候分析：

> a模拟点击，就是手指按下抬起（不考虑长按事件）基本没移动,这时候switchButton的开关状态改变,变为相反;
> b移动距离没有超过最大移动距离一半,我们这时认为状态改变不成功，即发生回弹效果; 
> c移动距离超过最大移动距离的一半,我们认为开关状态改变成功;
				
* 具体代码请看源码把。	

    	package com.gaobei.views;

		import android.annotation.TargetApi;
		import android.content.Context;
		import android.graphics.Canvas;
		import android.graphics.Color;
		import android.graphics.Paint;
		import android.graphics.RectF;
		import android.os.Build;
		import android.util.AttributeSet;
		import android.view.MotionEvent;
		import android.view.View;
		import android.view.animation.AccelerateInterpolator;


		/**
		 * Created by Administrator on 2016/5/27.
		 * @Description  自定义swithchButton
		 */
		public class MySwitchButtonView extends View {
		    /**
		     * 打开状态背景的画笔
		     */
		    private Paint mOpenPaint;
		    /**
		     * 关闭状态背景的画笔
		     */
		    private Paint mClosePaint;
		    /**
		     * 切换的实心园画笔
		     */
		    private Paint mCirclePaint;
		    /**
		     * 加速插值器
		     */
		    private final AccelerateInterpolator interpolator = new AccelerateInterpolator();
		    /**
		     * 横向移动的距离
		     */
		    private float moveX;
		    /**
		     * 中间矩形的长
		     */
		    private float mCenterRectFWidth;
		    /**
		     *当前横坐标
		     */
		    private float currentX;
		    /**
		     * 点击事件最小移动距离
		     */
		    private static final int slop = 8; //      ViewConfiguration.get(context).getScaledTouchSlop()
		    /**
		     * 手指按下时的横坐标
		     */
		    private float lastX;
		    /**
		     * 实心圆移动的 x坐标 的开始位置，结束位置
		     */
		    private float mCircleMoveStartX, mCircleMoveEndX;
		    /**
		     * 是否打开状态
		     */
		    private boolean isOpen;
		    /**
		     * 状态改变的回调函数
		     */
		    private OnStateChangeListener listener;
		    /**
		     * setState()需要刷新view的标识
		     */
		    private boolean isRefreshState;
		    /**
		     * 实心圆半径
		     */
		    private int mCircleRadius;
		    /**
		     * view的宽度
		     */
		    private int width;
		    /**
		     * view的高度
		     */
		    private int height;
		    /**
		     * 左、中、右 图片的矩形
		     */
		    private RectF oLeftRectF, oCenterRectF, oRightRectF;
		    /**
		     *关闭状态背景图缩放的左右中心点
		     * (mLScalePointX,mLScalePointY),(mRScalePointX,mRScalePointY)
		     */
		    private float mLScalePointX;
		    private float mLscalePointY;
		    private float mRScalePointX;
		    private float mRscalePointY;
	
	
		    public MySwitchButtonView(Context context) {
			this(context, null);
		    }
	
		    public MySwitchButtonView(Context context, AttributeSet attrs) {
			this(context, attrs, 0);
		    }
	
		    public MySwitchButtonView(Context context, AttributeSet attrs, int defStyleAttr) {
			super(context, attrs, defStyleAttr);
			init();
		    }
	
		    @TargetApi(Build.VERSION_CODES.M)
		    private void init() {
			mOpenPaint = new Paint();
			mOpenPaint.setColor(Color.parseColor("#FF4081"));
			mOpenPaint.setStyle(Paint.Style.FILL_AND_STROKE);
			mOpenPaint.setAntiAlias(true);
	
			mClosePaint = new Paint();
			mClosePaint.setColor(Color.parseColor("#FFDFD2"));
			mClosePaint.setStyle(Paint.Style.FILL_AND_STROKE);
			mClosePaint.setAntiAlias(true);
	
			mCirclePaint = new Paint();
			mCirclePaint.setColor(Color.parseColor("#eaf0f1"));
			mCirclePaint.setStyle(Paint.Style.FILL_AND_STROKE);
			mCirclePaint.setAntiAlias(true);
		    }
	
		    @Override
		    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
			//长宽比16：9
			setMeasuredDimension(widthMeasureSpec, widthMeasureSpec * 9 / 16);
		    }
	
	
		    @Override
		    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
		//        super.onSizeChanged(w, h, oldw, oldh);
			mCircleRadius = h / 2;
			height = h;
			width = w;
	
			oLeftRectF = new RectF(0, 0, h, h);
			oCenterRectF = new RectF(h / 2, 0, w - h / 2, h);
			oRightRectF = new RectF(w - h, 0, w, h);
	
			mLScalePointX = (oLeftRectF.right - oLeftRectF.left) / 2;
			mLscalePointY = (oLeftRectF.bottom - oLeftRectF.top) / 2;
			mRScalePointX = mLScalePointX + oCenterRectF.right - oCenterRectF.left;
			mRscalePointY = (oRightRectF.bottom - oRightRectF.top) / 2;
	
	
			mCenterRectFWidth = oCenterRectF.right - oCenterRectF.left;
			mCircleMoveStartX = mCircleRadius;
			mCircleMoveEndX = mCircleRadius + mCenterRectFWidth;
		    }
	
		    @Override
		    protected void onDraw(Canvas canvas) {
			super.onDraw(canvas);
			//open
			canvas.drawArc(oLeftRectF, 90, 180, true, mOpenPaint);
			canvas.drawRect(oCenterRectF, mOpenPaint);
			canvas.drawArc(oRightRectF, -90, 180, true, mOpenPaint);
	
			//setState()
			if (isRefreshState) {
			    moveX = isOpen ? getMoveX(mCenterRectFWidth + mCircleRadius) : getMoveX(mCircleRadius);
			    isRefreshState = false;
			}
			//close
		//        mClosePaint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST_OVER));
			float scale = 1 - moveX / mCenterRectFWidth;
			scale = interpolator.getInterpolation(scale);
			canvas.save();
			canvas.scale(scale, scale, mRScalePointX, mRscalePointY);
			canvas.drawArc(oLeftRectF, 90, 180, true, mClosePaint);
			canvas.drawRect(oCenterRectF, mClosePaint);
			canvas.drawArc(oRightRectF, -90, 180, true, mClosePaint);
			canvas.restore();
	
			//circle
			canvas.save();
			canvas.drawCircle(mCircleRadius + moveX, height / 2, mCircleRadius, mCirclePaint);
			canvas.restore();
	
		    }
	
	
		    @Override
		    public boolean onTouchEvent(MotionEvent event) {
			int action = event.getAction();
			switch (action) {
			    case MotionEvent.ACTION_DOWN:
				currentX = event.getX();
				lastX = event.getX();
				break;
			    case MotionEvent.ACTION_MOVE:
				currentX = event.getX();
				getMoveX(currentX);
				invalidate();
				break;
			    case MotionEvent.ACTION_UP:
				currentX = event.getX();
				//<slop 点击事件，状态改变
				if (Math.abs(currentX - lastX) < slop) {
				    if (isOpen) {
					setMoveXByXchange(mCircleMoveEndX, mCircleMoveStartX);
				    } else {
					setMoveXByXchange(mCircleMoveStartX, mCircleMoveEndX);
				    }
				    isOpen = !isOpen;
				    //移动距离小于一半，回弹事件，状态不变
				} else if (Math.abs(currentX - lastX) > slop && Math.abs((currentX - lastX)) <= (mCenterRectFWidth / 2)) {
				    if (isOpen) {
					setMoveXByXchange(currentX, mCircleMoveEndX);
				    } else {
					setMoveXByXchange(currentX, mCircleMoveStartX);
				    }
				    //移动距离大于一半，状态改变
				} else if (Math.abs((currentX - lastX)) > (mCenterRectFWidth / 2)) {
				    if (isOpen) {
					setMoveXByXchange(currentX, mCircleMoveStartX);
				    } else {
					setMoveXByXchange(currentX, mCircleMoveEndX);
				    }
				    isOpen = !isOpen;
				}
				if (listener != null) {
				    listener.onStateChangeListener(this, isOpen);
				}
	
				break;
			}
	
			return true;
		    }
	
	
		    public boolean getState() {
			return isOpen;
		    }
	
		    public void setState(boolean isOpen) {
			this.isOpen = isOpen;
			isRefreshState = true;
			postInvalidate();
		    }
	
		    /**
		     * @param startX 初始位置
		     * @param endX   最终位置
		     */
		    private void setMoveXByXchange(float startX, float endX) {
			int i = 1;
			float tempX;
			while (i < 11) {
			    tempX = startX + (endX - startX) / 10 * i;
			    getMoveX(tempX);
			    i++;
			    invalidate();
			}
		    }
	
	
		    /**
		     * @param x 最终位置X
		     * @return
		     */
		    private float getMoveX(float x) {
			moveX = x - mCircleRadius;
			if (moveX <= 0) {
			    moveX = 0;
			} else if (moveX > mCenterRectFWidth) {
			    moveX = mCenterRectFWidth;
			}
			return moveX;
		    }
	
	
		    public void setStateChangeListener(OnStateChangeListener listener) {
			this.listener = listener;
		    }
	
		    public interface OnStateChangeListener {
			void onStateChangeListener(View view, boolean isOpen);
		    }
		}



