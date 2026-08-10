# Crown Superior - Policy List PDF: what was changed and where

Everything below applies ONLY to the document downloaded from
`https://crownsuperior.com/policies/policy-list` (Joomla menu item id 350).
No Joomla core files, no RSForm! Pro files and no plugin files were modified.

---

## 1. Where the document actually lives

The PDF is NOT produced by the "System - RSForm! Pro PDF" plugin.
It is produced by the RSForm! Pro **Submissions - View** menu item, and the whole
document body is stored in that menu item's parameters:

    Menus -> Policies -> Policy List  (id = 350)
    Tab: "Submission Details" -> field: Details Layout
    URL: /administrator/index.php?option=com_menus&view=item&client_id=0&layout=edit&id=350

That is the same screen you already use to edit this document, and it is still the
only place you need to touch. Nothing moved.

Rendering engine: DOMPDF 2.0.8 (via RSFormPDF).

---

## 2. Fix 1 - signature block repeating on every page

### Cause
Both signature blocks were styled with:

    class="pdf-fixed-footer"
    style="position: fixed; bottom: 0; left: 0; right: 0; ..."

`position: fixed` in DOMPDF means "paint this on EVERY page" - that is by design in
DOMPDF, not a bug in the template. So the block was stamped onto all 12 pages.

### Change
Both blocks were switched to normal in-flow content, and told never to split:

    style="position: static; width:100%; box-sizing:border-box; margin-top:10px;
           text-align:center; border-top:1px solid black; padding:8px 0 0 0;
           font-size:12px; page-break-inside:avoid;"

* `position: static` - block flows with the content, so it prints once only.
* `page-break-inside: avoid` - forces the whole block (line + paragraph +
  signature/date rows) to stay together, so it can never spill onto a new page.

### Change
The spacer that used to reserve room for the fixed footer:

    <div style="padding-bottom:105px;">   ->   <div style="padding-bottom:0px;">

With the footer no longer fixed, that 105px reserve was pushing the signature block
onto page 12. Setting it to 0 keeps the signature block at the bottom of the
WYOMING cancellation page.

Result: signature block appears on the WYOMING page only, at the bottom, and the
document dropped from 12 pages to 11.

---

## 3. Fix 2 - page break before the UNDISCLOSED DRIVER notice

Inserted immediately before the `<table>` that opens the UNDISCLOSED DRIVER block
(the red line you drew):

    <div style="page-break-before: always;"></div>

---

## 4. Fix 3 - removed items

* `TIC_GADREXCL (09/22)` removed from the Named Driver Exclusion page.
* `Name: ____` field and its line removed from the first appended page.
* `ACCOUNT OR POLICY NUMBER(S): ____` field and its line removed.
* The only name field left anywhere is `NAME OF EXCLUDED DRIVER`.

---

## 5. The 9 additional disclosure pages

### Why they could not simply be pasted into the menu item
Joomla stores menu item parameters in `#__menu.params`, a MySQL `TEXT` column,
hard-capped at **65,535 characters**. This menu item's params are already ~61,500
characters, leaving roughly 4,000 characters of headroom. The new pages are ~22,000
characters. Joomla accepts an oversized save and reports "Menu item saved", but the
database silently truncates/rejects it - which is exactly why the first attempt
looked saved but changed nothing.

### How it was done instead
The same mechanism your site already uses for the Named Driver Exclusion document
(`nrde_doc`):

1. A **Free Text** component named `extra_pages` was added to form 11
   (Components -> RSForm! Pro -> Manage Forms -> Receipt).
   Component id: **3383**. It holds the 9 pages of disclosure HTML (~22,000 chars),
   stored in RSForm's own table, so the Joomla menu limit does not apply.

2. One line was added to the Details Layout in menu item 350, at the very end,
   just after the closing `{/if}`:

        <!-- CS-EXTRA-PAGES: additional disclosure pages (RSForm free-text
             component "extra_pages" on form 11) -->
        {extra_pages:value}

   That is a net change of +19 characters to the menu item, so it saves fine.

### Why this does not affect your CSV exports
RSForm! Pro excludes Free Text components from submissions and from every export
format. Verified on the live site: the export column list contains **174** columns
both before and after the change, and neither `extra_pages` nor the existing
`nrde_doc` appears in it, in any position. Your CSV files are byte-for-byte
unaffected.

### Why it does not show on the public form
A conditional field was added so the block is never displayed on
`/policies/create-a-policy`:

    Manage Forms -> Receipt -> Conditional Fields  (condition id 754)
    Show  block  extra_pages  if ALL of the following match:
        Add_Nrde  is      Yes
        Add_Nrde  is not  Yes

Those two rules can never both be true, so the block renders with
`display: none` and stays hidden - the same way `nrde_doc` and `cancel-notice`
already behave. Verified in a real browser on the live form.

(Conditions only control on-screen display. Free Text values are static, so the
condition has no effect at all on what the PDF prints.)

---

## 6. Verified on the real downloaded file

Downloaded from `https://crownsuperior.com/policies/policy-list` (not a local
mock-up), submissions 11932 and 11923:

* 20 pages (11 original + 9 appended).
* Signature block: exactly 1 occurrence, bottom of the WYOMING page (page 11).
* UNDISCLOSED DRIVER starts its own page.
* Pages 1-11 text is byte-for-byte identical to before the append.
* 0 occurrences of `TIC_GADREXCL`, `ACCOUNT OR POLICY`, `Name:`.
* 1 occurrence of `NAME OF EXCLUDED DRIVER`.
* 0 leftover template code (`{if`, `:value}`, `="Yes"}`) anywhere in the output.

---

## 7. Files in this folder

| File | What it is |
| --- | --- |
| `POLICYLIST_menu350_ORIGINAL.html` | Pristine backup of the Details Layout before any change. Paste this back to roll everything up to section 4 back. |
| `POLICYLIST_menu350_FINAL.html` | The Details Layout as it is live now. |
| `appended_pages_inline.html` | Exact content stored in the `extra_pages` Free Text component. All CSS is inline on purpose - see below. |
| `POLICY_LIST_FINAL_20pages.pdf` | The actual 20-page file downloaded from the live site. |
| `APPENDED_PAGES_preview.pdf` | The 9 appended pages on their own. |

---

## 8. Two gotchas worth knowing for future edits

**1. Never use a `<style>` block in these fields.**
Joomla's text filter rewrites `<style>` to `<s-tyle>` on save. It does this
silently - the save succeeds and the page looks fine in the editor, but every rule
inside is dead, so page breaks and fonts quietly stop working. That is why all
styling in the appended pages is written as inline `style="..."` attributes.

**2. The edit screen lies to you after saving.**
Re-opening the menu item after a save shows you what you submitted (from the
session), not what is in the database. If a save is rejected for size, it still
looks correct on screen. The only reliable check is to download the actual PDF
from `/policies/policy-list` and look at it.

---

## 9. To replicate this on staging or another form

1. Copy `POLICYLIST_menu350_FINAL.html` into the Details Layout of the equivalent
   Submissions - View menu item.
2. On the target form, add a Free Text component named `extra_pages` and paste in
   `appended_pages_inline.html`.
3. Add the conditional field described in section 5 so it stays hidden on the form.
4. Download the PDF from the front end and confirm the page count.

---

# ROUND 2 - conditional exclusion pages (10 Aug 2026)

## 10. What changed

Two of the nine appended pages are no longer printed on every policy.

### PUNITIVE AND EXEMPLARY DAMAGE EXCLUSION
Prints only when the form field `punitive_damages` ("Exclude punitive damages
coverage") is set to `Yes`. Otherwise the page is not in the document at all.

### NAMED DRIVER EXCLUSION ENDORSEMENT
Prints only for drivers actually marked as excluded, and fills itself in:

    NAME OF EXCLUDED DRIVER   first + last name of that driver
    DATE OF BIRTH             that driver's date of birth field
    RELATIONSHIP TO APPLICANT that driver's relation field

Each excluded driver gets their own endorsement page with its own signature and
date line. Two excluded drivers = two endorsement pages. None excluded = no page.

## 11. IMPORTANT gotcha - the "excluded?" dropdowns default to Y

`Is_Second_driver_excluded`, `Is_third_driver_excluded` and
`Is_fourth_driver_excluded` are dropdowns whose options are `Y` then `N`. RSForm
stores the FIRST option when the question is never answered, so on a policy with
only one driver all three of these read `Y`.

Testing this on live policies confirmed it - 11932, 11923 and 11900 all store
`X2=Y; X3=Y; X4=Y` while only having one extra driver.

So the exclusion test alone is not safe. Every block is guarded by two nested
tests, driver-exists FIRST:

    {if {Is_there_a_second_driver_:value}="Y"}{if {Is_Second_driver_excluded:value}="Y"} ... {/if}{/if}

If the existence test is ever removed, every policy will print three exclusion
pages. That is the single most important line to preserve.

## 12. Why the logic could not go in the stored text block

RSForm replaces placeholders in ONE pass. A `{field:value}` or `{if}` written
INSIDE a free-text component is not processed - it prints literally. Verified on
the live site with an invisible test marker: the PDF came back showing the raw
text `{First Name:value}` and `{if {punitive_damages:value}="Yes"}`.

So all conditions and all dynamic values must live in the menu 350 layout. The
free-text components hold only fixed wording.

The Directory "script called on view" PHP hook was considered and rejected:
placeholders are substituted into the PHP source as literal text, so a driver
named O'Brien would break the script. Keeping the values in HTML avoids that
class of failure entirely.

## 13. New free-text components on form 11

| Component | id | Holds |
| --- | --- | --- |
| `extra_pages` (existing, rewritten) | 3383 | first 3 disclosure pages |
| `punitive_page` | 3384 | the punitive exclusion page |
| `nde_page` | 3385 | exclusion endorsement heading + wording |
| `nde_sign` | 3386 | the signature/date line for that page |
| `extra_pages_2` | 3387 | remaining 4 disclosure pages |

The document order is unchanged from the version already approved - the two
conditional pages sit exactly where they were, between `extra_pages` and
`extra_pages_2`.

`nde_page` ends with an unclosed `<div>` which `nde_sign` closes. That is
deliberate: the driver's name/date-of-birth/relationship lines are written
between them by the menu layout, and the wrapper keeps the whole block from
splitting across two pages.

All four new components have a hiding condition (same contradictory-rule trick
as `extra_pages`) so they never appear on the public form. Confirmed in a real
browser: all render `display: none`, and none appear in the submissions list or
any export, so CSV exports are unaffected.

## 14. Verified on real downloads

| Policy | Stored values | Result |
| --- | --- | --- |
| 11932 | E2=Y X2=Y, E3=N E4=N, punitive empty | 19 pages, ONE endorsement page filled in "ELAINE STEVENS / 22 / 4 / 1982 / Spouse", no punitive page |
| 11700 | E2=N, no reimbursement section, punitive empty | 13 pages, no endorsement page, no punitive page |
| 11932 (punitive test) | condition temporarily matched | 20 pages, punitive page renders correctly in its original position (page 15) |

Zero leftover template code (`{if`, `:value}`, `{/if}`) in any output.

## 15. Open point - date format

Dates of birth print as stored by the form, which is set to day/month/year with
a " / " separator, e.g. `12 / 3 / 1979` meaning 12 March 1979. On a US document
that reads as 3 December to most people. Changing the `Dob_driver_2/3/4` fields
to month/day/year in RSForm would fix it everywhere - say the word and it is a
two-minute change.
