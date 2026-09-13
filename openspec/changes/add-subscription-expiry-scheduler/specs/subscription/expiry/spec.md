## Purpose

Фоновая проверка сроков подписки: напоминания до окончания и деактивация после истечения. В рамках сдачи change — временный тест-харнесс (короткая выдача срока из меню «Подписка» и минутные окна), затем финальное прод-поведение с дневными окнами.

## Traceability

| Scenario ID | Coverage |
|-------------|----------|
| SC-EXP-01 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-02 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-03 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-04 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-05 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-06 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-T01 | pending (ручная проверка / later pytest) |
| SC-EXP-T02 | pending (ручная проверка / later pytest) |
| SC-EXP-T03 | pending (ручная проверка) |
| SC-EXP-T04 | pending (проверка кода / rg + pytest перед archive) |

Связь: меню тем — `start/register` (stub «Подписка» расширяется только тестовыми кнопками выдачи). Gate — будущий `subscription/gate`.

## ADDED Requirements

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

### Requirement: Temporary test grant controls in subscription section

The system SHALL, while the temporary verification harness is enabled, offer exactly two inline actions in the subscription section that set the current user's subscription to active with an end time approximately one minute or five minutes ahead (UTC), without payment.

#### Scenario [SC-EXP-T01]: Grant one-minute test subscription

- **GIVEN** a registered user opens the subscription section
- **WHEN** the user chooses the one-minute test subscription action
- **THEN** that user's record is active with a non-null subscription end about one minute after the action time

#### Scenario [SC-EXP-T02]: Grant five-minute test subscription

- **GIVEN** a registered user opens the subscription section
- **WHEN** the user chooses the five-minute test subscription action
- **THEN** that user's record is active with a non-null subscription end about five minutes after the action time

### Requirement: Temporary minute-scale reminder windows for verification

The system SHALL, while the temporary verification harness is enabled, treat reminder lead intervals as 3, 2, and 1 UTC minute(s) (half-open windows analogous to the day-based production rule) and run the expiry check about once per minute, so a five-minute test subscription can receive a soon-to-expire reminder when roughly one minute remains.

#### Scenario [SC-EXP-T03]: Five-minute grant yields soon-expiry reminder then expiry

- **GIVEN** the temporary verification harness is enabled
- **AND** an active user has a subscription end about five minutes after grant time
- **WHEN** expiry checks run about once per minute
- **THEN** the user receives a Russian soon-to-expire reminder when the end falls in the one-minute-ahead window
- **AND** after the end time passes, the user is marked inactive and receives a Russian expired message

### Requirement: Production schedule and day windows after verification

Before the change is considered complete for production, the system SHALL use day-based reminder windows (3/2/1 UTC days) and a once-daily check in the morning Europe/Moscow, and MUST NOT leave every-minute cron as the permanent schedule.

#### Scenario [SC-EXP-T04]: Production mode restored after verification

- **GIVEN** the temporary verification harness has been used to observe reminders and expiry
- **WHEN** the change is prepared for production completion
- **THEN** reminder lead intervals are measured in UTC days (3, 2, and 1)
- **AND** the expiry check is scheduled once daily in the morning Europe/Moscow
- **AND** every-minute cron is not the permanent schedule
