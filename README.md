# ftp-nv

FTP, the File Transfer Protocol, moves files between a client and a
server. It is specified in [RFC 959](https://www.rfc-editor.org/rfc/rfc959)
and extended by RFC 2389 (FEAT and OPTS), RFC 2428 (EPRT and EPSV),
RFC 3659 (MDTM, SIZE, REST and MLSD) and RFC 4217 (FTP over TLS). This
package brings the client side of FTP to novo-lang.

## What it is

FTP uses two connections. The *control connection* carries commands and
replies as lines of text. Each file and each directory listing travels on
a separate *data connection*, opened for that one transfer. A command such
as `RETR` is sent on the control connection; the server answers `150` to
say the data is about to move, the bytes arrive or leave on the data
connection, and the server answers `226` on the control connection when
the transfer is complete.

The package has no socket and no clock. A program that uses it owns both
connections, and the TLS around them. Three things follow.

**The control connection is a step.** An `FtpClient` is a value the caller
holds in a `var`. `send` takes a command and answers the bytes to write.
`feed` takes the bytes that arrived and answers a list of events. A read
may hold half a reply or several; the session keeps what is not yet a
whole line.

**The data connection is the caller's.** In passive mode the caller sends
`PASV` or `EPSV`, receives an `FtpPassive` event with the address, connects
to it, and then sends `RETR`, `STOR`, `LIST` or `MLSD`. The package reads
the replies on the control connection and parses the bytes of a listing
when the caller hands them over. It never touches the data stream.

**TLS is the caller's.** `AUTH TLS` is a command like the others. When the
server answers `234` the session emits `FtpTlsReady`, the caller starts TLS
on the control connection and calls `tls_started`.
[tls-nv](https://novo-lang.org/packages/tls-nv) provides the TLS.

## Install

```
novo pkg add ftp-nv
```

## Example

This logs in, asks for a passive data connection, and reads a directory
listing. The server's replies are written out in the program so that it
runs anywhere. In a program that talks to a server, the bytes passed to
`feed` are what the control socket read, and the bytes `send` answers are
what it writes.

```novo
use std.bytes
use ftplist
use ftpsession

fn bytes_of(s: Str) -> [u8]
    bytes.to_byte_list(bytes.from_str(s))

fn main() [io]
    var c = ftpsession.new_client()
    let _g = ftpsession.feed(var c, bytes_of("220 (vsFTPd 3.0.3)\r\n"))

    // Log in.
    let _u = ftpsession.send(var c, FtpCmdUser("anonymous"))
    let _a = ftpsession.feed(var c, bytes_of("331 Please specify the password.\r\n"))
    let _p = ftpsession.send(var c, FtpCmdPass("guest@"))
    let _b = ftpsession.feed(var c, bytes_of("230 Login successful.\r\n"))
    println(ftpsession.phase_name(c.phase))   // logged in

    // Ask for a passive data connection.
    let _s = ftpsession.send(var c, FtpCmdPasv)
    let events = ftpsession.feed(var c, bytes_of("227 Entering Passive Mode (127,0,0,1,193,115).\r\n"))
    match events[0]
        FtpPassive(host, port) => println("connect to ${host}:${port}")   // connect to 127.0.0.1:49523
        _ => println("no address")

    // LIST: the replies come on the control connection, the lines on the
    // data connection.
    let _l = ftpsession.send(var c, FtpCmdList(""))
    let _c = ftpsession.feed(var c, bytes_of("150 Here comes the directory listing.\r\n226 Directory send OK.\r\n"))
    let data = bytes_of("drwxr-xr-x    2 ftp      ftp          4096 Jan 01  2024 pub\r\n-rw-r--r--    1 ftp      ftp            12 Feb 29 12:30 notes.txt\r\n")
    let listing = ftplist.parse_listing(data)
    println("${list.len(listing.entries)} entries, first ${listing.entries[0].name}")   // 2 entries, first pub
```

Build and test with `novo pkg build` and `novo test tests`.

## What the package contains

| Module | Contents |
| --- | --- |
| `ftpwire` | One line at a time: a reply line and the end of a multi-line reply, the command encoder, and the readers for the texts of 227, 229, 257, SIZE, MDTM and FEAT replies and the builders for PORT and EPRT arguments. |
| `ftplist` | Directory entries: MLSD and MLST facts, and the Unix and DOS formats of LIST. |
| `ftpsession` | The control connection as a step: `send`, `feed`, the phases, the commands and the events. |
| `ftperr` | The refusals a line, a command or a call can meet, each with a sentence. |

Every `pub` declaration is documented where it is declared, and
[the package's page](https://novo-lang.org/packages/ftp-nv) is that
reference.

## The commands and their events

| Command | Allowed | Event on success |
| --- | --- | --- |
| `FtpCmdUser`, `FtpCmdPass`, `FtpCmdAcct` | after the greeting | `FtpIntermediate` for a 3yz, `FtpLoggedIn` for `230` |
| `FtpCmdAuth`, `FtpCmdPbsz`, `FtpCmdProt` | after the greeting | `FtpTlsReady` for `234`, else `FtpOk` |
| `FtpCmdFeat`, `FtpCmdOpts`, `FtpCmdSyst`, `FtpCmdNoop` | after the greeting | `FtpFeatures` for FEAT, else `FtpOk` |
| `FtpCmdQuit` | after the greeting | `FtpBye` |
| `FtpCmdPwd`, `FtpCmdMkd` | after login | `FtpWorkingDir` |
| `FtpCmdCwd`, `FtpCmdCdup`, `FtpCmdRmd`, `FtpCmdDele`, `FtpCmdRnto` | after login | `FtpOk` |
| `FtpCmdRnfr`, `FtpCmdRest` | after login | `FtpIntermediate` |
| `FtpCmdType`, `FtpCmdMode`, `FtpCmdStru`, `FtpCmdPort`, `FtpCmdEprt` | after login | `FtpOk` |
| `FtpCmdPasv`, `FtpCmdEpsv` | after login | `FtpPassive` |
| `FtpCmdRetr`, `FtpCmdStor`, `FtpCmdAppe`, `FtpCmdList`, `FtpCmdNlst`, `FtpCmdMlsd` | after login | `FtpPreliminary` for the `1yz`, `FtpOk` for the `226` |
| `FtpCmdSize`, `FtpCmdMdtm` | after login | `FtpSize`, `FtpModified` |
| `FtpCmdMlst` | after login | `FtpFacts` |

A `4yz` or `5yz` reply is an `FtpRefused` event for any command.

## The rules a user needs

1. **One command at a time.** A reply belongs to the command sent last.
   `send` refuses a second command until the first is answered, with
   `FtpBusy`. A transfer command is unanswered until its final reply,
   which may arrive after the data has ended.
2. **A `1yz` reply does not settle a command.** `150` and `125` are
   `FtpPreliminary`. The final reply, `226` for a transfer, follows on the
   control connection (RFC 959 section 4.1.1).
3. **Set the type before a transfer.** The data representation defaults to
   ASCII. `FtpCmdType("I")` before `RETR` or `STOR` sends bytes unchanged.
   The package does not translate line endings for type A.
4. **A multi-line reply is one event.** It starts with `123-`, ends at the
   first line that begins `123 ` and is delivered whole. The text is the
   lines joined with a newline, the lines between kept as the server sent
   them (RFC 959 section 4.2).
5. **`421` closes the session.** The server may send it at any time. The
   session ends after it (RFC 959 section 4.2).
6. **EPSV carries a port only.** The `FtpPassive` event for `229` has an
   empty `host`. Connect to the host the control connection went to
   (RFC 2428 section 3).
7. **Prefer EPSV, and MLSD.** PASV has no IPv6 form and some servers return
   an internal address in it. LIST has no defined format; `parse_listing`
   reads the Unix and DOS formats as a best effort and counts the lines it
   skipped. MLSD is defined to be read by a program.
8. **A transfer is resumed with REST.** `FtpCmdRest` answers `350`, and
   the next `FtpCmdRetr` or `FtpCmdStor` starts at that offset
   (RFC 3659 section 5).
9. **Capabilities go stale.** The session forgets the FEAT list after
   `tls_started`. Ask again.
10. **A path is sent as it is.** An empty path, or one with CR, LF, NUL or
    the byte 0xFF, is refused before anything is sent. The byte 0xFF would
    need doubling for Telnet, which this package does not do.
11. **Dates in a Unix or DOS listing are text.** They have no time zone and,
    for recent files, no year. MDTM and the MLSD `modify` fact are UTC.

## What is not included

- **Sockets, TLS and timers.** The caller moves the bytes.
- **The data connection.** The caller reads and writes it, and for a
  protected data connection (`PROT P`) wraps it in TLS.
- **Active mode's listening socket.** The package builds `PORT` and `EPRT`;
  the caller listens.
- **Block and compressed modes, and record structure.** `MODE S` and
  `STRU F` only, as nearly every server supports.
- **ASCII translation.** Type A is sent and the bytes are the caller's.
- **ABOR, REIN, SITE, ALLO, STOU, HELP, STAT and the Telnet interrupt
  sequence.**
- **GSSAPI, ADAT and other security mechanisms** than TLS.
- **The server side.**

## Related packages

- [tls-nv](https://novo-lang.org/packages/tls-nv) is the TLS a caller
  starts after `AUTH TLS`, and wraps the data connection with.
- [tftp-nv](https://novo-lang.org/packages/tftp-nv) is the lock-step
  datagram protocol a bootloader uses where there is no TCP.
- [socks5-nv](https://novo-lang.org/packages/socks5-nv) is another
  protocol written as a step the caller drives.

## Tests

The suite is three files under `tests/`. What they assert comes from the
RFCs, and the reply texts are the ones vsftpd and ProFTPD write.

`tests/ftp_wire_tests.nv` holds the lines one at a time: the reply lines
and the end of a multi-line reply, the six numbers of RFC 959's PASV
reply, RFC 2428's `(|||6446|)` and its two EPRT examples, quoted paths
with doubled quotes, SIZE, the MDTM example of RFC 3659 and the FEAT reply
of RFC 2389.

`tests/ftp_list_tests.nv` holds the MLSD facts in the form of RFC 3659
section 7.7, and Unix and DOS listing lines, including a name with spaces,
a link, a missing group column and the lines that are refused.

`tests/ftp_session_tests.nv` plays the server by hand: the greeting in
each shape, login with a password, an account and neither, directories,
transfers with the preliminary and the final reply in one read and in two,
REST, passive and active mode, FEAT, MLST, `AUTH TLS`, `421`, and each way
a server or a caller can break the rules above.

`bash tests/coverage.sh` merges the coverage of the suites and reports the
lines of `src/`.

## Licence

Apache-2.0. See `LICENSE`.
