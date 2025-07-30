# @japa/runner - Technical Reference Guide

A comprehensive reference for backend development and testing with @japa/runner.

## 1. Basic Installation and Setup

### Requirements
> **Important:** Japa requires `Node.js >= 18` and works only with the **ES module system**.

### Installation
Use the `create-japa` initializer to set up Japa in an existing Node.js project:

```bash
# npm
npm init japa@latest .

# yarn
yarn create japa@latest .

# pnpm
pnpm create japa@latest .
```

The initializer will:
- Install `@japa/runner` as a development dependency
- Prompt you to select an assertion library
- Prompt you to select and install additional plugins
- Create `bin/test(.js|.ts)` entrypoint file

### Basic Configuration
Create your test entry point file `bin/test.js` or `bin/test.ts`:

```ts
import { assert } from '@japa/assert'
import { apiClient } from '@japa/api-client'
import { expectTypeOf } from '@japa/expect-type'
import { configure, processCLIArgs, run } from '@japa/runner'

// Process CLI arguments
processCLIArgs(process.argv.splice(2))

// Configure Japa
configure({
  files: ['tests/**/*.spec.js'],
  plugins: [
    assert(),
    expectTypeOf(),
  ],
})

// Run tests
run()
```

**Key Methods:**
- `processCLIArgs`: Processes command line arguments and tweaks configuration based on CLI flags
- `configure`: Configures Japa by registering plugins, reporters, defining test files, etc.
- `run`: Executes tests based on the applied configuration and exits the Node.js process

## 2. Structure of a Test File

### Basic Test Structure
Tests must be written inside the `tests` directory with `.spec.js` or `.spec.ts` extension:

```ts
// tests/example.spec.js
import { test } from '@japa/runner'

function sum(a, b) {
  return a + b
}

test('add two numbers', ({ assert }) => {
  assert.equal(sum(2, 2), 4)
})
```

### Test with Groups
Wrap tests inside groups for better organization and lifecycle management:

```ts
import { test } from '@japa/runner'

function sum(a, b) {
  return a + b
}

test.group('Maths.add', () => {
  test('add two numbers', ({ assert }) => {
    assert.equal(sum(2, 2), 4)
  })
})
```

### Assertion Libraries
Choose from multiple assertion libraries:

```ts
// With Chai assert
test('add two numbers', ({ assert }) => {
  assert.equal(2 + 2, 4)
})

// With Jest expect
test('add two numbers', ({ expect }) => {
  expect(2 + 2).toEqual(4)
})

// Asserting Types
test('find user by id', async ({ expectTypeOf }) => {
  const user = await User.find(1)
  expectTypeOf(user).toMatchTypeOf<User | null>()
})
```

## 3. Commands to Run Tests

### Basic Test Execution
```bash
node bin/test.js
```

### CLI Help
View available options:
```bash
node bin/test.js --help
```

### Running Specific Suites
```bash
# Run specific suite
node bin/test.js unit

# Run multiple suites
node bin/test.js functional unit
```

### Common CLI Flags
- `--help`: Show help information
- `--groups`: Filter tests by group names
- `--files`: Specify test files to run
- `--bail`: Stop on first failure

## 4. Use of Groups and Hooks

### Test Groups
Groups allow you to organize tests and define shared lifecycle hooks:

```ts
test.group('Users', (group) => {
  // Group configuration here
  
  test('create user', () => {
    // Test implementation
  })
  
  test('update user', () => {
    // Test implementation
  })
})
```

### Individual Test Hooks
```ts
test('add two numbers', () => {
  console.log('executed in the test')
})
.setup(() => {
  console.log('executed before the test')
})
.teardown(() => {
  console.log('executed after the test')
})
```

### Group Lifecycle Hooks

#### Each Test Hooks
Run before/after each test in the group:

```ts
test.group('Maths.add', (group) => {
  group.each.setup(() => {
    console.log('executed before each test')
  })

  group.each.teardown(() => {
    console.log('executed after each test')
  })

  test('add two numbers', () => {
    // Test implementation
  })
})
```

#### Group-Level Hooks
Run before/after all tests in the group:

```ts
test.group('Maths.add', (group) => {
  group.setup(() => {
    console.log('executed before all tests')
  })

  group.teardown(() => {
    console.log('executed after all tests')
  })
  
  // Tests here...
})
```

### Cleanup Functions
> **Best Practice:** Always use cleanup functions to destroy state created by setup hooks.

```ts
test.group('Users.create', (group) => {
  group.each.setup(async () => {
    await createTables()
    // Return cleanup function
    return async () => await dropTables()
  })

  test('create a new user', () => {
    // Test implementation
  })
})
```

**Why cleanup functions over teardown hooks?**
- Cleanup functions are tied to their specific setup hook
- If a setup hook fails, its cleanup won't run (avoiding errors)
- Teardown hooks run independently and can fail if setup didn't complete

### Hook Parameters
```ts
test.group((group) => {
  // Test hooks receive test instance
  group.each.setup((test) => {
    // Setup logic
  })

  // Cleanup functions receive error state and test instance
  group.each.setup(() => {
    return (hasError, test) => {
      // Cleanup logic
    }
  })

  // Group hooks receive group instance
  group.setup((self) => {
    console.log(self === group) // true
  })
})
```

## 5. Parallel vs Sequential Execution

### Default Behavior
> **Important:** Japa does **not** run tests in parallel by default. Tests run sequentially.

### Running Suites in Parallel
You can run different test suites in parallel using the `concurrently` package:

#### Installation
```bash
npm i -D concurrently
```

#### Package.json Configuration
```json
{
  "scripts": {
    "unit:tests": "node bin/test.js unit",
    "functional:tests": "node bin/test.js functional",
    "test": "concurrently \"npm:unit:tests\" \"npm:functional:tests\""
  }
}
```

#### Execution
```bash
npm test
```

### Test Suite Configuration
Organize tests by type using suites:

```ts
import { configure } from '@japa/runner'

configure({
  suites: [
    {
      name: 'unit',
      files: ['tests/unit/**/*.spec.js'],
    },
    {
      name: 'functional',
      files: ['tests/functional/**/*.spec.js'],
      configure(suite) {
        suite.setup(() => {
          const server = startHttpServer()
          return () => server.close()
        })
      }
    }
  ]
})
```

## 6. Test Coverage Integration

### Using c8 (Recommended)
```bash
# Installation
npm i -D c8

# Package.json
{
  "scripts": {
    "test": "c8 node bin/test.js"
  }
}

# Run with coverage
npm test
```

### Using nyc
```bash
# Installation
npm i -D nyc

# Package.json
{
  "scripts": {
    "test": "nyc node bin/test.js"
  }
}

# Run with coverage
npm test
```

> **Note:** Consult the documentation of c8 or nyc for advanced configuration options.

## 7. Database Testing Best Practices

While @japa/runner doesn't provide specific database testing utilities, you can implement robust database testing using lifecycle hooks:

### Database Setup and Cleanup Pattern
```ts
test.group('User Database Tests', (group) => {
  group.each.setup(async () => {
    // Start database transaction
    await db.beginTransaction()
    
    // Run migrations if needed
    await runMigrations()
    
    // Cleanup function to rollback transaction
    return async () => {
      await db.rollbackTransaction()
    }
  })

  test('create user', async ({ assert }) => {
    const user = await User.create({
      email: 'test@example.com',
      name: 'Test User'
    })
    
    assert.exists(user.id)
    assert.equal(user.email, 'test@example.com')
  })
})
```

### Suite-Level Database Setup
```ts
configure({
  suites: [
    {
      name: 'database',
      files: ['tests/database/**/*.spec.js'],
      configure(suite) {
        suite.setup(async () => {
          // Setup test database
          await setupTestDatabase()
          
          return async () => {
            // Cleanup test database
            await cleanupTestDatabase()
          }
        })
      }
    }
  ]
})
```

### Testing Database Constraints
```ts
test('prevent duplicate emails', async ({ assert }) => {
  await User.create({ email: 'test@example.com' })
  
  await assert.rejects(
    async () => User.create({ email: 'test@example.com' }),
    /Unique constraint/
  )
})
```

## 8. CI/CD Integration

### Basic GitHub Actions Example
```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [18, 20]
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Run tests with coverage
        run: npm run test:coverage
```

### Package.json Scripts for CI
```json
{
  "scripts": {
    "test": "node bin/test.js",
    "test:coverage": "c8 node bin/test.js",
    "test:unit": "node bin/test.js unit",
    "test:functional": "node bin/test.js functional"
  }
}
```

### Environment-Specific Configuration
```ts
// bin/test.js
import { configure } from '@japa/runner'

const isCI = process.env.CI === 'true'

configure({
  files: ['tests/**/*.spec.js'],
  plugins: [
    assert(),
  ],
  // Use different reporters for CI
  reporters: isCI ? ['dot'] : ['spec'],
  // Bail on first failure in CI
  bail: isCI,
})
```

## 9. Common Errors and Troubleshooting

### Exception Handling Patterns

#### Problems with try/catch
❌ **Avoid this pattern:**
```ts
test('validate email format', ({ assert }) => {
  try {
    validateEmail('foo')
  } catch (error) {
    assert.equal(error.message, '"foo" is not a valid email address')
  }
  // Problem: Test passes even if no exception is thrown!
})
```

#### Using Assertion Methods
✅ **Use dedicated assertion methods:**

**For synchronous functions:**
```ts
test('validate email format', ({ assert }) => {
  assert.throws(
    () => validateEmail('foo'),
    '"foo" is not a valid email address'
  )
})
```

**For asynchronous functions:**
```ts
test('do not insert duplicate emails', async ({ assert }) => {
  await createUser({ email: 'foo@bar.com' })
  
  await assert.rejects(
    async () => createUser({ email: 'foo@bar.com' }),
    /Unique constraint/
  )
})
```

#### Using expect syntax
```ts
// Synchronous
test('validate email format', ({ expect }) => {
  expect(() => validateEmail('foo'))
    .toThrow('"foo" is not a valid email address')
})

// Asynchronous
test('do not insert duplicate emails', async ({ expect }) => {
  await createUser({ email: 'foo@bar.com' })
  
  await expect(createUser({ email: 'foo@bar.com' }))
    .rejects
    .toThrow(/Unique constraint/)
})
```

#### High-Order Assertions
✅ **Cleanest approach:**
```ts
test('validate email format', () => {
  validateEmail('foo')
})
.throws('"foo" is not a valid email address')

test('do not insert duplicate emails', async () => {
  await createUser({ email: 'foo@bar.com' })
  await createUser({ email: 'foo@bar.com' }) // Will throw
})
.throws(/Unique constraint/)
```

### Common Issues and Solutions

#### ES Module Issues
**Problem:** `SyntaxError: Cannot use import statement outside a module`

**Solution:** Ensure your `package.json` has:
```json
{
  "type": "module"
}
```

#### Node.js Version
**Problem:** Compatibility issues

**Solution:** Ensure you're using Node.js >= 18

#### File Extensions
**Problem:** Tests not being discovered

**Solution:** Ensure test files end with `.spec.js` or `.spec.ts` and match the configured pattern:
```ts
configure({
  files: ['tests/**/*.spec.js'], // Adjust pattern as needed
})
```

#### Hook Cleanup Issues
**Problem:** Resources not being cleaned up properly

**Solution:** Always use cleanup functions instead of teardown hooks:
```ts
// ✅ Good
group.each.setup(() => {
  const resource = createResource()
  return () => resource.cleanup()
})

// ❌ Avoid
group.each.setup(() => {
  createResource()
})
group.each.teardown(() => {
  // This might fail if setup failed
  resource.cleanup()
})
```

---

## Additional Resources

- **VSCode Extension:** Install the [official Japa extension](https://marketplace.visualstudio.com/items?itemName=jripouteau.japa-vscode) to run tests directly from your editor
- **Plugins:** Explore additional plugins for API testing, browser testing, file system testing, and more
- **Documentation:** Refer to the complete documentation for advanced configuration options

> **Tip:** Start with simple test structures and gradually add complexity as your testing needs grow. Focus on clear test names, proper cleanup, and consistent patterns across your test suite.
