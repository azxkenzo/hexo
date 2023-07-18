---
title: Android - SystemUI - MediaControl
date: 2023-07-18 11:08:00
tags: Android 
---

媒体控制的相关类和方法：
* MediaHost
* MediaHostStateHolder
* MediaHierarchyManager
* MediaDataManager
* MediaHostStatesManager
* com.android.systemui.util.Utils#useQsMediaPlayer


## MediaHost
MediaHost 由 dagger 注入，有 4 个 Provider，分别对应 4 个显示位置，每个位置的 MediaHost 都是单例的。
```java
    @Provides
    @SysUISingleton
    @Named(QS_PANEL)
    static MediaHost providesQSMediaHost(MediaHost.MediaHostStateHolder stateHolder,
            MediaHierarchyManager hierarchyManager, MediaDataManager dataManager,
            MediaHostStatesManager statesManager) {
        return new MediaHost(stateHolder, hierarchyManager, dataManager, statesManager);
    }

    /** */
    @Provides
    @SysUISingleton
    @Named(QUICK_QS_PANEL)
    static MediaHost providesQuickQSMediaHost(MediaHost.MediaHostStateHolder stateHolder,
            MediaHierarchyManager hierarchyManager, MediaDataManager dataManager,
            MediaHostStatesManager statesManager) {
        return new MediaHost(stateHolder, hierarchyManager, dataManager, statesManager);
    }

    /** */
    @Provides
    @SysUISingleton
    @Named(KEYGUARD)
    static MediaHost providesKeyguardMediaHost(MediaHost.MediaHostStateHolder stateHolder,
            MediaHierarchyManager hierarchyManager, MediaDataManager dataManager,
            MediaHostStatesManager statesManager) {
        return new MediaHost(stateHolder, hierarchyManager, dataManager, statesManager);
    }

    /** */
    @Provides
    @SysUISingleton
    @Named(DREAM)
    static MediaHost providesDreamMediaHost(MediaHost.MediaHostStateHolder stateHolder,
            MediaHierarchyManager hierarchyManager, MediaDataManager dataManager,
            MediaHostStatesManager statesManager) {
        return new MediaHost(stateHolder, hierarchyManager, dataManager, statesManager);
    }
```

初始化：
```java
// class QuickQSPanelController
protected void onInit() {
    super.onInit();
    updateMediaExpansion();
    mMediaHost.setShowsOnlyActiveMedia(true);
    mMediaHost.init(MediaHierarchyManager.LOCATION_QQS);
}
```

```java
// class QSPanelController
public void onInit() {
    super.onInit();
    mMediaHost.setExpansion(MediaHostState.EXPANDED);
    mMediaHost.setShowsOnlyActiveMedia(false);
    mMediaHost.init(MediaHierarchyManager.LOCATION_QS);
    mQsCustomizerController.init();
    mBrightnessSliderController.init();
}
```

#### init()
```java
fun init(@MediaLocation location: Int) {
    if (inited) {
        return
    }
    inited = true

    this.location = location
    hostView = mediaHierarchyManager.register(this)
    // Listen by default, as the host might not be attached by our clients, until
    // they get a visibility change. We still want to stay up to date in that case!
    setListeningToMediaData(true)
    hostView.addOnAttachStateChangeListener(
        object : OnAttachStateChangeListener {
            override fun onViewAttachedToWindow(v: View?) {
                setListeningToMediaData(true)
                updateViewVisibility()
            }

            override fun onViewDetachedFromWindow(v: View?) {
                setListeningToMediaData(false)
            }
        }
    )

    // Listen to measurement updates and update our state with it
    hostView.measurementManager =
        object : UniqueObjectHostView.MeasurementManager {
            override fun onMeasure(input: MeasurementInput): MeasurementOutput {
                // Modify the measurement to exactly match the dimensions
                if (
                    View.MeasureSpec.getMode(input.widthMeasureSpec) == View.MeasureSpec.AT_MOST
                ) {
                    input.widthMeasureSpec =
                        View.MeasureSpec.makeMeasureSpec(
                            View.MeasureSpec.getSize(input.widthMeasureSpec),
                            View.MeasureSpec.EXACTLY
                        )
                }
                // This will trigger a state change that ensures that we now have a state
                // available
                state.measurementInput = input
                return mediaHostStatesManager.updateCarouselDimensions(location, state)
            }
        }

    // Whenever the state changes, let our state manager know
    state.changedListener = { mediaHostStatesManager.updateHostState(location, state) }

    updateViewVisibility()
}
```
1. 调用 `mediaHierarchyManager.register()` 创建容器 view - `UniqueObjectHostView`
2. 调用 `setListeningToMediaData()`，向 `MediaDataManager` 添加 `Listener`


#### listener
```
private val listener =
    object : MediaDataManager.Listener {
        override fun onMediaDataLoaded(
            key: String,
            oldKey: String?,
            data: MediaData,
            immediately: Boolean,
            receivedSmartspaceCardLatency: Int,
            isSsReactivated: Boolean
        ) {
            if (immediately) {
                updateViewVisibility()
            }
        }

        override fun onSmartspaceMediaDataLoaded(
            key: String,
            data: SmartspaceMediaData,
            shouldPrioritize: Boolean
        ) {
            updateViewVisibility()
        }

        override fun onMediaDataRemoved(key: String) {
            updateViewVisibility()
        }

        override fun onSmartspaceMediaDataRemoved(key: String, immediately: Boolean) {
            if (immediately) {
                updateViewVisibility()
            }
        }
    }
```

```
fun updateViewVisibility() {
    state.visible =
        if (showsOnlyActiveMedia) {
            mediaDataManager.hasActiveMediaOrRecommendation()
        } else {
            mediaDataManager.hasAnyMediaOrRecommendation()
        }
    val newVisibility = if (visible) View.VISIBLE else View.GONE
    if (newVisibility != hostView.visibility) {
        hostView.visibility = newVisibility
        visibleChangedListeners.forEach { it.invoke(visible) }
    }
}
```
`QSPanelControllerBase` 在 `onViewAttached()` 中把 listener 添加到了 `MediaHost` 的 visibleChangedListeners 里。
```java
protected void onViewAttached() {
    mQsTileRevealController = createTileRevealController();
    if (mQsTileRevealController != null) {
        mQsTileRevealController.setExpansion(mRevealExpansion);
    }

    mMediaHost.addVisibilityChangeListener(mMediaHostVisibilityListener); // here
    mView.addOnConfigurationChangedListener(mOnConfigurationChangedListener);
    mHost.addCallback(mQSHostCallback);
    setTiles();
    mLastOrientation = getResources().getConfiguration().orientation;
    mQSLogger.logOnViewAttached(mLastOrientation, mView.getDumpableTag());
    switchTileLayout(true);

    mDumpManager.registerDumpable(mView.getDumpableTag(), this);
}
```


## MediaDataManager
初始化：
```
    init {
        dumpManager.registerDumpable(TAG, this)

        // Initialize the internal processing pipeline. The listeners at the front of the pipeline
        // are set as internal listeners so that they receive events. From there, events are
        // propagated through the pipeline. The end of the pipeline is currently mediaDataFilter,
        // so it is responsible for dispatching events to external listeners. To achieve this,
        // external listeners that are registered with [MediaDataManager.addListener] are actually
        // registered as listeners to mediaDataFilter.
        addInternalListener(mediaTimeoutListener)
        addInternalListener(mediaResumeListener)
        addInternalListener(mediaSessionBasedFilter)
        mediaSessionBasedFilter.addListener(mediaDeviceManager)
        mediaSessionBasedFilter.addListener(mediaDataCombineLatest)
        mediaDeviceManager.addListener(mediaDataCombineLatest)
        mediaDataCombineLatest.addListener(mediaDataFilter)

        // Set up links back into the pipeline for listeners that need to send events upstream.
        mediaTimeoutListener.timeoutCallback = { key: String, timedOut: Boolean ->
            setTimedOut(key, timedOut)
        }
        mediaTimeoutListener.stateCallback = { key: String, state: PlaybackState ->
            updateState(key, state)
        }
        mediaTimeoutListener.sessionCallback = { key: String -> onSessionDestroyed(key) }
        mediaResumeListener.setManager(this)
        mediaDataFilter.mediaDataManager = this

        val suspendFilter = IntentFilter(Intent.ACTION_PACKAGES_SUSPENDED)
        broadcastDispatcher.registerReceiver(appChangeReceiver, suspendFilter, null, UserHandle.ALL)

        val uninstallFilter =
            IntentFilter().apply {
                addAction(Intent.ACTION_PACKAGE_REMOVED)
                addAction(Intent.ACTION_PACKAGE_RESTARTED)
                addDataScheme("package")
            }
        // BroadcastDispatcher does not allow filters with data schemes
        context.registerReceiver(appChangeReceiver, uninstallFilter)

        // Register for Smartspace data updates.
        smartspaceMediaDataProvider.registerListener(this)
        smartspaceSession =
            smartspaceManager.createSmartspaceSession(
                SmartspaceConfig.Builder(context, SMARTSPACE_UI_SURFACE_LABEL).build()
            )
        smartspaceSession?.let {
            it.addOnTargetsAvailableListener(
                // Use a main uiExecutor thread listening to Smartspace updates instead of using
                // the existing background executor.
                // SmartspaceSession has scheduled routine updates which can be unpredictable on
                // test simulators, using the backgroundExecutor makes it's hard to test the threads
                // numbers.
                uiExecutor,
                SmartspaceSession.OnTargetsAvailableListener { targets ->
                    smartspaceMediaDataProvider.onTargetsAvailable(targets)
                }
            )
        }
        smartspaceSession?.let { it.requestSmartspaceUpdate() }
        tunerService.addTunable(
            object : TunerService.Tunable {
                override fun onTuningChanged(key: String?, newValue: String?) {
                    allowMediaRecommendations = allowMediaRecommendations(context)
                    if (!allowMediaRecommendations) {
                        dismissSmartspaceRecommendation(
                            key = smartspaceMediaData.targetId,
                            delay = 0L
                        )
                    }
                }
            },
            Settings.Secure.MEDIA_CONTROLS_RECOMMENDATION
        )
    }
```
1. 调用 `addInternalListener()` 添加 listener


### 数据加载过程
```
fun onNotificationAdded(key: String, sbn: StatusBarNotification) {
    if (useQsMediaPlayer && isMediaNotification(sbn)) {
        var logEvent = false
        Assert.isMainThread()
        val oldKey = findExistingEntry(key, sbn.packageName)
        if (oldKey == null) {
            val instanceId = logger.getNewInstanceId()
            val temp = LOADING.copy(packageName = sbn.packageName, instanceId = instanceId)
            mediaEntries.put(key, temp)
            logEvent = true
        } else if (oldKey != key) {
            // Resume -> active conversion; move to new key
            val oldData = mediaEntries.remove(oldKey)!!
            logEvent = true
            mediaEntries.put(key, oldData)
        }
        loadMediaData(key, sbn, oldKey, logEvent)
    } else {
        onNotificationRemoved(key)
    }
}
```
当有新 Notification 添加时，检查 `useQsMediaPlayer` 和 `isMediaNotification()`，然后调用 `loadMediaData()`

```
private fun loadMediaData(
    key: String,
    sbn: StatusBarNotification,
    oldKey: String?,
    logEvent: Boolean = false
) {
    backgroundExecutor.execute { loadMediaDataInBg(key, sbn, oldKey, logEvent) }
}
```
backgroundExecutor 中执行 `loadMediaDataInBg()`
```
    fun loadMediaDataInBg(
        key: String,
        sbn: StatusBarNotification,
        oldKey: String?,
        logEvent: Boolean = false
    ) {
        val token = sbn.notification.extras.getParcelable(...)
        if (token == null) {
            return
        }
        val mediaController = mediaControllerFactory.create(token)
        val metadata = mediaController.metadata
        val notif: Notification = sbn.notification

        val appInfo = ...

        // Album art
        var artworkBitmap = metadata?.let { loadBitmapFromUri(it) }
        if (artworkBitmap == null) {
            artworkBitmap = metadata?.getBitmap(MediaMetadata.METADATA_KEY_ART)
        }
        if (artworkBitmap == null) {
            artworkBitmap = metadata?.getBitmap(MediaMetadata.METADATA_KEY_ALBUM_ART)
        }
        val artWorkIcon =
            if (artworkBitmap == null) {
                notif.getLargeIcon()
            } else {
                Icon.createWithBitmap(artworkBitmap)
            }

        // App name
        val appName = getAppName(sbn, appInfo)

        // App Icon
        val smallIcon = sbn.notification.smallIcon

        // Song name
        var song: CharSequence? = metadata?.getString(MediaMetadata.METADATA_KEY_DISPLAY_TITLE)
        if (song == null) {
            song = metadata?.getString(MediaMetadata.METADATA_KEY_TITLE)
        }
        if (song == null) {
            song = HybridGroupManager.resolveTitle(notif)
        }

        // Explicit Indicator
        var isExplicit = false
        if (mediaFlags.isExplicitIndicatorEnabled()) {
            val mediaMetadataCompat = MediaMetadataCompat.fromMediaMetadata(metadata)
            isExplicit =
                mediaMetadataCompat?.getLong(MediaConstants.METADATA_KEY_IS_EXPLICIT) ==
                    MediaConstants.METADATA_VALUE_ATTRIBUTE_PRESENT
        }

        // Artist name
        var artist: CharSequence? = metadata?.getString(MediaMetadata.METADATA_KEY_ARTIST)
        if (artist == null) {
            artist = HybridGroupManager.resolveText(notif)
        }

        // Device name (used for remote cast notifications)
        var device: MediaDeviceData? = null
        if (isRemoteCastNotification(sbn)) {
            ...
        }

        // Control buttons
        // If flag is enabled and controller has a PlaybackState, create actions from session info
        // Otherwise, use the notification actions
        var actionIcons: List<MediaAction> = emptyList()
        var actionsToShowCollapsed: List<Int> = emptyList()
        val semanticActions = createActionsFromState(sbn.packageName, mediaController, sbn.user)
        if (semanticActions == null) {
            val actions = createActionsFromNotification(sbn)
            actionIcons = actions.first
            actionsToShowCollapsed = actions.second
        }

        val playbackLocation =
            if (isRemoteCastNotification(sbn)) MediaData.PLAYBACK_CAST_REMOTE
            else if (
                mediaController.playbackInfo?.playbackType ==
                    MediaController.PlaybackInfo.PLAYBACK_TYPE_LOCAL
            )
                MediaData.PLAYBACK_LOCAL
            else MediaData.PLAYBACK_CAST_LOCAL
        val isPlaying = mediaController.playbackState?.let { isPlayingState(it.state) } ?: null

        val currentEntry = mediaEntries.get(key)
        val instanceId = currentEntry?.instanceId ?: logger.getNewInstanceId()
        val appUid = appInfo?.uid ?: Process.INVALID_UID

        val lastActive = systemClock.elapsedRealtime()
        foregroundExecutor.execute {
            val resumeAction: Runnable? = mediaEntries[key]?.resumeAction
            val hasCheckedForResume = mediaEntries[key]?.hasCheckedForResume == true
            val active = mediaEntries[key]?.active ?: true
            onMediaDataLoaded(
                key,
                oldKey,
                MediaData(...)
            )
        }
    }
```
构造好 `MediaData` 后，在 foregroundExecutor 中调用 `onMediaDataLoaded()`
```
fun onMediaDataLoaded(key: String, oldKey: String?, data: MediaData) =
    traceSection("MediaDataManager#onMediaDataLoaded") {
        Assert.isMainThread()
        if (mediaEntries.containsKey(key)) {
            // Otherwise this was removed already
            mediaEntries.put(key, data)
            notifyMediaDataLoaded(key, oldKey, data)
        }
    }
```

```
private fun notifyMediaDataLoaded(key: String, oldKey: String?, info: MediaData) {
    internalListeners.forEach { it.onMediaDataLoaded(key, oldKey, info) }
}
```

## MediaHierarchyManager

### mediaFrame
```
private val mediaFrame
    get() = mediaCarouselController.mediaFrame
```

#### applyState()
```
private fun applyState(
    bounds: Rect,
    alpha: Float,
    immediately: Boolean = false,
    clipBounds: Rect = EMPTY_RECT
) =
    traceSection("MediaHierarchyManager#applyState") {
        currentBounds.set(bounds)
        currentClipping = clipBounds
        carouselAlpha = if (isCurrentlyFading()) alpha else 1.0f
        val onlyUseEndState = !isCurrentlyInGuidedTransformation() || isCurrentlyFading()
        val startLocation = if (onlyUseEndState) -1 else previousLocation
        val progress = if (onlyUseEndState) 1.0f else getTransformationProgress()
        val endLocation = resolveLocationForFading()
        mediaCarouselController.setCurrentState(
            startLocation,
            endLocation,
            progress,
            immediately
        )
        updateHostAttachment()
        if (currentAttachmentLocation == IN_OVERLAY) {
            // Setting the clipping on the hierarchy of `mediaFrame` does not work
            if (!currentClipping.isEmpty) {
                currentBounds.intersect(currentClipping)
            }
            mediaFrame.setLeftTopRightBottom(
                currentBounds.left,
                currentBounds.top,
                currentBounds.right,
                currentBounds.bottom
            )
        }
    }
```
调用 `updateHostAttachment()` 把 mediaFrame 添加到 hostView 中

#### updateHostAttachment()
```
private fun updateHostAttachment() =
    traceSection("MediaHierarchyManager#updateHostAttachment") {
        var newLocation = resolveLocationForFading()
        // Don't use the overlay when fading or when we don't have active media
        var canUseOverlay = !isCurrentlyFading() && hasActiveMediaOrRecommendation
        if (isCrossFadeAnimatorRunning) {
            if (
                getHost(newLocation)?.visible == true &&
                    getHost(newLocation)?.hostView?.isShown == false &&
                    newLocation != desiredLocation
            ) {
                // We're crossfading but the view is already hidden. Let's move to the overlay
                // instead. This happens when animating to the full shade using a button click.
                canUseOverlay = true
            }
        }
        val inOverlay = isTransitionRunning() && rootOverlay != null && canUseOverlay
        newLocation = if (inOverlay) IN_OVERLAY else newLocation
        if (currentAttachmentLocation != newLocation) {
            currentAttachmentLocation = newLocation

            // Remove the carousel from the old host
            (mediaFrame.parent as ViewGroup?)?.removeView(mediaFrame)

            // Add it to the new one
            if (inOverlay) {
                rootOverlay!!.add(mediaFrame)
            } else {
                val targetHost = getHost(newLocation)!!.hostView
                // This will either do a full layout pass and remeasure, or it will bypass
                // that and directly set the mediaFrame's bounds within the premeasured host.
                targetHost.addView(mediaFrame)  // 把 mediaFrame 添加到 hostView
            }
            if (isCrossFadeAnimatorRunning) {
                // When cross-fading with an animation, we only notify the media carousel of the
                // location change, once the view is reattached to the new place and not
                // immediately
                // when the desired location changes. This callback will update the measurement
                // of the carousel, only once we've faded out at the old location and then
                // reattach
                // to fade it in at the new location.
                mediaCarouselController.onDesiredLocationChanged(
                    newLocation,
                    getHost(newLocation),
                    animate = false
                )
            }
        }
    }
```
把 mediaFrame 添加到 hostView


## MediaCarouselController

### mediaFrame
```
mediaFrame = inflateMediaCarousel()

private fun inflateMediaCarousel(): ViewGroup {
    val mediaCarousel =
        LayoutInflater.from(context)
            .inflate(R.layout.media_carousel, UniqueObjectHostView(context), false) as ViewGroup
    // Because this is inflated when not attached to the true view hierarchy, it resolves some
    // potential issues to force that the layout direction is defined by the locale
    // (rather than inherited from the parent, which would resolve to LTR when unattached).
    mediaCarousel.layoutDirection = View.LAYOUT_DIRECTION_LOCALE
    return mediaCarousel
}
```
mediaFrame 使用的布局是 `R.layout.media_carousel`

### 初始化
```
init {

    mediaFrame = inflateMediaCarousel()
    mediaCarousel = mediaFrame.requireViewById(R.id.media_carousel_scroller)
    pageIndicator = mediaFrame.requireViewById(R.id.media_page_indicator)
    mediaContent = mediaCarousel.requireViewById(R.id.media_carousel)


    mediaManager.addListener(
        object : MediaDataManager.Listener {
            override fun onMediaDataLoaded(
                key: String,
                oldKey: String?,
                data: MediaData,
                immediately: Boolean,
                receivedSmartspaceCardLatency: Int,
                isSsReactivated: Boolean
            ) {
                debugLogger.logMediaLoaded(key, data.active)
                if (addOrUpdatePlayer(key, oldKey, data, isSsReactivated)) {
                    // Log card received if a new resumable media card is added
                    MediaPlayerData.getMediaPlayer(key)?.let {
                        /* ktlint-disable max-line-length */
                        logSmartspaceCardReported(...)
                        /* ktlint-disable max-line-length */
                    }
                    if (
                        mediaCarouselScrollHandler.visibleToUser &&
                            mediaCarouselScrollHandler.visibleMediaIndex ==
                                MediaPlayerData.getMediaPlayerIndex(key)
                    ) {
                        logSmartspaceImpression(mediaCarouselScrollHandler.qsExpanded)
                    }
                } else if (receivedSmartspaceCardLatency != 0) {
                    // Log resume card received if resumable media card is reactivated and
                    // resume card is ranked first
                    MediaPlayerData.players().forEachIndexed { index, it ->
                        if (it.recommendationViewHolder == null) {
                            it.mSmartspaceId =
                                SmallHash.hash(
                                    it.mUid + systemClock.currentTimeMillis().toInt()
                                )
                            it.mIsImpressed = false
                            /* ktlint-disable max-line-length */
                            logSmartspaceCardReported(...)
                            /* ktlint-disable max-line-length */
                        }
                    }
                    // If media container area already visible to the user, log impression for
                    // reactivated card.
                    if (
                        mediaCarouselScrollHandler.visibleToUser &&
                            !mediaCarouselScrollHandler.qsExpanded
                    ) {
                        logSmartspaceImpression(mediaCarouselScrollHandler.qsExpanded)
                    }
                }

                val canRemove = data.isPlaying?.let { !it } ?: data.isClearable && !data.active
                if (canRemove && !Utils.useMediaResumption(context)) {
                    // This view isn't playing, let's remove this! This happens e.g. when
                    // dismissing/timing out a view. We still have the data around because
                    // resumption could be on, but we should save the resources and release
                    // this.
                    if (isReorderingAllowed) {
                        onMediaDataRemoved(key)
                    } else {
                        keysNeedRemoval.add(key)
                    }
                } else {
                    keysNeedRemoval.remove(key)
                }
            }

            ......

        }
    )
}
```
在 `onMediaDataLoaded()` 回调中调用 `addOrUpdatePlayer()` 添加 player


#### addOrUpdatePlayer()
```
    private fun addOrUpdatePlayer(
        key: String,
        oldKey: String?,
        data: MediaData,
        isSsReactivated: Boolean
    ): Boolean =
        traceSection("MediaCarouselController#addOrUpdatePlayer") {
            MediaPlayerData.moveIfExists(oldKey, key)
            val existingPlayer = MediaPlayerData.getMediaPlayer(key)
            val curVisibleMediaKey =
                MediaPlayerData.visiblePlayerKeys()
                    .elementAtOrNull(mediaCarouselScrollHandler.visibleMediaIndex)
            if (existingPlayer == null) {
                val newPlayer = mediaControlPanelFactory.get()
                newPlayer.attachPlayer(
                    MediaViewHolder.create(LayoutInflater.from(context), mediaContent)
                )
                newPlayer.mediaViewController.sizeChangedListener = this::updateCarouselDimensions
                val lp =
                    LinearLayout.LayoutParams(
                        ViewGroup.LayoutParams.MATCH_PARENT,
                        ViewGroup.LayoutParams.WRAP_CONTENT
                    )
                newPlayer.mediaViewHolder?.player?.setLayoutParams(lp)
                newPlayer.bindPlayer(data, key)
                newPlayer.setListening(
                    mediaCarouselScrollHandler.visibleToUser && currentlyExpanded
                )
                MediaPlayerData.addMediaPlayer(
                    key,
                    data,
                    newPlayer,
                    systemClock,
                    isSsReactivated,
                    debugLogger
                )
                updatePlayerToState(newPlayer, noAnimation = true)
                // Media data added from a recommendation card should starts playing.
                if (
                    (shouldScrollToKey && data.isPlaying == true) ||
                        (!shouldScrollToKey && data.active)
                ) {
                    reorderAllPlayers(curVisibleMediaKey, key)
                } else {
                    needsReordering = true
                }
            } else {
                existingPlayer.bindPlayer(data, key)
                MediaPlayerData.addMediaPlayer(
                    key,
                    data,
                    existingPlayer,
                    systemClock,
                    isSsReactivated,
                    debugLogger
                )
                val packageName = MediaPlayerData.smartspaceMediaData?.packageName ?: String()
                // In case of recommendations hits.
                // Check the playing status of media player and the package name.
                // To make sure we scroll to the right app's media player.
                if (
                    isReorderingAllowed ||
                        shouldScrollToKey &&
                            data.isPlaying == true &&
                            packageName == data.packageName
                ) {
                    reorderAllPlayers(curVisibleMediaKey, key)
                } else {
                    needsReordering = true
                }
            }
            updatePageIndicator()
            mediaCarouselScrollHandler.onPlayersChanged()
            mediaFrame.requiresRemeasuring = true
            return existingPlayer == null
        }
```
1. 创建 `MediaControlPanel`
2. 调用 `MediaViewHolder.create()` 创建 `MediaViewHolder`，并调用 `MediaControlPanel.attachPlayer()` 附加到 `MediaControlPanel` 中
3. 调用 `MediaControlPanel.bindPlayer()` 绑定数据


#### reorderAllPlayers
```
private fun reorderAllPlayers(
    previousVisiblePlayerKey: MediaPlayerData.MediaSortKey?,
    key: String? = null
) {
    mediaContent.removeAllViews()
    for (mediaPlayer in MediaPlayerData.players()) {
        mediaPlayer.mediaViewHolder?.let { mediaContent.addView(it.player) }
            ?: mediaPlayer.recommendationViewHolder?.let {
                mediaContent.addView(it.recommendations)
            }
    }
    mediaCarouselScrollHandler.onPlayersChanged()
    MediaPlayerData.updateVisibleMediaPlayers()
    // Automatically scroll to the active player if needed
    if (shouldScrollToKey) {
        shouldScrollToKey = false
        val mediaIndex = key?.let { MediaPlayerData.getMediaPlayerIndex(it) } ?: -1
        if (mediaIndex != -1) {
            previousVisiblePlayerKey?.let {
                val previousVisibleIndex =
                    MediaPlayerData.playerKeys().indexOfFirst { key -> it == key }
                mediaCarouselScrollHandler.scrollToPlayer(previousVisibleIndex, mediaIndex)
            }
                ?: mediaCarouselScrollHandler.scrollToPlayer(destIndex = mediaIndex)
        }
    }
}
```
调用 `mediaContent.addView(it.player)`


## MediaControlPanel


## MediaViewHolder

```
/** Holder class for media player view */
class MediaViewHolder constructor(itemView: View) {
    val player = itemView as TransitionLayout
    
    ......
```

### create()
```
@JvmStatic
fun create(inflater: LayoutInflater, parent: ViewGroup): MediaViewHolder {
    val mediaView = inflater.inflate(R.layout.media_session_view, parent, false)
    mediaView.setLayerType(View.LAYER_TYPE_HARDWARE, null)
    // Because this media view (a TransitionLayout) is used to measure and layout the views
    // in various states before being attached to its parent, we can't depend on the default
    // LAYOUT_DIRECTION_INHERIT to correctly resolve the ltr direction.
    mediaView.layoutDirection = View.LAYOUT_DIRECTION_LOCALE
    return MediaViewHolder(mediaView).apply {
        // Media playback is in the direction of tape, not time, so it stays LTR
        seekBar.layoutDirection = View.LAYOUT_DIRECTION_LTR
    }
}
```
MediaViewHolder 用的布局是 `R.layout.media_session_view`



## QSPanelControllerBase
`QSPanelControllerBase` 在 `onViewAttached()` 中把 listener 添加到了 `MediaHost` 的 visibleChangedListeners 里。
```java
private final Function1<Boolean, Unit> mMediaHostVisibilityListener = (visible) -> {
    if (mMediaVisibilityChangedListener != null) {
        mMediaVisibilityChangedListener.accept(visible);
    }
    switchTileLayout(false);
    return null;
};
```

```java
boolean switchTileLayout(boolean force) {
    /* Whether or not the panel currently contains a media player. */
    boolean horizontal = shouldUseHorizontalLayout();
    if (horizontal != mUsingHorizontalLayout || force) {
        mQSLogger.logSwitchTileLayout(horizontal, mUsingHorizontalLayout, force,
                mView.getDumpableTag());
        mUsingHorizontalLayout = horizontal;
        mView.setUsingHorizontalLayout(mUsingHorizontalLayout, mMediaHost.getHostView(), force);
        updateMediaDisappearParameters();
        if (mUsingHorizontalLayoutChangedListener != null) {
            mUsingHorizontalLayoutChangedListener.run();
        }
        return true;
    }
    return false;
}
```
调用 `setUsingHorizontalLayout()`

## QSPanel
```java
void setUsingHorizontalLayout(boolean horizontal, ViewGroup mediaHostView, boolean force) {
    if (horizontal != mUsingHorizontalLayout || force) {
        Log.d(getDumpableTag(), "setUsingHorizontalLayout: " + horizontal + ", " + force);
        mUsingHorizontalLayout = horizontal;
        ViewGroup newParent = horizontal ? mHorizontalContentContainer : this;
        switchAllContentToParent(newParent, mTileLayout);
        reAttachMediaHost(mediaHostView, horizontal);
        if (needsDynamicRowsAndColumns()) {
            mTileLayout.setMinRows(horizontal ? 2 : 1);
            mTileLayout.setMaxColumns(horizontal ? 2 : 4);
        }
        updateMargins(mediaHostView);
        mHorizontalLinearLayout.setVisibility(horizontal ? View.VISIBLE : View.GONE);
    }
}
```
调用 `reAttachMediaHost()`
```java
private void reAttachMediaHost(ViewGroup hostView, boolean horizontal) {
    if (!mUsingMediaPlayer) {
        return;
    }
    mMediaHostView = hostView;
    ViewGroup newParent = horizontal ? mHorizontalLinearLayout : this;
    ViewGroup currentParent = (ViewGroup) hostView.getParent();
    Log.d(getDumpableTag(), "Reattaching media host: " + horizontal
            + ", current " + currentParent + ", new " + newParent);
    if (currentParent != newParent) {
        if (currentParent != null) {
            currentParent.removeView(hostView);
        }
        newParent.addView(hostView);
        LinearLayout.LayoutParams layoutParams = (LayoutParams) hostView.getLayoutParams();
        layoutParams.height = ViewGroup.LayoutParams.WRAP_CONTENT;
        layoutParams.width = horizontal ? 0 : ViewGroup.LayoutParams.MATCH_PARENT;
        layoutParams.weight = horizontal ? 1f : 0;
        // Add any bottom margin, such that the total spacing is correct. This is only
        // necessary if the view isn't horizontal, since otherwise the padding is
        // carried in the parent of this view (to ensure correct vertical alignment)
        layoutParams.bottomMargin = !horizontal || displayMediaMarginsOnMedia()
                ? Math.max(mMediaTotalBottomMargin - getPaddingBottom(), 0) : 0;
        layoutParams.topMargin = mediaNeedsTopMargin() && !horizontal
                ? mMediaTopMargin : 0;
        // Call setLayoutParams explicitly to ensure that requestLayout happens
        hostView.setLayoutParams(layoutParams);
    }
}
```