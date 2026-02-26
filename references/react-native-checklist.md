# React Native Checklist

## Lifecycle & Memory Management

### Component Lifecycle Issues

- **Memory leaks in useEffect**: Missing cleanup function that unsubscribes, removes listeners, or clears timers
  ```javascript
  // Bad: Memory leak
  useEffect(() => {
    const listener = EventEmitter.on('event', handler)
    // Missing removal
  }, [])
  
  // Good: Cleanup
  useEffect(() => {
    const listener = EventEmitter.on('event', handler)
    return () => listener.remove()
  }, [])
  ```
- **Unhandled promise rejections**: Async operations that don't handle errors
- **Mounting operations on unmounted components**: Setting state after component unmounts
  ```javascript
  // Bad: Can crash unmounted component
  useEffect(() => {
    fetch(url).then(data => setState(data))
  }, [])
  
  // Good: Check if mounted or use AbortController
  useEffect(() => {
    let isMounted = true
    fetch(url).then(data => isMounted && setState(data))
    return () => { isMounted = false }
  }, [])
  ```
- **Missing effect dependencies**: Stale closures, infinite loops, or missed updates
- **Class component lifecycle misuse**: Async operations in constructor, missing `_unsubscribe` cleanup in `componentWillUnmount`

### Questions to Ask
- "What cleanup does this component need when unmounting?"
- "Can this component be unmounted while a promise is pending?"
- "Are all dependencies of this effect listed?"

---

## Performance & Optimization

### List Rendering (FlatList/SectionList)

- **No `keyExtractor` or dynamic keys**: Using array index as key, causing re-renders and state issues
  ```javascript
  // Bad: Index key
  <FlatList data={items} renderItem={item => ...} />
  
  // Good: Stable key
  <FlatList data={items} renderItem={item => ...} keyExtractor={item => item.id} />
  ```
- **Rendering large lists without virtualization**: All items rendered at once
- **Missing `initialNumToRender`/`maxToRenderPerBatch`**: Not tuned for performance
- **Complex renderItem without memo**: Re-rendering on every parent state change
- **No separator or dynamic separator logic**: Rendering extra views
- **`removeClippedSubviews` not set**: On Android, large lists should clip off-screen items

### Component Memoization

- **Missing React.memo for list items**: Child component re-renders unnecessarily
  ```javascript
  const ListItem = React.memo(({ item, onPress }) => (
    <TouchableOpacity onPress={onPress}>
      <Text>{item.name}</Text>
    </TouchableOpacity>
  ))
  ```
- **Memo with object/function props**: Props recreated on every render, bypassing memo
  ```javascript
  // Bad: Function created every render
  <MemoChild onPress={() => handlePress(id)} />
  
  // Good: Memoized callback
  const handlePress = useCallback(() => handlePress(id), [id])
  ```
- **useMemo/useCallback used incorrectly**: Not worth the cost or wrong dependencies

### Threading & Native Bridge

- **Expensive operations on main thread**: Heavy computation blocks UI
  ```javascript
  // Bad: Blocks UI
  const result = expensiveCalculation(largeData)
  
  // Good: Use InteractionManager or native module
  InteractionManager.runAfterInteractions(() => {
    expensiveCalculation(largeData)
  })
  ```
- **Too many bridge calls**: Batching not used, frequent JS-to-Native hops
- **Synchronous bridge calls**: `RCTBridgeModule` blocking main thread
- **Missing NativeModule error handling**: Crashes on exception from native code

### Questions to Ask
- "How does this perform with 1000+ items?"
- "Are child components re-rendering unnecessarily?"
- "Can this network/computation be deferred or moved to native?"

---

## Platform-Specific Code

### iOS/Android Differences

- **No platform check for API availability**: Using `react-native` API that doesn't exist on both platforms
  ```javascript
  // Bad: May not exist on iOS/Android
  NativeModules.CameraModule.startCamera()
  
  // Good: Check platform and have fallback
  if (Platform.OS === 'ios') {
    // iOS-specific code
  } else {
    // Android-specific code
  }
  ```
- **Platform-specific bugs not handled**: Different behavior on native platform (TouchID, Fingerprint)
- **Missing `Platform.select()`**: Duplicated conditions instead of elegant selection
  ```javascript
  // Better style
  const value = Platform.select({
    ios: 'iOS value',
    android: 'Android value',
    default: 'fallback'
  })
  ```
- **Hardcoded colors/sizes** without considering platform defaults (safe area, status bar height)
- **No testing on both platforms**: Same code behaves differently on iOS vs Android

### Safe Area & Screen Bounds

- **Ignoring safe area**: Content hidden under notch or gesture areas
  ```javascript
  // Bad: Ignores safe area
  <View style={{ flex: 1 }}>...</View>
  
  // Good: Use SafeAreaView or SafeAreaContext
  <SafeAreaView edges={['top']} style={{ flex: 1 }}>...</SafeAreaView>
  ```
- **Hardcoded status bar height or notch offset**: Use `useSafeAreaInsets()`
- **Missing `translucent` status bar handling**: Overlapping content

### Questions to Ask
- "Does this code work on both iOS and Android?"
- "Is safe area properly handled?"
- "Are platform-specific edge cases tested?"

---

## Navigation

### Navigation State & Stack Issues

- **Missing key in navigation params**: Screens share state unexpectedly
- **Not resetting navigation stack**: User can navigate back to unexpected screens
  ```javascript
  // Good practice when login
  navigation.reset({
    index: 0,
    routes: [{ name: 'Home' }]
  })
  ```
- **Accessing route params without fallback**: params might be undefined
- **Navigation listeners not cleaned up**: Memory leaks from unremoved listeners
- **Deeply nested navigation**: Complex navigation graphs difficult to reason about

### Deep Linking

- **No deep link handling**: URLs don't work
- **Insufficient link validation**: Malformed or malicious links crash app
- **Not handling link state restoration**: Deep link lost on app restart
- **Hardcoded host/path in linking config**: Not flexible enough

### Questions to Ask
- "What data persists across navigation?"
- "What happens on app restart from a deep link?"
- "Are navigation listeners cleaned up?"

---

## Storage & Persistence

### AsyncStorage Limits

- **Exceeding AsyncStorage limits**: Large objects or too many keys (size varies by platform)
- **Blocking main thread with large reads**: Use `getMultiple()` instead of multiple `getItem()`
- **Not handling AsyncStorage full**: No fallback when storage quota exceeded
- **Mixing async/await and `.then()`**: Promise handling inconsistency

### Data Persistence Issues

- **No error boundary for corrupted storage**: Crash on parse error
  ```javascript
  // Good: Handle parse errors
  try {
    const data = JSON.parse(await AsyncStorage.getItem('key'))
  } catch (e) {
    // Handle corrupted data
  }
  ```
- **Not versioning stored data**: Migration issues on schema changes
- **Storing sensitive data in AsyncStorage**: Use Keychain/Keystore instead
- **No TTL or refresh logic**: Stale data served indefinitely

### Questions to Ask
- "What's the maximum data size that can be stored?"
- "How is data migration handled on schema change?"
- "Is sensitive data properly encrypted?"

---

## Touch & User Interactions

### Gesture & Input Handling

- **Not preventing double-tap or rapid taps**: Submit button pressed multiple times
  ```javascript
  // Good: Debounce or disable
  const [isSubmitting, setIsSubmitting] = useState(false)
  const handlePress = useCallback(async () => {
    if (isSubmitting) return
    setIsSubmitting(true)
    try {
      await submitForm()
    } finally {
      setIsSubmitting(false)
    }
  }, [isSubmitting])
  ```
- **Missing `delayPressIn`/`delayPressOut`**: Unresponsive tap feedback
- **No accessibility for gestures**: Swiping or long-press not accessible
- **Missing `hitSlop`**: Buttons too small to tap accurately
- **Pan responder not released**: Touch gestures stick or become unresponsive

### Animations & Performance

- **Animations on main thread**: Janky UI (should use `useNativeDriver: true`)
  ```javascript
  // Good: Native driver for smooth animations
  Animated.timing(animValue, {
    toValue: 1,
    duration: 300,
    useNativeDriver: true  // Important!
  }).start()
  ```
- **Complex animated re-renders**: Calculated styles not memoized
- **Missing animation cleanup**: Animated values don't stop on unmount

### Questions to Ask
- "Can the user accidentally trigger this action twice?"
- "Is this animation smooth at 60fps?"
- "Is the touch target large enough (min 44x44 iOS, 48x48 Android)?"

---

## Image Handling

### Image Performance

- **Full-size images loaded**: Not resized, causes memory issues and slow scrolls
- **Missing image dimension specification**: Layout shift or tall/wide images
  ```javascript
  // Good: Specify dimensions
  <Image
    source={{ uri: 'https://...' }}
    style={{ width: 200, height: 200 }}
  />
  ```
- **No image caching strategy**: Re-downloading same images repeatedly
- **Synchronous image operations**: Blocking main thread
- **Missing placeholder or loading state**: Poor UX while loading

### Network Images

- **Unhandled image loading errors**: Broken images with no fallback
- **Not terminating image requests on unmount**: Memory leaks
- **Large image count in same view**: App crashes from OOM
- **No timeout for image requests**: Stuck loading indicators

### Questions to Ask
- "What size are these images and can they be optimized?"
- "Are images cached properly?"
- "What happens if image fails to load?"

---

## Common Crashes

- **FlatList with non-array data**: `Unsupported top level type for JSON: java.util.ArrayList`
- **setState on unmounted component**: Warning and memory leak
- **Missing `nativeID` for native ref access**: Crashes when accessing native module
- **JSON serialization of circular objects**: Crash when stringifying Redux state
- **Missing exception handling in promises**: Unhandled promise rejection crashes
- **Incompatible native modules/dependencies**: Version mismatch crashes

### Questions to Ask
- "Will this code crash if X fails?"
- "Is proper error handling throughout?"
