<h3 align="center">Next-ValidEnv</h3>
<p align="center">Typesafe environment variables for Next.js</p>
<p align="center">
    <a href="https://packagephobia.com/result?p=next-validenv">
        <img src="https://packagephobia.com/badge?p=next-validenv" alt="Bundle Size" />
    </a>
    <a href="https://www.npmtrends.com/next-validenv">
        <img src="https://img.shields.io/npm/dm/next-validenv" alt="Downloads" />
    </a>
    <a href="https://github.com/jacobadevore/next-validenv/stargazers">
        <img src="https://img.shields.io/github/stars/jacobadevore/next-validenv" alt="Github Stars" />
    </a>
    <a href="https://www.npmjs.com/package/next-validenv">
        <img src="https://img.shields.io/github/v/release/jacobadevore/next-validenv?label=latest"
            alt="Github Stable Release" />
    </a>
</p>
<p align="center">Created by</p>
<div align="center">
    <td align="center"><a href="https://twitter.com/JacobADevore"><img
                src="https://avatars.githubusercontent.com/u/20541754?v=4?s=100" width="100px;"
                alt="" /><br /><sub><b>Jacob Devore</b></sub></a></td>
</div>

---

Next-ValidEnv allows you to easily create typesafe environment variables in Next.js

---

### Installation

```sh
npm install zod next-validenv       # npm
yarn add zod next-validenv          # yarn
bun add zod next-validenv           # bun
pnpm add zod next-validenv          # pnpm
```

### First, create `env.mjs`

```js
// @ts-check
import { validateEnvironmentVariables } from "next-validenv";
import { z } from "zod";

/**
 * Specify your server-side environment variables schema here.
 * This way you can ensure the app isn't built with invalid env vars.
 */
export const serverSchema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]),
});

/**
 * Specify your client-side environment variables schema here.
 * This way you can ensure the app isn't built with invalid env vars.
 * To expose them to the client, prefix them with `NEXT_PUBLIC_`.
 */
export const clientSchema = z.object({
  // NEXT_PUBLIC_CLIENT: z.string(),
});

const environmentVariables = validateEnvironmentVariables(
  clientSchema,
  serverSchema
);

export const env = environmentVariables;
```

### Update ~~`next.config.js`~~ to `next.config.mjs`

```js
// @ts-check
/**
 * Run `build` or `dev` with `SKIP_ENV_VALIDATION` to skip env validation.
 * This is especially useful for Docker builds.
 */
!process.env.SKIP_ENV_VALIDATION && (await import("./env.mjs"));

/** @type {import("next").NextConfig} */
const config = {
  reactStrictMode: true,
};
export default config;
```

### That's it! Now you can use `env.[variable]`

```js
import { env } from "./env.mjs";

env.NODE_ENV; // Typesafe environment variables
```

<br />

---

## Manual Implementation

Follow the below guide to manually implement typesafe environment variables in Next.js without installing the Next-ValidEnv library

---

### Installation

```sh
npm install zod      # npm
yarn add zod         # yarn
bun add zod          # bun
pnpm add zod         # pnpm
```

### First, create `validation.mjs`

```js
// @ts-check

export const mapProcessEnvToObject = (
  /** @type {import('zod').ZodObject} */ schema
) => {
  /** @type {{ [key: string]: string | undefined; }} */
  let env = {};

  Object.keys(schema.shape).forEach((key) => (env[key] = process.env[key]));

  return schema.safeParse(env);
};

export const formatZodErrors = (
  /** @type {import('zod').ZodFormattedError<Map<string,string>,string>} */ errors
) =>
  Object.entries(errors)
    .map(([name, value]) => {
      if (value && "_errors" in value)
        return `${name}: ${value._errors.join(", ")}\n`;
    })
    .filter(Boolean);

export const formatErrors = (/** @type string[]} */ errors) =>
  errors.map((name) => `${name}\n`);

export const validateEnvironmentVariables = (
  /** @type {import('zod').ZodObject} */ clientSchema,
  /** @type {import('zod').ZodObject} */ serverSchema
) => {
  let serverEnv = mapProcessEnvToObject(serverSchema);
  let clientEnv = mapProcessEnvToObject(clientSchema);

  let invalidEnvErrors = [];

  if (!serverEnv.success) {
    invalidEnvErrors = [
      ...invalidEnvErrors,
      ...formatZodErrors(serverEnv.error.format()),
    ];
  }

  if (!clientEnv.success) {
    invalidEnvErrors = [
      ...invalidEnvErrors,
      ...formatZodErrors(clientEnv.error.format()),
    ];
  }

  if (!serverEnv.success || !clientEnv.success) {
    console.error("❌ Invalid environment variables:\n", ...invalidEnvErrors);
    throw new Error("Invalid environment variables");
  }

  let exposedServerEnvErrors = [];

  for (let key of Object.keys(serverEnv.data)) {
    if (key.startsWith("NEXT_PUBLIC_")) {
      exposedServerEnvErrors = [...exposedServerEnvErrors, key];
    }
  }

  if (exposedServerEnvErrors.length > 0) {
    console.error(
      "❌ You are exposing the following server-side environment variables to the client:\n",
      ...formatErrors(exposedServerEnvErrors)
    );
    throw new Error(
      "You are exposing the following server-side environment variables to the client"
    );
  }

  let notExposedClientEnvErrors = [];

  for (let key of Object.keys(clientEnv.data)) {
    if (!key.startsWith("NEXT_PUBLIC_")) {
      notExposedClientEnvErrors = [...notExposedClientEnvErrors, key];
    }
  }

  if (notExposedClientEnvErrors.length > 0) {
    console.error(
      "❌ All client-side environment variables must begin with 'NEXT_PUBLIC_', you are not exposing the following:\n",
      ...formatErrors(notExposedClientEnvErrors)
    );
    throw new Error(
      "All client-side environment variables must begin with 'NEXT_PUBLIC_', you are not exposing the following:"
    );
  }

  return { ...serverEnv.data, ...clientEnv.data };
};
```

### Second, create `env.mjs`

```js
// @ts-check
import { validateEnvironmentVariables } from "./validation.mjs";
import { z } from "zod";

/**
 * Specify your server-side environment variables schema here.
 * This way you can ensure the app isn't built with invalid env vars.
 */
export const serverSchema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]),
});

/**
 * Specify your client-side environment variables schema here.
 * This way you can ensure the app isn't built with invalid env vars.
 * To expose them to the client, prefix them with `NEXT_PUBLIC_`.
 */
export const clientSchema = z.object({
  // NEXT_PUBLIC_CLIENT: z.string(),
});

const environmentVariables = validateEnvironmentVariables(
  clientSchema,
  serverSchema
);

export const env = environmentVariables;
```

### Third, update ~~`next.config.js`~~ to `next.config.mjs`

```js
// @ts-check
/**
 * Run `build` or `dev` with `SKIP_ENV_VALIDATION` to skip env validation.
 * This is especially useful for Docker builds.
 */
!process.env.SKIP_ENV_VALIDATION && (await import("./env.mjs"));

/** @type {import("next").NextConfig} */
const config = {
  reactStrictMode: true,
};
export default config;
```

### That's it! Now you can use `env.[variable]`

```js
import { env } from "./env.mjs";

env.NODE_ENV; // Typesafe environment variables
```