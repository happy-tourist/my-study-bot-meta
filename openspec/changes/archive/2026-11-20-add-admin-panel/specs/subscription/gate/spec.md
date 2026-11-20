# subscription/gate Delta

Связь: бан и админка — `admin/panel`. Меню — `start/register`.

## Traceability

| Scenario ID | Coverage |
|-------------|----------|
| SC-GATE-05 | covered (`tests/test_handlers_subscription.py` — `test_banned_refused_cars_sc_gate_05`) |

## ADDED Requirements

### Requirement: Refuse topic sections for banned users

The system SHALL refuse Cars and Houses when the user is marked banned, responding in Russian that access is closed. The system MUST NOT show the topic stub as allowed content, MUST acknowledge the callback, and MUST NOT present the ordinary «subscription required» refusal as the primary outcome for a ban. A banned user MUST also be refused the Subscription tariffs screen with the same access-closed outcome (ban overrides the usual always-open Subscription rule).

#### Scenario [SC-GATE-05]: Banned user is refused Cars with access closed

- **GIVEN** a registered user marked banned
- **WHEN** the user presses the Cars section action
- **THEN** the user receives a Russian message that access is closed
- **AND** the Cars stub content is not shown as an allowed section screen
- **AND** the callback is acknowledged
