# Issue #256

https://github.com/Pylons/webob/issues/256


    >>> from webob import Request
    >>> r = Request.blank("/", accept_language="en-GB, en;q=0.8")
    >>> r.accept_language.best_match(offers=["en", "en-GB"])
    'en'

We would expect the best match to be "en-GB".
Returning 'en' here would not be in line with the sense given by the example in
RFC4647, Section 2.3:
> 'A speaker of Breton in France, for example, can specify "br" followed by "fr",
meaning that if Breton is available, it is preferred, but otherwise French is
the best alternative.'

and really, the whole idea of a priority list.

Unless the ordering in the server's offers are seen as more important than the
user's priority list, but I don't see any support for this in the RFCs.


## To-dos

* RFC7231, Section 5.3.1, on quality values:
	* "The same parameter name is often used within server configurations
	  to assign a weight to the relative quality of the various
	  representations that can be selected for a resource." Related to
	  `server_quality` in `.best_match()`?
	* 'a value of 0 means "not acceptable"'. Is this information important?
	  Is this information sometimes dropped by the current implementation?
	* Are incoming qvalues validated to check that they are not more than
	  three digits? Should they be? Do the use of floats and the resulting
	  floating point errors matter? Note: RFC2616, Section 3.9 Quality
	  Values: 'HTTP content negotiation (section 12) uses short "floating
	  point" numbers to indicate the relative importance..." But also:
	  "HTTP/1.1 applications MUST NOT generate more than three digits after
	  the decimal point. User configuration of these values SHOULD also be
	  limited in this fashion.'
* RFC7231, Section 5.3.5:
	* "Some recipients treat the order in which language tags are listed as
	  an indication of descending priority, particularly for tags that are
	  assigned equal quality values (no value is the same as q=1). However,
	  this behaviour cannot be relied upon. For consistency and to maximize
	  interoperability, many user agents assign each language tag a unique
	  quality value while also listing them in order of decreasing quality."
	  `__iter__` appears to be sort `self._parsed_nonzero` by `q` -- but
	  should it be `_nonzero`? 0 means "not acceptable", but when we get a
	  `list()` of the header attribute, the q=0 information is just
	  dropped? (Noticed that this information is not dropped in `__str__`
	  or `.quality()`, which use `self._parsed`.)
	  (Also, does `sorted` keep original parsed order for items with same
	  qvalue?)
	* Read Section 14.4 of [RFC2616]. Is "Basic Filtering" scheme the only
	  one defined for HTTP in RFC2616, and what WebOb has implemented? (But
	  what WebOb has implemented doesn't look like "Basic Filtering", or
	  any filtering, since it only returns one single item?)
	* Compare the "Note:" at the end of the section and its example of the
	  user's assumption on selecting "en-gb", with the three matching
	  schemes in RFC4647. If I remember right, the filtering schemes work
	  one way, and lookup works the opposite way. Which one(s) sound like
	  the one in this example?
* RFC5646, Section 4.2:
	* "Languages that begin with the same sequence of subtags are NOT
	  guaranteed to be mutually intelligible." So a matching scheme that
	  matches a language range with a language tag that is a prefix of the
	  language range should not be implemented as the default scheme? (But
	  rather as an option?)
* RFC5646, Section 4.3:
	* "In some applications, a single content item might best be associated
	  with more than one language tag. Examples of such a usage
	  include:..." Examples of use cases where `.best_match()` does not
	  suffice?
* RFC4647, Secion 2.2:
	* "...wildcards outside the first position are ignored by Extended
	  Filtering (see Section 3.2.2)."
	* "The use or absence of one or more wildcards cannot be taken to imply
	  that a certain number of subtags will appear in the matching set of
	  language tags." Not sure what this means. I think it follows from the
	  way Extended Filtering ignores wildcards outside the first position,
	  where it drops those positions and shifts all following subtags up a
	  position? (Check how Extended Filtering works.)

* In `.best_match` docstring: "If two matches have equal weight, then the one
  that shows up first in the `offers` list will be returned." Why follow the
  `offers` order instead of the parsed order?
* In `Accept` class, why is `.parse()` a staticmethod?
* The way `.parse()` is used, what is the advantage to it being a generator?
* Is there any reason why the four Accept headers are based on the same class?
  Because it seems to make it harder to reason out the small differences
  between the four headers, where they need to be handled differently, and
  where shared code may be causing bugs. Might it be clearer to use composition
  instead of inheritance?


## Related GSoC project

https://github.com/Pylons/pyramid/wiki/GSoC-2017

"Re-work and fix WebOb's Accept handling for languages. This work could also be
extended to improving Pyramid's usage of webob for stronger accept handling
support."


## RFCs

* RFC7231
* RFC4647
* RFC3282
* RFC5646
* RFC4646 (obsoleted by 5646)
* RFC2616 (obsolete, but may explain previous implementation decisions)

### RFC7231

#### Section 3.1.3.1  Audience Language > Language Tags

* Defined in RFC5646.


#### Section 3.4.1.  Representations > Content Negotiation > Proactive Negotiation

Not directly relevant to issue, but background on proactive negotiation.


#### Section 5.3.  Request Header Fields > Content Negotiation

##### 5.3.1.  Quality Values

<pre>
weight = OWS ";" OWS "q=" qvalue
qvalue = ( "0" [ "." 0*3DIGIT ] )
       / ( "1" [ "." 0*3("0") ] )
</pre>

* Common parameter named "q" (case-insensitive) for many of the request header
  fields for proactive negotiation. Referred to as "quality value", or
  "qvalue"; is a relative weight.
* > "The same parameter name is often used within server configurations to
  assign a weight to the relative quality of the various representations that
  can be selected for a resource."
* > 'The weight is normalized to a real number in the range 0 through 1, where 0.001 is the least preferred and 1 is the most preferred; a value of 0 means "not acceptable". If no "q" parameter is present, the default weight is 1.'
* > "A sender of qvalue MUST NOT generate more than three digits after the
  decimal point. User configuration of these values ought to be limited in the
  same fashion."


##### 5.3.2.  Accept

<pre>
Accept = #( media-range [ accept-params ] )

media-range    = ( "*/*"
                 / ( type "/" "*" )
                 / ( type "/" subtype )
                 ) *( OWS ";" OWS parameter )
accept-params  = weight *( accept-ext )
accept-ext = OWS ";" OWS token [ "=" ( token / quoted-string ) ]
</pre>

* Used by user agents to specify response media types that are acceptable.


##### 5.3.5  Accept-Language

<pre>
Accept-Language = 1#( language-range [ weight ] )
language-range  =
          <language-range, see [RFC4647], Section 2.1>
</pre>

* Used by user agents to indicate the set of natural languages that are
  preferred in the response.
* Language tags are defined in Section 3.1.3.1.
* Each language range can be given an associated quality value representing an
  estimate of the user's preference for the languages specified by that range,
  as defined in Section 5.3.1.
* The example here involving `da`, `en-gb` and `en` here is confusing (so I
  won't quote it here): it doesn't make clear that British English would be
  preferred to English, which I believe is the intended meaning.
* A request without any `Accept-Language` header field implies that the user
  agent will accept any language in response.
* If the header field is present in a request and none of the available
  representations for the response have a matching language tag, the origin
  server can either disregard the header field by treating the response as if
  it is not subject to content negotiation, or honour the header field by
  sending a 406 (Not Acceptable) response. However, the latter is not
  encouraged as doing so can prevent users from accessing content that they
  might be able to use (with translation software, for example).
* Some recipients treat the order in which language tags are listed as an
  indication of descending priority, particularly for tags that are assigned
  equal quality values (no value is the same as q=1). However, this behaviour
  cannot be relied upon. For consistency and to maximize interoperability, many
  user agents assign each language tag a unique quality value while also
  listing them in order of decreasing quality.
* For matching, Section 3 of RFC4647 defines several matching schemes.
  Implementations can offer the most appropriate matching scheme for their
  requirements. [Unclear if they mean the most appropriate matching scheme of
  the three in RFC4647, or any appropriate scheme.] The "Basic Filtering"
  scheme ([RFC4647], Section 3.3.1) is identical to the matching scheme that
  was previous defined for HTTP in Section 14.4 of [RFC2616].
* In the "Note:" at the end of the section: 'For example, users might assume
  that on selecting "en-gb", they will be served any kind of English document
  if British English is not available. A user agent might suggest, in such a
  case, to add "en" to the list for better matching behavior.' This does not
  make clear whether users should not make that assumption because the chosen
  matching scheme (out of the three?) implemented might not work that way, or
  whether *none* of the matching schemes for this header should work that way.


### RFC2616

##### 14.4 Accept-Language

* A language-range matches a language-tag if it exactly equals the tag, or if
  it exactly equals a prefix of the tag such that the first tag character
  following the prefix is "-".
  [This is the same as "Basic Filtering" in RFC4647, except the mention of the
  comparison being case-insensitive in 4647. There's also a helpful example in
  RFC4647.]
* The special range "\*", if present in the Accept-Language field, matches every
  tag not matched by any other range present in the Accept-Language field.
  [Does that basically says "Use whatever language you like"?]
* The "Note:" on prefix matching rule is confusing, in that the section seems
  to specify one approach, but doesn't make clear whether the intention is for
  us to drop that approach if the approach doesn't appear appropriate, e.g.
  when a use understands a language with a certain tag, but not another
  language with another tag for which this tag is a prefix. (I believe this is
  possible -- check RFC4646?)
* It's completely unclear what the language quality factor is. Not mentioned in
  RFC7231 or RFC4647.


### RFC5646

##### 4.2.  Meaning of the Language Tag

> Language tags are related when they contain a similar sequence of subtags.  For
example, if a language tag B contains language tag A as a prefix, then B is
typically "narrower" or "more specific" than A.  Thus, "zh-Hant-TW" is more
specific than "zh-Hant".

> This relationship is not guaranteed in all cases: specifically, languages that
begin with the same sequence of subtags are NOT guaranteed to be mutually
intelligible, although they might be.  For example, the tag "az" shares a
prefix with both "az-Latn" (Azerbaijani written using the Latin script) and
"az-Cyrl" (Azerbaijani written using the Cyrillic script).  A person fluent in
one script might not be able to read the other, even though the linguistic
content (e.g., what would be heard if both texts were read aloud) might be
identical.  Content tagged as "az" most probably is written in just one script
and thus might not be intelligible to a reader familiar with the other script.


### RFC4647

#### 2.  The Language Range

> There are different types of language range, whose specific attributes vary
according to their application.  Language ranges are similar to language tags:
they consist of a sequence of subtags separated by hyphens.  In a language
range, each subtag MUST either be a sequence of ASCII alphanumeric characters
or the single character '*' (%x2A, ASTERISK).  The character '*' is a
"wildcard" that matches any sequence of subtags.  The meaning and uses of
wildcards vary according to the type of language range.

> Language tags and thus language ranges are to be treated as case- insensitive:
there exist conventions for the capitalization of some of the subtags, but
these MUST NOT be taken to carry meaning.  Matching of language tags to
language ranges MUST be done in a case- insensitive manner.

##### 2.1  Basic Language Range

> A "basic language range" has the same syntax as an [RFC3066] language tag or is
the single character "\*".  The basic language range was originally described by
HTTP/1.1 [RFC2616] and later [RFC3066].  It is defined by the following ABNF
[RFC4234]:

<pre>
language-range   = (1*8ALPHA *("-" 1*8alphanum)) / "*"
alphanum         = ALPHA / DIGIT
</pre>

> A basic language range differs from the language tags defined in [RFC4646] only
in that there is no requirement that it be "well- formed" or be validated
against the IANA Language Subtag Registry.  Such ill-formed ranges will
probably not match anything.  Note that the ABNF [RFC4234] in [RFC2616] is
incorrect, since it disallows the use of digits anywhere in the
'language-range' (see [RFC2616errata]).

##### 2.2  Extended Language Range

> Occasionally, users will wish to select a set of language tags based on the
presence of specific subtags.  An "extended language range" describes a user's
language preference as an ordered sequence of subtags.  For example, a user
might wish to select all language tags that contain the region subtag 'CH'
(Switzerland).  Extended language ranges are useful for specifying a particular
sequence of subtags that appear in the set of matching tags without having to
specify all of the intervening subtags.

> An extended language range can be represented by the following ABNF:

<pre>
extended-language-range = (1*8ALPHA / "*")
			  *("-" (1*8alphanum / "*"))
</pre>

> The wildcard subtag '\*' can occur in any position in the extended language
range, where it matches any sequence of subtags that might occur in that
position in a language tag.  However, wildcards outside the first position are
ignored by Extended Filtering (see Section 3.2.2).  The use or absence of one
or more wildcards cannot be taken to imply that a certain number of subtags
will appear in the matching set of language tags.

##### 2.3  The Language Priority List

Example:
> A speaker of Breton in France, for example, can specify "br" followed by "fr",
meaning that if Breton is available, it is preferred, but otherwise French is
the best alternative.

Refers to RFC2616 and RFC3282.

> A simple list of ranges is considered to be in descending order of priority.
Other language priority lists provide "quality weights" for the language ranges
in order to specify the relative priority of the user's language preferences.
An example of this is the use of "q" values in the syntax of the
"Accept-Language" header (defined in [RFC2616], Section 14.4, and [RFC3282]).


### RFC3282

#### The Accept-Language header

<blockquote>
The "Accept-Language" header is intended for use in cases where a user or a
process desires to identify the preferred language(s) when RFC 822-like
headers, such as MIME body parts or Web documents, are used.

The RFC 822 EBNF of the Accept-Language header is:

<pre>
Accept-Language = "Accept-Language" ":"
		       1#( language-range [ ";" "q" "=" qvalue ] )
</pre>

A slightly more restrictive RFC 2234 ABNF definition is:

<pre>
Accept-Language = "Accept-Language:" [CFWS] language-q
		  *( "," [CFWS] language-q )
language-q = language-range [";" [CFWS] "q=" qvalue ] [CFWS]
qvalue         = ( "0" [ "." 0*3DIGIT ] )
	       / ( "1" [ "." 0*3("0") ] )
</pre>

A more liberal RFC 2234 ABNF definition is:

<pre>
Obs-accept-language = "Accept-Language" *WSP ":" [CFWS]
	 obs-language-q *( "," [CFWS] obs-language-q ) [CFWS]
obs-language-q = language-range
	   [ [CFWS] ";" [CFWS] "q" [CFWS] "=" qvalue ]
</pre>

Like RFC 2822, this specification says that conforming implementations MUST
accept the obs-accept-language syntax, but MUST NOT generate it; all generated
messages MUST conform to the Accept- Language syntax.

The syntax and semantics of language-range is defined in [TAGS].  The
Accept-Language header may list several language-ranges in a comma- separated
list, and each may include a quality value Q.  If no Q values are given, the
language-ranges are given in priority order, with the leftmost language-range
being the most preferred language; this is an extension to the HTTP/1.1 rules,
but matches current practice.

If Q values are given, refer to HTTP/1.1 [RFC 2616] for the details on how to
evaluate it.
</blockquote>
