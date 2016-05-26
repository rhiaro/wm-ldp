# Webmention <-> LDP bridge

## From a webmention implementation 

Could be considered an extension to the webmention spec. Ie. if you have implemented sending and receiving webmentions, and want to also send them in a way that LDP servers will accept, and receive updates from applications which expect to be writing to LDP servers.

### Sending

Publish a document (`<source>`) with a link to another document (`<target>`).

Parse `<target>` for a webmention endpoint (`<ep>`) according to the webmention spec, but also check for `http://w3.org/ns/webmention#webmention`:
* check Link headers first as normal.
* if the target is served as HTML check for the rel value in the body as normal.
* if the target is served as RDF (most likely Turtle or JSON-LD) look for the value of property `http://w3.org/ns/webmention#webmention`.

Send a POST request with RDF in the body RDF to `<ep>`

```
@prefix wm: <http://w3.org/ns/webmention#> .
<> a wm:Webmention ;
   wm:source <source> ;
   wm:target <target> .
```

Even if this doesn't automatically trigger verification and display anywhere on the LDP end, it is at least written as data the LDP server can read.

### Receiving

Advertise your webmention endpoint in the document body (using `<link>` or `<a>` according to the webmention spec), but with a URL (`http://w3.org/ns/webmention#webmention`) so it can be parsed as RDFa.

Convert a POST request received to your endpoint with RDF in the body from:

```
@prefix wm: <http://w3.org/ns/webmention#> .
<> a wm:Webmention ;
   wm:source <source> ;
   wm:target <target> .
```

or

```
{
  "@context": "http://www.w3.org/ns/webmention#",
  "@type": "Webmention",
  "source": "...source...",
  "target": "...target..."
}
```

to: `source=<source>&target=<target>` and proceed with validation. Note that you may want to parse the `<source>` document as RDFa, Turtle or JSON-LD during verification to grab the contents of the source, though string-matching for the `<target>` URL will still work on all of these if you don't care about the semantics of the notification.

## From a LDP implementation

Consider this an extension to the LDP spec. Ie. if you have an LDP server and want to modify it to receive and process webmentions correctly, or if you have Solid application and you want to use it to send webmentions.

### Sending

This could be implemented by a clientside Solid application, or an extension to an LDP server (eg. a script that is triggered to check for links out whenever a new resource is created).

Publish a document (`<s>`) with a link (`<p>`) to another document (`<o>`), where `<s>` is accessible by `<o>`.

This could feasibly also be a qualified relation like:

```
@prefix <http://www.w3.org/ns/activitystreams#> .
<s> a as:Like ;
   as:object <o> ;
   as:actor <you> ;
   as:published "2015-05-25" .
```

Parse `<o>` for a webmention endoint according to webmention spec and interpret it as `<> wm:webmention <ep>` which is essentially a Container into which webmentions are writeable.

Convert notification from `<s> <p> <o>` (or as appropriate for your `<s>`) to `source=<s>&target=<o>` and send a form-encoded POST request to `<ep>`.

### Receiving

Resources must advertise the container into which received webmentions are writeable (the 'webmention endpoint') via link `rel="webmention"`. This can be a HTTP header or html `<link>` or `<a>` element.

* Issues: non-URI rel value will not be parsed as RDFa so this might not be desireable. HTTP header is probably acceptable though. (Seealso: https://github.com/mnot/I-D/issues/39 about IANA URLs as link relations which is ultimately 'wontfix').

Interpret an incoming form-encoded POST requests to this Container `source=<s>&target=<o>` as 

```
@prefix wm: <http://www.w3.org/ns/webmention#> .
<> a wm:Webmention ;
   wm:source <s> ;
   wm:target <o> .
```

(or some other structure that you prefer, maybe you want to translate it to AS2) and create this RDF resource according to LDP.

Return a 201 or 202 and the URL of the new resource, which is in accordance with both Webmention and LDP specs.

Validate webmention by parsing `<o>` for a link to `<s>`. Add `<> :verified "date"` to newly created webmention resource if verified.
