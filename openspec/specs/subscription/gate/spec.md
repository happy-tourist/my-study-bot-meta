# subscription/gate Specification

## Purpose

Проверка активной подписки перед доступом к тематическим разделам бота: отказ по-русски без активной подписки; раздел выбора тарифов остаётся доступным всегда; забаненные получают отказ «доступ закрыт».

## Traceability

| Scenario ID | Coverage |
|-------------|----------|
| SC-GATE-01 | covered (`tests/test_handlers_subscription.py`) |
| SC-GATE-02 | covered (`tests/test_handlers_subscription.py`) |
| SC-GATE-03 | covered (`tests/test_handlers_subscription.py`) |
| SC-GATE-04 | covered (`tests/test_handlers_subscription.py` — см. SC-MENU-03) |
| SC-GATE-05 | covered (`tests/test_handlers_subscription.py` — `test_banned_refused_cars_sc_gate_05`) |

Связь: меню тем — `start/register` (SC-MENU-01/02). Каталог тарифов — `subscription/tariffs`. Деактивация по сроку — `subscription/expiry`. Бан и админка — `admin/panel`.

## Requirements

### Requirement: Allow topic sections only with active subscription

The system SHALL treat a user as having an active subscription only when the user row exists, `is_active` is true, and `subscription_end` is not null and strictly after the check time (same clock basis as stored ends). The system SHALL open the Cars or Houses section stub only when the user has an active subscription.

#### Scenario [SC-GATE-01]: Active subscriber opens Cars

- **GIVEN** a registered user with an active subscription
- **WHEN** the user presses the Cars section action
- **THEN** the bot message is updated to the Russian Cars stub
- **AND** a Back-to-menu control is shown
- **AND** the callback is acknowledged

#### Scenario [SC-GATE-02]: Active subscriber opens Houses

- **GIVEN** a registered user with an active subscription
- **WHEN** the user presses the Houses section action
- **THEN** the bot message is updated to the Russian Houses stub
- **AND** a Back-to-menu control is shown
- **AND** the callback is acknowledged

### Requirement: Refuse topic sections without active subscription

The system SHALL refuse Cars and Houses when the user has no active subscription (missing user, inactive flag, missing end, or end not after now). The system SHALL respond in Russian that access requires a subscription, MUST NOT show the topic stub content, SHALL acknowledge the callback, and SHOULD point the user toward the Subscription section. The Subscription tariffs screen MUST remain reachable without an active subscription.

#### Scenario [SC-GATE-03]: Expired or inactive user is refused Cars

- **GIVEN** a registered user without an active subscription
- **WHEN** the user presses the Cars section action
- **THEN** the user receives a Russian refusal that a subscription is required
- **AND** the Cars stub content is not shown as an allowed section screen
- **AND** the callback is acknowledged

#### Scenario [SC-GATE-04]: Subscription section stays open without active subscription

- **GIVEN** a registered user without an active subscription
- **WHEN** the user presses the Subscription section action
- **THEN** the tariffs screen is shown
- **AND** the system does not refuse access based on subscription status

### Requirement: Refuse topic sections for banned users

The system SHALL refuse Cars and Houses when the user is marked banned, responding in Russian that access is closed. The system MUST NOT show the topic stub as allowed content, MUST acknowledge the callback, and MUST NOT present the ordinary «subscription required» refusal as the primary outcome for a ban. A banned user MUST also be refused the Subscription tariffs screen with the same access-closed outcome (ban overrides the usual always-open Subscription rule).

#### Scenario [SC-GATE-05]: Banned user is refused Cars with access closed

- **GIVEN** a registered user marked banned
- **WHEN** the user presses the Cars section action
- **THEN** the user receives a Russian message that access is closed
- **AND** the Cars stub content is not shown as an allowed section screen
- **AND** the callback is acknowledged
