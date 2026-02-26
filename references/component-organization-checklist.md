# Component Organization & Size Checklist

## Component Size & Complexity

### Large Component Anti-patterns

- **Component file exceeds 300+ lines**: Likely doing too much
  ```typescript
  // Bad: All logic in one component (~400 lines)
  export const UserProfileScreen = () => {
    // Layout logic
    // Data fetching
    // Form handling
    // Validation
    // Analytics tracking
    // Error display
    // Loading states
    // Caching logic
    // Permission checks
    // Navigation
    return (
      <View>
        {/* JSX for all of above */}
      </View>
    )
  }
  
  // Good: Broken down into smaller components
  export const UserProfileScreen = () => {
    return (
      <View>
        <UserHeader />
        <UserForm />
        <UserActions />
      </View>
    )
  }
  ```
- **Multiple responsibilities in single component**: Layout + fetching + validation + business logic
- **Deeply nested JSX**: Makes component hard to read and test
  ```typescript
  // Bad: Impossible to parse visually
  return (
    <View>
      <ScrollView>
        <View>
          <View>
            <FlatList
              data={data}
              renderItem={({ item }) => (
                <View>
                  <View>
                    {/* Too nested */}
                  </View>
                </View>
              )}
            />
          </View>
        </View>
      </ScrollView>
    </View>
  )
  ```
- **Too many useState hooks**: More than 5-7 state variables suggests component needs splitting
  ```typescript
  // Bad: Too much local state
  const [user, setUser] = useState(null)
  const [posts, setPosts] = useState([])
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState(null)
  const [formData, setFormData] = useState({})
  const [isEditing, setIsEditing] = useState(false)
  const [selectedPost, setSelectedPost] = useState(null)
  const [sortBy, setSortBy] = useState('date')
  const [filterBy, setFilterBy] = useState('all')
  
  // Good: Separate concerns into smaller components
  const UserSection = () => {
    const [user, setUser] = useState(null)
    // ...
  }
  const PostsSection = () => {
    const [posts, setPosts] = useState([])
    // ...
  }
  ```
- **Complex conditional rendering**: Multiple `{condition && <Component />}` or nested ternaries
  ```typescript
  // Bad: Conditional rendering scattered everywhere
  return (
    <View>
      {isLoading ? (
        <Spinner />
      ) : error ? (
        <ErrorMessage />
      ) : data ? (
        <View>
          {data.items.map(item => (
            <View key={item.id}>
              {item.type === 'A' ? (
                <TypeAComponent />
              ) : item.type === 'B' ? (
                <TypeBComponent />
              ) : (
                <TypeCComponent />
              )}
            </View>
          ))}
        </View>
      ) : (
        <EmptyState />
      )}
    </View>
  )
  
  // Good: Extract into helper or separate component
  const renderContent = () => {
    if (isLoading) return <Spinner />
    if (error) return <ErrorMessage />
    if (!data) return <EmptyState />
    return <DataDisplay data={data} />
  }
  return <View>{renderContent()}</View>
  ```

### Questions to Ask
- "Does this component have a single, clear responsibility?"
- "How many different reasons might this component change?"
- "Can I understand this component at a glance?"

---

## Component Responsibilities

### Separating Concerns

- **Data fetching + UI rendering**: Business logic leaking into presentation
  ```typescript
  // Bad: Fetching in presentation component
  export const UserCard = ({ userId }) => {
    const [user, setUser] = useState(null)
    useEffect(() => {
      fetch(`/users/${userId}`)
        .then(r => r.json())
        .then(data => setUser(data))
    }, [userId])
    return <View>{user?.name}</View>
  }
  
  // Good: Separate container component
  export const UserCardContainer = ({ userId }) => {
    const user = useUserData(userId)
    return <UserCard user={user} />
  }
  
  export const UserCard = ({ user }) => {
    return <View>{user?.name}</View>
  }
  ```
- **Business logic mixed with UI logic**: Validation, transforms, calculations in component
  ```typescript
  // Better: Extract business logic
  const useUserValidation = (formData) => {
    return useMemo(() => ({
      isValid: validateEmail(formData.email),
      errors: getValidationErrors(formData)
    }), [formData])
  }
  ```
- **Multiple unrelated features in one component**: Auth + Profile + Settings together
- **Navigation logic in component**: Component shouldn't know about navigation routes
  ```typescript
  // Bad: Component owns navigation
  const goToDetails = () => navigation.navigate('Details', { id })
  
  // Good: Pass callback
  <Card onPress={() => onSelectItem(item)} />
  ```
- **Styling and layout tightly coupled**: Style calculation in component
  ```typescript
  // Better: Use StyleSheet or component
  const styles = StyleSheet.create({
    container: { flex: 1, padding: 16 }
  })
  ```

### Questions to Ask
- "Is this component responsible for its own data?"
- "Does this component know too much about unrelated features?"
- "Can this component be used without the rest of the app?"

---

## JSX & Render Structure

### Readable Rendering

- **Render method too long**: More than 80-100 lines of JSX
- **Unformatted JSX**: Hard to see structure at a glance
  ```typescript
  // Bad: Hard to follow
  return <View style={{flex: 1, padding: 10, backgroundColor: 'white'}}><Text style={{fontSize: 16, fontWeight: 'bold'}}>{title}</Text><FlatList data={items} renderItem={({item}) => <View style={{marginBottom: 8}}><Text>{item.name}</Text></View>} keyExtractor={item => item.id} /></View>
  
  // Good: Clear structure
  return (
    <View style={styles.container}>
      <Text style={styles.title}>{title}</Text>
      <ItemsList items={items} />
    </View>
  )
  ```
- **No component extraction for repeated patterns**: Same JSX repeated multiple times
  ```typescript
  // Bad: Duplication
  return (
    <View>
      <View style={styles.card}>
        <Text>{item1.title}</Text>
        <Text>{item1.description}</Text>
      </View>
      <View style={styles.card}>
        <Text>{item2.title}</Text>
        <Text>{item2.description}</Text>
      </View>
    </View>
  )
  
  // Good: Extract component
  const CardList = ({ items }) => (
    <View>
      {items.map(item => <Card key={item.id} item={item} />)}
    </View>
  )
  ```
- **Business logic in `render` or JSX**: Calculations, array operations in return
  ```typescript
  // Bad: Logic in JSX
  return (
    <FlatList
      data={items.filter(i => i.active).sort((a, b) => b.date - a.date).slice(0, 10)}
      renderItem={...}
    />
  )
  
  // Good: Pre-compute
  const visibleItems = useMemo(() =>
    items
      .filter(i => i.active)
      .sort((a, b) => b.date - a.date)
      .slice(0, 10),
    [items]
  )
  ```

### Questions to Ask
- "Can I understand this JSX structure in 10 seconds?"
- "Are there repeated patterns I should extract?"
- "Is non-rendering logic mixed in the return statement?"

---

## Hook Patterns & Effects

### Effect Organization

- **Multiple independent effects stuffed into one**: `useEffect` doing unrelated things
  ```typescript
  // Bad: Multiple concerns in one effect
  useEffect(() => {
    // Fetch data
    loadUser()
    // Track event
    trackPageView()
    // Setup listener
    attachListener()
    // Clear cache
    clearOldCache()
  }, [])
  
  // Good: Separate effects by concern
  useEffect(() => { loadUser() }, [])
  useEffect(() => { trackPageView() }, [])
  useEffect(() => { attachListener() }, [])
  useEffect(() => { clearOldCache() }, [])
  ```
- **Effect does too much in cleanup**: Cleanup function handling multiple teardowns
- **Complex cleanup logic**: Should be own function/hook
  ```typescript
  // Good: Extract cleanup logic
  const useListener = (handler) => {
    useEffect(() => {
      const listener = EventEmitter.on('event', handler)
      return () => listener.remove()
    }, [handler])
  }
  ```
- **Missing dependency array**: Effect runs on every render
- **Over-specified dependencies**: Including more than necessary
  ```typescript
  // Bad: userId is already in formData
  useEffect(() => {
    saveForm(formData)
  }, [formData, userId, email, name])
  
  // Good: Just formData
  useEffect(() => {
    saveForm(formData)
  }, [formData])
  ```

### Hook Best Practices

- **Creating custom hooks inside component**: Hook definition buried in component
  ```typescript
  // Better: Extract custom hook to separate file
  const useUserData = (userId) => {
    const [user, setUser] = useState(null)
    useEffect(() => { /* fetch */ }, [userId])
    return user
  }
  
  // Then in component
  const MyComponent = ({ userId }) => {
    const user = useUserData(userId)
  }
  ```
- **Hooks with unclear dependencies**: Hard to understand when effect runs
- **Not using useMemo for expensive computations**: Recomputes every render
- **useCallback overused**: Every function wrapped, defeats purpose
  ```typescript
  // Okay: Only memo callbacks that are dependencies
  const handlePress = useCallback(() => {
    onSelect(item)
  }, [item, onSelect])
  ```

### Questions to Ask
- "Does this effect do one thing or multiple unrelated things?"
- "Are all dependencies listed?"
- "Should this be extracted into a custom hook?"

---

## Code Extraction Strategies

### When to Extract Components

- **Component can be used independently**: Render and work in isolation
  ```typescript
  // Good: Can test independently
  <Button title="Save" onPress={handleSave} isLoading={isLoading} />
  ```
- **Component has clear inputs and outputs**: Props and callbacks well-defined
- **Component is reused in multiple places**: Shouldn't be duplicated
- **Component is hard to understand**: Breaking down increases clarity
- **Testing is difficult**: Large component needs multiple mocks/setup

### Composition Patterns

- **Container + Presentational**: Data logic separated from UI
  ```typescript
  // Container: Logic
  export const UserProfileContainer = ({ userId }) => {
    const user = useUserData(userId)
    const handleUpdate = useUpdateUser()
    return <UserProfile user={user} onUpdate={handleUpdate} />
  }
  
  // Presentational: Display
  export const UserProfile = ({ user, onUpdate }) => {
    return (
      <View>
        <UserInfo user={user} />
        <EditButton onPress={onUpdate} />
      </View>
    )
  }
  ```
- **Higher-Order Components (HOC)**: Wrapping components for shared logic
  ```typescript
  const withLoading = (Component) => ({ isLoading, ...props }) =>
    isLoading ? <Spinner /> : <Component {...props} />
  ```
- **Render Props**: Pass function as prop of how to render
  ```typescript
  <UserData userId={userId}>
    {(user) => <UserCard user={user} />}
  </UserData>
  ```
- **Custom Hooks**: Most modern approach for logic reuse
  ```typescript
  const user = useUserData(userId)
  const { handleUpdate, isLoading } = useUpdateUser()
  ```

### Component File Structure

- **Large file with many components**: Should be split into separate files
  ```
  // Bad: All in one file
  components/User.tsx (500+ lines)
    - UserProfile
    - UserForm
    - UserHeader
    - UserCard
    - UserList
    - UserActions
  
  // Good: Organized by concern
  components/
    UserProfile/
      index.tsx (container)
      UserProfile.tsx (presentational)
      useUserData.ts (hook)
    UserForm/
      index.tsx
      UserForm.tsx
    UserHeader/
      index.tsx
      UserHeader.tsx
  ```

### Questions to Ask
- "Can this component be tested independently?"
- "Would someone unfamiliar with the code understand this component's purpose?"
- "Am I repeating this component structure elsewhere?"

---

## Performance & Re-renders

### Component Memoization

- **Child components re-render unnecessarily**: No memo, parent updates cause full re-render
  ```typescript
  // Bad: Header re-renders every time parent updates
  const UserProfile = ({ userId }) => {
    const [selectedPost, setSelectedPost] = useState(null)
    return (
      <View>
        <UserHeader userId={userId} /> {/* Re-renders on selectedPost change */}
        <PostsList userId={userId} onSelect={setSelectedPost} />
      </View>
    )
  }
  
  // Good: Memoize non-dependent children
  const UserHeader = React.memo(({ userId }) => ...)
  const PostsList = React.memo(({ userId, onSelect }) => ...)
  ```
- **Props recreated every render**: Objects/arrays/functions recreated
  ```typescript
  // Bad: New object every render
  <UserForm initialValues={{ name: user.name }} />
  
  // Good: Memoize
  const initialValues = useMemo(() => ({ name: user.name }), [user.name])
  ```

### List Performance

- **Non-optimized FlatList rendering large lists**: No memo on renderItem
  ```typescript
  // Better: Memoized list item
  const ListItem = React.memo(({ item, onPress }) => ...)
  
  <FlatList
    data={items}
    renderItem={({ item }) => <ListItem item={item} />}
    keyExtractor={item => item.id}
  />
  ```
- **Missing keys on list items**: React can't track identity
- **Complex list item rendering**: Should be separate component

### Questions to Ask
- "Will this component re-render when parent's unrelated state changes?"
- "Are props creating new object references every render?"
- "Could this list be optimized with keys or memo?"

---

## Testing & Maintainability

### Testability Issues

- **Component too large to test thoroughly**: Need to mock too many dependencies
- **Too many implicit dependencies**: Hard to set up test fixtures
- **Logic tightly coupled to UI**: Can't test business logic separately
  ```typescript
  // Better: Separate logic for testing
  export const calculateTotal = (items) => item.reduce((sum, i) => sum + i.price, 0)
  
  export const CartSummary = ({ items }) => {
    const total = calculateTotal(items)
    return <Text>{total}</Text>
  }
  ```
- **No clear props interface**: Hard to know what to pass
  ```typescript
  // Good: Clear, documented props
  interface UserCardProps {
    user: User
    onPress?: () => void
    style?: StyleProp<ViewStyle>
  }
  ```

### Maintainability

- **Component name doesn't match responsibility**: `UserContainer` but it's primarily a form
- **No JSDoc or comments**: Purpose unclear
  ```typescript
  /**
   * Displays user profile and handles editing
   * @param userId - The ID of the user to display
   * @param readonly - If true, disables editing
   */
  export const UserProfile = ({ userId, readonly = false }) => { ... }
  ```
- **Magic numbers and strings scattered**: Should be constants
  ```typescript
  // Bad: Magic number
  if (items.length > 50) { /* pagination */ }
  
  // Good: Named constant
  const MAX_ITEMS_PER_PAGE = 50
  ```

### Questions to Ask
- "Can I test this component without mocking the entire app?"
- "Is it obvious what this component does from its name?"
- "If someone new joined the team, could they understand this component?"

---

## Common Large Component Pattern to Break Down

```typescript
// ❌ Bad: 350+ lines, multiple responsibilities
export const UserProfileScreen = ({ route, navigation }) => {
  const userId = route.params.userId
  
  // Data fetching
  const [user, setUser] = useState(null)
  const [posts, setPosts] = useState([])
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState(null)
  
  // Form state
  const [isEditing, setIsEditing] = useState(false)
  const [formData, setFormData] = useState({})
  const [validationErrors, setValidationErrors] = useState({})
  
  // More state...
  const [selectedPost, setSelectedPost] = useState(null)
  const [showOptions, setShowOptions] = useState(false)
  
  useEffect(() => { /* load user */ }, [userId])
  useEffect(() => { /* load posts */ }, [userId])
  useEffect(() => { /* subscribe to updates */ }, [userId])
  useEffect(() => { /* analytics */ }, [userId])
  
  const handleFormChange = (field, value) => { /* ... */ }
  const handleValidation = () => { /* ... */ }
  const handleSubmit = async () => { /* ... */ }
  const handleDelete = async () => { /* ... */ }
  const handleShare = () => { /* ... */ }
  
  return (
    <ScrollView>
      {loading && <Spinner />}
      {error && <ErrorBanner error={error} />}
      {!loading && !error && (
        <View>
          {/* Profile section */}
          {/* Form section */}
          {/* Posts list */}
          {/* Options menu */}
          {/* All JSX here... */}
        </View>
      )}
    </ScrollView>
  )
}

// ✅ Good: Broken into focused components
export const UserProfileScreen = ({ route, navigation }) => {
  const userId = route.params.userId
  return (
    <ScrollView>
      <UserProfileHeader userId={userId} />
      <UserForm userId={userId} />
      <UserPostsList userId={userId} />
    </ScrollView>
  )
}

// Each component is focused and testable
const UserProfileHeader = ({ userId }) => {
  const user = useUserData(userId)
  return user ? <Header user={user} /> : <Spinner />
}

const UserForm = ({ userId }) => {
  const [formData, setFormData] = useState({})
  const { handleSubmit } = useUserUpdate(userId)
  // Form logic only
  return <Form {...} />
}

const UserPostsList = ({ userId }) => {
  const posts = useUserPosts(userId)
  // List logic only
  return <FlatList data={posts} {...} />
}
```

---

## Size Guidelines & Metrics

### Component Size Recommendations

| Size | Lines | Assessment | Action |
|------|-------|-----------|--------|
| **Small** | < 100 | Single responsibility, focused | ✓ Good |
| **Medium** | 100-200 | Likely okay, review closely | Consider splitting |
| **Large** | 200-300 | Multiple responsibilities likely | Split into smaller |
| **XLarge** | 300+ | Definitely too large | Must refactor |

### Complexity Metrics
- **Too many useState hooks**: > 7 hooks = likely too many states
- **Too many useEffect hooks**: > 5 effects = likely unrelated concerns
- **Nesting depth**: > 4 levels deep = JSX hard to understand
- **File size**: > 400 lines = time to split

### Questions to Ask
- "How long would it take a new dev to understand this component?"
- "How many different reasons could this component need to change?"
- "Would this component be hard to test?"
