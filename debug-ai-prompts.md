# Prompts for week 8

## step 2

Read LearningAPI/views/student_view.py and focus on the `update` method.

**Problem:** A user reports that updating their Slack handle appears to succeed (the API returns a success response and the UI shows no error), but after a page refresh the old value is back.

**Task 1: Walk through `update` step by step.**
Trace the method from the incoming request to the response. For each step, note:
- Where the request data is read (`request.data`, serializer input, etc.)
- How the target student/user object is looked up
- Which fields are modified, and whether `slack_handle` is among them
- Where (or whether) `.save()`, `serializer.save()`, or a queryset `.update()` is called
- What gets returned, and whether it reflects the saved state or just the incoming data

**Task 2: Identify why `slack_handle` might not persist.**
Check specifically for:
1. `slack_handle` missing from the serializer's `fields`, or marked `read_only`
2. `slack_handle` living on a different model than the one being saved (e.g., Student vs. User vs. a related profile)
3. A missing or misplaced `.save()` call, or a save on the wrong object
4. Field name mismatches between the frontend payload and the backend (`slack_handle` vs. `slackHandle`)
5. Exceptions caught and swallowed, so a failure still returns a success response
6. A response built from the request data rather than the refreshed database record
7. `partial=True` handling, validation silently dropping the field, or transaction rollbacks

**Output format:**
1. Step-by-step walkthrough, with line numbers or code snippets
2. The most likely root cause, with the exact line(s) responsible
3. A corrected version of the relevant code, ready to paste in
4. A brief explanation of why the fix works, and any tradeoffs between fix options if more than one exists

If you can't determine the cause from this file alone (for example, the serializer or model lives elsewhere), tell me which file you need to see rather than guessing.

Which specific line is missing, and what Django concept explains why the change does not reach the database without it?

## step 4
Read LearningAPI/tests/test_student_update.py in full before writing anything. Then implement only the body of `test_patch_slack_handle_persists_to_database`, replacing its `fail("Not Implemented yet")` call. Do not modify setUp, other tests, imports that are already used, or any other file.

**Context:** The test class already creates a user, a token, and an NssUser with slack_handle='@old_handle'. Reuse those exact attribute names from setUp. Do not create new fixtures.

**The test must:**
1. Authenticate the request the same way the other tests in this file do (for example, `self.client.credentials(HTTP_AUTHORIZATION=f"Token {self.token.key}")`). Copy the file's existing convention if it differs.
2. Send a PUT to `/students/{pk}/` with a new slack_handle value, using `format="json"`. Use the pk the view actually looks up (check whether it is the NssUser pk or the User pk) and include any other fields the view requires so validation doesn't fail for an unrelated reason.
3. Assert the response status is 204.
4. Re-fetch the NssUser from the database with a fresh query (`NssUser.objects.get(pk=...)`), not by reusing the in-memory object from setUp.
5. Assert the re-fetched slack_handle equals the value sent.

**Constraints:**
- Use a new value that differs from '@old_handle' (for example '@new_handle'), so the test can't pass vacuously.
- Match the file's existing style: `self.assertEqual`, naming, and spacing.
- Replace `fail(...)` entirely. Do not leave it in.

**Before writing, check these and tell me instead of guessing:**
- The method name says "patch" but the request is PUT. If the view only supports one of them, tell me which and why.
- If the view's lookup key, required payload fields, or auth scheme can't be determined from the test file and views/student_view.py, name the file you need.

**Output:**
1. The complete method, ready to paste in.
2. A one-line note on each assumption you made.
3. Run the test and report the result. If it fails, say whether the failure is in the test or in the view.

## step 6
What is the single line that fixes the bug in the `update` method, and where exactly in the method does it go?