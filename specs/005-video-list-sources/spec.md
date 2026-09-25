# Feature Specification: Video List Sources

**Feature Branch**: `005-video-list-sources`
**Created**: 2026-09-25
**Status**: Draft
**Input**: User description: "A new feature to support backing up arbitrary list of (additional) videos. Target issue with some considerations in https://github.com/con/annextube/issues/37"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Archive a curated list of videos as a named collection (Priority: P1)

An archivist maintains a DataLad mirror of a website (e.g. a book site such as
mlbenchmarks.org) whose pages embed or link to YouTube videos from many
different channels. None of those videos share a channel or a YouTube
playlist. The archivist wants to back up exactly those videos — with full
metadata, captions, comments and thumbnails — and see them in the archive's
web UI as one named group, in the order they were listed.

They declare a "video list" source in the archive configuration, give it a
name and optional human-readable title, and list the videos either inline or
in a list file stored in the archive. Running a backup fetches every listed
video and presents the list as a local playlist.

**Why this priority**: This is the core capability requested by the issue;
today there is no way to archive videos that are not grouped by a single
channel or YouTube playlist.

**Independent Test**: Configure a video list source with three
`@AnnexTubeTesting` videos, run a backup, and confirm that each video is in
the archive's regular video tree and that a local playlist with the chosen
name lists exactly those three videos in list order.

**Acceptance Scenarios**:

1. **Given** an archive whose configuration declares a video list source named
   `mlbenchmarks-org` with three video URLs, **When** the user runs a backup,
   **Then** all three videos are archived with the same metadata, captions,
   comments and thumbnails a channel backup would produce, and a local
   playlist `mlbenchmarks-org` lists them in the configured order.
2. **Given** a video list source whose entries are in a list file inside the
   archive, **When** the user runs a backup, **Then** the entries are read
   from that file, and the result is the same as for inline entries.
3. **Given** a source with both inline entries and a list file, **When** the
   user runs a backup, **Then** the union of both is archived, inline entries
   first, then file entries, with duplicates dropped.
4. **Given** a video list source, **When** the user opens the web UI, **Then**
   the local playlist is shown alongside YouTube playlists, with its title,
   and is visibly marked as locally defined (not a YouTube playlist).

---

### User Story 2 - Accept video references in any common URL form (Priority: P1)

Lists of videos are usually harvested from web pages, so the same video
appears as a watch link, a short link, an embed link, a Shorts link, a
privacy-enhanced embed, a mobile link, or a bare video ID — often with a
start time or an unrelated playlist parameter attached. The user wants to paste
or generate such lists without cleaning them up by hand.

**Why this priority**: Without robust recognition of video references, User
Story 1 is unusable for its main use case (lists produced by scraping
websites).

**Independent Test**: Feed a list containing every supported form of a
reference to the same video plus some invalid lines, and confirm that exactly
one video is archived, and that each invalid line is reported with its line
number.

**Acceptance Scenarios**:

1. **Given** entries in any of these forms — watch link (with any extra
   parameters, including a playlist or time parameter), short link, embed /
   Shorts / live / legacy `v/` link, privacy-enhanced embed link, mobile or
   music link, or a bare 11-character video ID — **When** the list is read,
   **Then** each entry resolves to the same canonical video identifier.
2. **Given** the same video listed several times, **When** the list is read,
   **Then** it appears once, at the position of its first occurrence.
3. **Given** an entry carrying a start offset (`t=` or `start=`), **When** the
   video is shown from the local playlist in the web UI, **Then** playback can
   start at that offset.
4. **Given** an entry that is not a recognizable video reference (e.g. a
   channel URL, a non-YouTube URL, a malformed ID), **When** the list is read,
   **Then** the entry is reported with its source (file and line, or inline
   position) and skipped, and all valid entries are still processed.

---

### User Story 3 - Keep provenance of where each video was found (Priority: P2)

For a list harvested from a website, the archivist wants to record *where*
each video was referenced (the "referrer" page) and keep any other notes they
attached, so that the archive documents why each video is there and the web
UI can link back to the referencing page.

**Why this priority**: Provenance is highly valuable for the motivating use
case (archiving next to a website mirror), but the archive is useful without
it.

**Independent Test**: Provide a tabular list file with `url`, `referrer` and
`note` columns, back up, and confirm that each video's entry in the local
playlist carries its referrer(s) and note, and that the web UI shows a
"referenced from" link.

**Acceptance Scenarios**:

1. **Given** a tabular list file with a header row containing a `url` column
   and optionally a `referrer` column and other columns, **When** the list is
   read, **Then** the referrer and all extra columns are kept verbatim with
   that entry.
2. **Given** the same video listed on several rows with different referrers,
   **When** the list is read, **Then** the video appears once and all its
   distinct referrers are kept, in first-seen order.
3. **Given** a plain list file (one reference per line, `#` comments and blank
   lines allowed), **When** the list is read, **Then** it is accepted without
   a header, with no referrers or annotations.
4. **Given** an entry with a referrer, **When** the user views that video from
   the local playlist in the web UI, **Then** a link to the referrer is shown.

---

### User Story 4 - Incrementally maintain the list over time (Priority: P2)

The website keeps changing: pages gain new videos and lose old ones. The
archivist regenerates the list file and re-runs the regular archive update.
They expect new videos to be fetched, removed entries to disappear from the
local playlist without losing the already-archived videos, and unavailable
videos not to be retried on every run.

**Why this priority**: Archives are maintained on a schedule; a one-shot
import that does not update cleanly would force manual cleanup.

**Independent Test**: Back up a list of three videos, remove one entry and add
a new one, run an update, and confirm the playlist now lists the new set in
order, the removed video is still in the archive, and only the new video was
downloaded.

**Acceptance Scenarios**:

1. **Given** an archived video list source, **When** a new entry is added and
   the update is re-run, **Then** only the new video is fetched and it appears
   in the local playlist at its listed position.
2. **Given** an archived video list source, **When** an entry is removed and
   the update is re-run, **Then** the video disappears from the local playlist
   but remains in the archive (as happens when a video is removed from a
   YouTube playlist).
3. **Given** a listed video that is private, deleted or otherwise unavailable,
   **When** the backup runs, **Then** the video is recorded as unavailable the
   same way as for playlists, the run continues with the other entries, and
   later runs do not retry the video on every run.
4. **Given** the existing update modes that refresh social data, comments or
   captions, **When** the user runs one of those modes, **Then** it covers the
   listed videos the same way it covers channel or playlist videos.
5. **Given** a `limit` or date filter configured for the source, **When** the
   backup runs, **Then** the filter applies to the list in its order, the same
   way as for a playlist (`limit` N means the first N entries).

---

### User Story 5 - Back up individual videos ad hoc from the command line (Priority: P2)

A user wants to quickly archive one or a few individual videos, or a list
from a file (or standard input), without first editing the configuration. They
may mix channels, playlists and individual videos in one command, and may want
the individual videos grouped under a named local playlist.

Today, passing a single-video link to the backup command is misinterpreted as
a channel, and a watch link that carries a playlist parameter silently backs up
the whole playlist. Both must be fixed.

**Why this priority**: This removes a real trap in the current command-line
behavior and makes quick one-off archiving possible, but the configuration
source (User Story 1) covers the maintained-archive use case.

**Independent Test**: Run the backup command with one watch link that carries
a playlist parameter, and confirm that only that one video is archived and that
a warning explains how to back up the whole playlist.

**Acceptance Scenarios**:

1. **Given** a single-video link, **When** the user runs the backup command
   with it, **Then** exactly that video is archived (not treated as a channel).
2. **Given** a watch link that also carries a playlist parameter, **When** the
   user runs the backup command with it, **Then** only that video is archived,
   and a warning points to the playlist link form for backing up the whole
   playlist.
3. **Given** a mix of channel, playlist and video links in one command,
   **When** the backup runs, **Then** each link is handled according to its
   kind, using the same classification everywhere in the tool (command line
   and configuration).
4. **Given** a list file (or `-` for standard input) passed via an option,
   **When** the backup runs, **Then** its entries are read with the same rules
   as a configured list file.
5. **Given** the user requests grouping under a named local playlist, **When**
   the backup runs, **Then** the ad-hoc videos are presented as a local
   playlist with that name. When a list file is given and no name is given,
   the playlist is named after the list file. When only bare video links are
   given and no name is given, the videos are archived without a local
   playlist.

---

### User Story 6 - Use a video list as a member of a multi-channel collection (Priority: P3)

A user who manages a multi-channel collection (feature 003) wants a video list
to be one of the collection's members, next to channel members, so that it is
backed up, aggregated and shown in the collection's web UI like any other
member.

**Why this priority**: This is useful for consistency with 003, but a
stand-alone archive covers the motivating use case.

**Independent Test**: Add a video list member to a collection, run the
collection backup and aggregation, and confirm that the list's videos and its
local playlist appear in the collection-level summaries and web UI.

**Acceptance Scenarios**:

1. **Given** a collection, **When** the user adds a video list member, **Then**
   it is backed up and aggregated the same way as a channel member.

---

### Edge Cases

- **Video already covered by another source**: A listed video that is also
  archived through a channel or playlist source in the same archive is not
  downloaded again. It is stored once, and the local playlist references it.
- **Empty list** (after comments, blank lines and invalid entries are
  removed): The source is reported as empty. The run does not fail, and an
  existing local playlist for it becomes empty. Its archived videos stay.
- **List file missing or unreadable**: The source fails with a clear error
  naming the path. Other sources in the same run continue.
- **Tabular list file without a `url` column**: This is rejected with a clear
  error. It is not treated as a plain list.
- **Two sources with the same name**, or a local playlist name that clashes
  with an existing YouTube playlist's directory: This is rejected at
  configuration load / before any download, with an error naming both
  sources.
- **Name unsafe as a directory name** (path separators, `..`, empty): This is
  rejected at configuration load.
- **Channel or playlist URL placed in a video list**: The entry is reported as
  invalid and skipped. A video list holds only individual videos.
- **Live stream or premiere that has not happened yet**: It is handled as for
  channel and playlist backups.
- **Referrer or annotation values containing tabs or newlines** in inline
  entries: These are stored faithfully and do not corrupt summary files.
- **Very large lists** (thousands of entries): These are processed without
  re-reading metadata for already-archived videos on every run.
- **Uploader channels**: Each listed video's uploader channel is recorded so
  the UI can group by channel, but that channel is not backed up.

## Requirements *(mandatory)*

### Functional Requirements

**Source definition**

- **FR-001**: The archive configuration MUST support a third source type,
  "video list" (configured as `type = "videos"`), next to channel and
  playlist sources.
- **FR-002**: A video list source MUST have a `name`, which becomes the
  identifier of its local playlist in the archive and UI. It MAY have a
  human-readable `title` and MAY be disabled with the same `enabled` flag as
  other sources.
- **FR-003**: A video list source MUST accept entries inline, from a list file
  whose path is relative to the archive root, or both. At least one of them
  MUST be present.
- **FR-004**: List files MUST be accepted in two formats: a plain format (one
  reference per line; `#` comments and blank lines ignored) and a tab-separated
  format with a header row. The tab-separated format requires a `url` column
  and supports an optional `referrer` column and any number of other columns.
- **FR-005**: The tool MUST decide the list file format without any extra
  setting from the user. A header row containing a `url` column means
  tab-separated. Anything else is plain.

**Reference recognition**

- **FR-006**: The system MUST resolve every supported video reference form
  (see User Story 2, scenario 1) to a canonical 11-character video identifier
  and reject anything else with a message that says where the entry came from.
- **FR-007**: The system MUST preserve list order, keep only the first
  occurrence of a duplicated video, and merge the referrers and annotations of
  all its occurrences into that first entry.
- **FR-008**: The system MUST keep any start offset given in an entry and make
  it available to the web UI.
- **FR-009**: The system MUST use one shared URL classification (channel,
  playlist, or single video) everywhere in the tool. A watch link that carries
  a playlist parameter MUST be classified as a single video and produce a
  warning that points to the playlist form.

**Storage and presentation**

- **FR-010**: Listed videos MUST be stored in the archive's regular video tree,
  using the archive's configured video path layout. Every stored video is
  archived as fully as a channel-sourced video (metadata, captions, comments,
  thumbnails, and media according to the archive's settings).
- **FR-011**: Each video list source MUST be materialized as a local playlist
  in the archive's playlist area, using the same structure as YouTube
  playlists. Its metadata MUST mark it as local, MUST carry no YouTube
  playlist identifier, and MUST record where its entries came from (inline
  and/or list file path).
- **FR-012**: Each entry of a local playlist MUST record its referrers (zero or
  more), its annotations, and its start offset, if any.
- **FR-013**: The uploader channel of every listed video MUST be recorded in
  the archive's per-video and summary metadata. The uploader channel itself
  MUST NOT be backed up as a result.
- **FR-014**: The web UI MUST list local playlists together with other
  playlists, mark them as local, and show a "referenced from" link for entries
  that have a referrer.

**Incremental behavior**

- **FR-015**: Every backup or update run MUST re-read the list. It MUST fetch
  only entries that are not already archived, and MUST rebuild the local
  playlist to match the current list.
- **FR-016**: An entry removed from the list MUST be removed from the local
  playlist. The archived video MUST be kept.
- **FR-017**: Unavailable videos MUST be recorded and skipped the same way as
  unavailable playlist videos. A single unavailable video MUST NOT abort the
  run.
- **FR-018**: The existing update modes (for example, refreshing social
  counts, comments or captions) and the existing `limit` and date filters
  MUST apply to video list sources, with order-based semantics like those for
  playlists.
- **FR-019**: A video already archived through any other source in the same
  archive MUST NOT be fetched again because of a video list. The local
  playlist references the existing copy.

**Command line**

- **FR-020**: The backup command MUST accept any number of links in one
  invocation, mixing channels, playlists and individual videos.
- **FR-021**: The backup command MUST accept a list file, or `-` for standard
  input, parsed with the same rules as configured list files.
- **FR-022**: The backup command MUST allow ad-hoc videos to be grouped under
  a named local playlist. When a list file is given without a name, the
  playlist name defaults to the file's base name. Bare video links given
  without a name are archived without a local playlist.
- **FR-023**: An ad-hoc command-line backup MUST NOT modify the archive's
  configuration file. To make the videos part of scheduled updates, the user
  adds a video list source to the configuration.

**Validation**

- **FR-024**: Configuration loading MUST reject duplicate source names, names
  that are unsafe as directory names, and local playlist names that clash
  with existing playlist directories, before anything is downloaded.

**Collections (003)**

- **FR-025**: A video list MUST be usable as a member of a multi-channel
  collection, and its videos and local playlist MUST be included in
  collection-level aggregation.

### Key Entities *(include if feature involves data)*

- **Video List Source**: A user-declared set of videos to archive. It has a
  name, an optional title, an enabled flag, inline entries and/or a list file
  path, and optional filters. It is the counterpart of channel and playlist
  sources.
- **List Entry**: One requested video. It has the original reference as
  written, the canonical video identifier, an optional start offset, zero or
  more referrers, free-form annotations (extra columns), and its origin (file
  and line, or inline position).
- **Local Playlist**: The archived representation of a Video List Source. It
  is structured like a YouTube playlist, with the entries in order, but is
  marked as local, has no YouTube identifier, and records its origin (inline,
  list file path). Entries carry the provenance fields of their List Entry.
- **Video**: The existing archived video entity, unchanged except that it can
  now be reached from a Local Playlist. Its uploader channel is recorded
  without that channel being archived.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can archive an arbitrary set of videos from different
  channels, and see it as one ordered, titled group in the web UI, by adding a
  single source definition and running one backup, with no manual file
  editing inside the archive.
- **SC-002**: 100% of the reference forms listed in User Story 2 resolve to
  the correct video, and 100% of invalid entries are reported with their
  location without stopping the run.
- **SC-003**: Re-running an update on an unchanged list downloads no media and
  leaves the archive content unchanged, except for metadata refreshes that
  the chosen update mode explicitly requests.
- **SC-004**: After an entry is added to and another removed from a list, one
  update run fetches exactly one new video, changes the local playlist to
  match the list, and loses no previously archived video.
- **SC-005**: Passing a single-video link to the backup command archives
  exactly one video in 100% of cases. This includes links that carry a
  playlist parameter, which today back up an entire playlist.
- **SC-006**: Every listed entry that has a referrer shows a working
  "referenced from" link in the web UI.
- **SC-007**: A list with a mix of available and unavailable videos completes
  in one run. Unavailable videos are not retried by the next routine update.

## Assumptions

- **Source type name**: `type = "videos"` is used, not `type = "list"`, which
  matches the issue title and the names `channel`/`playlist`, which describe
  what is being archived. (Open question 1 in the issue.)
- **Default ad-hoc grouping**: A local playlist is created by default only
  when a list file is given, and is then named after the file. Bare links
  without an explicit name are archived without grouping. (Open question 2 in
  the issue.)
- **Cross-references**: A video list may reference videos that other sources
  already cover. They are deduplicated by video identifier and never fetched
  twice. (Open question 3 in the issue, answered "yes" as the issue proposes.)
- The inline and file entries of one source are combined in this order:
  inline first, then file.
- The list file is expected to be committed to the archive so that the
  archive records what was requested. The tool reads the file but does not
  write it.
- Unavailable videos are handled with the archive's existing mechanism for
  playlists. This feature does not add a new retry policy.
- Only YouTube video references are in scope. Other video hosts are out of
  scope.
- Scraping websites to produce lists is out of scope. The feature consumes
  lists produced by any external means.

## Dependencies

- Feature 001 (YouTube backup): video archiving, playlist structure, update
  modes, filters, unavailable-video tracking.
- Feature 003 (multi-channel collections): collection membership and
  aggregation, for User Story 6.
- The `@AnnexTubeTesting` channel's videos, used as integration test fixtures.
