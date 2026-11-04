# Backup

Backup is a local Java 21 desktop backup tool built with Swing.
The interface contains Backup, Restore, Automation and Network tabs; no external service is needed.

## Features

- Recursive file discovery, including symbolic and hard links where supported
- Special files: pipes, character/block devices and unix sockets are detected on POSIX
  platforms, recorded in manifests and recreated on restore when the platform allows
- File selection by relative path/name glob, extension, modification time, byte size and owner
- File metadata capture and best-effort restoration with warnings
- ZIP and GZIP archives, with optional AES-GCM encryption
- Manifest-based incremental backups grouped as one full backup plus up to nine increments
- Optional retention of complete groups, isolated by backup task
- Scheduled backups with `ScheduledExecutorService` and real-time backups with `WatchService`
- Archive restore with path traversal and destination symlink protection
- Compatibility with existing version 1 ZIP, GZIP and encrypted archives
- Cloud mode: a built-in TLS receiver with PBKDF2-protected accounts, per-user storage,
  manifest metadata management, incremental uploads and server-side group retention

## Run and test

Requirements: JDK 21 or newer; Maven 3.9 or newer for the standard build/test flow.

```bash
mvn test
mvn clean package
java -jar target/backup-1.0-SNAPSHOT.jar
```

The repository does not include a Maven wrapper. Maven may not be on the shell PATH. IntelliJ IDEA can bundle Maven; on this
macOS environment it is available at
`/Applications/IntelliJ IDEA.app/Contents/plugins/maven-plugin/lib/maven3/bin/mvn`.
Use `mvn test` as the standard test command (or invoke an installed Maven by its
absolute path). To compile and run without Maven:

```bash
mkdir -p target/classes
javac --release 21 -d target/classes $(find src/main/java -name '*.java' -print)
java -cp target/classes com.backup.BackupApplication
```

Direct application compilation does not run the JUnit tests. See
[本地功能验收](doc/本地功能验收.md) for disposable local demonstrations and the
platform-dependent checks to record separately.

## Selection and retention

Path globs use source-relative paths with `/`; name globs match only the basename.
Separate alternatives with `;` (OR). Different criteria combine with AND, and path
exclusions take precedence. Globs are case sensitive; extensions ignore case and
an optional leading dot; owner names are compared exactly but without case
sensitivity, and a blank recorded owner never matches a nonempty owner criterion.
Modification-time and byte-size bounds are inclusive.
With extensions empty, the extensionless option unchecked means no type restriction;
checked means only extensionless files. With extensions set, checking it additionally
includes extensionless files. Dotfiles and names ending in a dot are extensionless.

A group contains sequence 1 (full) through at most sequence 10 (nine increments).
The eleventh backup starts another full group. `keepGroups=0` disables deletion;
it still starts a new group after ten snapshots. Positive values retain the newest
N whole valid groups for that task, while protecting the latest snapshot's group.
Unknown files, invalid chains and unsafe catalog records are preserved with warnings.
Retention never infers ownership from a ZIP/GZIP extension.

A task is identified by the canonical source path, backup name and normalized filter.
The destination separates its stored baseline. A new `BackupService` or restarted
application begins with a full backup. Changing compression, encryption, an active
encryption password, or switching retention between disabled/enabled starts a full
group. Changing a positive retention count alone does not force a full backup.

Archives use `<name>-<full|incremental>-<timestamp>-<UUID><extension>[.enc]`.
Each archive has `.manifest.properties` and `.snapshot.properties` sidecars.
The latter records group ownership and parent relationships for safe retention.
The legacy `.backpack-manifest.properties` is still written for inspection, but is
not loaded to resume another process's incremental baseline.

## Network mode

The Network tab turns the tool into a small cloud backup service:

- **Server**: pick a storage directory and a port, press *Start server*. The first start
  provisions a self-signed TLS certificate with the JDK's `keytool` and stores it in
  `server-keystore.p12` inside the storage directory. Add accounts with a username and
  password; only salted PBKDF2 verifiers are persisted.
- **Client**: enter host, port and account, then back up a source directory with the same
  compression/encryption/incremental options as local backups. Archives travel over TLS,
  manifests and snapshot catalogs are stored per user on the server, and increments are
  computed against the server-side manifest chain.
- **Restore**: list snapshots, select one, and restore it into a local directory. Incremental
  chains still restore in order (full first, then increments), exactly like local archives.
- **Retention and cleanup**: `keep groups` prunes whole old groups on the server after each
  successful upload; *Delete selected group* removes one older group and always protects the
  newest snapshot.

The client pins the server identity by trusting the server's keystore file, so both sides can
run on the same machine or on a LAN host without a certificate authority.

## Restore

Restore a full archive, then each incremental archive in sequence into the same
destination. The interface restores one selected archive at a time; it does not
automatically assemble a chain. Incremental restore overlays changed entries and
does not delete files removed from the source. Keep the whole chain when copying
an incremental backup elsewhere. Access/creation times recorded in an incremental
manifest are reapplied to unchanged files only after verifying the restored baseline;
this does not require storing their contents again. Missing or different baseline files
produce warnings instead of receiving unrelated snapshot metadata.

Restoration recreates supported links and attempts permissions, owner/group,
modification/access/creation times. Unsupported filesystem operations or insufficient
privileges produce warnings; these capabilities vary by platform. For safety,
owner/group/permission changes are skipped for symbolic links with an explicit
warning because some platforms can follow the link despite no-follow options.
Destination symlinks are refused. Regular files are replaced atomically so an existing hard link
outside the restore directory does not have its contents overwritten.


## Presentation handoff

See [Claude Code PPT 制作说明](doc/Claude-Code-PPT制作说明.md) for the verified project facts, two-minute slide plan, screenshots, and eight-minute demonstration flow.
