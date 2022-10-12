# Email Link Strategy - Remix Auth

> This strategy is heavily based on **kcd** strategy present in the [v2 of Remix Auth](https://github.com/sergiodxa/remix-auth/blob/v2.6.0/docs/strategies/kcd.md). The major difference being we are using `crypto-js` instead of `crypto` so that it can be deployed on CF.

The Email Link Strategy implements the authentication strategy used on [kentcdodds.com](https://kentcdodds.com).

This strategy uses passwordless flow with magic links. A magic link is a special URL generated when the user tries to login, this URL is sent to the user via email, after the click on it the user is automatically logged in.

You can read more about how this work in the [kentcdodds.com/how-i-built-a-modern-website-in-2021](https://kentcdodds.com/blog/how-i-built-a-modern-website-in-2021#authentication-with-magic-links).

## Supported runtimes

| Runtime    | Has Support |
| ---------- | ----------- |
| Node.js    | ✅          |
| Cloudflare | ✅          |

<!-- If it doesn't support one runtime, explain here why -->

## How to use

<!-- Explain how to use the strategy, here you should tell what options it expects from the developer when instantiating the strategy -->

## Setup

Because of how this strategy works you need a little bit more setup than other strategies, but nothing specially crazy.

### Email Service

You will need to have some email service configured in your application. What you actually use to send emails is not important, as far as you can create a function with this type:

```ts
type SendEmailOptions<User> = {
  emailAddress: string
  magicLink: string
  user?: User | null
  domainUrl: string
  form: FormData
}

type SendEmailFunction<User> = {
  (options: SendEmailOptions<User>): Promise<void>
}
```

So if you have something like `app/services/email-provider.server.ts` file exposing a generic function like `sendEmail` function receiving an email address, subject and body, you could use it like this:

```tsx
// app/services/email.server.tsx
import { renderToString } from 'react-dom/server'
import type { SendEmailFunction } from 'remix-auth-email-link'
import type { User } from '~/models/user.model'
import * as emailProvider from '~/services/email-provider.server'

export let sendEmail: SendEmailFunction<User> = async (options) => {
  let subject = "Here's your Magic sign-in link"
  let body = renderToString(
    <p>
      Hi {options.user?.name || 'there'},<br />
      <br />
      <a href={options.magicLink}>Click here to login on example.app</a>
    </p>
  )

  await emailProvider.sendEmail(options.emailAddress, subject, body)
}
```

Again, what you use as email provider is not important, you could use a third party service like [Mailgun](https://mailgun.com) or [Sendgrid](https://sendgrid.com), if you are using AWS you could use SES.

### Create the strategy instance

Now that you have your sendEmail email function you can create an instance of the Authenticator and the EmailLinkStrategy.

```ts
// app/services/auth.server.ts
import { Authenticator } from 'remix-auth'
import { EmailLinkStrategy } from 'remix-auth-email-link'
import { sessionStorage } from '~/services/session.server'
import { sendEmail } from '~/services/email.server'
import { User, getUserByEmail } from '~/models/user.model'

// This secret is used to encrypt the token sent in the magic link and the
// session used to validate someone else is not trying to sign-in as another
// user.
let secret = process.env.MAGIC_LINK_SECRET
if (!secret) throw new Error('Missing MAGIC_LINK_SECRET env variable.')

export let auth = new Authenticator<User>(sessionStorage)

// Here we need the sendEmail, the secret and the URL where the user is sent
// after clicking on the magic link
auth.use(
  new EmailLinkStrategy(
    { sendEmail, secret, callbackURL: '/magic' },
    // In the verify callback,
    // you will receive the email address, form data and whether or not this is being called after clicking on magic link
    // and you should return the user instance
    async ({
      email,
      form,
      magicLinkVerify,
    }: {
      email: string
      form: FormData
      magicLinkVerify: boolean
    }) => {
      let user = await getUserByEmail(email)
      return user
    }
  )
)
```

### Setup your routes

Now you can proceed to create your routes and do the setup.

```tsx
// app/routes/login.tsx
import {
  Form,
  LoaderFunction,
  ActionFunction,
  json,
  useLoaderData,
} from 'remix'
import { auth } from '~/services/auth.server'
import { sessionStorage } from '~/services/session.server'

export let loader: LoaderFunction = async ({ request }) => {
  await auth.isAuthenticated(request, { successRedirect: '/me' })
  let session = await sessionStorage.getSession(request.headers.get('Cookie'))
  // This session key `auth:magiclink` is the default one used by the EmailLinkStrategy
  // you can customize it passing a `sessionMagicLinkKey` when creating an
  // instance.
  return json({
    magicLinkSent: session.has('auth:magiclink'),
    magicLinkEmail: session.get('auth:email'),
  })
}

export let action: ActionFunction = async ({ request }) => {
  // The success redirect is required in this action, this is where the user is
  // going to be redirected after the magic link is sent, note that here the
  // user is not yet authenticated, so you can't send it to a private page.
  await auth.authenticate('email-link', request, {
    successRedirect: '/login',
    // If this is not set, any error will be throw and the ErrorBoundary will be
    // rendered.
    failureRedirect: '/login',
  })
}

// app/routes/login.tsx
export default function Login() {
  let { magicLinkSent, magicLinkEmail } =
    useLoaderData<{ magicLinkSent: boolean; magicLinkEmail?: string }>()

  return (
    <Form action="/login" method="post">
      {magicLinkSent ? (
        <p>
          Successfully sent magic link{' '}
          {magicLinkEmail ? `to ${magicLinkEmail}` : ''}
        </p>
      ) : (
        <>
          <h1>Log in to your account.</h1>
          <div>
            <label htmlFor="email">Email address</label>
            <input id="email" type="email" name="email" required />
          </div>
          <button>Email a login link</button>
        </>
      )}
    </Form>
  )
}
```

```tsx
// app/routes/magic.tsx
import { LoaderFunction, ActionFunction, json } from 'remix'
import { auth } from '~/services/auth.server'
import { sessionStorage } from '~/services/session.server'

export let loader: LoaderFunction = async ({ request }) => {
  await auth.authenticate('email-link', request, {
    // If the user was authenticated, we redirect them to their profile page
    // This redirect is optional, if not defined the user will be returned by
    // the `authenticate` function and you can render something on this page
    // manually redirect the user.
    successRedirect: '/me',
    // If something failed we take them back to the login page
    // This redirect is optional, if not defined any error will be throw and
    // the ErrorBoundary will be rendered.
    failureRedirect: '/login',
  })
}
```

```tsx
// app/routes/me.tsx
import { LoaderFunction, json } from 'remix'
import { auth } from '~/services/auth.server'

export let loader: LoaderFunction = async ({ request }) => {
  // If the user is here, it's already authenticated, if not redirect them to
  // the login page.
  let user = await auth.isAuthenticated(request, { failureRedirect: '/login' })
  return json({ user })
}

export default function Me() {
  let { user } = useLoaderData<{ user: User }>()
  return (
    <div>
      <h1>Welcome {user.name}</h1>
      <p>You are logged in as {user.email}</p>
    </div>
  )
}
```

## Email validation

The EmailLinkStrategy also supports email validation, this is useful if you want to prevent someone from signing-in with a disposable email address or you have some denylist of emails for some reason.

By default, the EmailStrategy will validate every email against the regular expression `/.+@.+/`, if it doesn't pass it will throw an error.

If you want to customize it you can create a function with this type and pass it to the EmailLinkStrategy.

```ts
type VerifyEmailFunction = {
  (email: string): Promise<void>
}
```

**Example**

```ts
// app/services/verifier.server.ts
import { VerifyEmailFunction } from 'remix-auth-email-link'
import { isEmailBurner } from 'burner-email-providers'
import isEmail from 'validator/lib/isEmail'

export let verifyEmailAddress: VerifyEmailFunction = async (email) => {
  if (!isEmail(email)) throw new Error('Invalid email address.')
  if (isEmailBurner(email)) throw new Error('Email not allowed.')
}
```

```ts
// app/services/auth.server.ts
import { Authenticator } from 'remix-auth'
import { Authenticator, EmailLinkStrategy } from 'remix-auth-email-link'
import { sessionStorage } from '~/services/session.server'
import { sendEmail } from '~/services/email.server'
import { User, getUserByEmail } from '~/models/user.model'
import { verifyEmailAddress } from '~/services/verifier.server'

// This secret is used to encrypt the token sent in the magic link and the
// session used to validate someone else is not trying to sign-in as another
// user.
let secret = process.env.MAGIC_LINK_SECRET
if (!secret) throw new Error('Missing MAGIC_LINK_SECRET env variable.')

let auth = new Authenticator<User>(sessionStorage)

// Here we need the sendEmail, the secret and the URL where the user is sent
// after clicking on the magic link
auth.use(
  new EmailLinkStrategy(
    { verifyEmailAddress, sendEmail, secret, callbackURL: '/magic' },
    // In the verify callback you will only receive the email address and you
    // should return the user instance
    async ({ email }: { email: string }) => {
      let user = await getUserByEmail(email)
      return user
    }
  )
)
```

## Options options

The EmailLinkStrategy supports a few more optional configuration options you can set. Here's the whole type with each option commented.

```ts
type EmailLinkStrategyOptions<User> = {
  /**
   * The endpoint the user will go after clicking on the email link.
   * A whole URL is not required, the pathname is enough, the strategy will
   * detect the host of the request and use it to build the URL.
   * @default "/magic"
   */
  callbackURL?: string
  /**
   * A function to send the email. This function should receive the email
   * address of the user and the URL to redirect to and should return a Promise.
   * The value of the Promise will be ignored.
   */
  sendEmail: SendEmailFunction<User>
  /**
   * A function to validate the email address. This function should receive the
   * email address as a string and return a Promise. The value of the Promise
   * will be ignored, in case of error throw an error.
   *
   * By default it only test the email against the RegExp `/.+@.+/`.
   */
  verifyEmailAddress?: VerifyEmailFunction
  /**
   * A secret string used to encrypt and decrypt the token and magic link.
   */
  secret: string
  /**
   * The name of the form input used to get the email.
   * @default "email"
   */
  emailField?: string
  /**
   * The param name the strategy will use to read the token from the email link.
   * @default "token"
   */
  magicLinkSearchParam?: string
  /**
   * How long the magic link will be valid. Default to 30 minutes.
   * @default 1_800_000
   */
  linkExpirationTime?: number
  /**
   * The key on the session to store any error message.
   * @default "auth:error"
   */
  sessionErrorKey?: string
  /**
   * The key on the session to store the magic link.
   * @default "auth:magicLink"
   */
  sessionMagicLinkKey?: string
  /**
   * Add an extra layer of protection and validate the magic link is valid.
   * @default false
   */
  validateSessionMagicLink?: boolean

  /**
   * The key on the session to store the email.
   * It's unset the same time the sessionMagicLinkKey is.
   * @default "auth:email"
   */
  sessionEmailKey?: string
}
```
