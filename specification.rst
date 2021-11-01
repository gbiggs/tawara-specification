Tawara format specification
==========================

Copyright 2021 Geoffrey Biggs geoff@openrobotics.org

This work is licensed under the `Creative Commons
Attribution-ShareAlike 4.0 International License`. To view a copy of this
license, visit http://creativecommons.org/licenses/by-sa/4.0/.

.. _`Creative Commons Attribution-ShareAlike 4.0`:
   http://creativecommons.org/licenses/by-sa/4.0/


Introduction
============

Logging of data in robotics is useful for applications including
reproduction of experiments, visualisation of sensor data and offline
system analysis.

The major features of the Tawara format are:

 - Ability to store data type definitions, for long-term compatibility.

 - Flexibility of data format. There are minimal restrictions on the
   serialised data that can be stored in a Tawara file.

 - Space to store source information, such as the properties of the
   device or connection that provided the data.

 - Index data for efficient seeking within the data.

 - Efficiency for streaming, both when reading and writing.

 - Ability to compress data records, allowing a smaller file size while
   maintaining random-access capability.

Tawara is designed to be a container format for serialised data.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 1 [1]_.


Relationship to Matroska
========================

Tawara is closely based on the Matroska media container format. The
following changes were made:

 - The Video (0x) and Audio (0x) master elements (children of the Track
   element) are not supported. Tawara does not directly support
   media-specific elements; instead, it treats them as it does any other
   codec or serialisation method. These elements are not mandatory in
   Matroska.

 - The TrackType (0x83) element, found in the TrackEntry element of
   Matroska, is modified slightly. The allowable values are:

     0x11:
       subtitle
     0x70:
       data

   These do not conflict with the Matroska track types (the subtitle
   track type is identical).

 - The FlagDefault (0x88) element is not used. It is read, if present,
   to maintain some compatibility with Matroska, but the value is
   ignored.  It is never written (Matroska specifies a default value for
   it).

 - The ContentEncodings element (0x6D80) and all child elements are not
   supported. Content compression and encryption methods are defined by
   the codec used. This element is not mandatory in Matroska.

 - A simplified version of chapters is used. The ChapProcess (0x6944)
   element and all its child elements are not supported, meaning that
   chapters are simply named index points without any additional
   processing. The related ChapterTranslate (0x6924) element and its
   child elements, found in the SegmentInfo element of Matroska, are
   similarly not supported, nor is the TrackTranslate (0x6624) element
   or its children. None of these elements are mandatory in Matroska.

 - Some of the localisation features, such as multiple track languages,
   are not supported:
   - Language (0x22B59C).

   - ChapLanguage (0x437C) and ChapCountry (0x437E) are supported to
     maintain compatibility with Matroska.

   - TagLanguage (0x447A) and TagDefault (0x4484) are supported to
     maintain compatibility with Matroska.

 - The MKV3D features are (for obvious reasons) not supported. These are
   not mandatory in Matroska.
   - TrackCombinePlanes (0xE3) and children.

 - The TargetTypeValue (0x68CA) and TargetType (0x63CA) elements are not
   used. These are not mandatory in Matroska.

 - Not actually a difference from Matroska, but from the EBML RFC draft.
   CRC-32 elemnents are not master elements. They are placed inside a
   Master element, and provide the CRC of *all other* child elements of
   that Master element. This is as specified in the Matroska
   documentation and how libmatroska is implemented, but differs from
   the EBML RFC draft, which states that CRC-32 elements provide the CRC
   of *their* children (which is not actually possible, as an element
   cannot be a Master and hold its own value at the same time).

 - Unlike Matroska, but in keeping with the EBML RFC draft, all strings
   are defined as UTF-8. If only characters in the range 0-127 are
   stored, then the string is equivalent to an ASCII string anyway.


Format
======

A Tawara file is stored in EBML format [2]_. EBML is a binary mark-up format
similar to XML in purpose. It uses binary tags (known as "IDs") and sizes to
provide a flexible, hierarchical document structure.  Refer to the EBML RFC
draft for more details on EMBL. Tawara implementors not using an existing EBML
implementation should in particular pay attention to the encoding of IDs and
size values, which are encoded using a UTF-8-like system, and the fact that
EBML uses network byte order (big endian) with byte-alignment.

File structure
--------------

The diagram below outlines the structure of a Tawara file (not to scale):

  +-------------------------+
  | Header*                 |
  +-------------------------+
  | Segment                 |
  +-+-----------------------+
  | | Metaseek information  |
  | +-----------------------+
  | | Segment information*  |
  | +-----------------------+
  | | Track information*    |
  | +-----------------------+
  | | Chapters              |
  | +-----------------------+
  | | Attachments           |
  | +-----------------------+
  | | Tags                  |
  | +-----------------------+
  | |                       |
  | | Clusters*             |
  | |                       |
  | +-----------------------+
  | | Cueing data           |
  +-+-----------------------+

Elements marked with a "*" are REQUIRED. All elements other than the
Header are children of a Segment element.

Apart from the header element, the ordering of the Level 1 elements in
this way within a segment is OPTIONAL but RECOMMENDED for playback.
Other orderings may be easier for writing but reduce the efficiency of
playback or editing.  More information on the ordering of the elements
is given below_.

.. _below: `Element ordering`_

The file MUST begin with the Header element. This is an EBML-standard
element that provides format identification and meta-data.

Following the Header element is the first Segment element. Typically,
there will only be one Segment element per file, but multiple may be
present, particularly in files that have been edited to combine multiple
source files. Each segment is a stand-alone entity, being linked to
other segments only in the sense of a loose ordering for playback.
Segments MUST NOT be combined in any other way.

The Metaseek information contains a global index of where the other Level 1
elements, such as clusters (usually just the first) and cueing data, are
located in the file. This element is important to the efficiency of initial
file reading, regardless of the order of of the other elements. Without a
Metaseek element, the entire file must be searched to find the other elements.
If a Metaseek element is present in a segment, there MUST only be one.

The Segment information contains the meta-data about the segment in
which it is contained, such as the title and unique ID.

The Track information stores the meta-data for each track that has data
stored in the segment. This may include information such as the track
name (which must be unique within this segment and all linked segments),
the source information and data format.

The Chapters element contains a simplified version of Matroska chapters.
They are essentially index points within the segment.

The Attachment section allows other data to be attached to the file.
This can include virtually anything - even a binary library to
de-serialise and read the data, if portability and security are not
concerns.

The Tagging section stores tags, which allow information such as the
file's author and comments to be stored.

The bulk of the data is contained in the clusters. This is where the
serialised data is stored.

The cueing data is used to store indices for track data in the clusters.
This information is useful for speeding up seeking by reducing the
quantity of data that must be read to find a particular time index.

The diagram below gives a more detailed example of a Tawara file layout.
Note that, as before, the ordering of most elements is optional.

  +-------+------------------+---------------+-----------------+------------------+
  |Level 0|Level 1           |Level 2        |Level 3          |Level 4           |
  +=======+==================+===============+=================+==================+
  |Header |EBMLVersion                                                            |
  |       +-----------------------------------------------------------------------+
  |       |EBMLReadVersion                                                        |
  |       +-----------------------------------------------------------------------+
  |       |EBMLMaxIDLength                                                        |
  |       +-----------------------------------------------------------------------+
  |       |EBMLMaxSizeLength                                                      |
  |       +-----------------------------------------------------------------------+
  |       |DocType                                                                |
  |       +-----------------------------------------------------------------------+
  |       |DocTypeVersion                                                         |
  |       +-----------------------------------------------------------------------+
  |       |DocTypeReadVersion                                                     |
  +-------+------------------+---------------+------------------------------------+
  |Segment|SeekHead          |Seek           |SeekID                              |
  |       |                  |               +------------------------------------+
  |       |                  |               |SeekPosition                        |
  |       |                  +---------------+------------------------------------+
  |       |                  |Seek           |SeekID                              |
  |       |                  |               +------------------------------------+
  |       |                  |               |SeekPosition                        |
  |       +------------------+---------------+------------------------------------+
  |       |SegmentInfo       |CRC-32                                              |
  |       |                  +----------------------------------------------------+
  |       |                  |SegmentUID                                          |
  |       |                  +----------------------------------------------------+
  |       |                  |SegmentFileName                                     |
  |       |                  +----------------------------------------------------+
  |       |                  |TimecodeScale                                       |
  |       |                  +----------------------------------------------------+
  |       |                  |DateUTC                                             |
  |       |                  +----------------------------------------------------+
  |       |                  |Title                                               |
  |       |                  +----------------------------------------------------+
  |       |                  |MuxingApp                                           |
  |       |                  +----------------------------------------------------+
  |       |                  |WritingApp                                          |
  |       +------------------+---------------+------------------------------------+
  |       |Tracks            |TrackEntry     |TrackNumber                         |
  |       |                  |               +------------------------------------+
  |       |                  |               |TrackUID                            |
  |       |                  |               +------------------------------------+
  |       |                  |               |FlagEnabled                         |
  |       |                  |               +------------------------------------+
  |       |                  |               |FlagForced                          |
  |       |                  |               +------------------------------------+
  |       |                  |               |FlagLacing                          |
  |       |                  |               +------------------------------------+
  |       |                  |               |MinCache                            |
  |       |                  |               +------------------------------------+
  |       |                  |               |TrackTimecodeScale                  |
  |       |                  |               +------------------------------------+
  |       |                  |               |MaxBlockAdditionID                  |
  |       |                  |               +------------------------------------+
  |       |                  |               |Name                                |
  |       |                  |               +------------------------------------+
  |       |                  |               |CodecID                             |
  |       |                  |               +------------------------------------+
  |       |                  |               |CodecName                           |
  |       |                  |               +------------------------------------+
  |       |                  |               |CodecDecodeAll                      |
  |       |                  |               +------------------------------------+
  |       |                  |               |SourceType                          |
  |       |                  |               +------------------------------------+
  |       |                  |               |SourceInformation                   |
  |       |                  +---------------+------------------------------------+
  |       |                  |TrackEntry     |TrackNumber                         |
  |       |                  |               +------------------------------------+
  |       |                  |               |TrackUID                            |
  |       |                  |               +------------------------------------+
  |       |                  |               |FlagEnabled                         |
  |       |                  |               +------------------------------------+
  |       |                  |               |FlagForced                          |
  |       |                  |               +------------------------------------+
  |       |                  |               |FlagLacing                          |
  |       |                  |               +------------------------------------+
  |       |                  |               |MinCache                            |
  |       |                  |               +------------------------------------+
  |       |                  |               |MaxCache                            |
  |       |                  |               +------------------------------------+
  |       |                  |               |TrackTimecodeScale                  |
  |       |                  |               +------------------------------------+
  |       |                  |               |MaxBlockAdditionID                  |
  |       |                  |               +------------------------------------+
  |       |                  |               |Name                                |
  |       |                  |               +------------------------------------+
  |       |                  |               |CodecID                             |
  |       |                  |               +------------------------------------+
  |       |                  |               |CodecName                           |
  |       |                  |               +------------------------------------+
  |       |                  |               |CodecDecodeAll                      |
  |       |                  |               +------------------------------------+
  |       |                  |               |SourceType                          |
  |       |                  |               +------------------------------------+
  |       |                  |               |SourceInformation                   |
  |       |                  |               +-----------------+------------------+
  |       |                  |               |TrackOperation   |TrackJoinBlocks   |
  |       |                  |               +-----------------+------------------+
  |       |                  |               |                 |TrackJoinUID      |
  |       +------------------+---------------+-----------------+------------------+
  |       |Chapters          |EditionEntry   |EditionUID                          |
  |       |                  |               +------------------------------------+
  |       |                  |               |EditionFlagHidden                   |
  |       |                  |               +------------------------------------+
  |       |                  |               |EditionFlagDefault                  |
  |       |                  |               +-----------------+------------------+
  |       |                  |               |ChapterAtom      |ChapterUID        |
  |       |                  |               |                 +------------------+
  |       |                  |               |                 |ChapterTimeStart  |
  |       |                  |               |                 +------------------+
  |       |                  |               |                 |ChapterFlagHidden |
  |       |                  |               |                 +------------------+
  |       |                  |               |                 |ChapterFlagEnabled|
  |       |                  |               +-----------------+------------------+
  |       |                  |               |ChapterAtom      |ChapterUID        |
  |       |                  |               |                 +------------------+
  |       |                  |               |                 |ChapterTimeStart  |
  |       |                  |               |                 +------------------+
  |       |                  |               |                 |ChapterFlagHidden |
  |       |                  |               |                 +------------------+
  |       |                  |               |                 |ChapterFlagEnabled|
  |       +------------------+---------------+-----------------+------------------+
  |       |Attachments       |AttachedFile   |FileDescription                     |
  |       |                  |               +------------------------------------+
  |       |                  |               |FileName                            |
  |       |                  |               +------------------------------------+
  |       |                  |               |FileMimeType                        |
  |       |                  |               +------------------------------------+
  |       |                  |               |FileData                            |
  |       |                  |               +------------------------------------+
  |       |                  |               |FileUID                             |
  |       +------------------+---------------+------------------------------------+
  |       |Tags              |Tag            |Targets                             |
  |       |                  |               +-----------------+------------------+
  |       |                  |               |SimpleTag        |TagName           |
  |       |                  |               |                 +------------------+
  |       |                  |               |                 |TagDefault        |
  |       |                  |               |                 +------------------+
  |       |                  |               |                 |TagString         |
  |       |                  +---------------+-----------------+------------------+
  |       |                  |Tag            |Targets                             |
  |       |                  |               +-----------------+------------------+
  |       |                  |               |SimpleTag        |TagName           |
  |       |                  |               |                 +------------------+
  |       |                  |               |                 |TagDefault        |
  |       |                  |               |                 +------------------+
  |       |                  |               |                 |TagBinary         |
  +-------+------------------+---------------+-----------------+------------------+
  |       |Cluster           |TimeCode                                            |
  |       |                  +----------------------------------------------------+
  |       |                  |Position                                            |
  |       |                  +----------------------------------------------------+
  |       |                  |PrevSize                                            |
  |       |                  +----------------------------------------------------+
  |       |                  |SimpleBlock                                         |
  |       |                  +----------------------------------------------------+
  |       |                  |SimpleBlock                                         |
  |       |                  +----------------------------------------------------+
  |       |                  |SimpleBlock                                         |
  |       +------------------+---------------+-----------------+------------------+
  |       |Cluster           |TimeCode                                            |
  |       |                  +----------------------------------------------------+
  |       |                  |Position                                            |
  |       |                  +----------------------------------------------------+
  |       |                  |PrevSize                                            |
  |       |                  +----------------------------------------------------+
  |       |                  |SimpleBlock                                         |
  |       |                  +---------------+------------------------------------+
  |       |                  |BlockGroup     |Block                               |
  |       |                  |               +------------------------------------+
  |       |                  |               |ReferencePriority                   |
  |       |                  +---------------+------------------------------------+
  |       |                  |BlockGroup     |Block                               |
  |       |                  |               +------------------------------------+
  |       |                  |               |ReferencePriority                   |
  |       |                  |               +------------------------------------+
  |       |                  |               |CodecState                          |
  |       +------------------+---------------+------------------------------------+
  |       |Cues              |CuePoint       |CueTime                             |
  |       |                  |               +-----------------+------------------+
  |       |                  |               |CueTrackPositions|CueTrack          |
  |       |                  |               |                 +------------------+
  |       |                  |               |                 |CueClusterPosition|
  |       |                  |               |                 +------------------+
  |       |                  |               |                 |CueBlockNumber    |
  |       |                  +---------------+-----------------+------------------+
  |       |                  |CuePoint       |CueTime                             |
  |       |                  |               +-----------------+------------------+
  |       |                  |               |CueTrackPositions|CueTrack          |
  |       |                  |               |                 +------------------+
  |       |                  |               |                 |CueClusterPosition|
  |       |                  |               +-----------------+------------------+
  |       |                  |               |CueTrackPositions|CueTrack          |
  |       |                  |               |                 +------------------+
  |       |                  |               |                 |CueClusterPosition|
  +-------+------------------+---------------+-----------------+------------------+

Specification
-------------

A Tawara file, as an EBML file, is made up of a large number of elements
of varying level. Level 0 elements are at the top of the document
structure. They contain within them the level 1 elements, which contain
level 2 elements, and so on. Some elements can appear at any level, but
most are restricted to a particular level.

The Header element defines the file as an EBML file, and in particular
defines the version of EBML in use, so that the EBML implementation can
determine if the file is readable. It also contains a DocType value that
identifies the document type as a Tawara document. If the DocType value is
different, Tawara parsers should not attempt to read the file.

All the Tawara-specific elements are contained within the Segment level 0
element.

The Metaseek element provides an index into the file for the other
level 1 elements in the same segment. Each Seek entry contains the class
ID of a level 1 element and the byte position of the instance of that
element. The Metaseek element is intended to be used when the file is
opened so that all level 1 elements can be located quickly. Seeking
while replaying data uses the Cues.

The Segment Information element provides information for uniquely
identifying the file, such as a title and a SegmentUID. The UIDs of any
other segments (which are usually stored in other files) related to this
segment are also mentioned here.

The Track element provides the information to understand the data stored
in each track, such as the codec used (i.e. the serialisation method, in
most cases), as well as useful meta-data such as track names and
information about the source of the data stored in the track.  The
information stored here is vital to the usefulness of the data stored in
the clusters. Each track also has a unique ID, the TrackUID. This is
useful when editing files and is also used by Tags.

The data is stored within the many Cluster elements. These are used to
break up the blocks of data to improve seeking and robustness to errors.
There is no limit on how much data an individual Cluster element can
contain, but it must fit within the timecode range of the Block timecode
value. A timecode is placed at the beginning of a Cluster, which usually
indicates the time of first Block in the Cluster, although it is not
required to. Within the Cluster are Blocks, which are contained in
BlockGroups along with any other information relevant to that Block.
Each Block contains a single item of data, along with its timecode,
track and any other relevant data.

The Cues element is used to make seeking efficient. They form a
time-based index into the clusters. A single CuePoint stores a CueTime
and an exact position in the file for each block at that timecode. What
is indexed by the Cues is flexible.  You can index every Block, or you
can only index Blocks spaced a number of seconds apart. A suitable
balance between index size and seeking efficiency must be chosen when
writing Tawara files.

The Attachments element simply contains named chunks of binary data,
and optionally includes a MIME-type to go with that data for
identification purposes.

The Tags element provides a flexible tagging system for storing
meta-data about the file. Tags can be specified on a per-item or
per-file basis.

The Chapters element provides index points into the file. Each "edition"
is a set of chapters to be played back in sequence. Within each edition
is one or more ChapterAtom elements, providing chapter start and end
timecodes.

File beginning
''''''''''''''

An EBML file always starts at the first occurrence of 0x1A. This allows
ASCII text to be included before the EBML data that can be displayed by
nearly anything. Beginning immediately with the first 0x1A byte MUST be
the EBML Header element.

Elements
''''''''

The table below formally defines the elements in a Tawara document.

==================== ===== =========== == == ===== ======= = ===========
EBML Header
------------------------------------------------------------------------
Element name         Level EBML ID     Ma Mu Rng   Default T Description
==================== ===== =========== == == ===== ======= = ===========
EBML                 0     1A 45 DF A3 \* \*               m Set the EBML characteristics of the data to follow. Each EBML document MUST start with this.
EBMLVersion          1     42 86       \*          1       u EBML parser version used to create the file.
EBMLReadVersion      1     42 F7       \*          1       u The minimum EBML parser version required to read this file.
EBMLMaxIDLength      1     42 F2       \*          4       u Maximum length of the IDs found in this file (4 or less in Tawara).
EBMLMaxSizeLength    1     42 F3       \*          8       u The maximum length of sizes found in this file (8 or less in Tawara).
                                                             This does not override the element size indicated at the beginning of
                                                             an element. Elements that have an indicated size which is larger than
                                                             the value allowed by this are invalid.
DocType              1     42 82       \*          tawara  s An ASCII string describing the document that follows the EBML header.
                                                             "tawara" for Tawara files.
DocTypeVersion       1     42 87       \*          1       u The version of DocType intepreter used to create the file.
DocTypeReadVersion   1     42 85       \*          1       u The minimum DocType version an interpreter requires to read this file.
==================== ===== =========== == == ===== ======= = ===========


==================== ===== =========== == == ===== ======= = ===========
Global elements (used everywhere in the format)
------------------------------------------------------------------------
Element name         Level EBML ID     Ma Mu Rng   Default T Description
==================== ===== =========== == == ===== ======= = ===========
Void                 g     EC                              b Used to void damaged data. The content of the element is discarded. Can also be used to reserve space in a sub-element for later use.
CRC-32               g     BF                              b A CRC value computed on all the data of the master element it is in,
                                                             excluding the CRC-32 element itself. This element SHOULD appear first
                                                             in its parent element for easier reading. All level 1 elements are
                                                             RECOMMENDED to include a CRC-32 element. The CRC MUST be calculated
                                                             using the IEEE CRC32 Little Endian CRC.
==================== ===== =========== == == ===== ======= = ===========


==================== ===== =========== == == ===== ======= = ===========
Segment
------------------------------------------------------------------------
Element name         Level EBML ID     Ma Mu Rng   Default T Description
==================== ===== =========== == == ===== ======= = ===========
Segment              0     18 53 80 67 \* \*               m This element contains all other top-level (level 1) Tawara elements. Typically, a Tawara file has one Segment.
==================== ===== =========== == == ===== ======= = ===========


==================== ===== =========== == == ===== ======= = ===========
Metaseek Information
------------------------------------------------------------------------
Element name         Level EBML ID     Ma Mu Rng   Default T Description
==================== ===== =========== == == ===== ======= = ===========
SeekHead             1     11 4D 9B 74                     m Contains the position of other level 1 elements.
Seek                 2     4D BB       \* \*               m Contains a single seek entry to an EBML element.
SeekID               3     53 AB       \*                  b The binary ID of the target element.
SeekPosition         3     53 AC       \*                  u The position of the element in the segment in bytes. 0 for the first
                                                             level 1 element.
==================== ===== =========== == == ===== ======= = ===========


==================== ===== =========== == == ===== ======= = ===========
Segment Information
------------------------------------------------------------------------
Element name         Level EBML ID     Ma Mu Rng   Default T Description
==================== ===== =========== == == ===== ======= = ===========
Info                 1     15 49 A9 66 \* \*               m Contains general information about the segment.
SegmentUID           2     73 A4             not 0         b A unique 128-bit ID to identify this segment among others.
SegmentFileName      2     73 84                           s A file name corresponding to this segment.
PrevUID              2     3C B9 23                        b A unique 128-bit ID to identify the previous linked segment.
PrevFileName         2     3C 83 AB                        s A file name corresponding to the previous segment.
NextUID              2     3C B9 23                        b A unique 128-bit ID to identify the next linked segment.
NextFileName         2     3C 83 AB                        s A file name corresponding to the next segment.
SegmentFamily        2     44 44          \*               b A unique 128-bit ID that all linked segments must use.
TimecodeScale        2     2A D7 B1    \*          1000000 u Timecode scale in nanoseconds.
Duration             2     44 89             >0            f Duration of this segment.
DateUTC              2     44 61                           d Date of the origin of the segment (the basis for the first cluster
                                                             timecode). i.e. the production date.
Title                2     7B A9                           s Name of the segment.
MuxingApp            2     4D 80                           s Muxing application or library (e.g. "libtawara-1.0").
WritingApp           2     57 41                           s Writing application (e.g. "pclrecord").
==================== ===== =========== == == ===== ======= = ===========


==================== ===== =========== == == ===== ======= = ===========
Cluster
------------------------------------------------------------------------
Element name         Level EBML ID     Ma Mu Rng   Default T Description
==================== ===== =========== == == ===== ======= = ===========
Cluster              1     1F 43 B6 75    \*               m Contains a set of Blocks.
Timecode             2     E7          \*                  u The timecode of this Cluster relative to the Segment.
SilentTracks         2     58 54                           m The list of tracks to be ignored for this cluster. This is useful for blanking out some tracks for part of the stream. All tracks not specified here are used, regardless of earlier silencing.
SilentTrackNumber    3     58 D7          \*               u The track number of a track that should be made silent.
Position             2     A7                              u The position of this Cluster in the segment. This is helpful for
                                                             re-synchronisation in damaged streams.
PrevSize             2     AB                              u The size of the previous Cluster, in bytes. This can be useful when
                                                             playing backwards.
SimpleBlock          2     A3             \*               b A simplified version of the Block element. Allows for storing just
                                                             data without extra information in order to reduce overhead. See
                                                             `SimpleBlock format`_.
BlockGroup           2     A0             \*               m A container element for a single block and any related information.
Block                3     A1          \*                  b A block, containing the actual data and a timecode. See
                                                             `Block format`_ for more information.
BlockAdditions       3     75 A1                           m Additional blocks needed to complete this block. EBML parsers that
                                                             have no knowledge of the Block structure can skip these easily.
BlockMore            4     A6          \* \*               m The BlockAdditional and some parameters.
BlockAddID           5     EE          \*    Not 0 1       u The ID to identify the BlockAdditional level.
BlockAdditional      5     A5          \*                  b Interpreted by the codec as it wishes, using the BlockAddID.
BlockDuration        3     9B                      Track   u The duration of this Block, based on TimecodeScale. If a default
                                                   Durata-   duration is set for for the track, this element is mandatory (but
                                                   toin      will be set to the default value if not actually present). When not
                                                             written and no DefaultDuration is set, it is calculated as the time
                                                             between this block and the next block in the track in time order (not
                                                             file order). This element can be useful at the end of a track, or
                                                             when there should be a break in the track.
ReferencePriority    3     FA          \*          0       u This block is referenced and has the specified cache priority. When
                                                             cached, only a block of the same or higher priority can replace
                                                             this block. If the value is zero (the default), it means the block
                                                             is not referenced.
ReferenceBlock       3     FB             \*               i Time code of another block in the same track used as a reference
                                                             when understanding this frame. The time code is relative to the block
                                                             it is attached to.
CodecState           3     A4                              b The new codec state to use from this block on. How this data is
                                                             interpreted is private to the codec used. This information should
                                                             always be referenced by a seek entity.
==================== ===== =========== == == ===== ======= = ===========


==================== ===== =========== == == ===== ======= = ===========
Track
------------------------------------------------------------------------
Element name         Level EBML ID     Ma Mu Rng   Default T Description
==================== ===== =========== == == ===== ======= = ===========
Tracks               1     16 54 AE 6B    \*               m Contains information on one or more tracks.
TrackEntry           2     AE          \* \*               m Describes a single track.
TrackNumber          3     D7          \*    not 0         u The track number used in the Block Header.
TrackUID             3     73 C5       \*    not 0         u A unique ID to identify the track. This should be kept the same when making a direct stream copy of the track to another file.
TrackType            3     83          \*    1-254         u The type of track. 0x11 for subtitles, 0x70 for data.
FlagEnabled          3     B9          \*    0-1   1       u Set to 1 if the track is used.
FlagDefault          3     88          \*    0-1   1       u Ignored.  Must be read if present. Do not write it.
FlagForced           3     55 AA       \*    0-1   0       u Set to 1 if the track MUST be used during playback.
FlagLacing           3     9C          \*    0-1   1       u Set to 1 if the track may contain blocks using lacing.
MinCache             3     6D E7       \*          0       u The minimum number of blocks that should be able to be cached
                                                             during reply.
MaxCache             3     6D F8       \*          0       u The maximum number of blocks that need to be cached for reference
                                                             blocks and the current block.
DefaultDuration      3     23 E3 83          not 0         u The number of nanoseconds per block if no individual time codes are
                                                             given in the Blocks.
TrackTimecodeScale   3     23 31 4F    \*    >0    1.0     f The scale to apply on this track to work at normal speed in relation
                                                             to other tracks.
MaxBlockAdditionID   3     55 EE       \*          0       u The maximum value of BlockAddID. A value of 0 means there are no
                                                             BlockAdditions for this track.
Name                 3     53 6E                           s A human-readable track name.
CodecID              3     86          \*                  s An ID corresponding to the codec used in this track. Must not be empty.
CodecPrivate         3     63 A2                           b Private data known only to the codec.
CodecName            3     25 86 88                        s A human-readable string identifying the codec.
AttachmentLink       3     74 46             not 0         u The UID of an attachment that is used by the codec.
CodecDecodeAll       3     AA          \*    0-1   0       u Damaged data can potentially be decoded by the codec.
TrackOverlay         3     6F 24          \*               u Specify that this track is an overlay for the track specified by
                                                             this UID. This means that when this track has a gap (using
                                                             SilentTracks), the overlay track should be used instead. The ordering
                                                             matters - the first available TrackOverlay will be used.
TrackOperation       3     E2                              m The operation that needs to be applied on the source tracks to
                                                             create this virtual track. See `Track operations`_.
TrackJoinBlocks      4     E9                              m Contains a list of all tracks whose Blocks need to be combined to
                                                             create this virtual track.
TrackJoinUID         5     ED          \* \* not 0         u The TrackUID of a track whose Blocks are used in creating this
                                                             virtual track.
==================== ===== =========== == == ===== ======= = ===========


==================== ===== =========== == == ===== ======= = ===========
Cueing data
------------------------------------------------------------------------
Element name         Level EBML ID     Ma Mu Rng   Default T Description
==================== ===== =========== == == ===== ======= = ===========
Cues                 1     1C 53 BB 6B                     m Speeds seeking by providing an index. All entries are local to the segment. It is RECOMMENDED that this element is used.
CuePoint             2     BB          \* \*               m Contains all information for a single seek point in the segment.
CueTime              3     B3          \*                  u Time code relative to the segment time base.
CueTrackPositions    3     B7          \* \*               m Positions corresponding to the time code for each track.
CueTrack             4     F7          \*    not 0         u The track for which a position is given.
CueClusterPosition   4     F1          \*                  u The position of the cluster containing the required Block.
CueBlockNumber       4     53 78             not 0 1       u Number of the Block in the specified cluster.
CueCodecState        4     EA                      0       u The position of the CodecState corresponding to this Cue element. 0
                                                             means that the data is taken from the initial TrackEntry.
CueReference         4     DB             \*               m The Clusters containing the required reference Blocks.
CueRefTime           5     96          \*                  u Timecode of the referenced Block.
==================== ===== =========== == == ===== ======= = ===========


==================== ===== =========== == == ===== ======= = ===========
Attachments
------------------------------------------------------------------------
Element name         Level EBML ID     Ma Mu Rng   Default T Description
==================== ===== =========== == == ===== ======= = ===========
Attachments          1     19 41 A4 69                     m Contains attached data.
AttachedFile         2     61 A7       \* \*               m An attached file.
FileDescription      3     46 7E                           s A human-friendly name for the attached file.
FileName             3     46 6E       \*                  s File name of the attached file.
FileMimeType         3     46 60       \*                  s MIME type of the file.
FileData             3     46 5C       \*                  b The data of the file.
FileUID              3     46 AE       \*    not 0         u A unique ID representing the file.
==================== ===== =========== == == ===== ======= = ===========

==================== ===== =========== == == ===== ======= = ===========
Chapters
------------------------------------------------------------------------
Element name         Level EBML ID     Ma Mu Rng   Default T Description
==================== ===== =========== == == ===== ======= = ===========
Chapters             1     10 43 A7 70                     m Chapters and chapter sets for defining index points in the data.
EditionEntry         2     45 B9       \* \*               m Contains all information about a segment edition.
EditionUID           3     45 BC             not 0         u A unique ID to identify the edition. Typically used to tag editions.
EditionFlagHidden    3     45 BD       \*    0-1   0       u If an edition is hidden (1), it should not be available to the user.
EditionFlagDefault   3     45 DB       \*    0-1   0       u If this flag is set (1), the edition is the default to use.
EditionFlagOrdered   3     45 DD             0-1   0       u Specify if the chapters in this edition can be defined multiple times
                                                             and if the order to play them is enforced.
ChapterAtom          3+    B6          \* \*               m Contains a single chapter.
ChapterUID           4     73 C4       \*    not 0         u A unique ID for the chapter.
ChapterTimeStart     4     91          \*                  u The timecode of the start of the chapter.
ChapterTimeEnd       4     92                              u The timecode of the end of the chapter (this timecode excluded).
ChapterFlagHidden    4     98          \*    0-1   0       u If 1, the chapter is hidden from the user.
ChapterFlagEnabled   4     45 98       \*    0-1   0       u Specify whether the chapter is enabled. When disabled, all data
                                                             between the start and end of this chapter should be skipped.
ChapterSegmentUID    4     6E 67             >0            u A segment to play in place of this chapter.
ChapterTrack         4     8F                              m A list of tracks on which the chapter applies. If this element is not
                                                             present, all tracks apply. Choosing this chapter will select the listed
                                                             tracks and deselect the tracks that are not listed.
ChapterTrackNumber   5     89          \* \* not 0         u The UID of a track to apply this chapter to.
ChapterDisplay       4     80                              m How the chapter is displayed.
ChapString           5     85          \*                  s Contains the string to use to display the chapter.
ChapLanguage         5     43 7C       \* \*       eng     s Specifies the language of the tag, in the `ISO 639.2`_ form.
ChapCountry          5     43 7E          \*               s The countries corresponding to the string, using the same 2 octects as in
                                                             Internet domains.
==================== ===== =========== == == ===== ======= = ===========

==================== ===== =========== == == ===== ======= = ===========
Tagging
------------------------------------------------------------------------
Element name         Level EBML ID     Ma Mu Rng   Default T Description
==================== ===== =========== == == ===== ======= = ===========
Tags                 1     12 54 C3 67    \*               m A set of tags.
Tag                  2     73 73       \* \*               m A single tag.
Targets              3     63 C0                           m All UIDs where the tags applies. Empty if the tag applies to everything in the segment.
TagTrackUID          4     63 C5          \*       0       u A unique ID to identify a Track(s) the tag applies to. If the value is
                                                             zero at this level, the tag applies to all tracks in the segment.
TagEditionUID        4     63 C9          \*       0       u A unique ID to identify the EditionEntry(s) the tag applies to. If the
                                                             value is zero at this level, the tag applies to all editions in the
                                                             Segment.
TagChapterUID        4     63 C4          \*       0       u A unique ID to identify the Chapter(s) the tag applies to. If the
                                                             value is zero at this level, the tag applies to all chapters in the
                                                             Segment.
TagAttachmentUID     4     63 C6          \*       0       u A unique ID to identify an Attachment(s) the tag applies to. If the
                                                             value is zero at this level, the tag applies to all the attachments in
                                                             the segment.
SimpleTag            3+    67 C8       \* \*               m Contains general information about the target.
TagName              4+    45 A3       \*                  s The name of the Tag.
TagLanguage          4+    44 7A       \*          und     s Specifies the language of the tag, in the `ISO 639.2`_ form.
TagDefault           4+    44 84       \*    0-1   1       u Indicates if this is the default or original language for the tag.
TagString            4+    44 87                           s The value of the Tag. This cannot be used in the same SimpleTag as
                                                             TagBinary.
TagBinary            4+    44 85                           b The value of the Tag if it is binary data. This cannot be used in the
                                                             same SimpleTag as TagString.
==================== ===== =========== == == ===== ======= = ===========

.. _`ISO 639.2`:
   http://www.loc.gov/standards/iso639-2/php/English_list.php


Table columns
'''''''''''''

Element name
  The full name of the element.

Level
  The level within the EBML tree that the element may occur at. A "+"
  symbol after the number indicates that the element may occur at any
  level at or after the number mentioned. With the exception of Global
  Elements, an element MUST only occur within the nearest element
  preceding it in level. An element at level 3 may only be a child of
  the nearest preceding level 2 element.

EBML ID
  The Element ID displayed as octets.

Mandatory (Ma)
  This element MUST be present in the file if its parent element is
  present and no default value is available.

Multiple (Mu)
  The element may appear multiple times within its parent element.
  Elements that are not marked as multiple MUST NOT appear more than
  once in each instance of the parent element.

Range (Rng)
  Valid range of values to store in the element. Two hyphenated numbers
  indicates a value between the two numbers, inclusive. Numeric values
  are expressed in decimal. "Not 0" indicates any value allowed by the
  element type other than zero. ">0" indicates that a positive, non-zero
  value is required.

Default
  The default value of the element, if it has one. The default value is
  used if the element's parent is present but the element is not. If
  both the element and its parent are not present, the default value is
  ignored.

Element type (T)
  The data type of the element's value. The 8 basic EBML types ONLY are
  used:

  - Signed integer (i)
  - Unsigned integer (u)
  - Float (f)
  - String (s) - All strings are in UTF-8 format, except where
    explicitly specified.
  - Date (d) - Dates are represented as a signed 8-byte integer in nanoseconds,
    with the origin (0) placed at 2001-01-01T00:00:00,0 UTC.
  - Sub-elements (m)
  - Binary, i.e. raw bytes (b)

Description
  A short description of the element's purpose.


Blocks
======

The data within a SimpleBlock or a Block is a binary blob, but part of
its format is well-known, consisting of a binary header and the data
itself.

Block format
------------

The header contains the track number for the Block, the timecode of the
Block (relative to its parent Cluster's timecode), and a small number of
flags. The header format is as follows:

+--------+------------------------------------------------------------------+
|Offset  |Description                                                       |
+--------+------------------------------------------------------------------+
|0x00    |Track number. Encoded as an EBML unsigned variable-length integer.|
+--------+------------------------------------------------------------------+
|0x01+   |Timecode, relative to the Cluster timecode, as a signed           |
|Size    |16-bit integer.                                                   |
|of track|                                                                  |
|number  |                                                                  |
+--------+------------------------------------------------------------------+
|0x03+   |Flags                                                             |
|        +---+--------------------------------------------------------------+
|        |Bit|Description                                                   |
|        +---+--------------------------------------------------------------+
|        |0-3|Reserved, set to 0.                                           |
|        +---+--------------------------------------------------------------+
|        |4  |Invisible, the codec should decode this block but not use it. |
|        +---+--------------------------------------------------------------+
|        |5-6|Lacing:                                                       |
|        |   | 00:                                                          |
|        |   |   No lacing                                                  |
|        |   | 11:                                                          |
|        |   |   EBML lacing                                                |
|        |   | 10:                                                          |
|        |   |   Fixed-size lacing                                          |
|        +---+--------------------------------------------------------------+
|        |7  |Not used                                                      |
+--------+---+--------------------------------------------------------------+

When lacing is enabled for a Block, the header is immediately followed
by the following bytes:

+--------+------------------------------------------------------------------+
|Offset  |Description                                                       |
+--------+------------------------------------------------------------------+
|0x00    |Number of frames in the lace - 1, as an unsigned 8-bit integer.   |
+--------+------------------------------------------------------------------+
|0x01 -  |Lace-coded size of each frame in the lace, except for the last    |
|0xXX    |frame. This value is not used and MUST NOT be present when fixed- |
|        |size lacing is used, as the size of each frame can be calculated. |
+--------+------------------------------------------------------------------+

Following this is the binary data of the Block, which may consist of
multiple Blocks' data if lacing is in use.

SimpleBlock format
------------------

The SimpleBlock format is very similar to the standard Block format. The
only change is the addition of two flags: Keyframe and Discardable.

+--------+------------------------------------------------------------------+
|Offset  |Description                                                       |
+--------+------------------------------------------------------------------+
|0x00    |Track number. Encoded as an EBML unsigned variable-length integer.|
+--------+------------------------------------------------------------------+
|0x01+   |Timecode, relative to the Cluster timecode, as a signed           |
|Size    |16-bit integer.                                                   |
|of track|                                                                  |
|number  |                                                                  |
+--------+------------------------------------------------------------------+
|0x03+   |Flags                                                             |
|        +---+--------------------------------------------------------------+
|        |Bit|Description                                                   |
|        +---+--------------------------------------------------------------+
|        |0  |When set, this block is a Keyframe block.                     |
|        +---+--------------------------------------------------------------+
|        |1-3|Reserved, set to 0.                                           |
|        +---+--------------------------------------------------------------+
|        |4  |Invisible, the codec should decode this block but not use it. |
|        +---+--------------------------------------------------------------+
|        |5-6|Lacing:                                                       |
|        |   | 00:                                                          |
|        |   |   No lacing                                                  |
|        |   | 11:                                                          |
|        |   |   EBML lacing                                                |
|        |   | 10:                                                          |
|        |   |   Fixed-size lacing                                          |
|        +---+--------------------------------------------------------------+
|        |7  |Discardable; this block can be safely discarded during        |
|        |   |playback if necessary.                                        |
+--------+---+--------------------------------------------------------------+

Data after this is in the same format as normal Blocks.

Lacing
------

Lacing can be used to save space when storing data. It is typically used
for small blocks of data. For example, if three blocks of data each
contain 800, 500 and 1000 bytes of data, they can be stored in a lace,
allowing them to share a block and save space. An example of where this
is useful is audio data, which often has small sample sizes. However,
there is a trade-off: the longer a lace, the more difficult it is to
seek within the track. Generally, laces should be kept short.

Two types of lacing are supported by Tawara: EBML-based lacing and
fixed-size lacing.

EBML lacing
'''''''''''

In EBML lacing, the size of each block's data in a lace is stored as the
difference with the previous size, encoded as an EBML unsigned
variable-length integer. The first value is always an unsigned integer,
while the remainder are stored as signed integers. Range-shifting is
used to get a sign in the value by subtracting half the available range:

============== ========
Integer length Offset
============== ========
1 byte         2^6 - 1
2 bytes        2^13 - 1
3 bytes        2^20 - 1
4 bytes        2^27 - 1
5 bytes        2^34 - 1
6 bytes        2^41 - 1
7 bytes        2^48 - 1
============== ========

The Block format then consists of:

 - The Block head, with lacing bits set to 0b11.
 - The lacing head, containing the number of blocks in the lace - 1.
 - The lacing sizes for all blocks except the last. For the above
   example, the first size (800 bytes) will be 0x320+0x4000 = 0x4320,
   the second size (500 bytes) will be stored as -300: -0x12C + 0x1FFF +
   0x4000 = 0x5ED3. The size of the last block's data does not need to
   be stored.
 - Data in the first block.
 - Data in the second block.
 - Data in the third block.

Fixed-size lacing
'''''''''''''''''

Fixed-size lacing is very simple, relying on each Block's data in the
lace being the same size. Only the number of Blocks stored in the lace
needs to be saved. For example, for three frames, the Block format
consists of:

 - The Block head, with lacing bits set to 0b11.
 - The lacing head, containing the number of frames in the lace - 1. In
   the example, this will be 2.
 - Data in the first block.
 - Data in the second block.
 - Data in the third block.


Element ordering
================

Apart from the Header element appearing first, as required by EBML,
there is no required order of level 1 elements. However, optimal
orderings exist for efficient playback, seeking, editing and writing.
This section gives some guidelines on the order of level 1 elements.

Only the SegmentInfo, Tracks and Cluster level 1 elements are necessary
for a Tawara file to be usable. The Segment Info and Track Info must be
read before the clusters can be used. The Cues element greatly improves
seeking. The Metaseek element improves file opening times by removing
the need to scan the entire file for level 1 elements (a process that
can take some time if there are a large number of clusters).

A Tawara file may be edited after it has been recorded. For example, tags
could be set, attachments added, or even new tracks added. When the file
is edited, typically the Metaseek element must be updated and other
elements may need to be voided or grown. It is a good idea to add some
padding after these level 1 elements to ease editing.

Metaseek placement
------------------

The Metaseek element provides an index for the other elements in a
Segment. If it is not the first element read in a segment, then it
would be a lot less useful, so it SHOULD be the first element in a
segment. To facilitate adding new level 1 elements after file creation
(for example, to add some attachments), there SHOULD be some padding
placed outside and after the Metaseek element.

Cues placement
--------------

The size of the information stored in the Cues element is usually hard
to predict while the file is being created, so it is typically placed
after the clusters in the file. This does not degrade performance if a
Metaseek element is also used, as the seeking process will require at
least one jump anyway.  Adding another to first seek to the cues section
is not significant.

If the Cues information size is known in advance, it can be placed
before the clusters.

A concern while recording live data for an unknown length of time is
maintaining the cues to be written after the clusters once recording is
complete. The cue data will continue to grow during recording,
requiring an increasingly-large quantity of memory to maintain. This may
not be maintainable, depending on the computing resources available.
Recorders should consider regularly dumping cue data to disk during
recording and automatically reading it into the final file when
recording is complete. Two possible methods are:

 #. The cue data could be dumped into a temporary file, properly
    formatted as tags ready to be copied into the final file when
    recording is complete.

 #. Regularly write cue tags when completing clusters, interspersing the
    two types of tag. When recording is complete, merge all the cue tags
    into a single cue tag and re-order the file to place cues before or
    after the clusters.

The first option is preferable because Tawara only allows one cue element
per segment. If recording terminates early because of an error, the file
will not be a valid Tawara file without a repair operation to remove the
multiple cue elements.

Attachments placement
---------------------

Attachments are optional information. They MUST NOT be required to be
read in order to read the file. The optimal position depends on the
purpose of the attachments. However, if the attachments are intended to
be edited after creating the file, it is RECOMMENDED that they are
placed after the clusters and close to the end of the file, and that a
Metaseek element be used.

Tags placement
--------------

While tags are often edited after the file is created, they are also
typically given values at creation time. It is beneficial to be able to
read tags out quickly when scanning files. To facilitate this, tags that
are known at creation time should be placed before the clusters. If the
tags are later edited (and sufficient padding is not available for the
new data), the original element should be voided out and the tags
re-written at the end of the file.  The size of tags is typically
relatively small compared with the data.

Tracks placement
----------------

The Tracks must be read before reading the Clusters in order to
understand the data they contain. This means that the optimal placement
of the Tracks for reading is before the Clusters. However, during
recording of live data, the Tracks information may not be known in
advance, instead being accumulated during recording. The optimal
placement of the Tracks for recording is therefore *after* the Clusters.
In such cases it is highly RECOMMENDED that a Metaseek element be used
so that the Tracks can be quickly found during later playback without
needing to scan through all the Clusters looking for it. It should be
noted, however, that writing the Tracks after recording is complete is
vulnerable to crashes making the entire file unreadable.

Cluster Timecode placement
--------------------------

Each Block/BlockGroup/SimpleBlock in a cluster needs that cluster's time
code.  Therefore, the Cluster Timecode MUST be the first element in the
Cluster.

Additionally, it is RECOMMENDED that the SimpleBlock/Block elements are
placed after the other elements in a Cluster, as this simplifies
reading.

CRC-32 placement
----------------

The EBML CRC-32 element value applies to all the data enclosed in its
parent EBML element except for itself. It MUST be the first element so
that readers are aware of the presence of a CRC and can apply a CRC
check to all data that follows.

Block Ordering
--------------

Blocks that reference other blocks require those blocks to be loaded
prior to decoding. To assist this, it is REQUIRED that reference blocks
appear before blocks that reference them within their Cluster.

Blocks could appear in any order with respect to their timecodes.
However, this makes efficient playback difficult, requiring considerable
seeking in the file. It is therefore REQUIRED that blocks in the same
track are stored in time-order within their Cluster, except where that
conflicts with the above requirement for reference blocks. Additionally,
Clusters are RECOMMENDED to be stored in time-order within their
Segment, if possible (often, merging clusters from two files will create
a segment with clusters that are not stored in time order, the clusters
from one file placed after the clusters from another).


Notes
=====

CuePoints
---------

The Tawara format only defines the format of cueing data. It does not
specify whether a single CueTime must contain a reference to every track
or only those tracks with data at that point in time. Referring to the
next Block in every track at or after the CueTime simplifies seeking,
but complicates recording. On the other hand, referring only to the
Blocks that specifically occur at that point in time simplifies
recording while potentially complicating playback (finding the next
Block in tracks not mentioned in the CuePoint requires searching the
file). Fortunately, the requirement for Clusters and Blocks to be stored
in time-order means that the file must only be searched forwards, and
the nature of robot data is such that it is typically not necessary to
have an entry for every track available at once.

Default values
--------------

The default value of an element is assumed when that element is not
present in the document. It is assumed only within the scope of its
parent element; if the parent is not present or assumed, then the
element's value cannot be assumed.

EBML Class
----------

A larger EBML class ID indicates an element with a lower probability or
higher importance.

Larger class IDs can often be used as sychronisation points if the file
is damaged. Elements that are used more frequently but do not need to
act as a synchronisation point should have a small class ID.

For example, the Cluster element has a four byte ID, so it can be used
as a synchronisation point if the file is damaged. On the other hand,
the BlockGroup, which is very common, has a single byte ID to conserve
space.

Position references
-------------------

Position references often refer to the position, in bytes, from the
beginning of the first Segment element. This means that Position 0
refers to the first Segment. In data spanning multiple linked segments,
either in the same file or different files, the position is the
accumulated offset from the first Segment. For example, a reference to a
position in the third Segment will be the total size of the first
segment plus the total size of the second segment plus the offset of the
element in the third segment.

Segment linking
---------------

Segments can be linked together. All linked segments should be treated
as a single segment for the purposes of playback. Within the Segment
Info element, the NextUID and PrevUID elements MUST point to the
appropriate SegmentUIDs. The timecodes in one segment are considered
relative to the final time in the previous segment. This allows a set of
files to be played as a single file, while also allowing those files to
be used independently.

Within a set of linked segments, track names and all UIDs MUST follow
the guidelines in `Track names`_, below.

SegmentUIDs
-----------

The 128-bit SegmentUID values SHOULD be as unique as possible. One
method is to use UUIDs. This is the preferred method. An alternative is
to compute MD5 sums, such as of some or all of the data in a segment.

Tags
----

Formatting
''''''''''

- The TagName MUST always be written in all capitals and contain no
  spaces.

- The fields with dates MUST have the following format:

  YYYY-MM-DD hh:mm:ss.mss

  YYYY:
    Year

  MM:
    Month

  DD:
    Day

  hh:
    Hour

  mm:
    Minute

  ss:
    Second

  mss:
    Millisecond.

  It is valid to not include more accurate values. For example, the
  seconds and milliseconds may be omitted. However, if, for example, the
  minutes value is present, the year, month, day and hour values must
  also be present.

- Fields that require a float MUST use a period (".") rather than a
  comma (","). Thousands separators MUST NOT be used.

- For currency quantities, there MUST be only a numeric value in the
  tag. For example, you would store "15.59" instead of "$15.59USD".

Valid Tags
''''''''''

This specification does not place any restrictions on valid tags other
than those described above. In addition, it does not define an official
list of tags. Future versions of the specification may do so.

Timecodes
---------

Timecodes in a Tawara document are hierarchical. The timecode in a Block
is relative to the timecode of its parent Cluster, the result of which
is then multiplied by the TimecodeScale to get the Raw Timecode in
nanoseconds. The Cluster's timecode is relative to the timecode of its
Segment.

If absolutely sample-accurate seeking is required, then the
TimecodeScale must be set appropriately to avoid rounding errors leading
to missed samples while seeking. To achieve this, set the TimecodeScale
to a value that is a divisor of the maximum data rate expected. Note
that the TimecodeScale can be recalculated at a later date, but that
this may lead to loss in timecode precision if the new value is not a
divisor of the original value.

The Block's timecode is a 16-bit signed integer, with a range of -32768
to 32767 units. The size of a unit is defined by the TimecodeScale. When
using the default TimecodeScale of 1000000ns, each unit is 1ms. This
means that the maximum length of time that can be stored in a single
cluster when using the default TimecodeScale is 65536ms (reduced to
32767ms if the Cluster's timecode is 0). Value of a Block's timecode
that will, when offset by the Cluster's timecode, give a negative value
are invalid.

The TrackTimecodeScale is used to adjust the replay speed of a track. It
can be used to re-align tracks that have differing playback speeds, for
example.

Track names
-----------

Tracks are uniquely identified by their TrackUIDs. Track names are NOT
REQUIRED to be unique, although for the purposes of ease-of-use it is
RECOMMENDED that they are.

Track operations
----------------

The TrackOperation element allows multiple real tracks to be combined
into a virtual track. This virtual track is a proper track with its own
TrackUID and flags. The blocks from each track are joined into a single
stream. Usually, this is so that two tracks that have a gap between them
can be played as a single track (such as when muxing two files
together). It is NOT RECOMMENDED that the timecodes of the joined tracks
overlap. The results will vary significantly by codec and are so
undefined.

When playing back a virtual track, the Blocks from the source tracks
MUST be treated as if they were for a single track.


Chapters
========

The Chapters in the Tawara format are essentially an index. They provide
index points into the data that can be grouped together into Editions.
An Edition is essentially a particular way of replaying a file, giving
the data from a specified collection of tracks for specified time
slices.

Consider a file containing various types of sensor data recorded during
a long run with several distinct environments:

 - 0min - 5min: Office
 - 5min - 7min: Hallway
 - 7min - 15min: Carpark
 - 15min - 20min: Warehouse
 - 20min - 25min: Carpark

(Timecodes in minutes are for example only; chapters can reference any
timecode available in the Tawara file.)

This would require the following Chapters elements:

================ =====
Element          Value
================ =====
EditionEntry
ChapterAtom
ChapterUID       0x12345678
ChapterTimeStart 0min
ChapterTimeEnd   5min
ChapterDisplay
ChapterString    Office
ChapterAtom
ChapterUID       0x23456781
ChapterTimeStart 5min
ChapterTimeEnd   7min
ChapterDisplay
ChapterString    Hallway
ChapterAtom
ChapterUID       0x34567812
ChapterTimeStart 7min
ChapterTimeEnd   15min
ChapterDisplay
ChapterString    Carpark
ChapterAtom
ChapterUID       0x45678123
ChapterTimeStart 15min
ChapterTimeEnd   20min
ChapterDisplay
ChapterString    Warehouse
ChapterAtom
ChapterUID       0x56781234
ChapterTimeStart 20min
ChapterTimeEnd   25min
ChapterDisplay
ChapterString    Carpark 2
================ =====


Acknowledgements
================

This format described in this document is heavily based on the Matroska
container format for multimedia data [3]_.


References
==========

.. [1] EBML Homepage (retrieved 2011-04-27)
   http://ebml.sourceforge.net/

.. [2] EMBL RFC (Draft) (retrieved 2011-04-27)
   http://matroska.org/technical/specs/rfc/index.html

.. [3] Matroska Media Container (retrieved 2011-04-27)
   http://matroska.org/



..
   Local Variables:
   mode: indented-text
   indent-tabs-mode: nil
   sentence-end-double-space: t
   fill-column: 70
   coding: utf-8
   End:

