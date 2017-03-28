```toml
title = "自定义View：圆形仪表盘，实现展示不同级别范围"
slug = "04-circle-range-view"
desc = "04-circle-range-view"
date = "2017-03-28 15:48:06"
update_date = "2017-03-28 15:48:06"
author = "wangganxin"
thumb = ""
tags = ["自定义View","仪表盘"]
```

>目录<br>
>一、前言<br>
>二、用法<br>
>三、实现<br>
>参考资料<br>

###一、前言

公司项目需要实现一个类似圆形仪表盘，展示不同级别范围的View，本着不重复造轮子的原则，Google了一番愣是没有找到到合适的，于是只能撸起袖子自己干。

先来看最终效果图：

![demo](/media/2017/circle-range-view.gif)

文末会附上源码链接。

<!--more-->

###二、用法

- 1.布局文件引入：

```XML

    <com.ganxin.circlerangeview.CircleRangeView
        android:id="@+id/circleRangeView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        app:rangeColorArray="@array/circlerangeview_colors"
        app:rangeTextArray="@array/circlerangeview_txts"
        app:rangeValueArray="@array/circlerangeview_values"/>

```

自定义属性：

> rangeColorArray：等级颜色数组，必填
> 
> rangeValueArray：等级数值数组，数组长度同rangeColorArray保持一致，必填
> 
> rangeTextArray：等级文本数组，数组长度同rangeColorArray保持一致，必填
> 
> borderColor：外圆弧颜色，可选
> 
> cursorColor：指示标颜色，可选
> 
> extraTextColor：附加文本颜色，可选
> 
> rangeTextSize：等级文本字体大小，可选
> 
> extraTextSize：附加文本字体大小，可选
> 


- 2.在你的onCreate方法或者fragment的onCreateView方法中，根据id绑定该控件

```Java

    CircleRangeView circleRangeView= (CircleRangeView) findViewById(R.id.circleRangeView);

```

- 3.在合适的时机，调用方法给控件设值

```Java

    List<String> extras =new ArrayList<>();
    extras.add("收缩压：116");
    extras.add("舒张压：85  ");

    //circleRangeView.setValueWithAnim(value);
    circleRangeView.setValueWithAnim(value,extras);

```

###三、实现

基本思想就是根据需要旋转的角度大小，利用ValueAnimat的线性变化监听来postInvalidate，引起onDraw的不断重绘

- 1. 初始View大小，计算画布的中心点坐标

```Java

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);

        mPadding = Math.max(
                Math.max(getPaddingLeft(), getPaddingTop()),
                Math.max(getPaddingRight(), getPaddingBottom())
        );
        setPadding(mPadding, mPadding, mPadding, mPadding);

        mLength1 = mPadding + mSparkleWidth / 2f + dp2px(12);

        int width = resolveSize(dp2px(220), widthMeasureSpec);
        mRadius = (width - mPadding * 2) / 2;

        setMeasuredDimension(width, width - dp2px(30));

        mCenterX = mCenterY = getMeasuredWidth() / 2f;

    }

```

- 2. 绘制外圆弧、指示游标

```Java

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        
        /**
         * 画圆弧背景
         */
        mPaint.setStrokeCap(Paint.Cap.ROUND);
        mPaint.setStyle(Paint.Style.STROKE);
        mPaint.setStrokeWidth(borderSize);
        mPaint.setAlpha(80);
        mPaint.setColor(borderColor);
        canvas.drawArc(mRectFProgressArc, mStartAngle + 1, mSweepAngle - 2, false, mPaint);

        mPaint.setAlpha(255);

        /**
         * 画指示标
         */
        if (isAnimFinish) {

            float[] point = getCoordinatePoint(mRadius - mSparkleWidth / 2f, mStartAngle + calculateAngleWithValue(currentValue));
            mPaint.setColor(cursorColor);
            mPaint.setStyle(Paint.Style.FILL);
            canvas.drawCircle(point[0], point[1], mSparkleWidth / 2f, mPaint);

        } else {

            float[] point = getCoordinatePoint(mRadius - mSparkleWidth / 2f, mAngleWhenAnim);
            mPaint.setColor(cursorColor);
            mPaint.setStyle(Paint.Style.FILL);
            canvas.drawCircle(point[0], point[1], mSparkleWidth / 2f, mPaint);
        }
    }


    /**
     * 根据角度和半径进行三角函数计算坐标
     * @param radius
     * @param angle
     * @return
     */
    private float[] getCoordinatePoint(float radius, float angle) {
        float[] point = new float[2];

        double arcAngle = Math.toRadians(angle); //将角度转换为弧度
        if (angle < 90) {
            point[0] = (float) (mCenterX + Math.cos(arcAngle) * radius);
            point[1] = (float) (mCenterY + Math.sin(arcAngle) * radius);
        } else if (angle == 90) {
            point[0] = mCenterX;
            point[1] = mCenterY + radius;
        } else if (angle > 90 && angle < 180) {
            arcAngle = Math.PI * (180 - angle) / 180.0;
            point[0] = (float) (mCenterX - Math.cos(arcAngle) * radius);
            point[1] = (float) (mCenterY + Math.sin(arcAngle) * radius);
        } else if (angle == 180) {
            point[0] = mCenterX - radius;
            point[1] = mCenterY;
        } else if (angle > 180 && angle < 270) {
            arcAngle = Math.PI * (angle - 180) / 180.0;
            point[0] = (float) (mCenterX - Math.cos(arcAngle) * radius);
            point[1] = (float) (mCenterY - Math.sin(arcAngle) * radius);
        } else if (angle == 270) {
            point[0] = mCenterX;
            point[1] = mCenterY - radius;
        } else {
            arcAngle = Math.PI * (360 - angle) / 180.0;
            point[0] = (float) (mCenterX + Math.cos(arcAngle) * radius);
            point[1] = (float) (mCenterY - Math.sin(arcAngle) * radius);
        }

        return point;
    }

    /**
     * 根据起始角度计算对应值应显示的角度大小
     */
    private float calculateAngleWithValue(String level) {

        int pos = -1;

        for (int j = 0; j < rangeValueArray.length; j++) {
            if (rangeValueArray[j].equals(level)) {
                pos = j;
                break;
            }
        }

        float degreePerSection = 1f * mSweepAngle / mSection;

        if (pos == -1) {
            return 0;
        } else if (pos == 0) {
            return degreePerSection / 2;
        } else {
            return pos * degreePerSection + degreePerSection / 2;
        }
    }

```

- 3. 绘制内圆弧（按颜色数组绘制）

```Java

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        
        /**
         * 画等级圆弧
         */
        mPaint.setShader(null);
        mPaint.setStyle(Paint.Style.STROKE);
        mPaint.setColor(Color.BLACK);
        mPaint.setAlpha(255);
        mPaint.setStrokeCap(Paint.Cap.SQUARE);
        mPaint.setStrokeWidth(mCalibrationWidth);

        if (rangeColorArray != null) {
            for (int i = 0; i < rangeColorArray.length; i++) {
                mPaint.setColor(Color.parseColor(rangeColorArray[i].toString()));
                float mSpaces = mSweepAngle / mSection;
                if (i == 0) {
                    canvas.drawArc(mRectFCalibrationFArc, mStartAngle + 3, mSpaces, false, mPaint);
                } else if (i == rangeColorArray.length - 1) {
                    canvas.drawArc(mRectFCalibrationFArc, mStartAngle + (mSpaces * i), mSpaces, false, mPaint);
                } else {
                    canvas.drawArc(mRectFCalibrationFArc, mStartAngle + (mSpaces * i) + 3, mSpaces, false, mPaint);
                }
            }
        }
    }

```

- 4. 绘制中心点文字及下方的附加信息

```Java

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        
        /**
         * 画等级对应值的文本（居中显示）
         */
        if (rangeColorArray != null && rangeValueArray != null && rangeTextArray != null) {

            if (!TextUtils.isEmpty(currentValue)) {
                int pos = 0;

                for (int i = 0; i < rangeValueArray.length; i++) {
                    if (rangeValueArray[i].equals(currentValue)) {
                        pos = i;
                        break;
                    }
                }

                mPaint.setColor(Color.parseColor(rangeColorArray[pos].toString()));
                mPaint.setTextAlign(Paint.Align.CENTER);

                String txt=rangeTextArray[pos].toString();

                if (txt.length() <= 4) {
                    mPaint.setTextSize(rangeTextSize);
                    canvas.drawText(txt, mCenterX, mCenterY + dp2px(10), mPaint);
                } else {
                    mPaint.setTextSize(rangeTextSize - 10);
                    String top = txt.substring(0, 4);
                    String bottom = txt.substring(4, txt.length());

                    canvas.drawText(top, mCenterX, mCenterY, mPaint);
                    canvas.drawText(bottom, mCenterX, mCenterY + dp2px(30), mPaint);
                }
            }
        }

        /**
         * 画附加信息
         */
        if (extraList != null && extraList.size() > 0) {
            mPaint.setAlpha(160);
            mPaint.setColor(extraTextColor);
            mPaint.setTextSize(extraTextSize);
            for (int i = 0; i < extraList.size(); i++) {
                canvas.drawText(extraList.get(i), mCenterX, mCenterY + dp2px(50) + i * dp2px(20), mPaint);
            }
        }

    }

```

- 5.添加线性变化监听，使onDraw重绘

```Java

    /**
     * 设置值并播放动画
     *
     * @param value  值
     * @param extras 底部附加信息
     */
    public void setValueWithAnim(String value, List<String> extras) {
        if (!isAnimFinish) {
            return;
        }

        this.currentValue = value;
        this.extraList=extras;

        // 计算最终值对应的角度，以扫过的角度的线性变化来播放动画
        float degree = calculateAngleWithValue(value);

        ValueAnimator degreeValueAnimator = ValueAnimator.ofFloat(mStartAngle, mStartAngle + degree);
        degreeValueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                mAngleWhenAnim = (float) animation.getAnimatedValue();
            }
        });

        ValueAnimator creditValueAnimator = ValueAnimator.ofInt(0, (int) degree);
        creditValueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                postInvalidate();
            }
        });

        long delay = 1500;

        AnimatorSet animatorSet = new AnimatorSet();
        animatorSet
                .setDuration(delay)
                .playTogether(creditValueAnimator, degreeValueAnimator);
        animatorSet.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationStart(Animator animation) {
                super.onAnimationStart(animation);
                isAnimFinish = false;
            }

            @Override
            public void onAnimationEnd(Animator animation) {
                super.onAnimationEnd(animation);
                isAnimFinish = true;
            }

            @Override
            public void onAnimationCancel(Animator animation) {
                super.onAnimationCancel(animation);
                isAnimFinish = true;
            }
        });
        animatorSet.start();
    }

```

- [源码传送门](https://github.com/WangGanxin/CircleRangeView) 

###参考资料
- [Android自定义仪表盘View，仿新旧两版芝麻信用分、炫酷汽车速度仪表盘](https://github.com/woxingxiao/DashboardView) 
