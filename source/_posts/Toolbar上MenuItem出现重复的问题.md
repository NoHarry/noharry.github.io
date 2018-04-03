title: Toolbar上MenuItem出现重复的问题
tags: Android Bug
toc: true
---

﻿# Toolbar上MenuItem出现重复的问题






 最近碰到一个关于Toolbar的问题，这个问题的发生过程是这样的:
    当程序发生异常退出或者横竖屏转换后，Toolbar上的图标发生了重复出现的情况
    ![toolbar中menuitem重复.png-17.7kB][1]


  > 如上图所示，右边的item都出现了2次

  * 首先我认为是Toolbar的问题（左边的图标是写在布局文件中，右边是通过代码添加的Menu）
  ```java
   @Override
    public void onCreateOptionsMenu(Menu menu, MenuInflater inflater) {

        inflater.inflate(R.menu.menu_main,menu);
        MenuItem item=menu.findItem(R.id.action_search);
        MenuItem add = menu.findItem(R.id.add);
        add.setOnMenuItemClickListener(new MenuItem.OnMenuItemClickListener() {
            @Override
            public boolean onMenuItemClick(MenuItem item) {
               startActivitys(OneKeyConfig.class);
                return false;
            }
        });
        mSearchView.setMenuItem(item);
    }
  ```

  * 因此我找到了第一种解决方法
  ```
    @Override
    public void onCreateOptionsMenu(Menu menu, MenuInflater inflater) {
        menu.clear();
        inflater.inflate(R.menu.menu_main,menu);
        MenuItem item=menu.findItem(R.id.action_search);
        MenuItem add = menu.findItem(R.id.add);
        add.setOnMenuItemClickListener(new MenuItem.OnMenuItemClickListener() {
            @Override
            public boolean onMenuItemClick(MenuItem item) {
               startActivitys(OneKeyConfig.class);
                return false;
            }
        });
        mSearchView.setMenuItem(item);
    }
  ```
  > 通过添加 menu.clear()看似是解决了这个问题，这时候又发现了一些问题
  ![轮播图.png-42kB][2]
通过上图可以发现轮播图也出现了重叠，这时我意识到问题可能不止出现在了toolbar上

> 通过打印日志发现，fragment创建了2次
```
 L.d("LHY","状态：onResume："+"fragment:"+this.hashCode()+" activity:"+this.getActivity().hashCode());
```
![log日志.png-7.2kB][3]
>这时想到了onSaveInstanceState这个方法，打印出日志后可以发现，Activity在异常销毁时会调用onSaveInstanceState方法，系统会默认保存一些数据，包括Fragment
![onSaveInstacne.png-26.4kB][4]


    >从Activity源码的onSaveInstanceState方法中可以知道：
```
 protected void onSaveInstanceState(Bundle outState) {
        outState.putBundle(WINDOW_HIERARCHY_TAG, mWindow.saveHierarchyState());
        Parcelable p = mFragments.saveAllState();
        if (p != null) {
            outState.putParcelable(FRAGMENTS_TAG, p);
        }
        getApplication().dispatchActivitySaveInstanceState(this, outState);
    }
```
> 该方法调用了mFragments（FragmentManager）的saveAllState方法把Fragment数据保存到p(Parcelable) 变量中，然后通过调用outState.putParcelable(FRAGMENTS_TAG, p);
保存到Bundle 中，FRAGMENTS_TAG这个常量
```
static final String FRAGMENTS_TAG = "android:support:fragments";
```
>而在Acitvity的onCreate方法中调用mFragments（FragmentController）中的restoreAllState方法
```
protected void onCreate(@Nullable Bundle savedInstanceState) {
        if (DEBUG_LIFECYCLE) Slog.v(TAG, "onCreate " + this + ": " + savedInstanceState);
        if (mLastNonConfigurationInstances != null) {
            mFragments.restoreLoaderNonConfig(mLastNonConfigurationInstances.loaders);
        }
        if (mActivityInfo.parentActivityName != null) {
            if (mActionBar == null) {
                mEnableDefaultActionBarUp = true;
            } else {
                mActionBar.setDefaultDisplayHomeAsUpEnabled(true);
            }
        }
        if (savedInstanceState != null) {
            Parcelable p = savedInstanceState.getParcelable(FRAGMENTS_TAG);
            mFragments.restoreAllState(p, mLastNonConfigurationInstances != null
                    ? mLastNonConfigurationInstances.fragments : null);
        }
        mFragments.dispatchCreate();
        getApplication().dispatchActivityCreated(this, savedInstanceState);
        if (mVoiceInteractor != null) {
            mVoiceInteractor.attachActivity(this);
        }
        mCalled = true;
    }
```
>用于恢复所有保存的Fragment状态
```
 /**
     * Restores the saved state for all Fragments. The given FragmentManagerNonConfig are Fragment
     * instances retained across configuration changes, including nested fragments
     *
     * @see #retainNestedNonConfig()
     */
    public void restoreAllState(Parcelable state, FragmentManagerNonConfig nonConfig) {
        mHost.mFragmentManager.restoreAllState(state, nonConfig);
    }
```
> 所以fragment会被创建2次，一次是我代码中添加的fragment，一次是bundle中保存的异常退出时的fragment

* 第二种解决方法
```
@Override
    protected void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);

    }
```
>重写onSaveInstanceState只保存自己需要的数据即可

  [1]: http://static.zybuluo.com/NoHarry/qjkwhh83ykizzoaskjogefqy/toolbar%E4%B8%ADmenuitem%E9%87%8D%E5%A4%8D.png
  [2]: http://static.zybuluo.com/NoHarry/pv73bpllg3idna0ccivejv8h/%E8%BD%AE%E6%92%AD%E5%9B%BE.png
  [3]: http://static.zybuluo.com/NoHarry/253svlhyu0sdkqvo4yebfiem/log%E6%97%A5%E5%BF%97.png
  [4]: http://static.zybuluo.com/NoHarry/6uil8utp89qynttmg4jdbtgk/onSaveInstacne.png
