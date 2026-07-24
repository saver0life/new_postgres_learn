# Postgres/Supabase SQL — Practice Exercises & Solutions

Uses the `books` / `authors` / `reviews` schema from the main guide. Set it
up first if you haven't:

```sql
create table authors (
  id bigint generated always as identity primary key,
  name text not null unique,
  country text,
  birth_year int
);

create table books (
  id bigint generated always as identity primary key,
  title text not null,
  author_id bigint references authors(id),
  pages int check (pages > 0),
  published_at date,
  price numeric(6,2) default 0,
  metadata jsonb,
  tags text[]
);

create table reviews (
  id bigint generated always as identity primary key,
  book_id bigint references books(id) on delete cascade,
  rating int check (rating between 1 and 5),
  comment text,
  created_at timestamptz default now()
);

insert into authors (name, country, birth_year) values
  ('Frank Herbert', 'USA', 1920),
  ('Ursula K. Le Guin', 'USA', 1929),
  ('Liu Cixin', 'China', 1963);

insert into books (title, author_id, pages, published_at, price, tags) values
  ('Dune', 1, 412, '1965-08-01', 14.99, array['classic','space-opera']),
  ('Children of Dune', 1, 408, '1976-04-01', 12.99, array['classic','sequel']),
  ('The Left Hand of Darkness', 2, 304, '1969-03-01', 11.50, array['classic']),
  ('The Three-Body Problem', 3, 400, '2008-01-01', 13.25, array['modern','hard-sf']);

insert into reviews (book_id, rating, comment) values
  (1, 5, 'Masterpiece'),
  (1, 4, 'Dense but rewarding'),
  (3, 5, 'Beautiful prose'),
  (4, 4, 'Mind-bending');
```

---

## Level 1 — Basics

**1. List all books published before 1970, ordered by publish date.**

<details><summary>Solution</summary>

```sql
select title, published_at
from books
where published_at < '1970-01-01'
order by published_at;
```
</details>

**2. Find the average book price, rounded to 2 decimals.**

<details><summary>Solution</summary>

```sql
select round(avg(price), 2) as avg_price from books;
```
</details>

**3. List each author's name along with their number of books.**

<details><summary>Solution</summary>

```sql
select a.name, count(b.id) as num_books
from authors a
left join books b on b.author_id = a.id
group by a.name;
```
Use `left join` (not `join`) so authors with zero books still show, with count 0.
</details>

---

## Level 2 — Joins & aggregation

**4. Find each book's average rating and number of reviews, including books
with no reviews (show `null` or 0, not excluded).**

<details><summary>Solution</summary>

```sql
select b.title,
       count(r.id) as num_reviews,
       round(avg(r.rating), 2) as avg_rating
from books b
left join reviews r on r.book_id = b.id
group by b.title;
```
</details>

**5. Find the author whose books have the highest combined page count.**

<details><summary>Solution</summary>

```sql
select a.name, sum(b.pages) as total_pages
from authors a
join books b on b.author_id = a.id
group by a.name
order by total_pages desc
limit 1;
```
</details>

**6. List all books tagged `'classic'` (using the `tags` array column).**

<details><summary>Solution</summary>

```sql
select title from books where 'classic' = any(tags);
```
</details>

---

## Level 3 — Postgres-specific features

**7. For each author, show only their most expensive book (one row per author).**

<details><summary>Solution</summary>

```sql
select distinct on (a.id) a.name, b.title, b.price
from authors a
join books b on b.author_id = a.id
order by a.id, b.price desc;
```
`distinct on` keeps the first row per group according to the `order by` —
here, highest price first.
</details>

**8. Rank all books by price, and also show the overall average price next
to every row (window functions).**

<details><summary>Solution</summary>

```sql
select title, price,
       rank() over (order by price desc) as price_rank,
       round(avg(price) over (), 2) as overall_avg
from books;
```
</details>

**9. Using a CTE, find books priced above the overall average.**

<details><summary>Solution</summary>

```sql
with avg_price as (
  select avg(price) as v from books
)
select b.title, b.price
from books b, avg_price
where b.price > avg_price.v;
```
</details>

**10. Add a `genre` key to a book's `metadata` JSONB column, then query by it.**

<details><summary>Solution</summary>

```sql
update books set metadata = jsonb_set(coalesce(metadata, '{}'), '{genre}', '"sci-fi"')
where title = 'Dune';

select title from books where metadata->>'genre' = 'sci-fi';
```
</details>

**11. Upsert: insert a book, but if a book with the same `id` already exists,
update its price instead.**

<details><summary>Solution</summary>

```sql
insert into books (id, title, author_id, price)
values (1, 'Dune', 1, 16.99)
on conflict (id) do update set price = excluded.price;
```
</details>

---

## Level 4 — Supabase-flavored (RLS & auth)

Assume you add a `user_id uuid references auth.users(id)` column to `reviews`
so each review belongs to a logged-in user.

**12. Enable RLS on `reviews` so:**
- Everyone can read all reviews.
- Users can only insert reviews as themselves.
- Users can only update/delete their own reviews.

<details><summary>Solution</summary>

```sql
alter table reviews add column user_id uuid references auth.users(id);
alter table reviews enable row level security;

create policy "Public read"
on reviews for select
using (true);

create policy "Users insert own reviews"
on reviews for insert
with check (auth.uid() = user_id);

create policy "Users manage own reviews"
on reviews for update using (auth.uid() = user_id);

create policy "Users delete own reviews"
on reviews for delete using (auth.uid() = user_id);
```
Note `insert` policies use `with check`, while `select`/`update`/`delete`
use `using` — a common point of confusion.
</details>

**13. Write a trigger that automatically sets `reviews.user_id` to the
current authenticated user on insert, so the client never has to send it.**

<details><summary>Solution</summary>

```sql
create or replace function set_review_user()
returns trigger as $$
begin
  new.user_id = auth.uid();
  return new;
end;
$$ language plpgsql security definer;

create trigger before_review_insert
before insert on reviews
for each row execute function set_review_user();
```
</details>

---

## Level 5 — Challenge

**14. Full-text search: add a generated `tsvector` column over `title`, and
find all books matching the search term `"dark"`.**

<details><summary>Solution</summary>

```sql
alter table books add column search tsvector
  generated always as (to_tsvector('english', title)) stored;

create index idx_books_search on books using gin(search);

select title from books where search @@ to_tsquery('english', 'dark');
```
</details>

**15. Find authors who have written more than one book, along with a
comma-separated list of their book titles (string aggregation).**

<details><summary>Solution</summary>

```sql
select a.name, string_agg(b.title, ', ' order by b.published_at) as titles
from authors a
join books b on b.author_id = a.id
group by a.name
having count(b.id) > 1;
```
</details>

**16. Write a query returning, for every book, the previous book by the same
author by publish date (window function `lag`).**

<details><summary>Solution</summary>

```sql
select title, published_at,
       lag(title) over (partition by author_id order by published_at) as previous_book
from books;
```
</details>
