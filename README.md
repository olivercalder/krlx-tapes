# krlx-tapes
A server to search, assemble, and download radio show tapes for KRLX.

## Design Sketch

### API

The API should have the ability to search the KRLX database for show tapes according to certain parameters, and then assemble individual timestamped tapes into larger show recordings, and zip the resulting recording(s) into a zip file which can be downloaded by the client.
Flask is not particularly efficient, so it is probably best for a different web server to serve the resulting zip files, rather than serving them directly through a flask response with MIME type `application/zip`, though this is a possibility.
However, caching is also very important, both to save disc space (and compute time) and so that the download can be restarted on the existing file in case the connection cuts out mid-download.

The API should therefore have the ability to look up cached files, likely by some identifier which is shared between requests which will result in identical outputs.
Namely, the show ID (from the KRLX database) seems ideal to be used when downloading all recordings for a given show, and the start and end time seem should suffice when wishing only to download the recording of a particular time span.
The time of the request should also be included, so that the cache can be cleaned up regularly by removing zip files which have not been requested for some specified period of time, e.g. 2 days.
Thus, files ready for download may have a format similar to the following:

```txt
.../cache/show-id/<identifier>_<showname>_<timestamp>.zip
.../cache/timespan/<startdate>-<starttime>-<duration>_<timestamp>.zip
```

When a file is requested, the ID can be compared to the existing files in the cache (ignoring the timestamp).
If there is a cache hit, rather than renaming the cached file with a new timestamp (which could interfere with ongoing downloads), a hard link can instead be created with the same ID but the new timestamp.
When old files are cleaned up, the file data will remain unchanged until all links to the data are removed.

So, assuming the API can successfully create, cache, and look up zip files associated with queries, it remains to be decided how the API should get the resulting files back to the client.
I see three options:

1. The API could return the path of the parsed file as plaintext.  That path could then be used to populate a link or button on the rendered webpage of the client, or could be used directly if desired.  This is also perhaps desirable, since the webpage can inform the user of the (potentially long) file creation time and display a loading message until the response returns.
2. The API could [redirect](https://flask.palletsprojects.com/en/2.2.x/api/#flask.redirect) the request URL to the URL of the cached file, allowing the client to download the zip file from the fileserver without the user needing to know the timestamped filename of the zip file.  This could also be useful for sharing a persistent link to the recordings, since the server will either respond with a cached file or re-generate the file using the same parameters.
3. The API could [send the file](https://flask.palletsprojects.com/en/2.2.x/api/#flask.send_from_directory) directly to the client.  The benefit of this is that there would be no need for a separate fileserver.  Additionally, if it were desirable for the filepaths to be obfuscated, this would do so.

At the moment, it is unclear which of these options should be implemented.

#### API Routes

- `/show-id/<id>`
- `/timespan/<startdate_starttime_duration>` (Return 404 if duration too long, say >8 hours)
- `/search-show-name/<name>`
- `/search-email/<email>`
- `/search-dj-name/<fullname>`

Alternatively, request arguments could be used, which would simplify the searching and timespan APIs, and allow more constrained searches by combining parameters.

- `/show-id/<id>`
- `/timespan/?date=<startdate>&time=<starttime>&duration=<duration>` (Return 404 if duration too long, say >8 hours)
- `/search/?show=<showname>&email=<email>&dj=<dj-name>`


#### Searching

Searching should be done via SQL queries to the KRLX database.


### Frontend

There should be a search bar of some sort, which allows searching by any combination of the parameters described above.
Search results should populate a table of some sort, which provides show name, DJ names, show ID (maybe), show term, day of the week, and time, and a link to the API to download the tapes.
Searching is probably not required for downloading shows during a certain timespan, but there should be a fillable tool which will construct an API query for the user, given start date, start time, and duration.


## Lingering Questions

#### Downloading a show before the it goes off air

If a show were downloaded by show ID in the middle of term (let alone in the middle of a show itself!), a zip file would be created without tapes from later shows that term.
If those same tapes were repeatedly requested, it is possible that they could never get old enough to be cleaned up, and thus attempts to download the complete show recordings would fail.

Some options:

- Ignore this edge case
- Disallow downloading shows by ID when the show off-air time has not yet passed
  - Blocking these requests is a slippery slope that leads to blocked requests...
  - What about when the off-air time is wrong (it's often off by a week or so)?
  - What about when you want to download your show partway though the term?
- Devise some clever check at request time to see if request is for a show-in-progress (regarding the term), and...
  1. ...ignore the cache if so.  This is vulnerable to abuse, since a malicious user could repeatedly request the same show during the term and fill up disk space --- they could do this to a certain extent already via timespan requests, which is bad.
  2. ...use further cleverness to check whether a show instance has finished since the last cached request for this show.

#### Direct playback on the site

If cached files are stored as zips, they can't be played directly.
One solution could be to merge raw tapes into shows immediately as they are created, but this would require querying the database, and it would be better for the recording itself to be unreliant on any other system.
However, a similar (and fully independent) system could be created to assemble, cache, and serve raw MP3 files, which could then be played directly by the browser or by a media player on the site.

#### Downloading a single week of a show by ID

It may be preferable to download a single recording of a show, as opposed to all the recordings of that show from the term.
Should the show ID route handle this, or should the client automatically convert this into a show timespan request?
If it is converted into a timespan request, the files will not be named by the show name, but instead by time, which is probably undesirable.

