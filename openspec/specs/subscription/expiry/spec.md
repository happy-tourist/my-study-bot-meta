# subscription/expiry Specification

## Purpose

Фоновая проверка сроков подписки: напоминания до окончания и деактивация после истечения. **Прод-канон:** дневные окна 3/2/1 UTC и ежедневный cron утром Europe/Moscow. Временный минутный харнесс и тест-кнопки выдачи короткой подписки использовались только для живой проверки и **удалены** из runtime; продуктовый UX подписки — отдельный change.

## Traceability

| Scenario ID | Coverage |
|-------------|----------|
| SC-EXP-01 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-02 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-03 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-04 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-05 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-06 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-T01 | covered (manual; historical minute harness — not production) |
| SC-EXP-T02 | covered (manual; historical minute harness — not production) |
| SC-EXP-T03 | covered (manual; historical minute harness — not production) |
| SC-EXP-T04 | covered (code restore + pytest production defaults) |

Связь: меню тем — `start/register` (stub «Подписка» без grant-кнопок). Gate — будущий `subscription/gate`.

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

The system SHALL use day-based reminder windows (3/2/1 UTC days) and a once-daily check in the morning Europe/Moscow (`hour=10`, `minute=0`). The system MUST NOT use every-minute cron (`minute="*"`) or minute-scale reminder windows as the permanent production schedule, and MUST NOT ship temporary one-/five-minute grant buttons in the subscription section.

#### Scenario [SC-EXP-T04]: Production day mode and daily cron

- **GIVEN** the subscription expiry scheduler is configured for production
- **WHEN** the bot process starts
- **THEN** reminder lead intervals are measured in UTC days (3, 2, and 1)
- **AND** the expiry check is scheduled once daily at 10:00 Europe/Moscow
- **AND** every-minute cron is not the production schedule
- **AND** temporary one-/five-minute grant buttons are not present in the subscription section

## Historical note (not production MUST)

Minute-scale reminder windows, `minute="*"` cron, and temporary one-/five-minute grant buttons in the subscription section were used only during live verification. They are **not** permanent production requirements and MUST NOT remain in the shipping bot UI; production behavior is defined by the day-window and daily-cron requirements above. After verification the harness was removed and the subscription section returned to a stub.

#### Scenario [SC-EXP-T01]: One-minute grant yields expiry (historical)

- **GIVEN** the temporary verification harness was enabled
- **AND** an active user was granted a subscription end about one minute ahead
- **WHEN** expiry checks ran about once per minute
- **THEN** after the end time passed, the user was marked inactive and received a Russian expired message

#### Scenario [SC-EXP-T02]: Five-minute grant path observed (historical)

- **GIVEN** the temporary verification harness was enabled
- **AND** an active user was granted a subscription end about five minutes ahead
- **WHEN** expiry checks ran about once per minute
- **THEN** the harness path was used to observe soon-expiry reminder and/or expiry messaging in Telegram

#### Scenario [SC-EXP-T03]: Five-minute grant yields soon-expiry reminder then expiry (historical)

- **GIVEN** the temporary verification harness was enabled
- **AND** an active user had a subscription end about five minutes after grant time
- **WHEN** expiry checks ran about once per minute
- **THEN** the user received a Russian soon-to-expire reminder when the end fell in the one-minute-ahead window
- **AND** after the end time passed, the user was marked inactive and received a Russian expired message
