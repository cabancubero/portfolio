# Frontend - Portfolio Breakdown

## Context: Demo Application Architecture

This frontend is a **demo-ready mobile application** built to showcase a fully functional civic engagement platform without requiring a live backend. The architecture was designed around this constraint: every feature - event registration, community voting, messaging, organization matching - needs to work on any device, at any time, without a running server.

Rather than building a thin prototype, the approach was to build production-grade architecture that happens to run on local data. The service layer, state management, and component structure are all designed so that switching from demo mode to a live API is a configuration change (`USE_MOCK_DATA: false`), not a rewrite. The mock services simulate network latency, persist data across sessions via AsyncStorage, and handle the same CRUD operations and error cases a real API would. This means every loading spinner, error state, and data flow in the app represents real behavior - not a shortcut.

---

## Skills Demonstrated

| Skill | Where It Appears |
|-------|-------------------|
| **JavaScript (ES6+)** | Destructuring, spread operators, async/await, template literals, arrow functions, computed property names throughout |
| **React Native / Expo** | Mobile app built with Expo SDK, Expo Router for file-based navigation, platform-specific configuration |
| **Zustand** | State management via factory-created stores with persistence middleware and AsyncStorage |
| **React Context API** | `SearchContext`, `FilterContext`, and `MessageProvider` for cross-component shared state |
| **Custom Hooks** | 10+ hooks encapsulating data fetching, message handling, onboarding steps, and store access |
| **React Native Animated API** | Multi-layered animations (fill, pulse, flame) with easing curves, looped sequences, and interpolation |
| **Factory Pattern** | `createDataStore`, `createMockService`, `createRegistrationStore`, and service factories for events, communities, etc. |
| **AsyncStorage** | Persistent local storage powering the demo environment - mock data, user state, and Zustand store hydration |
| **Accessibility** | ARIA roles, labels, and hints on interactive components; `accessibilityElementsHidden` on decorative elements |

---

## Architectural Decisions

### 1. Service Factory Pattern: One Toggle Between Demo and Production
Every data operation flows through a service factory (e.g., `createEventsService`, `createCommunitiesService`). Each factory checks `API_CONFIG.USE_MOCK_DATA`. When `true`, it builds a mock service using `createMockService` - a generic CRUD layer backed by AsyncStorage with simulated network delay. When `false`, it returns stubs that call real API endpoints. The stores and hooks above the service layer don't know or care which mode is active.

This means the entire demo environment is created by architecture, not by faking it. The mock services simulate latency (`delay: 300ms`), persist data between app launches, handle filtering and search, and throw real errors on missing items. Every loading state, optimistic update, and error handler in the UI was developed against this layer and will work identically when pointed at a live API.

### 2. Triple-Layer Data Architecture (Service → Store → Hook)
The data layer is three composable factory layers:
- **Service factories** produce CRUD operations (mock or real)
- **`createDataStore`** wraps any service into a Zustand store with loading/error states
- **Domain hooks** (e.g., `useCommunitiesData`) add feature-specific methods on top

Each layer is independently testable and swappable. The store doesn't know if the service is mock or real. The hook doesn't know how the store manages state internally. This separation made it possible to build the entire frontend before the backend was ready.

### 3. Feature-Based Component Organization
Components are organized into three tiers: `ui/` (pure, reusable components with no business logic), `layout/` (app-wide structural components), and `features/` (feature-specific components organized by domain - events, messages, communities, etc.). This prevents the common problem of a flat components directory becoming unnavigable, and makes it clear which components are shared vs. feature-specific.

---

## Code Snippets (Core Competencies)

### 1. Mock Service Base - The Demo Environment Engine
**Skills:** AsyncStorage, factory pattern, generic CRUD, parameter-based filtering, multi-field search

This is the foundation of the demo environment. A single factory produces a complete data layer for any entity type - events, communities, messages, organizations. It handles storage reads/writes, simulates network latency so the UI develops realistic loading behavior, supports parameter-based filtering (both scalar and array values), and provides multi-field text search. Every service factory in the app extends this base.

```javascript
export const createMockService = (options) => {
  const {
    storageKey,
    defaultData = [],
    delay = 300,
    idGenerator = () => Date.now().toString()
  } = options;

  const simulateDelay = () =>
    new Promise(resolve => setTimeout(resolve, delay));

  const getData = async () => {
    try {
      const data = await AsyncStorage.getItem(storageKey);
      return data ? JSON.parse(data) : defaultData;
    } catch (error) {
      return defaultData;
    }
  };

  return {
    getAll: async (params = {}) => {
      await simulateDelay();
      let data = await getData();

      if (params && Object.keys(params).length > 0) {
        data = data.filter(item => {
          return Object.entries(params).every(([key, value]) => {
            if (Array.isArray(value)) return value.includes(item[key]);
            return item[key] === value;
          });
        });
      }
      return data;
    },

    create: async (itemData) => {
      await simulateDelay();
      const data = await getData();
      const newItem = {
        ...itemData,
        id: itemData.id || idGenerator(),
        createdAt: new Date().toISOString(),
        updatedAt: new Date().toISOString()
      };
      await AsyncStorage.setItem(storageKey, JSON.stringify([...data, newItem]));
      return newItem;
    },

    search: async (query, fields = ['name', 'description']) => {
      await simulateDelay();
      const data = await getData();
      if (!query) return data;

      const normalized = query.toLowerCase();
      return data.filter(item =>
        fields.some(field => {
          const value = item[field];
          return value && value.toString().toLowerCase().includes(normalized);
        })
      );
    }
  };
};
```

### 2. Domain Service Factory - Composing Mock Services for Complex Operations
**Skills:** Service composition, multi-collection coordination, factory pattern

Each domain gets its own service factory that composes multiple mock services together. The events factory, for example, coordinates three independent storage collections (events, attended, hosted) to handle operations like registration - which requires updating the attendee count on the event *and* adding the ID to the attended list. This mirrors the kind of multi-table coordination a real API would perform.

```javascript
export const createEventsService = () => {
  if (API_CONFIG.USE_MOCK_DATA) {
    const baseService = createMockService({
      storageKey: API_CONFIG.STORAGE_KEYS.EVENTS,
      defaultData: eventsData, delay: 300
    });
    const attendedService = createMockService({
      storageKey: API_CONFIG.STORAGE_KEYS.ATTENDED_EVENTS,
      defaultData: [], delay: 200
    });
    const hostedService = createMockService({
      storageKey: API_CONFIG.STORAGE_KEYS.HOSTED_EVENTS,
      defaultData: [], delay: 200
    });

    return {
      registerForEvent: async (id) => {
        const event = await baseService.getById(id);
        const attendedEvents = await attendedService.getAll();

        if (attendedEvents.includes(id)) {
          return { alreadyRegistered: true, event };
        }

        await attendedService.create(id);
        await baseService.update(id, {
          attendees: (event.attendees || 0) + 1
        });

        return { success: true, event };
      },

      getTrendingEvents: async (limit = 5) => {
        const events = await baseService.getAll();
        return events
          .sort((a, b) => (b.attendees || 0) - (a.attendees || 0))
          .slice(0, limit);
      },

      searchEvents: async (query) => {
        return baseService.search(query, ['title', 'description', 'location', 'category']);
      }
    };
  } else {
    // Real API implementation - same method signatures, different internals
    return {
      registerForEvent: async (id) => { /* fetch call */ },
      getTrendingEvents: async (limit) => { /* fetch call */ },
      searchEvents: async (query) => { /* fetch call */ }
    };
  }
};
```

### 3. Zustand Store Factory - Generic State Management with CRUD
**Skills:** Zustand, factory pattern, generic state management, error handling

A single factory function produces fully functional Zustand stores with CRUD operations, loading/error state tracking, and extension points for custom methods. Every data domain in the app uses this same factory, eliminating boilerplate. The `methods` parameter lets each store inject domain-specific logic (voting, saving, commenting) alongside the shared operations.

```javascript
export const createDataStore = (options) => {
  const {
    initialState = [],
    service = {},
    methods = () => ({}),
  } = options;

  return create((set, get) => ({
    data: initialState,
    isLoading: false,
    error: null,
    lastFetched: null,

    fetchAll: async (params) => {
      set({ isLoading: true, error: null });
      try {
        const data = await service.getAll(params);
        set({
          data,
          isLoading: false,
          lastFetched: new Date().toISOString()
        });
        return data;
      } catch (error) {
        set({
          error: error.message || 'Failed to fetch data',
          isLoading: false
        });
        return [];
      }
    },

    update: async (id, itemData) => {
      set({ isLoading: true, error: null });
      try {
        const updatedItem = await service.update(id, itemData);
        set(state => ({
          data: state.data.map(item => item.id === id ? updatedItem : item),
          isLoading: false
        }));
        return updatedItem;
      } catch (error) {
        set({
          error: error.message || `Failed to update item with id ${id}`,
          isLoading: false
        });
        return null;
      }
    },

    ...methods(set, get)
  }));
};
```

### 4. MatchIndicator - Multi-Layered Animation Component
**Skills:** React Native Animated API, `Easing` curves, looped sequences, interpolation, accessibility

A circular match indicator with three independent animation layers: a fill that eases in over 1200ms, a pulse loop for high matches (>80%), and a flame animation whose intensity and speed scale with the match percentage. The 100% match case gets a special 4-step flickering sequence. Colors shift from blue (low) through orange (high) based on the percentage.

```javascript
const MatchIndicator = ({ matchPercent = 0, size = 'medium', animated = true, clickable = false, onPress }) => {
  const normalizedPercent = Math.max(0, Math.min(100, matchPercent));

  const fillAnimation = useRef(new Animated.Value(0)).current;
  const pulseAnimation = useRef(new Animated.Value(1)).current;
  const flameAnimation = useRef(new Animated.Value(1)).current;

  useEffect(() => {
    if (animated) {
      Animated.timing(fillAnimation, {
        toValue: normalizedPercent / 100,
        duration: 1200,
        easing: Easing.out(Easing.cubic),
        useNativeDriver: false
      }).start();

      if (normalizedPercent > 80) {
        Animated.loop(
          Animated.sequence([
            Animated.timing(pulseAnimation, {
              toValue: 1.1, duration: 800,
              easing: Easing.inOut(Easing.ease), useNativeDriver: false
            }),
            Animated.timing(pulseAnimation, {
              toValue: 1, duration: 800,
              easing: Easing.inOut(Easing.ease), useNativeDriver: false
            })
          ])
        ).start();
      }

      const flameIntensity = 0.15 + (normalizedPercent / 100) * 0.35;

      if (normalizedPercent >= 98) {
        Animated.loop(
          Animated.sequence([
            Animated.timing(flameAnimation, { toValue: 1.5, duration: 400, easing: Easing.out(Easing.sin), useNativeDriver: false }),
            Animated.timing(flameAnimation, { toValue: 1.2, duration: 300, easing: Easing.in(Easing.sin), useNativeDriver: false }),
            Animated.timing(flameAnimation, { toValue: 1.35, duration: 250, easing: Easing.out(Easing.sin), useNativeDriver: false }),
            Animated.timing(flameAnimation, { toValue: 1.15, duration: 350, easing: Easing.in(Easing.sin), useNativeDriver: false })
          ])
        ).start();
      } else {
        Animated.loop(
          Animated.sequence([
            Animated.timing(flameAnimation, {
              toValue: 1 + flameIntensity,
              duration: 600 - (normalizedPercent * 2),
              easing: Easing.out(Easing.sin), useNativeDriver: false
            }),
            Animated.timing(flameAnimation, {
              toValue: 1 - (flameIntensity * 0.5),
              duration: 700 - (normalizedPercent * 2),
              easing: Easing.in(Easing.sin), useNativeDriver: false
            })
          ])
        ).start();
      }
    }
  }, [normalizedPercent, animated]);

  const WrapperComponent = clickable ? TouchableOpacity : View;

  return (
    <WrapperComponent
      {...(clickable ? {
        onPress,
        accessibilityRole: "button",
        accessibilityLabel: `View match details for ${normalizedPercent}% match`
      } : {
        accessibilityRole: "image",
        accessibilityLabel: `Match indicator: ${normalizedPercent}% match`
      })}
    >
      <Animated.View style={{ transform: [{ scale: pulseAnimation }] }}>
        <Animated.View style={{ width: fillAnimation.interpolate({
          inputRange: [0, 1], outputRange: ['0%', '100%']
        })}} />
        <LinearGradient colors={getColors()}>
          <Animated.View style={{ transform: [{ scale: flameAnimation }] }}>
            <Ionicons name="flame" size={getFlameIconSize()} color="white" />
          </Animated.View>
        </LinearGradient>
      </Animated.View>
    </WrapperComponent>
  );
};
```

### 5. Search Context - Debounced Global Search with History
**Skills:** React Context API, `useCallback` memoization, debouncing with refs, cleanup on unmount

A search provider that wraps any part of the app with debounced search, search history (deduplicated, capped at 10), and a generic `filterItems` function that accepts any filter predicate. The debounce uses a ref-tracked timeout rather than a library, keeping the dependency footprint small.

```javascript
export const SearchProvider = ({ children, initialQuery = '', onSearch = null, debounceTime = 300 }) => {
  const [searchQuery, setSearchQuery] = useState(initialQuery);
  const [searchHistory, setSearchHistory] = useState([]);
  const timeoutRef = useRef(null);

  const handleSearchQueryChange = useCallback((query) => {
    setSearchQuery(query);

    if (timeoutRef.current) clearTimeout(timeoutRef.current);

    if (onSearch) {
      timeoutRef.current = setTimeout(() => {
        onSearch(query);
      }, debounceTime);
    }
  }, [onSearch, debounceTime]);

  const addToSearchHistory = useCallback((term) => {
    if (!term || term.trim() === '') return;

    setSearchHistory(prevHistory => {
      const filteredHistory = prevHistory.filter(item => item !== term);
      return [term, ...filteredHistory].slice(0, 10);
    });
  }, []);

  const filterItems = useCallback((items, filterFn) => {
    if (!searchQuery || searchQuery.trim() === '') return items;
    return items.filter(item => filterFn(item, searchQuery));
  }, [searchQuery]);

  useEffect(() => {
    return () => {
      if (timeoutRef.current) clearTimeout(timeoutRef.current);
    };
  }, []);

  return (
    <SearchContext.Provider value={{
      searchQuery, setSearchQuery: handleSearchQueryChange,
      searchHistory, addToSearchHistory, filterItems
    }}>
      {children}
    </SearchContext.Provider>
  );
};
```

---

## Clever Implementations

### 1. Conversation Load Guard with Ref Tracking
The `useMessageHandler` hook uses two refs - `hasLoadedRef` and `currentConversationIdRef` - to prevent redundant conversation loads. When `loadConversation` is called, it checks whether the ID has changed and whether the current conversation is already loaded before doing any work. It also includes a 5-second safety timeout that forces loading to complete if something stalls, preventing the UI from being stuck in a loading state indefinitely.

The ref-based tracking solves a real React problem: `useEffect` dependencies can cause loops when state changes trigger re-renders that trigger the effect again. Refs break the cycle without suppressing the effect.

```javascript
const useMessageHandler = (conversationId = null) => {
  const hasLoadedRef = useRef(false);
  const currentConversationIdRef = useRef(conversationId);

  const loadConversation = useCallback(async (id) => {
    if (!id) { setIsLoading(false); return; }

    // Skip if same conversation already loaded
    if (id === currentConversationIdRef.current && hasLoadedRef.current) {
      setIsLoading(false);
      return;
    }

    setIsLoading(true);

    if (id !== currentConversationIdRef.current) {
      hasLoadedRef.current = false;
    }
    currentConversationIdRef.current = id;

    try {
      const conversationId = String(id);
      const conversation = conversations.find(c => String(c.id) === conversationId);
      if (!conversation) throw new Error(`Conversation not found: ${conversationId}`);

      const conversationMessages = getConversationMessages(conversationId);
      setMessages(conversationMessages.map(msg => ({
        ...msg,
        formattedTime: msg.formattedTime || formatMessageTime(msg.timestamp)
      })));

      markConversationAsRead(conversationId);
      hasLoadedRef.current = true;
      setIsLoading(false);
    } catch (err) {
      setError(err.message);
      setIsLoading(false);
    }
  }, [getConversationMessages, conversations, markConversationAsRead]);

  // Safety timeout prevents infinite loading states
  useEffect(() => {
    if (conversationId) {
      const loadingTimeout = setTimeout(() => {
        if (isLoading) setIsLoading(false);
      }, 5000);

      loadConversation(conversationId);
      return () => clearTimeout(loadingTimeout);
    }
  }, [conversationId, loadConversation]);
};
```

### 2. Deterministic Match Percentages with Entity Chaining
The `generateMatchPercentage` utility produces consistent match percentages for the demo using `(idNumber * 17) % 101`. The same entity always shows the same match score across sessions and screens. But the function also chains entity relationships: for events, it looks up the hosting organization and recursively calls itself with the org's ID, so all events from the same organization display the same match percentage. This makes the demo feel cohesive - navigating from an organization to its events shows consistent matching - even though no real algorithm is running.

```javascript
export const generateMatchPercentage = (entityId, options = {}) => {
  const { entityType = '', useRandom = false, organizerName = null } = options;

  // Events inherit their match score from the hosting organization
  if (entityType === 'event' && organizerName) {
    const { organizationsData } = require('../data/organizationsData');
    const organization = organizationsData.find(org => org.name === organizerName);
    if (organization) {
      return generateMatchPercentage(organization.id, { entityType: 'organization' });
    }
  }

  let idNumber;
  if (typeof entityId === 'string') {
    idNumber = entityId.split('').reduce((sum, char) => sum + char.charCodeAt(0), 0);
  } else {
    idNumber = Number(entityId);
  }

  // Deterministic: same ID always produces same match
  let baseMatch = (idNumber * 17) % 101;

  // Entity-type adjustments for realistic demo variety
  if (entityType === 'post') baseMatch = Math.min(100, baseMatch + 5);
  if (entityType === 'resource') baseMatch = Math.min(100, baseMatch + 7);

  if (useRandom) {
    const variation = Math.floor(Math.random() * 10) - 5;
    baseMatch = Math.max(0, Math.min(100, baseMatch + variation));
  }

  return baseMatch;
};
```
