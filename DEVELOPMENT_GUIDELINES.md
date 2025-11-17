# Development Guidelines - Preventing Initialization Errors

## Problem: Temporal Dead Zone (TDZ) Errors

Temporal Dead Zone errors occur when you try to access a `const` or `let` variable before it's been initialized. This is a common source of bugs in JavaScript.

## Best Practices to Prevent Initialization Errors

### 1. **Code Organization Pattern**

Always organize code in this order within functions:

```javascript
function initializeApp() {
    // 1. CONSTANTS FIRST (using var for hoisting, or const/let at top)
    var CONSTANT_NAME = { ... };
    const ANOTHER_CONSTANT = { ... };
    
    // 2. UTILITY FUNCTIONS (these can reference constants above)
    function utilityFunction() {
        // Can safely use CONSTANT_NAME here
    }
    
    // 3. STATE INITIALIZATION (can use constants and utilities)
    const state = { ... };
    
    // 4. VALIDATION (call AFTER everything is defined)
    validateInitialization();
    
    // 5. MAIN LOGIC (can use everything above)
    // ... rest of code
}
```

### 2. **Variable Declaration Strategy**

**Option A: Use `var` for constants that need hoisting**
```javascript
// var is hoisted and initialized as undefined (no TDZ)
var IMPACT_SCORE_WEIGHTS = { ... };
```

**Option B: Use `const`/`let` but define at top of scope**
```javascript
// Define all constants at the very top
const IMPACT_SCORE_WEIGHTS = { ... };
const POINTS = { ... };
const CLICK_POINTS = { ... };
```

### 3. **Validation Function Placement**

**❌ BAD: Validation before constants are defined**
```javascript
function initializeApp() {
    validateInitialization(); // ERROR: POINTS not defined yet!
    const POINTS = { ... };
}
```

**✅ GOOD: Validation after all definitions**
```javascript
function initializeApp() {
    const POINTS = { ... };
    const IMPACT_SCORE_WEIGHTS = { ... };
    // ... all other definitions
    
    validateInitialization(); // Safe: everything is defined
}
```

### 4. **Safe Validation Pattern**

Always wrap validation checks in try-catch to handle TDZ errors:

```javascript
function validateInitialization() {
    try {
        if (typeof CONSTANT_NAME === 'undefined') {
            errors.push('CONSTANT_NAME is not defined');
        }
    } catch (e) {
        // Handles TDZ errors gracefully
        errors.push('CONSTANT_NAME is not accessible');
    }
}
```

### 5. **Code Review Checklist**

Before committing, check:

- [ ] All constants defined before they're used
- [ ] Validation functions called AFTER all definitions
- [ ] No `const`/`let` variables accessed before initialization
- [ ] Try-catch blocks around validation checks
- [ ] Console logs show initialization order clearly

### 6. **Linting Rules**

Add ESLint rules to catch these issues:

```json
{
  "rules": {
    "no-use-before-define": ["error", { 
      "functions": false, 
      "classes": true, 
      "variables": true 
    }],
    "no-undef": "error"
  }
}
```

### 7. **Testing Strategy**

Add initialization tests:

```javascript
// Test that all required constants exist
test('All constants are defined', () => {
    expect(typeof IMPACT_SCORE_WEIGHTS).not.toBe('undefined');
    expect(typeof POINTS).not.toBe('undefined');
});

// Test initialization order
test('Initialization completes without errors', () => {
    expect(() => initializeApp()).not.toThrow();
});
```

### 8. **Development Workflow**

1. **Define constants first** - Always start functions with constant definitions
2. **Define functions second** - Then utility/helper functions
3. **Initialize state third** - Then state/object initialization
4. **Validate last** - Call validation after everything is defined
5. **Test immediately** - Check browser console for errors after each change

### 9. **Error Prevention Patterns**

**Pattern 1: Lazy Initialization**
```javascript
function getPoints() {
    if (typeof POINTS === 'undefined') {
        POINTS = { ... }; // Initialize on first access
    }
    return POINTS;
}
```

**Pattern 2: Default Values**
```javascript
const POINTS = window.POINTS || {
    PARTICIPATE: 100,
    // ... defaults
};
```

**Pattern 3: Module Pattern**
```javascript
(function() {
    'use strict';
    
    // All constants defined at top of IIFE
    const CONSTANTS = {
        POINTS: { ... },
        WEIGHTS: { ... }
    };
    
    // Functions can safely use CONSTANTS
    function calculate() {
        return CONSTANTS.POINTS.PARTICIPATE;
    }
    
    // Expose what's needed
    window.calculate = calculate;
})();
```

### 10. **Monitoring and Debugging**

Add initialization logging:

```javascript
function initializeApp() {
    console.log('Step 1: Defining constants...');
    const POINTS = { ... };
    
    console.log('Step 2: Defining functions...');
    function calculate() { ... }
    
    console.log('Step 3: Initializing state...');
    const state = { ... };
    
    console.log('Step 4: Validating...');
    validateInitialization();
    
    console.log('Step 5: Initialization complete');
}
```

## Quick Reference: Safe Initialization Order

```
1. Constants (var/const/let)
2. Utility Functions
3. State/Data Structures
4. Validation
5. Main Logic
6. Event Handlers
7. DOM Manipulation
```

## Common Pitfalls to Avoid

1. ❌ Calling validation before constants are defined
2. ❌ Using `const`/`let` in TDZ (accessing before declaration)
3. ❌ Circular dependencies between constants
4. ❌ Validation that throws instead of catching errors
5. ❌ No error handling around initialization

## When to Use Each Pattern

- **Use `var`**: When you need hoisting and don't mind `undefined` initial value
- **Use `const`/`let`**: When you want block scope and TDZ protection (define at top)
- **Use try-catch**: Always in validation functions
- **Use lazy init**: When constants depend on runtime conditions
- **Use module pattern**: For better encapsulation and organization




