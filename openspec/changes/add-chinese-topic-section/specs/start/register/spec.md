# start/register Delta

Связь: gate темы «Курсы китайского» — `subscription/gate` (этот change). Базовое меню без китайского — main `openspec/specs/start/register/spec.md`.

## Traceability

| Scenario ID | Coverage |
|-------------|----------|
| SC-START-01 | covered (`tests/test_handlers_subscription.py` — `test_start_first_visit_grants_trial_sc_start_01` via `_assert_main_topic_labels` + Chinese) |
| SC-START-02 | covered (`tests/test_handlers_subscription.py` — `test_start_returning_no_new_trial_sc_start_02` via `_assert_main_topic_labels` + Chinese) |
| SC-START-05 | covered (`tests/test_handlers_subscription.py` — `test_admin_sees_admin_on_start_sc_start_05` via `_assert_main_topic_labels` + Chinese) |
| SC-MENU-04 | covered (`tests/test_handlers_subscription.py` — `test_back_to_main_menu_sc_menu_04` via `_assert_main_topic_labels` + Chinese) |
| SC-MENU-07 | covered (`tests/test_handlers_subscription.py` — `test_active_subscriber_opens_chinese_sc_gate_09`) |
| SC-CMD-01 | covered (`tests/test_bot_commands.py` — `test_learner_bot_commands_sc_cmd_01`; `LEARNER_COMMANDS` / `SECTION_DESCRIPTIONS` + `chinese`) |
| SC-CMD-09 | covered-by-reuse (`tests/test_bot_commands.py` — `test_start_syncs_learner_commands_sc_cmd_09` / admin / banned via updated `LEARNER_COMMANDS`) |
| SC-CMD-10 | covered (`tests/test_bot_commands.py` — `test_cmd_chinese_active_subscriber_sc_cmd_10`) |
| SC-HELP-01 | covered-by-reuse (`tests/test_bot_commands.py` — `test_help_learner_omits_admin_sc_help_01` via updated `LEARNER_COMMANDS`) |

## MODIFIED Requirements

### Requirement: Register and greet on /start with topic menu

The system SHALL upsert the Telegram user on `/start` and reply in Russian with a greeting plus an invitation to choose a section, attaching an inline keyboard with section buttons (Cars, Houses, Computers, Chinese courses, Subscription) and, when the user is an admin and not banned, an Admin control. On first registration the system SHALL activate a one-time free trial: mark the trial as used, set an active subscription end three minutes ahead of the registration time, and include a Russian notice that a three-day trial is active (with the three-minute test duration shown in parentheses). A returning registered user MUST NOT receive another trial. If the user is banned, the system SHALL follow the banned `/start` refusal requirement instead of the normal greeting and menu.

#### Scenario [SC-START-01]: First visit registers and shows menu

- **GIVEN** the Telegram user has never been registered in the bot
- **WHEN** the user sends `/start`
- **THEN** the system persists the user with trial marked used
- **AND** the user's subscription end is about three minutes after registration
- **AND** the user is treated as active for subscription purposes
- **AND** replies with a first-visit greeting in Russian that mentions the trial (three days, with minutes in parentheses)
- **AND** invites the user to choose a section
- **AND** attaches an inline keyboard with Cars, Houses, Computers, Chinese courses, and Subscription actions
- **AND** does not show the Admin control unless the new user is a bootstrap admin

#### Scenario [SC-START-02]: Returning user sees menu

- **GIVEN** the Telegram user is already registered and not banned
- **WHEN** the user sends `/start`
- **THEN** the system does not create a duplicate user
- **AND** does not grant a new trial or extend subscription solely because of `/start`
- **AND** replies with a returning-user greeting in Russian
- **AND** invites the user to choose a section
- **AND** attaches the main section keyboard (Cars, Houses, Computers, Chinese courses, Subscription; Admin only if the user is an admin)

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
- **THEN** the attached main section keyboard includes Cars, Houses, Computers, Chinese courses, Subscription, and Admin

#### Scenario [SC-MENU-05]: Admin returns to menu with Admin control

- **GIVEN** an admin is viewing a section screen with Back-to-menu
- **AND** the admin is not banned
- **WHEN** the admin presses Back to menu
- **THEN** the main section keyboard is shown again including the Admin control

### Requirement: Navigate section stubs via inline callbacks

The system SHALL open stub screens for Cars, Houses, Computers, and Chinese courses when the corresponding inline action is pressed **and** the user has an active subscription per `subscription/gate`. Without an active subscription the system MUST refuse those sections as defined by `subscription/gate` (not open the stub as allowed content). When the Subscription section action is pressed, the system SHALL open the subscription tariffs screen (catalog of plans) instead of a placeholder stub, without refusing access based on subscription status. The system SHALL acknowledge the callback and allow returning to the main section menu by editing the same bot message.

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

#### Scenario [SC-MENU-07]: Open Chinese courses stub

- **GIVEN** the user sees the main section menu message
- **AND** the user has an active subscription
- **WHEN** the user presses the Chinese courses section action
- **THEN** the bot message is updated to the Russian Chinese courses stub text exactly `Раздел «Курсы китайского»\nЗдесь будет контент.`
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

- **GIVEN** the user is viewing a Cars, Houses, Computers, or Chinese courses stub or the Subscription tariffs screen with a Back-to-menu control
- **WHEN** the user presses Back to menu
- **THEN** the bot message is updated back to the section-choice prompt
- **AND** the main inline section keyboard is shown again
- **AND** the callback is acknowledged

### Requirement: Publish learner bot commands for the Menu Button

The system SHALL register a default Telegram bot command list for private chats so the client Menu Button near the message field opens commands. The learner list MUST include at least: `start`, `cars`, `houses`, `computers`, `chinese`, `subscription`, and `help`. Command **names** MUST be Latin slugs; command **descriptions** MUST be Russian. Section command descriptions SHOULD reuse the same emoji-prefixed section labels as the main inline menu (Cars, Houses, Computers, Chinese courses, Subscription). Descriptions for `start` and `help` MUST be short Russian phrases without section emojis (canon in design: launch/main menu and help/command list). The system MUST NOT put `admin` in the default learner command list. The inline section keyboard after `/start` remains the primary section navigation and MUST continue to work unchanged by this requirement.

#### Scenario [SC-CMD-01]: Learner Menu Button lists section commands without admin

- **GIVEN** the bot has applied its default learner command list
- **AND** the user is not an admin (or has no admin-only command override)
- **WHEN** the user opens the Telegram Menu Button command list in the private chat
- **THEN** the list includes `start`, `cars`, `houses`, `computers`, `chinese`, `subscription`, and `help`
- **AND** section command descriptions use the same emoji-prefixed Russian labels as the inline section controls
- **AND** `start` and `help` have short Russian descriptions without section emojis
- **AND** `admin` is not listed for that learner

### Requirement: Open sections via slash commands with the same access rules as inline

The system SHALL accept slash commands `cars`, `houses`, `computers`, `chinese`, and `subscription` as alternate entry points to the section outcomes of the corresponding inline actions in this capability and `subscription/gate` **for registered users**. Slash entry MUST check registration first: an unregistered sender (never registered in the bot) MUST receive a Russian «Сначала нажми /start» reply and MUST NOT receive stub or tariffs content (SC-CMD-06). This unregistered check is required for slash even if today's inline topic or Subscription actions do not short-circuit the same way for a missing registration; this change does not require fixing those inline paths for an unregistered sender. After registration is confirmed, topic commands (`cars`, `houses`, `computers`, `chinese`) MUST apply the same ban and active-subscription checks as the inline topic actions: banned users get the Russian access-closed refusal; users without an active subscription get the Russian subscription-required refusal without topic stub content. The `subscription` command MUST apply the same ban check as the inline Subscription action (banned → Russian access-closed refusal) and, when the user is registered and not banned, MUST open the tariffs screen without refusing based on subscription status. Because a slash command starts a new message (unlike inline edit of an existing menu message), the system SHALL reply with a new Russian message carrying the section content and the same Back-to-menu control style used for section screens.

#### Scenario [SC-CMD-02]: Active subscriber opens Cars via /cars

- **GIVEN** a registered user with an active subscription
- **AND** the user is not banned
- **WHEN** the user sends `/cars`
- **THEN** the bot replies with a new Russian Cars stub message
- **AND** a Back-to-menu control is shown
- **AND** the reply is not an inline edit of a previous menu message

#### Scenario [SC-CMD-03]: User without subscription is refused /houses

- **GIVEN** a registered user without an active subscription
- **AND** the user is not banned
- **WHEN** the user sends `/houses`
- **THEN** the user receives a Russian refusal that a subscription is required
- **AND** the Houses stub content is not shown as an allowed section screen

#### Scenario [SC-CMD-04]: Banned user is refused /computers

- **GIVEN** a registered user marked banned
- **WHEN** the user sends `/computers`
- **THEN** the bot replies in Russian that access is closed
- **AND** the Computers stub is not shown as an allowed section screen

#### Scenario [SC-CMD-05]: /subscription opens tariffs without gate

- **GIVEN** a registered user (with or without an active subscription)
- **AND** the user is not banned
- **WHEN** the user sends `/subscription`
- **THEN** the bot replies with a new Russian subscription tariffs screen listing the available plans
- **AND** the system does not refuse access based on subscription status
- **AND** a Back-to-menu control is shown

#### Scenario [SC-CMD-06]: Unregistered user is pointed to /start on a section command

- **GIVEN** the Telegram user has never been registered in the bot
- **WHEN** the user sends `/cars` (or another section command)
- **THEN** the bot replies in Russian that the user should press `/start` first
- **AND** section stub or tariffs content is not shown as an allowed session

#### Scenario [SC-CMD-07]: Active subscriber opens Computers via /computers

- **GIVEN** a registered user with an active subscription
- **AND** the user is not banned
- **WHEN** the user sends `/computers`
- **THEN** the bot replies with a new Russian Computers stub whose text matches the inline Computers stub content
- **AND** a Back-to-menu control is shown

#### Scenario [SC-CMD-08]: Banned user is refused /subscription

- **GIVEN** a registered user marked banned
- **WHEN** the user sends `/subscription`
- **THEN** the bot replies in Russian that access is closed
- **AND** the tariffs screen is not shown as an allowed section screen

#### Scenario [SC-CMD-10]: Active subscriber opens Chinese courses via /chinese

- **GIVEN** a registered user with an active subscription
- **AND** the user is not banned
- **WHEN** the user sends `/chinese`
- **THEN** the bot replies with a new Russian Chinese courses stub text exactly `Раздел «Курсы китайского»\nЗдесь будет контент.` (same as the inline Chinese courses stub)
- **AND** a Back-to-menu control is shown
