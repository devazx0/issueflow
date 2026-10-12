# Prisma Service Architecture and Configuration

This document outlines the technical implementation and configuration of the Prisma layer within **IssueFlow**, focusing on the service architecture, database adapters, and custom extensions.

---

## 1. Global Module Configuration
Prisma is integrated into the NestJS application via a globally-scoped module:
- **File**: `apps/api/src/modules/prisma/prisma.module.ts`
- **Pattern**: Marked with `@Global()`, ensuring the `PrismaService` is available throughout the application without needing to import it into every module.

---

## 2. Prisma Client Configuration (`@prisma/config`)
The project utilizes the new Prisma configuration standard to manage setup centrally:
- **File**: `apps/api/prisma.config.ts`
- **Feature**: Uses `defineConfig` to specify the schema location and dynamic datasource URLs, ensuring environment variables are loaded via `dotenv/config`. This is a modern approach for managing Prisma settings outside of the `schema.prisma` file.

---

## 3. Database Adapter & Pooling
For improved performance and reliability, **IssueFlow** uses an external driver adapter instead of the default Prisma engine:
- **Adapter**: `@prisma/adapter-pg`
- **Pool Management**: Uses `pg.Pool` from the `pg` library.
- **Benefits**: This allows the application to manage database connections more effectively than the standard Prisma driver, especially in serverless or highly concurrent environments, by providing fine-grained control over the connection pool.

---

## 4. Architectural Implementation: `PrismaService`
The `PrismaService` is the central component of the database layer, implementing a sophisticated inheritance and proxy pattern:
- **File**: `apps/api/src/modules/prisma/prisma.service.ts`

### Key Implementation Details:
1.  **Constructor Proxying**: The service returns a **JavaScript Proxy** from its constructor. This is a powerful pattern that redirects all model access (e.g., `this.prisma.user`) to an *extended* client while still maintaining the standard Injectable service interface.
2.  **Extension Injection**: The `Soft-Delete` extension is applied to the client at initialization, and the proxy ensures that all model-level operations pass through this extension.
3.  **Lifecycle Management**: It implements `OnModuleInit` to explicitly call `this.$connect()`, ensuring the database connection is established when the NestJS application starts.

---

## 5. Custom Extension System: Soft-Delete
The application uses a custom extension to handle soft-deletion transparently at the database layer.
- **File**: `apps/api/src/modules/prisma/soft-delete.extension.ts`

### Logic Flow:
- **Operation Interception**: The extension uses `Prisma.$extends` to hook into `$allOperations`.
- **Read Operations**: Automatically injects `{ deletedAt: null }` into `where` clauses for `findMany`, `findFirst`, `count`, `aggregate`, and `groupBy`.
- **Write Operations**: Re-routes `delete` and `deleteMany` calls to `update/updateMany` respectively, setting the `deletedAt` field to the current timestamp.
- **Point Lookups**: Specifically for `findUnique`, it checks the result after the fetch and returns `null` or throws an error if the record is soft-deleted.

---

## 6. Development Scripts and Environment
The `package.json` in `apps/api` defines the interface for interacting with this setup:

| Command | Configuration | Purpose |
| :--- | :--- | :--- |
| `prisma:migrate` | `.env` | Dev migrations |
| `prisma:migrate:prod` | `.env.prod` (via `dotenv-cli`) | Production schema updates |
| `prisma:generate` | `prisma.config.ts` | Client generation |

---

> [!IMPORTANT]
> The architectural use of a Proxy in the `PrismaService` constructor means that when you inject `PrismaService` into other services, you are actually interacting with a client that has already been invisibly extended with the soft-delete logic. No manual filtering for `deletedAt: null` is required in business logic.
