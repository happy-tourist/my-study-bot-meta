## Purpose

Каталог тарифов подписки в Telegram-боте и мгновенная выдача срока по выбору плана без реального платёжного провайдера (тест до подключения ЮKassa).

## Traceability

| Scenario ID | Coverage |
|-------------|----------|
| SC-TAR-01 | covered (`tests/test_handlers_subscription.py`) |
| SC-TAR-02 | covered (`tests/test_handlers_subscription.py`) |
| SC-TAR-03 | covered (`tests/test_handlers_subscription.py`) |
| SC-TAR-04 | covered (`tests/test_handlers_subscription.py`) |
| SC-TAR-05 | covered (`tests/test_handlers_subscription.py`) |

Связь: вход в раздел — `start/register` (SC-MENU-03). Gate тем — `subscription/gate` (раздел «Подписка» без gate). Expiry — `subscription/expiry`.

## ADDED Requirements

### Requirement: Show tariff catalog in Subscription section

The system SHALL present three subscription tariffs with Russian day-oriented titles and the corresponding test durations in minutes shown in parentheses: one month (30 minutes), three months (90 minutes), and forever (36500 minutes). Optional display prices MAY appear on the controls; the system MUST NOT charge money or call a payment provider in this capability. When the user's trial has not been used, the tariffs screen SHALL also offer a one-time free-trial claim control (see `start/register` SC-START-04); when the trial is already used, that control MUST NOT appear.

#### Scenario [SC-TAR-01]: Tariffs list shows three plans with minute hints

- **GIVEN** the user opens the Subscription section
- **WHEN** the tariffs screen is shown
- **THEN** three plan actions are available for one month, three months, and forever
- **AND** each plan label uses the day-oriented title with the minute duration in parentheses
- **AND** a Back-to-menu control is available

### Requirement: Grant subscription immediately on tariff selection

The system SHALL, when the user selects a tariff, treat the selection as a successful payment: set the user active and set the subscription end to the tariff duration in minutes measured from a base time of now, or from the current subscription end if that end is still in the future (stacking). The system SHALL confirm the grant in Russian and acknowledge the callback. The system MUST NOT require a payment provider.

#### Scenario [SC-TAR-02]: Select one-month tariff grants thirty minutes

- **GIVEN** a registered user with no future subscription end (or end already in the past)
- **WHEN** the user selects the one-month tariff
- **THEN** the user is marked active
- **AND** the subscription end is about thirty minutes after the grant time
- **AND** the user receives a Russian confirmation of the activated plan
- **AND** the callback is acknowledged

#### Scenario [SC-TAR-03]: Select three-month tariff grants ninety minutes

- **GIVEN** a registered user with no future subscription end (or end already in the past)
- **WHEN** the user selects the three-month tariff
- **THEN** the user is marked active
- **AND** the subscription end is about ninety minutes after the grant time
- **AND** the user receives a Russian confirmation of the activated plan

#### Scenario [SC-TAR-04]: Select forever tariff grants 36500 minutes

- **GIVEN** a registered user with no future subscription end (or end already in the past)
- **WHEN** the user selects the forever tariff
- **THEN** the user is marked active
- **AND** the subscription end is about 36500 minutes after the grant time
- **AND** the user receives a Russian confirmation of the activated plan

#### Scenario [SC-TAR-05]: Selecting a tariff stacks onto a future end

- **GIVEN** a registered active user whose subscription end is still in the future
- **WHEN** the user selects a tariff with a known minute duration
- **THEN** the new subscription end is that duration added to the previous future end
- **AND** the user remains active
- **AND** the user receives a Russian confirmation
