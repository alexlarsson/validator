# Validator

Validator is a tool that allows you to sign a file, or a set of files,
using a private (Ed25519) key, and then later install these files on
another system if the signatures are valid. You can also validate the
files directly.

# Trivial example

Lets start with some very simple examples. First generate a
public/secret key-pair to sign the files:

```
$ openssl genpkey -algorithm ed25519 -outform PEM -out secret.pem
$ openssl pkey -in secret.pem -pubout -out public.pem
```

Then generate and sign a file:

```
$ echo foobar > a-file.txt
$ validator sign --key=secret.pem a-file.txt
$ ls  a-file.txt*
a-file.txt  a-file.txt.sig
```

This generated a signature (a-file.txt.sig) which we can validate:

```
$ validator validate --key=public.pem a-file.txt
```

If the file changes content this is detected:

```
$ validator validate --key=public.pem a-file.txt
Signature of './a-file.txt' is invalid (as a-file.txt)
```

Or if the signature is missing:

```
$ validator validate --key=public.pem a-file.txt
No signature for './a-file.txt'
```

Or if the file has changed name:

```
$ validator validate --key=public.pem renamed-file.txt
Signature of './renamed-file.txt' is invalid (as renamed-file.txt)
```

This is important, because filenames often are important for
configuration files, not just the content.

Suppose you have a file like this, you can now distribute it (with the
signature files) to another machine and install it, while validating
the data:

```
$ validator install --key=public.pem source/dir/a-file.txt dest/dir
```

Or you can recursively copy-and-validate all the files in a source
directory:

```
$ validator install --key=public.pem -r source/dir dest/dir
```

validator also supports signing and validating symlinks, which is
commonly used for configuration:

```
$ ln -s source_file a-symlink
$ validator sign --key=secret.pem a-symlink
$ ls  a-symlink*
a-symlink  a-symlink.sig
$ ln -sf wrong-source_file a-symlink
$ validator validate --key=public.pem a-symlink
Signature of './a-symlink' is invalid (as a-symlink)
```

# Read-only system usecase

A common usecase for validator is when you have a machine with a
read-only system partition (including /etc), and on top of this you
want to install extra configuration files that are individaul to each
machine. However you don't want to allow *any* such configuration file
to be installable, just those from a trusted source.

A simple example is when you have a read-only, signed, system image,
and then install containers at runtime to /var. For this to work you
then need to install a systemd service file into /etc to have them
automatically start at boot. But, if you allow any files to persist in
/etc then you open yourself up for a malicious actor to e.g. persist a
rootkit to next boot.

So, instead you install the file in e.g `/var/extra-etc`, and have
validator validate-and-copy the files into `/etc` the next boot using
a trusted public key in the read-only system partition.

For such a setup to work at all, the system must be configured to
allow the locations to be writable in a non-persistend way. A common
way to achieve this is to use
[systemd-volatile-root.service](https://www.freedesktop.org/software/systemd/man/latest/systemd-volatile-root.service.html),
or configure ostree with [transient /etc](https://ostreedev.github.io/ostree/man/ostree-prepare-root.html).

# Signature details

The data signed is a blob comprised of the type, the relative path of
the file, and the sha512 content of the file, or the symlink
target. The Ed25519 signature of this blob is then put next to the
original file with the suffix `.sig`.  In addition, the signature
files have an 8 byte header containing the bytes "VALIDTR\001".

Signatures can be generated using `validator sign`, such as:
```
$ validator sign --key=secret.pem path/to/the/file.txt
```

By default this will create a signature for just the basename of the
file (`file.txt` here). However, sometimes it is useful to validate a
longer relative path.

For example, you might have a file called
`path-to/extra-etc/subdir/file.txt` and you want to validate this
relative to `extra-etc`. This way you can recursively install from
`extra-etc` into `/etc` and and trust that the destination file ends
up in the `/etc/subdir` directory.

If you want to validate a longer path you have
several options.

Option 1 is to recursively sign, starting at `extra-etc`:
```
$ validator sign --key=secret.pem -r path-to/extra-etc
```
Option 2 is to specify a relative path:
```
$ validator sign --key=secret.pem --relative-to=path-to/extra-etc  path-to/extra-etc/subdir/file.txt
```

Option 3 is to supply the prefix separately (for example, if the signed file is in some other location).
```
$ validator sign --key=secret.pem --path-prefix=subdir other-path/file.txt
```

Applying signatures is sometimes complicated and has to be done in a
controlled environment. For this reason validator can output (using
the `validator blob` command) a raw file with the exact data that
would be signed. This allows the signature can be done using any
external tool that supports standard RFC 8032 Ed25519 signatures.
