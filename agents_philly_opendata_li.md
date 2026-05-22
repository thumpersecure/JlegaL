# Navigating Philadelphia L&I Open Data — Guide for AI Agents and OSINT Researchers

A reproducible verification protocol for Philadelphia property-compliance records. Written for AI agents acting on behalf of investigative journalists, tenants-rights groups, civic-tech researchers, and anyone trying to confirm what a property's record actually says.

This guide is descriptive, not editorial. Every claim about a data-system behavior was observed during the cross-referencing work that produced the property record for 315–23 N 12th Street (Goldtex Apartments), OPA account 881519440, in May 2026.

---

## 1. Source authority — what to trust for what

Philadelphia publishes property-compliance data through three overlapping interfaces. They are not equally reliable for the same question.

| Source | URL pattern | Authoritative for |
|---|---|---|
| **L&I Eclipse — Property History** | `https://li.phila.gov/property-history/...` | **Current** license status, current case status, business mailing address on file. |
| **L&I Eclipse — eCLIPSE permitting portal** | `https://eclipse.phila.gov/` | Permits, contractor licenses, zoning appeals, and direct case lookup by license/permit number. |
| **Open Data Carto** | `https://phl.carto.com/api/v2/sql?q=...` | **Bulk historical** queries against the L&I Violations dataset. Useful for counting and aggregation; not authoritative for current status. |
| **OpenDataPhilly catalog** | `https://opendataphilly.org/` | Dataset discovery and schema documentation. Not a query endpoint. |

**Rule of thumb:** if the question is *"what is the state of this property right now,"* go to li.phila.gov. If the question is *"what has happened at this property over time,"* go to Carto. Never present a Carto status field as a current state determination without confirming it against Eclipse.

---

## 2. Known data-quality issues to test for

These have all been observed and should be assumed present until ruled out for the specific record being verified.

### 2.1 License-status mirror lag

The Carto open-data feed mirrors L&I license records but can lag by months or years for individual licenses. A license can show as *Active* on Carto while the authoritative Eclipse system shows it as *Expired*. **Always re-pull license status from `li.phila.gov` directly before relying on it.**

Verification pattern:

1. Pull license record from Carto.
2. Note the license number.
3. Open `https://li.phila.gov/property-history/...` for the property.
4. Find the same license number in the rendered table.
5. If the two disagree, the Eclipse value is authoritative.

### 2.2 Truncated property-history exports

The Property History page on `li.phila.gov` may present a violations list that is older than the most recent Carto-API result for the same address. The page does not warn the user about its cutoff date. Always check:

- The most recent violation date on the rendered page.
- The most recent violation date returned by a direct Carto SQL query.
- If they differ by more than one quarter, treat the rendered page as incomplete.

Example query (replace the address):

```sql
SELECT casenumber, casecreateddate, violationdate, violationcode, violationdescription, casestatus
FROM violations
WHERE upper(address) LIKE '%315%N%12TH%'
ORDER BY violationdate DESC
```

### 2.3 Inspector identity is not in the public data

Violation records contain `casenumber`, `violationcode`, `violationdescription`, `casestatus`, and `casepriority`. They do **not** contain an inspector name, badge number, or unit identifier. There is no public mechanism to:

- List all violations issued by a specific inspector.
- Track inspector-level patterns or geographic assignments.
- FOIA "all activity by Inspector X" without knowing X's identity from another source.

If the investigation requires an inspector identity, the only known path is a targeted FOIA to L&I referencing a specific case number and inspection date. Plan accordingly.

### 2.4 Non-standard closure statuses

The `casestatus` field is not binary. Observed values include:

- `COMPLIED` — actual compliance recorded.
- `NULL` — open / no compliance recorded (often rendered as "UNRESOLVED" in UI exports).
- `ERROR` — data-entry anomaly. Not a clean closure.
- `CLOSEDCASE` — administrative closure. Compliance not confirmed.
- `RESOLVE` — resolved through appeal or non-compliance path. Compliance not confirmed.

Aggregate summaries on `li.phila.gov` typically count everything that is not NULL as "closed." Do not propagate that framing. When reporting on a property, break out the status distribution explicitly.

### 2.5 311 complaints are a separate dataset

Tenant complaints filed via Philly311 are not part of the L&I Violations dataset. They live in a separate Carto table and are not surfaced in the Property History rendering. A property can show "no violations" while accumulating 311 complaints. If 311 history is relevant, query it separately:

```sql
SELECT requested_datetime, service_name, status, description
FROM public_cases_fc
WHERE upper(address) LIKE '%315%N%12TH%'
ORDER BY requested_datetime DESC
```

### 2.6 OPA assessment data is a third source

Property valuation, square footage, owner-of-record, and deed history come from the Office of Property Assessment, not L&I. The two systems share an `OPA Account` number but diverge on owner-name normalization. Use OPA for ownership questions; do not infer owner from L&I license records alone.

---

## 3. The minimum-viable verification workflow

For any property-compliance claim, an agent should execute the following before publishing or reporting:

1. **Resolve the property to an OPA account.**
   - Source: `https://atlas.phila.gov/` (search by address).
   - Capture: OPA account number, parcel boundary, current zoning.
2. **Pull live license status from L&I Eclipse.**
   - Source: `https://li.phila.gov/property-history/...`
   - Capture: every license listed, with status, issue date, last renewal, and licensee.
   - Flag any license whose status differs from what Carto reports for the same license number.
3. **Pull full violation history from Carto.**
   - Source: Carto SQL endpoint against the `violations` table.
   - Capture: every record returned, with `casenumber`, `violationdate`, `violationcode`, `violationdescription`, `casestatus`, `casepriority`.
   - Do **not** rely on the Property History page export — re-pull from Carto to confirm completeness.
4. **Pull 311 complaint history.**
   - Source: Carto SQL endpoint against the `public_cases_fc` table.
   - Capture: complaint type, status, date, free-text description.
5. **Cross-check ownership against OPA.**
   - Source: OPA records (linked from `atlas.phila.gov` or `property.phila.gov`).
   - Capture: owner name, mailing address, transfer history.
6. **Document what is missing.**
   - Note that inspector identity is not available in any of the above.
   - Note any closure statuses that are not `COMPLIED` and treat them as unresolved unless an explicit compliance record can be located.
   - Note the date of every pull. Re-pull before publication.

A complete record assembled this way has known boundaries. Treat anything not assembled this way as provisional.

---

## 4. Carto API quick reference

Endpoint:

```
https://phl.carto.com/api/v2/sql?q=<URL-ENCODED-SQL>&format=json
```

Useful tables (verify schema at `https://opendataphilly.org/` before relying on column names — schemas change):

| Table | Contents |
|---|---|
| `violations` | L&I code violations, cases, statuses |
| `permits` | Building, electrical, mechanical, plumbing, zoning permits |
| `business_licenses` | Active and historical business licenses (incl. Rental) |
| `appeals` | L&I appeals filed |
| `public_cases_fc` | 311 service requests |
| `opa_properties_public` | OPA assessment / ownership |
| `unsafe` | Unsafe structure designations |
| `imm_dang` | Imminently dangerous structure designations |

Output formats: `json`, `csv`, `geojson`, `kml`. Add `&format=csv` for direct CSV download. For large queries, paginate with `LIMIT` and `OFFSET`.

---

## 5. What the public data does not tell you

Document these explicitly when reporting; do not let the absence pass without comment.

- **Who inspected.** No inspector identity is published.
- **When the page was last refreshed.** Property History pages do not display a data-as-of timestamp.
- **What was filed but not yet adjudicated.** Cases in pre-violation stages may not appear.
- **Tenant identity.** Complaints are anonymized in the public 311 export; the identity of the complainant is not retrievable without FOIA.
- **Internal L&I correspondence.** Emails, inspection notes, and supervisor sign-offs are FOIA-only.
- **Whether an inspection actually occurred.** A "COMPLIED" status records that the case is closed, not necessarily that an inspector physically returned to the property.

---

## 6. FOIA — when to escalate

If verification requires any of the following, file a Right-to-Know request with the City of Philadelphia (RTKL, 65 P.S. § 67.101 et seq.). The L&I Open Records Officer handles property-record requests.

- Inspector identity for a specific case.
- Inspector handwritten notes or photographs from a site visit.
- Body-worn-camera footage from accompanying police response (filed separately with PPD).
- Internal communications about a property between L&I, PPD, or external counsel.
- Permit-application materials beyond the public abstract.

Standard response window: 5 business days, extendable to 30. Denials are appealable to the Office of Open Records.

---

## 7. Reproducibility note

The four data-integrity failures documented on the parent page (`./` or wherever this guide is linked from) were observed against the property at 315–23 N 12th Street, Philadelphia, PA 19107, OPA 881519440, on or before May 14, 2026. The full record set — the L&I Property History export, the Carto API query result, and the cross-source reconciliation report — is preserved and available on request.

An agent reproducing this verification today should expect:

- The Carto mirror's stated status for Rental License #602204 to continue diverging from Eclipse until the underlying data feed catches up.
- The Property History page to continue presenting records through Sep 2019 unless the city extends the export window.
- Two open 2026 Unfit Structure citations (CF-2026-012614, CF-2026-012633) to appear in Carto but not in the Property History page.

If any of those expectations fail when you reproduce the query, the data systems have changed; update this guide accordingly.

---

*Last reviewed: May 2026. Maintained at `jlegal.pro/agents_philly_opendata_li.md`. Corrections welcome via the issues tab on the underlying repository.*
