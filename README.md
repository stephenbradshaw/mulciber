# Mulciber - compiler service for Beacon Object Files

`mulciber` is a small HTTP service that compiles BOF (Beacon Object File)
source code into relocatable object files: Windows COFF objects for the
Cobalt Strike dialect and ELF relocatables for the Dark-Agent dialect
(Linux/macOS). It supports compilation with an obfuscating clang
(`-sobf/-sub/-split/-bcf/-fla` passes) at selectable levels.

This repository is being scaffolded; see `AGENTS.md` for the build brief
and integration contract.
