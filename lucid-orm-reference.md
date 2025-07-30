# Lucid ORM Technical Reference

A comprehensive developer reference for AdonisJS Lucid ORM based on official documentation.

## Table of Contents

1. [Models](#models)
2. [Migrations](#migrations)
3. [Relationships](#relationships)
4. [Querying Data](#querying-data)
5. [Persisting Data](#persisting-data)
6. [Serializers and Attributes](#serializers-and-attributes)
7. [Factories and Seeders](#factories-and-seeders)
8. [Advanced Topics](#advanced-topics)

---

## Models

### Short Technical Summary
Models are JavaScript classes built on the active record pattern that represent database tables. Each model extends BaseModel and encapsulates database interactions, business logic, and data transformations.

### Key Methods, APIs, and Concepts
- `BaseModel` - Base class for all models
- `@column` - Decorator for database columns
- `@column.date` / `@column.dateTime` - Date/time columns with Luxon DateTime
- `make:model` - Ace command to create models
- `primaryKey`, `table`, `connection` - Model configuration
- `selfAssignPrimaryKey` - For custom primary key generation

### Code Examples

**Basic Model Definition:**
```ts
import { DateTime } from 'luxon'
import { BaseModel, column } from '@adonisjs/lucid/orm'

export default class User extends BaseModel {
  @column({ isPrimary: true })
  declare id: number

  @column()
  declare username: string

  @column({ serializeAs: null })
  declare password: string

  @column.dateTime({ autoCreate: true })
  declare createdAt: DateTime

  @column.dateTime({ autoCreate: true, autoUpdate: true })
  declare updatedAt: DateTime
}
```

**Custom Configuration:**
```ts
export default class User extends BaseModel {
  static table = 'app_users'
  static primaryKey = 'email'
  static connection = 'pg'
  static selfAssignPrimaryKey = true
}
```

### Best Practices & Warnings
- ⚠️ **Clear separation**: Models handle data operations, migrations handle schema
- ⚠️ **Column naming**: Use snake_case in database, camelCase in models (automatic conversion)
- ✅ **Use @column decorator**: Required to distinguish database columns from class properties
- ✅ **Leverage prepare/consume**: Transform data before saving/after fetching

---

## Migrations

### Short Technical Summary
Migrations are version-controlled database schema changes written as TypeScript classes. They provide incremental, reversible database modifications with up() and down() methods.

### Key Methods, APIs, and Concepts
- `BaseSchema` - Base class for migrations
- `make:migration` - Create migration files
- `migration:run` - Execute migrations
- `migration:rollback` - Reverse migrations
- `this.schema` - Schema builder instance
- `this.defer()` - Execute additional database operations

### Code Examples

**Basic Migration:**
```ts
import { BaseSchema } from '@adonisjs/lucid/schema'

export default class extends BaseSchema {
  protected tableName = 'users'

  async up() {
    this.schema.createTable(this.tableName, (table) => {
      table.increments('id')
      table.string('username').unique()
      table.string('email').unique()
      table.timestamp('created_at', { useTz: true })
      table.timestamp('updated_at', { useTz: true })
    })
  }

  async down() {
    this.schema.dropTable(this.tableName)
  }
}
```

**Data Migration with defer:**
```ts
async up() {
  this.schema.createTable('user_emails', (table) => {
    // table columns
  })

  this.defer(async (db) => {
    const users = await db.from('users').select('*')
    await Promise.all(
      users.map((user) => {
        return db.table('user_emails').insert({ 
          user_id: user.id, 
          email: user.email 
        })
      })
    )
  })

  this.schema.alterTable('users', (table) => {
    table.dropColumn('email')
  })
}
```

### Best Practices & Warnings
- ⚠️ **Avoid rollbacks in production**: Can cause data loss
- ⚠️ **Always move forward**: Create new migrations instead of editing existing ones
- ✅ **Use transactions**: Each migration runs in a transaction by default
- ✅ **Test migrations**: Use dry-run mode to preview SQL queries

---

## Relationships

### Short Technical Summary
Lucid supports all major relationship types with decorators and provides powerful APIs for preloading, lazy loading, and managing related data. Relationships are defined using decorators and can include custom keys, pivot tables, and nested relationships.

### Key Methods, APIs, and Concepts
- `@hasOne()` - One-to-one relationship
- `@hasMany()` - One-to-many relationship
- `@belongsTo()` - Inverse of hasOne/hasMany
- `@manyToMany()` - Many-to-many with pivot table
- `@hasManyThrough()` - Relationship through intermediate model
- `preload()` - Eager loading relationships
- `load()` - Lazy loading relationships
- `related()` - Relationship query builder

### Code Examples

**Defining Relationships:**
```ts
// User model
export default class User extends BaseModel {
  @hasOne(() => Profile)
  declare profile: HasOne<typeof Profile>

  @hasMany(() => Post)
  declare posts: HasMany<typeof Post>

  @manyToMany(() => Skill, {
    pivotTable: 'user_skills',
    pivotColumns: ['proficiency']
  })
  declare skills: ManyToMany<typeof Skill>
}

// Profile model
export default class Profile extends BaseModel {
  @column()
  declare userId: number

  @belongsTo(() => User)
  declare user: BelongsTo<typeof User>
}
```

**Preloading Relationships:**
```ts
// Basic preloading
const users = await User.query().preload('profile')

// Nested preloading
const users = await User.query().preload('posts', (postsQuery) => {
  postsQuery.preload('comments', (commentsQuery) => {
    commentsQuery.preload('user')
  })
})

// Multiple relationships
const users = await User.query()
  .preload('profile')
  .preload('posts')
  .preload('skills')
```

**Creating Relationships:**
```ts
const user = await User.findOrFail(1)

// Create related record
const post = await user.related('posts').create({
  title: 'Hello World',
  content: 'This is my first post'
})

// Many-to-many attach
await user.related('skills').attach([1, 2, 3])

// With pivot data
await user.related('skills').attach({
  [skillId]: { proficiency: 'Expert' }
})
```

### Best Practices & Warnings
- ⚠️ **Define relationships on models first**: Factory relationships require model relationships
- ⚠️ **Use preload for multiple records**: Avoid N+1 queries
- ✅ **Leverage relationship query builder**: Add constraints to related queries
- ✅ **Use pivot columns**: Store additional data in many-to-many relationships

---

## Querying Data

### Short Technical Summary
Lucid provides a fluent query builder built on Knex with model-aware extensions. Supports advanced SQL operations, scopes, pagination, aggregates, and relationship filtering.

### Key Methods, APIs, and Concepts
- `Model.query()` - Get model query builder
- `db.from()` - Database query builder
- `select()`, `where()`, `orderBy()` - Basic query methods
- `preload()` - Eager load relationships
- `withCount()`, `withAggregate()` - Relationship aggregates
- `has()`, `whereHas()` - Filter by relationship existence
- `paginate()` - Offset-based pagination
- `scope()` - Reusable query functions

### Code Examples

**Basic Queries:**
```ts
// Model queries
const users = await User.all()
const user = await User.find(1)
const user = await User.findBy('email', 'user@example.com')

// Query builder
const activeUsers = await User.query()
  .where('isActive', true)
  .orderBy('createdAt', 'desc')

// Database queries
const posts = await db.from('posts')
  .select('*')
  .where('status', 'published')
  .limit(10)
```

**Query Scopes:**
```ts
// Define scope
export default class Post extends BaseModel {
  static published = scope((query) => {
    query.where('publishedOn', '<=', DateTime.utc().toSQLDate())
  })

  static visibleTo = scope((query, user: User) => {
    if (!user.isAdmin) {
      query.where('userId', user.id)
    }
  })
}

// Use scopes
const posts = await Post.query()
  .withScopes((scopes) => scopes.published())
  .withScopes((scopes) => scopes.visibleTo(auth.user))
```

**Pagination:**
```ts
const page = request.input('page', 1)
const posts = await Post.query()
  .orderBy('createdAt', 'desc')
  .paginate(page, 10)

// Set base URL for links
posts.baseUrl('/posts')

// Serialize to JSON
return posts.toJSON()
```

### Best Practices & Warnings
- ⚠️ **Use orderBy with pagination**: Ensures consistent results
- ⚠️ **Avoid N+1 queries**: Use preload() for relationships
- ✅ **Leverage query scopes**: Reusable and testable query logic
- ✅ **Use relationship aggregates**: Get counts without loading data

---

## Persisting Data

### Short Technical Summary
Lucid provides multiple ways to persist data including model methods, query builders, and transaction support. Supports CRUD operations, mass assignment, and idempotent operations.

### Key Methods, APIs, and Concepts
- `create()`, `save()` - Create/update records
- `createMany()`, `updateOrCreate()` - Batch operations
- `fill()`, `merge()` - Mass assignment
- `db.transaction()` - Database transactions
- `useTransaction()` - Model transaction binding
- `$isPersisted`, `$dirty` - Model state properties

### Code Examples

**Basic CRUD:**
```ts
// Create
const user = await User.create({
  username: 'john',
  email: 'john@example.com'
})

// Update
const user = await User.findOrFail(1)
user.username = 'jane'
await user.save()

// Delete
await user.delete()
```

**Batch Operations:**
```ts
// Create many
const users = await User.createMany([
  { username: 'user1', email: 'user1@example.com' },
  { username: 'user2', email: 'user2@example.com' }
])

// Idempotent operations
const user = await User.firstOrCreate(
  { email: 'user@example.com' },
  { username: 'user', password: 'secret' }
)
```

**Transactions:**
```ts
// Managed transaction
await db.transaction(async (trx) => {
  const user = new User()
  user.username = 'john'
  user.useTransaction(trx)
  await user.save()

  await user.related('profile').create({
    fullName: 'John Doe'
  })
})

// Manual transaction
const trx = await db.transaction()
try {
  await User.create({ username: 'john' }, { client: trx })
  await trx.commit()
} catch (error) {
  await trx.rollback()
}
```

### Best Practices & Warnings
- ⚠️ **Use transactions for related data**: Ensure data consistency
- ⚠️ **Check $isPersisted**: Verify save operations succeeded
- ✅ **Use managed transactions**: Automatic commit/rollback
- ✅ **Leverage idempotent methods**: Prevent duplicate records

---

## Serializers and Attributes

### Short Technical Summary
Lucid models provide powerful serialization capabilities for API responses, including computed properties, field transformation, relationship serialization, and cherry-picking fields.

### Key Methods, APIs, and Concepts
- `serialize()`, `toJSON()` - Convert model to JSON
- `@computed` - Additional serialized properties
- `serializeAs` - Rename or hide properties
- `prepare`, `consume` - Transform column values
- `serializeExtras` - Include query extras
- Cherry-picking with `fields` and `relations`

### Code Examples

**Basic Serialization:**
```ts
export default class User extends BaseModel {
  @column({ isPrimary: true })
  declare id: number

  @column()
  declare firstName: string

  @column()
  declare lastName: string

  @column({ serializeAs: null })
  declare password: string

  @computed()
  get fullName() {
    return `${this.firstName} ${this.lastName}`
  }
}

const user = await User.find(1)
console.log(user.serialize())
// { id: 1, firstName: 'John', lastName: 'Doe', fullName: 'John Doe' }
```

**Value Transformation:**
```ts
export default class User extends BaseModel {
  @column({
    prepare: (value: string) => value.toLowerCase(),
    consume: (value: string) => value.toUpperCase()
  })
  declare username: string

  @column.dateTime({
    serialize: (value: DateTime | null) => {
      return value ? value.setZone('utc').toISO() : value
    }
  })
  declare createdAt: DateTime
}
```

**Cherry-picking Fields:**
```ts
const posts = await Post.query().preload('author').preload('comments')

const serialized = posts.map(post => post.serialize({
  fields: {
    pick: ['id', 'title', 'createdAt']
  },
  relations: {
    author: {
      fields: ['id', 'username']
    },
    comments: {
      fields: ['id', 'content']
    }
  }
}))
```

### Best Practices & Warnings
- ⚠️ **serializeAs: null is secure**: Cannot be overridden by cherry-picking
- ⚠️ **Guard against null values**: In prepare/consume functions
- ✅ **Use computed properties**: For derived data in API responses
- ✅ **Leverage relationship serialization**: Automatic nested serialization

---

## Factories and Seeders

### Short Technical Summary
Model factories generate fake data for testing and development, while seeders populate databases with initial data. Factories support states, relationships, and stubbing, while seeders support environment-specific execution.

### Key Methods, APIs, and Concepts
- `Factory.define()` - Define model factory
- `create()`, `createMany()` - Persist factory data
- `makeStubbed()` - Create without database
- `state()`, `apply()` - Factory variations
- `with()` - Create with relationships
- `BaseSeeder` - Base class for seeders
- `db:seed` - Run seeders

### Code Examples

**Model Factory:**
```ts
import User from '#models/user'
import Factory from '@adonisjs/lucid/factories'

export const UserFactory = Factory.define(User, ({ faker }) => {
  return {
    username: faker.internet.userName(),
    email: faker.internet.email(),
    password: faker.internet.password()
  }
})
.state('admin', (user) => user.isAdmin = true)
.relation('posts', () => PostFactory)
.build()

// Usage
const user = await UserFactory.create()
const adminUser = await UserFactory.apply('admin').create()
const userWithPosts = await UserFactory.with('posts', 3).create()
```

**Database Seeder:**
```ts
import { BaseSeeder } from '@adonisjs/lucid/seeders'
import User from '#models/user'

export default class UserSeeder extends BaseSeeder {
  static environment = ['development', 'testing']

  async run() {
    await User.updateOrCreateMany('email', [
      {
        email: 'admin@example.com',
        username: 'admin',
        isAdmin: true
      },
      {
        email: 'user@example.com',
        username: 'user',
        isAdmin: false
      }
    ])
  }
}
```

### Best Practices & Warnings
- ⚠️ **No tracking for seeders**: Can run multiple times unlike migrations
- ⚠️ **Use environment flags**: Prevent accidental production seeding
- ✅ **Use idempotent operations**: updateOrCreateMany for seeders
- ✅ **Leverage factory states**: Different variations of test data

---

## Advanced Topics

### Short Technical Summary
Advanced Lucid features include lifecycle hooks, custom naming strategies, raw queries, soft deletes, and custom primary keys. These provide flexibility for complex applications and specific requirements.

### Key Methods, APIs, and Concepts
- `@beforeSave`, `@afterCreate` - Lifecycle hooks
- `NamingStrategy` - Custom naming conventions
- `db.rawQuery()`, `db.raw()` - Raw SQL execution
- `$dirty`, `$original` - Model state tracking
- Custom primary keys with `selfAssignPrimaryKey`
- Global scopes and query hooks

### Code Examples

**Lifecycle Hooks:**
```ts
import hash from '@adonisjs/core/services/hash'

export default class User extends BaseModel {
  @column()
  declare password: string

  @beforeSave()
  static async hashPassword(user: User) {
    if (user.$dirty.password) {
      user.password = await hash.make(user.password)
    }
  }

  @beforeFind()
  static ignoreDeleted(query: ModelQueryBuilderContract<typeof User>) {
    query.whereNull('deletedAt')
  }
}
```

**Custom Primary Keys:**
```ts
import { randomUUID } from 'node:crypto'

export default class User extends BaseModel {
  static selfAssignPrimaryKey = true

  @column({ isPrimary: true })
  declare id: string

  @beforeCreate()
  static assignUuid(user: User) {
    user.id = randomUUID()
  }
}
```

**Raw Queries:**
```ts
// Standalone raw query
const users = await db.rawQuery('SELECT * FROM users WHERE age > ?', [18])

// Raw query in builder
const posts = await db.from('posts').select(
  'title',
  db.raw('(SELECT COUNT(*) FROM comments WHERE post_id = posts.id) as comment_count')
)

// Named placeholders
const user = await db.rawQuery(
  'SELECT * FROM users WHERE :column: = :value',
  { column: 'email', value: 'user@example.com' }
)
```

**Custom Naming Strategy:**
```ts
import { CamelCaseNamingStrategy } from '@adonisjs/lucid/orm'

class CustomNamingStrategy extends CamelCaseNamingStrategy {
  tableName(model: typeof BaseModel) {
    return string.singular(string.snakeCase(model.name))
  }

  serializedName(_model: typeof BaseModel, propertyName: string) {
    return string.camelCase(propertyName)
  }
}

// Apply to model
export default class User extends BaseModel {
  static namingStrategy = new CustomNamingStrategy()
}
```

### Best Practices & Warnings
- ⚠️ **Use placeholders in raw queries**: Prevent SQL injection
- ⚠️ **Hooks are static methods**: Receive model instance as parameter
- ✅ **Check $dirty in hooks**: Only process changed values
- ✅ **Use global scopes carefully**: Can affect all queries unexpectedly

---

## Summary

This reference covers the essential Lucid ORM features for backend development and testing. Lucid provides a mature, flexible ORM built on proven technologies (Knex.js) with powerful features for modern web applications.

### Key Takeaways
- **Active Record Pattern**: Models encapsulate both data and behavior
- **Migration-Driven Schema**: Explicit, version-controlled database changes  
- **Relationship-Rich**: Comprehensive relationship support with eager/lazy loading
- **Developer-Friendly**: Intuitive APIs with TypeScript support
- **Production-Ready**: Transaction support, connection pooling, and performance features

### When to Use Lucid
- ✅ Building APIs with complex data relationships
- ✅ Applications requiring database migrations
- ✅ Teams preferring explicit over implicit behavior
- ✅ Projects needing advanced SQL features

### Alternatives to Consider
- **Kysely**: For maximum type safety
- **Prisma**: For schema-first development
- **Raw Knex**: For maximum SQL control

---

*This reference is based on the official AdonisJS Lucid documentation and represents current best practices as of the documentation version reviewed.*
