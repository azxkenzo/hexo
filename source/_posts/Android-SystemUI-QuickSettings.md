---
title: Android - SystemUI - QuickSettings
date: 2023-07-02 08:38:11
tags: Android
---
基于 Android 13 源码

# CentralSurfaces

UI 相关的 `CoreStartable` - `CentralSurfaces`，`CentralSurfaces` 的实现类 - `CentralSurfacesImpl`。

`CoreStartable` 的核心方法是 `start()`。看看 `CentralSurfacesImpl` 的 `start()` 里面做了什么。

```java
// class CentralSurfacesImpl
public void start() {
    mScreenLifecycle.addObserver(mScreenObserver);
    mWakefulnessLifecycle.addObserver(mWakefulnessObserver);
    mUiModeManager=mContext.getSystemService(UiModeManager.class);
    if(mBubblesOptional.isPresent()){
    mBubblesOptional.get().setExpandListener(mBubbleExpandListener);
    }

    // Do not restart System UI when the bugreport flag changes.
    mFeatureFlags.addListener(Flags.LEAVE_SHADE_OPEN_FOR_BUGREPORT,event->{
    event.requestNoRestart();
    });
    
    ......

    createAndAddWindows(result);

    ......
}
```

`createAndAddWindows()` 方法比较重要。先调用 `makeStatusBarView()` 初始化各个控件；然后调用 NotificationShadeWindowController.attach() 把 View 附加到 WindowManager 上。  

```java
// class CentralSurfacesImpl
public void createAndAddWindows(@Nullable RegisterStatusBarResult result) {
    makeStatusBarView(result);
    mNotificationShadeWindowController.attach();
    mStatusBarWindowController.attach();
}
```

`makeStatusBarView()` 中把用来显示 QS 的容器 `R.id.qs_frame` 替换成 fragment。这里使用的 fragment 是 `QSFragment`。 

```java
// class CentralSurfacesImpl
protected void makeStatusBarView(@Nullable RegisterStatusBarResult result) {
    ......

    final View container = mNotificationShadeWindowView.findViewById(R.id.qs_frame);
    if (container != null) {
        FragmentHostManager fragmentHostManager =
            mFragmentService.getFragmentHostManager(container);
        ExtensionFragmentListener.attachExtensonToFragment(
            mFragmentService,
            container,
            QS.TAG,
            R.id.qs_frame,
            mExtensionController
                .newExtension(QS.class)
                .withPlugin(QS.class)
                .withDefault(this::createDefaultQSFragment)
                .build());
        mBrightnessMirrorController = new BrightnessMirrorController(
            mNotificationShadeWindowView,
            mNotificationPanelViewController,
            mNotificationShadeDepthControllerLazy.get(),
            mBrightnessSliderFactory,
            (visible) -> {
                mBrightnessMirrorVisible = visible;
                updateScrimController();
            });
        fragmentHostManager.addTagListener(QS.TAG, (tag, f) -> {
            QS qs = (QS) f;
            if (qs instanceof QSFragment) {
                mQSPanelController = ((QSFragment) qs).getQSPanelController();
                ((QSFragment) qs).setBrightnessMirrorController(mBrightnessMirrorController);
            }
        });
    }
}
```

## QSFragment
看看 `QSFragment` 内部怎么处理的。

```java
// class QSFragment
public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container,
        Bundle savedInstanceState) {
    try {
        Trace.beginSection("QSFragment#onCreateView");
        inflater = inflater.cloneInContext(new ContextThemeWrapper(getContext(),
                R.style.Theme_SystemUI_QuickSettings));
        return inflater.inflate(R.layout.qs_panel, container, false);
    } finally {
        Trace.endSection();
    }
}
```

`QSFragment` 使用的布局是 `R.layout.qs_panel`。

```xml
<com.android.systemui.qs.QSContainerImpl xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/quick_settings_container"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:clipToPadding="false"
    android:clipChildren="false">

    <com.android.systemui.qs.NonInterceptingScrollView
        android:id="@+id/expanded_qs_scroll_view"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:elevation="@dimen/qs_panel_elevation"
        android:importantForAccessibility="no"
        android:scrollbars="none"
        android:clipChildren="false"
        android:clipToPadding="false"
        android:layout_weight="1">
        <com.android.systemui.qs.QSPanel
            android:id="@+id/quick_settings_panel"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:background="@android:color/transparent"
            android:focusable="true"
            android:accessibilityTraversalBefore="@android:id/edit"
            android:clipToPadding="false"
            android:clipChildren="false">

            <include layout="@layout/qs_footer_impl" />
        </com.android.systemui.qs.QSPanel>
    </com.android.systemui.qs.NonInterceptingScrollView>

    <include layout="@layout/quick_status_bar_expanded_header" />

    <include
        layout="@layout/footer_actions"
        android:id="@+id/qs_footer_actions"
        android:layout_height="@dimen/footer_actions_height"
        android:layout_width="match_parent"
        android:layout_gravity="bottom"
        />

    <include
        android:id="@+id/qs_customize"
        layout="@layout/qs_customize_panel"
        android:visibility="gone" />

</com.android.systemui.qs.QSContainerImpl>
```
其中，`@layout/qs_footer_impl` 是 QS 展开状态下磁贴下方的构建版本文本、指示器和磁贴编辑按钮那一栏；

`@layout/quick_status_bar_expanded_header` 是非展开状态下显示磁贴的布局；

展开状态下 `@layout/quick_status_bar_expanded_header` 显示的内容会隐藏，磁贴上面显示的内容由 `R.layout.combined_qs_header` 控制。`R.layout.combined_qs_header` 是替换了 `status_bar_expanded` 布局中的 `qs_header_stub` ViewStub 而来的。

`@layout/footer_actions` 是展开状态下底部的功能栏，包括设置、电源等。

`@layout/qs_customize_panel` 是磁贴显示的自定义界面。


再来看看 `QSFragment` 的 `onViewCreated()` 方法，各控件的初始化都在这里面。

```java
// class QSFragment
public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
    QSFragmentComponent qsFragmentComponent = mQsComponentFactory.create(this);
    mQSPanelController = qsFragmentComponent.getQSPanelController();
    mQuickQSPanelController = qsFragmentComponent.getQuickQSPanelController();

    mQSPanelController.init();
    mQuickQSPanelController.init();

    mQSFooterActionsViewModel = mFooterActionsViewModelFactory.create(/* lifecycleOwner */
            this);
    bindFooterActionsView(view);
    mFooterActionsController.init();

    mQSPanelScrollView = view.findViewById(R.id.expanded_qs_scroll_view);
    mQSPanelScrollView.addOnLayoutChangeListener(
            (v, left, top, right, bottom, oldLeft, oldTop, oldRight, oldBottom) -> {
                updateQsBounds();
            });
    mQSPanelScrollView.setOnScrollChangeListener(
            (v, scrollX, scrollY, oldScrollX, oldScrollY) -> {
                // Lazily update animators whenever the scrolling changes
                mQSAnimator.requestAnimatorUpdate();
                if (mScrollListener != null) {
                    mScrollListener.onQsPanelScrollChanged(scrollY);
                }
    });
    mHeader = view.findViewById(R.id.header);
    mFooter = qsFragmentComponent.getQSFooter();

    mQSContainerImplController = qsFragmentComponent.getQSContainerImplController();
    mQSContainerImplController.init();
    mContainer = mQSContainerImplController.getView();
    mDumpManager.registerDumpable(mContainer.getClass().getName(), mContainer);

    mQSAnimator = qsFragmentComponent.getQSAnimator();
    mQSSquishinessController = qsFragmentComponent.getQSSquishinessController();

    mQSCustomizerController = qsFragmentComponent.getQSCustomizerController();
    mQSCustomizerController.init();
    mQSCustomizerController.setQs(this);
    if (savedInstanceState != null) {
        setQsVisible(savedInstanceState.getBoolean(EXTRA_VISIBLE));
        setExpanded(savedInstanceState.getBoolean(EXTRA_EXPANDED));
        setListening(savedInstanceState.getBoolean(EXTRA_LISTENING));
        setEditLocation(view);
        mQSCustomizerController.restoreInstanceState(savedInstanceState);
        if (mQsExpanded) {
            mQSPanelController.getTileLayout().restoreInstanceState(savedInstanceState);
        }
    }
    mStatusBarStateController.addCallback(this);
    onStateChanged(mStatusBarStateController.getState());
    view.addOnLayoutChangeListener(
            (v, left, top, right, bottom, oldLeft, oldTop, oldRight, oldBottom) -> {
                boolean sizeChanged = (oldTop - oldBottom) != (top - bottom);
                if (sizeChanged) {
                    setQsExpansion(mLastQSExpansion, mLastPanelFraction,
                            mLastHeaderTranslation, mSquishinessFraction);
                }
            });
    mQSPanelController.setUsingHorizontalLayoutChangeListener(
            () -> {
                // The hostview may be faded out in the horizontal layout. Let's make sure to
                // reset the alpha when switching layouts. This is fine since the animator will
                // update the alpha if it's not supposed to be 1.0f
                mQSPanelController.getMediaHost().getHostView().setAlpha(1.0f);
                mQSAnimator.requestAnimatorUpdate();
            });

    mTunerService.addTunable(this, QS_TRANSPARENCY);
}
```
这里使用了各种 **_Controller_** 类来对布局进行控制。

## QuickQSPanelController

未展开QS对应的 Controller 是 `QuickQSPanelController`。
```java
// class QuickQSPanelController
protected void onInit() {
    super.onInit();
    updateConfig();
    updateMediaExpansion();
    mMediaHost.setShowsOnlyActiveMedia(true);
    mMediaHost.init(MediaHierarchyManager.LOCATION_QQS);
    mBrightnessSliderController.init();
}
```
这里调用 `updateConfig()` 更新配置。
```java
// class QuickQSPanelController
private void updateConfig() {
    int maxTiles = getResources().getInteger(R.integer.quick_qs_panel_max_tiles);
    int columns = getResources().getInteger(R.integer.quick_settings_num_columns);
    columns = TileUtils.getQSColumnsCount(getContext(), columns);
    mView.setMaxTiles(Math.max(columns, maxTiles));
    setTiles();
}
```
计算并设置最大磁贴数，然后调用 `setTiles()` 设置磁贴。
```java
// class QuickQSPanelController
public void setTiles() {
    List<QSTile> tiles = new ArrayList<>();
    for (QSTile tile : mHost.getTiles()) {
        tiles.add(tile);
        if (tiles.size() == mView.getNumQuickTiles()) {
            break;
        }
    }
    super.setTiles(tiles, /* collapsedView */ true);
}
```
调用 `QSHost` 的 `getTiles()` 获取所有磁贴。这里使用的 `QSHost` 是 `QSTileHost`。
```java
// class QSTileHost
public Collection<QSTile> getTiles() {
    return mTiles.values();
}
```
在获取到指定数量的磁贴后，就调用父类 `QSPanelControllerBase` 的 `setTiles()`。

## QSHost
往 `mTiles` 中填充数据的过程发生在 `onTuningChanged()` 中。`onTuningChanged()` 是 `TunerService` 的回调。
```java
// class QSTileHost
public void onTuningChanged(String key, String newValue) {
    if (!TILES_SETTING.equals(key)) {
        return;
    }
    if (newValue == null && UserManager.isDeviceInDemoMode(mContext)) {
        newValue = mContext.getResources().getString(R.string.quick_settings_tiles_retail_mode);
    }
    final List<String> tileSpecs = loadTileSpecs(mContext, newValue);
    int currentUser = mUserTracker.getUserId();
    if (currentUser != mCurrentUser) {
        mUserContext = mUserTracker.getUserContext();
        if (mAutoTiles != null) {
            mAutoTiles.changeUser(UserHandle.of(currentUser));
        }
    }
    if (tileSpecs.equals(mTileSpecs) && currentUser == mCurrentUser) return;
    Log.d(TAG, "Recreating tiles: " + tileSpecs);
    mTiles.entrySet().stream().filter(tile -> !tileSpecs.contains(tile.getKey())).forEach(
            tile -> {
                Log.d(TAG, "Destroying tile: " + tile.getKey());
                mQSLogger.logTileDestroyed(tile.getKey(), "Tile removed");
                tile.getValue().destroy();
            });
    final LinkedHashMap<String, QSTile> newTiles = new LinkedHashMap<>();
    for (String tileSpec : tileSpecs) {
        QSTile tile = mTiles.get(tileSpec);
        if (tile != null && (!(tile instanceof CustomTile)
                || ((CustomTile) tile).getUser() == currentUser)) {
            if (tile.isAvailable()) {
                if (DEBUG) Log.d(TAG, "Adding " + tile);
                tile.removeCallbacks();
                if (!(tile instanceof CustomTile) && mCurrentUser != currentUser) {
                    tile.userSwitch(currentUser);
                }
                newTiles.put(tileSpec, tile);
                mQSLogger.logTileAdded(tileSpec);
            } else {
                tile.destroy();
                Log.d(TAG, "Destroying not available tile: " + tileSpec);
                mQSLogger.logTileDestroyed(tileSpec, "Tile not available");
            }
        } else {
            // This means that the tile is a CustomTile AND the user is different, so let's
            // destroy it
            if (tile != null) {
                tile.destroy();
                Log.d(TAG, "Destroying tile for wrong user: " + tileSpec);
                mQSLogger.logTileDestroyed(tileSpec, "Tile for wrong user");
            }
            Log.d(TAG, "Creating tile: " + tileSpec);
            try {
                tile = createTile(tileSpec);
                if (tile != null) {
                    tile.setTileSpec(tileSpec);
                    if (tile.isAvailable()) {
                        newTiles.put(tileSpec, tile);
                        mQSLogger.logTileAdded(tileSpec);
                    } else {
                        tile.destroy();
                        Log.d(TAG, "Destroying not available tile: " + tileSpec);
                        mQSLogger.logTileDestroyed(tileSpec, "Tile not available");
                    }
                } else {
                    Log.d(TAG, "No factory for a spec: " + tileSpec);
                }
            } catch (Throwable t) {
                Log.w(TAG, "Error creating tile for spec: " + tileSpec, t);
            }
        }
    }
    mCurrentUser = currentUser;
    List<String> currentSpecs = new ArrayList<>(mTileSpecs);
    mTileSpecs.clear();
    mTileSpecs.addAll(newTiles.keySet()); // Only add the valid (available) tiles.
    mTiles.clear();
    mTiles.putAll(newTiles);
    if (newTiles.isEmpty() && !tileSpecs.isEmpty()) {
        // If we didn't manage to create any tiles, set it to empty (default)
        Log.d(TAG, "No valid tiles on tuning changed. Setting to default.");
        changeTilesByUser(currentSpecs, loadTileSpecs(mContext, ""));
    } else {
        String resolvedTiles = TextUtils.join(",", mTileSpecs);
        if (!resolvedTiles.equals(newValue)) {
            // If the resolved tiles (those we actually ended up with) are different than
            // the ones that are in the setting, update the Setting.
            saveTilesToSettings(mTileSpecs);
        }
        mTilesListDirty = false;
        for (int i = 0; i < mCallbacks.size(); i++) {
            mCallbacks.get(i).onTilesChanged();
        }
    }
}
```
这里调用 `loadTileSpecs()` 加载 TileSpec。
```java
// class QSTileHost
protected static List<String> loadTileSpecs(Context context, String tileList) {
    final Resources res = context.getResources();

    if (TextUtils.isEmpty(tileList)) {
        tileList = res.getString(R.string.quick_settings_tiles);
        if (DEBUG) Log.d(TAG, "Loaded tile specs from default config: " + tileList);
    } else {
        if (DEBUG) Log.d(TAG, "Loaded tile specs from setting: " + tileList);
    }
    final ArrayList<String> tiles = new ArrayList<String>();
    boolean addedDefault = false;
    Set<String> addedSpecs = new ArraySet<>();
    for (String tile : tileList.split(",")) {
        tile = tile.trim();
        if (tile.isEmpty()) continue;
        if (tile.equals("default")) {
            if (!addedDefault) {
                List<String> defaultSpecs = QSHost.getDefaultSpecs(context);
                for (String spec : defaultSpecs) {
                    if (!addedSpecs.contains(spec)) {
                        tiles.add(spec);
                        addedSpecs.add(spec);
                    }
                }
                addedDefault = true;
            }
        } else {
            if (!addedSpecs.contains(tile)) {
                tiles.add(tile);
                addedSpecs.add(tile);
            }
        }
    }
    return tiles;
}
```
这里会加载默认磁贴和其他磁贴。默认磁贴通过 `QSHost.getDefaultSpecs()` 获取。默认磁贴写在了 `R.string.quick_settings_tiles_default` 中。
```
<string name="quick_settings_tiles_default" translatable="false">
    internet,bt,flashlight,dnd,alarm,airplane,nfc,rotation,battery,controls,wallet,cast,screenrecord
</string>
```

再回到 `onTuningChanged()` 中。在调用 `loadTileSpecs()` 加载 TileSpec 完之后，就遍历 tileSpecs 创建或重用相应的 `QSTile`。
这里调用 `createTile()` 创建磁贴。
```java
// class QSTileHost
public QSTile createTile(String tileSpec) {
    for (int i = 0; i < mQsFactories.size(); i++) {
        QSTile t = mQsFactories.get(i).createTile(tileSpec);
        if (t != null) {
            return t;
        }
    }
    return null;
}
```
调用 `QSFactory` 的 `createTile()` 创建磁贴。`QSFactory` 的实现类是 `QSFactoryImpl`。
```java
// class QSFactoryImpl
public final QSTile createTile(String tileSpec) {
    QSTileImpl tile = createTileInternal(tileSpec);
    if (tile != null) {
        tile.initialize();
        tile.postStale(); // Tile was just created, must be stale.
    }
    return tile;
}


protected QSTileImpl createTileInternal(String tileSpec) {
    // Stock tiles.
    if (mTileMap.containsKey(tileSpec)
        // We should not return a Garbage Monitory Tile if the build is not Eng
        && (!tileSpec.equals(GarbageMonitor.MemoryTile.TILE_SPEC) || Build.IS_ENG)) {
        return mTileMap.get(tileSpec).get();
    }

    // Custom tiles
    if (tileSpec.startsWith(CustomTile.PREFIX)) {
        return CustomTile.create(
            mCustomTileBuilderProvider.get(), tileSpec, mQsHostLazy.get().getUserContext());
    }

    // Broken tiles.
    Log.w(TAG, "No stock tile spec: " + tileSpec);
    return null;
}
```
这里把磁贴分为了三类：Stock、Custom 和 Broken。Stock 为系统预制的磁贴；Custom 为 App 自定义的磁贴。
Stock 磁贴从 `mTileMap` 中获取。`mTileMap` 中的数据是使用 Dagger 注入的。

`QSFactoryImpl` 的注释中说明了创建默认磁贴的方法。
```
要在 SystemUI 中创建新磁贴，tile 类应扩展 {@link QSTileImpl} 并具有一个 public static final 字段 - TILE_SPEC，该字段充当该 tile 的唯一键。 
（例如{@link com.android.systemui.qs.tiles.DreamTile#TILE_SPEC}）

之后，创建或查找现有的 Module 类来容纳 tile 的 binding 方法（例如 {@link com.android.systemui.accessibility.AccessibilityModule}）。 
如果创建新 Module，请将 Module 添加到 SystemUI dagger graph 中，方法是将其包含在适当的 Module 中。
```

## QSPanelControllerBase
`QuickQSPanelController` 的 `setTiles()` 中，在获取到指定数量的磁贴后，就调用父类 `QSPanelControllerBase` 的 `setTiles()`。

```java
// class QSPanelControllerBase
public void setTiles(Collection<QSTile> tiles, boolean collapsedView) {
    if (!collapsedView && mQsTileRevealController != null) {
        mQsTileRevealController.updateRevealedTiles(tiles);
    }

    for (QSPanelControllerBase.TileRecord record : mRecords) {
        mView.removeTile(record);
        record.tile.removeCallback(record.callback);
    }
    mRecords.clear();
    mCachedSpecs = "";
    for (QSTile tile : tiles) {
        addTile(tile, collapsedView);
    }
}
```
先清空 `ArrayList<TileRecord>` 中的数据，再遍历调用 `addTile()`。
```java
// class QSPanelControllerBase
private void addTile(final QSTile tile, boolean collapsedView) {
    final TileRecord r =
            new TileRecord(tile, mHost.createTileView(getContext(), tile, collapsedView));
    try {
        QSTileViewImpl qsTileView = (QSTileViewImpl) (r.tileView);
        if (qsTileView != null) {
            qsTileView.setQsLogger(mQSLogger);
        }
    } catch (ClassCastException e) {
        Log.e(TAG, "Failed to cast QSTileView to QSTileViewImpl", e);
    }
    mView.addTile(r);
    mRecords.add(r);
    mCachedSpecs = getTilesSpecs();
}
```
先调用 `QSHost` 的 `createTileView()` 创建 `QSTileView`。
```java
// class QSTileHost
public QSTileView createTileView(Context themedContext, QSTile tile, boolean collapsedView) {
    for (int i = 0; i < mQsFactories.size(); i++) {
        QSTileView view = mQsFactories.get(i)
                .createTileView(themedContext, tile, collapsedView);
        if (view != null) {
            return view;
        }
    }
    throw new RuntimeException("Default factory didn't create view for " + tile.getTileSpec());
}
```
```java
// class QSFactoryImpl
public QSTileView createTileView(Context context, QSTile tile, boolean collapsedView) {
    QSIconView icon = tile.createTileView(context);
    return new QSTileViewImpl(context, icon, collapsedView);
}
```

然后调用 `QuickQSPanel` 的 `addTile()` 添加磁贴。
```java
// class QSPanel
void addTile(QSPanelControllerBase.TileRecord tileRecord) {
    final QSTile.Callback callback = new QSTile.Callback() {
        @Override
        public void onStateChanged(QSTile.State state) {
            drawTile(tileRecord, state);
        }
    };

    tileRecord.tile.addCallback(callback);
    tileRecord.callback = callback;
    tileRecord.tileView.init(tileRecord.tile);
    tileRecord.tile.refreshState();

    if (mTileLayout != null) {
        mTileLayout.addTile(tileRecord);
        tileClickListener(tileRecord.tile, tileRecord.tileView);
    }
}
```
## PagedTileLayout
`mTileLayout` 是一个 `PagedTileLayout`，继承自 `ViewPager`，实现了 `QSTileLayout` 接口。
```java
// class PagedTileLayout
public void addTile(TileRecord tile) {
    mTiles.add(tile);
    forceTilesRedistribution("adding new tile");
    requestLayout();
}
```

`PagedTileLayout` 的 `PagerAdapter`：
```java
// class PagedTileLayout
private final PagerAdapter mAdapter = new PagerAdapter() {
    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
            mLogger.d("Destantiating page at", position);
            container.removeView((View) object);
            updateListening();
    }
    
    @Override
    public Object instantiateItem(ViewGroup container, int position) {
            mLogger.d("Instantiating page at", position);
            if (isLayoutRtl()) {
            position = mPages.size() - 1 - position;
            }
            ViewGroup view = mPages.get(position);
            if (view.getParent() != null) {
            container.removeView(view);
            }
            container.addView(view);
            updateListening();
            return view;
    }
    
    @Override
    public int getCount() {
            return mPages.size();
    }
    
    @Override
    public boolean isViewFromObject(View view, Object object) {
            return view == object;
    }
};
```
`PagerAdapter` 的数据源是 `mPages`，`mPages` 是 `ArrayList<TileLayout>`，这里 `TileLayout` 的实现类是 `SideLabelTileLayout`。

`mPages` 的数据填充发生在 `onFinishInflate()` 和 `onMeasure()` 中。
```java
// class PagedTileLayout
protected void onFinishInflate() {
    super.onFinishInflate();
    mPages.add(createTileLayout());
    mAdapter.notifyDataSetChanged();
}

protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {

    final int nTiles = mTiles.size();
    // If we have no reason to recalculate the number of rows, skip this step. In particular,
    // if the height passed by its parent is the same as the last time, we try not to remeasure.
    if (mDistributeTiles || mLastMaxHeight != MeasureSpec.getSize(heightMeasureSpec)
    || mLastExcessHeight != mExcessHeight){

    mLastMaxHeight=MeasureSpec.getSize(heightMeasureSpec);
    mLastExcessHeight=mExcessHeight;
    // Only change the pages if the number of rows or columns (from updateResources) has
    // changed or the tiles have changed
    int availableHeight=mLastMaxHeight-mExcessHeight;
    if(mPages.get(0).updateMaxRows(availableHeight,nTiles)||mDistributeTiles){
        mDistributeTiles=false;
        distributeTiles();
    }
    ......
}
```
其中 `distributeTiles()` 会调用 `emptyAndInflateOrRemovePages()`：
```java
// class PagedTileLayout
private void emptyAndInflateOrRemovePages() {
    final int numPages = getNumPages();
    final int NP = mPages.size();
    for (int i = 0; i < NP; i++) {
        mPages.get(i).removeAllViews();
    }
    if (NP == numPages) {
        return;
    }
    while (mPages.size() < numPages) {
        mLogger.d("Adding new page");
        mPages.add(createTileLayout());
    }
    while (mPages.size() > numPages) {
        mLogger.d("Removing page");
        mPages.remove(mPages.size() - 1);
    }
    mPageIndicator.setNumPages(mPages.size());
    setAdapter(mAdapter);
    mAdapter.notifyDataSetChanged();
    if (mPageToRestore != NO_PAGE) {
        setCurrentItem(mPageToRestore, false);
        mPageToRestore = NO_PAGE;
    }
}
```
这里调用 `getNumPages()` 计算所需的 page 数。
```java
// class PagedTileLayout
public int getNumPages() {
    final int nTiles = mTiles.size();
    // We should always have at least one page, even if it's empty.
    int numPages = Math.max(nTiles / mPages.get(0).maxTiles(), 1);

    // Add one more not full page if needed
    if (nTiles > numPages * mPages.get(0).maxTiles()) {
        numPages++;
    }

    return numPages;
}
```
`SideLabelTileLayout` 的 `maxTiles()`：
```java
// class SideLabelTileLayout
public int maxTiles() {
    // Each layout should be able to hold at least one tile. If there's not enough room to
    // show even 1 or there are no tiles, it probably means we are in the middle of setting
    // up.
    return Math.max(mColumns * mRows, 1);
}
```

回到 `distributeTiles()` 中，处理完 mPages 的数据后，就把 `mTiles` 中的 Tile 分发到各个 page 中。
```java
// class PagedTileLayout
private void distributeTiles() {
    emptyAndInflateOrRemovePages();

    final int tilesPerPageCount = mPages.get(0).maxTiles();
    int index = 0;
    final int totalTilesCount = mTiles.size();
    mLogger.logTileDistributionInProgress(tilesPerPageCount, totalTilesCount);
    for (int i = 0; i < totalTilesCount; i++) {
        TileRecord tile = mTiles.get(i);
        if (mPages.get(index).mRecords.size() == tilesPerPageCount) index++;
        mLogger.logTileDistributed(tile.tile.getClass().getSimpleName(), index);
        mPages.get(index).addTile(tile);
    }
}
```
看下 `SideLabelTileLayout` 的 `addTile()` 怎么做的。`SideLabelTileLayout` 没有重写 `addTile()`，所以这里调用父类 `TileLayout` 中的。
```java
// class TileLayout
public void addTile(TileRecord tile) {
    mRecords.add(tile);
    tile.tile.setListening(this, mListening);
    addTileView(tile);
}

protected void addTileView(TileRecord tile) {
    addView(tile.tileView);
}
```


## Tile 更新流程：
* QSTileImpl.handleRefreshState() -> handleUpdateState()
* QSTileImpl.handleRefreshState() -> handleStateChanged()
* QSTileImpl.handleStateChanged() -> QSTile.Callback.onStateChanged()
* QSTile.Callback.onStateChanged() -> QSPanel.drawTile()
* QSPanel.drawTile() -> QSTileView.onStateChanged()
* QSTileView.onStateChanged() -> QSTileViewImpl.handleStateChanged()


## Tile 测量流程
从 `SideLabelTileLayout` 的 `onMeasure()` 开始。
```java
// class TileLayout
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // If called with AT_MOST, it will limit the number of rows. If called with UNSPECIFIED
    // it will show all its tiles. In this case, the tiles have to be entered before the
    // container is measured. Any change in the tiles, should trigger a remeasure.
    final int numTiles = mRecords.size();
    final int width = MeasureSpec.getSize(widthMeasureSpec);
    final int availableWidth = width - getPaddingStart() - getPaddingEnd();
    final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    if (heightMode == MeasureSpec.UNSPECIFIED) {
        mRows = (numTiles + mColumns - 1) / mColumns;
    }
    final int gaps = mColumns - 1;
    mCellWidth =
            (availableWidth - (mCellMarginHorizontal * gaps) - mSidePadding * 2) / mColumns;

    // Measure each QS tile.
    View previousView = this;
    int verticalMeasure = exactly(getCellHeight());
    for (TileRecord record : mRecords) {
        if (record.tileView.getVisibility() == GONE) continue;
        record.tileView.measure(exactly(mCellWidth), verticalMeasure);
        previousView = record.tileView.updateAccessibilityOrder(previousView);
        mCellHeight = record.tileView.getMeasuredHeight();
    }

    int height = (mCellHeight + mCellMarginVertical) * mRows;
    height -= mCellMarginVertical;

    if (height < 0) height = 0;

    setMeasuredDimension(width, height);
}
```
这里计算出 Tile 的宽高限制 `mCellWidth` 和 `verticalMeasure`，然后调用 `QSTileView` 的 `measure()`。


## Tile 布局流程
看 `SideLabelTileLayout` 的 `onLayout()`。
```java
// class TileLayout
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    layoutTileRecords(mRecords.size(), true /* forLayout */);
}

private void layoutTileRecords(int numRecords, boolean forLayout) {
    final boolean isRtl = getLayoutDirection() == LAYOUT_DIRECTION_RTL;
    int row = 0;
    int column = 0;
    mLastTileBottom = 0;

    // Layout each QS tile.
    final int tilesToLayout = Math.min(numRecords, mRows * mColumns);
    for (int i = 0; i < tilesToLayout; i++, column++) {
        // If we reached the last column available to layout a tile, wrap back to the next row.
        if (column == mColumns) {
            column = 0;
            row++;
        }

        final TileRecord record = mRecords.get(i);
        final int top = getRowTop(row);
        final int left = getColumnStart(isRtl ? mColumns - column - 1 : column);
        final int right = left + mCellWidth;
        final int bottom = top + record.tileView.getMeasuredHeight();
        if (forLayout) {
            record.tileView.layout(left, top, right, bottom);
        } else {
            record.tileView.setLeftTopRightBottom(left, top, right, bottom);
        }
        record.tileView.setPosition(i);

        // Set the bottom to the unoverriden squished bottom. This is to avoid fake bottoms that
        // are only used for QQS -> QS expansion animations
        float scale = QSTileViewImplKt.constrainSquishiness(mSquishinessFraction);
        mLastTileBottom = top + (int) (record.tileView.getMeasuredHeight() * scale);
    }
}

```
计算出 Tile 的四边坐标，然后调用 `QSTileView` 的 `layout()` 或 `setLeftTopRightBottom()`

