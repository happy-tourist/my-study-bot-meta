# subscription/expiry Specification

## Purpose

Фоновая проверка сроков подписки: напоминания до окончания и деактивация после истечения. **Текущая фаза (до ЮKassa):** минутные окна 3/2/1 UTC и cron каждую минуту. Прод-канон (дневные окна + daily 10:00 Europe/Moscow) вернётся отдельным change вместе с оплатой.

## Traceability

| Scenario ID | Coverage |
|-------------|----------|
| SC-EXP-01 | covered (`tests/test_subscription_expiry.py` — minute windows) |
| SC-EXP-02 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-03 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-04 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-05 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-06 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-T01 | covered (manual; historical minute harness — not current product path) |
| SC-EXP-T02 | covered (manual; historical minute harness — not current product path) |
| SC-EXP-T03 | covered (manual; historical minute harness — not current product path) |
| SC-EXP-T04 | covered (`tests/test_subscription_expiry.py` — minute mode + minutely cron) |

Связь: триал и тарифы задают `subscription_end` — `start/register`, `subscription/tariffs`. Gate — `subscription/gate`.

## Requirements

### Requirement: Remind active subscribers before expiry

The system SHALL, on each expiry check, send a Russian reminder to every active user whose subscription end falls in the half-open minute window that is exactly 3, 2, or 1 minute(s) after the check time, stating how many minutes remain.

#### Scenario [SC-EXP-01]: Reminder three days before end

- **GIVEN** an active user has a non-null subscription end falling in the UTC minute window that is 3 minutes after the check time
- **WHEN** the expiry check runs
- **THEN** the user receives a Russian message that the subscription expires in 3 minutes

#### Scenario [SC-EXP-02]: Reminder two days before end

- **GIVEN** an active user has a non-null subscription end falling in the UTC minute window that is 2 minutes after the check time
- **WHEN** the expiry check runs
- **THEN** the user receives a Russian message that the subscription expires in 2 minutes

#### Scenario [SC-EXP-03]: Reminder one day before end

- **GIVEN** an active user has a non-null subscription end falling in the UTC minute window that is 1 minute after the check time
- **WHEN** the expiry check runs
- **THEN** the user receives a Russian message that the subscription expires in 1 minute

#### Scenario [SC-EXP-04]: No reminder without subscription end

- **GIVEN** an active user has no subscription end date
- **WHEN** the expiry check runs
- **THEN** the system does not send an expiry reminder to that user
- **AND** does not change that user's active flag

### Requirement: Deactivate expired subscribers and notify

The system SHALL, on each expiry check, mark as inactive every still-active user whose subscription end is strictly before the check time, persist that change, and send a Russian message that the subscription has expired.

#### Scenario [SC-EXP-05]: Expired subscription is deactivated

- **GIVEN** an active user has a subscription end strictly before the check time
- **WHEN** the expiry check runs
- **THEN** the user is marked inactive in persistent storage
- **AND** the user receives a Russian message that the subscription has expired

#### Scenario [SC-EXP-06]: Failed delivery to one user does not abort the check

- **GIVEN** more than one user matches reminder or expiry processing in the same check
- **AND** delivering a Telegram message to one of them fails
- **WHEN** the expiry check runs
- **THEN** the system continues processing the remaining matching users
- **AND** expiry deactivations already determined for other users are still persisted

### Requirement: Production schedule and day windows

The system SHALL use minute-based reminder windows (3/2/1 UTC minutes) and an every-minute expiry check (`minute="*"`) for this temporary test phase until a later change restores day windows and the once-daily Europe/Moscow morning schedule together with real payments. The system MUST NOT keep the former temporary one-/five-minute-only grant buttons; subscription duration grants come from the tariffs catalog instead.

#### Scenario [SC-EXP-T04]: Production day mode and daily cron

- **GIVEN** the subscription expiry scheduler is configured for the current test phase
- **WHEN** the bot process starts
- **THEN** reminder lead intervals are measured in UTC minutes (3, 2, and 1)
- **AND** the expiry check is scheduled every minute
- **AND** the once-daily 10:00 Europe/Moscow schedule is not the active schedule for this phase
- **AND** temporary one-/five-minute-only grant buttons are not present in the subscription section

## Historical note (not current MUST)

Earlier live checks used dedicated one-/five-minute grant buttons in the subscription section. Those buttons are **not** part of the current product path; duration grants now come from the tariffs catalog (`subscription/tariffs`). Scenarios SC-EXP-T01…T03 document that historical harness only.

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
