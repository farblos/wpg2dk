# wpg2dk - Migrate from Windows Photo Gallery to digiKam

## Pros and Cons and Alternatives

As the title says, this tool should help migrating the metadata
of your photos or other media from Windows Photo Gallery.  In
contrast to the various other receipts and approaches you may
find on this topic in the net, it has the following pros and
cons:

Pros:

- It should cover *all* metadata manageable in Windows Photo
  Gallery

- with a consistent approach (no tweaking here and poking there)
  and

- it should do so correctly (no incorrectly rotated face tags).

Cons:

- The whole process is quite technical and currently geared to
  those who are not afraid of that beast called "command line".

Alternatives:

- [Digikam/Tutorials/Switch from Microsoft OneDrive or WLPG to
  digiKam](https://userbase.kde.org/Digikam/Tutorials/Switch_from_Microsoft_OneDrive_or_WLPG_to_digiKam#Microsoft_image_metadata)
  together with [Digikam/Tutorials/Setup of digiKam for Windows
  compatibility](https://userbase.kde.org/Digikam/Tutorials/Setup_of_digiKam_for_Windows_compatibility)

## Overview

Windows Photo Gallery (WPG) aka. Windows Live Photo Gallery
(WLPG) has been desupported by Microsoft.  To avoid similar
trouble with other proprietary products in the future, I decided
to migrate our photo collection to digiKam running on GNU/Linux.

Some initial tests showed that some of the metadata maintained by
WPG is already used by digiKam, some is not, and some is broken
(by WPG, not digiKam).  More tests showed that the most complete
and accurate source of WPG metadata is actually the WPG metadata
database itself, and not the metadata stored by WPG in the media
files.

So the overall migration process goes like this:

1.  Extract the WPG metadata from the database and store it in an
    SQLite database.

    See section [Extracting the WPG
    Metadata](#extracting-the-wpg-metadata).

2.  From the SQLite database, extract the WPG metadata and
    convert it into XMP files, called "intermediate XMP files" in
    the following.

    See section [Converting the WPG
    Metadata](#converting-the-wpg-metadata).

3.  Inject or merge the metadata contained in the intermediate
    XMP files into the media files with ExifTool.

    See section [Injecting the WPG
    Metadata](#injecting-the-wpg-metadata).

4.  Read the image metadata with digiKam.

    See section [Reading the Image Metadata with
    digiKam](#reading-the-image-metadata-with-digikam).

Ideally and most easily the whole process is done as a one-way
action where you start with exclusive WPG usage and end up with
exclusive digiKam usage.  Any past or future switching between
photo organization software has the potential of introducing
metadata inconsistencies.  In contrast to that, this [digiKam
tutorial](https://userbase.kde.org/Digikam/Tutorials/Setup_of_digiKam_for_Windows_compatibility),
tries to show how digiKam can be configured so that it handles
metadata as compatible as possible with Microsoft products.

This project has been developed and tested on "family photo
collection scope" with most of the media being JPEG images.
However, the general approach described above is not limited to
these as long as ExifTool is able to inject the metadata from the
intermediate XMP files into non-JPEG media files.

The tool itself would have be better named
WPG-metadata-database-to-XMP, meaning that it is not limited to
only digiKam as migration target.  Some of the generated XMP
attributes are specific to digiKam, but at least ExifTool should
be able to convert these to non-digiKam-specific attributes.

For the the tool versions I have used for developing and testing
`wpg2dk` and this readme, see section [Background and Version
Information](#background-and-version-information).

## Migration Process Prerequisites and Miscellaneous

This document describes the migration process as done on Windows
(for extracting the WPG metadata) and on GNU/Linux (for all
remaining steps).  It should be possible to complete the whole
migration process on Windows, but I have not described nor tested
that.

On Windows, you must still have the WPG metadata database below
directory `%LOCALAPPDATA%\Microsoft\Windows Live Photo Gallery`
and it must still be up-to-date with respect to your media files.
Without that, migration according to this document is impossible
or cannot be guaranteed to be complete.

Regardless of operating system, `wpg2dk` requires a recent Perl
version and non-standard Perl modules `DBI`, `DBD::SQLite`, and
`Image::ExifTool`, which (on GNU/Linux) should be available in
your distribution's package system or (on any operating system)
can be downloaded and installed from
[CPAN](https://www.cpan.org/).

For injecting the metadata into the media files as described in
this readme, one also needs a recent installation of
[ExifTool](https://exiftool.org).

When migrating your media files, use the general common sense for
such projects.  For example:

- Operate on small subsets of the media files for testing.

- Document when and what exactly you have done to migrate the
  media files, for example by keeping the transcripts of your
  terminal sessions and by keeping the results of any relevant
  `wpg2dk list` commands.

- Keep backups of your original media files.  ExifTool itself
  saves all media files before modifying them for the first time
  as explained in section [Undoing or Comitting ExifTool's
  Changes](#undoing-or-comitting-exiftools-changes), but more
  backups are always better.

- Keep *permanent* backups if not of your original media files,
  then at least of their metadata.  Otherwise you might be in
  trouble when you notice after a year that, while the majority
  of the media files migrated just fine, this one small set of
  photos got their metadata all screwed up.

You can use ExifTool to create recursive metadata-only backups
with option `-o`, like this:

    exiftool -all:all -o md-backup/%d%f.exv \
             -r. <media-directory> ...

This would create the backups in a directory structure parallel
to that of the specified media directories but below directory
`md-backup`.  For such backups, ExifTool author Phil Harvey
[recommends](https://exiftool.org/forum/index.php?topic=9458.0#msg48944)
using metadata file types `EXV` or `EXIF` depending on the media
format.  In contrast to these, file type `MIE`, while being
supported only by ExifTool itself, has the advantage of being
suitable for all media formats.

ExifTool is not only useful for backups or for the migration
itself, but also for inspecting and comparing media file
metadata.  My favorite with respect to that is:

    exiftool -j -G1 -struct --composite <media-file>

## Why Using ExifTool to Inject the WPG Metadata?

Originally, I planned to create complete sidecar XMP files
containing the merged metadata from both the media files and the
WPG database and let digiKam pick up these sidecar files.

However, it soon became clear that merging media metadata is a
highly non-trivial process, so I decided to better delegate that
to ExifTool with its plethora of features to select tags, files
to process, etc.

## Tags, Keywords, Attributes

To avoid confusion: "Tags" in WPG, digiKam, `wpg2dk` and this
documentation denote text attributes that can be attached to
media files, like geotags, people tags, or general descriptive
tags.  In ExifTool a "tag" denotes the smallest unit of metadata
that can be processed.  An attribute that contains a WPG tag is
called "keyword" in ExifTool.  Finally, this documentation and
`wpg2dk` use the term "attribute" to refer to parts of metadata.
So we have more or less this translation table:

| WPG, digiKam, `wpg2dk` | ExifTool | `wpg2dk`  |
|:-----------------------|:---------|:----------|
| tag                    | keyword  |           |
|                        | tag      | attribute |

## Extracting the WPG Metadata

1.  In a PowerShell, define the name of an (absent) directory
    used as working directory for the export process, create that
    directory, and copy the WPG Metadata to it:

    ```
    $wpgmdexpdir="<absolute-path-to-a-new-directory>"
    mkdir $wpgmdexpdir
    cp -rec "${ENV:LOCALAPPDATA}\Microsoft\Windows Live Photo Gallery\*" `
            $wpgmdexpdir
    ```

    Directory `$wpgmdexpdir` should now contain at least the
    following WPG metadata files:

    ```
    FaceExemplars.ed1
    FaceThumbs.fd1
    Pictures.pd6
    ```

2.  Download `exportsqlce31.zip` from [EricEJ's
    SqlCeToolbox](https://github.com/ErikEJ/SqlCeToolbox/releases/tag/3.5.2).
    Unpack it into `$wpgmdexpdir`, for example with

    ```
    unzip.exe -qd $wpgmdexpdir exportsqlce31.zip
    ```

3.  Export the SQL CE data as follows:

    ```
    & $wpgmdexpdir\ExportSqlCe31.exe "Data Source = '$wpgmdexpdir\FaceExemplars.ed1'" `
                                     $wpgmdexpdir\FaceExemplars.sql sqlite
    & $wpgmdexpdir\ExportSqlCe31.exe "Data Source = '$wpgmdexpdir\FaceThumbs.fd1'"    `
                                     $wpgmdexpdir\FaceThumbs.sql    sqlite
    & $wpgmdexpdir\ExportSqlCe31.exe "Data Source = '$wpgmdexpdir\Pictures.pd6'"      `
                                     $wpgmdexpdir\Pictures.sql      sqlite
    ```

    Each of the above calls should result in output along the
    lines of

    ```
    Initializing....
    Generating the tables....
    Generating the data....
    Generating the indexes....
    Sent script to output file(s) : <filename>.sql in <duration> ms
    ```

    plus one file `<filename>.sql` or (if the SQL file got larger
    than 10MB) multiple files `<filename>_<NNNN>.sql` created in
    directory `$wpgmdexpdir`.

4.  On a GNU/Linux host and in some Bourne shell, define the name
    of an (absent) directory used as working directory for the
    import process and create that directory:

    ```
    wpgsqlitedir="<absolute-path-to-a-new-directory>"
    mkdir "$wpgsqlitedir"
    ```

5.  Copy the WPG metadata SQL files to directory
    `$wpgsqlitedir`.

    Currently, `wpg2dk` only requires the WPG metadata from
    `Pictures*.sql` as input.  But it is probably a good idea to
    keep the data of all metadata files in a format that stays
    readable even when SQL CE database support ceases completely.

6.  Create the SQLite databases corresponding to the WPG metadata
    and import the SQL files:

    ```
    for wpgmddbid in "FaceExemplars" "FaceThumbs" "Pictures"; do
      set x; shift
      for sqlfile in "$wpgsqlitedir/$wpgmddbid"*.sql; do
        set ${1+"$@"} ".read '$sqlfile'"
      done
      sqlite3 "$wpgsqlitedir/$wpgmddbid.db" "$@"
    done
    ```

    This should result in three files

    ```
    FaceExemplars.db
    FaceThumbs.db
    Pictures.db
    ```

    created in directory `$wpgsqlitedir`.

## Converting the WPG Metadata

`wpg2dk` extracts only metadata from the WPG metadata database
which can be assigned in the right hand pane of WPG, which is:

- People tags
- Geotag
- Caption
- Descriptive tags
- Rating
- Flag

The general idea is that `wpg2dk` always extracts all available
metadata from the WPG metadata.  As explained in the next
section, the rich command line interface of ExifTool can then be
used to control what parts of that metadata are actually injected
into the media files.

### Mapping Media Object File Paths

To convert the WPG metadata, `wpg2dk` must have access to the
SQLite database `Pictures.db` created in the previous section
**plus** all the media files.  The challenge here is to map from
the file path information stored in the WPG metadata to the paths
of the actual media files in the file system.

As a first step, run `wpg2dk` with command `summary`, like this:

    [wpg2dk]$ ./wpg2dk summary "$wpgsqlitedir/Pictures.db"
    Media object counts:

      total                  4369
      with errors            4369
      with warnings             0
      without any issues        0

    Media object counts by error ID.  Objects having
    errors with different IDs are counted more than once.

      no_wpgmd               1248
      unmapped_file          3121

    Use command "listerr" to list objects with errors or warnings.

Initially, this most likely lists only media objects with error
IDs `no_wpgmd` and `unmapped_file`.  The former do not have any
WPG metadata (as listed in the previous section) stored in the
database, so there is nothing to export for these.  For media
objects with error ID `unmapped_file`, no mapping is defined to
map the file path information in the WPG metadata to actual media
files.  We will focus on these with command `listerr`:

    [wpg2dk]$ ./wpg2dk listerr -M unmapped_file "$wpgsqlitedir/Pictures.db" | head -5
    unmapped_file
      #2/images/00/dsc-0007.jpg:
        #2/images/00/dsc-0007.jpg
      #2/images/00/dsc-0008.jpg:
        #2/images/00/dsc-0008.jpg

Command `listerr` with parameter `-M unmapped_file` specified
lists all media objects having an error with that ID.  The lines
indented by two blanks provide the user interface ID (UIID) of
the media object, the lines indented by four blanks its (yet
unmapped) file name.  Both UIIDs and file names use path
separators according to the OS `wpg2dk` is running on, so on
Windows above would list backslash-separated UIIDs and file
names.

The UIID is some more or less readable and hopefully unique
identifier that is used to identify media objects with `wpg2dk`.
By default, the UIID coincides with the file name, but there are
different formats for the UIID available.

The WPG metadata refers to the volumes that contain the media
files by volume ID.  These are the numbers behind the initial
hash signs shown above.  The rest of the media file names
consists of their directory path on the volume and their file
name as found in the WPG metadata.  The result of these three
items joined together with the path separator in between we call
"pseudo file name".

`wpg2dk` does not have the information on the drive letters that
are used or have been used to mount the volumes referenced in
these file names.  So it is basically up to us to identify the
correct volumes by the remainder of the file paths.  In the case
of our example, volume 2 happens to be the NTFS volume labeled
"archive".  Assume we have mounted that on path `/media/archive`,
so we must map somehow the volume prefix `#2/` to
`/media/archive/`.

To do so, we specify a media file map with command line option
`--mmap '<source>=<destination>'`, like in the following example:

    [wpg2dk]$ ./wpg2dk summary --mmap '#2/images/00/=/media/archive/images/00/' \
              "$wpgsqlitedir/Pictures.db"
    Media object counts:

      total                  4369
      with errors            4216
      with warnings             0
      without any issues      153

    Media object counts by error ID.  Objects having
    errors with different IDs are counted more than once.

      no_wpgmd               1248
      unmapped_file          2968

    Use command "listerr" to list objects with errors or warnings.

In above example, we have chosen to map only a subset of the
media objects (those below subdirectory `/images/00`), just to
check whether the map works at all.  It does, since now 153 media
objects got processed "without any issues" according to the
report.  So we can now try to map all media objects:

    [wpg2dk]$ ./wpg2dk summary --mmap '#2/=/media/archive/' \
              "$wpgsqlitedir/Pictures.db"
    Media object counts:

      total                  4369
      with errors            1248
      with warnings           722
      without any issues     2399

    Media object counts by error ID.  Objects having
    errors with different IDs are counted more than once.

      no_wpgmd               1248

    Media object counts by warning ID.  Objects having errors
    are not counted here.  Objects having warnings with
    different IDs are counted more than once.

      exiftool_warnings       722

    Use command "listerr" to list objects with errors or warnings.

If the mapping has been done properly so that all media files can
be found according to it, above command can take quite some time
to complete since `wpg2dk`, if needed, opens the found media
files using ExifTool to determine their orientation.

ExifTool is quite picky with respect to metadata quality and
finds a lot of problems in the metadata generated by camera
vendors or by WPG itself.  As long as these problems are only
warnings (which are registered with warning ID
`exiftool_warnings`), they can be most likely simply ignored.

Furthermore, not all issues called "errors" by `wpg2dk` refer to
real errors that need some extra care or fixing.  Many of these
errors just mean: "with the current state of metadata it does not
make any sense to further process this media object."  The best
example is "error" `no_wpgmd` for some media object, where
`wpg2dk` simply has no work to do and, hence, does not continue
processing that object.

After the mapping has been established we can do the final step
and extract the WPG metadata to the intermediate XMP files:

    [wpg2dk]$ ./wpg2dk extract --mmap '#2/=/media/archive/' \
              "$wpgsqlitedir/Pictures.db"
    Media objects statistics:
    Total:      4369, with errors:  1248, with warnings:   722, OK:  2399
    Selected:   4369, with errors:  1248, with warnings:   722, OK:  2399
    Media objects with errors have not been processed.

In the default configuration, `wpg2dk` creates intermediate XMP
files in the same directory as the media files, but with
extension `.wpg2dk.xmp`.  Like this:

    [wpg2dk]$ ls -l /media/archive/images/00/ | head
    total 455840
    -rw-r--r-- 1 farblos farblos 1302611 Apr 21  2007 dsc-0005.jpg
    -rw-r--r-- 1 farblos farblos 1263321 Apr 21  2007 dsc-0006.jpg
    -rw-r--r-- 1 farblos farblos 1205224 Jan  6  2015 dsc-0007.jpg
    -rw-r--r-- 1 farblos farblos    1064 Mar  7 21:43 dsc-0007.jpg.wpg2dk.xmp
    -rw-r--r-- 1 farblos farblos 1281692 Jan  8  2015 dsc-0008.jpg
    -rw-r--r-- 1 farblos farblos    1062 Mar  7 21:43 dsc-0008.jpg.wpg2dk.xmp
    -rw-r--r-- 1 farblos farblos 1287181 Jan  6  2015 dsc-0009.jpg
    -rw-r--r-- 1 farblos farblos 1278844 Apr 21  2007 dsc-0010.jpg
    -rw-r--r-- 1 farblos farblos 1284972 Jan  8  2015 dsc-0011.jpg

(There are no intermediate XMP files for some of the images shown
above because these do not contain any WPG metadata.)

You can use command line option `--xmap` to change the extension
that `wpg2dk` appends to the media files.  You can use that
option also to let `wpg2dk` create the intermediate XMP files in
a completely different directory, separated from the media files.
The following instructions related to ExifTool, however, assume
the default configuration.

Besides the `wpg2dk` commands `summary`, `listerr`, and `extract`
demonstrated above there is also command `list`, that writes the
names of all media files for which intermediate XMP files have
been created (or will be created) to STDOUT:

    [wpg2dk]$ ./wpg2dk list --quiet --mmap '#2/=/media/archive/' \
              "$wpgsqlitedir/Pictures.db" | wc -l
    3121

Command `list` is particularly useful for calling ExifTool in the
so-called "list option" as shown in the next section, where the
file list written by `wpg2dk` is used as input for ExifTool.

By default `wpg2dk` processes all media objects it finds the WPG
metadata database.  To select only a subset either by inclusion
or exclusion, you can specify patterns matching their user
interface ID (UIID) on the command line.  For example, the
following selects all media objects from subdirectory
`.../images/00`:

    [wpg2dk]$ ./wpg2dk list --mmap '#2/=/media/archive/' \
              "$wpgsqlitedir/Pictures.db" 'm{/images/00/}' > /dev/null
    Media objects statistics:
    Total:      4369, with errors:  1248, with warnings:     3, OK:  3118
    Selected:    249, with errors:    96, with warnings:     0, OK:   153
    Media objects with errors have not been processed.

In above example, the row labelled `Total:` displays statistics
for all media objects, while the row labelled `Selected:`
statistics for the media objects from subdirectory
`.../images/00`, which are matched by UIID selector
`'m{/images/00/}'`.

To exclude certain media objects from processing, prefix the UIID
selector with parameter `-X`.  With the following command line,
for example, you can suppress processing of certain movie types,
which ExifTool cannot write:

    [wpg2dk]$ ./wpg2dk list --mmap '#2/=/media/archive/' \
              "$wpgsqlitedir/Pictures.db" -X 'm{\.(?i:mpeg|mpg|wmv|avi)$}' |
    Media objects statistics:
    Total:      4369, with errors:  1248, with warnings:     3, OK:  3118
    Selected:   4344, with errors:  1246, with warnings:     3, OK:  3095
    Media objects with errors have not been processed.

In general, the `wpg2dk` commands are completely interchangeable.
So if you have set up a command line with a complicated map,
exclusions, and other options for the `extract` command, you can
simply exchange command `extract` by command `list` and call
`wpg2dk` with the resulting command line.

## Injecting the WPG Metadata

At this point we have the media files and the intermediate XMP
files alongside with them, so it should be as simple as calling
ExifTool on these recursively to merge them.  Unfortunately, this
turned out to be not as simple as I initially thought.

The main reason is that one has to decide how to handle merge
conflicts between the metadata possibly present in the media
files and the metadata present in the intermediate XMP files:

- In the case of GPS coordinates, for example, one would not like
  to overwrite any existing GPS coordinates with the much coarser
  ones created by `wpg2dk` from the WPG geotags.

- In other cases, there might by the option to overwrite existing
  metadata or to append to it.

- Finally, for most attributes there is more than one way to
  specify them in the metadata.  `wpg2dk` uses
  `XMP-digiKam:TagsList` to write tags, for example, but digiKam
  can use at least seven different other attributes to store
  tags.

As a side note, for some metadata attributes digiKam can be
configured (in `Settings` &rarr; `Configure digiKam` &rarr;
`Metadata` &rarr; `Advanced`) how to exactly access the
attributes.  I decided to leave that all on default and rather
use ExifTool to come out with (hopefully) unambiguous metadata.

In the next section I first provide the bare skeleton ExifTool
command to specify which intermediate XMP files to merge into
what media files, not caring about merge conflicts at all.  After
that, I describe for each attribute group processed by `wpg2dk`
(people tags, geotag, etc.) how to merge the attributes of that
group into the media files without causing any conflicts.

The general idea is that from these sections you pick those that
you actually need and apply the contained ExifTool commands step
by step, possibly further modifying them to your needs.  See the
the examples in the [ExifTool application
documentation](https://exiftool.org/exiftool_pod.html) for more
information.

Except for the GPS coordinates, I chose not to care about any
metadata possibly present in the media files and to simply
overwrite all attributes with what `wpg2dk` has generated.
Again, refer to the ExifTool documentation and change the
statements provided below to customize that approach to your
needs.

Above reference to "all attributes" requires probably some
explanation: The ExifTool invocations in the following sections
clean up all metadata attributes that both WPG or digiKam may
have written to a media file to store any of [people
tags](#people-tags), [geotag](#geotag), [caption](#caption),
[descriptive tags](#descriptive-tags), [rating](#rating), or
[flag](#flag).  See the metadata attributes labeled "WPG
attributes" and "DK attributes" in the respective referenced
sections for details.

Finally, digiKam has the option to manage metadata in sidecar XMP
files (and these are the main reason why the intermediate XMP
files are not called just "XMP files").  So in theory you can
also use ExifTool to inject the intermediate XMP files into the
sidecar XMP files read from (and written by) digiKam, thus
leaving your precious media files completely unchanged.  Which I
have not tested nor documented.

### Specifying the Files to Process

When you call `wpg2dk` with the default intermediate XMP file
map, it creates the intermediate XMP files alongside with the
media files but with additional extension `.wpg2dk.xmp`.

You can merge the metadata of the intermediate XMP files into the
media files with the following generic ExifTool command line:

    exiftool -if '-f "$directory/$filename.wpg2dk.xmp"' \
             [<tag-to-clean-up> ...]                    \
             -tagsFromFile '%d%F.wpg2dk.xmp'            \
             [<tag-to-copy-from-xmp> ...]               \
             -q -q -r. <media-directory> ...

where the command line parameters have the following meaning:

- `-if '-f "$directory/$filename.wpg2dk.xmp"'`: process only
  media files actually having accompanying intermediate XMP files

- `<tag-to-clean-up> ...`: list of attributes to clean up before
  merging new tags from the intermediate XMP files - to be
  detailed in the following sections.  If not specified, no
  attributes are cleaned up.

- `-tagsFromFile '%d%F.wpg2dk.xmp'`: read metadata attributes
  from the intermediate XMP files associated to the media files
  (`%d%F`) and inject them into the media files

- `<tag-to-copy-from-xmp> ...`: list of attributes to merge from
  the intermediate XMP files - to be detailed in the following
  sections.  If not specified, all attributes are merged.

- `-q -q`: suppress all warnings.  There might be plenty of
  these, and without these options the more important errors may
  go unnoticed.

- `-r.`: process the following media directories recursively
  including any "hidden" directories

- `<media-directory> ...`: the directories containing the media
  files

To continue our above example, we would execute ExifTool as
follows to merge all attributes from the intermediate XMP files
without cleaning up any existing attributes (which is probably
not advisable!):

    exiftool -if '-f "$directory/$filename.wpg2dk.xmp"' \
             -tagsFromFile '%d%F.wpg2dk.xmp'            \
             -q -q -r. /media/archive

As an alternative to the above approach (called "recursive
option"), we can feed the output of `wpg2dk list` to ExifTool to
specify the files it should process (called "list option").
Suppose we have generated the intermediate XMP files like this:

    wpg2dk extract <map-and-other-parameters>

Then we can change that command line to `wpg2dk` command `list`
and create a list of processed media files named `media.list` in
the current working directory:

    wpg2dk list <map-and-other-parameters> > media.list

That file we can then use to drive ExifTool processing, like
this:

    exiftool [<tag-to-clean-up> ...]                    \
             -tagsFromFile '%d%F.wpg2dk.xmp'            \
             [<tag-to-copy-from-xmp> ...]               \
             -q -q -@ - < media.list

where the only ExifTool command line parameter `-@ -` not yet
described directs ExifTool to read the names of the media files
to process from STDIN which, in our case, comes from file
`media.list`.

### Cleaning Up all Tags

`wpg2dk` uses tags to provide information on people tags,
geotags, and regular, descriptive tags.  It is somewhat difficult
to clean up each of these groups separately, so I decided to add
an initial preparation step that simply cleans up all possible
existing tags from the media files.

Specify the following parameter to ExifTool to achieve that
(recursive option):

    exiftool -if '-f "$directory/$filename.wpg2dk.xmp"'                \
             -IFD0:XPKeywords=              -IPTC:Keywords=            \
             -XMP-acdsee:Categories=        -XMP-dc:Subject=           \
             -XMP-lr:HierarchicalSubject=   -XMP-mediapro:CatalogSets= \
             -XMP-microsoft:LastKeywordXMP= -XMP-digiKam:TagsList=     \
             -m -IFD0:ResolutionUnit=inches -r. <media-directory> ...

The parameters `-m -IFD0:ResolutionUnit=inches` used above are
optional.  They direct Exiftool to ignore and possibly fix minor
issues (`-m`) and, for that option to have a lasting effect, to
force-rewrite all media files (`-IFD0:ResolutionUnit=inches`).
Fixing these minor issues during the tag clean-up has (hopefully)
the beneficial effect that ExifTool will not bother you with them
during the following steps.

Using parameter `-m` is not always the best option to get rid of
warnings.  For example, if it is used when copying people tags or
geotags, it may result in empty strings written as tags.  In
these cases, one would better use the previously described
parameters `-q -q`, which only suppresses display of warnings,
without having other side-effects.

For the list option use the following command:

    exiftool -IFD0:XPKeywords=              -IPTC:Keywords=            \
             -XMP-acdsee:Categories=        -XMP-dc:Subject=           \
             -XMP-lr:HierarchicalSubject=   -XMP-mediapro:CatalogSets= \
             -XMP-microsoft:LastKeywordXMP= -XMP-digiKam:TagsList=     \
             -m -IFD0:ResolutionUnit=inches -@ - < media.list

### Merging People Tags

`wpg2dk` converts WPG's people tags to:

- face region information
- people tags starting with prefix `People`
- optional people completeness color label, if command line
  parameter `-pccl` is specified

To merge all of these, specify the following parameter to
ExifTool (recursive option):

    exiftool -if '-f "$directory/$filename.wpg2dk.xmp"' \
             -XMP-MP:RegionInfoMP=                      \
             -XMP-mwg-rs:RegionInfo=                    \
             -XMP-digiKam:ColorLabel=                   \
             -XMP-xmp:Label=                            \
             -tagsFromFile '%d%F.wpg2dk.xmp'            \
             -XMP-MP:RegionInfoMP                       \
             -XMP-digiKam:ColorLabel                    \
             -separator $'\001'                         \
             '-XMP-digiKam:TagsList+<${XMP-digiKam:TagsList@;
              $_ = undef unless m{^People/}}'           \
             -q -q -r. <media-directory> ...

If you do not use that dubious people completeness color label,
you can omit the parameters related to attributes
`XMP-digiKam:ColorLabel` and `XMP-xmp:Label` from the above
command line.

Command line parameter `-separator $'\001'` (ASCII character 001,
Control-A) and the lengthy `-XMP-digiKam:TagsList+<...` parameter
following it is required so that we can filter the
`XMP-digiKam:TagsList` attribute element-wise.  The parameter
uses ExifTool's advanced formatting feature to remove (`$_ =
undef`) those elements that are not people tags (`unless
m{^People/}`).

For the list option use the following command:

    exiftool -XMP-MP:RegionInfoMP=                      \
             -XMP-mwg-rs:RegionInfo=                    \
             -XMP-digiKam:ColorLabel=                   \
             -XMP-xmp:Label=                            \
             -addTagsfromfile '%d%F.wpg2dk.xmp'         \
             -XMP-MP:RegionInfoMP                       \
             -XMP-digiKam:ColorLabel                    \
             -separator $'\001'                         \
             '-XMP-digiKam:TagsList+<${XMP-digiKam:TagsList@;
              $_ = undef unless m{^People/}}'           \
             -q -q -@ - < media.list

### Merging Geotags

`wpg2dk` converts WPG's geotags to:

- geotags starting with the prefix provided by command line
  parameter `-geotagroot`, by default `Location`
- GPS coordinates of the most specific geotag

Unfortunately, we have to process these in two separate ExifTool
calls, one for the geotags and one for the GPS coordinates.

Below command lines for merging the geotags assume that you have
used the default `Location` as geotag root tag.  If you have used
a different root tag, you must change the reference to it in
below command line accordingly.  Use the following ExifTool
parameters for the recursive option:

    exiftool -if '-f "$directory/$filename.wpg2dk.xmp"' \
             -tagsFromFile '%d%F.wpg2dk.xmp'            \
             -separator $'\001'                         \
             '-XMP-digiKam:TagsList+<${XMP-digiKam:TagsList@;
              $_ = undef unless m{^Location/}}'         \
             -q -q -r. <media-directory> ...

And these for the list option:

    exiftool -tagsFromFile '%d%F.wpg2dk.xmp'            \
             -separator $'\001'                         \
             '-XMP-digiKam:TagsList+<${XMP-digiKam:TagsList@;
              $_ = undef unless m{^Location/}}'         \
             -q -q -@ - < media.list

Since the GPS coordinates determined from WPG's geotags are
rather coarse, any existing GPS coordinates should not be
overwritten with these.  So we need another condition to avoid
touching any media files that have any GPS metadata attributes
set already, like this:

    -if '! %{$self->GetInfo("GPS*")}'

As an additional complexity, digiKam versions older than 7.6.0
write bogus attributes `XMP-exif:GPSLatitudeRef` and
`XMP-exif:GPSLongitudeRef` to media files (see [digiKam bug
450982](https://bugs.kde.org/show_bug.cgi?id=450982)).  ExifTool
does not support these out of the box, even not for removal.  To
get rid of these with ExifTool one needs an additional
configuration file that declares them as custom attributes:

    %Image::ExifTool::UserDefined = (
      'Image::ExifTool::XMP::exif' => {
        GPSLatitudeRef  => {},
        GPSLongitudeRef => {},
      },
    );
    1;

Save above lines to a file, say, `xmp-exif-gpsref.pm` that we
will reference in the command lines below.

All combined we get for the recursive option:

    exiftool -config xmp-exif-gpsref.pm                 \
             -if '-f "$directory/$filename.wpg2dk.xmp"' \
             -if '! %{$self->GetInfo("GPS*")}'          \
             -XMP-exif:GPSLatitudeRef=                  \
             -XMP-exif:GPSLongitudeRef=                 \
             -tagsFromFile '%d%F.wpg2dk.xmp'            \
             -XMP-exif:GPSL*itude                       \
             -q -q -r. <media-directory> ...

And for the list option:

    exiftool -config xmp-exif-gpsref.pm                 \
             -if '! %{$self->GetInfo("GPS*")}'          \
             -XMP-exif:GPSLatitudeRef=                  \
             -XMP-exif:GPSLongitudeRef=                 \
             -tagsFromFile '%d%F.wpg2dk.xmp'            \
             -XMP-exif:GPSL*itude                       \
             -q -q -@ - < media.list

### Merging Captions

Lots of attributes to clean up, one to merge (recursive option):

    exiftool -if '-f "$directory/$filename.wpg2dk.xmp"' \
             -IFD0:ImageDescription=                    \
             -IFD0:XPTitle=                             \
             -XMP-dc:Description=                       \
             -XMP-acdsee:Caption=                       \
             -XMP-dc:Title=                             \
             -IPTC:ObjectName=                          \
             -tagsFromFile '%d%F.wpg2dk.xmp'            \
             -XMP-dc:Title                              \
             -q -q -r. <media-directory> ...

List option:

    exiftool -IFD0:ImageDescription=                    \
             -IFD0:XPTitle=                             \
             -XMP-dc:Description=                       \
             -XMP-acdsee:Caption=                       \
             -XMP-dc:Title=                             \
             -IPTC:ObjectName=                          \
             -tagsFromFile '%d%F.wpg2dk.xmp'            \
             -XMP-dc:Title                              \
             -q -q -@ - < media.list

### Merging Descriptive Tags

The following command lines assume that you have used the default
`Location` as geotag root tag.  If you have used a different root
tag, you must change the reference to it in below command line
accordingly.

Since we have cleaned up all tags already in the first step, we
only need to merge the non-people-tag, non-geotags (recursive
option):

    exiftool -tagsFromFile '%d%F.wpg2dk.xmp'            \
             -separator $'\001'                         \
             '-XMP-digiKam:TagsList+<${XMP-digiKam:TagsList@;
              $_ = undef if m{^(People|Location)/}}'    \
             -q -q -r. <media-directory> ...

List option:

    exiftool -addTagsFromFile '%d%F.wpg2dk.xmp'         \
             -separator $'\001'                         \
             '-XMP-digiKam:TagsList+<${XMP-digiKam:TagsList@;
              $_ = undef if m{^(People|Location)/}}'    \
             -q -q -@ - < media.list

### Merging Ratings

This one is a bit simpler (recursive option):

    exiftool -if '-f "$directory/$filename.wpg2dk.xmp"' \
             -IFD0:Rating=                              \
             -IFD0:RatingPercent=                       \
             -XMP-acdsee:Rating=                        \
             -XMP-microsoft:RatingPercent=              \
             -XMP-xmp:Rating=                           \
             -tagsFromFile '%d%F.wpg2dk.xmp'            \
             -XMP-xmp:Rating                            \
             -q -q -r. <media-directory> ...

List option:

    exiftool -IFD0:Rating=                              \
             -IFD0:RatingPercent=                       \
             -XMP-acdsee:Rating=                        \
             -XMP-microsoft:RatingPercent=              \
             -XMP-xmp:Rating=                           \
             -tagsFromFile '%d%F.wpg2dk.xmp'            \
             -XMP-xmp:Rating                            \
             -q -q -@ - < media.list

### Merging Flags

And this one even more simple (recursive option):

    exiftool -if '-f "$directory/$filename.wpg2dk.xmp"' \
             -XMP-digiKam:PickLabel=                    \
             -tagsFromFile '%d%F.wpg2dk.xmp'            \
             -XMP-digiKam:PickLabel                     \
             -q -q -r. <media-directory> ...

List option:

    exiftool -XMP-digiKam:PickLabel=                    \
             -addTagsfromfile '%d%F.wpg2dk.xmp'         \
             -XMP-digiKam:PickLabel                     \
             -q -q -@ - < media.list

### Undoing or Comitting ExifTool's Changes

ExifTool saves copies of the original media files (with suffix
`_original`) before it modifies their metadata. But it saves them
only when it modifies them for the first time.  So when you
inject metadata in multiple steps as described in the preceding
sections, ExifTool saves only the state of the media files before
the first modifying step.

You can restore the original media files with the following
command (recursive option):

    exiftool -restore_original \
             -q -q -r. <media-directory> ...

For the list option use the following command:

    exiftool -restore_original \
             -q -q -@ - < media.list

When you are sure that you do not need the original media files
any longer, you can remove them with (recursive option):

    exiftool -delete_original \
             -q -q -r. <media-directory> ...

or (list option):

    exiftool -delete_original \
             -q -q -@ - < media.list

## Reading the Image Metadata with digiKam

If you start digiKam with the media files prepared in the
previous section *and* if digiKam has not yet seen these media
files before, then digiKam should read their metadata and store
it in its own metadata database.  Since we have tried hard to
avoid metadata inconsistencies by using only a limited set of
metadata attributes in the media files, digiKam's metadata should
be likewise consistent.

The problems start if you have, for example, processed the media
files already with digiKam *before* injecting the WPG metadata as
described above.  Depending on your digiKam configuration, one of
the following can happen then:

- digiKam does not notice and pick up the injected metadata at
  all,

- digiKam picks up the injected metadata from the media files and
  overwrites completely what it has in its own database, or

- digiKam tries to merge the injected metadata with what it has
  in its own database.

None of the above options really seems desirable and so I will
not go into the details on how to configure digiKam properly to
achieve any of the above.  User Sdx has some material on this in
his tutorial [Digikam/Tutorials/Setup of digiKam for Windows
compatibility](https://userbase.kde.org/Digikam/Tutorials/Setup_of_digiKam_for_Windows_compatibility).

Instead, I describe in the next section what minimum actions and
configuration are necessary in a freshly started, unconfigured
digiKam with freshly prepared media files.  If you have used
digiKam previously or during the recommended test phase of the
migration, you can reset it to an unconfigured state as described
in [this section](#resetting-digikam-to-an-unconfigured-state).

### Minimum digiKam Configuration

The following describes a mix a my preferences and

1. During the initial digiKam set-up wizard, leave everything at
   default and `Next` through.

2. Select to download the binary files required for the face
   engine and the red eye removal tool.

3. In menu `View` ensure `Include Album Sub-Tree` and `Include
   Tag Sub-Tree` are both check-marked.

That's it for the minimum.

### Advanced Face-Recognition Configuration

You open digiKam's face recognition panels in the left side bar,
tab `People`.  In the `Settings` tab of the lower left panel,
select `Use YOLO v3 detection model` and `Work on all process
cores`.

Expand the `People` tree on the upper left panel.  For the known
people listed there, you can select their names.  From the
thumbnails that show up in the right panel, you can select one as
tag thumbnail by right-clicking it and selecting `Set as Tag
Thumbnail`.

To train digiKam's face recognition from the face regions you
have marked already in WPG, go to `Tools` &rarr; `Maintenance
...`.  There expand `Detect and recognize faces`, select it, and
select `Clear and rebuild all training data`.  Select `OK` and
wait for the rebuild to complete.

Rectangles that WPG has detected already as faces but which have
not yet been assigned people in WPG may now be recognized by
digiKam and moved from category `Unknown` to `Unconfirmed` in the
`People` tree on the upper left panel.

The face detection done by digiKam can be rather time and
resource consuming, much more than usually on WPG (at least for
something WPG should be better).  In addition, I have not found
any documentation on how digiKam's face recognition handles
existing face rectangles detected, for example, by WPG.

Likewise, I have not found documentation what it means precisely
to "skip images already scanned" when detecting faces, as one can
select in the `Workflow` tab of the lower left face recognition
panel.  A bit of reverse engineering showed, however, that
digiKam uses an internal tag named
`_Digikam_Internal_Tags_/Scanned for Faces` to mark media files
which it has scanned for face rectangles.

Since the tag is internal, digiKam does not *export* it to media
files.  But it *imports* that tag happily from media files, if
present, which can be used to exempt media files that already
have been scanned completely by WPG.

This section is rather (incomplete).

- exiftool -XMP-digiKam:TagsList+='_Digikam_Internal_Tags_/Scanned for Faces' *.jpg

- `Settings` &rarr; `Configure digiKam` &rarr; `Metadata` &rarr;
  `Behavior` &rarr; `Rescan file when files are modified`

- <album> -> right button -> Refresh

As soon as digiKam has picked up the `Scanned for Faces` tag, you
should remove it from the media files again, like this:

    exiftool -XMP-digiKam:TagsList-='_Digikam_Internal_Tags_/Scanned for Faces' *.jpg

### Resetting digiKam to an Unconfigured State

If you have used digiKam previously or during the recommended
test phase of the migration, you can reset it to an unconfigured
state as follows:

1. Find out the location of the database files in digiKam.  To do
   so, navigate to `Settings` &rarr; `Configure digiKam` &rarr;
   `Database Settings`.  If you use the default database type
   `SQLite` (and only this case I cover here), the location of
   the database files should be mentioned at the bottom of the
   configuration panel.

2. Exit digiKam.

3. In the directory determined above, remove or rename all files
   with extension `.db`.  In my installation, there are four of
   these: `digikam4.db`, `recognition.db`, `similarity.db`, and
   `thumbnails-digikam.db`.

4. Restart digiKam.  It might present a dialog that its SQLite
   core database was not found and that, to reset the
   configuration, you need to remove the `digikamrc` file.  Take
   note of that file.  In my installation, it is named
   `~/.var/app/org.kde.digikam/config/digikamrc`.

5. OK the dialog, cancel the next one, if any, remove the file
   from the previous step and restart digiKam again.

Of course, you loose all previous digiKam configuration and
metadata when doing as described above, so beware.

## `wpg2dk` Command Line Reference

This section is almost completely incomplete.  Refer to the
source of `wpg2dk` and to above tutorial for the time being.

For easier exchange of commands on the command line, all commands
accept all parameters, even if the parameters do not have any
effect on the command.  For example, almost all XMP-related
parameters have an effect only on command `extract`.

### `wpg2dk` Media Object Errors and Warnings

`wpg2dk` issues errors and warnings related to specific media
objects with the IDs listed below.  Message IDs not explicitly
marked as "warning" are errors.

- `invalid_wpgmd` (warning): The WPG metadata is invalid, for
  example, due to WPG metadata database inconsistencies.
  `wpg2dk` tries to recover from such issues as good as possible,
  but regardless of that it is probably advisable to dump
  affected media objects with this warning to understand what
  could be wrong.

- `no_wpgmd`: There is no WPG metadata (as listed in section
  [Converting the WPG Metadata](#converting-the-wpg-metadata))
  stored in the WPG metadata database.

- `empty_file`: The media file map (command line parameters
  `--mmap`) maps to an empty string.

- `unmapped_file`: There is no media file map or, more precisely,
  the specified media file map does not change the pseudo file
  name derived from the WPG metadata.

- `missing_file`: The media file map maps to an absent media
  file.

- `duplicate_file`: The media file map maps to a media file name
  that has been seen already for some different media object.

- `empty_xfile`: The XMP file map (command line parameters
  `--xmap`) maps to an empty string.

- `unmapped_xfile`: There is no XMP file map or, more precisely,
  the specified XMP file map does not change the initial media
  file name.

- `missing_xdir`: The XMP file map maps to an XMP file the
  directory of which is absent.  Use command line parameter
  `--mkdir` to avoid this error and let `wpg2dk` create all
  required directories automatically.

- `existing_xfile`: The XMP file map maps to an existing XMP
  file.  Use command line parameter `--force` to avoid this error
  and let `wpg2dk` overwrite that XMP file.

- `duplicate_xfile`: The XMP file map maps to a XMP file name
  that has been seen already for some different media object.

- `exiftool_warnings` (warning): ExifTool has reported one or
  more warnings when accessing the media file.

- `exiftool_errors`: ExifTool has reported one or more errors
  when accessing the media file.

## Background and Version Information

- I considered using Shotwell (Gnome) or digiKam (KDE) as
  migration targets.  The former has a cleaner UI, the latter
  much more features.  Selected digiKam, hence.

- Newer digiKam is better, in particular with respect to face
  detection and recognition.  My Debian GNU/Linux 11 came with
  digiKam 7.1, so I installed the `org.kde.digikam` Flatpak from
  Flathub with digiKam 7.4.0.  Later I installed digiKam 7.6.0
  without the need to change anything in the previously written
  documentation.

- On my particular setup the digiKam Flatpak has some severe
  display problems on Wayland.  All fine on X11 or if called with
  `QT_QPA_PLATFORM=xcb`.

- My WPG version is "Windows Photo Gallery 2012", build
  16.4.3528.331 running on Windows 8.1.

- My ExifTool version is 12.16.

## Technical Details

The following rather technical sections describe in more details
how `wpg2dk` extracts the WPG metadata, converts it, and writes
it to the intermediate XMP files:

- [Media File Path](#media-file-path)
- [People tags](#people-tags)
- [Geotag](#geotag)
- [Caption](#caption)
- [Descriptive tags](#descriptive-tags)
- [Rating](#rating)
- [Flag](#flag)

Unfortunately, the WPG database does not seem to define any
constraints on the contained tables (or they get lost somewhere
during the conversion process).  Accordingly, `wpg2dk` does not
rely on any assumptions when reading the WPG metadata from the
database and in particular does not really use the nice SQL joins
described below.

The following sections provide information on what metadata
attributes WPG, digiKam, and `wpg2dk` process and how.  The
sections provide the following information per metadata
attribute:

| Item           | Meaning                                                       |
|:---------------|:--------------------------------------------------------------|
| Source         | (Pseudo-)SQL to select the attribute(s) from the WPG database |
| Type           | Type information on the results returned by the SQL           |
| Target         | XMP snippet create by `wpg2dk` for the attribute(s)           |
| Notes          | Comment(s) on the attribute(s)                                |
| WPG Attributes | Attribute names used by WPG to store the attribute(s)         |
| DK Attributes  | Attribute names used by digiKam to store the attribute(s)     |

### Media File Path

*Source*:

    -- compose complete path from volume, path, and file name
    select    v.volumeid, p.path, o.filename
    from      tblobject o
    left join tblpath   p on o.filepathid = p.pathid
    left join tblvolume v on p.volumeid   = v.volumeid
    where     o.objectid = ...

### People Tags

*Source*:

    select    r.personid, p.name,
              r.left, r.top, r.width, r.height
    from      tblregion r
    left join tblperson p on r.personid = p.personid
    where     r.objectid = ...

    select (syncstatus & 2048) = 2048 from tblobject where objectid = ...

*Type*: non-negative integer (`r.personid`), UTF-8 encoded string
(`p.name`), float in range [0.0, 1.0) (region dimensions), zero
or one (`syncstatus`)

*Target*:

    <rdf:Description rdf:about=''
     xmlns:MP='http://ns.microsoft.com/photo/1.2/'
     xmlns:MPRI='http://ns.microsoft.com/photo/1.2/t/RegionInfo#'
     xmlns:MPReg='http://ns.microsoft.com/photo/1.2/t/Region#'>
     <MP:RegionInfo rdf:parseType='Resource'>
      <MPRI:Regions>
       <rdf:Bag>
        <!-- for known persons -->
        <rdf:li rdf:parseType='Resource'>
         <MPReg:PersonDisplayName>[person-name]</MPReg:PersonDisplayName>
         <MPReg:Rectangle>[left], [top], [width], [height]</MPReg:Rectangle>
        </rdf:li>
        <!-- for unknown persons -->
        <rdf:li rdf:parseType='Resource'>
         <MPReg:Rectangle>[left], [top], [width], [height]</MPReg:Rectangle>
        </rdf:li>
       </rdf:Bag>
      </MPRI:Regions>
     </MP:RegionInfo>
    </rdf:Description>

    <rdf:Description rdf:about=''
     xmlns:digiKam='http://www.digikam.org/ns/1.0/'>
     <digiKam:TagsList>
      <rdf:Seq>
        <!-- for known persons, both attached and not attached to
             a face region -->
       <rdf:li>People/[person-name]</rdf:li>
      </rdf:Seq>
     </digiKam:TagsList>
    </rdf:Description>

    <rdf:Description rdf:about=''
     xmlns:digiKam='http://www.digikam.org/ns/1.0/'>
     <digiKam:ColorLabel>[people-completeness-color-label]</digiKam:ColorLabel>
    </rdf:Description>

*Notes*:

- Table `tblregion` provides face regions in correct orientation
  for non-rotated photos, that is, where `IFD0:Orientation` is
  absent or denotes "normal" orientation.  For these, `r.left`
  gives distance between right photo margin and left region
  margin, `r.top` likewise for the top region margin.

- For photos with non-normal EXIF orientation, table `tblregion`
  provides face regions *after* applying the rotation.  In other
  words, a photo will always have identical (up to rounding) face
  regions in table `tblregion`, regardless of its EXIF
  orientation.

- In contrast to that, the `MP:RegionInfo` tag used as target
  provides the region *before* applying the rotation.  Values
  `<left>`, `<top>`, etc., are specified with a precision of 6
  places after the decimal point.

- `r.personid` for a face region equals zero if and only if that
  face has not been tagged yet ("unknown person").

- All four region parameters equal zero for people tags that are
  not attached to a face region, but rather to the whole photo.

- It seems that bit 12 of column `synctstatus` of table
  `tblobject` flips to one when all faces detected by WPG have
  been tagged or ignored for some photo.  If you specify command
  line option `-pccl <people-completeness-color-label>`, `wpg2dk`
  adds the specified color label (an integer from zero to nine)
  to the intermediate XMP of media files having that bit set.

*WPG Attributes*:

`XMP-MP:RegionInfoMP`

*DK Attributes*:

`XMP-MP:RegionInfoMP`
`XMP-mwg-rs:RegionInfo`
(for the region information)

`XMP-digiKam:ColorLabel`
`XMP-xmp:Label`
(for the color label information)

plus attributes mentioned in [Descriptive
Tags](#descriptive-tags) (for the `People` people tags)

### Geotag

*Source*:

    select    u.locationid, l.locationname,
              l.locationlat, l.locationlong
    from      tblocationusage u         -- [sic!]
    left join tbllocation l on u.locationid = l.locationid
    where     u.objectid = ...

*Type*: integer (`u.locationid`), UTF-8 encoded string
(`l.locationname`), float (all others)

*Target*:

For `l.locationlat`, `l.locationlong`:

    <rdf:Description rdf:about=''
     xmlns:exif='http://ns.adobe.com/exif/1.0/'>
     <exif:GPSLatitude>[latdegs,latdec.mins][latdir]</exif:GPSLatitude>
     <exif:GPSLongitude>[longdegs,longdec.mins][longdir]</exif:GPSLongitude>
    </rdf:Description>

For `l.locationname`:

    <rdf:Description rdf:about=''
     xmlns:digiKam='http://www.digikam.org/ns/1.0/'>
     <digiKam:TagsList>
      <rdf:Seq>
       <rdf:li>[geotag-root]/[geotag-node-0]</rdf:li>
       <rdf:li>[geotag-root]/[geotag-node-0]/[geotag-node-1]</rdf:li>
       <rdf:li>[geotag-root]/[geotag-node-0]/[geotag-node-1]/[geotag-leaf]</rdf:li>
      </rdf:Seq>
     </digiKam:TagsList>
    </rdf:Description>

*Notes*:

- Table `tbllocation` does not have any separate information on
  the direction of the latitude ("South" or "North") and
  longitude ("West" or "East").  I guess that the sign of columns
  `locationlat` and `locationlong` provide that information, but
  have too little information to verify that, in particular since
  one cannot geotag in WPG any longer.

- `l.locationname` selected in the source above is only the "leaf
  location" in a location tree.  The structure of that is
  provided by the parent - child (`locationparentid` -
  `locationid`) relations in table `tbllocation`.

- You can configure the geotag root tag (defaulting to
  `Location`) with command line parameter
  `-geotagroot <geotag-root>`.

- You can control the structure of the generated geotags with
  command line option `-geotags`.  Suppose you have a geotag with
  path `country/state/city` in your WPG metadata.  Then you can
  select to get the following geotags added to the intermediate
  XMP file:

  - `-geotags path` (default)

      ```
      <geotag-root>/country/state/city
      ```

  - `-geotags rec`

      ```
      <geotag-root>/country
      <geotag-root>/country/state
      <geotag-root>/country/state/city
      ```

  - `-geotags nodes`

      ```
      <geotag-root>/country
      <geotag-root>/state
      <geotag-root>/city
      ```

  - `-geotags leaf`

      ```
      <geotag-root>/city
      ```

*WPG Attributes*:

`XMP-iptcExt:LocationCreated`

*DK Attributes*:

`GPS:GPSLatitudeRef`
`GPS:GPSLatitude`
`GPS:GPSLongitudeRef`
`GPS:GPSLongitude`
`XMP-exif:GPSLatitudeRef`
`XMP-exif:GPSLatitude`
`XMP-exif:GPSLongitudeRef`
`XMP-exif:GPSLongitude`
(for the GPS coordinates.  The reference attributes in the
`XMP-exif` group were actually written on error by digiKam
versions older than 7.6.0, see [digiKam bug
450982](https://bugs.kde.org/show_bug.cgi?id=450982).)

plus attributes mentioned in [Descriptive
Tags](#descriptive-tags) (for the `Location` geotags)

### Caption

*Source*: `select title from tblobject where objectid = ...`

*Type*:   UTF-8 encoded string

*Target:*

    <rdf:Description rdf:about=''
     xmlns:dc='http://purl.org/dc/elements/1.1/'>
     <dc:title>
      <rdf:Alt>
       <rdf:li xml:lang='x-default'>[title]</rdf:li>
      </rdf:Alt>
     </dc:title>
    </rdf:Description>

*Notes*:

- `wpg2dk` adds a title to the intermediate XMP only if it is
  non-empty.

*WPG Attributes*:

`IFD0:ImageDescription`
`IFD0:XPTitle`
`XMP-dc:Description`
`XMP-dc:Title`

*DK Attributes*:

`IPTC:ObjectName`
`XMP-acdsee:Caption`
`XMP-dc:Title`

### Descriptive Tags

*Source*:

    select    u.labelid, l.labelname
    from      tbllabelusage u
    left join tbllabel l on u.labelid = l.labelid
    where     u.objectid = ...

*Type*: integer (`u.labelid`), non-empty UTF-8 encoded string
not containing any slashes (`l.labelname`)

*Target*:

    <rdf:Description rdf:about=''
     xmlns:digiKam='http://www.digikam.org/ns/1.0/'>
     <digiKam:TagsList>
      <rdf:Seq>
       <rdf:li>[tag-node-0]</rdf:li>
       <rdf:li>[tag-node-0]/[tag-node-1]</rdf:li>
       <rdf:li>[tag-node-0]/[tag-node-1]/[tag-leaf]</rdf:li>
      </rdf:Seq>
     </digiKam:TagsList>
    </rdf:Description>

*Notes*:

- `l.labelname` selected in the source above is only the "leaf
  tag" in a tag tree.  The structure of that is provided by the
  parent - child (`parentlabelid` - `labelid`) relations in table
  `tbllabel`.

- You can control the structure of the generated tags with
  command line option `-tags`.  Suppose you have a tag with path
  `a/b/c` in your WPG metadata.  Then you can select to get the
  following tags added to the intermediate XMP file:

  - `-tags path` (default)

      ```
      a/b/c
      ```

  - `-tags rec`

      ```
      a
      a/b
      a/b/c
      ```

  - `-tags nodes`

      ```
      a
      b
      c
      ```

  - `-tags leaf`

      ```
      c
      ```

*WPG Attributes*:

`IFD0:XPKeywords`
`XMP-dc:Subject`
`XMP-microsoft:LastKeywordXMP`

*DK Attributes*:

`IPTC:Keywords`
`XMP-acdsee:Categories`
`XMP-dc:Subject`
`XMP-digiKam:TagsList`
`XMP-lr:HierarchicalSubject`
`XMP-mediapro:CatalogSets`
`XMP-microsoft:LastKeywordXMP`

### Rating

*Source*: `select rating from tblobject where objectid = ...`

*Type*:   integer ranging from zero (not rated) to five (max.
          rating)

*Target*:

    <rdf:Description rdf:about=''
     xmlns:xmp='http://ns.adobe.com/xap/1.0/'>
     <xmp:Rating>[rating]</xmp:Rating>
    </rdf:Description>

*Notes*:

- `wpg2dk` adds a rating to the intermediate XMP only if the
  source rating is larger than zero.

*WPG Attributes*:

`IFD0:Rating`
`IFD0:RatingPercent`
`XMP-microsoft:RatingPercent`
`XMP-xmp:Rating`

*DK Attributes*:

`IFD0:Rating`
`IFD0:RatingPercent`
`XMP-acdsee:Rating`
`XMP-microsoft:RatingPercent`
`XMP-xmp:Rating`

### Flag

*Source*: `select flagged from tblobject where objectid = ...`

*Type*:   boolean

*Target*:

    <rdf:Description rdf:about=''
     xmlns:digiKam='http://www.digikam.org/ns/1.0/'>
     <digiKam:PickLabel>[pick-label]</digiKam:PickLabel>
    </rdf:Description>

*Notes*:

- Table `tblobject` also provides a column `everflagged` which
  equals `1` if the media has ever been flagged, otherwise
  `NULL`.  However, that attribute is not extracted by `wpg2dk`.

- You can configure the pick label (defaulting to 3, "Accepted")
  via command line parameter `-pl <pick-label>`.

*WPG Attributes*:

not available

*DK Attributes*:

`XMP-digiKam:PickLabel`
