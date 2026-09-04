# Lab 2 Starter: Availability Calculator

A small reservation component. Given a room's bookings and the day's business hours,
`AvailabilityCalculator.freeSlots` computes when the room is free. It is the code you
work in for Lab 2.

It ships with a generated test suite that passes, and a property-based test harness
(jqwik) with one example property. Everything is green. Your job in Lab 2 is to decide
whether green actually means correct.

**Read `ARCHITECTURE.md` before the code.**

## Build and test

```
mvn test
```

`mvn test` runs both files, the ordinary example-based tests (`AvailabilityCalculatorTest`)
and the property-based tests (`AvailabilityProperties`). A code-coverage report is written
to `target/site/jacoco/index.html`.

## Continuous integration

This repository has CI configured in `.github/workflows/ci.yml`. GitHub disables workflows on a
fresh fork, so enable them once on your fork (the handout shows where). After that, every
push runs `mvn test`. You will watch the gate go red when your new property finds the bug, then
green once you fix it.

## Where things are

- Component: `src/main/java/edu/cmu/cs214/availability/`
- Example-based tests: `src/test/java/edu/cmu/cs214/availability/AvailabilityCalculatorTest.java`
- Property-based tests: `src/test/java/edu/cmu/cs214/availability/AvailabilityProperties.java`
- Setup: `SETUP.md`

See the Lab 2 handout on the course page for the three milestones you show a TA.

## Milestone 1: the stronger property and its failing sample

The provided `freeSlotsNeverOverlapABooking` only checks that returned slots are
genuinely free; it says nothing about free time the calculator fails to return, so an
empty (or truncated) result list passes it trivially. `everyMinuteIsBookedXorFree`
in `AvailabilityProperties.java` pins down full correctness instead: every minute of
`[dayStart, dayEnd)` must be booked or reported free, never both, never neither.

Run against the pre-fix calculator, jqwik shrinks the failure to:

```
Scenario[dayStart=0, dayEnd=1, bookings=[]]
```

For this input `freeSlots(0, 1, [])` returned `[]` instead of `[TimeInterval(0, 1)]`.

- `freeSlotsNeverOverlapABooking` still passes on it: with zero returned slots there is
  nothing to overlap a booking, so the assertion holds vacuously.
- `everyMinuteIsBookedXorFree` fails: minute 0 is not covered by any booking and is not
  covered by any returned slot either, so `booked ^ reportedFree` is `false ^ false`,
  violating the "exactly one holds" invariant.

## Milestone 2: fix the bug

`freeSlots` merged bookings into free gaps as it walked the sorted, clipped booking
list, but nothing ran after that loop to flush the gap between the last booking's end
(or `dayStart`, if there were no bookings at all) and `dayEnd`. On the Milestone 1
sample this meant the entire day was silently dropped instead of reported free.

Pushing the property alone (commit `3fb3f26`) turned CI red; adding the missing flush
step after the loop (commit `35b4a43`) turned it green again, without weakening the
property.

## Milestone 3: why the generated suite stayed green over a real bug

`AvailabilityCalculator.freeSlots` never emitted the free interval between the last
booking's end (or `dayStart`, if there were no bookings) and `dayEnd` — nothing ran
after the merge loop to flush that trailing gap. `AvailabilityCalculatorTest` had 100%
JaCoCo instruction and branch coverage and stayed green through it anyway. Three
concrete weaknesses:

1. **Controllability gap.** No test ever calls `freeSlots` with an empty bookings list.
   The bug's simplest trigger — `freeSlots(dayStart, dayEnd, List.of())`, which should
   return the whole day as one free slot — is never constructed.
2. **Controllability gap.** In every test that does supply bookings
   (`fullyBookedDayHasNoFreeSlots`, `bookingUntilEndOfDayLeavesTheMorningFree`,
   `gapsBetweenBookingsAreReturned`, `unsortedBookingsAreHandled`,
   `overlappingBookingsAreMerged`), the last booking after merging ends exactly at
   `dayEnd`. So `cursor` always reaches `dayEnd` by the end of the loop, and the
   missing flush step is a no-op on every one of these inputs — the exact state that
   would expose it is never driven.
3. **Observability gap.** `returnedSlotsNeverOverlapABooking` does drive the right
   input (a booking `(600, 660)` that ends before `dayEnd = 1020`, so the trailing gap
   really is dropped), but its assertion only checks that returned slots don't overlap
   the booking. That's satisfied just as well by an empty or truncated free list as by
   a correct one, so even though the buggy code path ran, the assertion couldn't see
   that a slot was missing.

High coverage didn't save the suite because coverage measures whether the lines and
branches *that exist* executed — it has nothing to say about a missing statement.
Weaknesses 1 and 2 mean the trailing-flush code path was never a required behavior for
any test input (an omission, not a wrong branch, so there was no line for coverage to
miss). Weakness 3 shows that even landing on the right input isn't enough without an
assertion strong enough to detect the deviation. The property-based test that catches
this, `AvailabilityProperties.everyMinuteIsBookedXorFree`, closes both gaps at once: it
generates day/booking combinations at random (so it isn't limited to "nice" boundary
cases) and asserts a completeness invariant — every minute is booked or free, never
both, never neither — instead of checking specific input/output pairs.

## Tools used

Claude Code (Sonnet 5, model id `claude-sonnet-5`) was used to write the
`everyMinuteIsBookedXorFree` property, reproduce its shrunk failing sample, diagnose
and fix the `freeSlots` bug it found, and draft the Milestone 1-3 writeups.
