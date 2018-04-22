---
title: PRS-7
category: PSRs
order: 6
---

| Informações Adicionais                                           |
| -----------------------------------------------------------------|
| [HTTP message interfaces][HTTP message interfaces]               |
| [HTTP Message Meta Document][HTTP Message Meta Document]         |

[HTTP message interfaces]: #HTTP message interfaces
[HTTP Message Meta Document]: #HTTP Message Meta Document

<h1 id="HTTP message interfaces">HTTP message interfaces</h1>

This document describes common interfaces for representing HTTP messages as
described in [RFC 7230](http://tools.ietf.org/html/rfc7230) and
[RFC 7231](http://tools.ietf.org/html/rfc7231), and URIs for use with HTTP
messages as described in [RFC 3986](http://tools.ietf.org/html/rfc3986).

HTTP messages are the foundation of web development. Web browsers and HTTP
clients such as cURL create HTTP request messages that are sent to a web server,
which provides an HTTP response message. Server-side code receives an HTTP
request message, and returns an HTTP response message.

HTTP messages are typically abstracted from the end-user consumer, but as
developers, we typically need to know how they are structured and how to
access or manipulate them in order to perform our tasks, whether that might be
making a request to an HTTP API, or handling an incoming request.

Every HTTP request message has a specific form:

~~~http
POST /path HTTP/1.1
Host: example.com

foo=bar&baz=bat
~~~

The first line of a request is the "request line", and contains, in order, the
HTTP request method, the request target (usually either an absolute URI or a
path on the web server), and the HTTP protocol version. This is followed by one
or more HTTP headers, an empty line, and the message body.

HTTP response messages have a similar structure:

~~~http
HTTP/1.1 200 OK
Content-Type: text/plain

This is the response body
~~~

The first line is the "status line", and contains, in order, the HTTP protocol
version, the HTTP status code, and a "reason phrase," a human-readable
description of the status code. Like the request message, this is then
followed by one or more HTTP headers, an empty line, and the message body.

The interfaces described in this document are abstractions around HTTP messages
and the elements composing them.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](http://tools.ietf.org/html/rfc2119).

### References

- [RFC 2119](http://tools.ietf.org/html/rfc2119)
- [RFC 3986](http://tools.ietf.org/html/rfc3986)
- [RFC 7230](http://tools.ietf.org/html/rfc7230)
- [RFC 7231](http://tools.ietf.org/html/rfc7231)

## 1. Specification

### 1.1 Messages

An HTTP message is either a request from a client to a server or a response from
a server to a client. This specification defines interfaces for the HTTP messages
`Psr\Http\Message\RequestInterface` and `Psr\Http\Message\ResponseInterface` respectively.

Both `Psr\Http\Message\RequestInterface` and `Psr\Http\Message\ResponseInterface` extend
`Psr\Http\Message\MessageInterface`. While `Psr\Http\Message\MessageInterface` MAY be
implemented directly, implementors SHOULD implement
`Psr\Http\Message\RequestInterface` and `Psr\Http\Message\ResponseInterface`.

From here forward, the namespace `Psr\Http\Message` will be omitted when
referring to these interfaces.

### 1.2 HTTP Headers

#### Case-insensitive header field names

HTTP messages include case-insensitive header field names. Headers are retrieved
by name from classes implementing the `MessageInterface` in a case-insensitive
manner. For example, retrieving the `foo` header will return the same result as
retrieving the `FoO` header. Similarly, setting the `Foo` header will overwrite
any previously set `foo` header value.

~~~php
$message = $message->withHeader('foo', 'bar');

echo $message->getHeaderLine('foo');
// Outputs: bar

echo $message->getHeaderLine('FOO');
// Outputs: bar

$message = $message->withHeader('fOO', 'baz');
echo $message->getHeaderLine('foo');
// Outputs: baz
~~~

Despite that headers may be retrieved case-insensitively, the original case
MUST be preserved by the implementation, in particular when retrieved with
`getHeaders()`.

Non-conforming HTTP applications may depend on a certain case, so it is useful
for a user to be able to dictate the case of the HTTP headers when creating a
request or response.

#### Headers with multiple values

In order to accommodate headers with multiple values yet still provide the
convenience of working with headers as strings, headers can be retrieved from
an instance of a `MessageInterface` as an array or a string. Use the
`getHeaderLine()` method to retrieve a header value as a string containing all
header values of a case-insensitive header by name concatenated with a comma.
Use `getHeader()` to retrieve an array of all the header values for a
particular case-insensitive header by name.

~~~php
$message = $message
    ->withHeader('foo', 'bar')
    ->withAddedHeader('foo', 'baz');

$header = $message->getHeaderLine('foo');
// $header contains: 'bar, baz'

$header = $message->getHeader('foo');
// ['bar', 'baz']
~~~

Note: Not all header values can be concatenated using a comma (e.g.,
`Set-Cookie`). When working with such headers, consumers of
`MessageInterface`-based classes SHOULD rely on the `getHeader()` method
for retrieving such multi-valued headers.

#### Host header

In requests, the `Host` header typically mirrors the host component of the URI, as
well as the host used when establishing the TCP connection. However, the HTTP
specification allows the `Host` header to differ from each of the two.

During construction, implementations MUST attempt to set the `Host` header from
a provided URI if no `Host` header is provided.

`RequestInterface::withUri()` will, by default, replace the returned request's
`Host` header with a `Host` header matching the host component of the passed
`UriInterface`.

You can opt-in to preserving the original state of the `Host` header by passing
`true` for the second (`$preserveHost`) argument. When this argument is set to
`true`, the returned request will not update the `Host` header of the returned
message -- unless the message contains no `Host` header.

This table illustrates what `getHeaderLine('Host')` will return for a request
returned by `withUri()` with the `$preserveHost` argument set to `true` for
various initial requests and URIs.

Request Host header<sup>[1](#rhh)</sup> | Request host component<sup>[2](#rhc)</sup> | URI host component<sup>[3](#uhc)</sup> | Result
----------------------------------------|--------------------------------------------|----------------------------------------|--------
''                                      | ''                                         | ''                                     | ''
''                                      | foo.com                                    | ''                                     | foo.com
''                                      | foo.com                                    | bar.com                                | foo.com
foo.com                                 | ''                                         | bar.com                                | foo.com
foo.com                                 | bar.com                                    | baz.com                                | foo.com

- <sup id="rhh">1</sup> `Host` header value prior to operation.
- <sup id="rhc">2</sup> Host component of the URI composed in the request prior
  to the operation.
- <sup id="uhc">3</sup> Host component of the URI being injected via
  `withUri()`.

### 1.3 Streams

HTTP messages consist of a start-line, headers, and a body. The body of an HTTP
message can be very small or extremely large. Attempting to represent the body
of a message as a string can easily consume more memory than intended because
the body must be stored completely in memory. Attempting to store the body of a
request or response in memory would preclude the use of that implementation from
being able to work with large message bodies. `StreamInterface` is used in
order to hide the implementation details when a stream of data is read from
or written to. For situations where a string would be an appropriate message
implementation, built-in streams such as `php://memory` and `php://temp` may be
used.

`StreamInterface` exposes several methods that enable streams to be read
from, written to, and traversed effectively.

Streams expose their capabilities using three methods: `isReadable()`,
`isWritable()`, and `isSeekable()`. These methods can be used by stream
collaborators to determine if a stream is capable of their requirements.

Each stream instance will have various capabilities: it can be read-only,
write-only, or read-write. It can also allow arbitrary random access (seeking
forwards or backwards to any location), or only sequential access (for
example in the case of a socket, pipe, or callback-based stream).

Finally, `StreamInterface` defines a `__toString()` method to simplify
retrieving or emitting the entire body contents at once.

Unlike the request and response interfaces, `StreamInterface` does not model
immutability. In situations where an actual PHP stream is wrapped, immutability
is impossible to enforce, as any code that interacts with the resource can
potentially change its state (including cursor position, contents, and more).
Our recommendation is that implementations use read-only streams for
server-side requests and client-side responses. Consumers should be aware of
the fact that the stream instance may be mutable, and, as such, could alter
the state of the message; when in doubt, create a new stream instance and attach
it to a message to enforce state.

### 1.4 Request Targets and URIs

Per RFC 7230, request messages contain a "request-target" as the second segment
of the request line. The request target can be one of the following forms:

- **origin-form**, which consists of the path, and, if present, the query
  string; this is often referred to as a relative URL. Messages as transmitted
  over TCP typically are of origin-form; scheme and authority data are usually
  only present via CGI variables.
- **absolute-form**, which consists of the scheme, authority
  ("[user-info@]host[:port]", where items in brackets are optional), path (if
  present), query string (if present), and fragment (if present). This is often
  referred to as an absolute URI, and is the only form to specify a URI as
  detailed in RFC 3986. This form is commonly used when making requests to
  HTTP proxies.
- **authority-form**, which consists of the authority only. This is typically
  used in CONNECT requests only, to establish a connection between an HTTP
  client and a proxy server.
- **asterisk-form**, which consists solely of the string `*`, and which is used
  with the OPTIONS method to determine the general capabilities of a web server.

Aside from these request-targets, there is often an 'effective URL' which is
separate from the request target. The effective URL is not transmitted within
an HTTP message, but it is used to determine the protocol (http/https), port
and hostname for making the request.

The effective URL is represented by `UriInterface`. `UriInterface` models HTTP
and HTTPS URIs as specified in RFC 3986 (the primary use case). The interface
provides methods for interacting with the various URI parts, which will obviate
the need for repeated parsing of the URI. It also specifies a `__toString()`
method for casting the modeled URI to its string representation.

When retrieving the request-target with `getRequestTarget()`, by default this
method will use the URI object and extract all the necessary components to
construct the _origin-form_. The _origin-form_ is by far the most common
request-target.

If it's desired by an end-user to use one of the other three forms, or if the
user wants to explicitly override the request-target, it is possible to do so
with `withRequestTarget()`.

Calling this method does not affect the URI, as it is returned from `getUri()`.

For example, a user may want to make an asterisk-form request to a server:

~~~php
$request = $request
    ->withMethod('OPTIONS')
    ->withRequestTarget('*')
    ->withUri(new Uri('https://example.org/'));
~~~

This example may ultimately result in an HTTP request that looks like this:

~~~http
OPTIONS * HTTP/1.1
~~~

But the HTTP client will be able to use the effective URL (from `getUri()`),
to determine the protocol, hostname and TCP port.

An HTTP client MUST ignore the values of `Uri::getPath()` and `Uri::getQuery()`,
and instead use the value returned by `getRequestTarget()`, which defaults
to concatenating these two values.

Clients that choose to not implement 1 or more of the 4 request-target forms,
MUST still use `getRequestTarget()`. These clients MUST reject request-targets
they do not support, and MUST NOT fall back on the values from `getUri()`.

`RequestInterface` provides methods for retrieving the request-target or
creating a new instance with the provided request-target. By default, if no
request-target is specifically composed in the instance, `getRequestTarget()`
will return the origin-form of the composed URI (or "/" if no URI is composed).
`withRequestTarget($requestTarget)` creates a new instance with the
specified request target, and thus allows developers to create request messages
that represent the other three request-target forms (absolute-form,
authority-form, and asterisk-form). When used, the composed URI instance can
still be of use, particularly in clients, where it may be used to create the
connection to the server.

### 1.5 Server-side Requests

`RequestInterface` provides the general representation of an HTTP request
message. However, server-side requests need additional treatment, due to the
nature of the server-side environment. Server-side processing needs to take into
account Common Gateway Interface (CGI), and, more specifically, PHP's
abstraction and extension of CGI via its Server APIs (SAPI). PHP has provided
simplification around input marshaling via superglobals such as:

- `$_COOKIE`, which deserializes and provides simplified access to HTTP
  cookies.
- `$_GET`, which deserializes and provides simplified access to query string
  arguments.
- `$_POST`, which deserializes and provides simplified access for urlencoded
  parameters submitted via HTTP POST; generically, it can be considered the
  results of parsing the message body.
- `$_FILES`, which provides serialized metadata around file uploads.
- `$_SERVER`, which provides access to CGI/SAPI environment variables, which
  commonly include the request method, the request scheme, the request URI, and
  headers.

`ServerRequestInterface` extends `RequestInterface` to provide an abstraction
around these various superglobals. This practice helps reduce coupling to the
superglobals by consumers, and encourages and promotes the ability to test
request consumers.

The server request provides one additional property, "attributes", to allow
consumers the ability to introspect, decompose, and match the request against
application-specific rules (such as path matching, scheme matching, host
matching, etc.). As such, the server request can also provide messaging between
multiple request consumers.

### 1.6 Uploaded files

`ServerRequestInterface` specifies a method for retrieving a tree of upload
files in a normalized structure, with each leaf an instance of
`UploadedFileInterface`.

The `$_FILES` superglobal has some well-known problems when dealing with arrays
of file inputs. As an example, if you have a form that submits an array of files
— e.g., the input name "files", submitting `files[0]` and `files[1]` — PHP will
represent this as:

~~~php
array(
    'files' => array(
        'name' => array(
            0 => 'file0.txt',
            1 => 'file1.html',
        ),
        'type' => array(
            0 => 'text/plain',
            1 => 'text/html',
        ),
        /* etc. */
    ),
)
~~~

instead of the expected:

~~~php
array(
    'files' => array(
        0 => array(
            'name' => 'file0.txt',
            'type' => 'text/plain',
            /* etc. */
        ),
        1 => array(
            'name' => 'file1.html',
            'type' => 'text/html',
            /* etc. */
        ),
    ),
)
~~~

The result is that consumers need to know this language implementation detail,
and write code for gathering the data for a given upload.

Additionally, scenarios exist where `$_FILES` is not populated when file uploads
occur:

- When the HTTP method is not `POST`.
- When unit testing.
- When operating under a non-SAPI environment, such as [ReactPHP](http://reactphp.org).

In such cases, the data will need to be seeded differently. As examples:

- A process might parse the message body to discover the file uploads. In such
  cases, the implementation may choose *not* to write the file uploads to the
  file system, but instead wrap them in a stream in order to reduce memory,
  I/O, and storage overhead.
- In unit testing scenarios, developers need to be able to stub and/or mock the
  file upload metadata in order to validate and verify different scenarios.

`getUploadedFiles()` provides the normalized structure for consumers.
Implementations are expected to:

- Aggregate all information for a given file upload, and use it to populate a
  `Psr\Http\Message\UploadedFileInterface` instance.
- Re-create the submitted tree structure, with each leaf being the appropriate
  `Psr\Http\Message\UploadedFileInterface` instance for the given location in
  the tree.

The tree structure referenced should mimic the naming structure in which files
were submitted.

In the simplest example, this might be a single named form element submitted as:

~~~html
<input type="file" name="avatar" />
~~~

In this case, the structure in `$_FILES` would look like:

~~~php
array(
    'avatar' => array(
        'tmp_name' => 'phpUxcOty',
        'name' => 'my-avatar.png',
        'size' => 90996,
        'type' => 'image/png',
        'error' => 0,
    ),
)
~~~

The normalized form returned by `getUploadedFiles()` would be:

~~~php
array(
    'avatar' => /* UploadedFileInterface instance */
)
~~~

In the case of an input using array notation for the name:

~~~html
<input type="file" name="my-form[details][avatar]" />
~~~

`$_FILES` ends up looking like this:

~~~php
array(
    'my-form' => array(
        'details' => array(
            'avatar' => array(
                'tmp_name' => 'phpUxcOty',
                'name' => 'my-avatar.png',
                'size' => 90996,
                'type' => 'image/png',
                'error' => 0,
            ),
        ),
    ),
)
~~~

And the corresponding tree returned by `getUploadedFiles()` should be:

~~~php
array(
    'my-form' => array(
        'details' => array(
            'avatar' => /* UploadedFileInterface instance */
        ),
    ),
)
~~~

In some cases, you may specify an array of files:

~~~html
Upload an avatar: <input type="file" name="my-form[details][avatars][]" />
Upload an avatar: <input type="file" name="my-form[details][avatars][]" />
~~~

(As an example, JavaScript controls might spawn additional file upload inputs to
allow uploading multiple files at once.)

In such a case, the specification implementation must aggregate all information
related to the file at the given index. The reason is because `$_FILES` deviates
from its normal structure in such cases:

~~~php
array(
    'my-form' => array(
        'details' => array(
            'avatars' => array(
                'tmp_name' => array(
                    0 => '...',
                    1 => '...',
                    2 => '...',
                ),
                'name' => array(
                    0 => '...',
                    1 => '...',
                    2 => '...',
                ),
                'size' => array(
                    0 => '...',
                    1 => '...',
                    2 => '...',
                ),
                'type' => array(
                    0 => '...',
                    1 => '...',
                    2 => '...',
                ),
                'error' => array(
                    0 => '...',
                    1 => '...',
                    2 => '...',
                ),
            ),
        ),
    ),
)
~~~

The above `$_FILES` array would correspond to the following structure as
returned by `getUploadedFiles()`:

~~~php
array(
    'my-form' => array(
        'details' => array(
            'avatars' => array(
                0 => /* UploadedFileInterface instance */,
                1 => /* UploadedFileInterface instance */,
                2 => /* UploadedFileInterface instance */,
            ),
        ),
    ),
)
~~~

Consumers would access index `1` of the nested array using:

~~~php
$request->getUploadedFiles()['my-form']['details']['avatars'][1];
~~~

Because the uploaded files data is derivative (derived from `$_FILES` or the
request body), a mutator method, `withUploadedFiles()`, is also present in the
interface, allowing delegation of the normalization to another process.

In the case of the original examples, consumption resembles the following:

~~~php
$file0 = $request->getUploadedFiles()['files'][0];
$file1 = $request->getUploadedFiles()['files'][1];

printf(
    "Received the files %s and %s",
    $file0->getClientFilename(),
    $file1->getClientFilename()
);

// "Received the files file0.txt and file1.html"
~~~

This proposal also recognizes that implementations may operate in non-SAPI
environments. As such, `UploadedFileInterface` provides methods for ensuring
operations will work regardless of environment. In particular:

- `moveTo($targetPath)` is provided as a safe and recommended alternative to calling
  `move_uploaded_file()` directly on the temporary upload file. Implementations
  will detect the correct operation to use based on environment.
- `getStream()` will return a `StreamInterface` instance. In non-SAPI
  environments, one proposed possibility is to parse individual upload files
  into `php://temp` streams instead of directly to files; in such cases, no
  upload file is present. `getStream()` is therefore guaranteed to work
  regardless of environment.

As examples:

~~~
// Move a file to an upload directory
$filename = sprintf(
    '%s.%s',
    create_uuid(),
    pathinfo($file0->getClientFilename(), PATHINFO_EXTENSION)
);
$file0->moveTo(DATA_DIR . '/' . $filename);

// Stream a file to Amazon S3.
// Assume $s3wrapper is a PHP stream that will write to S3, and that
// Psr7StreamWrapper is a class that will decorate a StreamInterface as a PHP
// StreamWrapper.
$stream = new Psr7StreamWrapper($file1->getStream());
stream_copy_to_stream($stream, $s3wrapper);
~~~

## 2. Package

The interfaces and classes described are provided as part of the
[psr/http-message](https://packagist.org/packages/psr/http-message) package.

## 3. Interfaces

### 3.1 `Psr\Http\Message\MessageInterface`

~~~php
<?php
namespace Psr\Http\Message;

/**
 * HTTP messages consist of requests from a client to a server and responses
 * from a server to a client. This interface defines the methods common to
 * each.
 *
 * Messages are considered immutable; all methods that might change state MUST
 * be implemented such that they retain the internal state of the current
 * message and return an instance that contains the changed state.
 *
 * @see http://www.ietf.org/rfc/rfc7230.txt
 * @see http://www.ietf.org/rfc/rfc7231.txt
 */
interface MessageInterface
{
    /**
     * Retrieves the HTTP protocol version as a string.
     *
     * The string MUST contain only the HTTP version number (e.g., "1.1", "1.0").
     *
     * @return string HTTP protocol version.
     */
    public function getProtocolVersion();

    /**
     * Return an instance with the specified HTTP protocol version.
     *
     * The version string MUST contain only the HTTP version number (e.g.,
     * "1.1", "1.0").
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * new protocol version.
     *
     * @param string $version HTTP protocol version
     * @return static
     */
    public function withProtocolVersion($version);

    /**
     * Retrieves all message header values.
     *
     * The keys represent the header name as it will be sent over the wire, and
     * each value is an array of strings associated with the header.
     *
     *     // Represent the headers as a string
     *     foreach ($message->getHeaders() as $name => $values) {
     *         echo $name . ': ' . implode(', ', $values);
     *     }
     *
     *     // Emit headers iteratively:
     *     foreach ($message->getHeaders() as $name => $values) {
     *         foreach ($values as $value) {
     *             header(sprintf('%s: %s', $name, $value), false);
     *         }
     *     }
     *
     * While header names are not case-sensitive, getHeaders() will preserve the
     * exact case in which headers were originally specified.
     *
     * @return string[][] Returns an associative array of the message's headers.
     *     Each key MUST be a header name, and each value MUST be an array of
     *     strings for that header.
     */
    public function getHeaders();

    /**
     * Checks if a header exists by the given case-insensitive name.
     *
     * @param string $name Case-insensitive header field name.
     * @return bool Returns true if any header names match the given header
     *     name using a case-insensitive string comparison. Returns false if
     *     no matching header name is found in the message.
     */
    public function hasHeader($name);

    /**
     * Retrieves a message header value by the given case-insensitive name.
     *
     * This method returns an array of all the header values of the given
     * case-insensitive header name.
     *
     * If the header does not appear in the message, this method MUST return an
     * empty array.
     *
     * @param string $name Case-insensitive header field name.
     * @return string[] An array of string values as provided for the given
     *    header. If the header does not appear in the message, this method MUST
     *    return an empty array.
     */
    public function getHeader($name);

    /**
     * Retrieves a comma-separated string of the values for a single header.
     *
     * This method returns all of the header values of the given
     * case-insensitive header name as a string concatenated together using
     * a comma.
     *
     * NOTE: Not all header values may be appropriately represented using
     * comma concatenation. For such headers, use getHeader() instead
     * and supply your own delimiter when concatenating.
     *
     * If the header does not appear in the message, this method MUST return
     * an empty string.
     *
     * @param string $name Case-insensitive header field name.
     * @return string A string of values as provided for the given header
     *    concatenated together using a comma. If the header does not appear in
     *    the message, this method MUST return an empty string.
     */
    public function getHeaderLine($name);

    /**
     * Return an instance with the provided value replacing the specified header.
     *
     * While header names are case-insensitive, the casing of the header will
     * be preserved by this function, and returned from getHeaders().
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * new and/or updated header and value.
     *
     * @param string $name Case-insensitive header field name.
     * @param string|string[] $value Header value(s).
     * @return static
     * @throws \InvalidArgumentException for invalid header names or values.
     */
    public function withHeader($name, $value);

    /**
     * Return an instance with the specified header appended with the given value.
     *
     * Existing values for the specified header will be maintained. The new
     * value(s) will be appended to the existing list. If the header did not
     * exist previously, it will be added.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * new header and/or value.
     *
     * @param string $name Case-insensitive header field name to add.
     * @param string|string[] $value Header value(s).
     * @return static
     * @throws \InvalidArgumentException for invalid header names.
     * @throws \InvalidArgumentException for invalid header values.
     */
    public function withAddedHeader($name, $value);

    /**
     * Return an instance without the specified header.
     *
     * Header resolution MUST be done without case-sensitivity.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that removes
     * the named header.
     *
     * @param string $name Case-insensitive header field name to remove.
     * @return static
     */
    public function withoutHeader($name);

    /**
     * Gets the body of the message.
     *
     * @return StreamInterface Returns the body as a stream.
     */
    public function getBody();

    /**
     * Return an instance with the specified message body.
     *
     * The body MUST be a StreamInterface object.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return a new instance that has the
     * new body stream.
     *
     * @param StreamInterface $body Body.
     * @return static
     * @throws \InvalidArgumentException When the body is not valid.
     */
    public function withBody(StreamInterface $body);
}
~~~

### 3.2 `Psr\Http\Message\RequestInterface`

~~~php
<?php
namespace Psr\Http\Message;

/**
 * Representation of an outgoing, client-side request.
 *
 * Per the HTTP specification, this interface includes properties for
 * each of the following:
 *
 * - Protocol version
 * - HTTP method
 * - URI
 * - Headers
 * - Message body
 *
 * During construction, implementations MUST attempt to set the Host header from
 * a provided URI if no Host header is provided.
 *
 * Requests are considered immutable; all methods that might change state MUST
 * be implemented such that they retain the internal state of the current
 * message and return an instance that contains the changed state.
 */
interface RequestInterface extends MessageInterface
{
    /**
     * Retrieves the message's request target.
     *
     * Retrieves the message's request-target either as it will appear (for
     * clients), as it appeared at request (for servers), or as it was
     * specified for the instance (see withRequestTarget()).
     *
     * In most cases, this will be the origin-form of the composed URI,
     * unless a value was provided to the concrete implementation (see
     * withRequestTarget() below).
     *
     * If no URI is available, and no request-target has been specifically
     * provided, this method MUST return the string "/".
     *
     * @return string
     */
    public function getRequestTarget();

    /**
     * Return an instance with the specific request-target.
     *
     * If the request needs a non-origin-form request-target — e.g., for
     * specifying an absolute-form, authority-form, or asterisk-form —
     * this method may be used to create an instance with the specified
     * request-target, verbatim.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * changed request target.
     *
     * @see http://tools.ietf.org/html/rfc7230#section-5.3 (for the various
     *     request-target forms allowed in request messages)
     * @param mixed $requestTarget
     * @return static
     */
    public function withRequestTarget($requestTarget);

    /**
     * Retrieves the HTTP method of the request.
     *
     * @return string Returns the request method.
     */
    public function getMethod();

    /**
     * Return an instance with the provided HTTP method.
     *
     * While HTTP method names are typically all uppercase characters, HTTP
     * method names are case-sensitive and thus implementations SHOULD NOT
     * modify the given string.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * changed request method.
     *
     * @param string $method Case-sensitive method.
     * @return static
     * @throws \InvalidArgumentException for invalid HTTP methods.
     */
    public function withMethod($method);

    /**
     * Retrieves the URI instance.
     *
     * This method MUST return a UriInterface instance.
     *
     * @see http://tools.ietf.org/html/rfc3986#section-4.3
     * @return UriInterface Returns a UriInterface instance
     *     representing the URI of the request.
     */
    public function getUri();

    /**
     * Returns an instance with the provided URI.
     *
     * This method MUST update the Host header of the returned request by
     * default if the URI contains a host component. If the URI does not
     * contain a host component, any pre-existing Host header MUST be carried
     * over to the returned request.
     *
     * You can opt-in to preserving the original state of the Host header by
     * setting `$preserveHost` to `true`. When `$preserveHost` is set to
     * `true`, this method interacts with the Host header in the following ways:
     *
     * - If the Host header is missing or empty, and the new URI contains
     *   a host component, this method MUST update the Host header in the returned
     *   request.
     * - If the Host header is missing or empty, and the new URI does not contain a
     *   host component, this method MUST NOT update the Host header in the returned
     *   request.
     * - If a Host header is present and non-empty, this method MUST NOT update
     *   the Host header in the returned request.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * new UriInterface instance.
     *
     * @see http://tools.ietf.org/html/rfc3986#section-4.3
     * @param UriInterface $uri New request URI to use.
     * @param bool $preserveHost Preserve the original state of the Host header.
     * @return static
     */
    public function withUri(UriInterface $uri, $preserveHost = false);
}
~~~

#### 3.2.1 `Psr\Http\Message\ServerRequestInterface`

~~~php
<?php
namespace Psr\Http\Message;

/**
 * Representation of an incoming, server-side HTTP request.
 *
 * Per the HTTP specification, this interface includes properties for
 * each of the following:
 *
 * - Protocol version
 * - HTTP method
 * - URI
 * - Headers
 * - Message body
 *
 * Additionally, it encapsulates all data as it has arrived at the
 * application from the CGI and/or PHP environment, including:
 *
 * - The values represented in $_SERVER.
 * - Any cookies provided (generally via $_COOKIE)
 * - Query string arguments (generally via $_GET, or as parsed via parse_str())
 * - Upload files, if any (as represented by $_FILES)
 * - Deserialized body parameters (generally from $_POST)
 *
 * $_SERVER values MUST be treated as immutable, as they represent application
 * state at the time of request; as such, no methods are provided to allow
 * modification of those values. The other values provide such methods, as they
 * can be restored from $_SERVER or the request body, and may need treatment
 * during the application (e.g., body parameters may be deserialized based on
 * content type).
 *
 * Additionally, this interface recognizes the utility of introspecting a
 * request to derive and match additional parameters (e.g., via URI path
 * matching, decrypting cookie values, deserializing non-form-encoded body
 * content, matching authorization headers to users, etc). These parameters
 * are stored in an "attributes" property.
 *
 * Requests are considered immutable; all methods that might change state MUST
 * be implemented such that they retain the internal state of the current
 * message and return an instance that contains the changed state.
 */
interface ServerRequestInterface extends RequestInterface
{
    /**
     * Retrieve server parameters.
     *
     * Retrieves data related to the incoming request environment,
     * typically derived from PHP's $_SERVER superglobal. The data IS NOT
     * REQUIRED to originate from $_SERVER.
     *
     * @return array
     */
    public function getServerParams();

    /**
     * Retrieve cookies.
     *
     * Retrieves cookies sent by the client to the server.
     *
     * The data MUST be compatible with the structure of the $_COOKIE
     * superglobal.
     *
     * @return array
     */
    public function getCookieParams();

    /**
     * Return an instance with the specified cookies.
     *
     * The data IS NOT REQUIRED to come from the $_COOKIE superglobal, but MUST
     * be compatible with the structure of $_COOKIE. Typically, this data will
     * be injected at instantiation.
     *
     * This method MUST NOT update the related Cookie header of the request
     * instance, nor related values in the server params.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * updated cookie values.
     *
     * @param array $cookies Array of key/value pairs representing cookies.
     * @return static
     */
    public function withCookieParams(array $cookies);

    /**
     * Retrieve query string arguments.
     *
     * Retrieves the deserialized query string arguments, if any.
     *
     * Note: the query params might not be in sync with the URI or server
     * params. If you need to ensure you are only getting the original
     * values, you may need to parse the query string from `getUri()->getQuery()`
     * or from the `QUERY_STRING` server param.
     *
     * @return array
     */
    public function getQueryParams();

    /**
     * Return an instance with the specified query string arguments.
     *
     * These values SHOULD remain immutable over the course of the incoming
     * request. They MAY be injected during instantiation, such as from PHP's
     * $_GET superglobal, or MAY be derived from some other value such as the
     * URI. In cases where the arguments are parsed from the URI, the data
     * MUST be compatible with what PHP's parse_str() would return for
     * purposes of how duplicate query parameters are handled, and how nested
     * sets are handled.
     *
     * Setting query string arguments MUST NOT change the URI stored by the
     * request, nor the values in the server params.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * updated query string arguments.
     *
     * @param array $query Array of query string arguments, typically from
     *     $_GET.
     * @return static
     */
    public function withQueryParams(array $query);

    /**
     * Retrieve normalized file upload data.
     *
     * This method returns upload metadata in a normalized tree, with each leaf
     * an instance of Psr\Http\Message\UploadedFileInterface.
     *
     * These values MAY be prepared from $_FILES or the message body during
     * instantiation, or MAY be injected via withUploadedFiles().
     *
     * @return array An array tree of UploadedFileInterface instances; an empty
     *     array MUST be returned if no data is present.
     */
    public function getUploadedFiles();

    /**
     * Create a new instance with the specified uploaded files.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * updated body parameters.
     *
     * @param array $uploadedFiles An array tree of UploadedFileInterface instances.
     * @return static
     * @throws \InvalidArgumentException if an invalid structure is provided.
     */
    public function withUploadedFiles(array $uploadedFiles);

    /**
     * Retrieve any parameters provided in the request body.
     *
     * If the request Content-Type is either application/x-www-form-urlencoded
     * or multipart/form-data, and the request method is POST, this method MUST
     * return the contents of $_POST.
     *
     * Otherwise, this method may return any results of deserializing
     * the request body content; as parsing returns structured content, the
     * potential types MUST be arrays or objects only. A null value indicates
     * the absence of body content.
     *
     * @return null|array|object The deserialized body parameters, if any.
     *     These will typically be an array or object.
     */
    public function getParsedBody();

    /**
     * Return an instance with the specified body parameters.
     *
     * These MAY be injected during instantiation.
     *
     * If the request Content-Type is either application/x-www-form-urlencoded
     * or multipart/form-data, and the request method is POST, use this method
     * ONLY to inject the contents of $_POST.
     *
     * The data IS NOT REQUIRED to come from $_POST, but MUST be the results of
     * deserializing the request body content. Deserialization/parsing returns
     * structured data, and, as such, this method ONLY accepts arrays or objects,
     * or a null value if nothing was available to parse.
     *
     * As an example, if content negotiation determines that the request data
     * is a JSON payload, this method could be used to create a request
     * instance with the deserialized parameters.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * updated body parameters.
     *
     * @param null|array|object $data The deserialized body data. This will
     *     typically be in an array or object.
     * @return static
     * @throws \InvalidArgumentException if an unsupported argument type is
     *     provided.
     */
    public function withParsedBody($data);

    /**
     * Retrieve attributes derived from the request.
     *
     * The request "attributes" may be used to allow injection of any
     * parameters derived from the request: e.g., the results of path
     * match operations; the results of decrypting cookies; the results of
     * deserializing non-form-encoded message bodies; etc. Attributes
     * will be application and request specific, and CAN be mutable.
     *
     * @return mixed[] Attributes derived from the request.
     */
    public function getAttributes();

    /**
     * Retrieve a single derived request attribute.
     *
     * Retrieves a single derived request attribute as described in
     * getAttributes(). If the attribute has not been previously set, returns
     * the default value as provided.
     *
     * This method obviates the need for a hasAttribute() method, as it allows
     * specifying a default value to return if the attribute is not found.
     *
     * @see getAttributes()
     * @param string $name The attribute name.
     * @param mixed $default Default value to return if the attribute does not exist.
     * @return mixed
     */
    public function getAttribute($name, $default = null);

    /**
     * Return an instance with the specified derived request attribute.
     *
     * This method allows setting a single derived request attribute as
     * described in getAttributes().
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * updated attribute.
     *
     * @see getAttributes()
     * @param string $name The attribute name.
     * @param mixed $value The value of the attribute.
     * @return static
     */
    public function withAttribute($name, $value);

    /**
     * Return an instance that removes the specified derived request attribute.
     *
     * This method allows removing a single derived request attribute as
     * described in getAttributes().
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that removes
     * the attribute.
     *
     * @see getAttributes()
     * @param string $name The attribute name.
     * @return static
     */
    public function withoutAttribute($name);
}
~~~

### 3.3 `Psr\Http\Message\ResponseInterface`

~~~php
<?php
namespace Psr\Http\Message;

/**
 * Representation of an outgoing, server-side response.
 *
 * Per the HTTP specification, this interface includes properties for
 * each of the following:
 *
 * - Protocol version
 * - Status code and reason phrase
 * - Headers
 * - Message body
 *
 * Responses are considered immutable; all methods that might change state MUST
 * be implemented such that they retain the internal state of the current
 * message and return an instance that contains the changed state.
 */
interface ResponseInterface extends MessageInterface
{
    /**
     * Gets the response status code.
     *
     * The status code is a 3-digit integer result code of the server's attempt
     * to understand and satisfy the request.
     *
     * @return int Status code.
     */
    public function getStatusCode();

    /**
     * Return an instance with the specified status code and, optionally, reason phrase.
     *
     * If no reason phrase is specified, implementations MAY choose to default
     * to the RFC 7231 or IANA recommended reason phrase for the response's
     * status code.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * updated status and reason phrase.
     *
     * @see http://tools.ietf.org/html/rfc7231#section-6
     * @see http://www.iana.org/assignments/http-status-codes/http-status-codes.xhtml
     * @param int $code The 3-digit integer result code to set.
     * @param string $reasonPhrase The reason phrase to use with the
     *     provided status code; if none is provided, implementations MAY
     *     use the defaults as suggested in the HTTP specification.
     * @return static
     * @throws \InvalidArgumentException For invalid status code arguments.
     */
    public function withStatus($code, $reasonPhrase = '');

    /**
     * Gets the response reason phrase associated with the status code.
     *
     * Because a reason phrase is not a required element in a response
     * status line, the reason phrase value MAY be empty. Implementations MAY
     * choose to return the default RFC 7231 recommended reason phrase (or those
     * listed in the IANA HTTP Status Code Registry) for the response's
     * status code.
     *
     * @see http://tools.ietf.org/html/rfc7231#section-6
     * @see http://www.iana.org/assignments/http-status-codes/http-status-codes.xhtml
     * @return string Reason phrase; must return an empty string if none present.
     */
    public function getReasonPhrase();
}
~~~

### 3.4 `Psr\Http\Message\StreamInterface`

~~~php
<?php
namespace Psr\Http\Message;

/**
 * Describes a data stream.
 *
 * Typically, an instance will wrap a PHP stream; this interface provides
 * a wrapper around the most common operations, including serialization of
 * the entire stream to a string.
 */
interface StreamInterface
{
    /**
     * Reads all data from the stream into a string, from the beginning to end.
     *
     * This method MUST attempt to seek to the beginning of the stream before
     * reading data and read the stream until the end is reached.
     *
     * Warning: This could attempt to load a large amount of data into memory.
     *
     * This method MUST NOT raise an exception in order to conform with PHP's
     * string casting operations.
     *
     * @see http://php.net/manual/en/language.oop5.magic.php#object.tostring
     * @return string
     */
    public function __toString();

    /**
     * Closes the stream and any underlying resources.
     *
     * @return void
     */
    public function close();

    /**
     * Separates any underlying resources from the stream.
     *
     * After the stream has been detached, the stream is in an unusable state.
     *
     * @return resource|null Underlying PHP stream, if any
     */
    public function detach();

    /**
     * Get the size of the stream if known.
     *
     * @return int|null Returns the size in bytes if known, or null if unknown.
     */
    public function getSize();

    /**
     * Returns the current position of the file read/write pointer
     *
     * @return int Position of the file pointer
     * @throws \RuntimeException on error.
     */
    public function tell();

    /**
     * Returns true if the stream is at the end of the stream.
     *
     * @return bool
     */
    public function eof();

    /**
     * Returns whether or not the stream is seekable.
     *
     * @return bool
     */
    public function isSeekable();

    /**
     * Seek to a position in the stream.
     *
     * @see http://www.php.net/manual/en/function.fseek.php
     * @param int $offset Stream offset
     * @param int $whence Specifies how the cursor position will be calculated
     *     based on the seek offset. Valid values are identical to the built-in
     *     PHP $whence values for `fseek()`.  SEEK_SET: Set position equal to
     *     offset bytes SEEK_CUR: Set position to current location plus offset
     *     SEEK_END: Set position to end-of-stream plus offset.
     * @throws \RuntimeException on failure.
     */
    public function seek($offset, $whence = SEEK_SET);

    /**
     * Seek to the beginning of the stream.
     *
     * If the stream is not seekable, this method will raise an exception;
     * otherwise, it will perform a seek(0).
     *
     * @see seek()
     * @see http://www.php.net/manual/en/function.fseek.php
     * @throws \RuntimeException on failure.
     */
    public function rewind();

    /**
     * Returns whether or not the stream is writable.
     *
     * @return bool
     */
    public function isWritable();

    /**
     * Write data to the stream.
     *
     * @param string $string The string that is to be written.
     * @return int Returns the number of bytes written to the stream.
     * @throws \RuntimeException on failure.
     */
    public function write($string);

    /**
     * Returns whether or not the stream is readable.
     *
     * @return bool
     */
    public function isReadable();

    /**
     * Read data from the stream.
     *
     * @param int $length Read up to $length bytes from the object and return
     *     them. Fewer than $length bytes may be returned if underlying stream
     *     call returns fewer bytes.
     * @return string Returns the data read from the stream, or an empty string
     *     if no bytes are available.
     * @throws \RuntimeException if an error occurs.
     */
    public function read($length);

    /**
     * Returns the remaining contents in a string
     *
     * @return string
     * @throws \RuntimeException if unable to read.
     * @throws \RuntimeException if error occurs while reading.
     */
    public function getContents();

    /**
     * Get stream metadata as an associative array or retrieve a specific key.
     *
     * The keys returned are identical to the keys returned from PHP's
     * stream_get_meta_data() function.
     *
     * @see http://php.net/manual/en/function.stream-get-meta-data.php
     * @param string $key Specific metadata to retrieve.
     * @return array|mixed|null Returns an associative array if no key is
     *     provided. Returns a specific key value if a key is provided and the
     *     value is found, or null if the key is not found.
     */
    public function getMetadata($key = null);
}
~~~

### 3.5 `Psr\Http\Message\UriInterface`

~~~php
<?php
namespace Psr\Http\Message;

/**
 * Value object representing a URI.
 *
 * This interface is meant to represent URIs according to RFC 3986 and to
 * provide methods for most common operations. Additional functionality for
 * working with URIs can be provided on top of the interface or externally.
 * Its primary use is for HTTP requests, but may also be used in other
 * contexts.
 *
 * Instances of this interface are considered immutable; all methods that
 * might change state MUST be implemented such that they retain the internal
 * state of the current instance and return an instance that contains the
 * changed state.
 *
 * Typically the Host header will also be present in the request message.
 * For server-side requests, the scheme will typically be discoverable in the
 * server parameters.
 *
 * @see http://tools.ietf.org/html/rfc3986 (the URI specification)
 */
interface UriInterface
{
    /**
     * Retrieve the scheme component of the URI.
     *
     * If no scheme is present, this method MUST return an empty string.
     *
     * The value returned MUST be normalized to lowercase, per RFC 3986
     * Section 3.1.
     *
     * The trailing ":" character is not part of the scheme and MUST NOT be
     * added.
     *
     * @see https://tools.ietf.org/html/rfc3986#section-3.1
     * @return string The URI scheme.
     */
    public function getScheme();

    /**
     * Retrieve the authority component of the URI.
     *
     * If no authority information is present, this method MUST return an empty
     * string.
     *
     * The authority syntax of the URI is:
     *
     * <pre>
     * [user-info@]host[:port]
     * </pre>
     *
     * If the port component is not set or is the standard port for the current
     * scheme, it SHOULD NOT be included.
     *
     * @see https://tools.ietf.org/html/rfc3986#section-3.2
     * @return string The URI authority, in "[user-info@]host[:port]" format.
     */
    public function getAuthority();

    /**
     * Retrieve the user information component of the URI.
     *
     * If no user information is present, this method MUST return an empty
     * string.
     *
     * If a user is present in the URI, this will return that value;
     * additionally, if the password is also present, it will be appended to the
     * user value, with a colon (":") separating the values.
     *
     * The trailing "@" character is not part of the user information and MUST
     * NOT be added.
     *
     * @return string The URI user information, in "username[:password]" format.
     */
    public function getUserInfo();

    /**
     * Retrieve the host component of the URI.
     *
     * If no host is present, this method MUST return an empty string.
     *
     * The value returned MUST be normalized to lowercase, per RFC 3986
     * Section 3.2.2.
     *
     * @see http://tools.ietf.org/html/rfc3986#section-3.2.2
     * @return string The URI host.
     */
    public function getHost();

    /**
     * Retrieve the port component of the URI.
     *
     * If a port is present, and it is non-standard for the current scheme,
     * this method MUST return it as an integer. If the port is the standard port
     * used with the current scheme, this method SHOULD return null.
     *
     * If no port is present, and no scheme is present, this method MUST return
     * a null value.
     *
     * If no port is present, but a scheme is present, this method MAY return
     * the standard port for that scheme, but SHOULD return null.
     *
     * @return null|int The URI port.
     */
    public function getPort();

    /**
     * Retrieve the path component of the URI.
     *
     * The path can either be empty or absolute (starting with a slash) or
     * rootless (not starting with a slash). Implementations MUST support all
     * three syntaxes.
     *
     * Normally, the empty path "" and absolute path "/" are considered equal as
     * defined in RFC 7230 Section 2.7.3. But this method MUST NOT automatically
     * do this normalization because in contexts with a trimmed base path, e.g.
     * the front controller, this difference becomes significant. It's the task
     * of the user to handle both "" and "/".
     *
     * The value returned MUST be percent-encoded, but MUST NOT double-encode
     * any characters. To determine what characters to encode, please refer to
     * RFC 3986, Sections 2 and 3.3.
     *
     * As an example, if the value should include a slash ("/") not intended as
     * delimiter between path segments, that value MUST be passed in encoded
     * form (e.g., "%2F") to the instance.
     *
     * @see https://tools.ietf.org/html/rfc3986#section-2
     * @see https://tools.ietf.org/html/rfc3986#section-3.3
     * @return string The URI path.
     */
    public function getPath();

    /**
     * Retrieve the query string of the URI.
     *
     * If no query string is present, this method MUST return an empty string.
     *
     * The leading "?" character is not part of the query and MUST NOT be
     * added.
     *
     * The value returned MUST be percent-encoded, but MUST NOT double-encode
     * any characters. To determine what characters to encode, please refer to
     * RFC 3986, Sections 2 and 3.4.
     *
     * As an example, if a value in a key/value pair of the query string should
     * include an ampersand ("&") not intended as a delimiter between values,
     * that value MUST be passed in encoded form (e.g., "%26") to the instance.
     *
     * @see https://tools.ietf.org/html/rfc3986#section-2
     * @see https://tools.ietf.org/html/rfc3986#section-3.4
     * @return string The URI query string.
     */
    public function getQuery();

    /**
     * Retrieve the fragment component of the URI.
     *
     * If no fragment is present, this method MUST return an empty string.
     *
     * The leading "#" character is not part of the fragment and MUST NOT be
     * added.
     *
     * The value returned MUST be percent-encoded, but MUST NOT double-encode
     * any characters. To determine what characters to encode, please refer to
     * RFC 3986, Sections 2 and 3.5.
     *
     * @see https://tools.ietf.org/html/rfc3986#section-2
     * @see https://tools.ietf.org/html/rfc3986#section-3.5
     * @return string The URI fragment.
     */
    public function getFragment();

    /**
     * Return an instance with the specified scheme.
     *
     * This method MUST retain the state of the current instance, and return
     * an instance that contains the specified scheme.
     *
     * Implementations MUST support the schemes "http" and "https" case
     * insensitively, and MAY accommodate other schemes if required.
     *
     * An empty scheme is equivalent to removing the scheme.
     *
     * @param string $scheme The scheme to use with the new instance.
     * @return static A new instance with the specified scheme.
     * @throws \InvalidArgumentException for invalid schemes.
     * @throws \InvalidArgumentException for unsupported schemes.
     */
    public function withScheme($scheme);

    /**
     * Return an instance with the specified user information.
     *
     * This method MUST retain the state of the current instance, and return
     * an instance that contains the specified user information.
     *
     * Password is optional, but the user information MUST include the
     * user; an empty string for the user is equivalent to removing user
     * information.
     *
     * @param string $user The user name to use for authority.
     * @param null|string $password The password associated with $user.
     * @return static A new instance with the specified user information.
     */
    public function withUserInfo($user, $password = null);

    /**
     * Return an instance with the specified host.
     *
     * This method MUST retain the state of the current instance, and return
     * an instance that contains the specified host.
     *
     * An empty host value is equivalent to removing the host.
     *
     * @param string $host The hostname to use with the new instance.
     * @return static A new instance with the specified host.
     * @throws \InvalidArgumentException for invalid hostnames.
     */
    public function withHost($host);

    /**
     * Return an instance with the specified port.
     *
     * This method MUST retain the state of the current instance, and return
     * an instance that contains the specified port.
     *
     * Implementations MUST raise an exception for ports outside the
     * established TCP and UDP port ranges.
     *
     * A null value provided for the port is equivalent to removing the port
     * information.
     *
     * @param null|int $port The port to use with the new instance; a null value
     *     removes the port information.
     * @return static A new instance with the specified port.
     * @throws \InvalidArgumentException for invalid ports.
     */
    public function withPort($port);

    /**
     * Return an instance with the specified path.
     *
     * This method MUST retain the state of the current instance, and return
     * an instance that contains the specified path.
     *
     * The path can either be empty or absolute (starting with a slash) or
     * rootless (not starting with a slash). Implementations MUST support all
     * three syntaxes.
     *
     * If an HTTP path is intended to be host-relative rather than path-relative
     * then it must begin with a slash ("/"). HTTP paths not starting with a slash
     * are assumed to be relative to some base path known to the application or
     * consumer.
     *
     * Users can provide both encoded and decoded path characters.
     * Implementations ensure the correct encoding as outlined in getPath().
     *
     * @param string $path The path to use with the new instance.
     * @return static A new instance with the specified path.
     * @throws \InvalidArgumentException for invalid paths.
     */
    public function withPath($path);

    /**
     * Return an instance with the specified query string.
     *
     * This method MUST retain the state of the current instance, and return
     * an instance that contains the specified query string.
     *
     * Users can provide both encoded and decoded query characters.
     * Implementations ensure the correct encoding as outlined in getQuery().
     *
     * An empty query string value is equivalent to removing the query string.
     *
     * @param string $query The query string to use with the new instance.
     * @return static A new instance with the specified query string.
     * @throws \InvalidArgumentException for invalid query strings.
     */
    public function withQuery($query);

    /**
     * Return an instance with the specified URI fragment.
     *
     * This method MUST retain the state of the current instance, and return
     * an instance that contains the specified URI fragment.
     *
     * Users can provide both encoded and decoded fragment characters.
     * Implementations ensure the correct encoding as outlined in getFragment().
     *
     * An empty fragment value is equivalent to removing the fragment.
     *
     * @param string $fragment The fragment to use with the new instance.
     * @return static A new instance with the specified fragment.
     */
    public function withFragment($fragment);

    /**
     * Return the string representation as a URI reference.
     *
     * Depending on which components of the URI are present, the resulting
     * string is either a full URI or relative reference according to RFC 3986,
     * Section 4.1. The method concatenates the various components of the URI,
     * using the appropriate delimiters:
     *
     * - If a scheme is present, it MUST be suffixed by ":".
     * - If an authority is present, it MUST be prefixed by "//".
     * - The path can be concatenated without delimiters. But there are two
     *   cases where the path has to be adjusted to make the URI reference
     *   valid as PHP does not allow to throw an exception in __toString():
     *     - If the path is rootless and an authority is present, the path MUST
     *       be prefixed by "/".
     *     - If the path is starting with more than one "/" and no authority is
     *       present, the starting slashes MUST be reduced to one.
     * - If a query is present, it MUST be prefixed by "?".
     * - If a fragment is present, it MUST be prefixed by "#".
     *
     * @see http://tools.ietf.org/html/rfc3986#section-4.1
     * @return string
     */
    public function __toString();
}
~~~

### 3.6 `Psr\Http\Message\UploadedFileInterface`

~~~php
<?php
namespace Psr\Http\Message;

/**
 * Value object representing a file uploaded through an HTTP request.
 *
 * Instances of this interface are considered immutable; all methods that
 * might change state MUST be implemented such that they retain the internal
 * state of the current instance and return an instance that contains the
 * changed state.
 */
interface UploadedFileInterface
{
    /**
     * Retrieve a stream representing the uploaded file.
     *
     * This method MUST return a StreamInterface instance, representing the
     * uploaded file. The purpose of this method is to allow utilizing native PHP
     * stream functionality to manipulate the file upload, such as
     * stream_copy_to_stream() (though the result will need to be decorated in a
     * native PHP stream wrapper to work with such functions).
     *
     * If the moveTo() method has been called previously, this method MUST raise
     * an exception.
     *
     * @return StreamInterface Stream representation of the uploaded file.
     * @throws \RuntimeException in cases when no stream is available.
     * @throws \RuntimeException in cases when no stream can be created.
     */
    public function getStream();

    /**
     * Move the uploaded file to a new location.
     *
     * Use this method as an alternative to move_uploaded_file(). This method is
     * guaranteed to work in both SAPI and non-SAPI environments.
     * Implementations must determine which environment they are in, and use the
     * appropriate method (move_uploaded_file(), rename(), or a stream
     * operation) to perform the operation.
     *
     * $targetPath may be an absolute path, or a relative path. If it is a
     * relative path, resolution should be the same as used by PHP's rename()
     * function.
     *
     * The original file or stream MUST be removed on completion.
     *
     * If this method is called more than once, any subsequent calls MUST raise
     * an exception.
     *
     * When used in an SAPI environment where $_FILES is populated, when writing
     * files via moveTo(), is_uploaded_file() and move_uploaded_file() SHOULD be
     * used to ensure permissions and upload status are verified correctly.
     *
     * If you wish to move to a stream, use getStream(), as SAPI operations
     * cannot guarantee writing to stream destinations.
     *
     * @see http://php.net/is_uploaded_file
     * @see http://php.net/move_uploaded_file
     * @param string $targetPath Path to which to move the uploaded file.
     * @throws \InvalidArgumentException if the $targetPath specified is invalid.
     * @throws \RuntimeException on any error during the move operation.
     * @throws \RuntimeException on the second or subsequent call to the method.
     */
    public function moveTo($targetPath);

    /**
     * Retrieve the file size.
     *
     * Implementations SHOULD return the value stored in the "size" key of
     * the file in the $_FILES array if available, as PHP calculates this based
     * on the actual size transmitted.
     *
     * @return int|null The file size in bytes or null if unknown.
     */
    public function getSize();

    /**
     * Retrieve the error associated with the uploaded file.
     *
     * The return value MUST be one of PHP's UPLOAD_ERR_XXX constants.
     *
     * If the file was uploaded successfully, this method MUST return
     * UPLOAD_ERR_OK.
     *
     * Implementations SHOULD return the value stored in the "error" key of
     * the file in the $_FILES array.
     *
     * @see http://php.net/manual/en/features.file-upload.errors.php
     * @return int One of PHP's UPLOAD_ERR_XXX constants.
     */
    public function getError();

    /**
     * Retrieve the filename sent by the client.
     *
     * Do not trust the value returned by this method. A client could send
     * a malicious filename with the intention to corrupt or hack your
     * application.
     *
     * Implementations SHOULD return the value stored in the "name" key of
     * the file in the $_FILES array.
     *
     * @return string|null The filename sent by the client or null if none
     *     was provided.
     */
    public function getClientFilename();

    /**
     * Retrieve the media type sent by the client.
     *
     * Do not trust the value returned by this method. A client could send
     * a malicious media type with the intention to corrupt or hack your
     * application.
     *
     * Implementations SHOULD return the value stored in the "type" key of
     * the file in the $_FILES array.
     *
     * @return string|null The media type sent by the client or null if none
     *     was provided.
     */
    public function getClientMediaType();
}
~~~


<h1 id="HTTP Message Meta Document">HTTP Message Meta Document</h1>

## 1. Summary

The purpose of this proposal is to provide a set of common interfaces for HTTP
messages as described in [RFC 7230](http://tools.ietf.org/html/rfc7230) and
[RFC 7231](http://tools.ietf.org/html/rfc7231), and URIs as described in
[RFC 3986](http://tools.ietf.org/html/rfc3986) (in the context of HTTP messages).

- RFC 7230: http://www.ietf.org/rfc/rfc7230.txt
- RFC 7231: http://www.ietf.org/rfc/rfc7231.txt
- RFC 3986: http://www.ietf.org/rfc/rfc3986.txt

All HTTP messages consist of the HTTP protocol version being used, headers, and
a message body. A _Request_ builds on the message to include the HTTP method
used to make the request, and the URI to which the request is made. A
_Response_ includes the HTTP status code and reason phrase.

In PHP, HTTP messages are used in two contexts:

- To send an HTTP request, via the `ext/curl` extension, PHP's native stream
  layer, etc., and process the received HTTP response. In other words, HTTP
  messages are used when using PHP as an _HTTP client_.
- To process an incoming HTTP request to the server, and return an HTTP response
  to the client making the request. PHP can use HTTP messages when used as a
  _server-side application_ to fulfill HTTP requests.

This proposal presents an API for fully describing all parts of the various
HTTP messages within PHP.

## 2. HTTP Messages in PHP

PHP does not have built-in support for HTTP messages.

### Client-side HTTP support

PHP supports sending HTTP requests via several mechanisms:

- [PHP streams](http://php.net/streams)
- The [cURL extension](http://php.net/curl)
- [ext/http](http://php.net/http) (v2 also attempts to address server-side support)

PHP streams are the most convenient and ubiquitous way to send HTTP requests,
but pose a number of limitations with regards to properly configuring SSL
support, and provide a cumbersome interface around setting things such as
headers. cURL provides a complete and expanded feature-set, but, as it is not a
default extension, is often not present. The http extension suffers from the
same problem as cURL, as well as the fact that it has traditionally had far
fewer examples of usage.

Most modern HTTP client libraries tend to abstract the implementation, to
ensure they can work on whatever environment they are executed on, and across
any of the above layers.

### Server-side HTTP Support

PHP uses Server APIs (SAPI) to interpret incoming HTTP requests, marshal input,
and pass off handling to scripts. The original SAPI design mirrored [Common
Gateway Interface](http://www.w3.org/CGI/), which would marshal request data
and push it into environment variables before passing delegation to a script;
the script would then pull from the environment variables in order to process
the request and return a response.

PHP's SAPI design abstracts common input sources such as cookies, query string
arguments, and url-encoded POST content via superglobals (`$_COOKIE`, `$_GET`,
and `$_POST`, respectively), providing a layer of convenience for web developers.

On the response side of the equation, PHP was originally developed as a
templating language, and allows intermixing HTML and PHP; any HTML portions of
a file are immediately flushed to the output buffer. Modern applications and
frameworks eschew this practice, as it can lead to issues with
regards to emitting a status line and/or response headers; they tend to
aggregate all headers and content, and emit them at once when all other
application processing is complete. Special care needs to be paid to ensure
that error reporting and other actions that send content to the output buffer
do not flush the output buffer.

## 3. Why Bother?

HTTP messages are used in a wide number of PHP projects -- both clients and
servers. In each case, we observe one or more of the following patterns or
situations:

1. Projects use PHP's superglobals directly.
2. Projects will create implementations from scratch.
3. Projects may require a specific HTTP client/server library that provides
   HTTP message implementations.
4. Projects may create adapters for common HTTP message implementations.

As examples:

1. Just about any application that began development before the rise of
   frameworks, which includes a number of very popular CMS, forum, and shopping
   cart systems, have historically used superglobals.
2. Frameworks such as Symfony and Zend Framework each define HTTP components
   that form the basis of their MVC layers; even small, single-purpose
   libraries such as oauth2-server-php provide and require their own HTTP
   request/response implementations. Guzzle, Buzz, and other HTTP client
   implementations each create their own HTTP message implementations as well.
3. Projects such as Silex, Stack, and Drupal 8 have hard dependencies on
   Symfony's HTTP kernel. Any SDK built on Guzzle has a hard requirement on
   Guzzle's HTTP message implementations.
4. Projects such as Geocoder create redundant [adapters for common
   libraries](https://github.com/geocoder-php/Geocoder/tree/6a729c6869f55ad55ae641c74ac9ce7731635e6e/src/Geocoder/HttpAdapter).

Direct usage of superglobals has a number of concerns. First, these are
mutable, which makes it possible for libraries and code to alter the values,
and thus alter state for the application. Additionally, superglobals make unit
and integration testing difficult and brittle, leading to code quality
degradation.

In the current ecosystem of frameworks that implement HTTP message abstractions,
the net result is that projects are not capable of interoperability or
cross-pollination. In order to consume code targeting one framework from
another, the first order of business is building a bridge layer between the
HTTP message implementations. On the client-side, if a particular library does
not have an adapter you can utilize, you need to bridge the request/response
pairs if you wish to use an adapter from another library.

Finally, when it comes to server-side responses, PHP gets in its own way: any
content emitted before a call to `header()` will result in that call becoming a
no-op; depending on error reporting settings, this can often mean headers
and/or response status are not correctly sent. One way to work around this is
to use PHP's output buffering features, but nesting of output buffers can
become problematic and difficult to debug. Frameworks and applications thus
tend to create response abstractions for aggregating headers and content that
can be emitted at once - and these abstractions are often incompatible.

Thus, the goal of this proposal is to abstract both client- and server-side
request and response interfaces in order to promote interoperability between
projects. If projects implement these interfaces, a reasonable level of
compatibility may be assumed when adopting code from different libraries.

It should be noted that the goal of this proposal is not to obsolete the
current interfaces utilized by existing PHP libraries. This proposal is aimed
at interoperability between PHP packages for the purpose of describing HTTP
messages.

## 4. Scope

### 4.1 Goals

* Provide the interfaces needed for describing HTTP messages.
* Focus on practical applications and usability.
* Define the interfaces to model all elements of the HTTP message and URI
  specifications.
* Ensure that the API does not impose arbitrary limits on HTTP messages. For
  example, some HTTP message bodies can be too large to store in memory, so we
  must account for this.
* Provide useful abstractions both for handling incoming requests for
  server-side applications and for sending outgoing requests in HTTP clients.

### 4.2 Non-Goals

* This proposal does not expect all HTTP client libraries or server-side
  frameworks to change their interfaces to conform. It is strictly meant for
  interoperability.
* While everyone's perception of what is and is not an implementation detail
  varies, this proposal should not impose implementation details. As
  RFCs 7230, 7231, and 3986 do not force any particular implementation,
  there will be a certain amount of invention needed to describe HTTP message
  interfaces in PHP.

## 5. Design Decisions

### Message design

The `MessageInterface` provides accessors for the elements common to all HTTP
messages, whether they are for requests or responses. These elements include:

- HTTP protocol version (e.g., "1.0", "1.1")
- HTTP headers
- HTTP message body

More specific interfaces are used to describe requests and responses, and more
specifically the context of each (client- vs. server-side). These divisions are
partly inspired by existing PHP usage, but also by other languages such as
Ruby's [Rack](https://rack.github.io),
Python's [WSGI](https://www.python.org/dev/peps/pep-0333/),
Go's [http package](http://golang.org/pkg/net/http/),
Node's [http module](http://nodejs.org/api/http.html), etc.

### Why are there header methods on messages rather than in a header bag?

The message itself is a container for the headers (as well as the other message
properties). How these are represented internally is an implementation detail,
but uniform access to headers is a responsibility of the message.

### Why are URIs represented as objects?

URIs are values, with identity defined by the value, and thus should be modeled
as value objects.

Additionally, URIs contain a variety of segments which may be accessed many
times in a given request -- and which would require parsing the URI in order to
determine (e.g., via `parse_url()`). Modeling URIs as value objects allows
parsing once only, and simplifies access to individual segments. It also
provides convenience in client applications by allowing users to create new
instances of a base URI instance with only the segments that change (e.g.,
updating the path only).

### Why does the request interface have methods for dealing with the request-target AND compose a URI?

RFC 7230 details the request line as containing a "request-target". Of the four
forms of request-target, only one is a URI compliant with RFC 3986; the most
common form used is origin-form, which represents the URI without the
scheme or authority information. Moreover, since all forms are valid for
purposes of requests, the proposal must accommodate each.

`RequestInterface` thus has methods relating to the request-target. By default,
it will use the composed URI to present an origin-form request-target, and, in
the absence of a URI instance, return the string "/".  Another method,
`withRequestTarget()`, allows specifying an instance with a specific
request-target, allowing users to create requests that use one of the other
valid request-target forms.

The URI is kept as a discrete member of the request for a variety of reasons.
For both clients and servers, knowledge of the absolute URI is typically
required. In the case of clients, the URI, and specifically the scheme and
authority details, is needed in order to make the actual TCP connection. For
server-side applications, the full URI is often required in order to validate
the request or to route to an appropriate handler.

### Why value objects?

The proposal models messages and URIs as [value objects](http://en.wikipedia.org/wiki/Value_object).

Messages are values where the identity is the aggregate of all parts of the
message; a change to any aspect of the message is essentially a new message.
This is the very definition of a value object. The practice by which changes
result in a new instance is termed [immutability](http://en.wikipedia.org/wiki/Immutable_object),
and is a feature designed to ensure the integrity of a given value.

The proposal also recognizes that most clients and server-side
applications will need to be able to easily update message aspects, and, as
such, provides interface methods that will create new message instances with
the updates. These are generally prefixed with the verbiage `with` or
`without`.

Value objects provides several benefits when modeling HTTP messages:

- Changes in URI state cannot alter the request composing the URI instance.
- Changes in headers cannot alter the message composing them.

In essence, modeling HTTP messages as value objects ensures the integrity of
the message state, and prevents the need for bi-directional dependencies, which
can often go out-of-sync or lead to debugging or performance issues.

For HTTP clients, they allow consumers to build a base request with data such
as the base URI and required headers, without needing to build a brand new
request or reset request state for each message the client sends:

~~~php
$uri = new Uri('http://api.example.com');
$baseRequest = new Request($uri, null, [
    'Authorization' => 'Bearer ' . $token,
    'Accept'        => 'application/json',
]);;

$request = $baseRequest->withUri($uri->withPath('/user'))->withMethod('GET');
$response = $client->send($request);

// get user id from $response

$body = new StringStream(json_encode(['tasks' => [
    'Code',
    'Coffee',
]]));;
$request = $baseRequest
    ->withUri($uri->withPath('/tasks/user/' . $userId))
    ->withMethod('POST')
    ->withHeader('Content-Type', 'application/json')
    ->withBody($body);
$response = $client->send($request)

// No need to overwrite headers or body!
$request = $baseRequest->withUri($uri->withPath('/tasks'))->withMethod('GET');
$response = $client->send($request);
~~~

On the server-side, developers will need to:

- Deserialize the request message body.
- Decrypt HTTP cookies.
- Write to the response.

These operations can be accomplished with value objects as well, with a number
of benefits:

- The original request state can be stored for retrieval by any consumer.
- A default response state can be created with default headers and/or message body.

Most popular PHP frameworks have fully mutable HTTP messages today. The main
changes necessary in consuming true value objects are:

- Instead of calling setter methods or setting public properties, mutator
  methods will be called, and the result assigned.
- Developers must notify the application on a change in state.

As an example, in Zend Framework 2, instead of the following:

~~~php
function (MvcEvent $e)
{
    $response = $e->getResponse();
    $response->setHeaderLine('x-foo', 'bar');
}
~~~

one would now write:

~~~php
function (MvcEvent $e)
{
    $response = $e->getResponse();
    $e->setResponse(
        $response->withHeader('x-foo', 'bar')
    );
}
~~~

The above combines assignment and notification in a single call.

This practice has a side benefit of making explicit any changes to application
state being made.

### New instances vs returning $this

One observation made on the various `with*()` methods is that they can likely
safely `return $this;` if the argument presented will not result in a change in
the value. One rationale for doing so is performance (as this will not result in
a cloning operation).

The various interfaces have been written with verbiage indicating that
immutability MUST be preserved, but only indicate that "an instance" must be
returned containing the new state. Since instances that represent the same value
are considered equal, returning `$this` is functionally equivalent, and thus
allowed.

### Using streams instead of X

`MessageInterface` uses a body value that must implement `StreamInterface`. This
design decision was made so that developers can send and receive (and/or receive
and send) HTTP messages that contain more data than can practically be stored in
memory while still allowing the convenience of interacting with message bodies
as a string. While PHP provides a stream abstraction by way of stream wrappers,
stream resources can be cumbersome to work with: stream resources can only be
cast to a string using `stream_get_contents()` or manually reading the remainder
of a string. Adding custom behavior to a stream as it is consumed or populated
requires registering a stream filter; however, stream filters can only be added
to a stream after the filter is registered with PHP (i.e., there is no stream
filter autoloading mechanism).

The use of a well- defined stream interface allows for the potential of
flexible stream decorators that can be added to a request or response
pre-flight to enable things like encryption, compression, ensuring that the
number of bytes downloaded reflects the number of bytes reported in the
`Content-Length` of a response, etc. Decorating streams is a well-established
[pattern in the Java](http://docs.oracle.com/javase/7/docs/api/java/io/package-tree.html)
and [Node](http://nodejs.org/api/stream.html#stream_class_stream_transform_1)
communities that allows for very flexible streams.

The majority of the `StreamInterface` API is based on
[Python's io module](http://docs.python.org/3.1/library/io.html), which provides
a practical and consumable API. Instead of implementing stream
capabilities using something like a `WritableStreamInterface` and
`ReadableStreamInterface`, the capabilities of a stream are provided by methods
like `isReadable()`, `isWritable()`, etc. This approach is used by Python,
[C#, C++](http://msdn.microsoft.com/en-us/library/system.io.stream.aspx),
[Ruby](http://www.ruby-doc.org/core-2.0.0/IO.html),
[Node](http://nodejs.org/api/stream.html), and likely others.

#### What if I just want to return a file?

In some cases, you may want to return a file from the filesystem. The typical
way to do this in PHP is one of the following:

~~~php
readfile($filename);

stream_copy_to_stream(fopen($filename, 'r'), fopen('php://output', 'w'));
~~~

Note that the above omits sending appropriate `Content-Type` and
`Content-Length` headers; the developer would need to emit these prior to
calling the above code.

The equivalent using HTTP messages would be to use a `StreamInterface`
implementation that accepts a filename and/or stream resource, and to provide
this to the response instance. A complete example, including setting appropriate
headers:

~~~php
// where Stream is a concrete StreamInterface:
$stream   = new Stream($filename);
$finfo    = new finfo(FILEINFO_MIME);
$response = $response
    ->withHeader('Content-Type', $finfo->file($filename))
    ->withHeader('Content-Length', (string) filesize($filename))
    ->withBody($stream);
~~~

Emitting this response will send the file to the client.

#### What if I want to directly emit output?

Directly emitting output (e.g. via `echo`, `printf`, or writing to the
`php://output` stream) is generally only advisable as a performance optimization
or when emitting large data sets. If it needs to be done and you still wish
to work in an HTTP message paradigm, one approach would be to use a
callback-based `StreamInterface` implementation, per [this
example](https://github.com/phly/psr7examples#direct-output). Wrap any code
emitting output directly in a callback, pass that to an appropriate
`StreamInterface` implementation, and provide it to the message body:

~~~php
$output = new CallbackStream(function () use ($request) {
    printf("The requested URI was: %s<br>\n", $request->getUri());
    return '';
});
return (new Response())
    ->withHeader('Content-Type', 'text/html')
    ->withBody($output);
~~~

#### What if I want to use an iterator for content?

Ruby's Rack implementation uses an iterator-based approach for server-side
response message bodies. This can be emulated using an HTTP message paradigm via
an iterator-backed `StreamInterface` approach, as [detailed in the
psr7examples repository](https://github.com/phly/psr7examples#iterators-and-generators).

### Why are streams mutable?

The `StreamInterface` API includes methods such as `write()` which can
change the message content -- which directly contradicts having immutable
messages.

The problem that arises is due to the fact that the interface is intended to
wrap a PHP stream or similar. A write operation therefore will proxy to writing
to the stream. Even if we made `StreamInterface` immutable, once the stream
has been updated, any instance that wraps that stream will also be updated --
making immutability impossible to enforce.

Our recommendation is that implementations use read-only streams for
server-side requests and client-side responses.

### Rationale for ServerRequestInterface

The `RequestInterface` and `ResponseInterface` have essentially 1:1
correlations with the request and response messages described in
[RFC 7230](http://www.ietf.org/rfc/rfc7230.txt). They provide interfaces for
implementing value objects that correspond to the specific HTTP message types
they model.

For server-side applications there are other considerations for
incoming requests:

- Access to server parameters (potentially derived from the request, but also
  potentially the result of server configuration, and generally represented
  via the `$_SERVER` superglobal; these are part of the PHP Server API (SAPI)).
- Access to the query string arguments (usually encapsulated in PHP via the
  `$_GET` superglobal).
- Access to the parsed body (i.e., data deserialized from the incoming request
  body; in PHP, this is typically the result of POST requests using
  `application/x-www-form-urlencoded` content types, and encapsulated in the
  `$_POST` superglobal, but for non-POST, non-form-encoded data, could be
  an array or an object).
- Access to uploaded files (encapsulated in PHP via the `$_FILES` superglobal).
- Access to cookie values (encapsulated in PHP via the `$_COOKIE` superglobal).
- Access to attributes derived from the request (usually, but not limited to,
  those matched against the URL path).

Uniform access to these parameters increases the viability of interoperability
between frameworks and libraries, as they can now assume that if a request
implements `ServerRequestInterface`, they can get at these values. It also
solves problems within the PHP language itself:

- Until 5.6.0, `php://input` was read-once; as such, instantiating multiple
  request instances from multiple frameworks/libraries could lead to
  inconsistent state, as the first to access `php://input` would be the only
  one to receive the data.
- Unit testing against superglobals (e.g., `$_GET`, `$_FILES`, etc.) is
  difficult and typically brittle. Encapsulating them inside the
  `ServerRequestInterface` implementation eases testing considerations.

### Why "parsed body" in the ServerRequestInterface?

Arguments were made to use the terminology "BodyParams", and require the value
to be an array, with the following rationale:

- Consistency with other server-side parameter access.
- `$_POST` is an array, and the 80% use case would target that superglobal.
- A single type makes for a strong contract, simplifying usage.

The main argument is that if the body parameters are an array, developers have
predictable access to values:

~~~php
$foo = isset($request->getBodyParams()['foo'])
    ? $request->getBodyParams()['foo']
    : null;
~~~

The argument for using "parsed body" was made by examining the domain. A message
body can contain literally anything. While traditional web applications use
forms and submit data using POST, this is a use case that is quickly being
challenged in current web development trends, which are often API-centric, and
thus use alternate request methods (notably PUT and PATCH), as well as
non-form-encoded content (generally JSON or XML) that _can_ be coerced to arrays
in many cases, but in many cases also _cannot_ or _should not_.

If forcing the property representing the parsed body to be only an array,
developers then need a shared convention about where to put the results of
parsing the body. These might include:

- A special key under the body parameters, such as `__parsed__`.
- A specially named attribute, such as `__body__`.

The end result is that a developer now has to look in multiple locations:

~~~php
$data = $request->getBodyParams();
if (isset($data['__parsed__']) && is_object($data['__parsed__'])) {
    $data = $data['__parsed__'];
}

// or:
$data = $request->getBodyParams();
if ($request->hasAttribute('__body__')) {
    $data = $request->getAttribute('__body__');
}
~~~

The solution presented is to use the terminology "ParsedBody", which implies
that the values are the results of parsing the message body. This also means
that the return value _will_ be ambiguous; however, because this is an attribute
of the domain, this is also expected. As such, usage will become:

~~~php
$data = $request->getParsedBody();
if (! $data instanceof \stdClass) {
    // raise an exception!
}
// otherwise, we have what we expected
~~~

This approach removes the limitations of forcing an array, at the expense of
ambiguity of return value. Considering that the other suggested solutions —
pushing the parsed data into a special body parameter key or into an attribute —
also suffer from ambiguity, the proposed solution is simpler as it does not
require additions to the interface specification. Ultimately, the ambiguity
enables the flexibility required when representing the results of parsing the
body.

### Why is no functionality included for retrieving the "base path"?

Many frameworks provide the ability to get the "base path," usually considered
the path up to and including the front controller. As an example, if the
application is served at `http://example.com/b2b/index.php`, and the current URI
used to request it is `http://example.com/b2b/index.php/customer/register`, the
functionality to retrieve the base path would return `/b2b/index.php`. This value
can then be used by routers to strip that path segment prior to attempting a
match.

This value is often also then used for URI generation within applications;
parameters will be passed to the router, which will generate the path, and
prefix it with the base path in order to return a fully-qualified URI. Other
tools — typically view helpers, template filters, or template functions — are
used to resolve a path relative to the base path in order to generate a URI for
linking to resources such as static assets.

On examination of several different implementations, we noticed the following:

- The logic for determining the base path varies widely between implementations.
  As an example, compare the [logic in ZF2](https://github.com/zendframework/zf2/blob/release-2.3.7/library/Zend/Http/PhpEnvironment/Request.php#L477-L575)
  to the [logic in Symfony 2](https://github.com/symfony/symfony/blob/2.7/src/Symfony/Component/HttpFoundation/Request.php#L1858-L1877).
- Most implementations appear to allow manual injection of a base path to the
  router and/or any facilities used for URI generation.
- The primary use cases — routing and URI generation — typically are the only
  consumers of the functionality; developers usually do not need to be aware
  of the base path concept as other objects take care of that detail for them.
  As examples:
  - A router will strip off the base path for you during routing; you do not
    need to pass the modified path to the router.
  - View helpers, template filters, etc. typically are injected with a base path
    prior to invocation. Sometimes this is manually done, though more often it
    is the result of framework wiring.
- All sources necessary for calculating the base path *are already in the
  `RequestInterface` instance*, via server parameters and the URI instance.

Our stance is that base path detection is framework and/or application
specific, and the results of detection can be easily injected into objects that
need it, and/or calculated as needed using utility functions and/or classes from
the `RequestInterface` instance itself.

### Why does getUploadedFiles() return objects instead of arrays?

`getUploadedFiles()` returns a tree of `Psr\Http\Message\UploadedFileInterface`
instances. This is done primarily to simplify specification: instead of
requiring paragraphs of implementation specification for an array, we specify an
interface.

Additionally, the data in an `UploadedFileInterface` is normalized to work in
both SAPI and non-SAPI environments. This allows the creation of processes to parse
the message body manually and assign contents to streams without first writing
to the filesystem, while still allowing proper handling of file uploads in SAPI
environments.

### What about "special" header values?

A number of header values contain unique representation requirements which can
pose problems both for consumption as well as generation; in particular, cookies
and the `Accept` header.

This proposal does not provide any special treatment of any header types. The
base `MessageInterface` provides methods for header retrieval and setting, and
all header values are, in the end, string values.

Developers are encouraged to write commodity libraries for interacting with
these header values, either for the purposes of parsing or generation. Users may
then consume these libraries when needing to interact with those values.
Examples of this practice already exist in libraries such as
[willdurand/Negotiation](https://github.com/willdurand/Negotiation) and
[Aura.Accept](https://github.com/auraphp/Aura.Accept). So long as the object
has functionality for casting the value to a string, these objects can be
used to populate the headers of an HTTP message.

## 6. People

### 6.1 Editor(s)

* Matthew Weier O'Phinney

### 6.2 Sponsors

* Paul M. Jones
* Beau Simensen (coordinator)

### 6.3 Contributors

* Michael Dowling
* Larry Garfield
* Evert Pot
* Tobias Schultze
* Bernhard Schussek
* Anton Serdyuk
* Phil Sturgeon
* Chris Wilkinson
