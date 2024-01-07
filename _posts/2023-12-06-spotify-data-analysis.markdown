---
layout: post
title:  "An Analysis of My Entire Spotify Streaming History (2017-2023)"
date:   2023-12-28 00:22:04 -0700
toc: true
---

## The Problem With Spotify Wrapped

Spotify Wrapped is nice, but it just doesn't give me enough insight over just how bad my music taste is, and it's
only pre-year. I have a couple of questions I'd like answered:

- Just how bad is my album/artist diversity?
- How many times have I listened to each album, all the way through?
- How much of my listening history is just Taylor Swift?
- Do the songs I listen to significantly differ based on what time of day it is?

I recently realized that I had actually learned enough during my undergraduate courses and internships
to actually answer all of these questions, so it's time to dig in.

## Getting Your Spotify Data

As of 2023, you can download all of your streaming history through Spotify's privacy settings. There are three options, Account Data, Extended Streaming History, and Technical Log Information. For this project, we want Extended Streaming History since it will contain the UUIDs of the songs which we will need for the Spotify API. It'll take up to 30 days to arrive, for me, it took around 3 weeks.

## Parsing the Logs

I'm using Rust for this part. Outside of embedded programming and as a C/C++ replacement, I find Rust to be a nice general-purpose
langugage with okay library support and very nice compile-time checks.

The Spotify download gives a nice table about what data to expect, a JSON array with each element being an object
of the following form:

<table>
<thead><th>Technical field</th><th>Contains</th></thead>
<tr>
<td> ts </td><td> This field is a timestamp indicating when the track stopped
playing in UTC (Coordinated Universal Time). The order is
year, month and day followed by a timestamp in military time </td>
</tr>



<tr><td>username</td> <td>This field is your Spotify username.</td></tr>

<tr><td>platform</td> <td>This field is the platform used when streaming the track (e.g.
Android OS, Google Chromecast).
ms_played This field is the number of milliseconds the stream was
played.</td></tr>

<tr><td>conn_country</td> <td>This field is the country code of the country where the stream
was played (e.g. SE - Sweden).</td></tr>

<tr><td>ip_addr_decrypted</td> <td>This field contains the IP address logged when streaming the
track.</td></tr>

<tr><td>user_agent_decrypted</td> <td>This field contains the user agent used when streaming the
track (e.g. a browser, like Mozilla Firefox, or Safari)</td></tr>

<tr><td>master_metadata_track_name</td> <td>This field is the name of the track.</td></tr>

<tr><td>master_metadata_album_artist_name</td> <td>This field is the name of the artist, band or podcast.</td></tr>

<tr><td>master_metadata_album_album_name</td> <td>This field is the name of the album of the track.</td></tr>

<tr><td>spotify_track_uri</td> <td>A Spotify URI, uniquely identifying the track in the form of
“spotify:track:&lt;base-62 string&gt;”
A Spotify URI is a resource identifier that you can enter, for
example, in the Spotify Desktop client’s search box to locate
an artist, album, or track.</td></tr>

<tr><td>episode_name</td> <td>This field contains the name of the episode of the podcast.</td></tr>

<tr><td>episode_show_name</td> <td>This field contains the name of the show of the podcast.</td></tr>

<tr><td>spotify_episode_uri</td> <td>A Spotify Episode URI, uniquely identifying the podcast
episode in the form of “spotify:episode:&lt;base-62 string&gt;”
A Spotify Episode URI is a resource identifier that you can
enter, for example, in the Spotify Desktop client’s search box
to locate an episode of a podcast.</td></tr>

<tr><td>reason_start</td> <td>This field is a value telling why the track started (e.g.
“trackdone”)</td></tr>

<tr><td>reason_end</td> <td>This field is a value telling why the track ended (e.g.
“endplay”).</td></tr>

<tr><td>shuffle</td> <td>This field has the value True or False depending on if shuffle
mode was used when playing the track.</td></tr>

<tr><td>skipped</td> <td>This field indicates if the user skipped to the next song</td></tr>

<tr><td>offline</td> <td>This field indicates whether the track was played in offline
mode (“True”) or not (“False”).</td></tr>

<tr><td>offline_timestamp</td> <td>This field is a timestamp of when offline mode was used, if
used.</td></tr>

<tr><td>incognito_mode</td> <td>This field indicates whether the track was played in incognito
mode (“True”) or not (“Falsets This field is a timestamp indicating when the track stopped
playing in UTC (Coordinated Universal Time). The order is
year, month and day followed by a timestamp in military time</td></tr>
</table>

So most of these fields aren't too useful, but just in case we need any later, let's put all of this data into our database. I'm using a simple MySQL instance locally and the sqlx library in Rust to execute SQL queries. But first, to parse the JSON, the natural choice is serde_json in Rust. Unfortunately, which fields are nullable isn't very well documented, but a little trial and error, we arrive
at this Rust struct for deserialization:

```rust
#[derive(Debug, Serialize, Deserialize)]
struct StreamingData {
    ts: String,
    username: String,
    platform: String,
    ms_played: i32,
    conn_country: String,
    ip_addr_decrypted: Option<String>,
    user_agent_decrypted: Option<String>,
    master_metadata_track_name: Option<String>,
    master_metadata_album_artist_name: Option<String>,
    master_metadata_album_album_name: Option<String>,
    spotify_track_uri: Option<String>,
    episode_name: Option<String>,
    episode_show_name: Option<String>,
    spotify_episode_uri: Option<String>,
    reason_start: String,
    reason_end: Option<String>,
    shuffle: Option<bool>,
    skipped: Option<bool>,
    offline: Option<bool>,
    offline_timestamp: Option<i64>,
    incognito_mode: Option<bool>
}
```

with the corresponding SQL table DDL:

```sql
CREATE TABLE streams (
    ts TIMESTAMP NOT NULL,
    username VARCHAR(255) NOT NULL,
    platform VARCHAR(255) NOT NULL,
    ms_played INT NOT NULL,
    conn_country VARCHAR(255) NOT NULL,
    ip_addr_decrypted VARCHAR(255),
    user_agent_decrypted VARCHAR(255),
    master_metadata_track_name VARCHAR(255),
    master_metadata_album_artist_name VARCHAR(255),
    master_metadata_album_album_name VARCHAR(255),
    spotify_track_uri VARCHAR(255),
    episode_name VARCHAR(255),
    episode_show_name VARCHAR(255),
    spotify_episode_uri VARCHAR(255),
    reason_start VARCHAR(255) NOT NULL,
    reason_end VARCHAR(255),
    shuffle BOOLEAN,
    skipped BOOLEAN,
    offline BOOLEAN,
    offline_timestamp BIGINT,
    incognito_mode BOOLEAN
);
```

With some bulk inserts, we're able to load the data in under a minute, all 112,526 rows.
Time to play around a little bit of the data.

## Exploratory Data Analysis

Let's first look at my top 10 streamed songs of all time:

```sql
SELECT
	master_metadata_track_name,
	master_metadata_album_artist_name,
	SUM(ms_played) / 3600000 AS hours_played
FROM streams
WHERE spotify_track_uri IS NOT NULL
GROUP BY spotify_track_uri, 1, 2
ORDER BY 3 DESC
LIMIT 10
```

|----------------------------|-----------------------------------|--------------|
| master_metadata_track_name | master_metadata_album_artist_name | hours_played |
|----------------------------|-----------------------------------|--------------|
| On Some Emo Shit           | blink-182                         |      27.7208 |
| Beautiful Days Piano       | Masafumi Takada                   |      25.0852 |
| No Capes                   | Waterparks                        |      23.1735 |
| Spring Day                 | BTS                               |      21.9136 |
| Roses                      | The Chainsmokers                  |      21.6337 |
| Rhapsody in Blue           | George Gershwin                   |      19.9754 |
| Reset (feat. Jinsil)       | Tiger JK                          |      19.7157 |
| Hawaii (Stay Awake)        | Waterparks                        |      19.5447 |
| It's Time                  | Imagine Dragons                   |      18.6422 |
| whatever it takes          | convolk                           |      17.9429 |
|----------------------------|-----------------------------------|--------------|

Concerningly, if we don't group by `spotify_track_uri`, we get a different result:

|----------------------------|-----------------------------------|--------------|
| master_metadata_track_name | master_metadata_album_artist_name | hours_played |
|----------------------------|-----------------------------------|--------------|
| Spring Day                 | BTS                               |      30.9958 |
| On Some Emo Shit           | blink-182                         |      27.7208 |
| Beautiful Days Piano       | Masafumi Takada                   |      25.0852 |
| whatever it takes          | convolk                           |      24.5301 |
| Sanctuary                  | Joji                              |      24.0277 |
| No Capes                   | Waterparks                        |      23.1735 |
| KIDS ON MOLLY              | Aries                             |      21.8589 |
| Rhapsody in Blue           | George Gershwin                   |      21.7469 |
| Roses                      | The Chainsmokers                  |      21.6337 |
| Reset (feat. Jinsil)       | Tiger JK                          |      19.7157 |
|----------------------------|-----------------------------------|--------------|

We see that Spring Day by BTS grows by over 9 hours. This is probably due to the fact that the same song has different
tracks on Spotify. Let's see all the different versions of Spring Day:

```sql
SELECT
	spotify_track_uri,
	master_metadata_album_album_name,
	SUM(ms_played) / 3600000 AS hours_played
FROM streams
WHERE master_metadata_track_name = 'Spring Day'
  AND master_metadata_album_artist_name = 'BTS'
GROUP BY 1, 2;
```

|--------------------------------------|----------------------------------|--------------|
| spotify_track_uri                    | master_metadata_album_album_name | hours_played |
|--------------------------------------|----------------------------------|--------------|
| spotify:track:02q0ZnV2L4XByzEvWZJqBC | YOU NEVER WALK ALONE             |       8.0562 |
| spotify:track:0WNGsQ1oAuHzNTk8jivBKW | You Never Walk Alone             |      21.9136 |
| spotify:track:2j1fFjWHCI9KJSwcuYAOyF | You Never Walk Alone             |       1.0259 |
|--------------------------------------|----------------------------------|--------------|

This confirms our suspicions, we're going to have to treat two songs as the same if they have the same title and artist, ignoring case, and
even so, a couple of the same songs might be treated as different, and some different songs treated as the same if an aritst
decides to name two different songs the same name,
but there's not much we can do.

Let's also make sure not too much of our data is unusable, which is, what percent of streams by time has a `NULL` ID?

```sql
SELECT
( SELECT SUM(ms_played) WHERE spotify_track_uri IS NULL ) /
( SELECT SUM(ms_played) ) * 100
AS percent_null;
```

|--------------|
| percent_null |
|--------------|
|   0.57201341 |
|--------------|

0.57% is not too bad. Let's move onto a more interesting question. I consider myself to be primarily an album listener,
so what album have I listend to the most? I don't want to be skewed by listening to a single song of the album, so let's
define the playtime of an album as the minimum playtime of any song in that album if all songs were listened to in that album,
zero otherwise.

To once again avoid issues with the same album being identified with different IDs within Spotify, lets define an album uniquely
defined as its album name and artist name, case insensitive.

This is the point where we need more information, since album info isn't available in our Spotify download. It is found in the API,
which we can query with our track IDs. We have 11,091 distinct track IDs to query, and the API allows bulk queries of up to 50 IDs each. The exact rate limit for the API isn't known but we should be able to get away with 2 queries a second, so this should only take
two minutes or so.

We have to do a lot of insertions into our database, so let's factor out the logic in a helper function:
```rust
async fn bulk_insert<'a, T, F>(pool: &sqlx::Pool<sqlx::MySql>, table: &str, elements: &'a [T], row_closure: F)
where F: Fn(Separated<'_, 'a, MySql, &str>, &'a T) {
    const SQL_BATCH_SIZE: usize = 1000;

    for chunk in elements.chunks(SQL_BATCH_SIZE) {
        let mut query_builder: QueryBuilder<MySql> = QueryBuilder::new(format!("INSERT INTO {} ", table));
        query_builder.push_values(chunk, &row_closure);
        query_builder.build().execute(pool).await.unwrap();
    }
}
```

There's a fun little Rust lifetime puzzle we had to solve in the function parameters to ensure that the query builder
outlives the items being added to the query in the closure passed to this function.

In addition, we're going to be doing a lot of API requests, so let's factor out the behavior we want to exhibit
based on the return code received into another function:

```rust
async fn spotify_request(token: &mut String, url: reqwest::Url) -> String {
    let mut backoff = 1;

    loop {
        let request = reqwest::Client::new().get(url.clone()).bearer_auth(&token);
        let request_copy = request.try_clone().unwrap();
        let response = request_copy.send().await.unwrap();
        let status = response.status();

        if !matches!(status, reqwest::StatusCode::TOO_MANY_REQUESTS) {
            backoff = 1;
        }

        match status {
            reqwest::StatusCode::UNAUTHORIZED => {
                println!("Invalid token! Getting a new one...");
                *token = get_new_token().await;
            }
            reqwest::StatusCode::TOO_MANY_REQUESTS => {
                let retry_after = response.headers().get("Retry-After");
                let timeout = match retry_after {
                    // Use the value of retry_after given to us from Spotify if it exists
                    Some(value) => {
                        value.to_str().unwrap().to_owned().parse().unwrap()
                    }
                    None => backoff
                };
                println!("Too many requests! Backing off {timeout} second(s)...");
                tokio::time::sleep(std::time::Duration::from_secs(timeout)).await;
                backoff *= 2;
            }
            reqwest::StatusCode::OK => {
                return response.text().await.unwrap();
            }
            _ => {
                panic!("Unhandled return code {}!", status);
            }
        }
    }
}
```

The full code is available on my GitHub, I won't include most of it here since it's a lot of boilerplate.

Finally, with all the data we need, let's write a query to figure out my top albums, by the number of minimum
playthroughs of a song in the album, with the requirement that all songs in the album have been played.

First, as discussed, we identify a song uniquely by the key (track name, artist name), which gives us multiple track ids.
A little experimentation showed that all of these track ids for a given key (track name, artist name) pair yield the same album id,
so we can pick one arbitarily, such as the first one that appears. 

Next, we need to divide to aggregate all the time played for each key and divide it by the length of track. We then use a window
function to count how many distinct songs for each album show up.

Finally, filtering on the albums where the number of distinct
songs that show up is equal to the number of tracks in the album, we use aggregation to find the minimum
times played for each album id.

Our resulting query is

```sql
WITH cte1 AS (
    SELECT
        master_metadata_track_name AS track_name,
        master_metadata_album_artist_name AS artist_name,
        JSON_UNQUOTE(JSON_EXTRACT(JSON_ARRAYAGG(spotify_track_uri), '$[0]')) AS track_uri,
        SUM(ms_played) AS total_ms_played
    FROM streams
    GROUP BY 1, 2
    HAVING total_ms_played > 0
),
cte2 AS (
    SELECT
        cte1.track_name,
        cte1.artist_name,
        cte1.total_ms_played / tracks.duration_ms AS times_played,
        albums.id AS album_id,
        albums.name AS album_name,
        albums.total_tracks,
        COUNT(track_name) OVER (PARTITION BY albums.id) AS played_tracks
    FROM cte1 JOIN tracks ON SUBSTRING(cte1.track_uri, 15) = tracks.id
              JOIN albums ON tracks.album_id = albums.id
    WHERE tracks.duration_ms > 0
)
SELECT DISTINCT
    album_name,
    artist_name,
    total_tracks,
    MIN(times_played) AS times_played
FROM cte2
WHERE played_tracks = total_tracks AND total_tracks >= 5
GROUP BY album_id, 1, 2, 3
ORDER BY 4 DESC
LIMIT 20;
```

and our result is

|--------------------------------|---------------------|--------------|--------------|
| album_name                     | artist_name         | total_tracks | times_played |
|--------------------------------|---------------------|--------------|--------------|
| WELCOME HOME                   | Aries               |            9 |      94.5799 |
| BLOODLUST                      | nothing,nowhere.    |            6 |      87.5505 |
| Take Off Your Pants And Jacket | blink-182           |           13 |      85.5395 |
| Double Dare                    | Waterparks          |           13 |      61.7403 |
| deadroses                      | blackbear           |           10 |      58.3813 |
| Cluster                        | Waterparks          |            5 |      36.1772 |
| NINE                           | blink-182           |           15 |      35.8173 |
| Boys Like Girls                | BOYS LIKE GIRLS     |           12 |      35.0078 |
| help                           | blackbear           |           10 |      29.0632 |
| 5 Seconds Of Summer            | 5 Seconds of Summer |           16 |      28.6338 |
| BALLADS 1                      | Joji                |           12 |      26.2355 |
| Heart Flip                     | This Wild Life      |            8 |      25.7468 |
| Entertainment                  | Waterparks          |           10 |      24.3553 |
| X&Y                            | Coldplay            |           13 |      16.1856 |
| Songs About Jane               | Maroon 5            |           12 |      14.8786 |
| ÷ (Deluxe)                     | Ed Sheeran          |           16 |      14.0803 |
| Greatest Hits                  | Waterparks          |           17 |      13.8155 |
| Last Young Renegade            | All Time Low        |           10 |      12.6903 |
| Black Light                    | Waterparks          |            6 |      12.5680 |
| Astoria                        | Marianas Trench     |           17 |       9.1888 |
|--------------------------------|---------------------|--------------|--------------|

That's it for now! In the next part, we'll be trying to see if there's a statistically significant link
between things such as time of day or day of week and the mood of songs, which is something the Spotify API provides.
We'll also be developing a standalone application so people can see their own stats.