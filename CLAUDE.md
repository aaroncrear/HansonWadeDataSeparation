# HansonWadeDataSeparation

Salesforce metadata for the HansonWade org (`unpackaged/main/default/...`), deployed via Gearset. Work happens on feature branches, one branch per piece of work.

## Release Notes (required on every branch)

Every feature branch must include a release notes file at the root of the branch, built from `Release_Notes_Template.docx` (also at the repo root). Do not invent a different format — copy the template and fill it in.

**File name:** `<Branch Name> - Release Notes.docx`, where `<Branch Name>` is the actual branch name (`/` replaced with `-`, since `/` isn't valid in a filename). Example: for branch `claude/htt-42155-jsmo8z`, the file is `claude-htt-42155-jsmo8z - Release Notes.docx`. Do not use a generic name like `Release_Notes.docx` — each branch's file must be individually named so it's identifiable on sight.

Sections, in order:

1. **`<Branch Name>` – Release Notes** (Heading 1) — replace `<Branch Name>` with the actual branch name.
2. **Requirements** (Heading 2) — the business requirement(s): what problem existed and why it needed to be fixed/built.
3. **Release Notes** (Heading 2) — what was built and why it was built the way it was. Prefer a bulleted list of the individual changes.
4. **Acceptance Criteria** (Heading 2) — step-by-step instructions to test the build, in enough detail that someone unfamiliar with the change can validate it.
5. **Post Deployment Items** (Heading 2) — any manual steps needed after the metadata deploys: activating flows, deleting/deactivating superseded components, data loads, permission changes, etc. Call out clearly what breaks (and how) if a step is skipped.
6. **Component Manifest** (Heading 2)
   - **Github Branch** (Heading 3) — link to the feature branch.
   - A table listing every component touched: `Component Type | Object | API Name | Label | Created/Updated/Deleted | Description`. Use `TBD - confirm API name` rather than guessing when a component (e.g. a flow or field built directly in-org) hasn't been committed to the repo yet, so it doesn't read as verified when it isn't.

The template's first paragraph is instructional only ("For all Salesforce metadata builds use the template below...") — remove it from the actual release notes file; it should only ever appear in the template itself.
