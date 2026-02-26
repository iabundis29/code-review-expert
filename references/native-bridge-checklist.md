# Native Bridge & Native Code Checklist

## Bridge Architecture & Design

### Bridge Communication Pattern

- **No error handling across bridge**: Native errors don't propagate to JS
  ```swift
  // Bad: Errors ignored
  @objc func getData(_ resolve: @escaping RCTPromiseResolveBlock, rejecter reject: @escaping RCTPromiseRejectBlock) {
    let data = fetchData() // Might fail
    resolve(data)
  }
  
  // Good: Error handling
  @objc func getData(_ resolve: @escaping RCTPromiseResolveBlock, rejecter reject: @escaping RCTPromiseRejectBlock) {
    do {
      let data = try fetchData()
      resolve(data)
    } catch {
      reject("ERROR", error.localizedDescription, error)
    }
  }
  ```
- **Incorrect promise usage**: Not calling resolve/reject, calling both, or calling in main thread
- **Synchronous bridge calls**: Blocking main thread with native operations
  ```swift
  // Bad: Blocks main thread
  @objc func heavyComputation() -> String {
    return expensiveCalculation()
  }
  
  // Good: Async with promise
  @objc func heavyComputation(_ resolve: RCTPromiseResolveBlock, rejecter reject: RCTPromiseRejectBlock) {
    DispatchQueue.global().async {
      resolve(self.expensiveCalculation())
    }
  }
  ```
- **Missing callback cleanup**: Callbacks not cleared after use, causing memory leaks
- **Type mismatches across boundary**: JS Date/number/object not matching native expectations

### Bridge Module Registration

- **Missing module registration**: Native module not exported to JavaScript
  ```swift
  // Good: Proper RCT module registration
  @objc(MyModule)
  class MyModule: NSObject, RCTBridgeModule {
    static func moduleName() -> String! { "MyModule" }
    static func requiresMainQueueSetup() -> Bool { true }
  }
  ```
- **Inconsistent naming**: Module name in Swift doesn't match JS import
- **Not specifying `requiresMainQueueSetup`**: Defaults to false, but initialization needs main thread
- **Exposing all methods**: Methods without `@objc` shouldn't be exposed to JS

### Event Emitter Pattern

- **Memory leak from event listeners**: Not unsubscribing from native events on unmount
  ```javascript
  // Bad: Listener never removed
  useEffect(() => {
    const subscription = NativeEventEmitter.addListener('EVENT', handler)
    // Missing cleanup
  }, [])
  
  // Good: Cleanup listener
  useEffect(() => {
    const subscription = NativeEventEmitter.addListener('EVENT', handler)
    return () => subscription.remove()
  }, [])
  ```
- **Too many event emissions**: Flooding JS thread with events
- **Event data not being serialized**: Objects that can't cross bridge cause crashes
- **Unhandled event on JS side**: Event fires but no listener registered

### Questions to Ask
- "What happens if the native call fails?"
- "Could this call block the main thread?"
- "Are all listeners properly cleaned up?"

---

## Swift-Specific Issues

### Memory Management

- **Retain cycles in blocks**: Closure capturing `self` without `[weak self]`
  ```swift
  // Bad: Retain cycle
  func fetchData(completion: @escaping (Data) -> Void) {
    apiCall { data in
      self.store(data) // self captured strongly
      completion(data)
    }
  }
  
  // Good: Weak self
  func fetchData(completion: @escaping (Data) -> Void) {
    apiCall { [weak self] data in
      guard let self = self else { return }
      self.store(data)
      completion(data)
    }
  }
  ```
- **Strong self without unwrapping**: Using `self!` can crash if released
- **Not cancelling operations on deallocation**: Network requests continue after module destroyed
- **Caching without eviction**: Memory grows unbounded
- **Large objects retained**: Images, files not released after use

### Thread Safety

- **Main thread UI operations**: Updating UI from background thread
  ```swift
  // Bad: Not on main thread
  DispatchQueue.global().async {
    updateUI() // Crashes
  }
  
  // Good: Dispatch to main
  DispatchQueue.main.async {
    updateUI()
  }
  ```
- **Accessing shared state without locks**: Race conditions on mutable fields
  ```swift
  // Bad: Race condition
  var cache: [String: Data] = [:]
  func setCacheItem(_ key: String, _ value: Data) {
    cache[key] = value // Not thread-safe
  }
  
  // Good: Use queue or lock
  let cacheQueue = DispatchQueue(label: "cache")
  var cache: [String: Data] = [:]
  func setCacheItem(_ key: String, _ value: Data) {
    cacheQueue.async(flags: .barrier) {
      self.cache[key] = value
    }
  }
  ```
- **Callback called from wrong thread**: Promise resolve/reject must be main thread
- **NotificationCenter observers not removed**: Listeners leak memory

### Type Safety & Bridging

- **Force unwrapping in bridge**: `value as! String` can crash
  ```swift
  // Bad: Unsafe cast
  let name = arguments?[0] as! String
  
  // Good: Safe optional unwrap
  guard let name = arguments?[0] as? String else {
    reject("TYPE_ERROR", "Expected string", nil)
    return
  }
  ```
- **Insufficient type checking from JS**: Any type sent, not validated
- **Not converting NSNumber/NSString properly**: Type coercion issues
- **Dictionary/Array bridging mismatches**: Structure doesn't match JS expectation
  ```swift
  // Good: Explicit serialization
  func dictionaryToJSON(_ dict: [String: Any]) -> [String: Any] {
    return [
      "id": dict["id"] as? String ?? "",
      "count": dict["count"] as? NSNumber ?? 0
    ]
  }
  ```

### Error Handling

- **Swallowing errors in bridge calls**: Errors logged but not reported to JS
- **Inconsistent error codes**: Different errors, no standard format
  ```swift
  // Good: Standardized error format
  func rejectWithError(_ reject: RCTPromiseRejectBlock, code: String, message: String, error: Error? = nil) {
    reject(code, message, error)
  }
  ```
- **Not providing error metadata**: Stack trace or additional context lost
- **Timeout not set on long operations**: Promises hang indefinitely

### Questions to Ask
- "Is memory properly managed in closures?"
- "Are all operations thread-safe?"
- "Could this native call crash or hang JS?"

---

## Kotlin-Specific Issues

### Memory Management (JVM/Android)

- **Memory leaks from listeners**: Context leaked through listeners
  ```kotlin
  // Bad: context leak
  class MyModule(val context: Context) {
    init {
      context.registerReceiver(receiver, filter) // Leak if not unregistered
    }
  }
  
  // Good: Unregister in cleanup
  override fun onCatalystInstanceDestroy() {
    context.unregisterReceiver(receiver)
  }
  ```
- **Not unregistering listeners in onDestroy**: BroadcastReceiver, LocationListener leak
- **Static references to context**: Activity/context pinned in memory
- **Bitmap/drawable not recycled**: Large objects not freed
- **StringBuilder/StringBuffer in loops**: String concatenation not efficient

### Thread & Concurrency

- **UI operations off main thread**: AndroidX/Android requires main thread for UI
  ```kotlin
  // Bad: Off main thread
  thread {
    updateUI() // Crashes
  }
  
  // Good: Post to main
  Handler(Looper.getMainLooper()).post { updateUI() }
  ```
- **No synchronization on shared fields**: Race conditions in mutable state
  ```kotlin
  // Bad: Race condition
  var counter = 0
  fun increment() { counter++ }
  
  // Good: Synchronized
  private val lock = Object()
  fun increment() {
    synchronized(lock) {
      counter++
    }
  }
  ```
- **Promise callback not on main thread**: React Native expects main thread
  ```kotlin
  // Good: Main thread for promise
  Handler(Looper.getMainLooper()).post {
    promise.resolve(result)
  }
  ```

### Type Safety & Bridging

- **Unsafe casting**: `obj as String` without type check (throws)
  ```kotlin
  // Bad: Can throw ClassCastException
  val name = args[0] as String
  
  // Good: Safe cast
  val name = args.getOrNull(0) as? String
  if (name == null) {
    promise.reject("TYPE_ERROR", "Expected string")
    return
  }
  ```
- **Null safety ignored**: Using `!!` operator everywhere
  ```kotlin
  // Bad: Can NPE
  val value = obj.getSomething()!! 
  
  // Good: Null-safe
  val value = obj.getSomething()
  if (value != null) {
    // use value
  }
  ```
- **Not handling nullable returns**: SDK methods return nullable types
- **WritableMap/WritableArray type mismatches**: Setting wrong types

### Error Handling

- **Exceptions not caught in bridge**: Crashes propagate unhandled
  ```kotlin
  // Bad: Unhandled exception
  @ReactMethod
  fun getData(promise: Promise) {
    val data = fetchData()
    promise.resolve(data)
  }
  
  // Good: Exception handling
  @ReactMethod
  fun getData(promise: Promise) {
    try {
      val data = fetchData()
      promise.resolve(data)
    } catch (e: Exception) {
      promise.reject("FETCH_ERROR", e.message, e)
    }
  }
  ```
- **ANR (Application Not Responding)**: Long operations blocking main thread
- **Missing try-catch in async callbacks**: Exception crashes callback
- **Not validating input arguments**: Null or missing arguments cause crashes

### Android-Specific

- **Not checking Android version**: API not available on older versions
  ```kotlin
  // Good: Version check
  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    startForegroundService(intent)
  } else {
    startService(intent)
  }
  ```
- **SharedPreferences not committed**: Changes lost
- **Intent/Service not exported**: Restricted on Android 12+
  ```xml
  <!-- Good: Explicit exported attribute -->
  <service android:name=".MyService" android:exported="false" />
  ```
- **VectorDrawable resources not handled**: Crashes on older API levels
- **Hardware features assumed available**: Camera, GPS, etc. might not exist

### Questions to Ask
- "Are all listeners and resources properly cleaned up?"
- "Is this code thread-safe?"
- "Could this throw an uncaught exception?"
- "Is Android version compatibility checked?"

---

## Bridge Data Serialization

### Type Conversion

- **Complex objects not serializable**: Passing custom classes across bridge causes crashes
  ```swift
  // Bad: Custom object not serializable
  let customObj = MyObject()
  resolve(customObj) // Crashes
  
  // Good: Convert to dictionary
  resolve([
    "id": customObj.id,
    "name": customObj.name,
    "items": customObj.items.map { ["id": $0.id] }
  ])
  ```
- **Date/Timestamp format inconsistency**: JS and native using different formats
  ```javascript
  // Good: Use ISO8601 or timestamp
  const timestamp = Math.floor(Date.now() / 1000) // Seconds
  const iso = new Date().toISOString() // ISO8601
  ```
- **Number precision loss**: 64-bit integers truncated in JavaScript
  ```kotlin
  // Good: Send large numbers as string
  promise.resolve(WritableNativeMap().apply {
    putString("bigNumber", "9223372036854775807")
  })
  ```
- **Circular references in objects**: Can't serialize circular structures
- **Enum serialization**: Native enums not automatically mapped to JS

### Data Size & Performance

- **Sending large objects frequently**: Performance issues from bridge overhead
- **Not compressing images before sending**: Huge payloads across bridge
- **Batch operations instead of individual**: 100 calls instead of 1 batch call
  ```javascript
  // Bad: 100 bridge calls
  for (let i = 0; i < items.length; i++) {
    await sendToNative(items[i])
  }
  
  // Good: Single batch call
  await sendBatchToNative(items)
  ```
- **Downloading large files through bridge**: Should write directly to disk
- **No pagination or streaming**: All data loaded into memory at once

### Questions to Ask
- "Can this object be serialized across the bridge?"
- "What's the size of data being transferred?"
- "Could this be batched into fewer bridge calls?"

---

## Platform-Specific Bridge Issues

### iOS-Specific

- **Using deprecated APIs**: APIs removed in newer iOS versions
- **Not handling iOS 13+ changes**: Modal presentation, lifecycle changes
- **App Transport Security**: Not handling HTTPS requirements
- **Not requesting permissions**: Microphone, camera, location permissions needed
  ```swift
  // Good: Request permission first
  AVCaptureDevice.requestAccess(for: .video) { granted in
    if granted {
      startCamera()
    }
  }
  ```
- **Memory warnings not handled**: App crashes under memory pressure
- **UIKit operations not on main thread**: All UI code must be on main

### Android-Specific

- **Not handling permissions at runtime**: Permissions granted but not checked (API 23+)
  ```kotlin
  // Good: Runtime permission check
  if (ContextCompat.checkSelfPermission(context, permission) != PERMISSION_GRANTED) {
    ActivityCompat.requestPermissions(activity, permissions, REQUEST_CODE)
  }
  ```
- **Configuration changes not handled**: Activity recreated, state lost
- **Back button not handled**: Navigation stack issues
- **Battery optimization**: App killed in Doze mode without proper wake lock
- **Fragment lifecycle**: Fragment methods called before attached to activity

### Bridging Differences

- **Assuming same method availability on both platforms**: iOS has method, Android doesn't
  ```javascript
  // Good: Check platform availability
  if (Platform.OS === 'ios') {
    iosSpecificMethod()
  } else if (Platform.OS === 'android') {
    androidSpecificMethod()
  }
  ```
- **Behavior differences not documented**: Same method behaves differently on each platform
- **No fallback for missing native module**: Native module not built, causes crash

### Questions to Ask
- "Does this work on all supported iOS/Android versions?"
- "Are platform-specific permissions checked?"
- "Is there a fallback if native module missing?"

---

## Testing & Validation

### Unit Testing Native Code

- **Native code not tested**: Bugs only caught in production
- **Bridge contracts not validated**: JS and native expectations misaligned
- **Edge cases not covered**: Empty arrays, null values, large inputs
- **Mock objects not matching real behavior**: Tests pass, production fails

### Integration Testing

- **Bridge calls not mocked in JS tests**: Real native calls in unit tests
  ```javascript
  // Good: Mock native bridge
  jest.mock('@react-native', () => ({
    NativeModules: {
      MyModule: {
        getData: jest.fn().mockResolvedValue({ data: 'test' })
      }
    }
  }))
  ```
- **No E2E testing of bridge**: Bridge only tested manually
- **Not testing platform differences**: Different behavior on iOS vs Android not caught
- **Error scenarios not tested**: Happy path only, error handling untested

### Runtime Validation

- **Type validation only on one side**: JS strict, native loose (or vice versa)
- **No logging of bridge calls**: Difficult to debug bridge issues
- **Errors not logged with context**: Stack trace not helpful
- **No performance monitoring**: Slow bridge calls not detected
  ```swift
  // Good: Log timing
  let start = CFAbsoluteTimeGetCurrent()
  let result = expensiveOperation()
  let time = CFAbsoluteTimeGetCurrent() - start
  NSLog("Operation took %.2f seconds", time)
  ```

### Questions to Ask
- "Is this bridge call tested on both platforms?"
- "What happens if native code throws an error?"
- "Are bridge call timings monitored?"

---

## Common Bridge Crashes

- **Promise resolved/rejected twice**: `Promise already settled` error
- **Accessing Java objects from Kotlin**: Incorrect syntax for Java interop
- **Swift closure capture issues**: Memory leaks or early deallocation
- **Missing `@ReactMethod` annotation**: Method not exposed to JS
- **Dictionary key type mismatch**: Java Map expects String keys, gets other types
- **Module not deinitialized properly**: Native resources leaked

### Questions to Ask
- "Will this cause a crash if called rapidly?"
- "Are all error paths handled?"
- "Is the module properly cleaned up?"
