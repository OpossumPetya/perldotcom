
  {
    "title"       : "JSON, Unicode, and Perl … Oh My!",
    "authors"     : ["felipe-gasper"],
    "date"        : "2020-01-27T07:11:44",
    "tags"        : [],
    "draft"       : true,
    "image"       : "",
    "thumbnail"   : "",
    "description" : "A look at this popular serialization’s relationship with Perl",
    "categories"  : "development"
  }

Consider the following code:

    use Mojo::JSON;
    use Cpanel::JSON::XS;
    use Data::Dumper;

    my $e_acute = "\xc3\xa9";

    my $json = Mojo::JSON::encode_json([$e_acute]);
    my $decoded = Cpanel::JSON::XS->new()->decode($json)->[0];

You might think this a reasonable enough round-trip, just using two
widely-used but different JSON libraries. In fact, though, when you run
this you’ll see that $decode in the above is `"\x{c3}\x{83}\x{c2}\x{a9}"`,
not just the `"\xc3\xa9"` that we started with.

Now invert the encoder/decoder modules:

    use Mojo::JSON;
    use Cpanel::JSON::XS;
    use Data::Dumper;
    $Data::Dumper::Useqq = 1;

    my $e_acute = "\xc3\xa9";

    my $json = Cpanel::JSON::XS->new()->encode([$e_acute]);
use Devel::Peek;
Dump($json);
    my $decoded = Mojo::JSON::decode_json($json)->[0];
    print Dumper( $json, $decoded );

Now $decode is just `"\x{e9}"`. What’s going on here?

What’s in a string?
===================

To appreciate the above, we first have to grapple with what Perl strings
_are_, fundamentally. Unlike C strings, Perl strings aren’t mere arrays
of bytes … but unlike, say, Python 3 strings, Perl strings aren’t arrays of
Unicode characters, either. Perl strings, rather, are arrays of “code
points”, in no particular encoding.

In particular, unlike Python, JavaScript, and many other popular high-level
programming languages, Perl strings do not differentiate between “binary”
and “text”. For example, if Perl reads
bytes 0xff, 0xfe, 0xfd, and 0xfc off of a binary filehandle, the string
that Perl creates from those 4 bytes is understood to contain not 4 _bytes_,
but 4 _code points_, without reference to any particular character set,
stored in an abstract, internal-use encoding.
(The Perl interpreter may, in fact, use 4 bytes to store the string, but that
would be an implementation detail, of no concern to interpreted Perl code.)

This point must be stressed: Perl _does not care_—and does not _want_ to
care—whether a given string’s code points represent bytes or characters.

Back to JSON
============

In our examples above we compared round-tripping using different libraries
for the encode and decode. Let’s dig further by comparing just the
encoded JSON:

    use Mojo::JSON;
    use Cpanel::JSON::XS;
    use Data::Dumper;
    $Data::Dumper::Useqq = 1;

    my $e_acute = "\xc3\xa9";

    my $mojo_json = Mojo::JSON::encode_json([$e_acute]);
    my $cp_json = Cpanel::JSON::XS->new()->encode([$e_acute]);
    print Dumper( $mojo_json, $cp_json );

This will print:

    $VAR1 = "[\"\303\203\302\251\"]";
    $VAR2 = "[\"\x{c3}\x{a9}\"]";

(Note that Data::Dumper outputs one string using octal escapes
and the other using hex. This reflects another Perl interpreter
implementation detail which, for now, is of no concern.)

Our input string contains two code points, 0xc3 and 0xa9. Recall that
there is no specific encoding associated with those code points; they’re
just numbers. JSON, per all published standards, is purely Unicode—and the
latest standard mandates UTF-8 encoding specifically. So how do we translate
those code points—which have no associated encoding—to UTF-8 in order to
encode to JSON?

We can’t, strictly speaking. It would be like trying
to convert 5 “currency units” to U.S. dollars: we need to know the actual
source currency (Bitcoin? Euros?) to get an answer. Likewise, in Perl, to
express our stored “code points” in UTF-8 we need to know what _characters_
those code points represent. For example, your Perl string might store code
point 42 … but which character is that? Perl doesn’t know, and Perl doesn’t
care. Code points without an associated
character set—which is what Perl strings are—by definition can’t indicate
any actual characters.

To work around this problem, our JSON libraries make reasonable—though
not necessarily correct—assumptions about what the string’s code points
represent.

Mojo::JSON assumes that our 2 original code points are Unicode. So,
Mojo::JSON thinks we gave it the characters U+00C3 (Ã) and
U+00A9 (©). The reason for the “expansion” from 2 code points to 4 in the
encoded JSON is that
Mojo::JSON encodes our code points as UTF-8: U+00C3 becomes Perl
code points 0303 (0xc3) and 0203 (0x83), and U+00A9 becomes 0302 (0xc2) and
0251 (0xa9).

Cpanel::JSON::XS makes a different assumption that suits a different
interpretation: This encoder assumes that our 2 original code points
represent the bytes of the characters that should go into the eventual JSON.
Unlike with Mojo::JSON, there is no assumption about a desired encoding,
which allows the caller full control over the encoding.

(This flexibility allows the encoder’s caller to choose, e.g., UTF-16 rather
than UTF-8 for the encoded JSON. That made more sense prior to the latest
JSON specification, which mandates UTF-8 outside closed systems.)

The same difference in behavior applies to our two decoder functions. They,
too, face an “unsolvable” problem, the reverse of that for encoding. And
their solutions mirror the encoders’.

    use Mojo::JSON;
    use Data::Dumper;
    $Data::Dumper::Useqq = 1;

    my $from_mojo = "[\"\303\203\302\251\"]";
    my $from_cp = "[\"\x{c3}\x{a9}\"]";

    $from_mojo = Mojo::JSON::decode_json($from_mojo)->[0];
    $from_cp = Mojo::JSON::decode_json($from_cp)->[0];
    print Dumper( $from_mojo, $from_cp );

This will print:

    $VAR1 = "\x{c3}\x{a9}";
    $VAR2 = "\x{e9}";

Recall that Mojo::JSON’s encoder interprets its input as Unicode and that
its output code points represent bytes of UTF-8.
Above you’ll see that its decoder does the inverse: it interprets its
input as bytes of UTF-8 and outputs code points understood to be Unicode.
This means the number of code points output will be smaller than the number
input if the input contains any code points above 127 (0x7f), which UTF-8
represents as multiple bytes.

As for Cpanel::JSON::XS:

    use Mojo::JSON;
    use Data::Dumper;
    $Data::Dumper::Useqq = 1;

    my $from_mojo = "[\"\303\203\302\251\"]";
    my $from_cp = "[\"\x{c3}\x{a9}\"]";

    $from_mojo = Cpanel::JSON::XS->new()->decode($from_mojo)->[0];
    $from_cp = Cpanel::JSON::XS->new()->decode($from_cp)->[0];
    print Dumper( $from_mojo, $from_cp );

This gives:

    $VAR1 = "\x{c3}\x{83}\x{c2}\x{a9}";
    $VAR2 = "\x{c3}\x{a9}";

The `decode()` method, like `encode()`, assumes that the caller will
handle encoding manually and so simply copies code points.

Abusing the System
==================

Cpanel::JSON::XS’s `encode()` allows for a nonstandard use of JSON:
literal binary data. Consider the following:

    perl -MCpanel::JSON::XS -e'print Cpanel::JSON::XS->new()->encode(["\xff"])'

… will output 5 bytes: `[`, `"`, 0xff, `"`, and `]`. This is invalid JSON
because no Unicode encoding (let alone UTF-8) ever encodes a character to
a single 0xff byte. Only special decoders that understand this “literal
binary” JSON variant will parse this as intended. That reliance on a custom
mode of operation undercuts JSON’s
usefulness as a widely-supported standard—which may seem fine at first but
can easily bite if your application grows in scope.

Applications that need to serialize strings with arbitrary octets (i.e.,
binary) should apply a secondary encoding (e.g., Base64) to strings prior
to JSON encoding. Or, better yet, prefer a binary-friendly encoding like
[CBOR](http://cbor.io).

About That Flag Behind the Curtain …
====================================

If you run the output from our two encoder methods through Devel::Peek, you’ll
see something like this for Mojo::JSON’s output:
```
SV = PV(0x7fdc27802f30) at 0x7fdc27e59c58
  REFCNT = 1
  FLAGS = (POK,pPOK)
  PV = 0x7fdc28826350 "[\"\303\203\302\251\"]"\0
  CUR = 8
  LEN = 34
```
… and this for Cpanel::JSON::XS:
```
SV = PV(0x7fc0cd004d30) at 0x7fc0cd016228
  REFCNT = 1
  FLAGS = (POK,pPOK,UTF8)
  PV = 0x7fc0cce2ef60 "[\"\303\203\302\251\"]"\0 [UTF8 "["\x{c3}\x{a9}"]"]
  CUR = 8
  LEN = 34
```
Note the `UTF8` indications in the latter. This tells us that Perl’s
internal storage of the string’s code points uses UTF-8 encoding. This
difference is why, as we saw earlier, Data::Dumper encodes Mojo::JSON’s output
using
octal escapes but Cpanel::JSON::XS’s using hex: Data::Dumper recognizes the
UTF8 flag and renders its output based on it.

It is sometimes possible to regard UTF8-flagged strings as “character
strings” and non-UTF8-flagged strings as “byte strings”—indeed, multiple
serializers on CPAN, including two of my own, do exactly this. This isn’t
a supported model, though, for understanding Perl strings, and any code that
depends on it may behave differently in different Perl versions. Caveat
emptor!

Making Peace
============

JSON and Perl are odd bedfellows. Perl’s lack of distinction between numbers
and strings, for example, can lead to JSON that uses the wrong type for one
value or the other. Perl’s lack of native booleans can yield a similar effect.

The encoding problems discussed above, though, are particularly nefarious
because correcting for them requires a deeper understanding of all of this.
For example, most can accommodate `{"age": "9"}` easily enough
because typecasting from `"9"` (string) to `9` (number) is commonplace. But
how many would see `"Ã©"` and think, “ah! I simply have to treat those
characters’ code points as bytes then decode those bytes as UTF-8!” Some
would, to be sure—perhaps even many—but likely fewer than can easily coerce
`"9"` to `9`.

Binary-friendly encodings like [CBOR](http://cbor.io)
mitigate against this problem because whatever decodes the Perl-sourced
data can more easily recognize the need to decode from binary. Anyone
who doesn’t know about bytes and encodings will quickly learn! Fundamentally,
though, even CBOR doesn’t really fit Perl’s “pure code points” string model
very well because CBOR distinguishes strongly between binary and text strings,
which Perl does not.

At the end of the day, communication between Perl and other modern languages
is a challenge. The best we can do is to know of these
problems and deal with them as they arise.