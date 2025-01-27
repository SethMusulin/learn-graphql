---
title: "Setup Remix Root for Auth and CSRF"
metaTitle: "Setup Remix Root for Auth and CSRF | Remix Fullstack GraphQL Tutorial"
metaDescription: "Here we setup the root component of our app for auth and CSRF protection"
---

import GithubLink from "../../src/GithubLink.js";

<GithubLink link="https://github.com/hasura/learn-graphql/blob/master/tutorials/frontend/remix-firebase/app-final/app/root.tsx" text="root.tsx" />

In root.tsx we setup our [actions](https://remix.run/docs/en/v1/guides/data-writes) and [loaders](https://remix.run/docs/en/v1/guides/data-loading)

```typescript
import {
  json,
  Links,
  LiveReload,
  Meta,
  Outlet,
  Scripts,
  ScrollRestoration,
  LoaderFunction,
  useLoaderData,
  ActionFunction,
  redirect,
  Form,
} from "remix";
import {
  AuthenticityTokenInput,
  AuthenticityTokenProvider,
  createAuthenticityToken,
  useShouldHydrate,
} from "remix-utils";
import { commitSession } from "~/utils/sessions.server";
import { getSessionData } from "./utils/auth.server";

interface LoaderData {
  csrf?: string;
  isLoggedIn: boolean;
}

// Setup CSRF token only if they are heading to the login page.
// Usually we would assign a CSRF token to everyone but
// with Firebase Hosting caching is tied to the cookie.
// If someone isn't logged in we want them to hit the public cache
// https://firebase.google.com/docs/hosting/manage-cache
export const action: ActionFunction = async ({ request }) => {
  let { session } = await getSessionData(request);

  // Add CSRF token to session
  createAuthenticityToken(session);

  return redirect("/login", {
    headers: { "Set-Cookie": await commitSession(session) },
  });
};

export const loader: LoaderFunction = async ({ request }) => {
  const { csrf, idToken } = await getSessionData(request);
  return json<LoaderData>({ csrf, isLoggedIn: !!idToken });
};
```

Our root App component is the standard Remix one except we conditionally load Javascript and add a nav bar with login/logout. Throughout the app we make heavy use of the remix-utils npm library.

We use forms to POST to the our action endpoints

```typescript
export default function App() {
  // Control if page loads JS https://github.com/sergiodxa/remix-utils#useshouldhydrate
  const shouldHydrate = useShouldHydrate();

  const fetcher = useFetcher();

  const { csrf, isLoggedIn } = useLoaderData<LoaderData>();

  return (
    <html lang="en">
      <head>
        <meta charSet="utf-8" />
        <meta name="viewport" content="width=device-width,initial-scale=1" />
        <meta name="robots" content="noindex" />
        <Meta />
        <Links />
      </head>
      <body
        style={{
          margin: 0,
        }}
      >
        <AuthenticityTokenProvider token={csrf || ""}>
          <nav
            style={{
              display: "flex",
              justifyContent: "space-between",
              alignItems: "center",
            }}
          >
            <a href="/">Home</a>
            {/* We use fetcher.Form instead of Form because we dont want navigation events */}
            {isLoggedIn ? (
              <fetcher.Form action="/logout" method="post" replace>
                <AuthenticityTokenInput />
                <button type="submit">Logout</button>
              </fetcher.Form>
            ) : (
              <fetcher.Form action="/" method="post" replace>
                <button type="submit">Login</button>
              </fetcher.Form>
            )}
          </nav>
          <Outlet />
        </AuthenticityTokenProvider>
        <ScrollRestoration />
        {shouldHydrate && <Scripts />}
        {process.env.NODE_ENV === "development" && <LiveReload />}
      </body>
    </html>
  );
}
```
