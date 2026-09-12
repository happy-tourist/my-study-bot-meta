## Purpose

Фоновая проверка сроков подписки: напоминания пользователю за несколько дней до окончания и деактивация записи после истечения срока (без gate платных функций и без оплаты).

## Traceability

| Scenario ID | Coverage |
|-------------|----------|
| SC-EXP-01 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-02 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-03 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-04 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-05 | covered (`tests/test_subscription_expiry.py`) |
| SC-EXP-06 | covered (`tests/test_subscription_expiry.py`) |

Связь: регистрация и меню — `start/register` (без изменений). Gate платных команд — будущий `subscription/gate` (вне этого change).

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
