## Bug #1 — Infinite Re-render Loop

**File:** src/pages/ProductSearch.jsx  
**Line(s):** useEffect dependency array

### Expected Behaviour
The search effect should run only when the `query` changes and fetch matching products once.

### Actual Behaviour
The effect kept re-running continuously, causing repeated API calls and unnecessary re-renders. In DevTools Network tab, multiple identical requests were triggered.

### Root Cause
The `results` state was included in the `useEffect` dependency array. Since `setResults()` updates `results`, React detected a dependency change and re-ran the effect, causing an infinite loop.

This violates React’s **Effect Dependency Rule**: only values that should trigger the effect should be included in the dependency array.

### Fix Applied
Removed `results` from the dependency array and kept only `query`.

**Before**
```jsx
useEffect(() => {
  ...
}, [query, results]);
```

**After**
```jsx
useEffect(() => {
  ...
}, [query]);
```

---

## Bug #2 — Race Condition / Stale Search Results

**File:** src/pages/ProductSearch.jsx  
**Line(s):** inside `useEffect`

### Expected Behaviour
Only the latest search request should update the results, and unnecessary API calls should be avoided while typing.

### Actual Behaviour
Every keystroke triggered a new API request. In DevTools Network tab, multiple requests fired rapidly. Sometimes an older slow request resolved after a newer one and overwrote the latest search results with stale data.

### Root Cause
There was no debounce mechanism and no cleanup function inside `useEffect`.

This violates React’s **Effect Cleanup Rule**: asynchronous side effects should be cleaned up to prevent outdated updates.

### Fix Applied
Added:
- `setTimeout()` for 300ms debounce
- `clearTimeout()` in cleanup
- `ignore` flag to prevent stale responses from updating state

**Before**
```jsx
searchProducts(query).then((data) => {
  setResults(data);
  setLoading(false);
});
```

**After**
```jsx
let ignore = false;

const timer = setTimeout(() => {
  searchProducts(query).then((data) => {
    if (!ignore) {
      setResults(data);
      setLoading(false);
    }
  });
}, 300);

return () => {
  ignore = true;
  clearTimeout(timer);
};
```

---

## Bug #3 — Direct State Mutation

**File:** src/pages/OrderManager.jsx  
**Line(s):** `handleStatusChange()`

### Expected Behaviour
When an order status is changed, the badge color and status text should update immediately in the UI.

### Actual Behaviour
The API call succeeded, but the status badge did not update correctly because React did not detect the state change.

### Root Cause
The original code directly mutated the existing order object (`order.status = newStatus`) instead of creating a new object.

This violates React’s **State Immutability Rule**: state must never be mutated directly.

### Fix Applied
Used `map()` to create a new array and object spread (`...order`) to create a new updated object.

**Before**
```jsx
const updatedOrders = orders;
const order = updatedOrders.find((o) => o.id === orderId);

if (order) {
  order.status = newStatus;
}

setOrders(updatedOrders);
```

**After**
```jsx
const updatedOrders = orders.map((order) =>
  order.id === orderId
    ? { ...order, status: newStatus }
    : order
);

setOrders(updatedOrders);
```


## DEPLOYED LINK
https://shopwave-debug-fix-jet.vercel.app/