# Daily Dose

A mobile prototype for the Acorn Labs mobile engineering challenge, **Part B: the engagement core**. It covers the screen a patient opens every day (streak, weekly and monthly adherence, upcoming doses) and a just-in-time adaptive intervention (JITAI) layer. That layer learns which nudge works for each person and stops sending the ones that don't.

**Live app:** https://claude.ai/artifact/79m4Ff45fvEzsYubSK5LEQ (opens on any phone or laptop)

Built during the 4-hour on-site session on 7 Oct 2026.

## Try it

Open the live link on a phone, or on a laptop, where the app shows in a phone-sized frame.

**A 2-minute tour:**

1. **Home.** The streak is in the centre. Tap **Take now** on the next dose and watch the ring fill.
2. **Family.** Switch to **Mum's phone**. The dose you just logged is in her notifications. Try her "Notify me" options, then tap **Send a cheer** and go back to Home.
3. **Medicines → + Add medicines.**
   - Search "diabetes" and pick Gliclazide.
   - Choose the strength on the label and set a reminder time.
   - Add it.
4. **Nudges.** Pick a persona, tap **Run 60 days** and compare the three policies. Switch between Morning, Midday and Evening to see what the engine learned.
5. **Learn.** Short articles on missed doses, timing, side effects and supporting a family member.

## What's in it

### Home

The streak number sits in the middle of a ring that fills as today's doses are logged. Below it, from top to bottom:

- the last 7 days;
- adherence this week and this month;
- the next dose, with Take and Skip;
- the nudge the engine would send if that dose isn't logged within an hour, and whether family will be alerted ("Why this nudge?" opens the engine);
- a 30-day heatmap with the patient's weakest time of day.

A day counts towards the streak when every dose is taken. One day a week with a single missed dose is covered by a grace day. A long streak reset by one slip is a common point where people give up.

Adherence is measured per dose: doses taken ÷ doses scheduled over the last 7 or 30 days.

### Medicines

This tab shows today's doses with Take and Undo, and the patient's medicines with their 30-day adherence and editable reminder times.

**+ Add medicines** opens a 4-step flow:

1. Search and pick conditions (8 common long-term conditions).
2. Pick from the medicines usually prescribed for them (12 medicines), or add one by name.
3. Choose the strength printed on the prescription label, how many times a day, and the reminder times.
4. Review and add.

The app shows the usual regimen, for example "Usually 1–2 times a day, with meals", but it never fills in a dose. The patient copies the strength from the label.

- A strength is preselected only when a medicine comes in a single strength.
- As-needed medicines, such as a reliever inhaler, get no reminders.

I kept the prescription as the source of truth for two reasons. It is safer, and an app that recommends doses moves towards clinical decision support, which HSA regulates (see [Medical and privacy notes](#medical-and-privacy-notes)).

### Family

A relative installs the app, enters a one-time invite code and follows the patient. The toggle at the top of the tab switches between the patient's phone and Mum's phone, so both sides can be shown on one device.

The patient controls three switches:

- a notification to family for each dose taken;
- whether medicine names are shown (when off, family sees "the 8 PM dose");
- an alert to family when a dose still isn't logged an hour after its time, sent at the same moment as the patient's nudge.

The family member chooses whether to hear about every dose, only late doses, or get a daily summary at 9 PM. They can also send a cheer, which appears on the patient's home screen. The aim is encouragement, not monitoring.

### Learn

Seven short articles:

- why every dose matters;
- what to do after a missed dose;
- timing (with food, at night);
- side effects;
- supporting a family member;
- questions for your pharmacist;
- storing medicines in a hot, humid climate.

They are general information with a disclaimer, and would need a pharmacist's review before launch.

## The nudge engine

Take a dose scheduled for 8 PM:

1. **8 PM, the reminder.** It always fires at the time the patient set, as a local notification that works offline.
2. **9 PM, if the dose still isn't logged: the nudge.** The engine decides whether to follow up and what the message says. This is the adaptive part.
3. **9 PM, at the same moment: the family alert.** If the patient has turned it on, family members are told the dose hasn't been logged. This is a fixed safety rule, not learned. Its job is to keep family informed, not to persuade.

The timing is deliberately fixed and predictable, so patients and families know what to expect. What adapts is *whether* the patient gets a nudge and *which* one.

| JITAI component | In this prototype |
|---|---|
| Decision point | A dose still not logged 1 hour after its scheduled time |
| Tailoring variables | Time slot (morning before 11:00, midday, evening from 17:00), current streak, whether yesterday had a miss, nudges already sent today |
| Intervention options | No nudge · Reminder · Streak · Progress ("you're at 92% this month") · Fresh start (only after a day with a miss) |
| Decision rule | Thompson sampling: each user has a Beta belief for each time slot × option pair |
| Proximal outcome | The dose gets logged after the decision |
| Distal outcome | Adherence over weeks, and retention |

### How it learns what works for this person

Each option keeps a Beta(α, β) belief about its success rate in each time slot.

1. At a decision point, the engine draws one sample from each eligible option's belief and sends the option with the highest draw.
2. If the dose gets logged, that option's α goes up. If not, its β goes up.

Options with little data have wide beliefs, so they still win some draws and keep getting tried. Options that keep working win most draws.

New users start from a population prior, a mild belief that a plain reminder works, so the engine behaves sensibly from day one.

### How it stops sending nudges that don't work

- **Failing options win fewer draws.** Their belief shifts down. They are never fully retired, so they can come back if things change.
- **Silence is an option.** *No nudge* competes like any other option. A nudge that only gets a dose the patient would probably have logged anyway scores 0.8 instead of 1, so silence wins for people who don't need help.
- **Daily cap.** The engine sends at most 2 nudges a day, then stays quiet.
- **Slow forgetting.** Beliefs fade by 0.5% per decision, so the engine can follow changing habits.

The Nudges tab shows:

- each option's belief per time slot (the mean and a 90% interval);
- the chance each option gets picked;
- a 60-day simulation;
- the most recent decisions.

### Simulation results

There are four simulated personas. Each has hidden response rates for every time slot and option. The engine never sees these. It only sees whether a dose got logged after each decision.

Each run is 5 doses a day for 60 days, averaged over 50 simulated users per persona. Every row can be re-run in the Nudges tab.

| Persona (hidden preference) | Adaptive engine | Always remind | No nudges |
|---|---|---|---|
| The Streaker (streak messages) | **78.3%** · 101 nudges | 71.1% · 104 | 63.7% |
| Data lover (monthly progress) | **76.0%** · 105 | 69.9% · 108 | 61.0% |
| Night Owl (evening reminders, ignores mornings) | 64.0% · 105 | **65.0%** · 111 | 58.8% |
| Self-starter (logs late but reliably on their own) | 95.4% · **60** | 95.4% · 107 | 95.9% |

*Adherence = doses taken ÷ doses scheduled. Nudge counts are per user over 60 days, which is 300 doses.*

What this shows:

- **When one message works much better than a plain reminder, the engine finds it** and gains 6–7 points.
  - For one simulated Streaker, streak messages rose from 9% of decisions in days 1–10 to 41% in days 51–60.
  - For one simulated Data lover, progress messages rose from 20% to 55%.
- **The Self-starter gets the same adherence with 44% fewer nudges.** By days 51–60 the engine chose *No nudge* in 77% of its decisions.
- **When a plain reminder is already the best option, the engine costs about a point.** That is the case for the Night Owl in the evening, so "always remind" is close to the best possible policy, and the engine pays about a point for exploring. The Night Owl's real problem is mornings, where no message works. That needs the reminder moved to a better time, not a better message (see [What I'd build next](#what-id-build-next)).

**The simulation caught a design flaw.**

- My first version only allowed streak messages once a streak reached 3 days.
- The Streaker, who responds best to those messages, rarely got there. One missed dose reset the streak, and 2 nudges a day were not enough to protect it.
- The engine learned that streak messages worked but almost never got to send them. The Streaker did no better than with plain reminders.

Streak messages are now always available, with copy that fits the streak length:

- "Start a new streak today"
- "You're on a 2-day streak"
- "Your 12-day streak is still going"

The simulator also now uses the app's grace-day rule. Together, these changes took the Streaker from roughly level with "always remind" to 7 points ahead.

**Limits of the simulation:**

- The personas are my assumptions, and their preferences don't change over time. So this shows that the mechanism works, not what the effect would be with real patients.
- The 0.8 score for a nudge the patient didn't need relies on something only a simulation can know. A real system would charge every nudge a small fixed cost instead.

## Why a web app, and how it maps to Flutter and Firebase

Acorn prefers Flutter. I built a single-file web app (plain JavaScript, no build step) and made it installable as a Progressive Web App, for three reasons:

- In a 4-hour window I wanted the time to go into the adaptation logic and its evaluation, which the brief says matters most, not into toolchain and device setup.
- Judges can open it on any phone or laptop from a link, and it can be installed to a phone's home screen.
- The engine and simulator are under 100 lines of pure functions with no UI dependencies, so they port directly to a Dart class or a Cloud Function.

The production version on Acorn's stack would look like this.

**App (Flutter)**

- One widget per tab.
- A `NudgeEngine` Dart class with `choose(context)` and `update(option, reward)` methods, covered by unit tests.
- The patient's own reminder times scheduled as local notifications (`flutter_local_notifications`), so they fire offline.

**Decisions (Cloud Functions and Cloud Tasks)**

- A task fires 1 hour after each scheduled dose. If the dose isn't logged, it builds the context, samples an option and sends it to the patient through FCM. It also sends the family alert to members who should get it.
- Running this on the server keeps one source of truth across devices, and lets the policy change without an app release.

**Logging**

- Each decision is stored with its context, the option sent, and the probability that option had of being picked (the same estimate the Nudges tab shows). This makes it possible to evaluate new policies offline later.
- A `dose_logged` event closes the open decision and updates the beliefs in a transaction.

**Data (Firestore)**

- `users/{uid}/meds`, `doseEvents`, `engine/state` (30 numbers: 3 slots × 5 options × α and β) and `decisions`.
- `circles/{patientUid}/members/{memberUid}`, holding the relationship, status and notification preference.
- `invites/{code}`: single-use and expiring. Invites are claimed through a callable function and never written directly by clients.

**Family privacy**

Firestore security rules work per document, so they can't hide a single field such as the medicine name. Instead, a function writes a separate `sharedDoseEvents` feed containing only what the patient chose to share. Family members can read only that feed.

**Config and analytics**

- Remote Config for the nudge cap, forgetting rate and priors.
- Firebase Analytics events (`nudge_sent`, `dose_logged`) exported to BigQuery for evaluation.

## What I couldn't do, and why

- **No native build or real push notifications.** These need a Firebase project, app signing and device testing, which didn't fit in 4 hours. The installable web app and the two-phone toggle stand in for them.
- **No real accounts or linking.** The invite code and the "they join" step are simulated. There is no way yet to remove a family member.
- **Nothing is saved.** State lives in memory and resets on reload. The 30-day history is generated sample data.
- **A simpler decision point.** It uses the scheduled time, not each person's learned usual time.
- **Unreviewed content.**
  - The condition-to-medicine list (8 conditions, 12 medicines) is hand-written, not taken from a drug database.
  - Neither the list nor the articles have had a pharmacist's review.
  - The nudge wording hasn't been tested with users.
  - Quiet hours and an accessibility pass (for example, large text for older users) aren't done.
- **Part A not attempted.** The manual add-medicine flow is where reading a prescription label or importing health records would plug in.

## What I'd build next

1. **Port to Flutter and Firebase** as described above, starting with the engine and its tests.
2. **Learned habit timing.**
   - Decide at each person's usual logging time, for example the median of their last 14 logs for that dose.
   - Suggest moving reminder times that keep failing. This targets the Night Owl's mornings.
3. **Realistic rewards.**
   - Charge a fixed cost per nudge instead of using the simulated "would have logged anyway" score.
   - Count muting notifications or uninstalling as a strong negative.
4. **Richer context.**
   - Use day of week, minutes late and recent adherence as features, for example with linear Thompson sampling.
   - Add a hierarchical prior so new users borrow from similar users.
5. **Evaluate in the beta pilot with a micro-randomised trial.** Randomise at each decision point and log the probabilities. This is the standard way to measure each nudge type's effect within a JITAI.
6. **Learn when family alerts help.** Track whether doses get logged after a family alert. If alerts rarely help for a patient, suggest switching that family member to the daily summary, so the alerts that do go out stay meaningful.
7. **Line up with Acorn's existing rules and test change over time.**
   - Map the options and rewards onto Acorn's existing nudge and reward rules.
   - Add personas whose preferences change, for example streak messages wearing off after a month, to test the forgetting rate.

## Medical and privacy notes

- **Doses come from the prescription.**
  - HSA has guidelines on classifying standalone medical mobile apps, and on when clinical decision support software counts as a medical device (consultation draft, July 2021).
  - Under those guidelines, risk depends on how much the app's output drives a healthcare decision and how serious the patient's condition is.
  - Reminding someone to take what was prescribed is further from that line than suggesting a dose.
  - Where this app would actually sit still needs checking against HSA's current guidance.
- **Sharing with family is sharing health data.** Nothing is shared until the patient invites someone. The patient chooses what is shared and can change it at any time. Under the PDPA, that consent must be clear and possible to withdraw.
- **No shaming in nudges.** The missed-dose content never suggests taking a double dose.

## Files

| File | What it is |
|---|---|
| `index.html` | The whole app: UI, sample data, nudge engine and simulator |
| `manifest.webmanifest`, `sw.js`, `icon-*.png`, `apple-touch-icon.png` | Make the app installable and usable offline |

To run it locally, open `index.html` in a browser.

It can also run as an installable phone app. Turn on GitHub Pages for this repo (Settings → Pages → Deploy from a branch → `main`), open the Pages link on a phone and choose **Add to Home Screen**. The manifest, offline support and icons are already included.

## How this was built

I built this with help from Claude, Anthropic's AI assistant. It wrote much of the code and drafted this README, following my direction on features and design.

## References

- Nahum-Shani et al. (2018). Just-in-Time Adaptive Interventions (JITAIs) in Mobile Health: Key Components and Design Principles for Ongoing Health Behavior Support. *Annals of Behavioral Medicine*.
- Klasnja et al. (2015). Microrandomized trials: An experimental design for developing just-in-time adaptive interventions. *Health Psychology*.
- Russo et al. (2018). A Tutorial on Thompson Sampling. *Foundations and Trends in Machine Learning*.
- HSA: [Consultation on regulatory guidelines for classification of standalone medical mobile applications (SaMD) and qualification of clinical decision support software (CDSS)](https://www.hsa.gov.sg/announcements/consultation-on-regulatory-guidelines-for-classification-of-standalone-medical-mobile-applications--samd--and-qualification-of-clinical-decision-support-software--cdss-/)
