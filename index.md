# Webmention to LDP bridge

## From a webmention implementation 

Consider this an extension to the webmention spec.

### Sending

Publish a document (`<source>`) with a link to another document (`<target>`).

Parse `<target>` for a webmention endpoint (`<ep>`) according to the webmention spec, but also check for `http://w3.org/ns/webmention#webmention` or `http://www.iana.org/assignments/relation/webmention` as the rel value.

Send a POST request with rdf in the body RDF to `<ep>`

```
@prefix wm: <http://w3.org/ns/webmention#> .
<> a wm:Webmention ;
   wm:source <source> ;
   wm:target <target> .
```

### Receiving

Advertise your webmention endpoint in the document body (using `<link>` or `<a>` according to the webmention spec), but with a URL (`http://w3.org/ns/webmention#webmention`) so it can be parsed as RDFa.

Convert a POST request with RDF in the body from

```
@prefix wm: <http://w3.org/ns/webmention#> .
<> a wm:Webmention ;
   wm:source <source> ;
   wm:target <target> .
```

to `source=<source>&target=<target>` and proceed with validation. Note that you may need to parse the `<source>` document as RDFa, turtle or JSON-LD to validate (string-matching for the `<target>` URL will still work on all of these if you don't care about the semantics of the notification).

## From a LDP implementation

Consider this an extension to the LDP spec

### Sending

This could be implemented by a clientside Solid application, or an extension to an LDP server (eg. a script that is triggered to check for links out whenever a new resource is created).

Publish a document (`<s>`) with a link (`<p>`) to another document (`<o>`), where `<s>` is accessible by `<o>`.

Parse `<o>` for a webmention endoint according to webmention spec and interpret it as `<> wm:webmention <ep>` which is essentially a container into which webmentions are writeable.

Convert notification from `<s> <p> <o>` to `source=<s>&target=<o>` and send a form-encoded POST request to `<ep>`.

### Receiving

Resources must advertise the container into which received webmentions are writeable (the 'webmention endpoint') via link `rel="webmention"`. This can be a HTTP header or html `<link>` or `<a>` element.

* Issues: non-URI rel value will not be parsed as RDFa so LD purists probably aren't going to do it. HTTP header route might be acceptable though. (Seealso: https://github.com/mnot/I-D/issues/39)
** Possible solution that involves updating wm spec: add http://www.iana.org/assignments/relation/webmention or w3.org/ns/webmention as a value that everyone must check for.

Interpret an incoming form-encoded POST request `source=<s>&target=<o>` as 

```
@prefix wm: <http://w3.org/ns/webmention#> .
<> a wm:Webmention ;
   wm:source <s> ;
   wm:target <o> .
```

and create this RDF resource according to LDP.

Return a 201 or 202 and the URL of the new resource, which is in accordance with both Webmention and LDP specs.

Validate webmention by parsing `<o>` for a link to `<s>`. Add `<> :verified "date"` to newly created webmention resource if verified.