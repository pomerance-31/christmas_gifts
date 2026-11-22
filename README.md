# Family Christmas

A small private site where a family shares Christmas gift ideas for each other. Everyone has their own login. You can see and add ideas for everyone else, but you never see what others posted for you, or whether anything was bought, so surprises stay safe.

## How it works

- **My list** shows only the ideas you posted for yourself.
- **Other people's lists** show every idea for that person. Each idea has a bought checkbox (only the buyer can untick it), a "Posted by" line, a 👍 button and a comment thread. Ideas sort by 👍 count.
- **Shared ideas** can be for more than one person. They show on a Shared tab and on each recipient's tab with a "For A + B" line. Nobody they are for can see them.
- **Add an idea** with a name (required), link and notes. Tick one or more people. While you type, the form flags similar ideas already on the list.
- **Quick add** takes a casual note like "board game for Sam and Alex, they love co-op games". The page asks Claude Haiku to fill in the form, and you still click "Add idea" yourself.
- **Categories:** on a person's tab and the Shared tab, a sidebar lists AI generated categories for that list (Books, Clothing and so on) with counts, and clicking one filters the list. On a phone it becomes a row of chips.
- **Editing and deleting** are limited to ideas you posted. You can change the name, link and notes but not who it is for.
- You can't like or comment on ideas you posted for yourself.

The "never see what's for you" rule is enforced by the database, not only the page.

## Stack

- One static page, `index.html` (HTML, CSS and JS together, no build step), hosted on GitHub Pages from the `main` branch.
- [Supabase](https://supabase.com) free tier for logins, Postgres and row level security.
- One Supabase Edge Function, `parse-gift`, which calls the Anthropic API for Quick add.
- supabase-js v2 from the jsDelivr CDN, fonts from Google Fonts.

```
index.html                              the whole app
supabase/functions/parse-gift/index.ts  Edge Function for Quick add
README.md                               this file
```

The Supabase URL and publishable key sit at the top of the script in `index.html`. The publishable key is safe to expose because the database rules do the protecting.

## Setup

1. **Supabase project.** Create one, then run the SQL below in the SQL editor. Add one row to `profiles` per family member (name, email). Each person's email must match their login.
2. **Logins.** Turn off "Allow new users to sign up" under Authentication > Sign In / Providers. Create each user with "Create new user" and Auto Confirm and a temporary password, or send invites. On first sign in a person sets a new password and picks an icon.
3. **Page config.** Put your project URL and publishable key in `index.html` (`SUPABASE_URL`, `SUPABASE_KEY`).
4. **GitHub Pages.** Serve the `main` branch root.
5. **Quick add (optional).**
   - Create an Anthropic API key, add a few dollars of credit and set a low monthly spend limit.
   - In Supabase, add the secret `ANTHROPIC_API_KEY` under Edge Functions > Secrets.
   - Deploy `supabase/functions/parse-gift/index.ts` as a function named exactly `parse-gift`, and turn Verify JWT off for it. The function checks the login itself, and the gateway check can reject valid tokens on projects with the new signing keys.

Supabase's built-in email sender allows only about 2 emails an hour, so temporary passwords are easier than invites for the first round.

## Database

A signed-in user is matched to a `profiles` row by lowercase email through `my_profile_id()`. Every table is closed to `anon`, then opened narrowly with grants plus row level security.

<details>
<summary>Full SQL</summary>

```sql
create table profiles (
  id uuid primary key default gen_random_uuid(),
  name text not null unique,
  email text not null unique,
  icon text
);

create table gifts (
  id bigint generated always as identity primary key,
  recipient_ids uuid[] not null check (cardinality(recipient_ids) between 1 and 12),
  category text check (category is null or char_length(category) <= 40),
  posted_by uuid not null references profiles(id),
  name text not null,
  link text,
  notes text,
  created_at timestamptz not null default now()
);

create table purchases (
  gift_id bigint primary key references gifts(id) on delete cascade,
  buyer_id uuid not null references profiles(id),
  created_at timestamptz not null default now()
);

create table votes (
  gift_id bigint not null references gifts(id) on delete cascade,
  voter_id uuid not null references profiles(id),
  created_at timestamptz not null default now(),
  primary key (gift_id, voter_id)
);

create table comments (
  id bigint generated always as identity primary key,
  gift_id bigint not null references gifts(id) on delete cascade,
  author_id uuid not null references profiles(id),
  body text not null check (char_length(body) between 1 and 500),
  created_at timestamptz not null default now()
);

create table comment_likes (
  comment_id bigint not null references comments(id) on delete cascade,
  liker_id uuid not null references profiles(id),
  created_at timestamptz not null default now(),
  primary key (comment_id, liker_id)
);

create function my_profile_id() returns uuid
language sql stable security definer set search_path = public as $$
  select id from profiles where lower(email) = lower(auth.jwt() ->> 'email')
$$;

revoke all on profiles, gifts, purchases, votes, comments, comment_likes from anon, authenticated;
grant select on profiles to authenticated;
grant update (icon) on profiles to authenticated;
grant select, insert, delete on gifts, purchases, votes, comments, comment_likes to authenticated;
grant update (name, link, notes) on gifts to authenticated;

alter table profiles enable row level security;
alter table gifts enable row level security;
alter table purchases enable row level security;
alter table votes enable row level security;
alter table comments enable row level security;
alter table comment_likes enable row level security;

-- lets the categorize call fill in an empty category on an idea the caller can see
create function set_gift_category(gid bigint, cat text) returns void
language sql security definer set search_path = public as $$
  update gifts set category = left(trim(cat), 40)
  where id = gid and category is null
    and not (my_profile_id() = any(recipient_ids));
$$;
revoke all on function set_gift_category(bigint, text) from public, anon;
grant execute on function set_gift_category(bigint, text) to authenticated;

-- profiles
create policy "read profiles" on profiles for select to authenticated using (true);
create policy "update own profile" on profiles for update to authenticated
  using (id = my_profile_id()) with check (id = my_profile_id());

-- gifts: hidden from the people they are for, unless you posted them
create policy "see gifts" on gifts for select to authenticated
  using (not (my_profile_id() = any(recipient_ids)) or posted_by = my_profile_id());
create policy "post gifts" on gifts for insert to authenticated
  with check (
    posted_by = my_profile_id()
    and cardinality(recipient_ids) = (select count(*) from profiles where id = any(recipient_ids))
  );
create policy "edit own gifts" on gifts for update to authenticated
  using (posted_by = my_profile_id()) with check (posted_by = my_profile_id());
create policy "remove own gifts" on gifts for delete to authenticated
  using (posted_by = my_profile_id());

-- purchases
create policy "see purchases" on purchases for select to authenticated
  using (exists (select 1 from gifts g where g.id = gift_id and not (my_profile_id() = any(g.recipient_ids))));
create policy "mark bought" on purchases for insert to authenticated
  with check (
    buyer_id = my_profile_id()
    and exists (select 1 from gifts g where g.id = gift_id and not (my_profile_id() = any(g.recipient_ids)))
  );
create policy "unmark own" on purchases for delete to authenticated
  using (buyer_id = my_profile_id());

-- votes
create policy "see votes" on votes for select to authenticated
  using (exists (select 1 from gifts g where g.id = gift_id and not (my_profile_id() = any(g.recipient_ids))));
create policy "add vote" on votes for insert to authenticated
  with check (voter_id = my_profile_id()
    and exists (select 1 from gifts g where g.id = gift_id and not (my_profile_id() = any(g.recipient_ids))));
create policy "remove own vote" on votes for delete to authenticated
  using (voter_id = my_profile_id());

-- comments
create policy "see comments" on comments for select to authenticated
  using (exists (select 1 from gifts g where g.id = gift_id and not (my_profile_id() = any(g.recipient_ids))));
create policy "add comment" on comments for insert to authenticated
  with check (author_id = my_profile_id()
    and exists (select 1 from gifts g where g.id = gift_id and not (my_profile_id() = any(g.recipient_ids))));
create policy "remove own comment" on comments for delete to authenticated
  using (author_id = my_profile_id());

-- comment likes
create policy "see comment likes" on comment_likes for select to authenticated
  using (exists (select 1 from comments c join gifts g on g.id = c.gift_id
    where c.id = comment_id and not (my_profile_id() = any(g.recipient_ids))));
create policy "add comment like" on comment_likes for insert to authenticated
  with check (liker_id = my_profile_id()
    and exists (select 1 from comments c join gifts g on g.id = c.gift_id
      where c.id = comment_id and not (my_profile_id() = any(g.recipient_ids))));
create policy "remove own comment like" on comment_likes for delete to authenticated
  using (liker_id = my_profile_id());
```

</details>

## Quick add (`parse-gift`)

1. Checks the caller is signed in, using the key from the request's `apikey` header.
2. Reads `profiles` with the caller's token, so the family names and the current user come from the server and the page can't change them. A caller whose email isn't in `profiles` is refused.
3. Caps the note at 1000 characters and strips `<` and `>`.
4. Calls the Anthropic Messages API with a forced tool call, `fill_gift_form`, and an enum of the family names for the recipients.
5. Checks the result before returning it. Recipients must be family names, a link is kept only if it appears in the note, em dashes are replaced and lengths are capped.
6. Returns `{ recipients, gift, link, notes, extra_ideas, problem }`. The `problem` text is built in code, not by the model.

## Categories (`parse-gift`, mode `categorize`)

When a person's tab or the Shared tab opens and some ideas have no category yet, the page sends the list's idea ids to `parse-gift` with `mode: "categorize"`.

1. The function reads those ideas with the caller's login, so it only sees ideas the caller is allowed to see.
2. Only ideas without a category go to the model, along with the categories already used on that list and a starter list. Existing categories are reused first, so the same list keeps the same categories.
3. Each answer is saved through `set_gift_category`, which only fills empty categories and never overwrites one.
4. The function returns what is stored, so everyone sees the same categories. If the call fails, the sidebar stays hidden.

## Things to know

- Sign in work in `onAuthStateChange` is deferred with `setTimeout`. Awaiting Supabase calls inside that callback deadlocks.
- The `hidden` CSS class needs `!important`, because ID selectors beat class selectors.
- The password reset flag is read from the URL hash at page load, before Supabase clears it.
- Names and emails come from the database at runtime. Keep family names, the surname and email addresses out of the repo, and don't commit SQL that holds them (`*.sql` is in `.gitignore`).

## Ideas for later

- Fetch a product title from a pasted link.
- Custom SMTP in Supabase to lift the email limit.
- A per-person daily cap on Quick add calls.
