# Accept\* header issues in WebOb and Pyramid


## Questions

* First of all, confirm that we're no longer on 2616?
* Why do empty headers need separate NilAccept and NoAccept classes? Seems to
  lead to more duplicated code and complexity? Though I guess there are fewer
  conditional branches? NilAccept's __add__ and __radd__ look to be mostly
  duplicated --- why not implement like __add__ and __radd__ in Accept?
* Would like to make the argument to separate the implementations for the four
  headers, instead of the common base class as is done currently. It is
  difficult to reason and make changes to one, while having to worry about the
  small differences between all four. Common code could be re-used via
  composition rather than inheritance? See how much code actually would need to
  be repeated if done that way.
* The example for `Accept-Charset` in RFC7231 Section 5.3.3 involves charsets
  with two "-"s. Does WebOb handle this? (in relation to possible problem
  mentioned in Accept-Language issue)
* "\*"s seem to have slightly different meanings among the headers --- is WebOb
  handling them differently for the different headers?
* Does WebOb recognise the "identity" token in `Accept-Encoding`?
* With `Accept-Encoding`, a header field with a combined field-value that is
  empty implies that the user agent does not want any content-coding in
  response. Does WebOb handle this?
* Reading RFC7231 on the Accept\* headers, it doesn't seem as if finding a best
  match is necessarily the main functionality required for these headers. And
  yet implementations in both WebOb (and it seems, Werkzeug) seem to be centred
  around a `.best_match()` method. Is it because they were both implemented
  based on RFC2616? Check RFC2616 for clues.


## To-dos

* Read and understand the rest of RFCs on proactive negotiation.
* Read and understand WebOb's current handling of Accept\* headers.
* Write up current understanding of issues in WebOb.
* Read and understand issues in Pyramid. pyramid/config/views.py?


## Pyramid

* predicates.py, AcceptPredicate
	* Uses WebOb's `in` to match?
* config/views.py: MultiView.get_views()
* httpexceptions.py, HTTPException uses WebOb's MIMEAccept


* `Accept`
	* To indicate the media types that are acceptable.
	* Media type parameters not supported in WebOb?
	* Extension parameters not supported in WebOb?
	* Media ranges can be overridden by more specific media ranges or
	  specific media types. If more than one media range applies to a given
	  type, the most specific reference has precedence. In the example
	  given by the RFC, media type parameters (e.g.
	  `text/plain;format=flowed`) increases specificity.
	* So precedence is calculated by: sorting by specificity; then by
	  qvalue (see example re: media type quality factor). Position in list
	  not mentioned as relevant in RFC? (No tie-breaker?)
	* The media type quality factor associated with a given type is
	  determined by finding the media range with the highest precedence
	  that matches the type. (See example in RFC)
	* A request without any `Accept` header field implies that the user
	  agent will accept any media type in response.
	* Example: `Accept: text/plain; q=0.5, text/html, text/x-dvi; q=0.8, text/x-c`
* `Accept-Charset`
	* To indicate the charsets that are acceptable in textual response content.
	* The special value "\*", if present, matches every charset that is not
	  mentioned elsewhere in the field.
	* If no "\*" is present in the field, then any charsets not explicitly
	  mentioned in the field are considered "not acceptable" to the client.
	  This seems important.
	* A request without any `Accept-Charset` header field implies that the
	  user agent will accept any charset in response.
	* Example: `Accept-Charset: iso-8859-5, unicode-1-1;q=0.8`
* `Accept-Encoding`
	* To indicate the response content-codings that are acceptable (e.g.
	  compress and x-compress, deflate, gzip and x-gzip)
	* An "identity" token is used as a synonym for "no encoding" in order
	  to communicate when no encoding is preferred.
	* The asterisk "\*" symbol matches any available content-coding not
	  explicitly listed in the header field.
	* A request without an `Accept-Encoding` header field implies that the
	  user agent has no preferences regarding content-codings.
	* Algorithm in RFC.
	* The example appears to demonstrate multiple `Accept-Encoding` headers
	  in one request?
	* A header field with a combined field-value that is empty implies that
	  the user agent does not want any content-coding in response. What is
	  a "combined field-value"? Guessing that means combining the field
	  values from multiple `Accept-Encoding` headers in the same request?
* `Accept-Language`
