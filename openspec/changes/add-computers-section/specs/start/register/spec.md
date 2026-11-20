# start/register Delta

Связь: gate тем — `subscription/gate` (этот change). Базовое поведение `/start` и меню без «Компьютеров» — main `openspec/specs/start/register/spec.md`.

## Traceability

| Scenario ID | Coverage |
|-------------|----------|
| SC-START-01 | covered (`tests/test_handlers_subscription.py` — `test_start_first_visit_grants_trial_sc_start_01`) |
| SC-START-05 | covered (`tests/test_handlers_subscription.py` — `test_admin_sees_admin_on_start_sc_start_05`) |
| SC-MENU-04 | covered (`tests/test_handlers_subscription.py` — `test_back_to_main_menu_sc_menu_04`) |
| SC-MENU-06 | covered (`tests/test_handlers_subscription.py` — `test_active_subscriber_opens_computers_sc_gate_06`) |

## MODIFIED Requirements

### Requirement: Register and greet on /start with topic menu

The system SHALL upsert the Telegram user on `/start` and reply in Russian with a greeting plus an invitation to choose a section, attaching an inline keyboard with section buttons (Cars, Houses, Computers, Subscription) and, when the user is an admin and not banned, an Admin control. On first registration the system SHALL activate a one-time free trial: mark the trial as used, set an active subscription end three minutes ahead of the registration time, and include a Russian notice that a three-day trial is active (with the three-minute test duration shown in parentheses). A returning registered user MUST NOT receive another trial. If the user is banned, the system SHALL follow the banned `/start` refusal requirement instead of the normal greeting and menu.

#### Scenario [SC-START-01]: First visit registers and shows menu

- **GIVEN** the Telegram user has never been registered in the bot
- **WHEN** the user sends `/start`
- **THEN** the system persists the user with trial marked used
- **AND** the user's subscription end is about three minutes after registration
- **AND** the user is treated as active for subscription purposes
- **AND** replies with a first-visit greeting in Russian that mentions the trial (three days, with minutes in parentheses)
- **AND** invites the user to choose a section
- **AND** attaches an inline keyboard with Cars, Houses, Computers, and Subscription actions
- **AND** does not show the Admin control unless the new user is a bootstrap admin

#### Scenario [SC-START-02]: Returning user sees menu

- **GIVEN** the Telegram user is already registered and not banned
- **WHEN** the user sends `/start`
- **THEN** the system does not create a duplicate user
- **AND** does not grant a new trial or extend subscription solely because of `/start`
- **AND** replies with a returning-user greeting in Russian
- **AND** invites the user to choose a section
- **AND** attaches the main section keyboard (Cars, Houses, Computers, Subscription; Admin only if the user is an admin)

#### Scenario [SC-START-03]: Trial is one-time only after expiry

- **GIVEN** the Telegram user already used the trial and the trial subscription has expired
- **AND** the user is not banned
- **WHEN** the user sends `/start` again
- **THEN** the system does not reactivate a free trial
- **AND** does not create a duplicate user

#### Scenario [SC-START-04]: Unused trial can be claimed from Subscription

- **GIVEN** a registered user whose trial has not been used (`trial_used` is false)
- **AND** the user is not banned
- **WHEN** the user opens the Subscription section
- **THEN** a Russian control to claim the free trial is shown with the tariff plans
- **AND** selecting that control activates the same three-minute trial once
- **AND** a second claim attempt does not extend or re-grant the trial

### Requirement: Show Admin control on main menu for admins

The system SHALL include a Russian Admin control on the main section keyboard when the current user is an admin (bootstrap id or admin flag). Non-admin users MUST NOT see that control. Activating the Admin control SHALL open the admin panel entry as in `admin/panel`.

#### Scenario [SC-START-05]: Admin sees Admin on /start menu

- **GIVEN** the Telegram user is an admin
- **WHEN** the user sends `/start` and is not banned
- **THEN** the attached main section keyboard includes Cars, Houses, Computers, Subscription, and Admin

#### Scenario [SC-MENU-05]: Admin returns to menu with Admin control

- **GIVEN** an admin is viewing a section screen with Back-to-menu
- **AND** the admin is not banned
- **WHEN** the admin presses Back to menu
- **THEN** the main section keyboard is shown again including the Admin control

### Requirement: Navigate section stubs via inline callbacks

The system SHALL open stub screens for Cars, Houses, and Computers when the corresponding inline action is pressed **and** the user has an active subscription per `subscription/gate`. Without an active subscription the system MUST refuse those sections as defined by `subscription/gate` (not open the stub as allowed content). When the Subscription section action is pressed, the system SHALL open the subscription tariffs screen (catalog of plans) instead of a placeholder stub, without refusing access based on subscription status. The system SHALL acknowledge the callback and allow returning to the main section menu by editing the same bot message.

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

#### Scenario [SC-MENU-06]: Open Computers stub

- **GIVEN** the user sees the main section menu message
- **AND** the user has an active subscription
- **WHEN** the user presses the Computers section action
- **THEN** the bot message is updated to the Russian Computers stub text exactly `Раздел «Компьютеры»\nЗдесь будет контент.`
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

- **GIVEN** the user is viewing a Cars, Houses, or Computers stub or the Subscription tariffs screen with a Back-to-menu control
- **WHEN** the user presses Back to menu
- **THEN** the bot message is updated back to the section-choice prompt
- **AND** the main inline section keyboard is shown again
- **AND** the callback is acknowledged
