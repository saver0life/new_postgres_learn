# Generating a 100k-Row Dataset in Supabase

This uses Postgres's `generate_series` to create realistic-ish fake data
directly in SQL — no external tools needed. Paste this into the SQL Editor.

Good for practicing `explain analyze`, indexing, and pagination on data
that's actually big enough to show performance differences.

---

## 1. Schema (skip if you already ran the exercises setup)

```sql
create table if not exists authors (
  id bigint generated always as identity primary key,
  name text not null,
  country text,
  birth_year int
);

create table if not exists books (
  id bigint generated always as identity primary key,
  title text not null,
  author_id bigint references authors(id),
  pages int check (pages > 0),
  published_at date,
  price numeric(6,2) default 0,
  metadata jsonb,
  tags text[]
);

create table if not exists reviews (
  id bigint generated always as identity primary key,
  book_id bigint references books(id) on delete cascade,
  rating int check (rating between 1 and 5),
  comment text,
  created_at timestamptz default now()
);
```

---

## 2. Generate 2,000 authors

```sql
insert into authors (name, country, birth_year)
select
  'Author ' || i,
  (array['USA','UK','China','India','Nigeria','Brazil','Japan','France'])[1 + floor(random()*8)],
  1900 + floor(random()*100)::int
from generate_series(1, 2000) as i;
```

## 3. Generate 100,000 books

```sql
insert into books (title, author_id, pages, published_at, price, tags)
select
  'Book Title ' || i,
  1 + floor(random() * 2000)::bigint,
  50 + floor(random() * 600)::int,
  (date '1950-01-01' + (floor(random() * 27000))::int),
  round((5 + random() * 45)::numeric, 2),
  array[
    (array['classic','modern','sci-fi','fantasy','romance','thriller','non-fiction'])[1 + floor(random()*7)::int],
    (array['bestseller','award-winner','debut','sequel'])[1 + floor(random()*4)::int]
  ]
from generate_series(1, 100000) as i;
```

## 4. Generate ~300,000 reviews (a few per book)

```sql
insert into reviews (book_id, rating, comment)
select
  1 + floor(random() * 100000)::bigint,
  1 + floor(random() * 5)::int,
  (array['Loved it','Not for me','Solid read','Would recommend','Meh','Life-changing'])[1 + floor(random()*6)::int]
from generate_series(1, 300000) as i;
```

## 5. Sanity check

```sql
select
  (select count(*) from authors) as authors,
  (select count(*) from books) as books,
  (select count(*) from reviews) as reviews;
```

Expect roughly: 2,000 authors / 100,000 books / 300,000 reviews.
(This whole thing should take well under a minute on Supabase's free tier.)

---

## 6. Now the point: see performance actually matter

**Before indexing:**
```sql
explain analyze select * from books where author_id = 42;
```
Look for `Seq Scan` in the output — it's scanning all 100k rows.

**Add the index:**
```sql
create index idx_books_author_id on books(author_id);
```

**Re-run the same query:**
```sql
explain analyze select * from books where author_id = 42;
```
Now look for `Index Scan` and a much lower `actual time`.

**Try a couple more:**
```sql
-- Aggregation across the join — good candidate for testing index on reviews.book_id
explain analyze
select b.title, avg(r.rating)
from books b join reviews r on r.book_id = b.id
where b.id = 555
group by b.title;

create index idx_reviews_book_id on reviews(book_id);

-- Full-text-ish filter on JSON/array
explain analyze select * from books where 'sci-fi' = any(tags);

create index idx_books_tags on books using gin(tags);
```

## 7. Cleanup (if you want to reset)

```sql
truncate reviews, books, authors restart identity cascade;
```

---

### Notes
- `generate_series` + `random()` is the standard Postgres trick for quick
  synthetic data — no need for external seed/faker libraries for this scale.
- Supabase's free tier database (500MB–1GB depending on plan) handles this
  dataset comfortably; you'll only run into limits generating millions of rows.
- If inserts feel slow in the SQL Editor UI, that's normal for 100k+ row
  bulk inserts over the dashboard's connection — it should still complete,
  just give it a moment.
