## Cross reference Netflix shows with IMDB ratings

I wanted to cross reference Netflix shows with IMDB to see what's what. I know there are browser extensions to do this in a way, but I wanted to see it all at once.

### Netflix shows
First I needed a list of shows. I already had that from a different side project - you just need to go to a listing - here's [a subsite with all shows with given subtitles](https://www.netflix.com/browse/subtitles/cs). You scroll all the way to the bottom (while waiting for each new page to be downloaded) and once you reach the end, you open the developer console and run

```javascript
document.querySelectorAll('div.rowContainer div.fallback-text').forEach(x => console.log(x.textContent))
```

This gives you a list of shows listed on the current page (given current DOM, this may change in the future). Copy and paste this from the console, clean it up a little and save it as a line-delimited list.

From then on, it's up to you, what you want to do. Here's how you get it to Postgres.

### Postgres tables

```sql
create schema if not exists netflix;

drop table if exists netflix.shows;
create table netflix.shows (
	title varchar,
	primary key(title)
);

drop table if exists netflix.imdb_ratings;
create table netflix.imdb_ratings (
	tconst varchar not null,
	averageRating numeric(4,2) not null,
	numVotes int not null,
	primary key(tconst)
);

drop table if exists netflix.imdb_title_akas;
create table netflix.imdb_title_akas (
	titleId varchar not null, 
	ordering int not null,
	title varchar not null,
	region varchar, 
	language varchar, 
	types varchar, 
	attributes varchar,
	isOriginalTitle bool,
	primary key(titleid, ordering)
);

create index imdb_title on netflix.imdb_title_akas(title);
```

### IMDB data

IMDB offers some of its data for *non-commercial use*, it's quite easy to get it in the database. It's tab delimited and uses `\N` for nulls.

```sh
curl -O https://datasets.imdbws.com/title.akas.tsv.gz
curl -O https://datasets.imdbws.com/title.ratings.tsv.gz

gunzip -c title.ratings.tsv.gz | tail -n +2 | psql -c "copy netflix.imdb_ratings from stdin delimiter E'\t' NULL '\N'"
gunzip -c title.akas.tsv.gz | tail -n +2 | psql -c "copy netflix.imdb_title_akas from stdin delimiter E'\t' NULL '\N'"
```

Then you insert all your Netflix shows info

```sh
cat shows.txt | psql -c 'copy netflix.shows from stdin'
```

And you're all done.

### Basic joins

There are tons of way you can combine these datasets, I chose this trivial exact lookup based on the show's name, breaking conflicts by the number of votes on IMDB.

```sql
with imdb_data as (
	select titleid, ordering, title, averagerating, numvotes from netflix.imdb_title_akas ttl
	inner join netflix.imdb_ratings rt on rt.tconst = ttl.titleid
),
joined as (	
	select
	*,
	row_number() over(partition by title order by numvotes desc) as rn
	from netflix.shows
	left join imdb_data using(title)
)
select * from joined where rn = 1
```

You could also choose some fuzzy lookups, perhaps using n-grams, you just index the given columns using [`pg_trgm`](https://www.postgresql.org/docs/11/pgtrgm.html) and off you go, you can search by similarity.
