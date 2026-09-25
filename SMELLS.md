# reservation-service: Smells and One Fix

Fill in each section. One section per milestone. Keep it short and specific. Point at files
and methods, not adjectives.

---

## Milestone 1: Three smells

Three smells, each in a different part of the module. For each one, fill in all five parts.

### Smell 1

**The smell.** Duplicated business rule. The pricing formula (hourly rate, premium
surcharge ×1.15, long booking discount ×0.9 at 180+ minutes, evening discount ×0.95 from
17:00) is written out twice, with its own copy of the constants under different names
(`PREMIUM_MULTIPLIER` vs `PREMIUM_RATE_MULTIPLIER`, `EVENING_START_MINUTE` vs
`EVENING_CUTOFF`, and so on).

**Classic or agent-specific.** Agent-specific: re-implementation instead of reuse. The
cause is limited context: `ReportGenerator` was written as if `ReservationManager` did not
exist. The pricing already lived in `ReservationManager.calculatePrice`, which is public,
and the report re-derived it rather than calling it. The renamed constants give it away.
Someone copy-pasting by hand keeps the names. A generator that never saw the first copy
invents new ones.

**Where in the code.** `src/reservationManager.ts`, `calculatePrice` + `applyDiscounts`
(and the five constants at the top), and `src/reportGenerator.ts`, `priceOf` (and its five
constants).

**The principle it violates.** DRY / single source of truth. A business rule should have
one authoritative definition. Here "what a booking costs" has two.

**What it makes expensive.** Any pricing change, for example moving the evening cutoff to
18:00 or adding a weekend rate. You have to find and edit both copies. If you miss the one
in `ReportGenerator`, nothing fails loudly. The revenue report just quietly disagrees with
what customers were charged in `booking.priceCents`. The suite would only catch this for
the exact bookings in `reporting.test.ts` ("totals revenue over the window"). No test
prices a premium or long booking through the report.

### Smell 2

**The smell.** Dead / half-wired infrastructure (speculative generality). There is a
`QueryCache` with TTL, max entries and eviction, but nothing ever writes to it.
`listBookingsForRoom` calls `cache.get`, which always misses, and then falls through to
storage. Nothing in the module calls `cache.set` or `cache.invalidate`.

**Classic or agent-specific.** Agent-specific: plausible scaffolding that was never
connected. It looks like a finished feature (a config file, `withTtl`, `disabled`,
eviction), so a reader assumes caching works. The cause is the agent pattern-matching on
"services like this have a cache" without checking that the read path and the write path
were both wired.

**Where in the code.** `src/cache/queryCache.ts`, `src/cache/cacheConfig.ts`, and
`src/reservationManager.ts`: the constructor (`new QueryCache(DEFAULT_CACHE_CONFIG)`) and
`listBookingsForRoom`.

**The principle it violates.** YAGNI (don't build what isn't used). More seriously, the
cache has no invalidation owner. A cache must be invalidated by whoever writes the data,
and `createBooking` / `cancelBooking` don't know it exists.

**What it makes expensive.** The obvious "finish the cache" change is to add
`this.cache.set(cacheKey, result)` in `listBookingsForRoom`. It passes a quick review and
introduces a correctness bug. After `cancelBooking`, `formatDailySummary` would keep showing
the booking as `confirmed` and count it in "Confirmed total" for up to 30 seconds. After
`createBooking`, the new booking would be missing. Today it just costs reading time and a
false belief that reads are cached.

### Smell 3

**The smell.** Divergent change: presentation is mixed into the domain class.
`ReservationManager` owns booking rules (conflicts, lifecycle) and also the customer-facing
text (receipt layout, schedule layout, clock format, money format). It also builds its own
notifier from a hard-coded default config instead of receiving one.

**Classic or agent-specific.** Classic. It is the textbook "one class changes for many
unrelated reasons" smell. The class doc comment admits it: "holds the room registry, creates
and cancels bookings, prices them, sends the confirmations, and formats the receipts and
summaries."

**Where in the code.** `src/reservationManager.ts`: `formatReceipt`, `formatDailySummary`,
`formatClock`, `formatMoney`, `dispatchNotification`, and the constructor line
`this.notifier = createNotificationChannel(DEFAULT_NOTIFIER_CONFIG)`.

**The principle it violates.** Single Responsibility / separation of concerns (domain logic
vs. presentation). The constructor also violates Dependency Inversion. The manager depends
on a concrete, globally configured channel rather than an injected `NotificationChannel`.

**What it makes expensive.** Changing how a receipt looks (a different currency format,
24h → 12h clock, an SMS-length receipt) means editing and retesting the class that decides
whether bookings conflict. The formatting is also private, so it can't be reused.
`ReportGenerator` has no way to print `$234.00` the same way. Because the notifier is
hard-wired, a test cannot swap in a fake channel to assert what the email said. The only
thing observable is the `recentNotifications()` string log. `formatClock` is even used inside
the conflict error message in `createBooking`, so the domain error text depends on a display
helper.

---

## Milestone 2: One small fix

One fix, behavior preserved, suite green, zero test edits.

**Which smell you attacked.** Smell 1, the duplicated pricing rule. It is the only one of
the three where the two copies can silently drift apart and produce wrong numbers
(revenue ≠ what was charged). It can also be fixed with a small, purely mechanical change
because both copies are provably the same formula: same constants, same order (premium,
then long, then evening), same `Math.round` after each step.

**What changed.**
- New `src/pricing.ts` exports `priceFor(room, start, end)`. It holds the one copy of the
  formula and the only copy of the five pricing constants.
- `src/reservationManager.ts`: `calculatePrice` now just returns
  `priceFor(room, start, end)`. It stays public with the same signature so no caller changes.
  The private `applyDiscounts` and the five constants were deleted.
- `src/reportGenerator.ts`: `revenue` calls `priceFor(room, booking.start, booking.end)`.
  The private `priceOf` and `durationOf` (only used by `priceOf`) and the five duplicate
  constants were deleted. The now-unused `Booking` type import was removed as well.

The code computes the same numbers as before. There is just one place that computes them.

**What you deliberately did not touch.** Scope line: *remove the duplication, don't change
what gets computed.*
- `ReportGenerator.revenue` still re-prices each booking from the room's current rate
  instead of summing `booking.priceCents`. That is arguably a second bug: if a room's rate
  changes, historic revenue changes too. But switching to `priceCents` changes behavior, so
  it's a separate, deliberate decision and not part of a refactor.
- The overlap rule is also written three ways (`ReservationManager.hasConflict`,
  `ReportGenerator.overlapsWindow`, `availability.isSlotFree`). It is the same kind of
  smell, but a different rule with different call sites. Folding it in would double the diff
  and mix two changes in one review.
- `ReportGenerator` taking a snapshot `Room[]`, the cache (Smell 2), and the
  formatting/notifier split (Smell 3) are all left alone.
I stopped at the point where every remaining line of the diff is "delete a copy, call the
original". Anything past that would need its own argument about behavior.

**How you know behavior is preserved.** `npm test` stays at 39/39 and `npm run typecheck`
passes, with no test files modified. What the suite actually covers here: the four
`pricing` tests in `booking.test.ts` pin plain (12000), long (16200), premium (18400) and
evening (11400) prices through `calculatePrice`. The revenue tests in `reporting.test.ts`
pin the report's total to the sum of `priceCents` for a plain + evening booking, and check
that cancelled bookings are excluded. What it would *not* catch: combined rules (premium +
long + evening in one booking, where rounding order matters), or a long or premium booking
going through `revenue`. I relied on reading the two old copies side by side for those.
They had the same steps in the same order, so a shared function cannot differ from either.

---

## Milestone 3: Two proposals and one false positive

One proposal for each milestone 1 smell you did not fix.

### Proposal A (not coded): the unwired cache (Smell 2)

**The problem.** A read-through cache sits inside `ReservationManager` with no writer and no
invalidation. It does nothing today, and the natural way to "turn it on" makes the service
return stale bookings.

**The decomposition.** Take caching out of `ReservationManager` completely (delete the
`cache` field and the `cache.get` in `listBookingsForRoom`). If caching is ever actually
needed, put it at the storage boundary as a decorator: `CachingStorageProvider implements
StorageProvider`. It wraps another `StorageProvider`, serves `findByRoom` / `findAll` from a
`QueryCache`, and invalidates `bookings:<roomId>` and `all` inside its own `save`, `update`
and `clear`.
- `ReservationManager` owns booking rules and knows nothing about caching.
- `CachingStorageProvider` owns the invalidation rule: *every write that passes through
  me drops the keys it affects*. That rule lives next to the writes, which is the only
  place it can be enforced.
- `QueryCache` stays a dumb key/value store with TTL.
Callers choose caching at wiring time: `new ReservationManager(new CachingStorageProvider(new
InMemoryStorageProvider()))`.

**One cost.** The decorator must mirror every write method on `StorageProvider`, and it is
only correct if *all* writes go through it. A second process or a direct DB write would
leave it stale until the TTL expires. It also changes an aliasing guarantee.
`InMemoryStorageProvider` hands back fresh copies on every read, while a cache would hand
back the same array to every caller. One caller mutating a result would then corrupt it for
everyone, unless the cache copies too (and that eats most of the saving).

### Proposal B (not coded): presentation and notification inside the manager (Smell 3)

**The problem.** `ReservationManager` changes for domain reasons (conflict rules, lifecycle)
and for presentation reasons (receipt/summary text, clock and money format). It also builds
its own notifier, so the channel can't be substituted.

**The decomposition.**
- `BookingFormatter`: `formatReceipt(booking, room)`, `formatDailySummary(room, bookings)`,
  `formatClock`, `formatMoney`. A pure function of its inputs with no storage access. It owns
  every rule about how things *look*. `ReportGenerator` can reuse `formatMoney`.
- `BookingNotifier`: takes an injected `NotificationChannel` and a `BookingFormatter`,
  exposes `confirmed(booking, room)` / `cancelled(booking, room)`, and keeps the
  notification log. It owns *who gets told what*.
- `ReservationManager`: keeps the room registry, `createBooking`, `cancelBooking` and the
  conflict check. It receives `StorageProvider`, `BookingNotifier` (and later a pricing
  policy) through the constructor, and owns *whether a booking is allowed and what state it
  is in*.
To keep the public API (and the tests) unchanged, `ReservationManager` keeps thin
`formatReceipt` / `formatDailySummary` / `recentNotifications` methods that delegate, with
defaults in the constructor so `new ReservationManager(storage)` still works.

**One cost.** The conflict error message in `createBooking` ("already booked from 09:00 to
11:00") uses `formatClock`. So after the split the domain class either depends on the
formatter anyway (the dependency we were trying to remove, now explicit) or the error changes
to raw minutes, which is a visible behavior change. On top of that, the delegating
facade methods are pure indirection that has to be kept in sync, and wiring goes from one
constructor argument to three objects.

### The thing that looks smelly but is fine

**What it is.** `src/validation.ts`, `validateReservationRequest`. It is one long function
with eleven `if` checks in a row and section comments. At a glance that's "long method" or a
candidate for a rule-object / chain-of-responsibility refactor.

**Why it is fine.** It is a pure function with no state, no I/O and no dependency on storage.
Each check is independent and one line long. The only thing tying them together is *order*,
and the order is the contract. The doc comment says it returns on the first failure "so the
caller can report one clear reason". The order is also meaningful: shape checks come before
time checks, so `end - start` is only computed once both are known to be integers with
`end > start`. Reading top to bottom gives you the exact precedence. Splitting it into rule
objects would scatter that ordering across files and put it in a list somewhere else, making
it harder to see. It is also well tested. `validation.test.ts` has a table-driven case for
every rejection path plus the boundary accepts. Its length comes from the number of rules,
not from any tangling between them.

**What would flip your verdict.** Rules becoming *variable* instead of fixed. The section
labelled "Booking rules for this building" is the seam. If a second building needed different
opening hours, a different 15-minute boundary, or a different premium minimum, this function
would grow `if (building === ...)` branches. At that point the rules are data that varies per
room/building and belong in a per-building policy object. The same goes if a rule starts
needing storage (e.g. "an organizer may hold at most 3 bookings a day"): the function would
stop being pure, and first-failure ordering against I/O-backed rules becomes a real design
question.
