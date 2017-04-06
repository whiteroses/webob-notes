# Issue #256

https://github.com/Pylons/webob/issues/256


    >>> from webob import Request
    >>> r = Request.blank("/", accept_language="en-GB, en;q=0.8")
    >>> r.accept_language.best_match(offers=["en", "en-GB"])
    'en'


## To-dos

* RFC 7231, Section 5.3.1, on quality values:
	* "The same parameter name is often used within server configurations
	  to assign a weight to the relative quality of the various
	  representations that can be selected for a resource." Related to
	  `server_quality` in `.best_match()`?
	* 'a value of 0 means "not acceptable"'. Is this information important?
	  Is this information sometimes dropped by the current implementation?
	* Are incoming qvalues validated to check that they are not more than
	  three digits? Should they be? Do the use of floats and the resulting
	  floating point errors matter?
* In `Accept` class, why is `.parse()` a staticmethod?


## Related GSoC project

https://github.com/Pylons/pyramid/wiki/GSoC-2017

"Re-work and fix WebOb's Accept handling for languages. This work could also be
extended to improving Pyramid's usage of webob for stronger accept handling
support."


## RFCs

* RFC 7231
* RFC 4647


### RFC 7231

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

  Accept = #( media-range [ accept-params ] )

  media-range    = ( "*/*"
                   / ( type "/" "*" )
                   / ( type "/" subtype )
                   ) *( OWS ";" OWS parameter )
  accept-params  = weight *( accept-ext )
  accept-ext = OWS ";" OWS token [ "=" ( token / quoted-string ) ]

* Used by user agents to specify response media types that are acceptable.
