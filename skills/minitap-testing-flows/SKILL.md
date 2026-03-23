---
name: minitap-testing-flows
description: >
  Create and manage Minitap testing flow templates from a mobile app codebase.
  Use when user mentions "minitap testing flows", "create testing flows",
  "reconcile testing flows", or asks to generate test scenarios for their app.
---

# Minitap Testing Flows

You help users create and maintain **Minitap flow templates** — automated test
scenarios that an AI agent will execute on a real mobile device.

## Key Concepts

A **flow template** defines a user journey to test. It has:
- **name**: human-readable title (e.g. "User Login", "Add Item to Cart")
- **type**: one of `login`, `registration`, `checkout`, `onboarding`, `search`,
  `settings`, `navigation`, `form`, `profile`, `other`
- **description** (optional): context about the flow
- **acceptance criteria**: list of plain-text assertions an AI agent will verify
  visually on a phone screen

## Acceptance Criteria Rules

Each criterion **must** be:
- **Visually verifiable** — the AI agent only sees the screen. Write assertions
  about what is visible: text, UI elements, navigation state.
  - Good: "A success toast or confirmation message is displayed"
  - Bad: "The backend creates a new user record in the database"
- **Specific and unambiguous** — avoid vague assertions.
  - Good: "The cart badge shows the updated item count"
  - Bad: "The cart works correctly"
- **Scoped to a single check** — one assertion per criterion.
- **Ordered** — list criteria in the order they should be observed during the
  flow execution.

## MCP Tools

You have access to these tools via the `testing-service` MCP:

| Tool | Purpose |
|---|---|
| `get_user_apps` | List apps the user has access to (call first to get `app_id`) |
| `list_flow_templates` | List existing templates for an app |
| `get_flow_template` | Get a template with its acceptance criteria |
| `create_flow_template` | Create a new template |
| `update_flow_template` | Update name, description, type, or criteria |
| `delete_flow_template` | Remove a template |

## Workflow: Create Flow Templates

When the user asks to **create testing flows** for their app:

1. **Discover the app** — call `get_user_apps`. If there are multiple apps and
   it's not obvious from context (e.g. the codebase name or project config)
   which one to use, ask the user to pick one.

2. **Check existing templates** — call `list_flow_templates` to see what's
   already defined. Avoid creating duplicates.

3. **Analyse the codebase** — explore the project to identify testable user
   journeys. Look for:
   - Screen/view/page definitions and their visual content
   - Navigation structure (routes, stacks, tabs, drawers)
   - Authentication flows (login, signup, password reset)
   - Forms and user input screens
   - Search functionality
   - Settings and profile management
   - Checkout or payment flows
   - Onboarding sequences

4. **Map journeys to flow templates** — for each discovered journey:
   - Choose the most specific `type` from the enum
   - Write a clear `name` and optional `description`
   - Define acceptance criteria following the rules above
   - Cover the **happy path** end-to-end

5. **Create the templates** — call `create_flow_template` for each one.

6. **Summarise** — present the user with a table of created templates.

## Workflow: Reconcile Flow Templates

When the user asks to **reconcile** or **update flows given changes**:

1. **Fetch existing templates** — call `list_flow_templates` to get the current
   state.

2. **Analyse the codebase** — identify current user journeys (same analysis as
   creation).

3. **Diff** — compare existing templates against the codebase:
   - **New journeys** → create templates
   - **Removed journeys** → propose deletion (ask user to confirm)
   - **Changed journeys** → update templates with `update_flow_template`
   - **Unchanged journeys** → skip

4. **Execute changes** — create, update, or delete as needed.

5. **Summarise** — present a changelog showing what was added, updated, removed,
   and unchanged.

## Example Flow Template

**Name:** User Login  
**Type:** `login`  
**Description:** Standard email/password login from the app's welcome screen.  
**Acceptance Criteria:**
1. The login screen is displayed with email and password fields
2. After entering valid credentials and tapping the login button, a loading indicator appears
3. The home screen or main dashboard is displayed after successful login
4. The user's name or profile element is visible on the home screen

## Tips

- Prefer **fewer, well-defined flows** over many shallow ones.
- Group related screens into a single flow when they form a natural journey.
- Don't create separate flows for minor variations — use a single flow covering
  the main path.
- When the codebase uses feature flags or A/B tests, template the most common
  variant.
