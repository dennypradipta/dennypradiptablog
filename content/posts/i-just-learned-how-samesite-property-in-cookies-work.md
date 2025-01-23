+++
title = 'I Just Learned How "SameSite" Property in Cookies Work'
date = 2025-01-23T21:43:34+07:00
images = ['/i-just-learned-how-samesite-property-in-cookies-work.webp']
+++

Happy New Year!

Recently, I have been busy with not one, but two Next.js projects and a Node.js backend built with Fastify. The setup is straightforward: one Next.js app serves as a dashboard for admins, while the other is a front-facing site for end-users. Just another day at the office.

Just like you imagined, all business logic, authentication, and data fetching, happens on the backend. Both Next.js apps rely heavily on the Fastify server to fetch data, manage user sessions, and basically handle everything behind the scenes.

As I was setting up authentication across these projects, I stumbled upon some problem.

## What the hell?

Before we getting deeper, let me paint you a picture of the current setup.

For this project, I’ve got everything running smoothly under the same domain, let's just say the domain is nicelydone.com. Here’s how it’s structured:

- The API is served at api.nicelydone.com.
- The frontend is served at nicelydone.com.
- The dashboard is served at dashboard.nicelydone.com.

The backend, built on Fastify, handles authentication via a function I wrote to set cookies when a user logs in. Here’s the function:

```typescript
function setTokenCookie({
  maxAgeSeconds,
  name,
  res,
  value,
}: SetTokenCookieParameters) {
  return res.setCookie(name, value, {
    httpOnly: true,
    secure: true,
    sameSite: "strict",
    path: "/",
    ...(config.server.cookie.useExplicitDomain
      ? {
          domain: config.server.cookie.domain,
        }
      : {}),
    maxAge: maxAgeSeconds,
  });
}
```

Same shit, different day. This function sets a cookie on the backend during user login, whether they’re logging in through basic email/password authentication or via OAuth2, in this case Gogole.

Now, here’s where it gets interesting: for plain old authentication, everything works like a charm. The user logs in, and their session gets reflected seamlessly on both the frontend (nicelydone.com) and the dashboard (dashboard.nicelydone.com).

But when it comes to Sign in via Google, it behaves differently. After successfully logging in with Google, the cookie is set—but neither the frontend nor the dashboard shows the current logged-in user immediately. Instead, I have to refresh the page ONCE before I can see the current logged-in user.

The first thing that comes to mind is that I forgot to set the Cookie domain. Nope, that’s not it. I made sure of that. I tried to convince myself this is just a localhost problem, but it also happens on the staging server.

## So I Keep Digging...

I tried to debug the issue by setting the `httpOnly` to false, because my undestanding is that Next.js can not access the cookie via client-side JavaScript. But then I remember I use Next.js App Router, so I am sure I used it via server-side. Pass.

Tried to set the `secure` to false, but then I remember I use HTTPS. Pass.

Tried to set the `sameSite` to `lax`. **IT WORKS!**

So I changed it to `lax`, pushes it to the staging server, and voila. After logging in with Google, the cookie is set, and the user is reflected on both the frontend and the dashboard. No more refreshes, no more headaches.

Now that makes me wonder, why is this happening? Without further ado, time to [RTFM](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#samesitesamesite-value).

## Why Use `sameSite: 'lax'`?

Let's take a look at the SameSite attribute. There are three available values: `Strict`, `Lax`, and `None`. Now these values behave differently.

1. `sameSite: 'strict'`

- Cookies are only sent for requests originating from the same site.
- Example: If your domain is nicelydone.com, the cookie will only be sent for requests to nicelydone.com and its subdomains, like dashboard.nicelydone.com.
- Drawback: Cross-origin requests, such as those initiated during an OAuth2 login flow (where Google redirects back to your site), won’t include the cookie. That’s why the logged-in state wasn’t reflected immediately after logging in with Google.

2. `sameSite: 'lax'`

- Cookies are sent with requests made from the same site and for “safe” cross-site requests, such as GET requests and certain top-level navigations (e.g., when Google redirects back to your site after OAuth2 login).
- Why It Works: When Google redirects the user back to your site (e.g., nicelydone.com), the browser sends the cookie along with the request because this redirect qualifies as a “safe” request. That’s why the frontend and dashboard reflected the user immediately after logging in with Google.
- Best Use Case: Authentication flows and scenarios where you need to support cross-origin navigation while still maintaining some security.

3. `sameSite: 'none'`

- Cookies are sent with all requests, including cross-origin ones.
- Caveat: This setting requires the secure flag to be enabled (the cookie must be sent over HTTPS). It’s the least restrictive option, but it’s also the riskiest because it leaves your cookies vulnerable to cross-site request forgery (CSRF) attacks.
- Best Use Case: Third-party cookies or when your app genuinely requires cookies to be sent with all types of cross-origin requests.

With that in mind, switching to `lax` solved the problem.

## Conclusion

By simply switching from `strict` to `lax`, I was able to ensure that the Sign in via Google works smoothly in the dashboard and the frontend. No more refreshes. Nada. Moral of the story? Always RTFM. You might just save yourself hours of frustration.

And remember, if all else fails, there’s always coffee. Lots of coffee.
