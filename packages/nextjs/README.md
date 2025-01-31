# @supabase/auth-helpers-nextjs (BETA)

This submodule provides convenience helpers for implementing user authentication in Next.js applications.

## Installation

Using [npm](https://npmjs.org):

```sh
npm install @supabase/auth-helpers-nextjs

# Main components and hooks for React based frameworks (optional)
npm install @supabase/auth-helpers-react
```

Using [yarn](https://yarnpkg.com/):

```sh
yarn add @supabase/auth-helpers-nextjs

# Main components and hooks for React based frameworks (optional)
yarn add @supabase/auth-helpers-react
```

This library supports the following tooling versions:

- Node.js: `^10.13.0 || >=12.0.0`

- Next.js: `>=10`

## Getting Started

### Configuration

Set up the following env vars. For local development you can set them in a `.env.local` file. See an example [here](../../examples/nextjs/.env.local.example)).

```bash
# Find these in your Supabase project settings > API
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
```

### Basic Setup

Wrap your `pages/_app.js` component with the `SessionContextProvider` component:

```jsx
// pages/_app.js
import React, { useState }  from 'react';
import { useRouter } from 'next/router';
import { createBrowserSupabaseClient } from '@supabase/auth-helpers-nextjs';
import { SessionContextProvider } from '@supabase/auth-helpers-react';

function MyApp({ Component, pageProps }: AppProps) {
  const router = useRouter();
  const [supabaseClient] = useState(() => createBrowserSupabaseClient());

  return (
    <SessionContextProvider
      supabaseClient={supabaseClient}
      initialSession={pageProps.initialSession}
    >
      <button
        onClick={async () => {
          await supabaseClient.auth.signOut();
          router.push('/');
        }}
      >
        Logout
      </button>

      <Component {...pageProps} />
    </SessionContextProvider>
  );
}
```

You can now determine if a user is authenticated by checking that the `user` object returned by the `useUser()` hook is defined.

### Usage with TypeScript

You can pass types that were [generated with the Supabase CLI](https://supabase.com/docs/reference/javascript/next/typescript-support#generating-types) to the Supabase Client to get enhanced type safety and auto completion:

```js
// Creating a new supabase client object:
import { Database } from '../db_types';

const [supabaseClient] = useState(() =>
    createBrowserSupabaseClient<Database>()
  );
```

```js
// Retrieving a supabase client object from the SessionContext:
import { useSupabaseClient } from '@supabase/auth-helpers-react';
import { Database } from '../db_types';

const supabaseClient = useSupabaseClient<Database>();
```

## Client-side data fetching with RLS

For [row level security](https://supabase.com/docs/learn/auth-deep-dive/auth-row-level-security) to work properly when fetching data client-side, you need to make sure to use the `supabaseClient` from the `useSessionContext` hook and only run your query once the user is defined client-side in the `useUser()` hook:

```js
import { Auth, ThemeSupa } from '@supabase/auth-ui-react';
import { useUser, useSessionContext } from '@supabase/auth-helpers-react';
import { useEffect, useState } from 'react';

const LoginPage = () => {
  const { isLoading, session, error, supabaseClient } = useSessionContext();
  const user = useUser();
  const [data, setData] = useState();

  useEffect(() => {
    async function loadData() {
      const { data } = await supabaseClient.from('test').select('*');
      setData(data);
    }
    // Only run query once user is logged in.
    if (user) loadData();
  }, [user]);

  if (!user)
    return (
      <>
        {error && <p>{error.message}</p>}
        <Auth
          redirectTo="http://localhost:3000/"
          appearance={{ theme: ThemeSupa }}
          supabaseClient={supabaseClient}
          providers={['google', 'github']}
          socialLayout="horizontal"
        />
      </>
    );

  return (
    <>
      <button onClick={() => supabaseClient.auth.signOut()}>Sign out</button>
      <p>user:</p>
      <pre>{JSON.stringify(user, null, 2)}</pre>
      <p>client-side data fetching with RLS</p>
      <pre>{JSON.stringify(data, null, 2)}</pre>
    </>
  );
};

export default LoginPage;
```

### Server-side rendering (SSR) - withPageAuth

If you wrap your `getServerSideProps` with `withPageAuth` your props object will be augmented with the user object.

```js
// pages/profile.js
import { withPageAuth } from '@supabase/auth-helpers-nextjs';

export default function Profile({ user }) {
  return <div>Hello {user.name}</div>;
}

export const getServerSideProps = withPageAuth({ redirectTo: '/login' });
```

If there is no authenticated user, they will be redirect to your home page, unless you specify the `redirectTo` option.

You can pass in your own `getServerSideProps` method, the props returned from this will be merged with the
user props. You can also access the user session data by calling `getUser` inside of this method, eg:

```js
// pages/protected-page.js
import { withPageAuth } from '@supabase/auth-helpers-nextjs';

export default function ProtectedPage({ user, customProp }) {
  return <div>Protected content</div>;
}

export const getServerSideProps = withPageAuth({
  redirectTo: '/foo',
  async getServerSideProps(ctx, supabase) {
    // Access the user object
    const {
      data: { user }
    } = await supabase.auth.getUser();
    return { props: { email: user?.email } };
  }
});
```

### Server-side data fetching with RLS

Both `withApiAuth` and `withPageAuth` return a supabase client that you can use to run [row level security](https://supabase.com/docs/learn/auth-deep-dive/auth-row-level-security) authenticated queries server-side:

```js
import { User, withPageAuth } from '@supabase/auth-helpers-nextjs';

export default function ProtectedPage({
  user,
  data
}: {
  user: User,
  data: any
}) {
  return (
    <>
      <div>Protected content for {user.email}</div>
      <pre>{JSON.stringify(data, null, 2)}</pre>
      <pre>{JSON.stringify(user, null, 2)}</pre>
    </>
  );
}

export const getServerSideProps = withPageAuth({
  redirectTo: '/',
  async getServerSideProps(ctx, supabase) {
    // Run queries with RLS on the server
    const { data } = await supabase.from('test').select('*');
    return { props: { data } };
  }
});
```

### Server-side data fetching to OAuth APIs using `provider_token`

When using third-party auth providers, sessions are initiated with an additional `provider_token` field which is persisted as an HTTPOnly cookie upon logging in to enabled usage on the server side. The `provider_token` can be used to make API requests to the OAuth provider's API endpoints on behalf of the logged-in user. In the following example, we fetch the user's full profile from the third-party API during SSR using their id and auth token:

```js
import { User, withPageAuth } from '@supabase/auth-helpers-nextjs';

export default function ProtectedPage({
  user,
  allRepos
}: {
  user: User,
  allRepos: any
}) {
  return (
    <>
      <div>Protected content for {user.email}</div>
      <p>Data fetched with provider token:</p>
      <pre>{JSON.stringify(allRepos, null, 2)}</pre>
      <p>user:</p>
      <pre>{JSON.stringify(user, null, 2)}</pre>
    </>
  );
}

export const getServerSideProps = withPageAuth({
  redirectTo: '/',
  async getServerSideProps(ctx, supabase) {
    const {
      data: { session },
      error
    } = await supabase.auth.getSession();
    if (error) {
      throw error;
    }
    if (!session) {
      return { props: {} };
    }

    // Retrieve provider_token & logged in user's third-party id from metadata
    const { provider_token, user } = session;
    const userId = user.user_metadata.user_name;

    const allRepos = await (
      await fetch(
        `https://api.github.com/search/repositories?q=user:${userId}`,
        {
          method: 'GET',
          headers: {
            Authorization: `token ${provider_token}`
          }
        }
      )
    ).json();

    return { props: { allRepos, user } };
  }
});
```

## Protecting API routes

Wrap an API Route to check that the user has a valid session. If they're not logged in the handler will return a
401 Unauthorized.

```js
// pages/api/protected-route.js
import { withApiAuth } from '@supabase/auth-helpers-nextjs';

export default withApiAuth(async function ProtectedRoute(
  req,
  res,
  supabaseServerClient
) {
  // Run queries with RLS on the server
  const { data } = await supabaseServerClient.from('test').select('*');
  res.json(data);
});
```

If you visit `/api/protected-route` without a valid session cookie, you will get a 401 response.

## Protecting routes with [Nextjs Middleware](https://nextjs.org/docs/middleware)

As an alternative to protecting individual pages using `getServerSideProps` with `withPageAuth`, `withMiddlewareAuth` can be used from inside a `middleware` file to protect the entire directory or those that match the config object. In the following example, all requests to `/middleware-protected/*` will check whether a user is signed in, if successful the request will be forwarded to the destination route, otherwise the user will be redirected to `/login` (defaults to: `/`) with a 307 Temporary Redirect response status:

```ts
// middleware.ts
import { withMiddlewareAuth } from '@supabase/auth-helpers-nextjs';

export const middleware = withMiddlewareAuth({ redirectTo: '/login' });

export const config = {
  matcher: ['/middleware-protected/:path*']
};
```

It is also possible to add finer granularity based on the user logged in. I.e. you can specify a promise to determine if a specific user has permission or not.

```ts
// middleware.ts
import { withMiddlewareAuth } from '@supabase/auth-helpers-nextjs';

export const middleware = withMiddlewareAuth({
  redirectTo: '/login',
  authGuard: {
    isPermitted: async (user) => user.email?.endsWith('@example.com') ?? false,
    redirectTo: '/insufficient-permissions'
  }
});

export const config = {
  matcher: ['/middleware-protected/:path*']
};
```
