## Purpose

Регистрация пользователя Telegram на `/start` и навигация по разделам через инлайн-меню (темы и подписка как заглушки до отдельных product-change).

## Traceability

| Scenario ID | Coverage |
|-------------|----------|
| SC-START-01 | pending (нет pytest suite) |
| SC-START-02 | pending |
| SC-MENU-01 | pending |
| SC-MENU-02 | pending |
| SC-MENU-03 | pending |
| SC-MENU-04 | pending |

## ADDED Requirements

### Requirement: Register and greet on /start with topic menu

The system SHALL upsert the Telegram user on `/start` and reply in Russian with a greeting plus an invitation to choose a section, attaching an inline keyboard with three section buttons (Cars, Houses, Subscription).

#### Scenario [SC-START-01]: First visit registers and shows menu

- **GIVEN** the Telegram user has never been registered in the bot
- **WHEN** the user sends `/start`
- **THEN** the system persists the user
- **AND** replies with a first-visit greeting in Russian
- **AND** invites the user to choose a section
- **AND** attaches an inline keyboard with Cars, Houses, and Subscription actions

#### Scenario [SC-START-02]: Returning user sees menu

- **GIVEN** the Telegram user is already registered
- **WHEN** the user sends `/start`
- **THEN** the system does not create a duplicate user
- **AND** replies with a returning-user greeting in Russian
- **AND** invites the user to choose a section
- **AND** attaches the same inline section keyboard

### Requirement: Navigate section stubs via inline callbacks

The system SHALL open stub screens for Cars, Houses, and Subscription when the corresponding inline action is pressed, acknowledge the callback, and allow returning to the main section menu by editing the same bot message.

#### Scenario [SC-MENU-01]: Open Cars stub

- **GIVEN** the user sees the main section menu message
- **WHEN** the user presses the Cars section action
- **THEN** the bot message is updated to a Russian Cars stub (placeholder content)
- **AND** a Back-to-menu control is shown
- **AND** the callback is acknowledged

#### Scenario [SC-MENU-02]: Open Houses stub

- **GIVEN** the user sees the main section menu message
- **WHEN** the user presses the Houses section action
- **THEN** the bot message is updated to a Russian Houses stub (placeholder content)
- **AND** a Back-to-menu control is shown
- **AND** the callback is acknowledged

#### Scenario [SC-MENU-03]: Open Subscription stub without gate

- **GIVEN** the user sees the main section menu message
- **WHEN** the user presses the Subscription section action
- **THEN** the bot message is updated to a Russian Subscription stub (placeholder content)
- **AND** the system does not refuse access based on subscription status
- **AND** a Back-to-menu control is shown
- **AND** the callback is acknowledged

#### Scenario [SC-MENU-04]: Return to main menu

- **GIVEN** the user is viewing any section stub with a Back-to-menu control
- **WHEN** the user presses Back to menu
- **THEN** the bot message is updated back to the section-choice prompt
- **AND** the main inline section keyboard is shown again
- **AND** the callback is acknowledged
