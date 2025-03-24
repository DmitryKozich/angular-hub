# Standalone App Architecture

This document describes the recommended structure and practices for building scalable Angular applications using the **Standalone Component API**.

---

## 📁 Project Structure

```sh
angular-standalone-app/
├── src/
│   ├── app/
│   │   ├── core/                          # Application-wide infrastructure
│   │   │   ├── services/                  # Global singleton services (e.g. AuthService)
│   │   │   ├── guards/                    # Global route guards
│   │   │   ├── interceptors/              # HTTP interceptors
│   │   │   └── core.providers.ts          # Global providers for main.ts
│   │   ├── shared/                        # Reusable and generic building blocks
│   │   │   ├── components/                # UI primitives (e.g. Button, Card)
│   │   │   ├── directives/                # Generic reusable directives
│   │   │   ├── pipes/                     # Common pipes (e.g. truncate, capitalize)
│   │   │   ├── enums/                     # App-wide enums
│   │   │   ├── types/                     # Generic types and interfaces
│   │   │   ├── models/                    # Shared data models
│   │   │   └── constants/                 # Global constants
│   │   ├── features/                      # Business logic grouped by domain
│   │   │   ├── home/
│   │   │   │   ├── home.component.ts
│   │   │   │   ├── home.routes.ts
│   │   │   │   ├── components/            # Feature-specific UI components
│   │   │   │   ├── directives/            # Local-only directives
│   │   │   │   ├── pipes/                 # Local-only pipes
│   │   │   │   ├── services/              # Internal services
│   │   │   │   ├── models/                # Local models/interfaces
│   │   │   │   ├── constants/             # Feature-specific constants
│   │   │   │   └── store/                 # Local state (signals, stores, etc.)
│   │   │   ├── ...etc/
│   │   ├── app.component.ts               # Root standalone component
│   │   └── app.routes.ts                  # Root route configuration
│   ├── assets/                            # Static files (images, icons, etc.)
│   ├── environments/                      # Environment config files
│   ├── index.html                         # Main HTML entry point
│   ├── main.ts                            # Application bootstrap
│   └── styles.scss                        # Global styles
├── angular.json                           # Angular CLI configuration
├── package.json                           # Project dependencies
├── tsconfig.app.json                      # TypeScript config
└── README.md                              # Project documentation
```

---

### 🧱 `core/` — Application-wide infrastructure

**Purpose:**  
Contains **infrastructure-level code** used **once** across the app, such as services, interceptors, guards, and app setup logic. Responsible for **how** the app runs, not **what** it does.

**Typical contents:**
- Singleton services (e.g. `AuthService`, `LoggerService`)
- Route guards
- HTTP interceptors
- Global providers (`core.providers.ts`)
- App initializers (`APP_INITIALIZER`)
- Injection tokens and config providers

**When to use:**
- For logic that is **application-wide** and not feature-specific
- For bootstrapping and setup logic
- To separate infrastructure concerns from business logic

---

### 📦 `shared/` — Reusable and generic building blocks

**Purpose:**  
Holds **reusable, stateless, and generic code** used across multiple features. Focuses on **presentation and behavior**, not business logic.

**Typical contents:**
- UI components (e.g. `ButtonComponent`, `CardComponent`)
- Generic directives (e.g. `clickOutside`, `autofocus`)
- Common pipes (e.g. `truncate`, `capitalize`)
- Utility functions (e.g. `debounce`, `deepClone`)
- Shared types, enums, interfaces, and constants

**When to use:**
- When building **reusable** code needed in multiple places
- When designing **UI components** not tied to specific features
- For domain-agnostic logic and helpers

> 📌 `shared/` should contain logic that is **safe to use anywhere** in the app.

---

### 🧩 `features/` — Business logic & domain-specific UI

**Purpose:**  
Organizes code by **business domain** using a feature-based structure. Each feature is self-contained, including components, services, routes, constants, and optional state management.

**Typical contents:**
- Feature-specific components and pages (e.g. `ProfileComponent`)
- Internal services
- Local directives and pipes
- Route definitions (`*.routes.ts`)
- Local models, constants, types
- State management (e.g. signals, RxJS stores, etc.)

**When to use:**
- For **domain-specific logic and UI**
- When implementing **lazy-loaded** features
- To keep business logic **modular and isolated**

> 📌 A feature should be self-contained — **independent, testable, and scalable**

---

### 🚦 Summary: When to use what

| Characteristic                           | `core/`     | `shared/`           | `features/`                |
|-----------------------------------------|-------------|----------------------|-----------------------------|
| Used only once in the app               | ✅           | ❌                   | ❌                          |
| Used across multiple features           | ❌           | ✅                   | ❌                          |
| Contains business/domain-specific logic | ❌           | ❌                   | ✅                          |
| Used in app bootstrap or setup          | ✅           | ❌                   | ❌                          |
| Typically lazy-loaded                   | ❌           | ⚠️ *(rare)*          | ✅                          |
| UI components                           | ❌           | ✅ *(generic)*        | ✅ *(feature-specific)*     |

> ⚠️ `shared/` can be lazy-loaded in specific cases (e.g. heavy components or third-party integrations), but this is not common practice.

---

✅ **Tip:** Follow this structure from the start to keep your app **scalable, maintainable, and easy to reason about** as it grows.

