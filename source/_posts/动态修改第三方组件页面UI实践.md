---
title: 动态修改第三方组件页面UI实践
date: 2017-09-11 11:12:38
tags: Android
categories: 最佳实践
---
**需求场景举例：**

当我们项目中使用了第三方组件的页面，而组件现有的页面又不能满足我们项目的设计需求，此时你该如何做呢？
向组件开发方提需求，让他满足自己一次？
如果组件是自家人开发的或许还可以，那要是其他团队或者公司的开发的呢，人家会鸟你吗？
很显然，一个简单而特殊的需求，组件开发者不一定会给你做改动，那么该怎么办呢？
那只有我们自己改喽！

<!-- more -->

**一些思考：**

如何修改ui呢？
1. 修改布局文件
2. java代码中动态修改

好像1不太现实直接修改aar里面布局文件，那剩下的就是在java代码中动态修改喽。

**如何修改？**

关键是拿到Actvity或Fragment实例，进而获取想要动态进行修改的视图实例（若要添加视图元素需要获取到父视图实例）。

**改如何做呢？**

`ActivityLifecycleCallbacks`用于监听整个应用的`Activity`生命周期变化的接口，接口声明如下：
```java
   public interface ActivityLifecycleCallbacks {
        void onActivityCreated(Activity var1, Bundle var2);

        void onActivityStarted(Activity var1);

        void onActivityResumed(Activity var1);

        void onActivityPaused(Activity var1);

        void onActivityStopped(Activity var1);

        void onActivitySaveInstanceState(Activity var1, Bundle var2);

        void onActivityDestroyed(Activity var1);
    }
```
从接口生命可以发现，所有的抽象方法对应`Activity`的生命周期方法，在`Activity`每个生命周期阶段都会回调接口实现当中相应的方法。还可以发现每个方法的参数都要`Activity`，没错，这个Activity就是生命周期正在发生变化的那个，这样我们就可以通过这个`Activity`引用，就可以拿到他的View视图和Fragement实例等等，我们需要的对象引用，进而进行动态修改。
#### 具体实践举例
这里的例子是向其他组件的页面中添加两个视图元素，并响应点击事件。
##### 实现监听接口
```java
public class BeiKaoActivityLefecycleCallback implements Application.ActivityLifecycleCallbacks {
    private IConsultEntryController mController;
    private final String CURRENT_ACTIVITY_NAME = "com.nd.module_im.im.activity.ChatActivity";

    public BeiKaoActivityLefecycleCallback(Activity activity) {
        mController = new ConsultEntryControllerImpl(activity);
    }
    @Override
    public void onActivityCreated(Activity activity, Bundle bundle) {

    }

    @Override
    public void onActivityStarted(Activity activity) {

    }

    @Override
    public void onActivityResumed(Activity activity) {
        Log.d(activity.getClass().getSimpleName(), "onActivityResumed");
        String canonicalName = activity.getClass().getCanonicalName();
        if(CURRENT_ACTIVITY_NAME.equals(canonicalName)){
            mController.showFloatEntry(activity);
        }


    }

    @Override
    public void onActivityPaused(Activity activity) {

    }

    @Override
    public void onActivityStopped(Activity activity) {
        Log.d(activity.getClass().getSimpleName(), "onActivityStopped");
        String canonicalName = activity.getClass().getCanonicalName();
        if(CURRENT_ACTIVITY_NAME.equals(canonicalName)){
            mController.hideFloatEntry(activity);
        }
    }

    @Override
    public void onActivitySaveInstanceState(Activity activity, Bundle bundle) {

    }

    @Override
    public void onActivityDestroyed(Activity activity) {

    }
}
```
##### 添加生命周期监听
```java
context.registerActivityLifecycleCallbacks(new BeiKaoActivityLefecycleCallback(getActivity()));
```
##### 获取视图元素并动态修改视图
```java
ublic class ConsultEntryControllerImpl implements IConsultEntryController, View.OnClickListener {
    private TextView floatView = null;
    private LinearLayout mLinearLayout;

    public ConsultEntryControllerImpl(Activity activity) {
        initFloatView(activity);
    }

    protected void initFloatView(Activity activity) {
        floatView = new TextView(activity);
        floatView.setBackgroundResource(R.drawable.bg_primary_color_selector);
        floatView.setTextColor(AppContextUtil.getColor(R.color.white));
        floatView.setTextSize(TypedValue.COMPLEX_UNIT_SP,16);
        floatView.setText("人工客服");
        floatView.setGravity(Gravity.CENTER);
        floatView.setPadding(MixedUtils.dpToPx(activity,8),0,MixedUtils.dpToPx(activity,8),0);
        floatView.setOnClickListener(this);
        mLinearLayout = (LinearLayout) LayoutInflater.from(activity).inflate(R.layout.layout_consult_entry,null);
        mLinearLayout.setOnClickListener(this);
    }

    @Override
    public void showFloatEntry(final Activity activity) {
        Fragment fragment = ((AppCompatActivity) activity).getSupportFragmentManager().findFragmentByTag("chat");
        if(!fragment.getClass().getCanonicalName().equals("com.nd.module_im.psp.ui.activity.ChatFragment_Psp")){
            return;
        }
        RelativeLayout parent = (RelativeLayout) floatView.getParent();
        if(parent != null){
            return;
        }
        RelativeLayout frameLayout = (RelativeLayout) fragment.getView();
        RelativeLayout.LayoutParams topBtnParam = new RelativeLayout.LayoutParams(RelativeLayout.LayoutParams.WRAP_CONTENT,  MixedUtils.dpToPx(activity,56));
        topBtnParam.addRule(RelativeLayout.ALIGN_PARENT_RIGHT);
        RelativeLayout.LayoutParams linnearLayoutParam = new RelativeLayout.LayoutParams(RelativeLayout.LayoutParams.MATCH_PARENT, MixedUtils.dpToPx(activity,28));
        linnearLayoutParam.topMargin = MixedUtils.dpToPx(activity,56);
        frameLayout.addView(floatView, topBtnParam);
        frameLayout.addView(mLinearLayout,linnearLayoutParam);
        Log.d("ConsultEntryControllerI", "显示人工咨询入口图标");
    }

    @Override
    public void hideFloatEntry(Activity activity) {
        RelativeLayout parent = (RelativeLayout) floatView.getParent();
        if(parent == null){
            return;
        }
        parent.removeView(floatView);
        parent.removeView(mLinearLayout);
    }

    @Override
    public void onClick(View view) {
        CounselChatUtils.toXNCounselChat(AppContextUtil.getContext());
    }
}
```
