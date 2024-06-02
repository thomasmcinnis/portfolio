---
title: Implement Lucia email auth with Astro DB on Vercel
pubDate: 2024-04-22
description: Getting familiar with Lucia auth and the new Astro DB
tags: ['Astro', 'Lucia', 'Vercel']
isDraft: false
slug: lucia-auth-astro-db
---

Today I wanted to test out Astro's new DB feature as well as learn how the Lucia auth library works. I am relatively new to a bunch of this stuff so it took me a while jumping between various docs to get a working example. A linear walk-through with this specific stack would have helped, so I decided to write it up.

This is going to be very bare-bones demo of email and password verification. I will not implement real-world essentials like email verification and password resetting, or proper error handling. My goal is simply familiarisation myself with Lucia and Astro DB.

(I am also doing zero styling, and I won't be implementing any of the parts of a normal Astro project like decent base layouts.)

You should be able to follow along, I will try not to skip any steps. The relevant docs are the source of truth however, make sure to read them! There is a list of references at the end of the guide.

This guide is intended for someone who has a good handle on how Astro works, but you don't need to know anything about authentication or databases.

Here are the steps we will follow:

1. **Set up the Astro project and install dependencies**

   - Astro DB, Lucia, Astro DB adapter for Lucia, Vercel SSR adapter for Astro

2. **Create the database tables and test**

   - Set up types for Lucia, add tables to database schema, test the database actually works locally

3. **Build signup and test**

   - Connect the database and Lucia, add middleware, create signup endpoint and test locally

4. **Push minimal demo to cloud**

   - Link Astro DB to Astro Studio, deploy to Vercel

5. **Add a protected route**

   - Add a `/dashboard` protected page, login and logout endpoints, and deploy again

There's a lot to do - let's get going!

## Create an Astro project and add dependencies

Start by scaffolding a new Astro project with the friendly CLI wizard.

```bash
npm create astro@latest
```

Choose Typescript in strict mode, with an empty template. I named my project `astrodb-lucia-vercel`. (Also of course you should initialise git in the project, we aren't animals!)

Change into the new project directory and start the local dev server.

```bash
cd astrodb-lucia-vercel
npm run dev
```

Next install Astro DB to the project. We will come back and set up the required database tables later on.

```bash
npx astro add db
```

We also need Lucia and an additional library from Lucia called Oslo which has some convenience functions.

Lucia works with all kinds of databases through adapters. Astro DB doesn't have its own adapter included in the core project, but the Lucia creator does have a seperate adapter available so add that too.

```bash
npm install lucia oslo lucia-adapter-astrodb
```

Finally we need to turn our static Astro site into an SSR (server side rendered) project so that we can use Astro api endpoints and have dynamic routing. We will install the Vercel adaptor here as I found it is the most streamlined.

_(Cloudflare and Netlify have some complexity for accessing environment variables, and it looks like the default password hashing library for Lucia isn't compatible with Cloudflare workers.)_

```bash
npx astro add vercel
```

## Create our database and test locally

### General admin

After installing everything we have to attend to a little admin in some of our config files.

First make an adjustment to the `astro.config.mjs` in the project root to exclude the `astro:db` and `oslo` packages from the builder's optimisation stage. This will require adding the `vite` options in the exported config `json`.

```js
import { defineConfig } from 'astro/config';
import db from '@astrojs/db';

import vercel from '@astrojs/vercel/serverless';

export default defineConfig({
  integrations: [db()],
  vite: {
    optimizeDeps: {
      exclude: ['astro:db', 'oslo'],
    },
  },
  output: 'server',
  adapter: vercel(),
});
```

### Create the database tables

Astro DB sets up a database schema in the file `/db/config.ts`. For the purpose of authentication with Lucia we will need to create a `User` and a `Session` table in the schema.

Because we want to implement email and password auth, the `User` table will need fields for `email` and `hashed_passwords`. Note the `email` field in the table must be unique (for obvious reasons), and the `Session` table references sessions to users in the `User` table by the user `id`.

```ts
import { defineDb, defineTable, column } from 'astro:db';

const User = defineTable({
  columns: {
    id: column.text({ primaryKey: true }),
    email: column.text({ unique: true }),
    hashed_password: column.text(),
  },
});

const Session = defineTable({
  columns: {
    id: column.text({ primaryKey: true }),
    expiresAt: column.date(),
    userId: column.text({
      references: () => User.columns.id,
    }),
  },
});

export default defineDb({
  tables: {
    User,
    Session,
  },
});
```

Saving this file will automatically updates the database type definitions in the project. You can see the auto generated file in the project root at `.astro/db-types.d.ts`.

### Testing what we have so far

Before building anything else, lets just make sure that Astro DB is working locally and we can access the data in our dev environment.

Astro DB includes a helpful seed file where we can put mock table data for our dev environment. You can see this at `db/seed.ts`. Add the below fake users just so we can render it to the homepage quickly.

```ts
import { db, User } from 'astro:db';

export default async function () {
  await db.insert(User).values([
    { id: '123456', email: 'test1@email.com', hashed_password: '' },
    { id: '234567', email: 'test2@email.com', hashed_password: '' },
  ]);
}
```

Now on the homepage we can quickly map through the users after importing the `db` instance and querying the database in the frontmatter.

```jsx
// src/pages/index.astro
---
import { db, User } from 'astro:db'
const users = await db.select().from(User);
---
<html lang="en">
  <body>
    <h1>Astro DB</h1>
    {users.map(({ email }) => (
      <p>{email}</p>
    ))}
  </body>
</html>
```

If you still have the dev server running that should show up instantly, but in some cases I'd expect to have to give it a nudge and restart the server.

While we could test writing to the database as well at this point, it doesn't take much to get an api endpoint going with Lucia, so let's tackle that now.

## Connect Lucia and create a signup endpoint

### Create the Lucia init file and middleware

We need to set up the Lucia library and connect it to our database. I copied the examples from the Lucia docs with minor adjustment for the Astro DB adapter, creating the following file at `src/lib/auth.ts`.

```ts
import { Lucia } from 'lucia';
import { AstroDBAdapter } from 'lucia-adapter-astrodb';
import { db, User, Session } from 'astro:db';

const adapter = new AstroDBAdapter(db, Session, User);

export const lucia = new Lucia(adapter, {
  sessionCookie: {
    attributes: {
      secure: import.meta.env.PROD,
    },
  },
  getUserAttributes: (attributes) => {
    return {
      email: attributes.email,
    };
  },
});

declare module 'lucia' {
  interface Register {
    Lucia: typeof lucia;
    DatabaseUserAttributes: DatabaseUserAttributes;
  }
}

interface DatabaseUserAttributes {
  email: string;
}
```

Astro has built-in support for middleware, and we need that set up to intercept user requests and give Lucia access to the context object.

Create a middleware file at `src/middleware.ts` with the contents below. Note we import our `lucia` object which has the database adapter at the top of the file.

(Also check out the note about CSRF.)

```ts
import { lucia } from './lib/auth';
import { verifyRequestOrigin } from 'lucia';
import { defineMiddleware } from 'astro:middleware';

export const onRequest = defineMiddleware(async (context, next) => {
  if (context.request.method !== 'GET') {
    // NOTE: this first check can be removed if you enable the
    // experimental CSRF protection added in Astro 4.5
    // (https://astro.build/blog/astro-460/#experimental-support-for-csrf-protection)
    const originHeader = context.request.headers.get('Origin');
    const hostHeader = context.request.headers.get('Host');
    if (
      !originHeader ||
      !hostHeader ||
      !verifyRequestOrigin(originHeader, [hostHeader])
    ) {
      return new Response(null, {
        status: 403,
      });
    }
  }

  const sessionId = context.cookies.get(lucia.sessionCookieName)?.value ?? null;
  if (!sessionId) {
    context.locals.user = null;
    context.locals.session = null;
    return next();
  }

  const { session, user } = await lucia.validateSession(sessionId);
  if (session && session.fresh) {
    const sessionCookie = lucia.createSessionCookie(session.id);
    context.cookies.set(
      sessionCookie.name,
      sessionCookie.value,
      sessionCookie.attributes,
    );
  }
  if (!session) {
    const sessionCookie = lucia.createBlankSessionCookie();
    context.cookies.set(
      sessionCookie.name,
      sessionCookie.value,
      sessionCookie.attributes,
    );
  }
  context.locals.session = session;
  context.locals.user = user;
  return next();
});
```

We need to add types for the Lucia `Session` and `User`. Adjust the `/src/env.d.ts` file to include the following:

```ts
/// <reference path="../.astro/db-types.d.ts" />
/// <reference path="../.astro/types.d.ts" />
/// <reference types="astro/client" />

declare namespace App {
  interface Locals {
    session: import('lucia').Session | null;
    user: import('lucia').User | null;
  }
}
```

### Create a signup page and endpoint

Finally we can make something for the frontend! Sorry to disappoint but it will just be a plain html form...

Add a new page `src/pages/signup.astro` and create a form like below. You can see in the frontmatter we are declaring this to be a pre-rendered page rather than server rendered on each request.

```jsx
---
export const prerender = true;
---
<html lang="en">
  <body>
    <h1>Sign up</h1>
    <form method="post" action="/api/signup">
      <label for="email">Email</label>
      <input type="email" id="email" name="email" />
      <label for="password">Password</label>
      <input type="password" id="password" name="password" />
      <button>Continue</button>
    </form>
  </body>
</html>
```

As you can see the form makes a post request to the matching api endpoint. In Astro creating an endpoint is trivial. They can be located alongside our content anywhere in the pages directory, but by convention most people put them into an `api/` directory.

Create the first endpoint with a new file `src/pages/api/signup.ts`.

```ts
import { lucia } from '../../lib/auth';
import { generateId } from 'lucia';
import { Argon2id } from 'oslo/password';
import { db, User } from 'astro:db';

import type { APIContext } from 'astro';

export async function POST(context: APIContext): Promise<Response> {
  const formData = await context.request.formData();
  const email = (formData.get('email') as string).trim();

  if (
    typeof email !== 'string' ||
    email.length < 3 ||
    email.length > 255 ||
    !/.+@.+\..+/.test(email)
  ) {
    return new Response('Invalid email', {
      status: 400,
    });
  }
  const password = formData.get('password');

  if (
    typeof password !== 'string' ||
    password.length < 6 ||
    password.length > 255
  ) {
    return new Response('Invalid password', {
      status: 400,
    });
  }

  const userId = generateId(15);
  const hashedPassword = await new Argon2id().hash(password);

  // TODO: check if username is already used
  await db.insert(User).values({
    id: userId,
    email: email.toLowerCase(),
    hashed_password: hashedPassword,
  });

  const session = await lucia.createSession(userId, {});
  const sessionCookie = lucia.createSessionCookie(session.id);
  context.cookies.set(
    sessionCookie.name,
    sessionCookie.value,
    sessionCookie.attributes,
  );

  return context.redirect('/');
}
```

Thanks to the inclusion of Drizzle ORM with Astro DB this is all really straightforward. After doing some simple checks of the form data we create a new user in the database, and Lucia handles the heavy lifting creating a new session for the new user.

### Test our endpoint locally

With this endpoint in place, jump over to `localhost:4321/signup` and enter an email and password. If everything is correct you should be redirected back to the homepage where you will see your new user listed in addition to the mock data we started with.

Also check _Cookies_ in the _Application_ tab in dev tools where you will find a newly generated session cookie created by Lucia.

## Get everything remote

### Push to Astro Studio

Time to get the database into the cloud. The deep integration with Astro Studio makes this pretty seamless. The steps from the [Astro Studio docs](https://docs.astro.build/en/recipes/studio/) couldn't be easier, so just do that!

```bash
# Log in to Astro Studio with your GitHub account
npx astro login

# Link to a new project by following the prompts
npx astro link

# Push your local db configuration to the remote database
npx astro db push
```

You can now check out the project on the Astro Studio dashboard.

### Test deploy to Vercel

I like to deploy early and often so any issues with production environments get exposed sooner.

Install the Vercel CLI, then run the command to push the project to Vercel.

```bash
# Install the vercel cli globally
npm install -g vercel

# Run the vercel tool
vercel
```

By running the `vercel` command you will be prompted to authenticate via the web if you aren't already, then the CLI will set up the project with all the correct settings. (When asked `Want to override the settings? [y/N]`, choose `N`.)

**This attempt to deploy will not work, because we need to add the Astro Studio key to the project in Vercel.**

Grab a new key by creating a new App Token from the project settings on the Astro Studio dashboard - I named mine `vercel-key`.

Sign in to vercel.com and head to the newly created project. Tap on **Settings** > **Environment Variables** then add a new variable with the name `ASTRO_STUDIO_APP_TOKEN` and paste the token value from Astro studio.

Now back in your terminal run the `vercel` command once more and it will deploy the project again. (By default it will deploy to the Preview environment which is fine for our purposes, but you can use the command `vercel --prod` to push it to the Production environment.)

Enjoy the bare-bones demo you just created! Test the form at the `/signup` page with an email and password.

This time your actual production database in Astro Studio is updated with the new user in the `User` table.

Assuming things are working as expected Lucia also handled generating a session token in the `Session` table linked to the fake user you just created, and just like in our local environment we will have a matching browser cookie in dev tools.

Finally the pieces are coming together. We can get to making a protected route, a sign-in page, plus endpoints for sign-in and logout.

## Protected routes, sign-in and logout

### Create a Dashboard

Let's create a dashboard page that can only be accessed if the visitor is a `user`. We get access to the context local object within the frontmatter of Astro pages, so this is easy.

Create a new file at `src/pages/dashboard.astro`.

```tsx
---
const user = Astro.locals.user;

if (!user) {
  return Astro.redirect("/login");
}
---
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width" />
  </head>
  <body>
    <h1>Protected Dashboard</h1>
    <p>Welcome {user.email}</p>
    <form method="post" action="/api/logout">
    <button>Sign out</button>
    </form>
  </body>
</html>

```

Note we also pre-emptively included a logout button which calls a new API route. Let's add that now.

```ts
// src/pages/api/logout.ts

import { lucia } from '@lib/auth';
import type { APIContext } from 'astro';

export async function POST(context: APIContext): Promise<Response> {
  if (!context.locals.session) {
    return new Response(null, {
      status: 401,
    });
  }

  await lucia.invalidateSession(context.locals.session.id);

  const sessionCookie = lucia.createBlankSessionCookie();
  context.cookies.set(
    sessionCookie.name,
    sessionCookie.value,
    sessionCookie.attributes,
  );

  return context.redirect('/login');
}
```

It also makes sense to adjust the last line of the sign-up api endpoint to
redirect to this dashboard page on success.

```ts
// src/pages/api/signup.ts

// ...
  return context.redirect("/dashboard");
}
```

### Create the login page and endpoint

Obviously existing users should be able to sign in, so we need a page for that and an additional endpoint.

The login page is basically a copy of the signup page, I also included a link to the signup page below the form. Feel free to add a similar link on the signup page to this login page.

```tsx
---
export const prerender = true
---
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width" />
  </head>
  <body>
    <h1>Login</h1>
    <form method="post" action="/api/login">
      <label for="email">Email</label>
      <input type="email" id="email" name="email" />
      <label for="password">Password</label>
      <input type="password" id="password" name="password" />
      <button>Continue</button>
    </form>
	    <a href="/signup">Register</a>
  </body>
</html>
```

The endpoint for this follows a similar pattern to the signup endpoint, but one thing that should stand out is the use of the `eq` method from `astro:db` so we can query the User table for equivalence and retrieve the existing user.

```ts
import { lucia } from '../../lib/auth';
import { Argon2id } from 'oslo/password';
import { db, User, eq } from 'astro:db';
import type { APIContext } from 'astro';

export async function POST(context: APIContext): Promise<Response> {
  const formData = await context.request.formData();
  const email = (formData.get('email') as string).trim();

  if (
    typeof email !== 'string' ||
    email.length < 3 ||
    email.length > 255 ||
    !/.+@.+\..+/.test(email)
  ) {
    return new Response('Invalid email', {
      status: 400,
    });
  }

  const password = formData.get('password');

  if (
    typeof password !== 'string' ||
    password.length < 6 ||
    password.length > 255
  ) {
    return new Response('Invalid password', {
      status: 400,
    });
  }

  const existingUser = await db
    .select()
    .from(User)
    .where(eq(User.email, email.toLowerCase()))
    .get();

  if (!existingUser) {
    return new Response('Incorrect username or password', {
      status: 400,
    });
  }

  const validPassword = await new Argon2id().verify(
    existingUser.hashed_password,
    password,
  );

  if (!validPassword) {
    return new Response('Incorrect username or password', {
      status: 400,
    });
  }

  const session = await lucia.createSession(existingUser.id, {});
  const sessionCookie = lucia.createSessionCookie(session.id);
  context.cookies.set(
    sessionCookie.name,
    sessionCookie.value,
    sessionCookie.attributes,
  );

  return context.redirect('/dashboard');
}
```

### Update the homepage

We should change the homepage to remove the users list from the `User` table, and make it a static pre-rendered page.

```tsx
---
export const prerender = true
---
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width" />
  </head>
  <body>
    <h1>Astro DB</h1>
    <p>Welcome to a test of Lucia Auth</p>
    <a href='/dashboard'>Go to dashboard</>
  </body>
</html>
```

You should now be able to test the full signup → logout → login flow with a new user account in your dev environment.

Before calling it a day I need to test the production environment also works as expected, so finally lets run the `vercel` command one more time and get this live.

## Next steps

Overall I loved the experience of using both Astro DB and Lucia to get this working.

Further work is essential for a full-fledged auth solution in any production app.
Check out [The Copenhagen Book](https://thecopenhagenbook.com/) written by the creator of the Lucia library which describes many of the requirements.

I look forward to seeing how Astro DB develops. One thing I would need for my projects is the ability to add team members for databases in Astro Studio.

### References

- [lucia-auth.com](https://lucia-auth.com/)
- [thecopenhagenbook.com](https://thecopenhagenbook.com/)
- [github.com/lucia-auth/astro-example](https://github.com/lucia-auth/examples/tree/main/astro/username-and-password)
- [github.com/ElianCodes/lucia-astrodb](https://github.com/ElianCodes/lucia-astrodb)
- [github.com/pilcrowOnPaper/lucia-adapter-astrodb](https://github.com/pilcrowOnPaper/lucia-adapter-astrodb)
- [docs.astro.build/integrations-guide/vercel](https://docs.astro.build/en/guides/integrations-guide/vercel/)
- [docs.astro.build/guides/deploy/vercel](https://docs.astro.build/en/guides/deploy/vercel/)
- [docs.astro.build/guides/astro-db](https://docs.astro.build/en/guides/astro-db/)
- [docs.astro.build/recipes/studio](https://docs.astro.build/en/recipes/studio/)
