# subscription/expiry Specification

## Purpose

Фоновая проверка сроков подписки: напоминания до окончания и деактивация после истечения. **Прод-канон:** дневные окна 3/2/1 UTC и ежедневный cron утром Europe/Moscow. Временный минутный харнесс (minute-cron + минутные окна) использовался только для живой проверки и **не** является постоянным прод-поведением. Тестовые кнопки выдачи короткой подписки в меню «Подписка» могут оставаться до отдельного product-change.

## Traceability

| Scenario ID | Coverage |
|-------------|----------|
| SC-EXP-01 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-02 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-03 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-04 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-05 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-06 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-T01 | covered (manual; temporary test grant UX) |
| SC-EXP-T02 | covered (manual; temporary test grant UX) |
| SC-EXP-T03 | covered (manual; historical minute harness — not production) |
| SC-EXP-T04 | covered (code restore + pytest production defaults) |

Связь: меню тем — `start/register` (stub «Подписка» может содержать временные тест-кнопки). Gate — будущий `subscription/gate`.

## Requirements

### Requirement: Remind active subscribers before expiry

The system SHALL, on each production daily expiry check, send a Russian reminder to every active user whose subscription end falls in the calendar day that is exactly 3, 2, or 1 UTC day(s) after the check time, stating how many days remain.

#### Scenario [SC-EXP-01]: Reminder three days before end

- **GIVEN** an active user has a non-null subscription end falling in the UTC day that is 3 days after the check time
- **WHEN** the daily expiry check runs
- **THEN** the user receives a Russian message that the subscription expires in 3 days

#### Scenario [SC-EXP-02]: Reminder two days before end

- **GIVEN** an active user has a non-null subscription end falling in the UTC day that is 2 days after the check time
- **WHEN** the daily expiry check runs
- **THEN** the user receives a Russian message that the subscription expires in 2 days

#### Scenario [SC-EXP-03]: Reminder one day before end

- **GIVEN** an active user has a non-null subscription end falling in the UTC day that is 1 day after the check time
- **WHEN** the daily expiry check runs
- **THEN** the user receives a Russian message that the subscription expires in 1 day

#### Scenario [SC-EXP-04]: No reminder without subscription end

- **GIVEN** an active user has no subscription end date
- **WHEN** the daily expiry check runs
- **THEN** the system does not send an expiry reminder to that user
- **AND** does not change that user's active flag

### Requirement: Deactivate expired subscribers and notify

The system SHALL, on each production daily expiry check, mark as inactive every still-active user whose subscription end is strictly before the check time, persist that change, and send a Russian message that the subscription has expired.

#### Scenario [SC-EXP-05]: Expired subscription is deactivated

- **GIVEN** an active user has a subscription end strictly before the check time
- **WHEN** the daily expiry check runs
- **THEN** the user is marked inactive in persistent storage
- **AND** the user receives a Russian message that the subscription has expired

#### Scenario [SC-EXP-06]: Failed delivery to one user does not abort the check

- **GIVEN** more than one user matches reminder or expiry processing in the same check
- **AND** delivering a Telegram message to one of them fails
- **WHEN** the daily expiry check runs
- **THEN** the system continues processing the remaining matching users
- **AND** expiry deactivations already determined for other users are still persisted

### Requirement: Production schedule and day windows

The system SHALL use day-based reminder windows (3/2/1 UTC days) and a once-daily check in the morning Europe/Moscow (`hour=10`, `minute=0`). The system MUST NOT use every-minute cron (`minute="*"`) or minute-scale reminder windows as the permanent production schedule.

#### Scenario [SC-EXP-T04]: Production day mode and daily cron

- **GIVEN** the subscription expiry scheduler is configured for production
- **WHEN** the bot process starts
- **THEN** reminder lead intervals are measured in UTC days (3, 2, and 1)
- **AND** the expiry check is scheduled once daily at 10:00 Europe/Moscow
- **AND** every-minute cron is not the production schedule

### Requirement: Temporary test grant controls in subscription section

The system MAY keep temporary test-only inline actions in the subscription section that set the current user's subscription to active with an end time approximately one minute or five minutes ahead (UTC), without payment, until a product subscription UX replaces them. These controls are **not** payment or tariff product; they do not change the production day-window / daily-cron schedule.

#### Scenario [SC-EXP-T01]: Grant one-minute test subscription

- **GIVEN** a registered user opens the subscription section and temporary test grants are present
- **WHEN** the user chooses the one-minute test subscription action
- **THEN** that user's record is active with a non-null subscription end about one minute after the action time

#### Scenario [SC-EXP-T02]: Grant five-minute test subscription

- **GIVEN** a registered user opens the subscription section and temporary test grants are present
- **WHEN** the user chooses the five-minute test subscription action
- **THEN** that user's record is active with a non-null subscription end about five minutes after the action time

### Historical note (not production MUST)

Minute-scale reminder windows and `minute="*"` cron were used only during temporary live verification (SC-EXP-T03). They are **not** permanent production requirements; production behavior is defined by the day-window and daily-cron requirements above.
