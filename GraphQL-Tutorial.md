# GraphQL Tutorial: A Complete Guide with Ruby Examples

## Table of Contents
- [What is GraphQL?](#what-is-graphql)
- [GraphQL vs REST API](#graphql-vs-rest-api)
- [Core Concepts](#core-concepts)
- [Setting Up GraphQL in Ruby](#setting-up-graphql-in-ruby)
- [Real-World Examples](#real-world-examples)
- [Best Practices](#best-practices)

---

## What is GraphQL?

GraphQL is a query language for APIs and a runtime for executing those queries with your existing data. Developed by Facebook in 2012 and open-sourced in 2015, GraphQL provides a complete and understandable description of the data in your API.

### Key Features

- **Declarative Data Fetching**: Clients specify exactly what data they need
- **Single Endpoint**: Unlike REST, GraphQL uses a single endpoint for all operations
- **Strongly Typed**: Schema defines the API contract between client and server
- **No Over/Under-fetching**: Get exactly what you ask for, nothing more, nothing less
- **Real-time with Subscriptions**: Built-in support for real-time updates

---

## GraphQL vs REST API

### The Fundamental Difference

**REST** organizes data around resources with multiple endpoints, while **GraphQL** organizes data around a schema with a single endpoint.

### Comparison Table

| Feature | REST API | GraphQL |
|---------|----------|---------|
| **Endpoints** | Multiple endpoints per resource | Single endpoint (`/graphql`) |
| **Data Fetching** | Fixed data structure per endpoint | Client specifies exact data needed |
| **Over-fetching** | Common - get unnecessary data | Eliminated - request only what you need |
| **Under-fetching** | Common - need multiple requests | Eliminated - get all data in one request |
| **Versioning** | Often requires `/v1`, `/v2` endpoints | Schema evolution, deprecation fields |
| **Documentation** | Requires external tools (Swagger) | Self-documenting via introspection |
| **Learning Curve** | Lower - familiar HTTP methods | Moderate - new query language |
| **Caching** | Easy with HTTP caching | More complex - requires custom solutions |

### REST API Example

To get a user with their posts and comments, you might need 3 requests:

```ruby
# Request 1: Get user
GET /api/users/1
Response: { id: 1, name: "Alice", email: "alice@example.com", bio: "..." }

# Request 2: Get user's posts
GET /api/users/1/posts
Response: [
  { id: 101, title: "First Post", content: "...", created_at: "..." },
  { id: 102, title: "Second Post", content: "...", created_at: "..." }
]

# Request 3: Get comments for each post
GET /api/posts/101/comments
GET /api/posts/102/comments
```

**Problems:**
- Multiple network requests (N+1 problem)
- Over-fetching (getting `bio` when you don't need it)
- Under-fetching (need multiple requests)

### GraphQL Example

Get everything in **one request**:

```graphql
query {
  user(id: 1) {
    name
    email
    posts {
      title
      comments {
        author
        content
      }
    }
  }
}
```

**Response:**
```json
{
  "data": {
    "user": {
      "name": "Alice",
      "email": "alice@example.com",
      "posts": [
        {
          "title": "First Post",
          "comments": [
            { "author": "Bob", "content": "Great post!" }
          ]
        },
        {
          "title": "Second Post",
          "comments": [
            { "author": "Charlie", "content": "Interesting!" }
          ]
        }
      ]
    }
  }
}
```

---

## Core Concepts

### 1. Schema

The schema is the contract between client and server, defining what queries are possible.

```ruby
# app/graphql/types/user_type.rb
module Types
  class UserType < Types::BaseObject
    field :id, ID, null: false
    field :name, String, null: false
    field :email, String, null: false
    field :posts, [Types::PostType], null: true
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false
  end
end
```

### 2. Queries (Read Operations)

Queries fetch data from the server.

```ruby
# app/graphql/types/query_type.rb
module Types
  class QueryType < Types::BaseObject
    # Single user query
    field :user, UserType, null: true do
      argument :id, ID, required: true
    end

    def user(id:)
      User.find(id)
    end

    # List of users query
    field :users, [UserType], null: false

    def users
      User.all
    end
  end
end
```

**Usage:**
```graphql
# Get single user
query {
  user(id: 1) {
    name
    email
  }
}

# Get all users
query {
  users {
    id
    name
  }
}
```

### 3. Mutations (Write Operations)

Mutations modify data on the server.

```ruby
# app/graphql/mutations/create_user.rb
module Mutations
  class CreateUser < BaseMutation
    argument :name, String, required: true
    argument :email, String, required: true
    argument :password, String, required: true

    field :user, Types::UserType, null: true
    field :errors, [String], null: false

    def resolve(name:, email:, password:)
      user = User.new(name: name, email: email, password: password)
      
      if user.save
        { user: user, errors: [] }
      else
        { user: nil, errors: user.errors.full_messages }
      end
    end
  end
end
```

**Usage:**
```graphql
mutation {
  createUser(
    name: "Bob"
    email: "bob@example.com"
    password: "secure123"
  ) {
    user {
      id
      name
      email
    }
    errors
  }
}
```

### 4. Subscriptions (Real-time Updates)

Subscriptions enable real-time data updates.

```ruby
# app/graphql/types/subscription_type.rb
module Types
  class SubscriptionType < GraphQL::Schema::Object
    field :message_created, Types::MessageType, null: false do
      argument :room_id, ID, required: true
    end

    def message_created(room_id:)
      # Triggered when a new message is created
    end
  end
end
```

**Usage:**
```graphql
subscription {
  messageCreated(roomId: "123") {
    id
    content
    author {
      name
    }
  }
}
```

---

## Setting Up GraphQL in Ruby

### Step 1: Installation

Add to your `Gemfile`:

```ruby
gem 'graphql'
gem 'graphiql-rails' # GraphQL IDE for development
```

Run:
```bash
bundle install
rails generate graphql:install
```

### Step 2: Define Your Schema

```ruby
# app/graphql/my_app_schema.rb
class MyAppSchema < GraphQL::Schema
  mutation(Types::MutationType)
  query(Types::QueryType)
  subscription(Types::SubscriptionType)
end
```

### Step 3: Create a Controller

```ruby
# app/controllers/graphql_controller.rb
class GraphqlController < ApplicationController
  def execute
    variables = prepare_variables(params[:variables])
    query = params[:query]
    operation_name = params[:operationName]
    
    result = MyAppSchema.execute(
      query,
      variables: variables,
      operation_name: operation_name
    )
    
    render json: result
  rescue StandardError => e
    render json: { errors: [{ message: e.message }] }, status: 500
  end

  private

  def prepare_variables(variables_param)
    case variables_param
    when String
      JSON.parse(variables_param) || {}
    when Hash
      variables_param
    when ActionController::Parameters
      variables_param.to_unsafe_hash
    else
      {}
    end
  end
end
```

### Step 4: Add Routes

```ruby
# config/routes.rb
Rails.application.routes.draw do
  post "/graphql", to: "graphql#execute"
  
  if Rails.env.development?
    mount GraphiQL::Rails::Engine, at: "/graphiql", graphql_path: "/graphql"
  end
end
```

---

## Real-World Examples

### Example 1: Blog Application

#### Models

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_many :posts
  has_many :comments
end

# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :user
  has_many :comments
end

# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :user
  belongs_to :post
end
```

#### Types

```ruby
# app/graphql/types/post_type.rb
module Types
  class PostType < Types::BaseObject
    field :id, ID, null: false
    field :title, String, null: false
    field :content, String, null: false
    field :published, Boolean, null: false
    field :user, UserType, null: false
    field :comments, [CommentType], null: true
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false
    
    # Custom field with logic
    field :excerpt, String, null: true
    
    def excerpt
      object.content.truncate(100)
    end
  end
end

# app/graphql/types/comment_type.rb
module Types
  class CommentType < Types::BaseObject
    field :id, ID, null: false
    field :content, String, null: false
    field :user, UserType, null: false
    field :post, PostType, null: false
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false
  end
end
```

#### Queries with Arguments

```ruby
# app/graphql/types/query_type.rb
module Types
  class QueryType < Types::BaseObject
    # Search posts
    field :search_posts, [PostType], null: false do
      argument :query, String, required: true
      argument :published_only, Boolean, required: false, default_value: true
    end

    def search_posts(query:, published_only:)
      posts = Post.where("title ILIKE ? OR content ILIKE ?", "%#{query}%", "%#{query}%")
      posts = posts.where(published: true) if published_only
      posts
    end

    # Paginated posts
    field :posts, [PostType], null: false do
      argument :limit, Integer, required: false, default_value: 10
      argument :offset, Integer, required: false, default_value: 0
    end

    def posts(limit:, offset:)
      Post.limit(limit).offset(offset).order(created_at: :desc)
    end
  end
end
```

**Query Examples:**

```graphql
# Search posts
query {
  searchPosts(query: "GraphQL", publishedOnly: true) {
    title
    excerpt
    user {
      name
    }
  }
}

# Paginated posts
query {
  posts(limit: 5, offset: 10) {
    title
    createdAt
    comments {
      content
    }
  }
}
```

### Example 2: Mutations with Validation

```ruby
# app/graphql/mutations/update_post.rb
module Mutations
  class UpdatePost < BaseMutation
    argument :id, ID, required: true
    argument :title, String, required: false
    argument :content, String, required: false
    argument :published, Boolean, required: false

    field :post, Types::PostType, null: true
    field :errors, [String], null: false

    def resolve(id:, **attributes)
      post = Post.find(id)
      
      # Authorization check
      unless context[:current_user]&.id == post.user_id
        return { post: nil, errors: ["Unauthorized"] }
      end
      
      if post.update(attributes)
        { post: post, errors: [] }
      else
        { post: nil, errors: post.errors.full_messages }
      end
    rescue ActiveRecord::RecordNotFound
      { post: nil, errors: ["Post not found"] }
    end
  end
end
```

**Mutation Example:**

```graphql
mutation {
  updatePost(
    id: "1"
    title: "Updated Title"
    published: true
  ) {
    post {
      id
      title
      published
      updatedAt
    }
    errors
  }
}
```

### Example 3: N+1 Query Prevention with DataLoader

GraphQL can suffer from N+1 queries. Use batching to solve this:

```ruby
# Gemfile
gem 'graphql-batch'

# app/graphql/loaders/association_loader.rb
class AssociationLoader < GraphQL::Batch::Loader
  def initialize(model, association)
    @model = model
    @association = association
  end

  def perform(ids)
    records = @model.where(id: ids).includes(@association)
    records.each { |record| fulfill(record.id, record) }
    ids.each { |id| fulfill(id, nil) unless fulfilled?(id) }
  end
end

# app/graphql/types/post_type.rb
module Types
  class PostType < Types::BaseObject
    field :comments, [CommentType], null: true

    def comments
      AssociationLoader.for(Post, :comments).load(object.id).then(&:comments)
    end
  end
end
```

### Example 4: Authentication Context

```ruby
# app/controllers/graphql_controller.rb
class GraphqlController < ApplicationController
  def execute
    context = {
      current_user: current_user,
      current_ability: Ability.new(current_user)
    }
    
    result = MyAppSchema.execute(
      params[:query],
      variables: prepare_variables(params[:variables]),
      context: context
    )
    
    render json: result
  end

  private

  def current_user
    token = request.headers['Authorization']&.split(' ')&.last
    return nil unless token
    
    decoded = JWT.decode(token, Rails.application.secret_key_base)[0]
    User.find(decoded['user_id'])
  rescue
    nil
  end
end

# Usage in resolver
module Types
  class QueryType < Types::BaseObject
    field :me, UserType, null: true

    def me
      context[:current_user]
    end
  end
end
```

---

## Best Practices

### 1. Use Descriptive Field Names

```ruby
# Good
field :total_comments_count, Integer, null: false
field :is_published, Boolean, null: false

# Avoid
field :count, Integer, null: false
field :status, Boolean, null: false
```

### 2. Handle Errors Gracefully

```ruby
module Mutations
  class BaseMutation < GraphQL::Schema::RelayClassicMutation
    field :errors, [String], null: false
    field :success, Boolean, null: false
    
    def resolve(**args)
      result = perform(**args)
      {
        success: result[:errors].empty?,
        errors: result[:errors]
      }.merge(result)
    end
    
    def perform(**args)
      raise NotImplementedError
    end
  end
end
```

### 3. Implement Proper Authorization

```ruby
# app/graphql/types/query_type.rb
field :admin_users, [UserType], null: false

def admin_users
  raise GraphQL::ExecutionError, "Unauthorized" unless context[:current_user]&.admin?
  User.where(admin: true)
end
```

### 4. Use Enums for Fixed Values

```ruby
# app/graphql/types/post_status_enum.rb
module Types
  class PostStatusEnum < Types::BaseEnum
    value "DRAFT", value: "draft"
    value "PUBLISHED", value: "published"
    value "ARCHIVED", value: "archived"
  end
end

# Usage in type
field :status, PostStatusEnum, null: false
```

### 5. Document Your Schema

```ruby
module Types
  class UserType < Types::BaseObject
    description "A user of the application"
    
    field :id, ID, null: false, description: "Unique identifier"
    field :name, String, null: false, description: "User's full name"
    field :posts, [PostType], null: true, description: "All posts created by this user"
  end
end
```

### 6. Rate Limiting and Query Complexity

```ruby
# app/graphql/my_app_schema.rb
class MyAppSchema < GraphQL::Schema
  max_depth 10
  max_complexity 200
  
  query(Types::QueryType)
  mutation(Types::MutationType)
end
```

---

## Useful Resources

- **Official GraphQL Website**: [graphql.org](https://graphql.org)
- **GraphQL Ruby Gem**: [graphql-ruby.org](https://graphql-ruby.org)
- **GraphQL Best Practices**: [graphql.org/learn/best-practices](https://graphql.org/learn/best-practices)
- **Apollo GraphQL**: [apollographql.com](https://apollographql.com)

---

## Conclusion

GraphQL offers a powerful alternative to REST APIs, especially when:
- You have complex data relationships
- Mobile clients need to minimize data transfer
- You want to avoid versioning
- You need real-time capabilities

However, REST is still valuable for:
- Simple CRUD operations
- When HTTP caching is critical
- Legacy system integration
- Simpler infrastructure requirements

Choose the right tool for your specific use case!

---

**Happy Coding! 🚀**
