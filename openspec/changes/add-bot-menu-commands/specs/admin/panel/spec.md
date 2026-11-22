# admin/panel (delta)

Связь: learner-команды Menu Button — `start/register`. Вход `/admin` уже описан в main `admin/panel`.

## Traceability

| Scenario ID | Coverage |
|-------------|----------|
| SC-ADM-CMD-01 | covered — `tests/test_bot_commands.py::test_admin_bot_commands_sc_adm_cmd_01` |
| SC-ADM-CMD-02 | covered — `tests/test_bot_commands.py::test_learner_omits_admin_sc_adm_cmd_02` |
| SC-ADM-CMD-03 | covered — `tests/test_handlers_admin.py::test_promote_demote_syncs_menu_commands_sc_adm_cmd_03` |

## ADDED Requirements

### Requirement: Show /admin only in an admin's Menu Button command list

The system SHALL include the `admin` command in the Telegram Menu Button command list only for users who are admins (bootstrap admin id or admin flag). Non-admin users MUST NOT see `admin` in their Menu Button command list. The Russian description for `admin` SHOULD match the Admin control label used on the main inline menu (emoji-prefixed). Refusing `/admin` for a non-admin (if typed manually) remains as in the existing restrict-admin-panel requirement.

#### Scenario [SC-ADM-CMD-01]: Admin Menu Button lists admin

- **GIVEN** the sender is an admin
- **AND** the bot has applied the admin command list for that private chat
- **WHEN** the admin opens the Telegram Menu Button command list
- **THEN** the list includes `admin` in addition to the learner commands
- **AND** the `admin` description uses the same emoji-prefixed Russian Admin label as the inline Admin control

#### Scenario [SC-ADM-CMD-02]: Non-admin Menu Button omits admin

- **GIVEN** the sender is not an admin
- **WHEN** the user opens the Telegram Menu Button command list
- **THEN** `admin` is not listed among the commands for that user

### Requirement: Refresh Menu Button commands after promote or demote

When an admin successfully promotes another registered user to admin, the system SHALL update that target user's private-chat Menu Button command list so it includes `admin` (admin list applied for that chat). When an admin successfully demotes another user from admin, the system SHALL update that target user's Menu Button command list by **setting** the learner command list for that private chat (not merely clearing the chat-scoped override without replacing it), so `admin` is removed while learner commands remain. Ban and unban MUST NOT be required to change the Menu Button command list in this capability. The promote/demote Russian confirmation and callback acknowledgement behavior from the existing promote/demote requirement MUST still hold.

#### Scenario [SC-ADM-CMD-03]: Promote then demote updates the target Menu Button

- **GIVEN** an admin views a registered non-admin user's card
- **WHEN** the admin promotes that user
- **THEN** the target user's Menu Button command list includes `admin`
- **AND** when a later successful demote of that same user completes
- **THEN** the target user's Menu Button command list no longer includes `admin`
- **AND** learner commands remain available to the target
