# dtsl-mysql
Date/Time/Strings Libraries (DTSL) for MySQL

DTSL is collection of stored functions and procedures for manipulating date, time, and string values, for MySQL, MariaDB, and Percona Server.

The features here represent things I've had a need for -- real or imagined -- over the years as a DBA; the applications and general level of usefulness of the included functions vary wildly, but tend to coalesce around dates, times, datetimes, and strings.  The argument can be made that many of them are strictly "unnecessary" because they encapsulate code that you could simply include in your own queries, views, and stored programs... but sometimes that level of encapsulation is nice in its own right, since it keeps the calling code tidier.

Feature requests will be considered.  Paid feature requests will be considered even faster. 

----

This project and its actual code will be published soon.

----

The rest of this is a draft.  Still deciding on how to organize the documentation.

#### Read this, first.

I have a deep and abiding sense of irritation reserved for people who are nice enough to share their code but apparently not nice enough to explain how it works and how to use it... so I am determined not to be one of those people.

I will be writing a comprehensive guide to all of the features and capabilities this library provides.

If you "do not have time" to read this entire document carefully, please note that I will also not have time to re-answer any questions already covered, here. 

#### Text/String Functions

Sometimes, it's nice to let the database take care of something, even when the database is arguably the "wrong" place to do it.

##### RLIKE is all there is?  What about regex_replace?

I solved this one during a boring meeting.

MySQL, natively, can only *match* on regexes, it can't *replace* the pattern with something else.  But it can replace a chunk of a string with something else, given a character position and length.  Can we combine those into something useful?  We can.

This is not a perfect solution to all your regexing needs; its performance is limited by the capabilities exposed by MySQL (and probably by some optimization paths I overlooked), but if you need some simple regular expression-based replacing, watch this:

    mysql> SELECT dtsl.regex_replace('The quick brown fox','[aeiou]+','x');
    +----------------------------------------------------------+
    | dtsl.regex_replace('The quick brown fox','[aeiou]+','x') |
    +----------------------------------------------------------+
    | Thx qxck brxwn fxx                                       |
    +----------------------------------------------------------+
    1 row in set (0.00 sec)

How about that? For each occurrence of 1 or more characters in [aeiou], we replace with a single x.  You can also anchor a pattern to the `^` beginning of the string or `$` end.  Expressions that end with an unbounded modifier (such as `+` or `*`) are greedy.

    mysql> SELECT dtsl.regex_replace('The quick brown fox and the slow white fox','fox$','dog');
    +-------------------------------------------------------------------------------+
    | dtsl.regex_replace('The quick brown fox and the slow white fox','fox$','dog') |
    +-------------------------------------------------------------------------------+
    |   The quick brown fox and the slow white dog                                  |
    +-------------------------------------------------------------------------------+
    1 row in set (0.00 sec)

Sorry, no capturing groups (e.g. `\1` or `$1`) support.  If you really need regexes that badly, you need to be using something more efficient... but this is a sweet little coup that allow some easy matching and cleanup with replacements.

###### HTML entities in strings in your database?

Maybe it's old static content, who knows how it got that way -- I'm not judging -- but the data, for whatever reason, is now in your database and it's making you crazy.  When you send it straight to a web page, sure, it's fine, but when you put it in a report or something -- yuck.  You can clean these up, either on demand or en masse, with `decode_entities(longtext)`.

`LONGTEXT` in, `LONGTEXT` out, if there are any HTML entities -- either named or numeric, in decimal or hex -- they will be converted to their utf-8 equivalents in the output.  As many matching entitie as we can find in the input will be replaced in the output; things that look like they might have been entities, but we didn't recognize them (let's say, `&foo100;`) are passed through unaltered.  Strings with no matching entities are returned unchanged as soon as we realize there's nothing to do.

    mysql> SELECT dtsl.decode_entities('I &hearts; HTML&trade; &Eacute;&ntilde;t&igrave;ties &#10004; &amp; &#x2714;');
    +------------------------------------------------------------------------------------------------------+
    | dtsl.decode_entities('I &hearts; HTML&trade; &Eacute;&ntilde;t&igrave;ties &#10004; &amp; &#x2714;') |
    +------------------------------------------------------------------------------------------------------+
    | I ♥ HTML™ Éñtìties ✔ & ✔                                                                            |
    +------------------------------------------------------------------------------------------------------+
    1 row in set (0.00 sec)

Tell me that isn't beautiful to behold.

#### Date/Time Handling

##### Support for particularly useful subsets of iso8601 formatting of datetimes

MySQL has some excellent built-in date/time manipulation functions and a useful implementation of time zones, including the magic handling of `TIMESTAMP` columns in the current session `@@time_zone` but lacks the ability to accept timestamps you may receive from external APIs and other places.

We have that one covered:

    mysql> select dtsl.from_iso8601('2016-01-31T23:00:00-05:00');
    +------------------------------------------------+
    | dtsl.from_iso8601('2016-01-31T23:00:00-05:00') |
    +------------------------------------------------+
    | 2016-02-01 04:00:00                            |
    +------------------------------------------------+
    1 row in set (0.00 sec)

The resulting value is in the current session's time zone.  If you don't use UTC as your `@@system_time_zone`, that pretty much makes you a bad person, but we can handle them correctly for you all the same.

...unless you never got around to setting up the time zone tables on your server, in which case, you need to do that, first.

    mysql> SET @@time_zone = 'America/New_York';
    Query OK, 0 rows affected (0.00 sec)

    mysql> select dtsl.from_iso8601('2016-01-31T00:00:00Z');
    +-------------------------------------------+
    | dtsl.from_iso8601('2016-01-31T00:00:00Z') |
    +-------------------------------------------+
    | 2016-01-30 19:00:00                       |
    +-------------------------------------------+
    1 row in set (0.00 sec)

Pretty spiffy.  

The current implementation works with the following formats, because these are the formats I needed to handle when I wrote the function:

    YYYY-MM-DDTHH:MM:SSZ
    YYYY-MM-DDTHH:MM:SS.nnnnnnZ   # usec are currently discarded
    YYYY-MM-DDTHH:MM:SS[+/-]HH:MM # offset from UTC

Additional formats should be fairly straightforward to implement if needed.

MySQL Server 5.6 introduced millisecond and microsecond datetimes and timestamps, but stored functions don't support dynamic typing, so it's impossible to selectively return a `DATETIME` or a `DATETIME(3)` or a `DATETIME(6)` depending on the particular input parameters.  And yet, it's very arguably wrong to implicitly convert '2015-03-23T12:00:00Z' into '2015-03-23 12:00:00.000000' because of the false sense of precision this provides.  Future iterations of this function may actually resort to returning a `TINYTEXT` and letting MySQL implicitly cast it it to the appropriate type for your use -- so, for example, if you provide a timestamp with precision to the millisecond, we'll provide a return value with the same three decimals populated.

##### Find the short, common, human-friendly abbreviation of a particular time zone at a particular point in time.

This is useful for showing timestamps to humans who are unfamiliar with time zone names like "America/Chicago."  Of course, these are the same people who will use the phrase "Eastern Standard Time" in the summer, when it is in fact "Eastern Daylight Time," but we can only solve so many problems at the database level.

    mysql> SELECT dtsl.time_zone_abbrev_at_ref('America/Chicago','2015-01-01 00:00:00');
    +-----------------------------------------------------------------------+
    | dtsl.time_zone_abbrev_at_ref('America/Chicago','2015-01-01 00:00:00') |
    +-----------------------------------------------------------------------+
    | CST                                                                   |
    +-----------------------------------------------------------------------+
    1 row in set (0.01 sec)

    mysql> SELECT dtsl.time_zone_abbrev_at_ref('America/Chicago','2015-07-01 00:00:00');
    +-----------------------------------------------------------------------+
    | dtsl.time_zone_abbrev_at_ref('America/Chicago','2015-07-01 00:00:00') |
    +-----------------------------------------------------------------------+
    | CDT                                                                   |
    +-----------------------------------------------------------------------+
    1 row in set (0.00 sec)

Now, let's see if we can make this do something useful.

    mysql> SET @ltz = 'America/Chicago', @tpa = '2016-03-23 12:00:00'; -- @ltz local time zone @tpa took place at
    Query OK, 0 rows affected (0.00 sec)

    mysql> SELECT CONCAT('This event has a timestamp of ',CONVERT_TZ(@tpa,'UTC',@ltz),'(',dtst.time_zone_abbrev_at_ref(@ltz,@tpa),').') AS `human_readable`;
    +----------------------------------------------------------+
    | human_readable                                           |
    +----------------------------------------------------------+
    | This event has a timestamp of 2016-03-23 07:00:00 (CDT). |
    +----------------------------------------------------------+
    1 row in set (0.00 sec)

Seems legit.
 
##### Given a date or datetime, find the first day of the [ previous | current | next ] [ week | month | quarter | year ] based on that date.

Not surprisingly, these functions were actually the original inspiration for collecting similar functions into "dt," which originally stood for "date/time" and later became an internal backronym for "data tools" and finally evolved into "dtsl."  Tragically, they are perhaps the least exciting in appearance, despite the fact that they are tremendously useful for creating easy to write, easy to understand, easy to debug, and easy to optimize queries.

Queries for dates matching "this week" or "last month" are best handled by providing the query optimizer with something it can reduce to a constant expression, in the interest of sargability -- which is a huge thing, and if you don't know about it already, you'll be embarrassed when you discover it.

While logically correct, queries in the following form will force a full table scan, since functions in the `WHERE` clause with columns as arguments give the query optimizer no alternative.  Never, ever do this, and be sure to scoff at anyone who does.

    SELECT * FROM t1 
     WHERE YEAR(took_place_at) = 2016 
       AND MONTH(took_place_at) = 2;
       
This is a very naive query and performance is guaranteed to diminish as the data grows.  Don't write queries like this.  Ever.
  
The correct way to write this query is more along these lines:

    SELECT * FROM t1 
     WHERE took_place_at >= '2016-02-01 00:00:00' 
       AND took_place_at < '2016-03-01 00:00:00';
  
This form presents the optimizer with constant values, which it readily uses to select rows from the index on `took_place_at`.  
  
DTSL allows you to write this query without having to do the date math separately.

    -- assuming NOW() is any time in March, 2016.
    SELECT * FROM t1 
     WHERE took_place_at >= dtsl.prev_month_start(NOW()) 
       AND took_place_at < dtsl.curr_month_start(NOW());

The optimizer realizes that over the course of the query, `NOW()` cannot change, and therefore the results of the deterministic functions cannot change, and therefore the functions can be evaluated into constants only once, and used with the index on the column `took_place_at`.  

See these functions:

    mysql> select NOW(), dt.prev_month_start(NOW()), dt.curr_month_start(NOW()), dt.next_month_start(NOW());
    +---------------------+----------------------------+----------------------------+----------------------------+
    | NOW()               | dt.prev_month_start(NOW()) | dt.curr_month_start(NOW()) | dt.next_month_start(NOW()) |
    +---------------------+----------------------------+----------------------------+----------------------------+
    | 2016-04-03 13:25:52 | 2016-03-01 00:00:00        | 2016-04-01 00:00:00        | 2016-05-01 00:00:00        |
    +---------------------+----------------------------+----------------------------+----------------------------+
    1 row in set (0.01 sec)

TODO: we have these for year and quarter also.

----

Fix the Y2K bug in the other direction. 

This function is embarrassing.  It should be particularly embarrassing to the client that sends me this data -- sometimes it's in mm/dd/yyyy format, and other times it's in mm/dd/yy format -- in the same table, of course -- where it's populated into a column that was explicitly defined by the client as a *text* field.  I have to keep it in this hideous form because the client refreshes the data via an API call every time the site visitor logs in... but they expect me to interpret it for reporting.

MySQL's `STR_TO_DATE()` function support 2-digit years, and if given a 4 digit year, will still work correctly.

But, `STR_TO_DATE()` has the "wrong" (not really, but stick with me) cutoff for what a 2-digit year means.

    SELECT STR_TO_DATE('03/23/20','%m/%d/%Y');
    +------------------------------------+
    | STR_TO_DATE('03/23/20','%m/%d/%Y') |
    +------------------------------------+
    | 2020-03-23                         |
    +------------------------------------+
    1 row in set (0.00 sec)
    
The argument is, of course, valid that nobody should be sending 2-digit years, much less using the world's most ambiguous (not to mention USA-centric) date format MM/DD/YY, but sometimes sound arguments are outweighed by the size of the client's budget. 

Mind you, this is the same client that ridiculously expected me to set my servers' time zones "to the time zone where the server is located" -- in a cloud-hosted, global application.  I told the client representative to tell them that as far as I knew, the server was located 5.5 miles (8.9 km) east south-east of Charing Cross, in [Greenwich](https://en.wikipedia.org/wiki/Greenwich).  Somehow, I doubt that message was relayed.

So, here's the fix: assume MM/DD/YY dates always refer to something prior to `NOW()`.

    mysql> SELECT dtsl.coerce_future_yy_to_19yy('03/23/20');
    +-------------------------------------------+
    | dtsl.coerce_future_yy_to_19yy('03/23/20') |
    +-------------------------------------------+
    | 1920-03-23                                |
    +-------------------------------------------+
    1 row in set (0.00 sec)

Sigh.  And, of course, I have to include this here in the canonical distribution, because I need it on my servers, and I intend to deploy dtsl onto them from this distribution.

----

TODO: document a lot more of the functions we have available.
