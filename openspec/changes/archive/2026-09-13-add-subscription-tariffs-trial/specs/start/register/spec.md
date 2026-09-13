# start/register (delta)

Связь: каталог тарифов — `subscription/tariffs`. Gate тем — `subscription/gate`. Expiry — `subscription/expiry`.

## Traceability

| Scenario ID | Coverage |
|-------------|----------|
| SC-START-01 | covered (`tests/test_handlers_subscription.py`) |
| SC-START-02 | covered (`tests/test_handlers_subscription.py`) |
| SC-START-03 | covered (`tests/test_handlers_subscription.py`) |
| SC-START-04 | covered (`tests/test_handlers_subscription.py` — claim-trial CTA) |
| SC-MENU-01 | covered (`tests/test_handlers_subscription.py` — см. также SC-GATE-01) |
| SC-MENU-02 | covered (`tests/test_handlers_subscription.py` — см. также SC-GATE-02) |
| SC-MENU-03 | covered (`tests/test_handlers_subscription.py`) |
| SC-MENU-04 | covered (`tests/test_handlers_subscription.py`) |

## MODIFIED Requirements

### Requirement: Register and greet on /start with topic menu

The system SHALL upsert the Telegram user on `/start` and reply in Russian with a greeting plus an invitation to choose a section, attaching an inline keyboard with three section buttons (Cars, Houses, Subscription). On first registration the system SHALL activate a one-time free trial: mark the trial as used, set an active subscription end three minutes ahead of the registration time, and include a Russian notice that a three-day trial is active (with the three-minute test duration shown in parentheses). A returning registered user MUST NOT receive another trial.

#### Scenario [SC-START-01]: First visit registers and shows menu

- **GIVEN** the Telegram user has never been registered in the bot
- **WHEN** the user sends `/start`
- **THEN** the system persists the user with trial marked used
- **AND** the user's subscription end is about three minutes after registration
- **AND** the user is treated as active for subscription purposes
- **AND** replies with a first-visit greeting in Russian that mentions the trial (three days, with minutes in parentheses)
- **AND** invites the user to choose a section
- **AND** attaches an inline keyboard with Cars, Houses, and Subscription actions

#### Scenario [SC-START-02]: Returning user sees menu

- **GIVEN** the Telegram user is already registered
- **WHEN** the user sends `/start`
- **THEN** the system does not create a duplicate user
- **AND** does not grant a new trial or extend subscription solely because of `/start`
- **AND** replies with a returning-user greeting in Russian
- **AND** invites the user to choose a section
- **AND** attaches the same inline section keyboard

#### Scenario [SC-START-03]: Trial is one-time only after expiry

- **GIVEN** the Telegram user already used the trial and the trial subscription has expired
- **WHEN** the user sends `/start` again
- **THEN** the system does not reactivate a free trial
- **AND** does not create a duplicate user

#### Scenario [SC-START-04]: Unused trial can be claimed from Subscription

- **GIVEN** a registered user whose trial has not been used (`trial_used` is false)
- **WHEN** the user opens the Subscription section
- **THEN** a Russian control to claim the free trial is shown with the tariff plans
- **AND** selecting that control activates the same three-minute trial once
- **AND** a second claim attempt does not extend or re-grant the trial

### Requirement: Navigate section stubs via inline callbacks

The system SHALL open stub screens for Cars and Houses when the corresponding inline action is pressed **and** the user has an active subscription per `subscription/gate`. Without an active subscription the system MUST refuse those sections as defined by `subscription/gate` (not open the stub as allowed content). When the Subscription section action is pressed, the system SHALL open the subscription tariffs screen (catalog of plans) instead of a placeholder stub, without refusing access based on subscription status. The system SHALL acknowledge the callback and allow returning to the main section menu by editing the same bot message.

#### Scenario [SC-MENU-01]: Open Cars stub

- **GIVEN** the user sees the main section menu message
- **AND** the user has an active subscription
- **WHEN** the user presses the Cars section action
- **THEN** the bot message is updated to a Russian Cars stub (placeholder content)
- **AND** a Back-to-menu control is shown
- **AND** the callback is acknowledged

#### Scenario [SC-MENU-02]: Open Houses stub

- **GIVEN** the user sees the main section menu message
- **AND** the user has an active subscription
- **WHEN** the user presses the Houses section action
- **THEN** the bot message is updated to a Russian Houses stub (placeholder content)
- **AND** a Back-to-menu control is shown
- **AND** the callback is acknowledged

#### Scenario [SC-MENU-03]: Open Subscription tariffs without gate

- **GIVEN** the user sees the main section menu message
- **WHEN** the user presses the Subscription section action
- **THEN** the bot message is updated to a Russian subscription tariffs screen listing the available plans (replaces the former placeholder stub)
- **AND** the system does not refuse access based on subscription status
- **AND** a Back-to-menu control is shown
- **AND** the callback is acknowledged

#### Scenario [SC-MENU-04]: Return to main menu

- **GIVEN** the user is viewing a Cars or Houses stub or the Subscription tariffs screen with a Back-to-menu control
- **WHEN** the user presses Back to menu
- **THEN** the bot message is updated back to the section-choice prompt
- **AND** the main inline section keyboard is shown again
- **AND** the callback is acknowledged
