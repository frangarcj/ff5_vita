# Changelog

## 1.1

- Fix the crash during audio player destruction reported in issue #10 by
  creating the game thread through pthreads, which initializes the thread state
  required by OpenSLES condition waits. Keep the existing 1 MiB stack and Vita
  thread priority.
- Close the file opened by `getFileSize()` after each size query, including error
  paths. Return an error if the file cannot be opened, measured or closed, or if
  its size cannot be represented by the existing API.
- Update the build environment, package the runtime data matching the linked ICU
  version, and provide debug symbols and checksums alongside the VPK.

## 1.0

- Initial release.
