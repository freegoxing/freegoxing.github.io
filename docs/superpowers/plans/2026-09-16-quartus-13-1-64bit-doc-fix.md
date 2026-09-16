# Quartus II 13.1 64-bit Documentation Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Correct the Quartus II 13.1 article so it documents the verified 64-bit launch path and no longer instructs readers to install unnecessary i386 dependencies.

**Architecture:** Keep the existing single-article structure, but make every architecture-sensitive instruction consistently target Quartus's `linux64` executable. Use an unpacked amd64 `libpng12` in a user-local compatibility directory and select 64-bit mode explicitly in the wrapper.

**Tech Stack:** Hexo Markdown, Bash command examples, Debian package tools

## Global Constraints

- Modify only `source/_posts/linux/linux-安装-quartus-13.1.md` plus this implementation plan.
- Preserve the article's Chinese explanatory style.
- Do not require i386 multiarch, `libsm6:i386`, or `libice6:i386`.
- Launch Quartus through `bin/quartus --64bit`.
- Do not install the legacy Debian package system-wide; extract only the required shared library.

---

### Task 1: Replace the incorrect 32-bit workflow

**Files:**
- Modify: `source/_posts/linux/linux-安装-quartus-13.1.md`

**Interfaces:**
- Consumes: Quartus installation rooted at `/opt/altera/13.1/quartus` and the archived Debian amd64 `libpng12-0` package.
- Produces: A copy-pasteable 64-bit compatibility-library workflow and `quartus-13.1` wrapper.

- [x] **Step 1: Update the article metadata and explanation**

  Change the title, description, introduction, installer warning explanation, and architecture section to state that `--64bit` selects `linux64/quartus`; do not claim that the whole application requires i386 libraries.

- [x] **Step 2: Replace the package extraction commands**

  Use `libpng12-0_1.2.50-2+deb8u3_amd64.deb`, extract it into `/tmp/libpng12-amd64`, copy `lib/x86_64-linux-gnu/libpng12.so.0*` into `~/.local/lib/quartus-13.1-compat`, and verify that the library is x86-64.

- [x] **Step 3: Correct dependency inspection and launching**

  Run `ldd` against `$QUARTUS_ROOT/linux64/quartus`, remove the i386 installation sections, and pass `--64bit` both in the one-off launch command and in the persistent wrapper.

- [x] **Step 4: Validate the edited article**

  Run:

  ```bash
  rg -n 'i386|32 位|32-bit|linux/quartus|libpng12-i386|libsm6|libice6' source/_posts/linux/linux-安装-quartus-13.1.md
  ```

  Expected: no obsolete instruction remains; any remaining 32-bit wording only explains why the installer's generic warning is not followed.

  Run:

  ```bash
  rg -n -- '--64bit|linux64/quartus|amd64|x86_64-linux-gnu' source/_posts/linux/linux-安装-quartus-13.1.md
  ```

  Expected: all four parts of the 64-bit workflow are present.

  Run:

  ```bash
  git diff --check
  ```

  Expected: exit status 0 with no whitespace errors.
