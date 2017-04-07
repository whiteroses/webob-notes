# Issue #256

https://github.com/Pylons/webob/issues/256


    >>> from webob import Request
    >>> r = Request.blank("/", accept_language="en-GB, en;q=0.8")
    >>> r.accept_language.best_match(offers=["en", "en-GB"])
    'en'

We would expect the best match to be "en-GB".


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
	  floating point errors matter?
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
	  it doesn't look like "Basic Filtering", or any filtering?)

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
* RFC5646
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


### RFC2616

##### 14.4 Accept Language

* A language-range matches a language-tag if it exactly equals the tag, or if
  it exactly equals a prefix of the tag such that the first tag character
  following the prefix is "-".
  [This is the same as "Basic Filtering" in RFC4647, except the mention of the
  comparison being case-insensitive in 4647. There's also a helpful example in
  RFC4647.]
