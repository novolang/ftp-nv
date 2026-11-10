# Changelog

All notable changes to ftp-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0

The first release.

### Added

- **`ftpwire`**, the lines of FTP's control connection. Reply lines and
  the end of a multi-line reply (RFC 959 section 4.2), the command
  encoder, the texts of the 227 reply (best effort, with or without
  parentheses) and the 229 reply (RFC 2428), the PORT and EPRT arguments,
  the quoted path of a 257 reply, SIZE and MDTM replies with fractions of a
  second (RFC 3659), and FEAT lists with `has_feature` and `feature_arg`
  (RFC 2389).
- **`ftplist`**, directory entries from MLSD and MLST facts (RFC 3659
  section 7), from the Unix long format and from the Windows IIS format,
  with a count of the lines no format read. The Unix and DOS readers are a
  best effort and say so.
- **`ftpsession`**, the control connection as a step: `send` and `feed`
  over an `FtpClient`, thirty-six commands, four phases, one command in
  flight, preliminary replies that leave a command unsettled, multi-line
  replies delivered whole, `421` ending the session, FEAT kept and forgotten
  after TLS, and `AUTH TLS`, `PBSZ` and `PROT` as commands the caller
  follows with its own TLS (RFC 4217).
- **`ftperr`**, the refusals, each with a sentence.
- Three test suites: `tests/ftp_wire_tests.nv`, `tests/ftp_list_tests.nv`
  and `tests/ftp_session_tests.nv`. `tests/coverage.sh` merges the
  coverage: 100% of `src/`.

### Not included

- Sockets, TLS and timers, and the data connection itself. The caller
  moves the bytes.
- Block and compressed modes, record structure, ASCII translation, ABOR,
  REIN, SITE, ALLO, STOU, HELP, STAT, the Telnet interrupt sequence,
  GSSAPI and the server side.
