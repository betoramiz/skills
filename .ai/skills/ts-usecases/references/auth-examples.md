# Real Authentication Use Cases

These are actual examples from the `src/features/auth/` module showing authentication patterns with bcrypt and JWT.

## Register Use Case

### Domain Entity

`src/features/auth/Auth.ts`:

```ts
import { fail, ok, type Result } from "@shared/result.ts";
import { AppError } from "@shared/errors/AppError.ts";

export interface AuthProps {
  email: string;
  password: string;
}

export class Auth {
  constructor(private props: AuthProps) {}

  static register(email: string, password: string): Result<AuthProps> {
    if (!email.length) {
      return fail(AppError.BadRequest("El email es requerido."));
    }

    if (password.length < 6) {
      return fail(AppError.BadRequest("Password inválido, debe tener al menos 6 caracteres."));
    }

    return ok({ email, password });
  }

  reconstitute(props: AuthProps): Auth {
    return new Auth(props);
  }

  toPrimitives(): AuthProps {
    return { ...this.props };
  }
}
```

### Queries

`src/features/auth/auth.queries.ts`:

```ts
import type { Db } from "@db/connection.ts";
import { type authInsert, authTable } from "@db/schemas/auth.schema.ts";
import type { AuthProps } from "./Auth.ts";
import { eq } from "drizzle-orm";
import type { LoginDto } from "./auth.dtos.ts";

const toInsert = (props: AuthProps): authInsert => ({
  email: props.email,
  password: props.password,
  isActive: true,
});

export const makeAuthQueries = (database: Db) => ({
  isUniqueEmail: async (email: string): Promise<boolean> => {
    const [result] = await database
      .select({
        id: authTable.id,
      })
      .from(authTable)
      .where(eq(authTable.email, email));

    return !!result;
  },

  register: async (props: AuthProps): Promise<string> => {
    const [result] = await database.insert(authTable)
      .values(toInsert(props))
      .returning({ id: authTable.id });

    return result?.id || '';
  },

  getByEmail: async (email: string): Promise<LoginDto | null> => {
    const [result] = await database.select({
      id: authTable.id,
      email: authTable.email,
      password: authTable.password,
    })
      .from(authTable)
      .where(eq(authTable.email, email));

    return result || null;
  }
});

export type AuthQueries = ReturnType<typeof makeAuthQueries>;
```

### Register Use Case

`src/features/auth/usecases/register.usecase.ts`:

```ts
import type { RegisterCommand } from "../auth.schemas.ts";
import type { AuthQueries } from "../auth.queries.ts";
import { Auth } from "../Auth.ts";
import { fail, ok, type Result } from "@shared/result.ts";
import { AppError } from "@shared/errors/AppError.ts";
import bcrypt from "bcrypt";

export interface RegisterResponse {
  idCreated: string;
}

export class RegisterUseCase {
  constructor(private readonly authQueries: AuthQueries) {}

  async execute(request: RegisterCommand): Promise<Result<RegisterResponse>> {

    const isUniqueEmail = await this.authQueries.isUniqueEmail(request.email);
    if(isUniqueEmail)
      return fail(AppError.Conflict('No se puede registar un email duplicado.'));

    const passwordEncrypted = await bcrypt.hash(request.password, 10);
    const data = Auth.register(request.email, passwordEncrypted);
    if(!data.ok) {
      return fail(data.error);
    }

    const result = await this.authQueries.register(data.value);

    return ok({ idCreated: result });
  }
}
```

### Key Patterns

1. **Pre-validation Check**: Query to check email uniqueness before attempting insert
2. **Password Hashing**: Use bcrypt to hash password before domain validation
3. **Domain Validation**: Auth.register validates email and password rules
4. **Conflict Handling**: Return explicit conflict error for duplicate emails
5. **Return ID**: Return the created auth ID for potential user creation

## Login Use Case

### Login DTO

`src/features/auth/auth.dtos.ts`:

```ts
export interface LoginDto {
  id: string;
  email: string;
  password: string;
}
```

### Error Helpers

`src/features/auth/usecases/errors.ts`:

```ts
import { AppError } from "@shared/errors/AppError.ts";
import { fail } from "@shared/result.ts";

export const userNotFoundError = (message?: string) =>
  fail(AppError.NotFound(message ?? 'Usuario no encontrado'));

export const invalidPassword = fail(AppError.BadRequest('Contraseña inválida'));
```

### Login Use Case

`src/features/auth/usecases/login.usecase.ts`:

```ts
import { sign } from "hono/jwt";
import type { AuthQueries } from "../auth.queries.ts";
import { invalidPassword, userNotFoundError } from "./errors.ts";
import { ok, type Result } from "@shared/result.ts";
import bcrypt from "bcrypt";
import { envConfig } from "../../../env.config.ts";

export interface LoginResponse {
  token: string;
}

export class LoginUseCase {
  constructor(private readonly authQueries: AuthQueries) {}

  async execute(user: string, password: string): Promise<Result<LoginResponse | null>> {
    const register = await this.authQueries.getByEmail(user);
    if(register === null) {
      return userNotFoundError();
    }

    const isPasswordValid = await bcrypt.compare(password, register.password);
    if(!isPasswordValid) {
      return invalidPassword;
    }

    const payload = {
      sub: register.id,
      exp: Math.floor(Date.now() / 1000) + 60 * 5, // Token expires in 5 minutes
    }
    const token = await sign(payload, envConfig.SECRET, 'HS256');

    return ok({ token });
  }
}
```

### Routes

`src/features/auth/auth.routes.ts`:

```ts
import { Hono } from "hono";
import { zValidator } from "@hono/zod-validator";
import { loginSchema, registerSchema } from "./auth.schemas.ts";
import { validationErrorAsJson } from "@shared/validation-errors.ts";
import { RegisterUseCase } from "./usecases/register.usecase.ts";
import { respond } from "@shared/respond.ts";
import { pinoLogger } from "hono-pino";
import { appLogger } from "@shared/middleware/logger.middleware.ts";
import type { LoginUseCase } from "./usecases/login.usecase.ts";

export const makeAuthRoutes = (register: RegisterUseCase, login: LoginUseCase) => {

  const router = new Hono();
  router.use(pinoLogger({ pino: appLogger }))

  router.post('/register', zValidator('json', registerSchema, validationErrorAsJson), async c => {

    const body = c.req.valid('json');

    const result = await register.execute(body);
    return respond(c, result);
  });

  router.post('/login', zValidator('json', loginSchema), async c => {
    const { email, password } = c.req.valid('json');

    const result = await login.execute(email, password);
    return respond(c, result);
  });

  return router;

}
```

### Key Patterns

1. **Credential Lookup**: Query by email to get stored password hash
2. **Password Verification**: Use bcrypt.compare to verify password
3. **JWT Generation**: Use Hono's JWT sign function with expiration
4. **Error Differentiation**: Separate errors for user not found vs invalid password
5. **No Auth Middleware**: Auth routes are public, no authMiddleware applied
6. **Environment Config**: Use SECRET from validated env config

## Authentication Middleware

`src/shared/middleware/auth.middleware.ts`:

```ts
import { jwt } from 'hono/jwt';
import { envConfig } from "../../env.config.ts";

export const authMiddleware = jwt({
  secret: envConfig.SECRET,
  alg: "HS256"
});
```

### Usage in Protected Routes

```ts
export const makeUserRoutes = (...) => {
  const router = new Hono();

  // Apply JWT auth to all user routes
  router.use('*', authMiddleware);
  router.use(pinoLogger({ pino: appLogger }));

  // All routes below require valid JWT
  router.post('/', ...);
  router.get('/:id', ...);

  return router;
};
```

## Best Practices

1. **Hash Before Validation**: Hash passwords before passing to domain entities
2. **Explicit Conflict Checks**: Check uniqueness before insert when you want custom error messages
3. **Token Expiration**: Always set exp claim on JWT tokens
4. **Error Helpers**: Create reusable error helpers for common auth failures
5. **No Sensitive Data in Tokens**: Only store user ID (sub) in JWT payload
6. **Middleware Separation**: Public routes don't use authMiddleware
7. **Type Safety**: Use DTOs to type query results that don't need full domain entities
