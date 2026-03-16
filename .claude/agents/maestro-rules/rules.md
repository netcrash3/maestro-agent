# Coding Rules

## Frontend — JavaScript / TypeScript

- Global and environment variables and constants should be UPPER_SNAKE_CASE
- All class, enum, and interface names should be PascalCase
- All interface names should begin with "I" (e.g. `IMyInterface`)
- All function names should be camelCase
- All variable and function parameter names should be camelCase
- Do not use hard coded strings in logic — if a hard coded string is needed, make it a constant; if more than one value is possible, use an enumeration
- Prefix all boolean variable names with "is" (e.g. `isDeleted`)
- Never hardcode sensitive information such as API keys in the frontend
- Represent generic types with "T"

---

## Backend — JavaScript / TypeScript

- Global and environment variables and constants should be UPPER_SNAKE_CASE
- All class, enum, and interface names should be PascalCase
- All interface names should begin with "I" (e.g. `IMyClass`)
- All function names should be camelCase
- All variable and function parameter names should be camelCase
- Do not use hard coded strings in logic — if a hard coded string is needed, make it a constant; if more than one value is possible, use an enumeration
- Prefix all boolean variable names with "is" (e.g. `isDeleted`)
- Never hardcode sensitive information such as API keys — store them in an environment file
- Keep API endpoints thin; all business logic should be handled in service classes:
  ```ts
  async function getMyData(id: string) {
      var svc = NewDataService();
      return svc.Get(id);
  }
  ```
- All database access and query code should be handled in repository classes called by service classes — repository methods should contain no business logic
- Represent generic types with "T"

---

## Backend — Golang

- All type names and property names should be PascalCase
- Function names should be PascalCase unless they are meant to be private to the containing struct, in which case they should be camelCase
- All parameter names should be camelCase
- Keep API endpoints thin; all business logic should be handled by service structs
- All database access and query code should be handled in repository structs called by service structs — repository methods should contain no business logic
- Never hardcode sensitive information such as API keys — store them in an environment file
- Do not use hard coded strings in logic — if a hard coded string is needed, make it a constant; if more than one value is possible, use an enumeration
- Prefix all boolean variable names with "is" (e.g. `isDeleted`)

---

## Backend — .NET

- All class, enum, interface names, and property names should be PascalCase
- All function names should be PascalCase
- All parameter names should be camelCase
- All interface names should begin with "I" (e.g. `IMyClass`)
- Keep API endpoints thin; all business logic should be handled by service classes
- All database access and query code should be handled in repository classes called by service classes — repository methods should contain no business logic
- Never hardcode sensitive information such as API keys — store them in an environment configuration file
- Do not use hard coded strings in logic — if a hard coded string is needed, make it a constant; if more than one value is possible, use an enumeration
- Prefix all boolean variable names with "is" (e.g. `isDeleted`)
- Represent generic types with "T"
