# 🪲 Debug Diary

*I don't want to remember only the fixes. I want to remember the reasoning behind them.*

> Observation is my API.

---

## 📋 Pre-Submission (before the Aug 10 deadline)

### 🔁 002 — Same Bug, Different Symptom

**What happened:** `calendarShowsCurrentPeriod` failed about 1 in 5 runs, with no code changes between runs.

**What I found:** The wait covered the input becoming visible, not the dropdown's contents rendering — a race condition, not a slow page. 🐌

**What I changed:** Added an explicit wait inside the getters that read the dropdown's actual content.

---

### ⏳ 003 — Why Not Just Thread.sleep()?

**What happened:** After selecting dates, the next step's load time varied — sometimes fast, sometimes slow.

**What I found:** The delay depended on login state, not fixed page behavior — testing logged-in vs. logged-out confirmed it.

**What I changed:** Used Explicit Wait tied to the real condition instead of guessing a fixed number of seconds.

---

### 🪞 005 — The Assertion That Compared Itself

**What happened:** One assertion looked suspicious during a full code review: `startDate.isBefore(endDate)`.

**What I found:** Both dates were my own hardcoded test variables — never read from the app at all.

**What I changed:** Removed it. A separate assertion already verified the app's actual displayed date range. ✂️

---

### 🍀 006 — The Wait That "Worked" by Luck

**What happened:** `clickContinue()` passed in TC-02 but sometimes failed when reused in TC-03.

**What I found:** TC-02's prior steps accidentally gave the button time to render; TC-03 called it immediately, with no buffer.

**What I changed:** Added a wait directly inside `clickContinue()` itself, so it no longer depends on what ran before it.

---

### 🔠 007 — The Config File Whose Name Almost Matched

**What happened:** `pom.xml` referenced `testng.xml` (lowercase); the real file was `testNG.xml`.

**What I found:** Windows ignores the case difference — Mac/Linux and CI wouldn't.

**What I changed:** Corrected `pom.xml` to match the real filename exactly. 🧩

---

### 🖥️ 008 — The Test That Ran in the IDE but Not in the Terminal

**What happened:** `ApiTest` passed in IntelliJ, but `mvn clean test` only ran 6 of 7 tests.

**What I found:** `ApiTest` was never added to `testNG.xml`'s class list — IntelliJ's Run button had been masking the gap.

**What I changed:** Added it to `testNG.xml`. Confirmed 7/7 via `mvn clean test`. ✅

---

## 🛠️ Post-100/100 Polish (revisiting the project after Levan's feedback)

### 🚫 001 — The Assertion That Could Never Fail

**What happened:** `isTodayHighlighted()` always passed — but it located the element by the same class it then checked for. It could never fail.

**What I found:** The test proved `findElement` worked, not that today's date was actually highlighted.

**What I changed:** Located the date cell by XPath instead, then asserted the real `aria-current="date"` attribute. 🎯

---

### 🌙 004 — The Locator That Broke Overnight

**What happened:** Four passing tests suddenly failed when I revisited the project weeks later.

**What I found:** The site's `aria-label` date format had changed (`M/d/yyyy` → `dd/MM/yyyy`) — every locator built on that string broke at once.

**What I changed:** Rewrote the locator as XPath and checked `aria-current="date"` instead — an attribute that doesn't depend on date formatting. 🔗

---

### 👻 009 — The Import That Outlived Its Reason

**What happened:** A compile error appeared after revisiting the project — `cannot find symbol` on `DriverManager`.

**What I found:** A leftover static import from before `CalendarPage` inherited `driver` from `BasePage` — now illegal since the field is private.

**What I changed:** Removed the unused import. 🧹

---

### 🗺️ 010 — The Class TestNG Couldn't Find

**What happened:** After a package rename, tests compiled but TestNG found 0 tests to run.

**What I found:** A manual find-and-replace had reversed the class name's segments in `testNG.xml`, out of sync with the real file path.

**What I changed:** Corrected the class name order in `testNG.xml`. 🔄

---

### 📐 011 — The Expected Value That Was Never the Right Shape

**What happened:** A previously-passing test failed: expected `9/3/2026` but found `SEP 2026`.

**What I found:** The expected value was built as a full date, but the element it compared against was only ever a month+year label — the two could never have matched.

**What I changed:** Rebuilt the expected string to match the header's actual format. 📏

---
### 📅 012 - The Date That Was Only Wrong at the End of the Month

**What happened:** Three tests failed at once with `NoSuchElement` on `05/10/2026`, tests that had passed for weeks.

**What I found:** Nothing had changed on the site. My tests pick dates with `plusDays(7)` and `plusDays(13)`, which near the end of a month land in the next month, while the calendar only shows the current one. The dynamic dates were fine, they just never accounted for the month boundary.

**What I changed:** `selectDate()` now checks whether the target date is visible, and if not, clicks "Next month" first. 🗓️

---
