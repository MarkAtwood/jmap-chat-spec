---
title: "Updates to JMAP: Localized Display Text, Text Handling, URIs, and Problem Details"
abbrev: JMAP Updates
docname: draft-atwood-jmap-updates-00
category: std
submissiontype: IETF
updates: 8620, 9425

ipr: trust200902

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
  -
    fullname: Mark Atwood
    organization: Review Commit, LLC
    city: Seattle
    region: WA
    country: US
    email: mark@reviewcommit.com

normative:
  RFC2119:
  RFC3629:
  RFC3986:
  RFC4647:
  RFC5646:
  RFC8174:
  RFC8620:
  RFC8621:
  RFC8887:
  RFC8984:
  RFC9110:
  RFC9425:
  RFC9457:
  RFC9610:
  RFC9670:
  UAX29:
    title: "Unicode Standard Annex #29: Unicode Text Segmentation"
    author:
      - org: The Unicode Consortium
    target: https://www.unicode.org/reports/tr29/

informative:
  MNU:
    title: "Modern Network Unicode"
    author:
      - fullname: Carsten Bormann
    seriesinfo:
      Internet-Draft: draft-bormann-dispatch-modern-network-unicode-09
    date: 2026-09-02
  RFC4790:
  RFC5051:
  RFC5198:
  RFC6570:
  RFC7807:
  RFC7493:
  RFC3987:
  RFC8264:
  RFC8266:
  RFC9553:
  JMAP-CALENDARS:
    title: "JMAP for Calendars"
    author:
      - fullname: Neil Jenkins
      - fullname: Michael Douglass
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-calendars-33
    date: 2026-10-14
  UAX9:
    title: "Unicode Standard Annex #9: Unicode Bidirectional Algorithm"
    author:
      - org: The Unicode Consortium
    target: https://www.unicode.org/reports/tr9/
  UTS39:
    title: "Unicode Technical Standard #39: Unicode Security Mechanisms"
    author:
      - org: The Unicode Consortium
    target: https://www.unicode.org/reports/tr39/

--- abstract

JMAP (RFC 8620) carries all data as UTF-8 I-JSON, lets a client name a
collation for sorting, and lets a server localize user-visible strings
from the request's Accept-Language header. It has no way to store or
return display text, such as the name of a mailbox, address book, or
quota, in more than one language, and no way to say which language a
stored string is in. This document defines the
`urn:ietf:params:jmap:i18n` capability, which adds `locale` and
`localizations` properties, modelled on JSCalendar (RFC 8984), to the
display text of existing JMAP data types, and a `language` argument
with which a client asks for one language. It also updates RFC 8620
with guidance for all JMAP specifications on text profiles, Unicode
normalization, truncation, and the use of display text, and asks that
URI-valued properties follow RFC 3986 and that every JMAP HTTP
resource report errors as RFC 9457 problem details.

--- middle

# Introduction {#introduction}

## What JMAP Already Provides {#existing}

JMAP {{RFC8620}} already settles several internationalization
questions, and this document does not change them:

- **Encoding.** Everything exchanged, except binary upload and
  download, is I-JSON {{RFC7493}} encoded in UTF-8 {{RFC3629}}
  ({{RFC8620}} Section 1.5). A JMAP specification does not need to say
  again that a string is valid UTF-8.
- **Server-chosen strings.** When a server returns a custom string to
  be shown to the user, it chooses the localization from the request's
  Accept-Language header ({{RFC9110}} Section 12.5.4) and labels the
  response with Content-Language ({{RFC8620}} Section 3.8). The Quota `description` of {{RFC9425}}
  applies the same rule to a data property.
- **Collation.** A `/query` Comparator names a collation from the
  registry of {{RFC4790}}, and the server lists the ones it supports in
  the `collationAlgorithms` property of the `urn:ietf:params:jmap:core`
  capability ({{RFC8620}} Sections 2 and 5.5). The default collation
  is Unicode-aware, for example "i;unicode-casemap" {{RFC5051}}.
- **Content.** Email content carries its own charset and
  Content-Language, which {{RFC8621}} exposes on each body part.
  JSCalendar {{RFC8984}} and JSContact {{RFC9553}} objects carry a
  language and per-language localizations of their own.

## What Is Missing {#gaps}

- **Display text has one value and no language.** The name of a
  Mailbox, an AddressBook, a Calendar, or a Principal is a single
  string. A shared address book used by French and German speakers can
  have only one name, and nothing says which language that name is in.
- **Content-Language describes a whole response.** One HTTP response
  can carry objects in several languages, and a response sent over a
  WebSocket {{RFC8887}} has no header at all.
- **Text handling is unstated.** JMAP specifications do not say
  whether servers normalize text, where a server may cut a string it
  shortens, or whether display text may carry meaning.
- **URIs and errors are uneven.** {{RFC8620}} asks for problem
  details, citing {{RFC7807}}, which {{RFC9457}} obsoletes, on errors
  from the API, upload, and download resources, but not from the
  Session resource, the event source, or resources that extensions
  add. Properties whose values are URIs are defined one by one, with
  no common rule for their syntax or for non-ASCII characters.

## This Document {#this-document}

{{capability}} defines the `urn:ietf:params:jmap:i18n` capability.
{{localization}} defines the `locale` and `localizations` properties
and the `language` argument, and {{applicability}} lists the
properties of existing data types that they apply to. For clients
that opt in, this updates how {{RFC9425}} selects the Quota
`description` ({{quotas}}). Future specifications can adopt them by
listing their own localizable properties ({{new-specs}}).

{{text-handling}} and {{uris-problems}} update {{RFC8620}} with
guidance on text, URIs, and errors that applies to every JMAP
specification, whether or not the capability is in use. They add no
properties and no arguments.

## Conventions Used in This Document

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP 14
{{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals,
as shown here.

Type signatures, examples, and property descriptions follow
{{RFC8620}} Section 1.

Display text:
: A string intended to be shown to a person and not interpreted by
  software, such as a name, a description, or a label.

Localizable property:
: A property of a data type whose value is display text and that this
  document, or a specification that adopts it, lists as localizable.

Language tag:
: A language tag as defined in {{RFC5646}}.

# Capability {#capability}

A server that supports this document advertises the
`urn:ietf:params:jmap:i18n` capability in the `capabilities` property
of the JMAP Session resource ({{RFC8620}} Section 2). Its value is an
empty object.

The same capability appears in the `accountCapabilities` of each
account for which the server supports localizations. Its value there is
an object with one property:

`preferredLanguages`: `String[]|null`
: The languages the account's owner prefers, as language tags
  ({{RFC5646}}), most preferred first, or null when the server has no
  preference recorded. See {{account-preference}}.

A client that wants the properties and arguments of this document
includes `urn:ietf:params:jmap:i18n` in the `using` property of the
Request. When the capability is not in `using`, the server behaves
as if this document did not exist: it neither returns nor accepts
`locale` and `localizations`, and it rejects the `language` argument
with `invalidArguments`, as it would any unknown argument ({{RFC8620}}
Section 3.9). The same applies to an account whose
`accountCapabilities` does not include `urn:ietf:params:jmap:i18n`,
even when the capability is in `using`: a `/set` that names `locale` or
`localizations` is rejected with an `invalidProperties` SetError, and a
`language` argument with `invalidArguments`.

Example Session excerpt:

~~~json
{
  "capabilities": {
    "urn:ietf:params:jmap:core": {
      "collationAlgorithms": [ "i;unicode-casemap" ]
    },
    "urn:ietf:params:jmap:mail": {},
    "urn:ietf:params:jmap:i18n": {}
  },
  "accounts": {
    "a1": {
      "name": "jane@example.com",
      "isPersonal": true,
      "isReadOnly": false,
      "accountCapabilities": {
        "urn:ietf:params:jmap:mail": {},
        "urn:ietf:params:jmap:i18n": {
          "preferredLanguages": [ "fr-CH", "de" ]
        }
      }
    }
  }
}
~~~

## Account Language Preference {#account-preference}

Some text is produced when no request is in progress: the text of a
push notification sent through a push service, a message the server
sends on the user's behalf, or a server-set property computed in the
background. Accept-Language and the `language` argument are not
available then.

A server MAY use `preferredLanguages` to choose the language of such
text. How the preference is set is outside the scope of this document;
a server might take it from an account setting, from a directory, or
from the Accept-Language of the owner's recent requests. A server
MUST NOT use `preferredLanguages` to change the result of `/get` or
`/query`, for the reason given in {{language-argument}}; a client that
wants a language for data passes `language`.

A client MAY read `preferredLanguages` as a default for `language` and
for its own user interface.

# Localized Display Text {#localization}

## The locale and localizations Properties {#properties}

A data type that has at least one localizable property gains two
properties when `urn:ietf:params:jmap:i18n` is in `using`:

`locale`: `String|null` (default: null)
: The language tag of the values of the object's localizable
  properties. Null means the language is unknown. This has the
  meaning of `locale` in {{RFC8984}} Section 4.2.8.

`localizations`: `String[PatchObject]|null` (default: null)
: A map from a language tag to a PatchObject ({{RFC8984}} Section
  1.4.9) that, applied to the object, gives its localizable properties
  in that language. This has the meaning of `localizations` in
  {{RFC8984}} Section 4.6.1, with the restrictions below.

The restrictions on `localizations` are:

- Each key of a PatchObject is the name of a localizable property of
  the data type, and its value is of that property's type.
- A PatchObject need not cover every localizable property; a property
  it omits keeps the base value in that language.
- A null value is allowed only for a property whose type allows null,
  and means that the property has no value in that language; for
  example, `{"description": null}` says there is no description in
  that language.
- Language tags are case-insensitive ({{RFC5646}} Section 2.1.1). No
  two keys of `localizations` may differ only in case, and no key may
  equal `locale`, ignoring case. Servers SHOULD store tags in the
  canonical case that {{RFC5646}} Section 2.1.1 recommends.

The values of the localizable properties on the object itself are the
base values. Rules that the defining specification places on a
localizable property, such as a length limit, a non-empty value, or
uniqueness among siblings, apply to the base value. A server SHOULD
apply the same length and syntax limits to localized values. Uniqueness
rules apply only to base values unless the defining specification says
otherwise.

A server rejects a `/set` create or update whose `localizations`
breaks these restrictions, or has a key that is not a language tag,
with an `invalidProperties` SetError naming `localizations`. This
differs from {{RFC8984}} Section 4.6.1, under which a recipient ignores
an invalid patch: a JMAP server can report the error to the client
that sent it. A server MAY limit the number of
localizations it stores per object and reject more with the same
error.

When an update changes the base value of a localizable property, the
existing localized values of that property no longer translate the new
base value. For each language whose PatchObject the update does not
set, neither by setting `localizations` as a whole, nor the key for
that language, nor that property inside it, the server SHOULD remove
the property from that PatchObject, and remove the PatchObject if it is
left empty, as part of the same update. This keeps a client that does
not use this capability, or that updates only some languages, from
leaving stale translations behind when it renames an object. When the
server removes anything, it includes `localizations` in the object it
returns in `updated` ({{RFC8620}} Section 5.3) to clients that have
the capability in `using`.

For a server-set localizable property, such as Quota `name`, the server
populates `locale` and `localizations` itself. A server MAY also
populate them for objects it creates on the user's behalf, such as the
default Mailboxes of a new account.

## The language Argument {#language-argument}

When `urn:ietf:params:jmap:i18n` is in `using`, the standard `/get`,
`/query`, and `/queryChanges` methods ({{RFC8620}} Sections 5.1, 5.5,
and 5.6) of a data type with localizable properties take one
additional argument:

`language`: `String[]|null` (default: null)
: A list of language ranges ({{RFC4647}} Section 2.1), most preferred
  first. When null, objects are returned and compared as stored.

### In /get

When `language` is non-null, for each returned object the server:

1. Forms the set of available language tags: the keys of
   `localizations`, plus the object's `locale` when it is non-null.
2. Chooses one tag by the Lookup scheme of {{RFC4647}} Section 3.4,
   trying the ranges of `language` in order. When no range matches,
   it chooses the object's `locale`, which may be null.
3. Returns the localizable properties with the PatchObject for the
   chosen tag applied, when the chosen tag is a key of
   `localizations`, and with their base values otherwise.
4. Sets `locale` in the returned object to the chosen tag.

Lookup removes subtags from the end of a range but never adds them: the
range "de" matches a key "de" and not a key "de-CH" ({{RFC4647}}
Section 3.4). A client SHOULD therefore include a range for each
language it accepts in its general form, for example
`[ "de-CH", "de", "en" ]`. Lookup skips a range "*" that other ranges
follow, and a final "*" selects the default, which here is the base
values ({{RFC4647}} Section 3.4).

The returned object is then a view in one language, not the stored
object. Its `state` and the `/changes` state are those of the stored
object; choosing a different language does not change them.

When `urn:ietf:params:jmap:i18n` is in `using`, the server MUST NOT
let the Accept-Language header of the request change the result of
`/get` or `/query` for any localizable property; it applies a language
only when the client passes `language`. A client that caches objects
by id and state would otherwise hold a mix of languages and could not
tell. For the Quota `description`, this replaces the selection by
Accept-Language in {{RFC9425}} Section 4.1 for clients that opt in
({{quotas}}). {{RFC8620}} Section 3.8 still applies to custom strings
that are not properties of a data type, such as the `title` and
`detail` of a problem details object ({{problems}}).

A client that edits an object it obtained through `language` SHOULD
send only the properties it means to change. Writing a localized value
back to a base property replaces the base value, and writing back the
returned `locale` changes the base language.

### In /query

When `language` is non-null, a `/query` that sorts on, or filters on,
a localizable property compares the value that `/get` with the same
`language` would return, not the base value. When the Comparator names
no `collation`, the server MAY choose one suited to the language
chosen for each object, as {{RFC8620}} Section 5.5 already allows it to
do from Accept-Language.

### In /queryChanges

The `language` argument of `/queryChanges` has the same meaning as in
`/query`. The `queryState` of a `/query` result reflects the `language`
it was computed with. When the `language` of a `/queryChanges` differs
from that of the `/query` that produced `sinceQueryState`, the server
returns a `cannotCalculateChanges` error ({{RFC8620}} Section 5.6).

## Language of Responses over WebSocket {#websocket}

A Request sent over a WebSocket ({{RFC8887}} Section 4.3.2) has no Accept-Language
header of its own. A server SHOULD treat the Accept-Language header of
the WebSocket opening handshake as the Accept-Language of every Request
on that connection, for the purposes of {{RFC8620}} Section 3.8.

A Response sent over a WebSocket has no Content-Language header. The
display text in it is labelled by the `locale` of each object
({{properties}}), which serves both transports. This document defines
no response-level label: the custom strings of {{RFC8620}} Section 3.8
that a Response might carry, such as an error `description`, are
non-localized ({{RFC8620}} Sections 3.6.2 and 5.3).

# Applicability to Existing Data Types {#applicability}

When `urn:ietf:params:jmap:i18n` is in `using`, the following data
types gain `locale` and `localizations`, and accept the `language`
argument on `/get` and `/query`. Each table lists the localizable
properties.

## JMAP Mail {#mail}

| Data type | Localizable properties | Defined in |
|---|---|---|
| Mailbox | `name` | {{RFC8621}} Section 2 |

A Mailbox `name` is a Net-Unicode string {{RFC5198}} ({{RFC8621}}
Section 2); localized names SHOULD meet the same rule. The rule that
sibling Mailboxes have distinct names applies to base names only, so
two siblings can share a localized name; a client that shows localized
names MAY add the base name to tell them apart.
Clients that today show a translated name for a Mailbox with a `role`
can continue to do so; `localizations` lets a user or server give names
to Mailboxes that have no role.

Email content is not affected. Body parts already carry `charset` and
`language` ({{RFC8621}} Section 4.1.4).

Identity is not listed. Its `name` is the sender's own name as other
people see it in messages, so choosing it by the reader's `language`
is the wrong choice, and its signatures are message content, one of
them HTML. VacationResponse (Section 8 of {{RFC8621}}) is not listed
either: a vacation reply in several languages would be chosen by the
language of each incoming message, not by a client's `language`, and
is left to a future specification.

## JMAP Quotas {#quotas}

| Data type | Localizable properties | Defined in |
|---|---|---|
| Quota | `name`, `description` | {{RFC9425}} Section 4.1 |

Both properties are server-set. When the capability is not in
`using`, the server selects `description` from Accept-Language as
{{RFC9425}} Section 4.1 describes. When it is in `using`, this
document updates {{RFC9425}}: with `language` null, the server returns
the base values with their `locale` and all `localizations`, and with
`language` set, it selects as {{language-argument}} describes.

## JMAP Contacts {#contacts}

| Data type | Localizable properties | Defined in |
|---|---|---|
| AddressBook | `name`, `description` | {{RFC9610}} Section 2 |

A ContactCard is a JSContact Card {{RFC9553}}, which has its own
`language` and `localizations` (Sections 2.1.5 and 2.7.1 of
{{RFC9553}}). This document does not add properties to it, and the
`language` argument does not apply to `ContactCard/get`.

## JMAP Sharing {#sharing}

| Data type | Localizable properties | Defined in |
|---|---|---|
| Principal | `name`, `description` | {{RFC9670}} Section 2 |

A ShareNotification `name` records the name of an object at the time
of a change ({{RFC9670}} Section 3). It is not localizable. A server
SHOULD fill it with the base value.

## JMAP Calendars {#calendars}

JMAP for Calendars {{JMAP-CALENDARS}} is not yet published. When it
is, the following apply:

| Data type | Localizable properties |
|---|---|
| Calendar | `name`, `description` |

ParticipantIdentity is not listed, for the reason Identity is not
({{mail}}).

CalendarEvent objects are JSCalendar objects {{RFC8984}} and already
have `locale` and `localizations`. The `language` argument does not
apply to `CalendarEvent/get`.

## New Specifications {#new-specs}

A specification for a new JMAP data type that has display text can
adopt this document by listing that type's localizable properties and
requiring `urn:ietf:params:jmap:i18n` in `using` for them. A data type
whose objects are JSCalendar or JSContact objects uses their own
`localizations` and does not adopt this document.

# Text Handling in JMAP {#text-handling}

This section updates {{RFC8620}}. It applies to every JMAP
specification and implementation, whether or not
`urn:ietf:params:jmap:i18n` is in use, and changes nothing on the
wire.

## Encoding {#encoding}

{{RFC8620}} Section 1.5 makes every string UTF-8. Specifications
built on JMAP SHOULD cite that section instead of restating it. A
property that carries binary data or ciphertext as a string says how
it is encoded, for example base64url, and is not text for the purposes
of this section.

## Text Profiles {#profiles}

Display text SHOULD NOT contain:

- the control characters U+0000 to U+001F and U+007F to U+009F,
  except that multi-line display text, such as a description, uses
  U+000A LINE FEED to end lines;
- U+2028 LINE SEPARATOR or U+2029 PARAGRAPH SEPARATOR;
- the noncharacters U+FFFE and U+FFFF; or
- a leading U+FEFF used as a byte order mark.

A line ends with LF alone, not CR LF. These are the rules of
one-dimensional and two-dimensional Modern Network Unicode ({{MNU}}
Sections 2 and 3), stated here so that this document does not depend
on that one. A specification MAY require a stricter profile for a
property, as {{RFC8621}} does for Mailbox names with Net-Unicode
{{RFC5198}}. A server MAY reject display text that breaks these rules
with an `invalidProperties` SetError.

## Normalization {#normalization}

A server SHOULD NOT normalize text that it stores and returns, unless
the defining specification requires a normalization form, as
Net-Unicode {{RFC5198}} does for Mailbox names.
Normalization changes the bytes of text that may be signed, encrypted,
hashed, or compared with an earlier version, and it is not always
reversible.

Clients SHOULD send display text in Normalization Form C (NFC), as
{{RFC5198}} and {{MNU}} recommend. A new specification that compares
display text, for example to enforce a uniqueness rule, SHOULD say how;
comparing code point sequences is the simplest choice. This document
does not change the comparisons of existing specifications.

## Display Text Carries No Meaning {#no-meaning}

Servers and clients SHOULD NOT give display text meaning: they should
not decide behaviour by matching a name or description, in any
language, against a fixed string. Meaning belongs in an enumerated or
structured property. The Mailbox `role` of {{RFC8621}} is the model: a
client finds the inbox by its role, not by a name of "Inbox".

## Length Limits and Truncation {#lengths}

Length limits on strings in JMAP specifications are counted in octets
of UTF-8, as {{RFC9610}} and {{JMAP-CALENDARS}} do. New specifications
SHOULD state limits in octets.

When a server shortens text, for example to produce a preview such as
the Email `preview` of {{RFC8621}}, a notification body, or a push
payload, it SHOULD cut at an extended grapheme cluster boundary
({{UAX29}}), and never splits a code point. In particular it never
separates the two halves of a UTF-16 surrogate pair in an
implementation that holds strings as UTF-16, since the result would not
be valid I-JSON.

## Collation and Search {#collation}

{{RFC8620}} Section 5.5 already defines collation for sorting.
Specifications that define a text filter on display text SHOULD
compare using the collation the server would use to sort that
property, so that a case-insensitive sort gives a case-insensitive
filter. Tokenization and stemming for text search depend on
the language and are implementation-defined; a server MAY use `locale`
to choose them.

## Dates and Times {#time}

Instants on the wire are UTCDate ({{RFC8620}} Section 1.4) and carry
no locale. A property that holds a wall-clock or recurring time SHOULD
carry a time zone identifier from the IANA Time Zone Database, as the
Principal `timeZone` of {{RFC9670}} does and as {{RFC8984}} Section
1.4.8 recommends for TimeZoneId. Formatting dates, times, and numbers for display is the client's
business.

# URIs and Problem Details in JMAP {#uris-problems}

This section updates {{RFC8620}}. Like {{text-handling}}, it applies
to every JMAP specification and implementation, whether or not
`urn:ietf:params:jmap:i18n` is in use.

## URIs {#uris}

A property, argument, or Session member whose value is a URI or URL
SHOULD hold a URI as defined in {{RFC3986}} Section 3. Unless its
defining specification allows a relative reference, it SHOULD be a URI
and not a relative reference ({{RFC3986}} Section 4.2). A URI Template ({{RFC6570}}), such as
the `downloadUrl` of the Session, SHOULD expand to such a URI.

URIs in JMAP are ASCII. A client or server that holds an
Internationalized Resource Identifier ({{RFC3987}}), for example one a
user typed, SHOULD map it to a URI ({{RFC3987}} Section 3.1) before
sending it, and a client MAY convert a URI back to an IRI for display
({{RFC3987}} Section 3.2). Characters outside ASCII in a path, query,
or fragment are therefore sent as percent-encoded UTF-8. A domain name
SHOULD be sent in its A-label form, which {{RFC3987}} Section 3.1
permits, because many URI consumers do not accept a percent-encoded
host.

A server compares URIs that identify things, such as capability URIs,
as exact strings ({{RFC3986}} Section 6.2.1) unless the defining
specification names another comparison. A specification that defines
a URI-valued property SHOULD cite {{RFC3986}} instead of restating its
syntax.

## Problem Details {#problems}

Where {{RFC8620}} cites {{RFC7807}} for problem details (Sections 3.6.1,
6.1, and 6.2 of {{RFC8620}}), the reference is to {{RFC9457}}, which
obsoletes it. The problem types and members that {{RFC8620}} defines are
unchanged.

Every HTTP resource of a JMAP server SHOULD return a problem details
object ({{RFC9457}}) as the body of an error response with a 4xx or 5xx
status code. This includes the resources that {{RFC8620}} does not
cover: the Session resource, the event source ({{RFC8620}} Section
7.3), the WebSocket opening handshake ({{RFC8887}}), and any HTTP
resource that an extension adds. A specification that defines a new
HTTP resource for JMAP SHOULD say which problem types it returns and
SHOULD register the types it defines in the "JMAP Error Codes"
registry, which gives them the `urn:ietf:params:jmap:error:` prefix in
problem details ({{RFC8620}} Section 9.5).

The `title` and `detail` members are display text. A server MAY
localize them using Accept-Language ({{RFC9457}} Sections 3 and
3.1.3), and SHOULD label the response with Content-Language when it
does ({{RFC8620}} Section 3.8). Clients act on `type` and the HTTP
status code, never on `title` or `detail` ({{no-meaning}}).

# Security Considerations {#security}

## Confusable and Bidirectional Text {#confusables}

Display text from other users, such as the name of a shared address
book or of a Principal, can imitate other text. Characters from
different scripts can look alike ({{UTS39}}), and bidirectional
formatting characters can reorder what is shown ({{UAX9}}). Clients
SHOULD isolate display text from other users when they place it in
surrounding text, for example with U+2068 FIRST STRONG ISOLATE and
U+2069 POP DIRECTIONAL ISOLATE, or the HTML `bdi` element. Clients MAY
apply the confusable detection of {{UTS39}} to names they show next to
each other. Deployments that want enforcement can apply the PRECIS
Nickname profile {{RFC8266}} to names of people and the
IdentifierClass of {{RFC8264}} to identifiers.

## Different Text for Different Readers {#different-text}

Localizations let the owner of a shared object show each language a
different text. A localization need not be a translation of the base
value: a shared address book could be named "Team contacts" in English
and something unrelated in another language. Readers who use different
languages can be told different things about the same object. Clients
that let a user act on a shared object, for example by accepting a
share or granting rights, SHOULD show the base value and its `locale`
next to the localized value when they differ.

## Resource Use {#resources}

Each localization adds storage and response size. Servers limit the
number and size of localizations ({{properties}}) as they limit other
input.

## Language Preferences {#language-privacy}

A user's language preference is identifying information. The
`language` argument adds nothing beyond what Accept-Language already
reveals to the same server.

# Privacy Considerations {#privacy}

Localizations of a shared object are visible to everyone who can read
the object. A user who adds a localization reveals the languages they
use to those readers.

The Session lists shared accounts as well as the user's own. A server
SHOULD return `preferredLanguages` as null for an account that the
requesting user does not own, so that sharing an account does not
reveal its owner's languages ({{account-preference}}).

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP
Capabilities" registry established by {{RFC8620}}:

Capability Name:
: `urn:ietf:params:jmap:i18n`

Intended Use:
: common

Change Controller:
: IETF

Specification document:
: This document.

Security and Privacy Considerations:
: {{security}} and {{privacy}} of this document.

--- back

# Examples {#examples}

## Storing a Localized Address Book Name

~~~json
[[ "AddressBook/set", {
  "accountId": "a1",
  "update": {
    "ab1": {
      "locale": "en",
      "localizations": {
        "fr": { "name": "Contacts du groupe" },
        "de": { "name": "Team-Kontakte" }
      }
    }
  }
}, "0" ]]
~~~

## Reading It in One Language

Request, with `urn:ietf:params:jmap:i18n` in `using`:

~~~json
[[ "AddressBook/get", {
  "accountId": "a1",
  "ids": [ "ab1" ],
  "properties": [ "name", "locale" ],
  "language": [ "fr-CH", "de" ]
}, "0" ]]
~~~

The range "fr-CH" does not match, so Lookup tries its truncation "fr",
which does. Response:

~~~json
[[ "AddressBook/get", {
  "accountId": "a1",
  "state": "s42",
  "list": [ {
    "id": "ab1",
    "name": "Contacts du groupe",
    "locale": "fr"
  } ],
  "notFound": []
}, "0" ]]
~~~

# Relationship to Upstream JMAP Specifications {#upstream}

{{text-handling}} and {{uris-problems}} update {{RFC8620}} and would
fit in a revision of it. The capability and properties of {{localization}} could move into
a revision of {{RFC8620}}, with each data type's specification taking
its rows of {{applicability}}, at which point this document would be
obsoleted.

# Acknowledgements

The `locale` and `localizations` properties are those of JSCalendar
{{RFC8984}} by Neil Jenkins and Robert Stepanek. {{text-handling}}
relies on Modern Network Unicode {{MNU}} by Carsten Bormann.
