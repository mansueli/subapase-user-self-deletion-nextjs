# This is an extension on the [Nextjs-ts-user-management](https://github.com/supabase/supabase/tree/master/examples/user-management/nextjs-ts-user-management) to allow for user self-serve deletion. 

We included two different Edge Functions to allow for user self-deletion:

## User self deletion (HARD)

```js
import { serve } from 'https://deno.land/std@0.182.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2.14.0'
import { corsHeaders } from '../_shared/cors.ts'

console.log(`Function "user-self-deletion" up and running!`)

serve(async (req: Request) => {
  // This is needed if you're planning to invoke your function from a browser.
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders })
  }
  try {
    // Create a Supabase client with the Auth context of the logged in user.
    const supabaseClient = createClient(
      // Supabase API URL - env var exported by default.
      Deno.env.get('SUPABASE_URL') ?? '',
      // Supabase API ANON KEY - env var exported by default.
      Deno.env.get('SUPABASE_ANON_KEY') ?? '',
      // Create client with Auth context of the user that called the function.
      // This way your row-level-security (RLS) policies are applied.
      { global: { headers: { Authorization: req.headers.get('Authorization')! } } }
    )
    // Now we can get the session or user object
    const {
      data: { user },
    } = await supabaseClient.auth.getUser()
    // And we can run queries in the context of our authenticated user
    const { data: profiles, error: userError } = await supabaseClient.from('profiles').select('id, avatar_url')
    if (userError) throw userError
    const user_id = profiles[0].id
    const { data: list_of_files, error: storageError } = await supabaseClient.storage.from('avatars').list()
    if (storageError) throw storageError
    const file_urls = []
    for (let i = 0; i < list_of_files.length; i++) {
      file_urls.push(list_of_files[i].name)
    }
    // Create the admin client to delete files & user with the Admin API.
    const supabaseAdmin = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? ''
    )
    const { data: avatar_deletion, error: avatar_error } = await supabaseAdmin
      .storage
      .from('avatars')
      .remove(file_urls)
    if (avatar_error) throw avatar_error
    const { data: deletion_data, error: deletion_error } = await supabaseAdmin.auth.admin.deleteUser(user_id)
    if (deletion_error) throw deletion_error
    console.log("User & files deleted user_id: " + user_id)
    return new Response("User deleted: " + JSON.stringify(deletion_data, null, 2), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      status: 200,
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      status: 400,
    })
  }
})
```

## User self invalidation (soft-deletion)
```js
import { serve } from 'https://deno.land/std@0.182.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2.14.0'
import { corsHeaders } from '../_shared/cors.ts'

console.log(`Function "user-invalidation" up and running!`)

serve(async (req: Request) => {
  // This is needed if you're planning to invoke your function from a browser.
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders })
  }
  try {
    // Create a Supabase client with the Auth context of the logged in user.
    const supabaseClient = createClient(
      // Supabase API URL - env var exported by default.
      Deno.env.get('SUPABASE_URL') ?? '',
      // Supabase API ANON KEY - env var exported by default.
      Deno.env.get('SUPABASE_ANON_KEY') ?? '',
      // Create client with Auth context of the user that called the function.
      // This way your row-level-security (RLS) policies are applied.
      { global: { headers: { Authorization: req.headers.get('Authorization')! } } }
    )
    // Now we can get the session or user object
    const {
      data: { user },
    } = await supabaseClient.auth.getUser()
    // And we can run queries in the context of our authenticated user
    const { data: profiles, error: user_error } = await supabaseClient.from('profiles').select('id')
    if (user_error) throw user_error
    const user_id = profiles[0].id
    // Create the admin client to delete files & user with the Admin API.
    const supabaseAdmin = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? ''
    )
    const { data: invalidate_user, invalidation_error } = await supabaseAdmin.auth.admin.updateUserById(
      '6aa5d0d4-2a9f-4483-b6c8-0cf4c6c98ac4',
      {   
        email: user_id.concat('@deleted-users.example.com'), 
        phone: "", 
        user_metadata: { deleted: true },
        app_metadata:  { deleted: true }
      }
    )
    if (invalidation_error) throw invalidation_error
    const { data: invalidate_profile, invalid_profile_error } = await supabaseAdmin.from('profiles')
    .update({ full_name: '', username: '', avatar_url: '', website:''})
    .eq('id', user_id)
    if (invalid_profile_error) throw invalid_profile_error
    console.log('profile_invalidated:'+JSON.stringify(invalidate_profile, null, 2))
    return new Response("User invalidated: " + JSON.stringify(invalidate_user, null, 2), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      status: 200,
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      status: 400,
    })
  }
})

````
# Below you can see the readme from the original example:

## Supabase Next.js Auth & User Management Starter

This example will set you up for a very common situation: users can sign up or sign in and then update their account with public profile information, including a profile image.

This demonstrates how to use:

- User signups using Supabase [Auth](https://supabase.com/auth).
  - Supabase [Auth Helpers for Next.js](https://supabase.com/docs/guides/auth/auth-helpers/nextjs).
  - Supabase [pre-built Auth UI for React](https://supabase.com/docs/guides/auth/auth-helpers/auth-ui).
- User avatar images using Supabase [Storage](https://supabase.com/storage)
- Public profiles restricted with [Policies](https://supabase.com/docs/guides/auth#policies).
- Frontend using [Next.js](<[nextjs.org/](https://nextjs.org/)>).

## Technologies used

- Frontend:
  - [Next.js](https://github.com/vercel/next.js) - a React framework for production.
  - [Supabase.js](https://supabase.com/docs/library/getting-started) for user management and realtime data syncing.
  - Supabase [Auth Helpers for Next.js](https://supabase.com/docs/guides/auth/auth-helpers/nextjs).
  - Supabase [pre-built Auth UI for React](https://supabase.com/docs/guides/auth/auth-helpers/auth-ui).
- Backend:
  - [app.supabase.com](https://app.supabase.com/): hosted Postgres database with restful API for usage with Supabase.js.

## Instant deploy

The Vercel deployment will guide you through creating a Supabase account and project. After installation of the Supabase integration, all relevant environment variables will be set up so that the project is usable immediately after deployment 🚀.

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/git/external?repository-url=https%3A%2F%2Fgithub.com%2Fsupabase%2Fsupabase%2Ftree%2Fmaster%2Fexamples%2Fuser-management%2Fnextjs-ts-user-management&project-name=supabase-user-management&repository-name=supabase-user-management&demo-title=Supabase%20User%20Management&demo-description=An%20example%20web%20app%20using%20Supabase%20and%20Next.js&demo-url=https%3A%2F%2Fsupabase-nextjs-ts-user-management.vercel.app&demo-image=https%3A%2F%2Fi.imgur.com%2FZ3HkQqe.png&integration-ids=oac_jUduyjQgOyzev1fjrW83NYOv&external-id=nextjs-user-management)

## Build locally

Or create a new projec locally with `create-next-app`:

```bash
npx create-next-app -e with-supabase
```

### 1. Create new project

Sign up to Supabase - [https://app.supabase.com](https://app.supabase.com) and create a new project. Wait for your database to start.

### 2. Run "User Management" Quickstart

Once your database has started, head over to your project's `SQL Editor` and run the "User Management Starter" quickstart. On the `SQL editor` page, scroll down until you see `User Management Starter: Sets up a public Profiles table which you can access with your API`. Click that, then click `RUN` to execute that query and create a new `profiles` table. When that's finished, head over to the `Table Editor` and see your new `profiles` table.

### 3. Get the URL and Key

Go to the Project Settings (the cog icon), open the API tab, and find your API URL and `anon` key, you'll need these in the next step.

The `anon` key is your client-side API key. It allows "anonymous access" to your database, until the user has logged in. Once they have logged in, the keys will switch to the user's own login token. This enables row level security for your data. Read more about this [below](#postgres-row-level-security).

![image](https://user-images.githubusercontent.com/10214025/88916245-528c2680-d298-11ea-8a71-708f93e1ce4f.png)

**_NOTE_**: The `service_role` key has full access to your data, bypassing any security policies. These keys have to be kept secret and are meant to be used in server environments and never on a client or browser.

### 4. Env vars

Create a file in this folder `.env.local`

```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
```

Populate this file with your URL and Key.

### 5. Run the application

Run the application: `npm run dev`. Open your browser to `https://localhost:3000/` and you are ready to go 🚀.

## Supabase details

### Postgres Row level security

This project uses very high-level Authorization using Postgres' Role Level Security.
When you start a Postgres database on Supabase, we populate it with an `auth` schema, and some helper functions.
When a user logs in, they are issued a JWT with the role `authenticated` and their UUID.
We can use these details to provide fine-grained control over what each user can and cannot do.

This is a trimmed-down schema, with the policies:

```sql
-- Create a table for public profiles
create table profiles (
  id uuid references auth.users not null primary key on delete cascade,
  updated_at timestamp with time zone,
  username text unique,
  full_name text,
  avatar_url text,
  website text,
  constraint username_length check (char_length(username) >= 3)
);
-- Set up Row Level Security (RLS)
-- See https://supabase.com/docs/guides/auth/row-level-security for more details.
alter table profiles
  enable row level security;

create policy "Public profiles are viewable by everyone." on profiles
  for select using (true);

create policy "Users can insert their own profile." on profiles
  for insert with check (auth.uid() = id);

create policy "Users can update own profile." on profiles
  for update using (auth.uid() = id);

-- This trigger automatically creates a profile entry when a new user signs up via Supabase Auth.
-- See https://supabase.com/docs/guides/auth/managing-user-data#using-triggers for more details.
create function public.handle_new_user()
returns trigger as $$
begin
  insert into public.profiles (id, full_name, avatar_url)
  values (new.id, new.raw_user_meta_data->>'full_name', new.raw_user_meta_data->>'avatar_url');
  return new;
end;
$$ language plpgsql security definer;
create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure public.handle_new_user();

-- Set up Storage!
insert into storage.buckets (id, name)
  values ('avatars', 'avatars');

-- Set up access controls for storage.
-- See https://supabase.com/docs/guides/storage#policy-examples for more details.
create policy "Avatar images are publicly accessible." on storage.objects
  for select using (bucket_id = 'avatars');

create policy "Anyone can upload an avatar." on storage.objects
  for insert with check (bucket_id = 'avatars');

create policy "Anyone can update their own avatar." on storage.objects
  for update using ( auth.uid() = owner ) with check (bucket_id = 'avatars');
```

## More Supabase Examples & Resources

## Examples

These official examples are maintained by the Supabase team:

- [Next.js Subscription Payments Starter](https://github.com/vercel/nextjs-subscription-payments)
- [Next.js Slack Clone](https://github.com/supabase/supabase/tree/master/examples/slack-clone/nextjs-slack-clone)
- [Next.js 13 Data Fetching](https://github.com/supabase/supabase/tree/master/examples/caching/with-nextjs-13)
- [And more...](https://github.com/supabase/supabase/tree/master/examples)

## Other resources

- [[Docs] Next.js User Management Quickstart](https://supabase.com/docs/guides/getting-started/tutorials/with-nextjs)
- [[Egghead.io] Build a SaaS product with Next.js, Supabase and Stripe](https://egghead.io/courses/build-a-saas-product-with-next-js-supabase-and-stripe-61f2bc20)
- [[Blog] Fetching and caching Supabase data in Next.js 13 Server Components](https://supabase.com/blog/fetching-and-caching-supabase-data-in-next-js-server-components)

## Authors

- [Supabase](https://supabase.com)

Supabase is open source. We'd love for you to follow along and get involved at https://github.com/supabase/supabase

--
