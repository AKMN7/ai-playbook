# Coding style & communication charter

1. **Adhere to the established conventions**
    - All code, file names, exports, DTOs, schemas, services, and decorators **must** follow the rules defined in the other `.claude/rules/*` documents (project structure, naming conventions, feature workflow, error handling, etc.).
    - Assume NestJS decorators, class-validator DTOs, Mongoose schemas, and the controller-service-module pattern throughout.

2. **Prefer explicitness over assumption**
    - If a requirement, type, or edge-case is unclear, **ask** before generating code.
    - Clarifying questions take priority over silent assumptions—this prevents rework.

3. **Consistency is king**
    - New code should _blend in_—reuse existing patterns for services, DTOs, schemas, and controllers.
    - Divergence from conventions requires an Architecture Decision Record (ADR) or direct approval from Me.

4. **Autofix when trivial, question when non-trivial**
    - Minor lint or formatting issues may be fixed silently.
    - Anything affecting logic, API contracts, database schemas, or folder structure should trigger a question.

5. **Commenting style**
    - Every logical step in a service method gets a comment on the line above it.
    - Use the pattern: `// <Verb> <what>` (e.g., `// Get the workshop by event ID`, `// Check if the workshop exists`, `// Return success message`).
    - Match the density and style of the surrounding code—if the file is heavily commented, add comments; if it is not, keep the same level.

6. **Unused parameters**
    - When a controller method receives a parameter only for guard activation (e.g., `@CurrentUser()` on an admin-guarded route), prefix it with `_` (e.g., `_user`).

> Follow these principles on every turn; do not proceed with uncertain implementation details without first seeking clarification.
