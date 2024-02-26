---
title: "Correct Usage of Sql Transaction in Postgres From Go"
date: 2019-05-21T23:02:31+02:00
image: "img/postgres-flicker.jpg"
draft: false
---

## Problem

Once upon a time I got a task to merge duplicate URLs in our production database. It turned out that there were a lot of same urls like `https://example.com?fbclid=1234` and `https://example.com?fbclid=5678` we wanted to merge.  Code to normalize URL was easy
to develop. Code to migrate the database looked easy as weel. Until I turned the transaction on. Then very cryptic message appeared


{{< highlight auto >}}
 pq: unexpected Parse response 'C'
{{< / highlight >}}

The error message was telling me *nothing*. And what is worse [ducking](https://duck.com) did not revealed anything. Except notes about Postgres protocol internals., which was something I did not wanted to dig into.  It was a hint from [@karolhrdina](https://github.com/karolhrdina) who have explained me the root cause well enough, so I can share my experience in my blog. Lessons learned - always work with a great colleagues, you can learn a lot from them!

Following text assume reader know about Go, particularly `database/sql` and `lib/pq`,  Postgres or SQL databases in general.

## The code

Following snippet omits a lot of details, but shows the algorithm

{{< highlight go >}}
rows, err := tx.Query("SELECT id, url, visists FROM schema.url WHERE url LIKE $1 ORDER BY id", pattern)

for rows.Next() {
        err = rows.Scan(&id, &url, &visits)
        urlNormalized := URLNormalize(url)
        if url != urlNormalized {
                _, err = tx.Exec("INSERT INTO schema.url AS u (url, visits) VALUES ($1, 1) ON CONFLICT url DO UPDATE SET visists=u.visists+$2;", urlNormalized, vists)
                _, err = tx.Exec("DELETE FROM schema.url WHERE id=$1", id)
        }
}
{{< / highlight >}}

1. For all urls matching the bad pattern do
2. Try to normalize and if there is normalized version then
3. UPSERT new entry or increase a number of visits
4. ... and program crashes

The problem I was facing is connected with how Postgres protocol works. It turns out that you can't call `Exec` if you did not read all the data from previous `Query`. The problem is that most of Postgres drivers including its own`libpq` does *fetch all the rows* by default. It gives the false impression that following code is legal and works.  Go driver `lib/pq` does not do it. `plpgsql` does handle this case well. Other language bindings fetch the data by default.

However fetching the data was not an option as there are millions of entries. And using different language than Go was impractical. Url normalization has been written in Go and there are huge differences between language parsers of URLs between languages.  For example PHP handles query strings in really [bizzare](https://www.php.net/manual/en/function.parse-str.php#76792) way incompatible with CGI BIN, Perl and any other language. One can build Go code as (C like) shared library, but that would be somewhat big effort.

Fetching data was an option, however given the number of entries, one would need to play with `LIMIT` and `OFFSET` to achieve reasonable size of input data.

## Do it as plpgsql

There is object called [cursor](https://www.postgresql.org/docs/10/plpgsql-cursors.html). Cursors are intended exactly for this use case. They **encapsulates** the query and one can read a few results at the time. And `plpgsql` uses cursors under the hood. It turns out `CURSOR` can be used from Go code as well

So Go code changes a bit. There is no `*sql.Rows` and no `Next()` method to be used in the loop.

{{< highlight go >}}
_, err = tx.Exec("DECLARE url_cur CURSOR FOR SELECT id, url, visits FROM schema.url WHERE url LIKE $1 ORDER BY id", pattern)
defer tx.Exec("CLOSE url_cur")

for {
        err = tx.QueryRow("FETCH NEXT FROM url_cur").Scan(&id, &url, &visits)
	if err == sql.ErrNoRows {
	        break
	} else if err != nil {
	        _ = tx.Rollback()
		panic(err)
	}
        // call tx.Exec with UPSERT and DELETE
}
{{< / highlight >}}

The advantages are

1. No new language in the stack, neither complicated integration of Go code into another language
2. It is efficient and works with one entry at the time
3. There are minimal changes to Go code
4. It works, data are written on `tx.Commit()`

Big thanks comes to [@hypnoglow](https://github.com/hypnoglow) and his comment at Github [issue#635](https://github.com/lib/pq/issues/635#issuecomment-327800640)

Logo by kubina@Flickr: [https://www.flickr.com/photos/kubina/912714753]
