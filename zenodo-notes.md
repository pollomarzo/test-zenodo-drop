# Zenodo deposit notes

Gotchas and follow-ups discovered while wiring the deposit flow. These are
things the metadata mapping table in `claude-plan.md` doesn't capture on its
own.

## Funding lives in `index.md`, not `myst.yml`

In the current micropublication template, funding info sits in the `index.md`
frontmatter (or in a "Funding" section in the body), not in `project.funding`
of `myst.yml`. Our script reads from `myst.yml` and so currently never sees
funding for these papers.

Options when this becomes a priority:
1. Move funding into `project.funding` in the template's `myst.yml` (cleanest;
   also makes funding queryable across papers).
2. Have the script parse `index.md` frontmatter for funding and merge it in.
3. Leave funding off Zenodo entirely (it'll still be on the rendered page).

For now: free-text in the Zenodo description if/when `project.funding` is
present. Nothing else.

## ORCID placeholders wipe author names

Zenodo looks up the ORCID profile when you submit a creator with an ORCID, and
if the lookup fails (which it does for the template's placeholder ORCIDs like
`0000-0000-0000-0001`), Zenodo silently clears the `name` field on that
creator. Real papers with real ORCIDs are fine.

The script defends against this: it skips ORCIDs that don't match the basic
format `\d{4}-\d{4}-\d{4}-\d{3}[\dX]` or that start with `0000-0000-`.

## "Repository URL" field — deferred

Zenodo's UI shows a dedicated "Repository URL" field that looks more
appropriate than `related_identifiers` for the GitHub link. We haven't found
the corresponding API field (the legacy deposit API doesn't expose it; may be
an InvenioRDM custom field). Currently using
`related_identifiers` (`isVersionOf`, scheme `url`) for the GitHub URL; revisit
when convenient.

## "Do you already have a DOI?" prompt

Zenodo prompts maintainers at publish time: "Do you already have a DOI?
Yes/No". We can't suppress it from the API:

- Setting `metadata.doi = <Zenodo-prefixed value>` (e.g. `10.5072/...` or
  `10.5281/...`) is rejected: *"The prefix '10.5072' is managed by Zenodo.
  Please supply an external DOI or select 'No' to have a DOI generated for
  you."* `metadata.doi` is for **external** DOIs only (e.g. an arXiv DOI).
- `prereserve_doi: true` reserves a Zenodo DOI but doesn't suppress the prompt
  in the new InvenioRDM-based UI.

So the maintainer clicks "No, generate one for me" once per release. The
generated version DOI is what we predict (`{prefix}{dep['id']}`), and the
concept DOI is preserved across versions via the newversion mechanism — we
don't need to control either explicitly.

## Sandbox `prereserve_doi.doi` has the wrong prefix

On `sandbox.zenodo.org`, the API response field
`metadata.prereserve_doi.doi` returns DOIs with the production prefix
(`10.5281/...`) even though the actual assigned DOI uses the sandbox prefix
(`10.5072/...`). Don't trust that field; compute as `{prefix}{recid}` from the
deposition `id` / `conceptrecid` instead. The script already does this.

## Concept DOI before first publish

Zenodo only assigns `conceptdoi` at first publish time. We need the concept
DOI in `myst.yml` *before* publish so the PDF built from `main` carries it.
The script derives it from `conceptrecid` (which IS allocated at draft
creation): `concept_doi = "{prefix}{conceptrecid}"`. The post-publish
`conceptdoi` matches this — verified on sandbox draft 495462 (concept
`10.5072/zenodo.495461`, version `10.5072/zenodo.495462`).
