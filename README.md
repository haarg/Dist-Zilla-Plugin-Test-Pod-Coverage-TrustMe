# NAME

Archive::Tar::Wrapper - API wrapper around the 'tar' utility

# SYNOPSIS

```perl
use Archive::Tar::Wrapper;

my $arch = Archive::Tar::Wrapper->new();

# Open a tarball, expand it into a temporary directory
$arch->read("archive.tgz");

# Iterate over all entries in the archive
$arch->list_reset(); # Reset Iterator

# Iterate through archive
while(my $entry = $arch->list_next()) {
    my($tar_path, $phys_path) = @$entry;
    print "$tar_path\n";
}

# Get a huge list with all entries
for my $entry (@{$arch->list_all()}) {
    my($tar_path, $real_path) = @$entry;
    print "Tarpath: $tar_path Tempfile: $real_path\n";
}

# Add a new entry
$arch->add($logic_path, $file_or_stringref);

# Remove an entry
$arch->remove($logic_path);

# Find the physical location of a temporary file
my($tmp_path) = $arch->locate($tar_path);

# Create a tarball
$arch->write($tarfile, $compress);
```

# DESCRIPTION

**Archive::Tar::Wrapper** is an API wrapper around the `tar` command line
program. It never stores anything in memory, but works on temporary
directory structures on disk instead. It provides a mapping between
the logical paths in the tarball and the 'real' files in the temporary
directory on disk.

It differs from [Archive::Tar](https://metacpan.org/pod/Archive%3A%3ATar) in two ways:

- **Archive::Tar::Wrapper** almost doesn't hold anything in memory (see `write` method),
instead using disk as storage.
- **Archive::Tar::Wrapper** is 100% compliant with the platform's `tar`
utility because it uses it internally.

# METHODS

## new

```perl
my $arch = Archive::Tar::Wrapper->new();
```

Constructor for the `tar` wrapper class. Finds the `tar` executable
by searching `PATH` and returning the first hit. In case you want
to use a different tar executable, you can specify it as a parameter:

```perl
my $arch = Archive::Tar::Wrapper->new(tar => '/path/to/tar');
```

Since **Archive::Tar::Wrapper** creates temporary directories to store
`tar` data, the location of the temporary directory can be specified:

```perl
my $arch = Archive::Tar::Wrapper->new(tmpdir => '/path/to/tmpdir');
```

Tremendous performance increases can be achieved if the temporary
directory is located on a RAM disk. Check the ["Using RAM Disks" in Archive::Tar::Wrapper](https://metacpan.org/pod/Archive%3A%3ATar%3A%3AWrapper#Using-RAM-Disks)
section for details.

Additional options can be passed to the `tar` command by using the
`tar_read_options` and `tar_write_options` parameters. Example:

```perl
 my $arch = Archive::Tar::Wrapper->new(
               tar_read_options => 'p'
            );
```

will use `tar xfp archive.tgz` to extract the tarball instead of just
`tar xf archive.tgz`. GNU tar supports even more options, these can
be passed in via

```perl
 my $arch = Archive::Tar::Wrapper->new(
                tar_gnu_read_options => ["--numeric-owner"],
            );
```

Similarly, `tar_gnu_write_options` can be used to provide additional
options for GNU tar implementations. For example, the tar object

```perl
my $tar = Archive::Tar::Wrapper->new(
              tar_gnu_write_options => ["--exclude=foo"],
          );
```

will call the `tar` utility internally like

```
tar cf tarfile --exclude=foo ...
```

when the `write` method gets called.

By default, the `list_*()` functions will return only file entries:
directories will be suppressed. To have `list_*()` return directories
as well, use

```perl
 my $arch = Archive::Tar::Wrapper->new(
               dirs  => 1
            );
```

If more files are added to a tarball than the command line can handle,
**Archive::Tar::Wrapper** will switch from using the command

```
tar cfv tarfile file1 file2 file3 ...
```

to

```
tar cfv tarfile -T filelist
```

where `filelist` is a file containing all file to be added. The default
for this switch is 512, but it can be changed by setting the parameter
`max_cmd_line_args`:

```perl
 my $arch = Archive::Tar::Wrapper->new(
     max_cmd_line_args  => 1024
 );
```

The allowable parameters are:

- `tar`
- `tmpdir`
- `tar_read_options`
- `tar_write_options`
- `tar_gnu_read_options`
- `tar_gnu_write_options`
- `max_cmd_line_args`: defaults to 512
- `ramdisk`

Returns a new instance of the class.

## read

```
$arch->read("archive.tgz");
```

`read()` opens the given tarball, expands it into a temporary directory
and returns 1 on success or `undef` on failure.
The temporary directory holding the tar data gets cleaned up when `$arch`
goes out of scope.

`read` handles both compressed and uncompressed files. To find out if
a file is compressed or uncompressed, it tries to guess by extension,
then by checking the first couple of bytes in the tar file.

If only a limited number of files is needed from a tarball, they
can be specified after the tarball name:

```perl
$arch->read("archive.tgz", "path/file.dat", "path/sub/another.txt");
```

The file names are passed unmodified to the `tar` command, make sure
that the file paths match exactly what's in the tarball, otherwise
`read()` will fail.

## list\_reset

```
$arch->list_reset()
```

Resets the list iterator. To be used before the first call to `list_next()`.

## tardir

```
$arch->tardir();
```

Return the directory the tarball was unpacked in. This is sometimes useful
to play dirty tricks on **Archive::Tar::Wrapper** by mass-manipulating
unpacked files before wrapping them back up into the tarball.

## is\_compressed

Returns a string to identify if the tarball is compressed or not.

Expect as parameter a string with the path to the tarball.

Returns:

- a "z" character if the file is compressed with `gzip`.
- a "j" character if the file is compressed with `bzip2`.
- a "" character if the file is not compressed at all.

## locate

```
$arch->locate($logic_path);
```

Finds the physical location of a file, specified by `$logic_path`, which
is the virtual path of the file within the tarball. Returns a path to
the temporary file **Archive::Tar::Wrapper** created to manipulate the
tarball on disk.

## add

```
$arch->add($logic_path, $file_or_stringref, [$options]);
```

Add a new file to the tarball. `$logic_path` is the virtual path
of the file within the tarball. `$file_or_stringref` is either
a scalar, in which case it holds the physical path of a file
on disk to be transferred (i.e. copied) to the tarball, or it is
a reference to a scalar, in which case its content is interpreted
to be the data of the file.

If no additional parameters are given, permissions and user/group
id settings of a file to be added are copied. If you want different
settings, specify them in the options hash:

```perl
$arch->add($logic_path, $stringref,
           { perm => 0755, uid => 123, gid => 10 });
```

If $file\_or\_stringref is a reference to a Unicode string, the `binmode`
option has to be set to make sure the string gets written as proper UTF-8
into the tar file:

```perl
$arch->add($logic_path, $stringref, { binmode => ":utf8" });
```

## perm\_cp

Copies the permissions from a file to another.

Expects as parameters:

1. string of the path to the file which permissions will be copied from.
2. string of the path to the file which permissions will be copied to.

Returns 1 if everything works as expected.

## perm\_get

Gets the permissions from a file.

Expects as parameter the path to the source file.

Returns an array reference with only the permissions values, as returned by `stat`.

## perm\_set

Sets the permission on a file.

Expects as parameters:

1. The path to the file where the permissions should be applied to.
2. An array reference with the permissions (see `perm_set`)

Returns 1 if everything goes fine.

Ignore errors here, as we can't change uid/gid unless we're the superuser (see LIMITATIONS section).

## remove

```
$arch->remove($logic_path);
```

Removes a file from the tarball. `$logic_path` is the virtual path
of the file within the tarball.

## list\_all

```perl
my $items = $arch->list_all();
```

Returns a reference to a (possibly huge) array of items in the
tar file. Each item is a reference to an array, containing two
elements: the relative path of the item in the tar file and the
physical path to the unpacked file or directory on disk.

To iterate over the list, the following construct can be used:

```perl
# Get a huge list with all entries
for my $entry (@{$arch->list_all()}) {
    my($tar_path, $real_path) = @$entry;
    print "Tarpath: $tar_path Tempfile: $real_path\n";
}
```

If the list of items in the tar file is big, use `list_reset()` and
`list_next()` instead of `list_all`.

## list\_next

```perl
my ($tar_path, $phys_path, $type) = $arch->list_next();
```

Returns the next item in the tar file. It returns a list of three scalars:
the relative path of the item in the tar file, the physical path
to the unpacked file or directory on disk, and the type of the entry
(f=file, d=directory, l=symlink). Note that by default,
**Archive::Tar::Wrapper** won't display directories, unless the `dirs`
parameter is set when running the constructor.

## write

```
$arch->write($tarfile, $compress);
```

Write out the tarball by tarring up all temporary files and directories
and store it in `$tarfile` on disk. If `$compress` holds a true value,
compression is used.

## is\_gnu

```
$arch->is_gnu();
```

Checks if the tar executable is a GNU tar by running 'tar --version'
and parsing the output for "GNU".

Returns true or false (in Perl terms).

## is\_bsd

```
$arch->is_bsd();
```

Same as `is_gnu()`, but for BSD.

## ramdisk\_mount

Mounts a RAM disk.

It executes the `mount` program under the hood to mount a RAM disk.

Expects as parameter a hash with options to mount the RAM disk, like:

- `size`
- `type` (most probably `tmpfs`)
- `tmpdir`

Returns 1 if everything goes fine.

Be sure to check the ["Using RAM Disks" in Archive::Tar::Wrapper](https://metacpan.org/pod/Archive%3A%3ATar%3A%3AWrapper#Using-RAM-Disks) for full details on using RAM disks.

## ramdisk\_unmount

Unmounts the RAM disk already mounted with `ramdisk_mount`.

Don't expect parameters and returns 1 if everything goes fine.

Be sure to check the ["Using RAM Disks" in Archive::Tar::Wrapper](https://metacpan.org/pod/Archive%3A%3ATar%3A%3AWrapper#Using-RAM-Disks) for full details on using RAM disks.

# Using RAM Disks

On Linux, it's quite easy to create a RAM disk and achieve tremendous
speedups while untarring or modifying a tarball. You can either
create the RAM disk by hand by running

```perl
# mkdir -p /mnt/myramdisk
# mount -t tmpfs -o size=20m tmpfs /mnt/myramdisk
```

and then feeding the RAM disk as a temporary directory to
**Archive::Tar::Wrapper**, like

```perl
my $tar = Archive::Tar::Wrapper->new( tmpdir => '/mnt/myramdisk' );
```

or using **Archive::Tar::Wrapper**'s built-in option `ramdisk`:

```perl
my $tar = Archive::Tar::Wrapper->new(
    ramdisk => {
        type => 'tmpfs',
        size => '20m',   # 20 MB
    },
);
```

Only drawback with the latter option is that creating the RAM disk needs
to be performed as root, which often isn't desirable for security reasons.
For this reason, **Archive::Tar::Wrapper** offers a utility functions that
mounts the RAM disk and returns the temporary directory it's located in:

```perl
# Create new ramdisk (as root):
my $tmpdir = Archive::Tar::Wrapper->ramdisk_mount(
    type => 'tmpfs',
    size => '20m',   # 20 MB
);

# Delete a ramdisk (as root):
Archive::Tar::Wrapper->ramdisk_unmount();
```

Optionally, the `ramdisk_mount()` command accepts a `tmpdir` parameter
pointing to a temporary directory for the RAM disk if you wish to set it
yourself instead of letting **Archive::Tar::Wrapper** create it automatically.

# KNOWN LIMITATIONS

- Currently, only `tar` programs supporting the `z` option (for
compressing/decompressing) are supported. Future version will use
`gzip` alternatively.
- Currently, you can't add empty directories to a tarball directly.
You could add a temporary file within a directory, and then
`remove()` the file.
- If you delete a file, the empty directories it was located in
stay in the tarball. You could try to `locate()` them and delete
them. This will be fixed, though.
- Filenames containing newlines are causing problems with the list
iterators. To be fixed.
- If you ask **Archive::Tar::Wrapper** to add a file to a tarball, it copies it
into a temporary directory and then calls the system tar to wrap up that
directory into a tarball.

    This approach has limitations when it comes to file permissions: If the file to
    be added belongs to a different user/group, **Archive::Tar::Wrapper** will adjust
    the uid/gid/permissions of the target file in the temporary directory to
    reflect the original file's settings, to make sure the system tar will add it
    like that to the tarball, just like a regular tar run on the original file
    would. But this will fail of course if the original file's uid is different
    from the current user's, unless the script is running with superuser rights.
    The tar program by itself (without **Archive::Tar::Wrapper**) works differently:
    It'll just make a note of a file's uid/gid/permissions in the tarball (which it
    can do without superuser rights) and upon extraction, it'll adjust the
    permissions of newly generated files if the -p option is given (default for
    superuser).

# BUGS

**Archive::Tar::Wrapper** doesn't currently handle filenames with embedded
newlines.

## Microsoft Windows support

Support on Microsoft Windows is limited.

Versions below Windows 10 will not be supported for desktops, and for servers
only Windows 2012 and above.

The GNU `tar.exe` program doesn't work properly with the current interface of
**Archive::Tar::Wrapper**.

You must use the `bsdtar.exe` and make sure it appears first in the `PATH`
environment variable than the GNU tar (if it is installed). See
[http://libarchive.org/](http://libarchive.org/) for details about how to download and
install `bsdtar.exe`, or go to [http://gnuwin32.sourceforge.net/packages.html](http://gnuwin32.sourceforge.net/packages.html)
for a direct download. Be sure to look for the `bzip2` program package to
install it as well.

Windows 10 might come already with `bsdtar` program already installed. Please
search for that on the appropriate page (Microsoft keeps changing the link to
keep track of it here).

Having spaces in the path string to the tar program might be an issue too.
Although there is some effort in terms of workaround it, you best might avoid it
completely by installing in a different path than `C:\Program Files`.
Installing both `bsdtar` and `bzip2` in `C:\GnuWin32` will probably be enough
when running the installers.

# SEE ALSO

- Linux Gazette article from Ben Okopnik, [issue 87](https://linuxgazette.net/87/okopnik.html).

# BUGS

Please report any bugs or feature requests on the bugtracker website
[https://github.com/haarg/Archive-Tar-Wrapper/issues](https://github.com/haarg/Archive-Tar-Wrapper/issues)

When submitting a bug or request, please include a test-file or a
patch to an existing test-file that illustrates the bug or desired
feature.

# AUTHORS

- Mike Schilli <cpan@perlmeister.com>
- Alceu Rodrigues de Freitas Junior <glasswalk3r@yahoo.com.br>

# CONTRIBUTORS

- Chris Weyl <cweyl@alumni.drew.edu>
- David Cantrell <david@cantrell.org.uk>
- David Precious <davidp@preshweb.co.uk>
- Graham Knop <haarg@haarg.org>
- intrigeri <intrigeri@boum.org>
- Kent Fredric <kentfredric@gmail.com>
- Mark Gardner &lt;mjg+github@phoenixtrap.com>
- Mike Schilli <github@perlmeister.com>
- Mohammad S Anwar <mohammad.anwar@yahoo.com>
- Paulo Custodio <pauloscustodio@gmail.com>
- Randy Stauner <randy@magnificent-tears.com>
- Sanko Robinson <sanko@cpan.org>
- Shoichi Kaji <skaji@cpan.org>

# COPYRIGHT AND LICENSE

This software is Copyright (c) 2024 by Mike Schilli.

This is free software, licensed under:

```
The GNU General Public License, Version 3, June 2007
```
