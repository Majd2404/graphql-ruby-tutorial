# GraphQL Cheat Sheet

## Quick Syntax Reference

### 1. Queries

```graphql
# Basic query
query {
  user {
    name
  }
}

# Query with arguments
query {
  user(id: "123") {
    name
    email
  }
}

# Query with variables
query GetUser($userId: ID!) {
  user(id: $userId) {
    name
    email
  }
}

# Variables (sent separately)
{
  "userId": "123"
}

# Query with aliases
query {
  firstUser: user(id: "1") {
    name
  }
  secondUser: user(id: "2") {
    name
  }
}

# Query with fragments
query {
  user(id: "1") {
    ...userFields
  }
}

fragment userFields on User {
  name
  email
  posts {
    title
  }
}
```

### 2. Mutations

```graphql
# Basic mutation
mutation {
  createUser(name: "Alice", email: "alice@example.com") {
    id
    name
  }
}

# Mutation with variables
mutation CreateUser($name: String!, $email: String!) {
  createUser(name: $name, email: $email) {
    id
    name
    errors
  }
}

# Variables
{
  "name": "Alice",
  "email": "alice@example.com"
}

# Multiple mutations (sequential execution)
mutation {
  first: createUser(name: "Alice") {
    id
  }
  second: createUser(name: "Bob") {
    id
  }
}
```

### 3. Subscriptions

```graphql
# Subscribe to new messages
subscription {
  messageAdded(roomId: "123") {
    id
    content
    user {
      name
    }
  }
}
```

## Ruby Type Definitions

### Scalar Types

```ruby
# Built-in types
field :id, ID, null: false
field :name, String, null: false
field :age, Integer, null: true
field :price, Float, null: false
field :active, Boolean, null: false

# Custom scalar
field :created_at, GraphQL::Types::ISO8601DateTime, null: false
```

### Object Types

```ruby
module Types
  class UserType < Types::BaseObject
    description "A user in the system"
    
    field :id, ID, null: false
    field :name, String, null: false
    field :email, String, null: false
    field :posts, [Types::PostType], null: true
    
    # Custom method
    field :full_name, String, null: false
    def full_name
      "#{object.first_name} #{object.last_name}"
    end
  end
end
```

### Enums

```ruby
module Types
  class UserRoleEnum < Types::BaseEnum
    value "ADMIN", value: "admin", description: "Administrator"
    value "USER", value: "user", description: "Regular user"
    value "GUEST", value: "guest", description: "Guest user"
  end
end
```

### Input Types

```ruby
module Types
  class UserInputType < Types::BaseInputObject
    argument :name, String, required: true
    argument :email, String, required: true
    argument :age, Integer, required: false
  end
end

# Usage in mutation
argument :user, Types::UserInputType, required: true
```

### Interfaces

```ruby
module Types
  class NodeInterface < Types::BaseInterface
    description "An object with an ID"
    
    field :id, ID, null: false
  end
  
  class UserType < Types::BaseObject
    implements NodeInterface
    # ... other fields
  end
end
```

### Unions

```ruby
module Types
  class SearchResultUnion < Types::BaseUnion
    description "Types that can appear in search results"
    
    possible_types UserType, PostType, CommentType
    
    def self.resolve_type(object, context)
      case object
      when User
        UserType
      when Post
        PostType
      when Comment
        CommentType
      end
    end
  end
end
```

## Common Resolver Patterns

### Basic Query

```ruby
module Types
  class QueryType < Types::BaseObject
    field :users, [UserType], null: false
    
    def users
      User.all
    end
  end
end
```

### Query with Arguments

```ruby
field :user, UserType, null: true do
  argument :id, ID, required: true
end

def user(id:)
  User.find_by(id: id)
end
```

### Paginated Query

```ruby
field :posts, [PostType], null: false do
  argument :limit, Integer, required: false, default_value: 10
  argument :offset, Integer, required: false, default_value: 0
end

def posts(limit:, offset:)
  Post.limit(limit).offset(offset)
end
```

### Authenticated Resolver

```ruby
field :current_user, UserType, null: true

def current_user
  context[:current_user]
end
```

## Mutation Patterns

### Basic Mutation

```ruby
module Mutations
  class CreateUser < BaseMutation
    argument :name, String, required: true
    argument :email, String, required: true
    
    field :user, Types::UserType, null: true
    field :errors, [String], null: false
    
    def resolve(name:, email:)
      user = User.new(name: name, email: email)
      
      if user.save
        { user: user, errors: [] }
      else
        { user: nil, errors: user.errors.full_messages }
      end
    end
  end
end
```

### Mutation with Authorization

```ruby
def resolve(**args)
  raise GraphQL::ExecutionError, "Not authenticated" unless context[:current_user]
  
  # mutation logic
end
```

## Schema Configuration

```ruby
class MyAppSchema < GraphQL::Schema
  # Set mutation, query, subscription types
  mutation(Types::MutationType)
  query(Types::QueryType)
  subscription(Types::SubscriptionType)
  
  # Limit query complexity
  max_depth 15
  max_complexity 300
  
  # Error handling
  rescue_from(ActiveRecord::RecordNotFound) do |exception|
    raise GraphQL::ExecutionError, "Record not found"
  end
end
```

## Testing with RSpec

```ruby
RSpec.describe Types::QueryType do
  describe 'user query' do
    let(:user) { create(:user, name: 'Alice') }
    let(:query) do
      <<~GQL
        query($id: ID!) {
          user(id: $id) {
            name
            email
          }
        }
      GQL
    end
    
    it 'returns the user' do
      result = MyAppSchema.execute(
        query,
        variables: { id: user.id }
      )
      
      expect(result.dig('data', 'user', 'name')).to eq('Alice')
    end
  end
end
```

## Common Directives

```graphql
# Skip field if condition is true
query ($withEmail: Boolean!) {
  user {
    name
    email @skip(if: $withEmail)
  }
}

# Include field if condition is true
query ($withEmail: Boolean!) {
  user {
    name
    email @include(if: $withEmail)
  }
}

# Deprecated field
field :old_field, String, null: true, 
      deprecation_reason: "Use newField instead"
```

## Error Handling

```ruby
# Custom error class
class AuthenticationError < GraphQL::ExecutionError
  def to_h
    super.merge({ "extensions" => { "code" => "UNAUTHENTICATED" } })
  end
end

# Usage
raise AuthenticationError, "Please sign in"
```

## Performance Optimization

### Batch Loading

```ruby
# Gemfile
gem 'graphql-batch'

# Loader
class RecordLoader < GraphQL::Batch::Loader
  def perform(ids)
    Model.where(id: ids).each { |record| fulfill(record.id, record) }
    ids.each { |id| fulfill(id, nil) unless fulfilled?(id) }
  end
end

# Usage
def user
  RecordLoader.for(User).load(object.user_id)
end
```

### Avoid N+1 Queries

```ruby
# Use includes in resolver
def posts
  Post.includes(:comments, :user).all
end
```

## Introspection Queries

```graphql
# Get all types
{
  __schema {
    types {
      name
    }
  }
}

# Get type details
{
  __type(name: "User") {
    name
    fields {
      name
      type {
        name
      }
    }
  }
}
```

## Quick Tips

1. **Always handle null values**: Specify `null: true` or `null: false`
2. **Use descriptions**: Help API consumers understand your schema
3. **Batch queries**: Use DataLoader to prevent N+1 queries
4. **Limit complexity**: Set max_depth and max_complexity
5. **Version with deprecation**: Don't create v2 endpoints, deprecate fields
6. **Return errors array**: Always provide clear error messages
7. **Use enums**: For fixed sets of values
8. **Document everything**: GraphQL is self-documenting, use it!

---

## Common Patterns Comparison

| Task | REST | GraphQL |
|------|------|---------|
| Get user | `GET /users/1` | `query { user(id: 1) { name } }` |
| Create user | `POST /users` | `mutation { createUser(name: "X") { id } }` |
| Update user | `PUT /users/1` | `mutation { updateUser(id: 1, name: "X") { id } }` |
| Delete user | `DELETE /users/1` | `mutation { deleteUser(id: 1) { success } }` |
| Get nested data | Multiple requests | Single query with nested fields |
| Real-time updates | WebSockets | Subscriptions |

---

**Happy querying! 🚀**
