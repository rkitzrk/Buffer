# Problem 1 Summary: Parking Lot

## Scores (out of 5)

| Area | Score | Why |
|---|---|---|
| Requirements gathering | 4 | You covered vehicle types, gates, pricing, payment, and identification. You skipped the full-lot and lost-ticket cases. |
| Class design | 3 | Your relationships were good, but you leaned toward subclassing where an enum would do, and you put some responsibilities in the wrong class. |
| Extensibility | 3 | You got the decorator and DIP quickly. You missed Open/Closed on the EV change and reached for EVTicket and Payment changes first. |
| Code quality | 3 | Your structure was clean, but `ParkingSpot` had no `free()`, an unused parameter, no const, and a race condition. |

## What you did well
- You asked about identification unprompted, which led to Ticket.
- You chose abstract `Gate` correctly, and you came up with the wrapper idea for pricing yourself.
- You named Dependency Inversion correctly and gave the right payoff.
- When a failure came up, you thought about locking and atomicity.
- You were honest when you didn't know something, which is better than bluffing.

## What to fix
1. **Locate the problem before the fix.** You proposed locking inside `PaymentMethod` when the race was on the ticket's state. Ask "what state must change exactly once?" first.
2. **Don't force inheritance.** Subclasses need different behavior, not just different data.
3. **Answer every part of the question.** I often asked two things ("which classes change, and why?") and you answered one. One-word answers like "TicketService" score low.
4. **Name classes and responsibilities.** "Move all to new classes" needs names and one-line jobs.
5. **Cover failure paths early.** Payment failure and expired reservations were both "don't know".

## Polished model design

```
ParkingLot (facade)
 ├─ SpotAllocator ── has many ──> ParkingSpot (id, type, state)
 ├─ TicketService ── has many ──> Ticket (id, Vehicle, spotId, entryTime,
 │                                        state, optional ChargingSession)
 ├─ PricingStrategy <<interface>> ◄── HourlyPricing
 │                               ◄── ChargingFeeDecorator (wraps a strategy)
 ├─ PaymentProcessor ── uses ──> PaymentMethod <<interface>>
 └─ ReservationService ── has many ──> Reservation

Floor has many ParkingSpots; ParkingLot has many Floors.
Payment/Receipt references one Ticket (created later, so it points back).
```

**Interface Segregation (the question I left open):** split the interface by capability.

```cpp
class PaymentMethod {                 // every method supports this
public:
    virtual bool pay(double amount) = 0;
    virtual ~PaymentMethod() = default;
};
class ChangeGiving {                  // only cash needs this
public:
    virtual double giveChange(double paid, double due) = 0;
    virtual ~ChangeGiving() = default;
};
class CardPayment : public PaymentMethod { /* pay only */ };
class UpiPayment  : public PaymentMethod { /* pay only */ };
class CashPayment : public PaymentMethod, public ChangeGiving { /* both */ };
```

Cards and UPI never see `giveChange()`. Code that needs change checks for the `ChangeGiving` capability.

**Core class (thread-safe spot):**

```cpp
enum class SpotState { FREE, RESERVED, OCCUPIED };

class ParkingSpot {
    int id_;
    VehicleType type_;
    std::atomic<SpotState> state_{SpotState::FREE};
public:
    ParkingSpot(int id, VehicleType t) : id_(id), type_(t) {}
    virtual ~ParkingSpot() = default;
    virtual bool canFit(const Vehicle& v) const { return v.getType() == type_; }

    bool tryPark(const Vehicle& v) {
        if (!canFit(v)) return false;
        SpotState e = SpotState::FREE;
        return state_.compare_exchange_strong(e, SpotState::OCCUPIED);
    }
    void free() { state_.store(SpotState::FREE); }
    int getId() const { return id_; }
};
```

## How a strong candidate says it aloud

"First I'd clarify vehicle types, gates, floors, pricing, payment, ticket identity, and what happens when the lot is full. The core entities are ParkingLot, Floor, ParkingSpot, Vehicle, Ticket, and Receipt. I'd use a `VehicleType` enum instead of spot subclasses, and an interface only where behavior varies: `PricingStrategy` and `PaymentMethod`. `ParkingLot` is a thin facade over `SpotAllocator`, `TicketService`, and `PaymentProcessor`, all injected through the constructor. Every state change on a spot or ticket goes through one atomic transition, so two gates can never claim the same spot or exit the same ticket. On exit I claim the ticket first, charge second, and free the spot only after payment succeeds. EV charging is a decorator on pricing plus an optional charging session, so no existing class changes."

## Overall

Your strongest habit is thinking about what can go wrong with shared state. The one to build is giving complete answers: name the class, say what it owns, and say why. Practice saying the "aloud" version of each answer in full sentences.

Say "next" whenever you're ready, and I'll start Problem 2 in strict interviewer mode, where you drive.

---

SESSION STATE
- Mock type: LLD (Amazon-style OOD, campus candidate)
- Rounds done: 1 (coached mode; the next round is strict interviewer mode, where I drive)
- Questions already asked:
  1. Parking Lot (full flow: requirements, entities, relationships, abstractions, signatures, ParkingSpot code, probes on concurrency, EV extensibility, Decorator pricing, payment-failure handling, double-exit race, DIP, SRP split, reservations, ISP)
- Score trend (Problem 1): requirements 4, class design 3, extensibility 3, code quality 3. Average 3.25.
- Top 3 recurring weaknesses:
  1. One-line answers that skip half the question (e.g. "TicketService"), and no naming of classes and their responsibilities.
  2. Fixing the wrong place: proposed a lock in PaymentMethod when the race was on the ticket's state; reached for subclasses where an enum would do.
  3. Failure paths: "don't know" on payment failure and reservation expiry.
- Stories or answers I handled well: Decorator for pricing, Dependency Inversion (naming and payoff), abstract Gate, asking about identification, honesty when unsure. Project links to use: Authorization Service thread pool and connection pool, DLQ routing, Kafka replay.
- Facts I got wrong (correct version):
  - Free a spot only after payment succeeds.
  - Make ticket state transitions atomic (OPEN to PROCESSING to CLOSED).
  - ParkingSpot needs free(), const getters, and no unused parameters.
  - ISP: split by capability (ChangeGiving separate from PaymentMethod).
- Next focus: Problem 2 (new, not Parking Lot) in strict mode: full answers that name classes and their jobs, cover failure paths early, and say the "aloud" version in full sentences.

---
