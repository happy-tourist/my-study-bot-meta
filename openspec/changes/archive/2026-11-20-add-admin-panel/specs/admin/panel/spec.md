# admin/panel Specification

## Purpose

Операторская панель в Telegram-боте: статистика, поиск и списки пользователей, назначение админов, бан/разбан и ручное продление или снятие подписки с защитами ролей.

## Traceability

| Scenario ID | Coverage |
|-------------|----------|
| SC-ADM-01 | covered (`tests/test_handlers_admin.py` — `test_admin_opens_panel_sc_adm_01`) |
| SC-ADM-02 | covered (`tests/test_handlers_admin.py` — `test_non_admin_refused_admin_sc_adm_02`) |
| SC-ADM-03 | covered (`tests/test_handlers_admin.py` — `test_admin_stats_sc_adm_03`) |
| SC-ADM-04 | covered (`tests/test_handlers_admin.py` — `test_admin_search_by_id_sc_adm_04`) |
| SC-ADM-05 | covered (`tests/test_handlers_admin.py` — `test_admin_promote_sc_adm_05`) |
| SC-ADM-06 | covered (`tests/test_handlers_admin.py` — `test_admin_demote_sc_adm_06`) |
| SC-ADM-07 | covered (`tests/test_handlers_admin.py` — `test_admin_ban_non_admin_sc_adm_07`) |
| SC-ADM-08 | covered (`tests/test_handlers_admin.py` — `test_admin_cannot_ban_admin_sc_adm_08`) |
| SC-ADM-09 | covered (`tests/test_handlers_admin.py` — `test_admin_unban_sc_adm_09`) |
| SC-ADM-10 | covered (`tests/test_handlers_admin.py` — `test_cannot_promote_banned_sc_adm_10`) |
| SC-ADM-11 | covered (`tests/test_handlers_admin.py` — `test_admin_grant_tariff_sc_adm_11`) |
| SC-ADM-12 | covered (`tests/test_handlers_admin.py` — `test_admin_revoke_sc_adm_12`) |
| SC-ADM-13 | covered (`tests/test_handlers_admin.py` — `test_admin_users_list_pagination_sc_adm_13`) |
| SC-ADM-14 | covered (`tests/test_handlers_admin.py` — `test_cannot_demote_self_or_bootstrap_sc_adm_14`) |

Связь: вход с меню — `start/register`. Отказ забаненным в темах — `subscription/gate`. Длительности тарифов при продлении — как в `subscription/tariffs` (без оплаты).

## ADDED Requirements

### Requirement: Restrict admin panel to admins

The system SHALL treat a Telegram user as an admin when the user's numeric id is listed in the bootstrap admin ids configuration **or** the persisted user record is marked as admin. The system SHALL open the admin panel when an admin sends `/admin` or activates the Admin control from the main menu. The system SHALL refuse `/admin` for a non-admin with a Russian message that access is closed. Admin panel actions (stats, lists, search, card actions) MUST be available only to admins.

#### Scenario [SC-ADM-01]: Admin opens panel via /admin

- **GIVEN** the sender is an admin (bootstrap id or admin flag)
- **WHEN** the user sends `/admin`
- **THEN** the bot shows the Russian admin panel entry screen with navigation to statistics and user management

#### Scenario [SC-ADM-02]: Non-admin is refused /admin

- **GIVEN** the sender is not an admin
- **WHEN** the user sends `/admin`
- **THEN** the bot replies in Russian that access is closed
- **AND** the admin panel controls are not shown

### Requirement: Show operator statistics

The system SHALL present Russian aggregate statistics for registered users, including at least: total users, users with an active subscription (same meaning as subscription gate: active flag and subscription end strictly after now), users marked as having used the trial, and banned users.

#### Scenario [SC-ADM-03]: Admin views statistics

- **GIVEN** an admin is in the admin panel
- **WHEN** the admin opens statistics
- **THEN** the bot shows Russian counts for total users, active subscriptions, trial-used users, and banned users

### Requirement: List and search users with pagination

The system SHALL let an admin browse registered users in paginated lists and search by numeric Telegram id or `@username` among users who already have a persisted record. The system SHALL open a user card from a list row or a successful search. Empty or unknown search MUST yield a Russian not-found outcome without changing another user's data.

#### Scenario [SC-ADM-04]: Admin finds user by id or username

- **GIVEN** an admin is in user management
- **AND** a target user is already registered
- **WHEN** the admin searches by that user's numeric id or `@username`
- **THEN** the bot shows that user's card with identity and subscription/ban/admin status in Russian

#### Scenario [SC-ADM-13]: Admin pages through a user list

- **GIVEN** more registered users exist than fit on one admin list page
- **WHEN** the admin opens a user list and presses next/previous page
- **THEN** the bot shows another page of users
- **AND** the callback is acknowledged

### Requirement: Promote and demote admins from user card

The system SHALL allow an admin to mark another registered non-banned user as admin and to clear the admin mark on another admin, except: the actor MUST NOT demote themselves; the system MUST NOT clear admin for a bootstrap admin id; a banned user MUST NOT be promoted until unbanned. Promoting or demoting MUST confirm in Russian and acknowledge the callback when invoked from inline controls.

#### Scenario [SC-ADM-05]: Admin promotes a registered user

- **GIVEN** an admin views the card of a registered non-admin who is not banned
- **WHEN** the admin chooses to make that user an admin
- **THEN** the target is marked as admin
- **AND** the admin receives a Russian confirmation
- **AND** the callback is acknowledged

#### Scenario [SC-ADM-06]: Admin demotes another non-bootstrap admin

- **GIVEN** an admin views the card of another admin who is not a bootstrap admin and is not the actor
- **WHEN** the admin chooses to remove admin rights
- **THEN** the target is no longer marked as admin
- **AND** the admin receives a Russian confirmation

#### Scenario [SC-ADM-10]: Banned user cannot be promoted

- **GIVEN** an admin views the card of a banned non-admin user
- **WHEN** the admin attempts to make that user an admin
- **THEN** the target remains not an admin
- **AND** the admin receives a Russian refusal

#### Scenario [SC-ADM-14]: Actor cannot demote self or bootstrap

- **GIVEN** an admin views their own card or a bootstrap admin's card
- **WHEN** the admin attempts to remove admin rights from that card
- **THEN** the admin mark remains
- **AND** the admin receives a Russian refusal

### Requirement: Ban and unban non-admin users

The system SHALL allow an admin to ban a registered user who is not an admin, and to unban a banned user. The system MUST refuse ban attempts against an admin (including bootstrap admins) with a Russian refusal. A banned learner MUST receive a Russian access-closed outcome for learner-facing bot use (see also `start/register` and `subscription/gate`). Ban and unban MUST NOT be implemented by solely flipping the subscription `is_active` meaning used for expiry.

#### Scenario [SC-ADM-07]: Admin bans a non-admin user

- **GIVEN** an admin views the card of a registered non-admin who is not banned
- **WHEN** the admin chooses to ban that user
- **THEN** the target is marked banned
- **AND** the admin receives a Russian confirmation
- **AND** the callback is acknowledged

#### Scenario [SC-ADM-08]: Admin cannot ban an admin

- **GIVEN** an admin views the card of a user who is an admin
- **WHEN** the admin attempts to ban that user
- **THEN** the target is not marked banned
- **AND** the admin receives a Russian refusal

#### Scenario [SC-ADM-09]: Admin unbans a user

- **GIVEN** an admin views the card of a banned user
- **WHEN** the admin chooses to unban
- **THEN** the target is no longer marked banned
- **AND** the admin receives a Russian confirmation

### Requirement: Grant or revoke subscription from admin card

The system SHALL let an admin grant a target user the same three tariff durations as the learner catalog (one month / 30 minutes, three months / 90 minutes, forever / 36500 minutes), stacking onto a still-future subscription end the same way as learner self-grant, without charging money. The system SHALL let an admin revoke the target's subscription by clearing the future entitlement and marking the user inactive for subscription purposes. Grant and revoke MUST confirm in Russian.

#### Scenario [SC-ADM-11]: Admin grants a tariff to a user

- **GIVEN** an admin views a registered user's card
- **WHEN** the admin selects the one-month tariff grant for that user
- **THEN** the target is marked active for subscription
- **AND** the target's subscription end is about thirty minutes after the grant base time (or stacked onto a future end)
- **AND** the admin receives a Russian confirmation

#### Scenario [SC-ADM-12]: Admin revokes a user's subscription

- **GIVEN** an admin views a user who has an active or future subscription end
- **WHEN** the admin chooses to revoke the subscription
- **THEN** the target no longer has an active subscription entitlement
- **AND** the admin receives a Russian confirmation
