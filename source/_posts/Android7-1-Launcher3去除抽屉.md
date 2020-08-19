---
title: Android7.1 Launcher3去除抽屉
date: 2020-08-07 14:48:54
categories: 
- Android系统
tags:
- Android
- Launcher3
- 系统
---

最近接了个工作，去除公司设备上Launcher的抽屉，所有应用单层展示。网上也查找了一番，最终摸索着基本完成了工作，在此记录下去除抽屉的所有操作。



### 去掉searchBox bar

1. `packages/apps/Launcher3/res/layout/qsb_default_view.xml`

   屏蔽掉FrameLayout中的布局

2. `packages/apps/Launcher3/src/com/android/launcher3/QsbContainerView.java`

   ```java
   private View getDefaultView(LayoutInflater inflater, ViewGroup parent, boolean showSetup) {
               View v = inflater.inflate(R.layout.qsb_default_view, parent, false);
               // if (showSetup) {
               //     View setupButton = v.findViewById(R.id.btn_qsb_setup);
               //     setupButton.setVisibility(View.VISIBLE);
               //     setupButton.setOnClickListener(this);
               // }
               // v.findViewById(R.id.btn_qsb_search).setOnClickListener(this);
               return v;
           }
   ```

   ```java
   @Override
           public void onClick(View view) {
               // if (view.getId() == R.id.btn_qsb_search) {
               //     getActivity().startSearch("", false, null, true);
               // } else if (view.getId() == R.id.btn_qsb_setup) {
               //     // Allocate a new widget id for QSB
               //     sSavedWidgetId = Launcher.getLauncher(getActivity())
               //             .getAppWidgetHost().allocateAppWidgetId();
               //     // Start intent for bind the widget
               //     Intent intent = new Intent(AppWidgetManager.ACTION_APPWIDGET_BIND);
               //     intent.putExtra(AppWidgetManager.EXTRA_APPWIDGET_ID, sSavedWidgetId);
               //     intent.putExtra(AppWidgetManager.EXTRA_APPWIDGET_PROVIDER, mWidgetInfo.provider);
               //     startActivityForResult(intent, REQUEST_BIND_QSB);
               // }
           }
   ```

3. `packages/apps/Launcher3/src/com/android/launcher3/Workspace.java`

   ```java
   public void addToCustomContentPage(View customContent, CustomContentCallbacks callbacks,
               String description) {
           if (getPageIndexForScreenId(CUSTOM_CONTENT_SCREEN_ID) < 0) {
               throw new RuntimeException("Expected custom content screen to exist");
           }
   
           // Add the custom content to the full screen custom page
           CellLayout customScreen = getScreenWithId(CUSTOM_CONTENT_SCREEN_ID);
           int spanX = customScreen.getCountX();
           int spanY = customScreen.getCountY();
           // CellLayout.LayoutParams lp = new CellLayout.LayoutParams(0, 0, spanX, spanY);
           // lp.canReorder  = false;
           // lp.isFullscreen = true;
           if (customContent instanceof Insettable) {
               ((Insettable)customContent).setInsets(mInsets);
           }
   
           // Verify that the child is removed from any existing parent.
           if (customContent.getParent() instanceof ViewGroup) {
               ViewGroup parent = (ViewGroup) customContent.getParent();
               parent.removeView(customContent);
           }
           customScreen.removeAllViews();
           customContent.setFocusable(true);
           customContent.setOnKeyListener(new FullscreenKeyEventListener());
           customContent.setOnFocusChangeListener(mLauncher.mFocusHandler
                   .getHideIndicatorOnFocusListener());
           // customScreen.addViewToCellLayout(customContent, 0, 0, lp, true);
           mCustomContentDescription = description;
   
           mCustomContentCallbacks = callbacks;
       }
   ```



### 去掉向上滑动的箭头

`packages/apps/Launcher3/src/com/android/launcher3/pageindicators/PageIndicatorLineCaret.java`

```java
 @Override
    protected void onFinishInflate() {
        super.onFinishInflate();
        mAllAppsHandle = (ImageView) findViewById(R.id.all_apps_handle);
        mAllAppsHandle.setImageDrawable(getCaretDrawable());
        mAllAppsHandle.setOnTouchListener(mLauncher.getHapticFeedbackTouchListener());
        mAllAppsHandle.setOnClickListener(mLauncher);
        mAllAppsHandle.setOnLongClickListener(mLauncher);
        mAllAppsHandle.setOnFocusChangeListener(mLauncher.mFocusHandler);
        mAllAppsHandle.setVisibility(8); // View.GONE
        mLauncher.setAllAppsButton(mAllAppsHandle);
    }
```

### 去掉第一屏firstPage

`packages/apps/Launcher3/src/com/android/launcher3/Workspace.java`

```java
 public void removeAllWorkspaceScreens() {
        // Disable all layout transitions before removing all pages to ensure that we don't get the
        // transition animations competing with us changing the scroll when we add pages or the
        // custom content screen
        disableLayoutTransitions();

        // Since we increment the current page when we call addCustomContentPage via bindScreens
        // (and other places), we need to adjust the current page back when we clear the pages
        if (hasCustomContent()) {
            removeCustomContentPage();
        }

        // Recycle the QSB widget
        View qsb = findViewById(getEmbeddedQsbId());
        if (qsb != null) {
            ((ViewGroup) qsb.getParent()).removeView(qsb);
        }

        // Remove the pages and clear the screen models
        removeAllViews();
        mScreenOrder.clear();
        mWorkspaceScreens.clear();

        // Ensure that the first page is always present
        // bindAndInitFirstWorkspaceScreen(qsb); // 删除第一屏

        // Re-enable the layout transitions
        enableLayoutTransitions();
    }
```

### 去掉hotseat

1.`packages/apps/Launcher3/res/xml/default_workspace_5x6.xml`

   `packages/apps/Launcher3/res/xml/default_workspace_5x5.xml`

屏蔽Hotseat布局

```
 <!-- Hotseat -->
    <!-- <include launcher:workspace="@xml/dw_phone_hotseat" /> -->
```

2. `packages/apps/Launcher3/src/com/android/launcher3/DeviceProfile.java`

```java
// Layout the page indicators
        View pageIndicator = launcher.findViewById(R.id.page_indicator);
        if (pageIndicator != null) {
            lp = (FrameLayout.LayoutParams) pageIndicator.getLayoutParams();
            if (isVerticalBarLayout()) {
                if (mInsets.left > 0) {
                    lp.leftMargin = mInsets.left + pageIndicatorLandGutterLeftNavBarPx -
                            lp.width - pageIndicatorLandWorkspaceOffsetPx;
                } else if (mInsets.right > 0) {
                    lp.leftMargin = pageIndicatorLandGutterRightNavBarPx - lp.width -
                            pageIndicatorLandWorkspaceOffsetPx;
                }
                lp.bottomMargin = workspacePadding.bottom;
            } else {
                // Put the page indicators above the hotseat
                lp.gravity = Gravity.CENTER_HORIZONTAL | Gravity.BOTTOM;
                lp.height = pageIndicatorHeightPx;
                lp.bottomMargin = 10; // 距离底部10dp，占掉Hotseat的位置
            }
            pageIndicator.setLayoutParams(lp);
        }
```

3. `packages/apps/Launcher3/src/com/android/launcher3/Workspace.java`

   ```java
    mDragController.addDragListener(new AccessibileDragListenerAdapter(
                       this, CellLayout.WORKSPACE_ACCESSIBILITY_DRAG) {
                   @Override
                   protected void enableAccessibleDrag(boolean enable) {
                       super.enableAccessibleDrag(enable);
                       // setEnableForLayout(mLauncher.getHotseat().getLayout(),enable);
                 
                   }
               });
   ```

   ```java
    /**
        * Returns a list of all the CellLayouts in the workspace.
        */
       ArrayList<CellLayout> getWorkspaceAndHotseatCellLayouts() {
           ArrayList<CellLayout> layouts = new ArrayList<CellLayout>();
           int screenCount = getChildCount();
           for (int screen = 0; screen < screenCount; screen++) {
               layouts.add(((CellLayout) getChildAt(screen)));
           }
           if (mLauncher.getHotseat() != null) {
               // layouts.add(mLauncher.getHotseat().getLayout());
           }
           return layouts;
       }
   ```

   ```java
    if (mLauncher.getHotseat() != null && !isDragWidget(d)) {
               if (isPointInSelfOverHotseat(d.x, d.y)) {
                   // layout = mLauncher.getHotseat().getLayout();
               }
           }
   ```

4. `packages/apps/Launcher3/src/com/android/launcher3/Launcher.java`

   ```java
    private void loadExtractedColorsAndColorItems() {
           // TODO: do this in pre-N as well, once the extraction part is complete.
           if (Utilities.isNycOrAbove()) {
               mExtractedColors.load(this);
               // mHotseat.updateColor(mExtractedColors, !mPaused);
               ...
           }
       }
   ```

### 去掉向上滑动显示AllApps的动画效果



`packages/apps/Launcher3/src/com/android/launcher3/Launcher.java`

```java
public void showOverviewMode(boolean animated) {

        //showOverviewMode(animated, false);

    }
```

`packages/apps/Launcher3/src/com/android/launcher3/dragndrop/DragLayer.java`

```java
 if (mDragController.onInterceptTouchEvent(ev)) {
            mActiveController = mDragController;
            return true;
        }

        // if (FeatureFlags.LAUNCHER3_ALL_APPS_PULL_UP && mAllAppsController.onInterceptTouchEvent(ev)) {
        //     mActiveController = mAllAppsController;
        //     return true;
        // }

        if (mPinchListener != null && mPinchListener.onInterceptTouchEvent(ev)) {
            // Stop listening for scrolling etc. (onTouchEvent() handles the rest of the pinch.)
            mActiveController = mPinchListener;
            return true;
        }
```

7.`packages/apps/Launcher3/src/com/android/launcher3/Launcher.java`

```java
 /**
     * Shows the apps view.
     */
    public void showAppsView(boolean animated, boolean updatePredictedApps,
            boolean focusSearchBar) {
        // markAppsViewShown();
        // if (updatePredictedApps) {
        //     tryAndUpdatePredictedApps();
        // }
        // showAppsOrWidgets(State.APPS, animated, focusSearchBar);
    }
```



### 当安装新应用时，安装的应用添加在第一层上

`packages/apps/Launcher3/src/com/android/launcher3/LauncherModel.java`

```java
 final HashMap<ComponentName, AppInfo> addedOrUpdatedApps = new HashMap<>();

            if (added != null) {
                final ArrayList<ItemInfo> addedInfos = new ArrayList<ItemInfo>(added);
                addAndBindAddedWorkspaceItems(context, addedInfos);
 
                // addAppsToAllApps(context, added);
                for (AppInfo ai : added) {
                    addedOrUpdatedApps.put(ai.componentName, ai);
                }
            }
```



### 去掉长按时的删除选项

`packages/apps/Launcher3/src/com/android/launcher3/DeleteDropTarget.java`

```java

    /** @return true for items that should have a "Remove" action in accessibility. */
    public static boolean supportsAccessibleDrop(ItemInfo info) {
        return false;
        // return (info instanceof ShortcutInfo)
        //         || (info instanceof LauncherAppWidgetInfo)
        //         || (info instanceof FolderInfo);
    }

    @Override
    protected boolean supportsDrop(DragSource source, ItemInfo info) {
        return false;
    }
```



### 去掉桌面长按	、pinch捏动作

`packages/apps/Launcher3/src/com/android/launcher3/Launcher.java`

```java
 /**
     * Shows the overview button.
     */
    public void showOverviewMode(boolean animated) {
        // showOverviewMode(animated, false);
    }

```

`packages/apps/Launcher3/src/com/android/launcher3/PinchAnimationManager.java`

```java
 /**
     * Animates to the specified progress. This should be called repeatedly throughout the pinch
     * gesture to run animations that interpolate throughout the gesture.
     * @param interpolatedProgress The progress from 0 to 1, where 0 is overview and 1 is workspace.
     */
    public void setAnimationProgress(float interpolatedProgress) {
        float interpolatedScale = interpolatedProgress * (1f - mOverviewScale) + mOverviewScale;
        float interpolatedTranslationY = (1f - interpolatedProgress) * mOverviewTranslationY;
        // mWorkspace.setScaleX(interpolatedScale);
        // mWorkspace.setScaleY(interpolatedScale);
        // mWorkspace.setTranslationY(interpolatedTranslationY);
        // setOverviewPanelsAlpha(1f - interpolatedProgress, 0);
    }
```

```java
  public void animateThreshold(float threshold, Workspace.State startState,
            Workspace.State goingTowards) {
        // if (threshold == PinchThresholdManager.THRESHOLD_ONE) {
        //     if (startState == OVERVIEW) {
        //         animateOverviewPanelButtons(goingTowards == OVERVIEW);
        //     } else if (startState == NORMAL) {
        //         animateHotseatAndQsb(goingTowards == NORMAL);
        //     }
        // } else 
        // if (threshold == PinchThresholdManager.THRESHOLD_TWO) {
        //     if (startState == OVERVIEW) {
        //         animateHotseatAndQsb(goingTowards == NORMAL);
        //         animateScrim(goingTowards == OVERVIEW);
        //     } else if (startState == NORMAL) {
        //         animateOverviewPanelButtons(goingTowards == OVERVIEW);
        //         animateScrim(goingTowards == OVERVIEW);
        //     }
        // } else if (threshold == PinchThresholdManager.THRESHOLD_THREE) {
        //     // Passing threshold 3 ends the pinch and snaps to the new state.
        //     if (startState == OVERVIEW && goingTowards == NORMAL) {
        //         mLauncher.showWorkspace(true);
        //         mWorkspace.snapToPage(mWorkspace.getCurrentPage());
        //     } else if (startState == NORMAL && goingTowards == OVERVIEW) {
        //         mLauncher.showOverviewMode(true);
        //     }
        // } else {
            Log.e(TAG, "Received unknown threshold to animate: " + threshold);
        // }
    }
```



### 修改无抽屉时的异常

`packages/apps/Launcher3/src/com/android/launcher3/allapps/AllAppsTransitionController.java`

```java
 mDiscoBounceAnimation.setTarget(this);
        if(mAppsView != null){ // 增加判断
            mAppsView.post(new Runnable() {
                @Override
                public void run() {
                    if (mDiscoBounceAnimation == null) {
                        return;
                    }
                    mDiscoBounceAnimation.start();
                }
            });
        }
```

`packages/apps/Launcher3/src/com/android/launcher3/dragndrop/DragController.java`

```java
  private PointF isFlingingToDelete(DragSource source) {
        if (true) return null;
```



### 屏蔽生成文件夹

`packages/apps/Launcher3/src/com/android/launcher3/Workspace.java`

```java

    boolean createUserFolderIfNecessary(View newView, long container, CellLayout target,
            int[] targetCell, float distance, boolean external, DragView dragView,
            Runnable postAnimationRunnable) {
                if(container == -100){ //禁止形成文件夹
                    return false;
                }
```

