# start/register (delta)

Связь: gate тем — `subscription/gate`. Тарифы — `subscription/tariffs`. Админ-команды в Menu Button — `admin/panel`.

## Traceability

| Scenario ID | Coverage |
|-------------|----------|
| SC-CMD-01 | covered — `tests/test_bot_commands.py::test_learner_bot_commands_sc_cmd_01` |
| SC-CMD-02 | covered — `tests/test_bot_commands.py::test_cmd_cars_active_subscriber_sc_cmd_02` |
| SC-CMD-03 | covered — `tests/test_bot_commands.py::test_cmd_houses_without_subscription_sc_cmd_03` |
| SC-CMD-04 | covered — `tests/test_bot_commands.py::test_cmd_computers_banned_sc_cmd_04` |
| SC-CMD-05 | covered — `tests/test_bot_commands.py::test_cmd_subscription_opens_tariffs_sc_cmd_05` |
| SC-CMD-06 | covered — `tests/test_bot_commands.py::test_cmd_cars_unregistered_sc_cmd_06` |
| SC-CMD-07 | covered — `tests/test_bot_commands.py::test_cmd_computers_active_subscriber_sc_cmd_07` |
| SC-CMD-08 | covered — `tests/test_bot_commands.py::test_cmd_subscription_banned_sc_cmd_08` |
| SC-CMD-09 | covered — `tests/test_bot_commands.py::test_start_syncs_learner_commands_sc_cmd_09`, `test_start_syncs_admin_flag_and_bootstrap_sc_cmd_09`, `test_start_syncs_banned_user_sc_cmd_09` |
| SC-HELP-01 | covered — `tests/test_bot_commands.py::test_help_learner_omits_admin_sc_help_01` |
| SC-HELP-02 | covered — `tests/test_bot_commands.py::test_help_admin_includes_admin_sc_help_02` |
| SC-HELP-03 | covered — `tests/test_bot_commands.py::test_help_unregistered_and_banned_sc_help_03` |

## ADDED Requirements

### Requirement: Publish learner bot commands for the Menu Button

The system SHALL register a default Telegram bot command list for private chats so the client Menu Button near the message field opens commands. The learner list MUST include at least: `start`, `cars`, `houses`, `computers`, `subscription`, and `help`. Command **names** MUST be Latin slugs; command **descriptions** MUST be Russian. Section command descriptions SHOULD reuse the same emoji-prefixed section labels as the main inline menu (Cars, Houses, Computers, Subscription). Descriptions for `start` and `help` MUST be short Russian phrases without section emojis (canon in design: launch/main menu and help/command list). The system MUST NOT put `admin` in the default learner command list. The inline section keyboard after `/start` remains the primary section navigation and MUST continue to work unchanged by this requirement.

#### Scenario [SC-CMD-01]: Learner Menu Button lists section commands without admin

- **GIVEN** the bot has applied its default learner command list
- **AND** the user is not an admin (or has no admin-only command override)
- **WHEN** the user opens the Telegram Menu Button command list in the private chat
- **THEN** the list includes `start`, `cars`, `houses`, `computers`, `subscription`, and `help`
- **AND** section command descriptions use the same emoji-prefixed Russian labels as the inline section controls
- **AND** `start` and `help` have short Russian descriptions without section emojis
- **AND** `admin` is not listed for that learner

### Requirement: Open sections via slash commands with the same access rules as inline

The system SHALL accept slash commands `cars`, `houses`, `computers`, and `subscription` as alternate entry points to the section outcomes of the corresponding inline actions in this capability and `subscription/gate` **for registered users**. Slash entry MUST check registration first: an unregistered sender (never registered in the bot) MUST receive a Russian «Сначала нажми /start» reply and MUST NOT receive stub or tariffs content (SC-CMD-06). This unregistered check is required for slash even if today's inline topic or Subscription actions do not short-circuit the same way for a missing registration; this change does not require fixing those inline paths for an unregistered sender. After registration is confirmed, topic commands (`cars`, `houses`, `computers`) MUST apply the same ban and active-subscription checks as the inline topic actions: banned users get the Russian access-closed refusal; users without an active subscription get the Russian subscription-required refusal without topic stub content. The `subscription` command MUST apply the same ban check as the inline Subscription action (banned → Russian access-closed refusal) and, when the user is registered and not banned, MUST open the tariffs screen without refusing based on subscription status. Because a slash command starts a new message (unlike inline edit of an existing menu message), the system SHALL reply with a new Russian message carrying the section content and the same Back-to-menu control style used for section screens.

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

### Requirement: Sync Menu Button commands on /start

On `/start`, after the user is known as registered (newly persisted on first visit, or an existing loaded registration), the system SHALL sync that private chat's Menu Button command list before sending the usual `/start` reply. Sync MUST choose the admin command list when the sender is an admin (bootstrap admin id **or** admin flag on the persisted user), otherwise the learner list — so a bootstrap admin without an admin flag on the record still gets `/admin` in the Menu Button. Sync MUST also run for a banned returning user (the Menu Button list is updated even when the reply is the Russian access-closed refusal without the section menu).

#### Scenario [SC-CMD-09]: /start syncs commands including for banned users

- **GIVEN** a registered user (admin or learner; including when marked banned)
- **WHEN** the user sends `/start`
- **THEN** the system updates that chat's Menu Button command list to the admin list if the user is an admin, otherwise the learner list
- **AND** the update happens before the usual `/start` reply is sent
- **AND** a banned user still receives the Russian access-closed reply without the normal section-choice menu

### Requirement: Answer /help with a short guide and available commands

The system SHALL handle `/help` with a concise Russian explanation of the bot and a list of commands available to that sender. For a non-admin the list MUST include the learner commands (`start`, section commands, `help`) and MUST NOT present `admin` as an available command. For an admin the list MUST also include `admin`. `/help` MUST answer without requiring a prior `/start` registration and MUST still answer when the sender is banned (help is not gated section content). `/help` MUST NOT attach the main inline topic menu and MUST NOT replace the inline topic menu after `/start`.

#### Scenario [SC-HELP-01]: Learner sees help without admin command

- **GIVEN** the sender is not an admin
- **WHEN** the user sends `/help`
- **THEN** the bot replies in Russian with a short guide
- **AND** lists the learner commands including section commands
- **AND** does not present `admin` as an available command for that user

#### Scenario [SC-HELP-02]: Admin sees help including admin command

- **GIVEN** the sender is an admin
- **WHEN** the user sends `/help`
- **THEN** the bot replies in Russian with a short guide
- **AND** the listed commands include `admin` in addition to the learner commands

#### Scenario [SC-HELP-03]: Unregistered or banned sender still gets help

- **GIVEN** the Telegram user is unregistered or is a registered banned user
- **AND** the sender is not an admin
- **WHEN** the user sends `/help`
- **THEN** the bot replies in Russian with a short guide and the learner command list
- **AND** does not require the user to press `/start` first solely to receive help
- **AND** does not present section stub or tariffs content as an allowed session
