# ClickUp Pipeline Audit & Standardization Guide

## Why This Matters

You mentioned: *"It's fairly structured. It needs to be a little more structured."*

**The app can only be as good as your ClickUp data.** If loan data is inconsistent, voice queries will fail or give wrong answers. This audit ensures we can reliably pull the right information.

---

## Pre-Build Audit Checklist

### 1. ClickUp Organization Review

**Questions to Answer:**
- [ ] What Workspace is the pipeline in?
- [ ] What Space contains loan tasks?
- [ ] What Folder(s) organize the loans?
- [ ] What List(s) contain active loans?
- [ ] Are there separate lists for different loan types? (Purchase vs Refi)

**Why:** We need to know which List ID to query for all loans.

**Action:** Open ClickUp → Navigate to your pipeline → Copy the List ID from the URL
```
https://app.clickup.com/[workspace]/v/li/[THIS_IS_THE_LIST_ID]
```

---

### 2. Task Naming Convention Audit

**Current Pattern Check:**
Look at 10 random loan tasks and document how they're named:

**Examples to review:**
```
Task 1: "Johnson, Mike & Sarah - 123 Main St"
Task 2: "Smith Loan - 456 Oak Ave"
Task 3: "Garcia Purchase"
Task 4: "Brown, Jennifer"
Task 5: "Martinez - Refinance - 789 Elm"
```

**Questions:**
- [ ] Do all tasks include borrower last name?
- [ ] Is the format consistent? (Last, First vs First Last)
- [ ] Is property address always included?
- [ ] Are loan types indicated? (Purchase, Refi, etc.)
- [ ] Any special characters that could break voice search? (é, ñ, etc.)

**Standardization Recommendation:**
```
Format: [Last Name], [First Name(s)] - [Property Address] - [Loan Type]
Example: "Johnson, Mike & Sarah - 123 Main St - Purchase"

✅ Good: Consistent, includes all key info
❌ Bad: "Mike Johnson loan" (no address, harder to distinguish from other Johnsons)
```

**Why:** Voice queries like "Status of Johnson loan" need to find the right task. If you have 3 Johnson loans, the app needs property address or loan type to disambiguate.

---

### 3. Custom Fields Audit

**Step 1: Document ALL Custom Fields**

Go to ClickUp → Your List → Click any task → See all custom fields

**Make a table:**
| Custom Field Name | Type | Always Filled? | Sample Value | Purpose |
|-------------------|------|----------------|--------------|---------|
| Loan Officer | User | Yes | @Mike Jones | Who owns the loan |
| Processor | User | Yes | @Sarah Lee | Who processes it |
| Closing Date | Date | Yes | 11/15/2024 | Target close |
| Property Address | Text | Yes | 123 Main St | Property location |
| Loan Amount | Number | Sometimes | 450000 | Loan size |
| Loan Type | Dropdown | Yes | Purchase / Refi | Transaction type |
| Client Phone | Phone | Sometimes | (555) 123-4567 | Borrower contact |
| Client Email | Email | Sometimes | mike@example.com | Borrower email |
| Appraisal Date | Date | Sometimes | 10/25/2024 | Key milestone |
| Conditions Count | Number | No | 3 | Outstanding conditions |

**Step 2: Identify Critical Fields**

**Must-Have (app won't work without):**
- [ ] Loan Officer (to filter "my loans")
- [ ] Closing Date (for "when does X close?")
- [ ] Borrower name (in task name or custom field)

**Should-Have (important for UX):**
- [ ] Property Address (to distinguish loans)
- [ ] Loan Type (Purchase, Refi, etc.)
- [ ] Processor (who to contact)

**Nice-to-Have:**
- [ ] Client contact info
- [ ] Appraisal/inspection dates
- [ ] Loan amount

**Step 3: Check Data Quality**

For each critical field, spot-check 20 tasks:
- [ ] How many have this field filled in? (Goal: >95%)
- [ ] Is formatting consistent?
- [ ] Any obviously wrong data? (closing date in past, etc.)

**Red Flags:**
- ❌ Critical field missing in >5% of tasks
- ❌ Inconsistent formats (some dates as "11/15", others as "November 15")
- ❌ Outdated data (loans closed 6 months ago still marked "in process")

---

### 4. Status/Stage Audit

**Step 1: List All Statuses**

ClickUp → List Settings → Statuses

**Document:**
| Status Name | What It Means | Typical Duration | Next Status |
|-------------|---------------|------------------|-------------|
| Application | Loan app submitted | 1-3 days | Processing |
| Processing | Gathering docs | 5-10 days | Underwriting |
| Underwriting | UW review | 5-7 days | CTC |
| Clear to Close | All conditions met | 2-5 days | Docs Out |
| Docs Out | Awaiting signatures | 1-3 days | Funded |
| Funded | Loan complete | - | Archived |

**Questions:**
- [ ] Are statuses in logical order?
- [ ] Do all active loans have correct status?
- [ ] Any "stuck" loans in wrong status?
- [ ] Do you use all these statuses?

**Standardization Check:**
- ✅ Good: "Processing" → "Underwriting" → "Clear to Close"
- ❌ Bad: "In Process" vs "Processing" vs "Being Processed" (inconsistent names for same stage)

**Why:** Voice queries depend on stages. "How many loans in underwriting?" only works if all UW loans are in "Underwriting" status.

---

### 5. Tags Audit

**Step 1: List All Tags Used**

Look through tasks and document every tag:

**Examples:**
- Urgent
- FHA
- VA
- Conventional
- First-Time Buyer
- Investor
- Construction
- Jumbo

**Questions:**
- [ ] Are tags used consistently?
- [ ] Do tags add info not in custom fields?
- [ ] Are tags current? (remove "Urgent" after urgency passes?)

**Standardization Recommendation:**
Use tags for:
- ✅ Loan programs (FHA, VA, Conventional)
- ✅ Special circumstances (First-time buyer, Self-employed)
- ✅ Temporary flags (Urgent, On Hold)

Don't use tags for:
- ❌ Info that belongs in custom fields (Loan Officer name)
- ❌ Permanent attributes (Property state - use custom field instead)

---

### 6. Conditions Tracking Audit

**How are conditions currently tracked?**

**Option A: Checklist inside task**
```
☐ VOE (Verification of Employment)
☑ VOD (Verification of Deposit)
☐ Appraisal
☐ Home Insurance
```

**Option B: Subtasks**
```
Main Task: Johnson Loan
  ├─ Subtask: Get VOE
  ├─ Subtask: Get VOD (completed)
  ├─ Subtask: Order Appraisal
```

**Option C: Comments/Description**
```
Task description or comments:
"Needs VOE, VOD, appraisal"
```

**Option D: Separate tasks**
```
Each condition is its own task in a "Conditions" list
```

**Questions:**
- [ ] Which method do you use?
- [ ] Are condition names consistent? ("VOE" vs "Verification of Employment")
- [ ] Can you easily count outstanding conditions per loan?
- [ ] When a condition clears, is it marked consistently?

**App Integration Impact:**
- **Checklist (A):** ✅ Easy to integrate, ❌ No due dates per condition
- **Subtasks (B):** ✅ Easy to integrate, ✅ Supports due dates, ✅ Best option
- **Comments (C):** ❌ Hard to parse, inconsistent
- **Separate tasks (D):** ⚠️ Possible but requires linking logic

**Recommendation:** If not using subtasks (Option B), migrate to it before building app.

---

### 7. Communication Audit

**Where do LOs/processors communicate about loans?**

**Current channels to document:**
- [ ] ClickUp task comments
- [ ] ClickUp task descriptions
- [ ] Email (outside ClickUp)
- [ ] Phone calls (not tracked)
- [ ] Text messages (not tracked)
- [ ] Microsoft Teams/Slack
- [ ] Other: _______

**Questions:**
- [ ] What % of communication is in ClickUp? (Goal: >80%)
- [ ] Can you reconstruct a loan's history from ClickUp alone?
- [ ] Are important decisions documented?

**Recommendation for App Success:**
Move ALL loan-related communication to ClickUp task comments. Why?
1. App can display recent updates in voice readout
2. Creates audit trail (important for compliance)
3. Eliminates "he said/she said" confusion
4. Future: App can let users ADD comments via voice

**Standardization:**
```
✅ Good comment: "@Sarah - VOE received from employer. Moving to UW tomorrow."
❌ Bad comment: "got it" (what was received?)
```

---

### 8. Data Hygiene Audit

**Check for data quality issues:**

**Spot-check 30 random tasks:**
- [ ] How many have typos in borrower names?
- [ ] How many have missing critical fields?
- [ ] How many have outdated statuses? (marked "Processing" but closed 2 months ago)
- [ ] How many are duplicates?
- [ ] How many have wrong loan officer assigned?

**Goal:** <3% error rate

**Common Issues to Fix:**
1. **Duplicate tasks:** Merge or archive
2. **Closed loans in active list:** Move to archive
3. **Missing data:** Fill in or establish "required fields" policy
4. **Inconsistent capitalization:** "john smith" → "John Smith"

---

### 9. Access & Permissions Audit

**Questions:**
- [ ] Do all loan officers have ClickUp accounts?
- [ ] Do they have correct permissions? (Can they view all loans or just theirs?)
- [ ] Can processors update all fields?
- [ ] Are there any restricted fields?

**Why:** App will inherit ClickUp permissions. If LO can't see processor notes in ClickUp, they won't see them in the app either.

**Recommendation:** Review ClickUp permissions before app launch to ensure everyone has appropriate access.

---

## Standardization Action Plan

### Phase 1: Immediate Fixes (This Week)

**Priority fixes before app development starts:**

1. **Standardize task naming:**
   - [ ] Establish format: "Last, First - Address - Type"
   - [ ] Rename inconsistent tasks
   - [ ] Document standard in team guide

2. **Fill missing critical fields:**
   - [ ] Closing Date (required for every loan)
   - [ ] Loan Officer (required)
   - [ ] Property Address (if not in task name)

3. **Audit loan statuses:**
   - [ ] Move closed loans to "Funded" or archive
   - [ ] Fix any loans in wrong stage
   - [ ] Remove unused statuses

4. **Clean up duplicates:**
   - [ ] Search for duplicate borrower names
   - [ ] Merge or archive duplicates
   - [ ] Add note in merged task

### Phase 2: Process Improvements (Next 2 Weeks)

**Build habits that keep data clean:**

5. **Create ClickUp task template:**
   ```
   Template: "New Loan"
   Required fields:
   - Borrower Name (in task title)
   - Property Address
   - Loan Officer (auto-assign creator)
   - Processor (assign manually)
   - Closing Date
   - Loan Type
   Status: "Application"
   Checklist: Standard conditions list
   ```

6. **Establish update cadence:**
   - [ ] LOs update statuses daily (or when changes occur)
   - [ ] Processors add comments on every action taken
   - [ ] Close out tasks within 24 hours of funding

7. **Centralize communication:**
   - [ ] New policy: All loan updates go in ClickUp comments
   - [ ] No more text/email status checks
   - [ ] Use @mentions to notify team members

### Phase 3: Maintenance (Ongoing)

8. **Weekly audit:**
   - [ ] Review loans stuck in one status >10 days
   - [ ] Check for missing data in new loans
   - [ ] Archive closed loans

9. **Monthly cleanup:**
   - [ ] Remove unused tags
   - [ ] Archive loans >60 days old
   - [ ] Review and update custom fields

---

## Data Quality Metrics

**Before App Launch, Achieve:**
- ✅ 100% of active loans have Closing Date
- ✅ 100% of active loans have Loan Officer assigned
- ✅ 95%+ of loans have correct status
- ✅ 90%+ of important updates in ClickUp comments
- ✅ <5% of loans have missing critical fields
- ✅ Task naming follows standard format
- ✅ Zero closed loans in "active" statuses

**Track Monthly:**
- Data completeness score
- Days loans spend in each status (identify bottlenecks)
- Number of overdue closing dates
- Communication in ClickUp vs outside ClickUp

---

## Integration Mapping Document

**After audit, create this reference doc:**

```yaml
# ClickUp to App Mapping

Workspace: True North Mortgage
Space: Loan Pipeline
Folder: Active Loans
List: 2024 Pipeline
List ID: 123456789

Task Naming Format: "Last, First - Address - Type"
Example: "Johnson, Mike & Sarah - 123 Main St - Purchase"

Custom Fields:
  - loan_officer:
      clickup_name: "Loan Officer"
      type: user
      app_field: loan_officer_id
  - processor:
      clickup_name: "Processor"
      type: user
      app_field: processor_id
  - closing_date:
      clickup_name: "Closing Date"
      type: date
      app_field: closing_date
  - property_address:
      clickup_name: "Property Address"
      type: text
      app_field: property_address
  - loan_type:
      clickup_name: "Loan Type"
      type: dropdown
      app_field: loan_type
      options: [Purchase, Refinance, HELOC, Construction]

Statuses (in order):
  - Application
  - Processing
  - Underwriting
  - Clear to Close
  - Docs Out
  - Funded

Conditions Tracking: Subtasks (preferred) or Checklist

Tags Used:
  - FHA
  - VA
  - Conventional
  - Urgent
  - First-Time Buyer
```

---

## Red Flags That Will Break the App

**Critical issues that MUST be fixed before launch:**

1. ❌ **No consistent borrower identification**
   - Problem: Can't find "Johnson loan" if named "J. Smith property"
   - Fix: Standardize task naming

2. ❌ **Closing dates missing or wrong**
   - Problem: "When does X close?" query fails
   - Fix: Make Closing Date a required field

3. ❌ **Multiple List IDs for active loans**
   - Problem: App queries one list, misses others
   - Fix: Consolidate to single List or query multiple

4. ❌ **Conditions tracking inconsistent**
   - Problem: "What conditions does X need?" gives wrong answer
   - Fix: Migrate to subtasks or checklist (pick one method)

5. ❌ **Loan Officer field not filled**
   - Problem: LO asks "What's my pipeline?" and gets empty result
   - Fix: Assign LO to all tasks retroactively

---

## Testing Your ClickUp Structure

**Before app development, manually test these queries:**

```
Test Query 1: "Show me all loans closing this month"
→ Can you filter ClickUp by closing_date easily?

Test Query 2: "What conditions does the Johnson loan need?"
→ Can you find Johnson loan quickly? Are conditions clear?

Test Query 3: "How many loans are in underwriting?"
→ Can you count them accurately from status filter?

Test Query 4: "What's the status of the Garcia loan?"
→ Can you find it by name? Is status up-to-date?

Test Query 5: "Any updates today on my loans?"
→ Can you see recent comments on all your loans?
```

**If any query takes >30 seconds or gives unclear answers, you've found a data structure problem to fix.**

---

## Summary: Pre-Build Action Items

**Complete before starting app development:**

- [ ] Document ClickUp List ID(s) for loans
- [ ] Audit task naming consistency (>95% follow standard)
- [ ] Audit custom fields (100% of critical fields filled)
- [ ] Audit statuses (all loans in correct stage)
- [ ] Choose conditions tracking method (subtasks recommended)
- [ ] Assign Loan Officer to all existing loans
- [ ] Add Closing Date to all loans missing it
- [ ] Archive/close loans that are completed
- [ ] Create task template for new loans
- [ ] Document field mapping (ClickUp → App)
- [ ] Test 5 voice queries manually in ClickUp
- [ ] Train team on new standards

**Estimated time:** 10-15 hours (spread over 1-2 weeks)

**This upfront work will save 30+ hours of app debugging later.**

---

**Ready to start the audit? I can help you:**
1. Build a ClickUp audit checklist (exportable)
2. Create a data cleanup script
3. Design a task template for new loans
4. Write team training guide for standards

Which would be most helpful?
