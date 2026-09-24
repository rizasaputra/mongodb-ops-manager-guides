# Product Overview

This project is a collection of README markdown files that serve as tutorials for setting up and testing MongoDB Ops Manager.

## Purpose

- Provide clear, step-by-step guides for installing, configuring, and testing MongoDB Ops Manager.
- Act as a reference that a reader can follow from top to bottom to get a working setup.
- Serve as a self-contained environment setup guide so the reader does not need to go
  back and forth between MongoDB documentation pages while testing.

## What is being tested

This project documents an end-to-end set of Ops Manager test scenarios: provisioning
MongoDB from scratch, no-downtime automation changes, monitoring and Performance
Advisor on an existing production system, backup to a Dell appliance and restore,
native backup, database-level auditing, and finally safely uninstalling Ops Manager
while keeping production running. See the testing-environment steering file for the
full environment baseline and scenario list.

## Audience

- Engineers and operators who need to stand up or test an Ops Manager environment.
- Readers may be new to Ops Manager, so guides should not assume deep prior knowledge.

## Scope

- This is a documentation project. There is no application source code to build, run, or deploy.
- Deliverables are markdown files (`.md`): tutorials and setup/testing guides, plus a
  `README.md` index.
- A single static `index.html` (with a CDN markdown renderer) is included only to
  preview the guides nicely in a browser. It is a viewer for the docs, not application
  code, so the project is still documentation-only in spirit. Serve it over HTTP
  (`python3 -m http.server`) since it fetches the markdown files.
- Keep content practical and reproducible: every instruction should be something the reader can actually run or verify.
