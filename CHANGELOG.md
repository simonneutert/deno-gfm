# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

This changelog starts with changes made after version 0.12.0.

## [v0.13.0] - 2026-09-07

### Changed

- `strip()` now uses image alt text for both Markdown and raw HTML images;
  Markdown image titles are no longer emitted instead of alt text, and images
  without alt text contribute no plain text
  ([#113](https://github.com/denoland/deno-gfm/issues/113)).

### Fixed

- Preserve explicit ordered-list start values during HTML sanitization
  ([#121](https://github.com/denoland/deno-gfm/issues/121)).
- Keep KaTeX rendering and the bundled stylesheet on the same dependency
  version, preventing generated-output drift.

[Upstream]: https://github.com/denoland/deno-gfm/compare/0.12.0...HEAD
