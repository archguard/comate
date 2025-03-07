<p align="center">
  <a href="https://github.com/archguard/co-mate"><img alt="XMonad logo" src="docs/co-mate.svg" height=150></a>
</p>
<p align="center">
    <a href="https://openjdk.java.net/"><img alt="Java support" src="https://img.shields.io/badge/Java-11+-green?logo=java&logoColor=white"></a>
    <a href="https://coveralls.io/github/archguard/co-mate?branch=master"><img alt="Coverage Status" src="https://coveralls.io/repos/github/archguard/co-mate/badge.svg?branch=master"></a>
    <a href="https://kotlinlang.org/"><img alt="languages" src="https://img.shields.io/badge/language-kotlin-blueviolet?logo=kotlin&logoColor=white"></a>
    <a href="https://opensource.org/license/mpl-2-0/"><img alt="GitHub license" src="https://img.shields.io/github/license/archguard/co-mate"></a>
    <brs>
    <img alt="GitHub tag (latest by date)" src="https://img.shields.io/github/v/tag/archguard/co-mate">
</p>

# ArchGuard Co-mate

Co-mate is an AI-powered software architecture copilot, design and governance tools.

Project Kanban: https://github.com/orgs/archguard/projects/5/views/1

Supported languages by [ArchGuard](https://github.com/archguard/archguard)
and [Chapi](https://github.com/modernizing/chapi):
Java, Kotlin, TypeScript/JavaScript, Golang, Python

## Design Principles

1. DSL as Ubiquitous Language: Empower efficient communication between humans and machines by employing a
   Domain-Specific Language (DSL).
2. Atomic LLM for Orchestration: Harness the atomic capabilities of a Language Model (LLM) to construct intricate
   behaviors within the DSL.
3. Dynamic Context by Layered: Divide the context into layers to handle complexities effectively through the LLM.

## Usage

1. download the latest release from [release page](https://github.com/archguard/co-mate/releases)
2. please put `OPENAI_API_KEY=xxx` in `~/.comate/.env`
3. run `java -jar comate-cli-*-all.jar` replace `*` with the version number

`.env` file example:

```
OPENAI_API_KEY=xxx
OPENAI_API_PROXY=xxxx (optional, if you had a OpenAI proxy server)
```

## Development

### Setup

Prerequisites: Node.js 16+, Java 11+

1. Clone

```
git clone https://github.com/archguard/co-mate
```

2. Setup backend server

```
./gradlew :comate-server:run
```

3. Setup frontend GUI

```shell
cd comate-gui

pnpm install
pnpm dev
```

### Modules

- `comate-server`: server side which provide REST API
- `comate-gui`: web GUI
- `comate-cli`: command line interface  [todo]

Core modules:

- `comate-core`: core logic, handle ArchGuard API, LLM logic, etc.
- LLM modules:
    - `llm-core`: large language model core, like OpenAI call, Token calculate, Tokenizer, Embedding, Similarity for
      Embedding, etc.
- Architecture Spec
    - `spec-lang`: the Kotlin DSL for architecture's specification, like: Domain Driven Design, RESTful API, etc.
    - `spec-runtime`: the Kotlin REPL/Interpreter for Architecture Spec DSL
    - `spec-partitioner`: the partitioner for partitioning the architecture spec document
- Architecture Define
    - `architecture`: architecture model

### Tech Stack

- GUI: Next.js + React + AI.js
- Server: Kotlin + Ktor
- DSL: Kotlin Type-Safe DSL
- Code Engine: ArchGuard & ArchGuard CodeDB

## Specification Language/DSL

```kotlin
domain {
    context_map("TicketBooking") {
        context("Reservation") {}
        context("Ticket") {}

        mapping {
            context("Reservation") dependedOn context("Ticket")
            context("Reservation") dependedOn context("Movie")
        }
    }
}
```

### User Journey

```kotlin
caseflow("MovieTicketBooking", defaultActor = "User") {
    // activity's should consider all user activities
    activity("AccountManage") {
        // task part should include all user tasks under the activity
        task("UserRegistration") {
            // you should list key steps in the story
            stories = listOf("Register with email", "Register with phone")
        }
        task("UserLogin") {
            stories += "Login to the website"
        }
    }
    activity("MovieSelection") {}
    // ...
    activity("PaymentCancel") {
        task("ConfirmCancel") {
            actor = "Admin" // if some task is role-specific, you can specify it here
            //...
        }
    }
}
```

### Concepts 

```kotlin
concepts {
    val customer = Concept("Customer") {
        behavior("Place Order", "Place an order for a coffee")
        behaviors = listOf(
            "View Menu",
            "Add to Cart",
            "Remove from Cart",
            "Place Order",
            "Pay",
            "View Order Status",
            "View Order History",
            "Customize Order"
        )
    }

    val barista = Concept("Barista") {
        behavior("Make Coffee")
    }

    val deliveryPerson = Concept("Delivery Person") {
        behavior("Deliver Order")
    }

    val shoppingCart = Concept("Shopping Cart") {
        behavior("Add to Cart", "Add a coffee to the shopping cart")
        behavior("Remove from Cart", "Remove a coffee from the shopping cart")
        behavior("View Cart", "View the contents of the shopping cart")
        behavior("Checkout", "Proceed to checkout and place the order")
    }

    relations {
        customer["View Menu"] perform barista
        customer["View Order History"] perform barista

        customer["Add to Cart"] perform shoppingCart
        customer["Remove from Cart"] perform shoppingCart
        customer["View Cart"] perform shoppingCart

        customer["Checkout"] perform barista
        customer["Place Order"] perform barista

        customer["Pay"] perform deliveryPerson
        customer["View Order Status"] perform deliveryPerson
        customer["Customize Order"].perform(barista) {
            // condition("").action("") // when need
        }
    }
}
```

### Foundation

```kotlin
foundation {
    project_name {
        pattern("^([a-z0-9-]+)-([a-z0-9-]+)-([a-z0-9-]+)(-common)?\$")
        example("system1-servicecenter1-microservice1")
    }

    layered {
        layer("interface") {
            pattern(".*\\.interface") { name shouldBe endsWith("Controller", "Service") }
        }
        layer("application") {
            pattern(".*\\.application") {
                name shouldBe endsWith("DTO", "Request", "Response", "Factory", "Service")
            }
        }
        layer("domain") {
            pattern(".*\\.domain") { name shouldBe endsWith("Entity") }
        }
        layer("infrastructure") {
            pattern(".*\\.infrastructure") { name shouldBe endsWith("Repository", "Mapper") }
        }

        dependency {
            "interface" dependedOn "domain"
            "interface" dependedOn "application"
            "interface" dependedOn "infrastructure"
            "application" dependedOn "domain"
            "application" dependedOn "infrastructure"
            "domain" dependedOn "infrastructure"
        }
    }

    naming {
        class_level {
            style("CamelCase")
            pattern(".*") { name shouldNotBe contains("$") }
        }
        function_level {
            style("CamelCase")
            pattern(".*") { name shouldNotBe contains("$") }
        }
    }
}
```

for MVC:

```kotlin
foundation {
    project_name {
        pattern("^([a-z0-9-]+)-([a-z0-9-]+)(-common)?\$")
        example("system1-webapp1")
    }

    layered {
        layer("controller") {
            pattern(".*\\.controller") { name shouldBe endsWith("Controller") }
        }
        layer("service") {
            pattern(".*\\.service") {
                name shouldBe endsWith("DTO", "Request", "Response", "Factory", "Service")
            }
        }
        layer("repository") {
            pattern(".*\\.repository") { name shouldBe endsWith("Entity", "Repository", "Mapper") }
        }

        dependency {
            "controller" dependedOn "service"
            "controller" dependedOn "repository"
            "service" dependedOn "repository"
        }
    }

    naming {
        class_level {
            style("CamelCase")
            pattern(".*") { name shouldNotBe contains("$") }
        }
        function_level {
            style("CamelCase")
            pattern(".*") { name shouldNotBe contains("$") }
        }
    }
}
```

### REST API

```kotlin
rest_api {
    uri_construction {
        pattern("/api\\/[a-zA-Z0-9]+\\/v[0-9]+\\/[a-zA-Z0-9\\/\\-]+")
        example("/api/petstore/v1/pets/dogs")
    }

    http_action("GET", "POST", "PUT", "DELETE")
    status_code(200, 201, 202, 204, 400, 401, 403, 404, 500, 502, 503, 504)

    security(
        """
Token Based Authentication (Recommended) Ideally, microservices should be stateless so the service instances can be scaled out easily and the client requests can be routed to multiple independent service providers. A token based authentication mechanism should be used instead of session based authentication
        """.trimIndent()
    )

    misc("""""")
}
```

Similar Project:

- [AutoDoc](https://github.com/context-labs/autodoc). Autodoc is an experimental toolkit for auto-generating codebase documentation for git repositories using Large
  Language Models, like GPT-4 or Alpaca. Autodoc can be installed in your repo in about 5 minutes. It indexes your
  codebase through a depth-first traversal of all repository contents and calls an LLM to write documentation for each
  file and folder. These documents can be combined to describe the different components of your system and how they work
  together.

License
---

This code is distributed under the MPL license. See `LICENSE` in this directory.
