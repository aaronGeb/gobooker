# Concurrent Book Reservation System

## Overview
This service demonstrates safe concurrent reservations of books using:
- **Goroutines** to perform processing concurrently.
- **Channels** to queue reservation processing tasks.
- **sync.Mutex** to guard mutable state and prevent race conditions.

## Design Details

### Data Model
- `models.Book` contains fields:
  - `Available bool`
  - `ReservedBy int` (0 if not reserved)
  - `Borrowed bool`
  - `mu sync.Mutex` to protect the book's fields

- `models.Member` is simple (ID and Name).

### Reservation Flow
1. **Synchronous reservation check (ReserveBook)**:
   - A caller calls `ReserveBook(bookID, memberID)`.
   - The service fetches the `Book` and locks its mutex.
   - If the book is available and not reserved/borrowed, it's *immediately marked as reserved* (`ReservedBy = memberID`).
   - This synchronous reservation prevents races where two callers could both think they reserved the same book.

2. **Asynchronous processing (borrowing)**:
   - After marking the book as reserved, `ReserveBook` enqueues a `ReservationRequest` on a buffered `reqChan`.
   - A worker (in `concurrency/reservation_worker.go`) listens on `reqChan`.
   - For each request, the worker spawns a goroutine which calls the provided handler (`processBorrow`) to complete the borrow operation.
   - The borrow handler verifies that the book is still reserved by the same member and not already borrowed; if so, it marks the book as `Borrowed = true` and clears the reservation.

3. **Auto-Cancellation**:
   - `ReserveBook` also starts a 5-second timer goroutine to auto-cancel the reservation if the book is not borrowed within 5 seconds.
   - The timer checks the book under lock and clears `ReservedBy` if the reservation still belongs to the original member and the book is not borrowed.

### Concurrency Primitives Used
- **Mutex (per-book)**: All updates to a specific book (reserve, unreserve, borrow) happen while holding that book's `mu` lock. This provides fine-grained locking and avoids serializing all book operations on a single global lock.
- **RWMutex (books map)**: Access to the `books` map is protected by `booksMu` (allows concurrent readers).
- **Channels**: `reqChan` is used to queue borrow processing requests decoupling reservation from borrow processing.
- **Goroutines**: Worker spawns goroutines for each queued request; timers for auto-cancel are separate goroutines.

## Safety & Race Avoidance
- The immediate reservation assignment in `ReserveBook` ensures only one caller can successfully reserve a book at a time.
- The `processBorrow` handler confirms that the book is still reserved by that member — guarding against situations where a reservation timed out and was taken by someone else.
- The auto-cancel timer double-checks under lock to safely clear reservations.

## Notes & Extensions
- The worker currently spawns one goroutine per request. For very high throughput, consider a bounded worker pool.
- If persistence is required, adapt Acquire/Release semantics to the underlying store or use transactions.
- The `reqChan` buffer size can be adjusted depending on workload.

