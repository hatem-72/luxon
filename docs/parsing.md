# Parsing

Luxon is not an NLP tool and isn't suitable for all date parsing jobs. But it can do some parsing:

1.  Direct support for several well-known formats, including most valid ISO 8601 formats
2.  An ad-hoc parser for parsing specific formats

## Parsing technical formats

### ISO 8601

Luxon supports a wide range of valid ISO 8601 formats through the `fromISO` method.

```js
DateTime.fromISO("2016-05-25");
```

All of these are parsable by `fromISO`:

```
2016
2016-05
201605
2016-05-25
20160525
2016-05-25T09
2016-05-25T09:24
2016-05-25T09:24:15
2016-05-25T09:24:15.123
2016-05-25T0924
2016-05-25T092415
2016-05-25T092415.123
2016-05-25T09:24:15,123
2016-W21-3
2016W213
2016-W21-3T09:24:15.123
2016W213T09:24:15.123
2016-200
2016200
2016-200T09:24:15.123
09:24
09:24:15
09:24:15.123
09:24:15,123
```

- In addition, all the times support offset arguments like "Z" and "+06:00".
- Missing lower-order values are always set to the minimum possible value; i.e. it always parses to a full DateTime. For example, "2016-05-25" parses to midnight of that day. "2016-05" parses to the first of the month, etc.
- The time is parsed as a local time if no offset is specified, but see the method docs to see your options, and also check out [time zone docs](zones.md) for more details.

### HTTP and RFC2822

Luxon also provides parsing for strings formatted according to RFC 2822 and the HTTP header specs (RFC 850 and 1123):

```js
DateTime.fromRFC2822("Tue, 01 Nov 2016 13:23:12 +0630");
DateTime.fromHTTP("Sunday, 06-Nov-94 08:49:37 GMT");
DateTime.fromHTTP("Sun, 06 Nov 1994 08:49:37 GMT");
```

### SQL

Luxon accepts SQL dates, times, and datetimes, via `fromSQL`:

```js
DateTime.fromSQL("2017-05-15");
DateTime.fromSQL("2017-05-15 09:24:15");
DateTime.fromSQL("09:24:15");
```

It works similarly to `fromISO`, so see above for additional notes.

### Unix timestamps

Luxon can parse numerical [Unix timestamps](https://en.wikipedia.org/wiki/Unix_time):

```js
DateTime.fromMillis(1542674993410);
DateTime.fromSeconds(1542674993);
```

Both methods accept the same options, which allow you to specify a timezone, calendar, and/or numbering system.

### JS Date Object

A native JS `Date` object can be converted into a `DateTime` using `DateTime.fromJSDate`.

An optional `zone` parameter can be provided to set the zone on the resulting object.


## Ad-hoc parsing

### Consider alternatives

You generally shouldn't use Luxon to parse arbitrarily formatted date strings:

1.  If the string was generated by a computer for programmatic access, use a standard format like ISO 8601. Then you can parse it using `DateTime.fromISO`.
2.  If the string is typed out by a human, it may not conform to the format you specify when asking Luxon to parse it. Luxon is quite strict about the format matching the string exactly.

Sometimes, though, you get a string from some legacy system in some terrible ad-hoc format and you need to parse it.

### fromFormat

See `DateTime.fromFormat` for the method signature. A brief example:

```js
DateTime.fromFormat("May 25 1982", "LLLL dd yyyy");
```

### Intl

Luxon supports parsing internationalized strings:

```js
DateTime.fromFormat("mai 25 1982", "LLLL dd yyyy", { locale: "fr" });
```

Note, however, that Luxon derives the list of strings that can match, say, "LLLL" (and their meaning) by introspecting the environment's Intl implementation. Thus the exact strings may in some cases be environment-specific. You also need the Intl API available on the target platform (see the [support matrix](matrix.md)).

### Limitations

Not every token supported by `DateTime#toFormat` is supported in the parser. For example, there's no `ZZZZ` or `ZZZZZ` tokens. This is for a few reasons:

- Luxon relies on natively-available functionality that only provides the mapping in one direction. We can ask what the named offset is and get "Eastern Standard Time" but not ask what "Eastern Standard Time" is most likely to mean.
- Some things are ambiguous. There are several Eastern Standard Times in different countries and Luxon has no way to know which one you mean without additional information (such as that the zone is America/New_York) that would make EST superfluous anyway. Similarly, the single-letter month and weekday formats (EEEEE) that are useful in displaying calendars graphically can't be parsed because of their ambiguity.
- Because of the limitations above, Luxon also doesn't support the "macro" tokens that include offset names, such as "ttt" and "FFFF".

### Debugging

There are two kinds of things that can go wrong when parsing a string: a) you make a mistake with the tokens or b) the information parsed from the string does not correspond to a valid date. To help you sort that out, Luxon provides a method called `fromFormatExplain`. It takes the same arguments as `fromFormat` but returns a map of information about the parse that can be useful in debugging.

For example, here the code is using "MMMM" where "MMM" was needed. You can see the regex Luxon uses and see that it didn't match anything:

```js
> DateTime.fromFormatExplain("Aug 6 1982", "MMMM d yyyy")

{ input: 'Aug 6 1982',
  tokens:
   [ { literal: false, val: 'MMMM' },
     { literal: false, val: ' ' },
     { literal: false, val: 'd' },
     { literal: false, val: ' ' },
     { literal: false, val: 'yyyy' } ],
  regex: '(January|February|March|April|May|June|July|August|September|October|November|December)( )(\\d\\d?)( )(\\d{4})',
  matches: {},
  result: {},
  zone: null }
```

If you parse something and get an invalid date, the debugging steps are slightly different. Here, we're attempting to parse August 32nd, which doesn't exist:

```js
var d = DateTime.fromFormat("August 32 1982", "MMMM d yyyy");
d.isValid; //=> false
d.invalidReason; //=> 'day out of range'
```

For more on validity and how to debug it, see [validity](validity.md). You may find more comprehensive tips there. But as it applies specifically to `fromFormat`, again try `fromFormatExplain`:

```js
> DateTime.fromFormatExplain("August 32 1982", "MMMM d yyyy")

{ input: 'August 32 1982',
  tokens:
   [ { literal: false, val: 'MMMM' },
     { literal: false, val: ' ' },
     { literal: false, val: 'd' },
     { literal: false, val: ' ' },
     { literal: false, val: 'yyyy' } ],
  regex: '(January|February|March|April|May|June|July|August|September|October|November|December)( )(\\d\\d?)( )(\\d{4})',
  matches: { M: 8, d: 32, y: 1982 },
  result: { month: 8, day: 32, year: 1982 },
  zone: null }
```

Because Luxon was able to parse the string without difficulty, the output is a lot richer. And you can see that the "day" field is set to 32. Combined with the "out of range" explanation above, that should clear up the situation.

### Table of tokens

(Examples below given for `2014-08-06T13:07:04.054` considered as a local time in America/New_York). Note that many tokens supported by the [formatter](formatting.md) are **not** supported by the parser.

| Standalone token | Format token | Description                                                    | Example                     |
| ---------------- | ------------ | -------------------------------------------------------------- | --------------------------- |
| S                |              | millisecond, no padding                                        | `54`                        |
| SSS              |              | millisecond, padded to 3                                       | `054`                       |
| u                |              | fractional seconds, (5 is a half second, 54 is slightly more)  | `54`                        |
| uu               |              | fractional seconds, (one or two digits)                        | `05`                        |
| uuu              |              | fractional seconds, (only one digit)                           | `5`                         |
| s                |              | second, no padding                                             | `4`                         |
| ss               |              | second, padded to 2 padding                                    | `04`                        |
| m                |              | minute, no padding                                             | `7`                         |
| mm               |              | minute, padded to 2                                            | `07`                        |
| h                |              | hour in 12-hour time, no padding                               | `1`                         |
| hh               |              | hour in 12-hour time, padded to 2                              | `01`                        |
| H                |              | hour in 24-hour time, no padding                               | `13`                        |
| HH               |              | hour in 24-hour time, padded to 2                              | `13`                        |
| Z                |              | narrow offset                                                  | `+5`                        |
| ZZ               |              | short offset                                                   | `+05:00`                    |
| ZZZ              |              | techie offset                                                  | `+0500`                     |
| z                |              | IANA zone                                                      | `America/New_York`          |
| a                |              | meridiem                                                       | `AM`                        |
| d                |              | day of the month, no padding                                   | `6`                         |
| dd               |              | day of the month, padded to 2                                  | `06`                        |
| E                | c            | day of the week, as number from 1-7 (Monday is 1, Sunday is 7) | `3`                         |
| EEE              | ccc          | day of the week, as an abbreviate localized string             | `Wed`                       |
| EEEE             | cccc         | day of the week, as an unabbreviated localized string          | `Wednesday`                 |
| M                | L            | month as an unpadded number                                    | `8`                         |
| MM               | LL           | month as an padded number                                      | `08`                        |
| MMM              | LLL          | month as an abbreviated localized string                       | `Aug`                       |
| MMMM             | LLLL         | month as an unabbreviated localized string                     | `August`                    |
| y                |              | year, 1-6 digits, very literally                               | `2014`                      |
| yy               |              | two-digit year, interpreted as > 1960 (also accepts 4)         | `14`                        |
| yyyy             |              | four-digit year                                                | `2014`                      |
| yyyyy            |              | four- to six-digit years                                       | `10340`                     |
| yyyyyy           |              | six-digit years                                                | `010340`                    |
| G                |              | abbreviated localized era                                      | `AD`                        |
| GG               |              | unabbreviated localized era                                    | `Anno Domini`               |
| GGGGG            |              | one-letter localized era                                       | `A`                         |
| kk               |              | ISO week year, unpadded                                        | `17`                        |
| kkkk             |              | ISO week year, padded to 4                                     | `2014`                      |
| W                |              | ISO week number, unpadded                                      | `32`                        |
| WW               |              | ISO week number, padded to 2                                   | `32`                        |
| o                |              | ordinal (day of year), unpadded                                | `218`                       |
| ooo              |              | ordinal (day of year), padded to 3                             | `218`                       |
| q                |              | quarter, no padding                                            | `3`                         |
| D                |              | localized numeric date                                         | `9/6/2014`                  |
| DD               |              | localized date with abbreviated month                          | `Aug 6, 2014`               |
| DDD              |              | localized date with full month                                 | `August 6, 2014`            |
| DDDD             |              | localized date with full month and weekday                     | `Wednesday, August 6, 2014` |
| t                |              | localized time                                                 | `1:07 AM`                   |
| tt               |              | localized time with seconds                                    | `1:07:04 PM`                |
| T                |              | localized 24-hour time                                         | `13:07`                     |
| TT               |              | localized 24-hour time with seconds                            | `13:07:04`                  |
| TTT              |              | localized 24-hour time with seconds and abbreviated offset     | `13:07:04 EDT`              |
| f                |              | short localized date and time                                  | `8/6/2014, 1:07 PM`         |
| ff               |              | less short localized date and time                             | `Aug 6, 2014, 1:07 PM`      |
| F                |              | short localized date and time with seconds                     | `8/6/2014, 1:07:04 PM`      |
| FF               |              | less short localized date and time with seconds                | `Aug 6, 2014, 1:07:04 PM`   |
