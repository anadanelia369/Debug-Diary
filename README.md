# Debug Diary

I don't want to remember only the fixes. I want to remember the reasoning behind them.

Most bugs are solved once. The patterns behind them — the wrong assumption, the missed edge case, the race condition hiding behind a "flaky" test — are reusable for years.

This is where I document the process of moving from assumption to evidence, from symptom to root cause. Every entry follows the same shape: what I observed, what I first assumed, what turned out to actually be true, and what changed because of it.

> Observation is my API. It's the interface through which I investigate systems and learn how they really behave.

---

## Entries

### 001 — The Assertion That Could Never Fail

**Project:** GPI Calendar Automation
**Observation:** A test called `isTodayHighlighted()` was passing consistently. Green, every run.
**Assumption:** The test was correctly verifying that today's date is visually highlighted on the calendar.
**Investigation:** While reviewing the locator logic line by line, I noticed the element was being located *by the same CSS class the test then checked for*. The test was asking "does this element, which I only found because it has this class, have this class?"
**Root cause:** A self-referential assertion — structurally incapable of failing, regardless of what the actual UI did. It wasn't testing behavior; it was testing that `findElement` works.
**Fix:** Rewrote the locator to find the date cell independently (by date value via XPath), then assert on `aria-current="date"` — a real attribute the app sets, not the selector's own class.
**Lesson:** A passing test isn't the same as a meaningful one. Now I ask, for every assertion: *"What would the site have to do wrong for this to go red?"* If the answer is "nothing," the test is decoration, not verification.

---

### 002 — Same Bug, Different Symptom

**Project:** GPI Calendar Automation
**Observation:** One test (`calendarShowsCurrentPeriod`) failed intermittently — passed most runs, failed maybe 1 in 5, with no code changes between runs.
**Assumption:** First instinct: the site itself was unstable, or the wait time just needed to be longer.
**Investigation:** The existing wait covered the calendar *input* becoming visible, but the assertion read text from the calendar *dropdown's contents* — a different element that renders slightly later. The input was ready; its contents weren't always ready yet. A race condition, not a slow page.
**Root cause:** Waiting for the wrong element. The wait and the assertion were checking two different things that happened to usually — but not always — finish rendering close enough together to look fine.
**Fix:** Added an explicit `wait.until(visibilityOf(...))` directly inside the getter methods that read the dropdown's rendered content, not just the input.
**Lesson:** "Flaky" is rarely random. It's almost always a race condition wearing a disguise — and the fix is waiting for the *specific thing you're about to read*, not just "the page" in general.

---

### 003 — Why Not Just `Thread.sleep()`?

**Project:** GPI Calendar Automation
**Observation:** After selecting a date range, the page took a noticeably long pause before the next step became clickable — sometimes fast, sometimes several seconds slower.
**Assumption:** Easiest fix: hardcode a sleep long enough to cover the worst case.
**Investigation:** Testing the same flow logged-in vs. logged-out showed the delay wasn't constant — it depended on session state, not a fixed page behavior. Any hardcoded number would either be wastefully long on fast runs or still too short on slow ones.
**Root cause:** The delay was a *variable* condition, not a fixed duration — the wrong kind of problem for a fixed-time fix.
**Fix:** Explicit Wait tied to the actual condition (element becoming clickable), not a guessed number of seconds.
**Lesson:** `Thread.sleep()` answers "how long might this take?" Explicit Wait answers the actual question, which is "is it ready yet?" Those are different questions, and only one of them is reliably true.

---

*More entries added as I find them — this is a living log, not a finished document.*

---

### 004 — The Locator That Broke Overnight

**Project:** GPI Calendar Automation
**Observation:** Revisiting the project for post-grade polish, tests that passed on submission day started failing — four of them at once, all sharing one thing in common.
**Assumption:** A regression in my own code from recent edits.
**Investigation:** Checked the live site directly instead of the code first. The date buttons' `aria-label` format had changed from `M/d/yyyy` to `dd/MM/yyyy` sometime after I originally wrote the locators — every locator built on that exact string format broke the moment the format did.
**Root cause:** A third-party format change outside my code, on locators built directly from a formatted value that was never guaranteed to stay the same.
**Fix:** Rewrote the date locator as XPath and changed the "today is highlighted" check to look for `aria-current="date"` — a semantic attribute the app sets on purpose, not a string built from a date format someone else controls.
**Lesson:** A locator built on formatted data is a locator built on someone else's decision. Where possible, anchor to attributes with a fixed meaning, not ones assembled from values that can silently change.

---

### 005 — The Assertion That Compared Itself

**Project:** GPI Calendar Automation
**Observation:** During a full line-by-line review of the finished project, one line looked suspicious: `Assert.assertTrue(startDate.isBefore(endDate))`.
**Assumption:** This confirms the application put the selected dates in the correct order.
**Investigation:** Traced where `startDate` and `endDate` actually came from — they were my own local test variables, built from a hardcoded day offset inside the test itself, never read back from the application at all.
**Root cause:** The assertion compared my own test data against itself. It could not fail regardless of what the app actually did — a different mechanism than Entry 001, but the same underlying shape of problem.
**Fix:** Removed it. Real coverage was already provided by a separate assertion checking that the date-range text the app displays matches what was selected.
**Lesson:** False-positive assertions don't all look alike in code. Entry 001's bug was a self-referential locator; this one was self-referential data. The question that catches both is the same: *does this assertion depend on anything the app actually did?*

---

### 006 — The Wait That "Worked" by Luck

**Project:** GPI Calendar Automation
**Observation:** `clickContinue()` passed reliably in TC-02, but the same method occasionally failed to find the button when reused in TC-03 (empty-period validation, called immediately with no prior date selection).
**Assumption:** The button itself was inconsistent, or TestNG's execution order was somehow affecting it.
**Investigation:** In TC-02, several prior actions (selecting two dates) ran before `clickContinue()` was called — by the time it ran, enough time had already passed for the button to render. TC-03 called it immediately, with no such buffer.
**Root cause:** `clickContinue()` never had its own wait — it was "passing" in TC-02 only because other code accidentally gave it enough time.
**Fix:** Added `wait.until(visibilityOf(continueButton))` directly inside `clickContinue()`, so it no longer depends on what happened to run before it.
**Lesson:** A method with no wait of its own isn't reliable just because it passed once — it may be borrowing time from whatever called it first.

---

### 007 — The Config File Whose Name Almost Matched

**Project:** GPI Calendar Automation
**Observation:** During a full project review, `pom.xml`'s surefire config referenced `testng.xml` (lowercase), while the actual file on disk was `testNG.xml` (capital NG).
**Assumption:** Since tests ran fine locally on Windows, this wasn't an issue.
**Investigation:** Windows filesystems are case-insensitive, so the mismatch went unnoticed — but Mac/Linux filesystems are case-sensitive, meaning this would silently fail to find the suite file on another machine, or in CI.
**Root cause:** A filename reference that only "worked" because of the specific OS being used to run it.
**Fix:** Corrected `pom.xml` to reference `testNG.xml` exactly.
**Lesson:** "Works on my machine" can hide filesystem-specific assumptions — case sensitivity is invisible until you run somewhere else.

---

### 008 — The Test That Ran in the IDE but Not in the Terminal

**Project:** GPI Calendar Automation
**Observation:** `ApiTest` passed when run via IntelliJ's Run button, but `mvn clean test` from the terminal only reported 6 tests, not 7.
**Assumption:** The API test was being skipped due to a RestAssured/Maven configuration issue.
**Investigation:** Checked `testNG.xml`'s `<classes>` list directly — `ApiTest` was simply never added to it. IntelliJ's Run button executes the class directly, bypassing the suite file entirely, which had been masking the gap.
**Root cause:** Two different ways of running tests (IDE vs. Maven/suite file) were silently testing different things.
**Fix:** Added `ApiTest` to `testNG.xml`'s `<classes>` list; confirmed 7/7 via `mvn clean test`.
**Lesson:** Always validate the way the grader/CI will actually run the suite, not just the IDE's convenience button — they aren't guaranteed to cover the same tests.

---

### 009 — The Locator Class That Renamed Itself Overnight

**Project:** GPI Calendar Automation
**Observation:** Revisiting the project post-grade, a compile error appeared: `cannot find symbol — static driver, location: class calendarautomation.utils.DriverManager`.
**Assumption:** `DriverManager` itself was broken or missing the field.
**Investigation:** `DriverManager.driver` was `private` and always had been — the real issue was a leftover `import static calendarautomation.utils.DriverManager.driver;` in `CalendarPage`, dating from before `CalendarPage` inherited `driver` from `BasePage`. Once that inheritance existed, the static import became not just unnecessary but illegal.
**Root cause:** A stale import from an earlier architecture that no longer matched how the class actually got its `driver` reference.
**Fix:** Removed the unused static import.
**Lesson:** Compile errors after a refactor aren't always about the change you just made — check for leftover references to how the code used to be wired.

---

### 010 — The Class TestNG Couldn't Find

**Project:** GPI Calendar Automation
**Observation:** After renaming the package from `org.example` to `calendarautomation`, `mvn clean test` compiled but then failed with `Cannot find class in classpath: Tests.calendarautomation.CalendarTest` — 0 tests run.
**Assumption:** The rename itself had broken something in the test class.
**Investigation:** The actual file path was `.../java/calendarautomation/Tests/CalendarTest.java`, so the correct fully-qualified name was `calendarautomation.Tests.CalendarTest` — but `testNG.xml` still listed it reversed, left over from a manual edit during the rename.
**Root cause:** A manual find-and-replace swapped the segments in the wrong order in one config file, while the actual `.java` file path was correct.
**Fix:** Corrected the class name order in `testNG.xml`.
**Lesson:** A package rename touches config files as well as source files — and config files don't get the IDE's refactor-tooling safety net the way `.java` files do.

---

### 011 — The Expected Value That Was Never the Right Shape

**Project:** GPI Calendar Automation
**Observation:** Post-grade, `calendarShowsCurrentPeriod` failed with `expected [9/3/2026] but found [SEP 2026]` — even though this exact test had passed on submission day.
**Assumption:** The site's calendar-header format had changed sometime after submission.
**Investigation:** Compared the assertion's expected-value construction (`LocalDate.now().format(DateTimeFormatter.ofPattern("M/d/yyyy"))`) against what the header element actually is — a month+year label, not a full date. The two could never have matched, regardless of any site change.
**Root cause:** The expected value was built in the wrong format for what was actually being compared.
**Fix:** Rebuilt the expected string with `DateTimeFormatter.ofPattern("MMM yyyy", Locale.ENGLISH).toUpperCase()` to match the header's actual shape.
**Lesson:** A test that passed once isn't proof the comparison was ever logically sound — it's worth periodically asking whether both sides of an assertion are even the same kind of value.