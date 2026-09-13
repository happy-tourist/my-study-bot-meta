# start/register Delta

Связь: админ-панель — `admin/panel`. Gate тем — `subscription/gate`.

## Traceability

| Scenario ID | Coverage |
|-------------|----------|
| SC-START-05 | covered (`tests/test_handlers_subscription.py` — `test_admin_sees_admin_on_start_sc_start_05`) |
| SC-START-06 | covered (`tests/test_handlers_subscription.py` — `test_banned_start_access_closed_sc_start_06`) |
| SC-MENU-05 | covered (`tests/test_handlers_subscription.py` — `test_admin_back_menu_shows_admin_sc_menu_05`) |

## ADDED Requirements

### Requirement: Show Admin control on main menu for admins

The system SHALL include a Russian Admin control on the main section keyboard when the current user is an admin (bootstrap id or admin flag). Non-admin users MUST NOT see that control. Activating the Admin control SHALL open the admin panel entry as in `admin/panel`.

#### Scenario [SC-START-05]: Admin sees Admin on /start menu

- **GIVEN** the Telegram user is an admin
- **WHEN** the user sends `/start` and is not banned
- **THEN** the attached main section keyboard includes Cars, Houses, Subscription, and Admin

#### Scenario [SC-MENU-05]: Admin returns to menu with Admin control

- **GIVEN** an admin is viewing a section screen with Back-to-menu
- **AND** the admin is not banned
- **WHEN** the admin presses Back to menu
- **THEN** the main section keyboard is shown again including the Admin control

### Requirement: Refuse banned users on /start with access closed

The system SHALL, when a banned registered user sends `/start`, reply in Russian that access is closed and MUST NOT present the normal section-choice menu or trial greeting flow as an allowed learner session. The system SHOULD still keep the existing user row (no duplicate). A newly registered user is not banned by default.

#### Scenario [SC-START-06]: Banned user gets access closed on /start

- **GIVEN** the Telegram user is already registered and marked banned
- **WHEN** the user sends `/start`
- **THEN** the bot replies in Russian that access is closed
- **AND** the normal section-choice keyboard is not shown as an allowed learner menu

## MODIFIED Requirements

### Requirement: Register and greet on /start with topic menu

The system SHALL upsert the Telegram user on `/start` and reply in Russian with a greeting plus an invitation to choose a section, attaching an inline keyboard with section buttons (Cars, Houses, Subscription) and, when the user is an admin and not banned, an Admin control. On first registration the system SHALL activate a one-time free trial: mark the trial as used, set an active subscription end three minutes ahead of the registration time, and include a Russian notice that a three-day trial is active (with the three-minute test duration shown in parentheses). A returning registered user MUST NOT receive another trial. If the user is banned, the system SHALL follow the banned `/start` refusal requirement instead of the normal greeting and menu.

#### Scenario [SC-START-01]: First visit registers and shows menu

- **GIVEN** the Telegram user has never been registered in the bot
- **WHEN** the user sends `/start`
- **THEN** the system persists the user with trial marked used
- **AND** the user's subscription end is about three minutes after registration
- **AND** the user is treated as active for subscription purposes
- **AND** replies with a first-visit greeting in Russian that mentions the trial (three days, with minutes in parentheses)
- **AND** invites the user to choose a section
- **AND** attaches an inline keyboard with Cars, Houses, and Subscription actions
- **AND** does not show the Admin control unless the new user is a bootstrap admin

#### Scenario [SC-START-02]: Returning user sees menu

- **GIVEN** the Telegram user is already registered and not banned
- **WHEN** the user sends `/start`
- **THEN** the system does not create a duplicate user
- **AND** does not grant a new trial or extend subscription solely because of `/start`
- **AND** replies with a returning-user greeting in Russian
- **AND** invites the user to choose a section
- **AND** attaches the main section keyboard (with Admin only if the user is an admin)

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
