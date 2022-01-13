# wpg2dk - Migrate from Windows Photo Gallery to digiKam

## Overview

Windows Photo Gallery (WPG) aka. Windows Live Photo Gallery
(WLPG) has been desupported by Microsoft.  To avoid similar
trouble with other proprietary products in the future, I decided
to migrate our photo collection to digiKam running on GNU/Linux.

Some initial tests showed that some of the metadata maintained by
WPG is already used by digiKam, some is not, and some is broken
(by WPG, not digiKam).  More tests showed that the most complete
source of WPG metadata is actually the WPG metadata database
itself, and not the metadata stored by WPG in the media files.

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

3.  Inject the metadata contained in the intermediate XMP files
    into the media files with ExifTool.

    See section [Injecting the WPG
    Metadata](#injecting-the-wpg-metadata).

4.  Reread the image metadata with digiKam.

This project has been developed and tested on "family photo
collection scope".  Most of the media consisted of JPEGs.

For the WPG and digiKam version, see section section [Background
and Version Information](#background-and-version-information).

(incomplete) What about non-JPEGs and movies?

## Why Using ExifTool to Inject the WPG Metadata?

Originally, I planned to create complete XMP sidecar files
containing the merged metadata from both the media files and the
WPG database and let digiKam pick up these sidecar files.

However, it soon became clear that merging media metadata is a
highly non-trivial process, so I decided to better delegate that
to ExifTool.  (incomplete) Use ExifTool's featues to select tags,
files to process, etc.

To avoid confusion: "Tags" in WPG and digiKam denote descriptive
attributes that can be attached to media files, like geotags or
people tags.  In ExifTool, "tags" denote the smallest unit of
metadata that can be processed.  A metadata unit that contains a
WPG tag is called "keyword" in ExifTool.

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
    wpgsqltimpdir="<absolute-path-to-a-new-directory>"
    mkdir "$wpgsqltimpdir"
    ```

5.  Copy the WPG metadata SQL files to directory
    `$wpgsqltimpdir`.

    Currently, `wpg2dk` only requires the WPG metadata from
    `Pictures*.sql` as input.  But it is probably a good idea to
    keep the data of all metadata files in a format that stays
    readable even when SQL CE database support ceases completely.

6.  Create the SQLite databases corresponding to the WPG metadata
    and import the SQL files:

    ```
    for wpgmddbid in "FaceExemplars" "FaceThumbs" "Pictures"; do
      for sqlfile in "$wpgsqltimpdir/$wpgmddbid"*.sql; do
        sqlite3 "$wpgsqltimpdir/$wpgmddbid.db" ".read '$sqlfile'"
      done
    done
    ```

    This should result in three files

    ```
    FaceExemplars.db
    FaceThumbs.db
    Pictures.db
    ```

    created in directory `$wpgsqltimpdir`.

## Converting the WPG Metadata

Generally, `wpg2dk` extracts only metadata from the WPG metadata
database which can be assigned in the right hand pane of WPG,
which are:

- People tags
- Geotag
- Caption
- Descriptive tags
- Rating
- Flag

The general idea is that `wpg2dk` always extracts all metadata
from the WPG metadata.  As explained in the next section, the
rich command line interface of ExifTool can then be used to
control what parts of that metadata are actually injected into
the media files.

You can select by their file path which media files should be
processed by `wpg2dk`.

The following rather technical sections describe in more details
how the WPG metadata is extacted, converted, and written to the
intermediate XMP files:

- [Media File Path](#media-file-path)
- [People tags](#people-tags)
- [Geotag](#geotag)
- [Caption](#caption)
- [Descriptive tags](#descriptive-tags)
- [Rating](#rating)
- [Flag](#flag)

### Media File Path

*Source*:

    -- compose complete path from volume, path, and file name
    select    v.label, p.path, o.filename
    from      tblobject o
    left join tblpath p   on o.filepathid = p.pathid
    left join tblvolume v on p.volumeid   = v.volumeid
    where     o.objectid = ...

*Notes*:

- `wpg2dk` expects the media files exactly where specified by the
  WPG metadata database.  And it places the intermediate XMP
  files with extension `.xmp` alongside with the original media
  files.

- When executing `wpg2dk` with action `extract` you must specify
  the location of all volumes referenced in the WPG metadata of
  the processed media files on the command line with parameter
  `-volmap`.

- To get an idea what volumes are used and what their path should
  be, run `wpg2dk` with action `list`.

### People Tags

*Source*:

    select    r.personid, p.name,
              r.left, r.top, r.width, r.height
    from      tblregion r
    left join tblperson p on r.personid = p.personid
    where     r.objectid = ...

    select (syncstatus & 2048) = 2048 from tblobject where objectid = ...

*Type*: non-negative integer (`r.personid`), UTF-8 encoded string
(`p.name`), float in range [0.0, 1.0) (all others)

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
  for non-rotated photos, that is, where `EXIF:Orientation` is
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

- All four region parameteres equal zero for people tags that are
  not attached to a face region, but rather to the whole photo.

- It seems that bit 12 of column `synctstatus` of table
  `tblobject` flips to one when all faces detected by WPG have
  been tagged or ignored for some photo.  If you specify command
  line option `-pccl <people-completeness-color-label>`, `wpg2dk`
  adds the specified color label (an integer from zero to nine)
  to the intermediate XMP of media files having that bit set.

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
     <exif:GPSLatitude>[latitude,in.dms]N</exif:GPSLatitude>
     <exif:GPSLongitude>[longitude,in.dms]E</exif:GPSLongitude>
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

- `l.locationname` selected in the source above is only the "leaf
  location" in a location tree.  The structure of that is
  provided by the parent - child (`locationparentid` -
  `locationid`) relations in table `tbllocation`.

- You can configure the geotag root tag (defaulting to
  `Location`) with command line parameter
  `-geotags root:<geotag-root>`.

- You can control the structure of the generated geotags with
  command line option `-geotags`.  Suppose you have a geotag with
  path `country/state/city` in your WPG metadata.  Then you can
  select to get the following geotags added to the intermediate
  XMP file:

  - `-geotags struct=path` (default)

      ```
      <geotag-root>/country/state/city
      ```

  - `-geotags struct=rec`

      ```
      <geotag-root>/country
      <geotag-root>/country/state
      <geotag-root>/country/state/city
      ```

  - `-geotags struct=nodes`

      ```
      <geotag-root>/country
      <geotag-root>/state
      <geotag-root>/city
      ```

  - `-geotags struct=leaf`

      ```
      <geotag-root>/city
      ```

### Caption

*Source*: `select title from tblobject where objectid = ...`

*Type*:   non-empty UTF-8 encoded string

*Target:*

    <rdf:Description rdf:about=''
     xmlns:dc='http://purl.org/dc/elements/1.1/'>
     <dc:title>
      <rdf:Alt>
       <rdf:li xml:lang='x-default'>[title]</rdf:li>
      </rdf:Alt>
     </dc:title>
    </rdf:Description>

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

- `wpg2dk` always sets an explicit zero level rating to override
  possible other rating tags.

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
  `NULL`.

- You can configure the pick label (defaulting to 3, "Accepted")
  via command line parameter `-pl`.

## Injecting the WPG Metadata

(incomplete) Describe selection of metadata to inject.  As an
added complexity, describe how to inject only certain types of
tags.

    ```
    (incomplete)
    exiftool -tagsFromFile '%:3d%F.xmp' ~/Pictures/IMG_2882.jpg
    ```

An an alternative to the above procedure, you can also inject the
intermediate XMP files into sidecar XMP files written by (and
read from) digiKam, thus leaving your precious media files
unchanged.

## Background and Version Information

- I considered using Shotwell (Gnome) or digiKam (KDE) as
  migration targets.  The former has a cleaner UI, the latter
  much more features.  Selected digiKam, hence.

- Newer digiKam is better, in particular with respect to face
  detection and recognition.  My Debian GNU/Linux 11 came with
  digiKam 7.1, so I installed the `org.kde.digikam` Flatpak from
  Flathub with digiKam 7.4.0.

- On my particular setup the digiKam Flatpak has some severe
  display problems on Wayland.  All fine on X11.

- My WPG version is "Windows Photo Gallery 2012", build
  16.4.3528.331 running on Windows 8.1.

## Development Snippets

- Dump complete (?) image metadata of a JPEG as JSON with

  ```
  exiftool -j -G -struct <jpeg>
  ```
