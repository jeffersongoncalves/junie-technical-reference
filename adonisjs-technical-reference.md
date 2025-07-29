# AdonisJS v6 Technical Reference

A comprehensive technical reference for AdonisJS backend development and testing, extracted from the official documentation.

## Table of Contents

1. [Project Structure and Organization](#project-structure-and-organization)
2. [Routing System](#routing-system)
3. [Controllers and Validation](#controllers-and-validation)
4. [Lucid ORM](#lucid-orm)
5. [Authentication System](#authentication-system)
6. [Middleware, Exception Handling, and Lifecycle](#middleware-exception-handling-and-lifecycle)
7. [WebSockets and HTTP Server](#websockets-and-http-server)
8. [Testing with Japa](#testing-with-japa)
9. [Environment Configuration](#environment-configuration)
10. [IoC Container and Dependency Injection](#ioc-container-and-dependency-injection)

---

## Project Structure and Organization

### Technical Summary
AdonisJS follows a thoughtful default folder structure that promotes organization and maintainability. The framework uses Node.js sub-path imports for clean import aliases and provides multiple entry points for different environments.

### Key Directory Structure
```
├── app/                    # Domain logic (controllers, models, services)
├── bin/                    # Entry point files
├── config/                 # Configuration files
├── database/               # Migrations and seeders
├── providers/              # Service providers
├── resources/              # Frontend assets and views
├── start/                  # Boot lifecycle files
├── tests/                  # Test files
├── tmp/                    # Temporary files
└── types/                  # TypeScript type definitions
```

### Import Aliases (package.json)
```json
{
  "imports": {
    "#controllers/*": "./app/controllers/*.js",
    "#models/*": "./app/models/*.js",
    "#services/*": "./app/services/*.js",
    "#middleware/*": "./app/middleware/*.js",
    "#validators/*": "./app/validators/*.js",
    "#config/*": "./config/*.js",
    "#database/*": "./database/*.js",
    "#tests/*": "./tests/*.js"
  }
}
```

### Entry Points
- `bin/server.ts` - HTTP server environment
- `bin/console.ts` - CLI commands environment  
- `bin/test.ts` - Testing environment

### Best Practices
- Use import aliases for cleaner imports
- Organize code by domain in the `app` directory
- Keep configuration separate in `config` directory
- Use `start` directory for boot lifecycle files

---

## Routing System

### Technical Summary
AdonisJS provides a powerful routing system with support for route parameters, middleware, groups, and URL generation. Routes are defined in `start/routes.ts` and support various HTTP methods.

### Basic Route Definition
```ts
import router from '@adonisjs/core/services/router'

// Basic routes
router.get('/', () => 'Hello world')
router.post('/users', () => {})
router.put('/users/:id', () => {})
router.delete('/users/:id', () => {})

// Route with controller
const UsersController = () => import('#controllers/users_controller')
router.get('/users', [UsersController, 'index'])
```

### Route Parameters
```ts
// Required parameter
router.get('/posts/:id', ({ params }) => {
  return `Post ID: ${params.id}`
})

// Optional parameter
router.get('/posts/:id?', ({ params }) => {
  return params.id ? `Post ${params.id}` : 'All posts'
})

// Wildcard parameter
router.get('/docs/:category/*', ({ params }) => {
  console.log(params.category)    // string
  console.log(params['*'])        // array
})
```

### Parameter Validation
```ts
// Custom matcher
router
  .get('/posts/:id', ({ params }) => {})
  .where('id', {
    match: /^[0-9]+$/,
    cast: (value) => Number(value)
  })

// Built-in matchers
router.where('id', router.matchers.number())
router.where('uuid', router.matchers.uuid())
router.where('slug', router.matchers.slug())
```

### Route Groups
```ts
router.group(() => {
  router.get('/users', () => {})
  router.post('/users', () => {})
})
.prefix('/api')
.as('api')
.use(middleware.auth())
```

### Resource Routes
```ts
// Creates 7 RESTful routes
router.resource('posts', PostsController)

// API only (excludes create/edit forms)
router.resource('posts', PostsController).apiOnly()

// Specific routes only
router.resource('posts', PostsController).only(['index', 'store', 'show'])
```

### URL Generation
```ts
// Using route builder
const url = router.builder()
  .params({ id: 1 })
  .make('posts.show')  // /posts/1

// With query parameters
const urlWithQuery = router.builder()
  .qs({ page: 1, limit: 10 })
  .make('posts.index')  // /posts?page=1&limit=10

// Signed URLs
const signedUrl = router.builder()
  .params({ id: 1 })
  .makeSigned('posts.show', { expiresIn: '1 hour' })
```

### Best Practices
- Use route names for URL generation
- Group related routes together
- Apply middleware at group level when possible
- Use resource routes for RESTful APIs
- Validate route parameters with matchers

---

## Controllers and Validation

### Technical Summary
Controllers organize route handlers into dedicated classes and support dependency injection. Validation is handled using VineJS with compile-time schema validation and runtime type safety.

### Basic Controller
```ts
// Create controller: node ace make:controller users
export default class UsersController {
  index() {
    return [{ id: 1, username: 'virk' }]
  }
}
```

### Controller with Dependency Injection
```ts
import { inject } from '@adonisjs/core'
import UserService from '#services/user_service'

@inject()
export default class UsersController {
  constructor(private userService: UserService) {}

  index() {
    return this.userService.all()
  }
}
```

### Validation with VineJS
```ts
// Create validator: node ace make:validator post
import vine from '@vinejs/vine'

export const createPostValidator = vine.compile(
  vine.object({
    title: vine.string().trim().minLength(6),
    slug: vine.string().trim(),
    description: vine.string().trim().escape()
  })
)
```

### Using Validation in Controllers
```ts
import { createPostValidator } from '#validators/post_validator'

export default class PostsController {
  async store({ request }: HttpContext) {
    const payload = await request.validateUsing(createPostValidator)
    // payload is now validated and type-safe
    return payload
  }
}
```

### Best Practices
- Use dependency injection for services
- Validate all user input with VineJS
- Keep controllers thin, business logic in services
- Use resource controllers for RESTful operations

---

## Lucid ORM

### Technical Summary
Lucid is an Active Record ORM built on top of Knex, providing models, migrations, relationships, and a fluent query builder. It supports advanced SQL operations and database management.

### Basic Model
```ts
// Create model: node ace make:model User
import { DateTime } from 'luxon'
import { BaseModel, column } from '@adonisjs/lucid/orm'

export default class User extends BaseModel {
  @column({ isPrimary: true })
  declare id: number

  @column()
  declare email: string

  @column.dateTime({ autoCreate: true })
  declare createdAt: DateTime

  @column.dateTime({ autoCreate: true, autoUpdate: true })
  declare updatedAt: DateTime
}
```

### CRUD Operations
```ts
// Create
const newUser = await User.create({
  email: 'user@example.com',
  password: 'secret'
})

// Find
const user = await User.find(1)
const userByEmail = await User.findBy('email', 'user@example.com')

// Update
user.email = 'new@example.com'
await user.save()

// Delete
await user.delete()
```

### Query Builder
```ts
// Basic queries
const users = await User.query().where('isActive', true)
const user = await User.query().where('email', email).first()

// Advanced queries
const posts = await Post.query()
  .where('published', true)
  .orderBy('createdAt', 'desc')
  .limit(10)
```

### Migrations
```ts
// Create migration: node ace make:migration users
import { BaseSchema } from '@adonisjs/lucid/schema'

export default class extends BaseSchema {
  protected tableName = 'users'

  async up() {
    this.schema.createTable(this.tableName, (table) => {
      table.increments('id')
      table.string('email').unique()
      table.string('password')
      table.timestamps()
    })
  }

  async down() {
    this.schema.dropTable(this.tableName)
  }
}
```

### Best Practices
- Use migrations for database schema changes
- Define relationships properly for data integrity
- Use query builder for complex queries
- Leverage Lucid's built-in timestamps and soft deletes

---

## Authentication System

### Technical Summary
AdonisJS provides a robust authentication system with multiple guards (session, access tokens, basic auth) and providers. It supports JWT-like tokens, session-based auth, and remember me functionality.

### Session Guard Configuration
```ts
// config/auth.ts
import { defineConfig } from '@adonisjs/auth'
import { sessionGuard, sessionUserProvider } from '@adonisjs/auth/session'

const authConfig = defineConfig({
  default: 'web',
  guards: {
    web: sessionGuard({
      useRememberMeTokens: false,
      provider: sessionUserProvider({
        model: () => import('#models/user'),
      }),
    })
  },
})
```

### Session-Based Login
```ts
export default class SessionController {
  async store({ request, auth, response }: HttpContext) {
    const { email, password } = request.only(['email', 'password'])
    const user = await User.verifyCredentials(email, password)
    
    await auth.use('web').login(user)
    response.redirect('/dashboard')
  }

  async destroy({ auth, response }: HttpContext) {
    await auth.use('web').logout()
    response.redirect('/login')
  }
}
```

### Access Tokens Guard
```ts
// Configure User model for tokens
import { DbAccessTokensProvider } from '@adonisjs/auth/access_tokens'

export default class User extends BaseModel {
  static accessTokens = DbAccessTokensProvider.forModel(User)
}

// Issue tokens
const token = await User.accessTokens.create(user, ['*'], {
  name: 'API Token',
  expiresIn: '30 days'
})
```

### Protecting Routes
```ts
import { middleware } from '#start/kernel'

// Single guard
router.get('/dashboard', () => {}).use(middleware.auth())

// Multiple guards
router.get('/api/users', () => {}).use(
  middleware.auth({ guards: ['web', 'api'] })
)
```

### Accessing Authenticated User
```ts
export default class DashboardController {
  async index({ auth }: HttpContext) {
    // With auth middleware
    const currentUser = auth.user!
    
    // Or safely get user
    const authenticatedUser = auth.getUserOrFail()
    
    // Check if authenticated
    if (auth.isAuthenticated) {
      // User is logged in
    }
  }
}
```

### Best Practices
- Use session guard for web applications
- Use access tokens for APIs and mobile apps
- Always validate credentials securely
- Implement proper logout functionality
- Use remember me tokens for better UX

---

## Middleware, Exception Handling, and Lifecycle

### Technical Summary
Middleware provides request/response interception, exception handling centralizes error management, and lifecycle hooks allow customization of application boot process.

### Creating Middleware
```ts
// Create: node ace make:middleware auth
import { HttpContext } from '@adonisjs/core/http'
import { NextFn } from '@adonisjs/core/types/http'

export default class AuthMiddleware {
  async handle(ctx: HttpContext, next: NextFn) {
    // Before request processing
    console.log('Before request')
    
    // Continue to next middleware/handler
    await next()
    
    // After request processing (upstream)
    console.log('After request')
  }
}
```

### Middleware Stacks
```ts
// start/kernel.ts
import router from '@adonisjs/core/services/router'

// Server middleware (runs on every request)
server.use([
  () => import('@adonisjs/static/static_middleware')
])

// Router middleware (runs on matched routes)
router.use([
  () => import('@adonisjs/core/bodyparser_middleware')
])

// Named middleware
export const middleware = router.named({
  auth: () => import('#middleware/auth_middleware'),
  guest: () => import('#middleware/guest_middleware')
})
```

### Exception Handling
```ts
// app/exceptions/handler.ts
import { HttpContext, ExceptionHandler } from '@adonisjs/core/http'

export default class HttpExceptionHandler extends ExceptionHandler {
  protected debug = !app.inProduction
  
  async handle(error: unknown, ctx: HttpContext) {
    if (error instanceof ValidationError) {
      return ctx.response.status(422).send(error.messages)
    }
    
    return super.handle(error, ctx)
  }

  async report(error: unknown, ctx: HttpContext) {
    // Log or report errors
    return super.report(error, ctx)
  }
}
```

### Custom Exceptions
```ts
// Create: node ace make:exception Unauthorized
import { Exception } from '@adonisjs/core/exceptions'
import { HttpContext } from '@adonisjs/core/http'

export default class UnauthorizedException extends Exception {
  static status = 403
  static code = 'E_UNAUTHORIZED'

  async handle(error: this, ctx: HttpContext) {
    ctx.response.status(error.status).send({
      error: error.message
    })
  }
}
```

### Application Lifecycle Hooks
```ts
// Service Provider
export default class AppProvider {
  register() {
    // Register bindings
  }
  
  async boot() {
    // Boot services
  }
  
  async start() {
    // Pre-start actions
  }
  
  async ready() {
    // Post-start actions
  }
  
  async shutdown() {
    // Cleanup on shutdown
  }
}
```

### Best Practices
- Use middleware for cross-cutting concerns
- Handle exceptions centrally in the exception handler
- Use lifecycle hooks for application setup/teardown
- Keep middleware focused and lightweight

---

## WebSockets and HTTP Server

### Technical Summary
AdonisJS uses Transmit for Server-Sent Events (SSE) to provide real-time communication. The HTTP server is built from scratch with support for middleware, routing, and request lifecycle management.

### Transmit Setup
```ts
// Install: node ace add @adonisjs/transmit
// config/transmit.ts
import { defineConfig } from '@adonisjs/transmit'

export default defineConfig({
  pingInterval: false,
  transport: null, // or redis for multi-server sync
})
```

### Register Transmit Routes
```ts
// start/routes.ts
import transmit from '@adonisjs/transmit/services/main'

transmit.registerRoutes()

// Or manually
router.get('/__transmit/events', [EventStreamController])
router.post('/__transmit/subscribe', [SubscribeController])
router.post('/__transmit/unsubscribe', [UnsubscribeController])
```

### Broadcasting Events
```ts
import transmit from '@adonisjs/transmit/services/main'

// Broadcast to channel
transmit.broadcast('notifications', {
  message: 'New notification',
  timestamp: new Date()
})

// Broadcast except to specific user
transmit.broadcastExcept('chat/room1', message, userUid)
```

### Channel Authorization
```ts
// start/transmit.ts
transmit.authorize<{ id: string }>('users/:id', (ctx, { id }) => {
  return ctx.auth.user?.id === +id
})

transmit.authorize<{ id: string }>('chats/:id/messages', async (ctx, { id }) => {
  const chat = await Chat.findOrFail(+id)
  return ctx.bouncer.allows('accessChat', chat)
})
```

### Client-Side Usage
```ts
// Install: npm install @adonisjs/transmit-client
import { Transmit } from '@adonisjs/transmit-client'

const transmit = new Transmit({
  baseUrl: window.location.origin
})

// Create subscription
const subscription = transmit.subscription('notifications')
await subscription.create()

// Listen for messages
subscription.onMessage((data) => {
  console.log('Received:', data)
})

// Cleanup
await subscription.delete()
```

### HTTP Server Lifecycle
```ts
// HTTP Request Flow:
// 1. Create HttpContext
// 2. Execute server middleware
// 3. Find matching route
// 4. Execute router middleware
// 5. Execute route handler
// 6. Serialize response
```

### Best Practices
- Use channels for organizing real-time events
- Implement proper authorization for channels
- Handle client reconnections gracefully
- Disable GZip compression for SSE endpoints

---

## Testing with Japa

### Technical Summary
AdonisJS uses Japa as its testing framework with support for unit, functional, and HTTP tests. It provides plugins for API testing, authentication, sessions, and mocking.

### Test Structure
```ts
// Create test: node ace make:test users/create --suite=functional
import { test } from '@japa/runner'

test.group('Users', () => {
  test('create a new user', async ({ assert }) => {
    const user = await User.create({
      email: 'test@example.com',
      password: 'secret'
    })
    
    assert.exists(user.id)
    assert.equal(user.email, 'test@example.com')
  })
})
```

### HTTP Testing
```ts
// tests/bootstrap.ts - Setup API client
import { apiClient } from '@japa/api-client'

export const plugins: Config['plugins'] = [
  assert(),
  apiClient(),
  pluginAdonisJS(app)
]

// HTTP test example
test('get users list', async ({ client }) => {
  const response = await client.get('/users')
  
  response.assertStatus(200)
  response.assertBodyContains({
    data: [{ id: 1, email: 'user@example.com' }]
  })
})
```

### Authentication in Tests
```ts
// Setup auth plugin
import { authApiClient } from '@adonisjs/auth/plugins/api_client'

export const plugins: Config['plugins'] = [
  authApiClient(app)
]

// Test with authenticated user
test('get user profile', async ({ client }) => {
  const user = await User.create(userData)
  
  const response = await client
    .get('/profile')
    .loginAs(user)
    
  response.assertStatus(200)
})
```

### Session Testing
```ts
// Setup session plugin
import { sessionApiClient } from '@adonisjs/session/plugins/api_client'

// Test with session data
test('checkout with cart', async ({ client }) => {
  const response = await client
    .post('/checkout')
    .withSession({
      cartItems: [{ id: 1, name: 'Product' }]
    })
    
  response.assertStatus(200)
  response.assertSession('orderId')
})
```

### Mocks and Fakes
```ts
// Using container swaps
import app from '@adonisjs/core/services/app'

test('send email notification', async ({ client }) => {
  class FakeMailer {
    async send() {
      return { messageId: 'fake-id' }
    }
  }
  
  app.container.swap(MailService, () => new FakeMailer())
  
  // Test logic here
  
  app.container.restore(MailService)
})

// Using fakes API
import { MailManager } from '@adonisjs/mail'

test('send welcome email', async () => {
  const mail = app.container.make(MailManager)
  mail.fake()
  
  // Trigger email sending
  await UserService.sendWelcomeEmail(user)
  
  // Assert email was sent
  mail.assertSent((message) => {
    return message.to[0].address === user.email
  })
  
  mail.restore()
})
```

### Database Testing
```ts
test.group('User registration', (group) => {
  group.each.setup(async () => {
    await Database.beginGlobalTransaction()
  })
  
  group.each.teardown(async () => {
    await Database.rollbackGlobalTransaction()
  })
  
  test('creates user with hashed password', async ({ assert }) => {
    const user = await User.create({
      email: 'test@example.com',
      password: 'secret'
    })
    
    assert.isTrue(await Hash.verify(user.password, 'secret'))
  })
})
```

### Running Tests
```bash
# Run all tests
node ace test

# Run specific suite
node ace test functional

# Watch mode
node ace test --watch

# Filter tests
node ace test --tests="create user"
node ace test --files="users/*"
```

### Best Practices
- Use database transactions for test isolation
- Mock external services and APIs
- Use factories for test data generation
- Group related tests together
- Clean up resources in teardown hooks

---

## Environment Configuration

### Technical Summary
AdonisJS provides a robust environment variable system with validation, type safety, and support for multiple .env files. Environment variables are validated at startup and provide static type information.

### Basic Usage
```ts
// start/env.ts
import Env from '@adonisjs/core/env'

export default await Env.create(new URL('../', import.meta.url), {
  HOST: Env.schema.string({ format: 'host' }),
  PORT: Env.schema.number(),
  APP_KEY: Env.schema.string(),
  NODE_ENV: Env.schema.enum(['development', 'production', 'test'] as const),
  SESSION_DRIVER: Env.schema.string(),
  CACHE_VIEWS: Env.schema.boolean()
})

// Usage in application
import env from '#start/env'

const host = env.get('HOST')
const port = env.get('PORT', 3333) // with default
```

### Schema Validation
```ts
// String validation examples
const stringSchema = {
  APP_KEY: Env.schema.string(),
  EMAIL: Env.schema.string({ format: 'email' }),
  URL: Env.schema.string({ format: 'url' }),
  HOST: Env.schema.string({ format: 'host' })
}

// Number validation examples
const numberSchema = {
  PORT: Env.schema.number(),
  MAX_CONNECTIONS: Env.schema.number.optional()
}

// Boolean validation examples
const booleanSchema = {
  CACHE_VIEWS: Env.schema.boolean(),
  DEBUG: Env.schema.boolean.optional()
}

// Enum validation examples
const enumSchema = {
  NODE_ENV: Env.schema.enum(['development', 'production', 'test'] as const),
  LOG_LEVEL: Env.schema.enum.optional(['debug', 'info', 'warn', 'error'] as const)
}
```

### Environment Files
```bash
# File precedence (highest to lowest):
# 1. .env.[NODE_ENV].local
# 2. .env.local (not loaded in test)
# 3. .env.[NODE_ENV]
# 4. .env
```

### .env File Examples
```dotenv
# .env
PORT=3333
HOST=localhost
NODE_ENV=development
APP_KEY=your-secret-key
SESSION_DRIVER=cookie
CACHE_VIEWS=false

# Database
DB_CONNECTION=sqlite
DB_DATABASE=./database/database.sqlite

# Redis
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
```

```dotenv
# .env.test
NODE_ENV=test
SESSION_DRIVER=memory
DB_CONNECTION=sqlite
DB_DATABASE=:memory:
```

### Variable Substitution
```dotenv
# Reference other variables
HOST=localhost
PORT=3333
APP_URL=http://${HOST}:${PORT}

# With special characters
REDIS_USER=admin
REDIS_URL=localhost@${REDIS_USER}

# Escape dollar sign
PASSWORD=pa\$\$word
```

### Custom Validation
```ts
{
  PORT: (name, value) => {
    if (!value) {
      throw new Error(`${name} is required`)
    }
    
    const port = Number(value)
    if (isNaN(port) || port < 1 || port > 65535) {
      throw new Error(`${name} must be a valid port number`)
    }
    
    return port
  }
}
```

### Identifiers for Processing
```ts
// Define custom identifier
Env.defineIdentifier('base64', (value) => {
  return Buffer.from(value, 'base64').toString()
})
```

```dotenv
# Usage in .env file
APP_KEY=base64:U7dbSKkdb8wjVFOTq2osaDVz4djuA7BRLdoCUJEWxak=
```

### Best Practices
- Validate all environment variables at startup
- Use appropriate schema types for validation
- Keep sensitive data in environment variables
- Use different .env files for different environments
- Never commit .env files with secrets to version control

---

## IoC Container and Dependency Injection

### Technical Summary
AdonisJS features a powerful IoC container that enables automatic dependency injection using TypeScript decorators and provides container bindings for service registration and resolution.

### Basic Dependency Injection
```ts
// Service class
export default class UserService {
  async findAll() {
    return await User.all()
  }
}

// Controller with dependency injection
import { inject } from '@adonisjs/core'
import UserService from '#services/user_service'

@inject()
export default class UsersController {
  constructor(private userService: UserService) {}
  
  async index() {
    return this.userService.findAll()
  }
}
```

### Method Injection
```ts
import { inject } from '@adonisjs/core'
import { HttpContext } from '@adonisjs/core/http'

export default class UsersController {
  @inject()
  async index(ctx: HttpContext, userService: UserService) {
    return userService.findAll()
  }
}
```

### Container Bindings
```ts
// Register a binding
app.container.bind('cache', function () {
  return new CacheService()
})

// Register a singleton
app.container.singleton('logger', async (resolver) => {
  const config = await resolver.make('config')
  return new Logger(config.get('logging'))
})

// Bind a value directly
app.container.bindValue('version', '1.0.0')
```

### Using Container Directly
```ts
import app from '@adonisjs/core/services/app'

// Make an instance
const userService = await app.container.make(UserService)

// Call a method with injection
const result = await app.container.call(userService, 'someMethod')
```

### Container Swapping (Testing)
```ts
// Swap implementation for testing
test('user service test', async () => {
  class FakeUserService extends UserService {
    async findAll() {
      return [{ id: 1, name: 'Test User' }]
    }
  }
  
  app.container.swap(UserService, () => new FakeUserService())
  
  // Test logic here
  
  app.container.restore(UserService)
})
```

### Contextual Dependencies
```ts
// Different implementations for different classes
export default class AppProvider {
  register() {
    this.app.container
      .when(EmailService)
      .asksFor(MailDriver)
      .provide(async (resolver) => {
        const drive = await resolver.make('drive')
        return drive.use('smtp')
      })
      
    this.app.container
      .when(NotificationService)
      .asksFor(MailDriver)
      .provide(async (resolver) => {
        const drive = await resolver.make('drive')
        return drive.use('ses')
      })
  }
}
```

### Container Hooks
```ts
// Hook into container resolution
app.container.resolving('validator', (validator) => {
  validator.rule('unique', uniqueRule)
  validator.rule('exists', existsRule)
})
```

### Type Safety for Bindings
```ts
// Define types for container bindings
declare module '@adonisjs/core/types' {
  interface ContainerBindings {
    cache: CacheService
    logger: Logger
    version: string
  }
}
```

### Abstraction with Interfaces
```ts
// Abstract class as interface
export abstract class PaymentService {
  abstract charge(amount: number): Promise<void>
  abstract refund(amount: number): Promise<void>
}

// Concrete implementation
export class StripePaymentService implements PaymentService {
  async charge(amount: number) {
    // Stripe implementation
  }
  
  async refund(amount: number) {
    // Stripe implementation
  }
}

// Register in container
app.container.bind(PaymentService, () => {
  return app.container.make(StripePaymentService)
})
```

### Best Practices
- Use @inject decorator for automatic dependency injection
- Register services as singletons when appropriate
- Use container swapping for testing
- Define TypeScript interfaces for better type safety
- Avoid over-engineering with unnecessary abstractions
- Use contextual dependencies for different implementations

---

## Summary

This technical reference covers the essential aspects of AdonisJS v6 for backend development and testing. Each section provides practical examples, key APIs, and best practices to help you build robust applications efficiently.

For the most up-to-date information, always refer to the [official AdonisJS documentation](https://docs.adonisjs.com/).

---

*Generated from AdonisJS v6 official documentation*
