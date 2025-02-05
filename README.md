# ActionMCP

**ActionMCP** is a Ruby gem focused on providing Model Context Protocol (MCP) capability to Ruby on Rails applications, specifically as a server.

ActionMCP is designed for production Rails environments and does **not** support STDIO transport. STDIO is not included because it is not production-ready and is only suitable for desktop or script-based use cases. Instead, ActionMCP is built for robust, network-based deployments.

The client functionality in ActionMCP is intended to connect to remote MCP servers, not to local processes via STDIO.

It offers base classes and helpers for creating MCP applications, making it easier to integrate your Ruby/Rails application with the MCP standard.

With ActionMCP, you can focus on your app's logic while it handles the boilerplate for MCP compliance.

## Introduction

**Model Context Protocol (MCP)** is an open protocol that standardizes how applications provide context to large language models (LLMs).

Think of it as a universal interface for connecting AI assistants to external data sources and tools.

MCP allows AI systems to plug into various resources in a consistent, secure way, enabling two-way integration between your data and AI-powered applications.

This means an AI (like an LLM) can request information or actions from your application through a well-defined protocol, and your app can provide context or perform tasks for the AI in return.

**ActionMCP** is targeted at developers building MCP-enabled Rails applications. It simplifies the process of integrating Ruby and Rails apps with the MCP standard by providing a set of base classes and an easy-to-use server interface.

## Protocol Support

ActionMCP targets the released **MCP 2025-11-25** protocol. Older protocol versions and unreleased draft versions are not accepted. Core handling includes:

- **JSON-RPC 2.0** transport layer
- **Capability negotiation** during initialization
- **Error handling** with proper error codes (-32601 for method not found, -32002 for consent required)
- **Stateful session management** with explicit termination
- **Tasks, tools, prompts, resources, completion, logging, sampling, roots, and elicitation** message handling

The built-in Streamable HTTP endpoint returns one `application/json` message for
requests and intentionally returns HTTP 405 for GET because it does not provide
SSE streams. That behavior is allowed by the transport specification. Outbound
server requests and notifications are retained in session message storage, but
an application needs an SSE or custom transport to push those messages to a
client outside the response to an active request.

For a detailed (and entertaining) breakdown of protocol versions, features, and our design decisions, see [The Hitchhiker's Guide to MCP](The_Hitchhikers_Guide_to_MCP.md).

*Don't Panic: The guide contains everything you need to know about surviving MCP protocol versions.*

> **Note:** STDIO transport is not supported in ActionMCP. This gem is focused on production-ready, network-based deployments. STDIO is only suitable for desktop or script-based experimentation and is intentionally excluded.

Instead of implementing MCP support from scratch, you can subclass and configure the provided **Prompt**, **Tool**, and **ResourceTemplate** classes to expose your app's functionality to LLMs.

ActionMCP handles the underlying MCP message format and routing, so you can adhere to the open standard with minimal effort.

In short, ActionMCP helps you build an MCP server (the component that exposes capabilities to AI) more quickly and with fewer mistakes.

> **Client connections:** The client part of ActionMCP is meant to connect to remote MCP servers only. Connecting to local processes (such as via STDIO) is not supported.

## Requirements

- **Ruby**: 3.4.8+ or 4.0.0+
- **Rails**: 8.1.1+
- **Database**: PostgreSQL, MySQL, or SQLite3

ActionMCP is tested against Ruby 3.4.8 and 4.0.0 with Rails 8.1.1+.

## Installation

To start using ActionMCP, add it to your project:

```bash
# Add gem to your Gemfile
$ bundle add actionmcp

# Install dependencies
bundle install

# Copy migrations from the engine
bin/rails action_mcp:install:migrations

# Generate base classes and configuration
bin/rails generate action_mcp:install

# Create necessary database tables
bin/rails db:migrate
```

The `action_mcp:install` generator will:
- Create base application classes (ApplicationGateway, ApplicationMCPTool, etc.)
- Generate the MCP configuration file (`config/mcp.yml`)
- Set up the basic directory structure for MCP components (`app/mcp/`)

Database migrations are copied separately using `bin/rails action_mcp:install:migrations`.

## Core Components

ActionMCP provides three core abstractions to streamline MCP server development:

### ActionMCP::Prompt

`ActionMCP::Prompt` enables you to create reusable prompt templates that can be discovered and used by LLMs. Each prompt is defined as a Ruby class that inherits from `ApplicationMCPPrompt`.

Key features:
- Define expected arguments with descriptions and validation rules
- Build multi-step conversations with mixed content types
- Support for text, images, audio, and resource attachments
- Add messages with different roles (user/assistant)

**Example:**

```ruby
class AnalyzeCodePrompt < ApplicationMCPPrompt
  prompt_name "analyze_code"
  description "Analyze code for potential improvements"

  argument :language, description: "Programming language", default: "Ruby"
  argument :code, description: "Code to explain", required: true

  validates :language, inclusion: { in: %w[Ruby Python JavaScript] }

  def perform
    render(text: "Please analyze this #{language} code for improvements:")
    render(text: code)

    # You can add assistant messages too
    render(text: "Here are some things to focus on in your analysis:", role: :assistant)

    # Even add resources if needed
    render(resource: "file://documentation/#{language.downcase}_style_guide.pdf",
           mime_type: "application/pdf",
           blob: get_style_guide_pdf(language))
  end

  private

  def get_style_guide_pdf(language)
    # Implementation to retrieve style guide as base64
  end
end
```

Prompts can be executed by instantiating them and calling the `call` method:

```ruby
analyze_prompt = AnalyzeCodePrompt.new(language: "Ruby", code: "def hello; puts 'Hello, world!'; end")
result = analyze_prompt.call
```

### ActionMCP::Tool

`ActionMCP::Tool` allows you to create interactive functions that LLMs can call with arguments to perform specific tasks. Each tool is a Ruby class that inherits from `ApplicationMCPTool`.

Key features:
- Define input properties with types, descriptions, and validation
- Return multiple response types (text, images, errors)
- Progressive responses with multiple render calls
- Automatic input validation based on property definitions
- **Consent management for sensitive operations**

**Example:**

```ruby
class CalculateSumTool < ApplicationMCPTool
  tool_name "calculate_sum"
  description "Calculate the sum of two numbers"

  property :a, type: "number", description: "The first number", required: true
  property :b, type: "number", description: "The second number", required: true

  def perform
    sum = a + b
    render(text: "Calculating #{a} + #{b}...")
    render(text: "The sum is #{sum}")

    # You can report errors if needed
    if sum > 1000
      report_error("Warning: Sum exceeds recommended limit")
    end

    # Or even images
    render(image: generate_visualization(a, b), mime_type: "image/png")
  end

  private

  def generate_visualization(a, b)
    # Implementation to create a visualization as base64
  end
end
```

#### Consent Management

For tools that perform sensitive operations (file system access, database modifications, external API calls), you can require explicit user consent:

```ruby
class FileSystemTool < ApplicationMCPTool
  tool_name "read_file"
  description "Read contents of a file"

  # Require explicit consent before execution
  requires_consent!

  property :file_path, type: "string", description: "Path to file", required: true

  def perform
    # This code only runs after user grants consent
    content = File.read(file_path)
    render(text: "File contents: #{content}")
  end
end
```

**Consent Flow:**
1. When a consent-required tool is called without consent, it returns a JSON-RPC error with code `-32002`
2. The client must explicitly grant consent for the specific tool
3. Once granted, the tool can execute normally for that session
4. Consent is session-scoped and can be revoked at any time

**Managing Consent:**

```ruby
# Check if consent is granted
session.consent_granted_for?("read_file")

# Grant consent for a tool
session.grant_consent("read_file")

# Revoke consent
session.revoke_consent("read_file")
```

Tools can be executed by instantiating them and calling the `call` method:

```ruby
sum_tool = CalculateSumTool.new(a: 5, b: 10)
result = sum_tool.call
```

#### Structured output (output_schema)

Advertise a JSON Schema for your tool's structuredContent and return machine-validated results alongside any text output.

```ruby
class PriceQuoteTool < ApplicationMCPTool
  tool_name "price_quote"
  description "Return a structured price quote"

  property :sku, type: "string", description: "SKU to price", required: true

  output_schema do
    string :sku, required: true, description: "SKU that was priced"
    number :price_cents, required: true, description: "Total price in cents"
    object :meta do
      string :currency, required: true, enum: %w[USD EUR GBP]
      boolean :cached, default: false
    end
  end

  def perform
    price_cents = lookup_price_cents(sku) # Implement your lookup

    render structured: { sku: sku,
                         price_cents: price_cents,
                         meta: { currency: "USD", cached: false } }
  end
end
```

The schema is included in the tool definition, and the `structured` payload is emitted as `structuredContent` in the response while remaining compatible with text/audio/image renders.

#### Returning resource links from a tool

When you want to hand back a URI instead of embedding the payload, use the built-in `render_resource_link`, which produces the MCP `resource_link` content type.

```ruby
class ReportLinkTool < ApplicationMCPTool
  tool_name "report_link"
  description "Return a downloadable report link"

  property :report_id, type: "string", required: true

  def perform
    render_resource_link(
      uri: "reports://#{report_id}.json",
      name: "Report #{report_id}",
      description: "Downloadable JSON for report #{report_id}",
      mime_type: "application/json"
    )
  end
end
```

Clients can resolve the URI with a separate `resources/read` call, keeping tool responses lightweight while still discoverable.

#### Task-augmented tools (async execution with progress)

Use MCP Tasks when work might take seconds/minutes. Advertise task support with `task_required!` (or `task_optional!`) and let callers opt in by sending `_meta.task` on `tools/call`. While running as a task, you can emit progress updates with `report_progress!`.

```ruby
class BatchIndexTool < ApplicationMCPTool
  tool_name "batch_index"
  description "Index many items asynchronously with progress updates"

  task_required! # advertise that this tool is intended to run as a task
  property :items, type: "array_string", description: "Items to index", required: true

  def perform
    total = items.length
    items.each_with_index do |item, idx|
      index_item(item) # your indexing logic

      percent = ((idx + 1) * 100.0 / total).round
      report_progress!(percent: percent, message: "Indexed #{idx + 1}/#{total}")
    end

    render(text: "Indexed #{total} items")
  end

  private

  def index_item(item)
    # Implement your indexing logic here
  end
end
```

Call it as a task from a client by adding top-level `task` params (creates a Task record and runs the tool via `ToolExecutionJob`):

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "batch_index",
    "arguments": { "items": ["a", "b", "c"] },
    "task": { "ttl": 120000 }
  }
}
```

Poll task status with `tasks/get` or fetch the result with `tasks/result`.
As required by MCP 2025-11-25, `tasks/result` waits while a task is in any non-terminal state, including `input_required`, and returns once it reaches `completed`, `failed`, or `cancelled`. The internal database polling interval must be positive and can be tuned for your Rails app:

```ruby
config.action_mcp.tasks_result_poll_interval = 0.25.seconds
```

Use `tasks/cancel` to stop non-terminal tasks. `tasks/list` returns tasks in recent-first order and always paginates (default 50 per page, or `pagination_page_size` if configured). The response includes an opaque `nextCursor` when more results are available; treat cursors as opaque tokens.

### ActionMCP::ResourceTemplate

`ActionMCP::ResourceTemplate` facilitates the creation of URI templates for dynamic resources that LLMs can access.
This allows models to request specific data using parameterized URIs.

Templates support two MCP resource surfaces:
- **`resources/templates/list`** — parameterized URI patterns (existing behavior)
- **`resources/list`** — concrete resources with known URIs (via `self.list`)

> **Note:** Not all MCP clients support both resource endpoints. Claude Code (as of v2.1.50) only calls `resources/list`, and Codex stubs resource methods entirely. Implement `self.list` on your templates to ensure resources are visible to all clients. Crush and VS Code support both endpoints.

**Example:**

```ruby
class ProductResourceTemplate < ApplicationMCPResTemplate
  uri_template "product/{id}"
  description "Access product information by ID"
  mime_type "application/json"

  parameter :id, description: "Product identifier", required: true

  validates :id, format: { with: /\A\d+\z/, message: "must be numeric" }

  # Optional: enumerate concrete resources for resources/list
  def self.list(session: nil)
    Product.limit(50).map do |product|
      build_resource(
        uri: "product/#{product.id}",
        name: product.name,
        title: "Product ##{product.id}"
      )
    end
  end

  # Resolve a specific resource for resources/read
  def resolve
    product = Product.find_by(id: id)
    return unless product

    ActionMCP::Content::Resource.new(
      "product/#{id}",
      mime_type,
      text: product.to_json
    )
  end
end
```

**Callbacks:**

```ruby
before_resolve do |template|
  # Starting to resolve product: #{template.product_id}
end

after_resolve do |template|
  # Finished resolving product resource for product: #{template.product_id}
end

around_resolve do |template, block|
  start_time = Time.current
  resource = block.call
  resource
end
```

Resource templates are automatically registered and used when LLMs request resources matching their patterns.
See [RESOURCE_TEMPLATES.md](RESOURCE_TEMPLATES.md) for the full API reference including pagination, deduplication, and read contract details.

## 📚 Documentation

ActionMCP provides comprehensive documentation across multiple specialized guides. Each guide focuses on specific aspects to keep information organized and prevent context overload:

### Getting Started & Setup
- **[Installation & Configuration](README.md#installation)** - Initial setup, database migrations, and basic configuration
- **[Authentication with Gateway](README.md#authentication-with-gateway)** - User authentication and authorization patterns

### Component Development
- **[📋 TOOLS.MD](TOOLS.MD)** - Complete guide to developing MCP tools
  - Generator usage and best practices
  - Property definitions, validation, and consent management
  - Output schemas for structured responses
  - Error handling, testing, and security considerations
  - Advanced features like additional properties and authentication context

- **[📝 PROMPTS.MD](PROMPTS.MD)** - Prompt template development guide
  - Creating reusable prompt templates
  - Multi-step conversations and mixed content types
  - Argument validation and prompt chaining

- **[🔗 RESOURCE_TEMPLATES.md](RESOURCE_TEMPLATES.md)** - Resource template implementation
  - URI template patterns and parameter extraction
  - Dynamic resource resolution and collections
  - Callbacks and validation patterns

### Client & Integration
- **[🔌 CLIENTUSAGE.MD](CLIENTUSAGE.MD)** - Complete client implementation guide
  - Session management and resumability
  - Transport configuration and connection handling
  - Tool, prompt, and resource collections
  - Production deployment patterns
- **[🔐 GATEWAY.md](GATEWAY.md)** - Authentication gateway guide
  - Implementing `ApplicationGateway`
  - Identifier handling via `ActionMCP::Current`
  - Auth patterns, error handling, and hardening tips

### Protocol & Technical Details
- **[🚀 The Hitchhiker's Guide to MCP](The_Hitchhikers_Guide_to_MCP.md)** - Protocol versions and migration
  - Historical protocol comparison and the current 2025-11-25 behavior
  - Design decisions and architectural rationale
  - Migration paths and compatibility considerations
  - Feature evolution and technical specifications (*Don't Panic!*)

### Advanced Configuration
- **[Session Storage](README.md#session-storage)** - Volatile vs ActiveRecord vs custom session stores
- **[Thread Pool Management](README.md#thread-pool-management)** - Performance tuning and graceful shutdown
- **[Profiles System](README.md#profiles)** - Multi-tenant capability filtering
- **[Production Deployment](README.md#production-deployment-of-mcps0)** - Falcon, Unix sockets, and reverse proxy setup

### Development & Testing
- **[Generators](README.md#generators)** - Rails generators for scaffolding components
- **[Testing with TestHelper](README.md#testing-with-testhelper)** - Comprehensive testing strategies
- **[Development Commands](README.md#development-commands)** - Rake tasks for debugging and inspection
- **[MCP Inspector Integration](README.md#inspecting-your-mcp-server)** - Interactive testing and validation

### Troubleshooting & Production
- **[Error Handling](README.md#error-handling-and-troubleshooting)** - JSON-RPC error codes and debugging
- **[Production Considerations](README.md#production-considerations)** - Security, performance, and monitoring
- **[Middleware Conflicts](README.md#dealing-with-middleware-conflicts)** - Using `mcp_vanilla.ru` for production

> **💡 Pro Tip**: Start with the component-specific guides (TOOLS.MD, PROMPTS.MD, RESOURCE_TEMPLATES.md) for hands-on development, then reference the Hitchhiker's Guide for protocol details and CLIENTUSAGE.MD for integration patterns.

## Configuration

ActionMCP is configured via `config.action_mcp` in your Rails application.

By default, the name is set to your application's name and the version defaults to "0.0.1" unless your app has a version file.

You can override these settings in your configuration (e.g., in `config/application.rb`):

```ruby
module Tron
  class Application < Rails::Application
    config.action_mcp.name = "Friendly MCP (Master Control Program)"  # defaults to Rails.application.name
    config.action_mcp.version = "1.2.3"                               # defaults to "0.0.1"
    config.action_mcp.logging_enabled = true                          # defaults to true
    config.action_mcp.logging_level = :info                           # defaults to :info, can be :debug, :info, :warn, :error, :fatal

    # Server instructions - helps LLMs understand the server's purpose
    config.action_mcp.server_instructions = [
      "Use this server to access and control Tron system programs",
      "Helpful for managing system processes and user data"
    ]
  end
end
```

For dynamic versioning, consider adding the `rails_app_version` gem.

### Pagination

All list endpoints (`tools/list`, `prompts/list`, `resources/list`, `resources/templates/list`) support cursor-based pag