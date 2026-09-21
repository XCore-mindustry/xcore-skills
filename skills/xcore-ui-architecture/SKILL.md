---
name: xcore-ui-architecture
description: Architecture guide for creating reactive server-driven UI in Mindustry v160 using xcore-ui. Use when designing menus, dialogs, forms, HUD components, or .msui templates.
---

# XCore-UI Architecture Guide

`xcore-ui` is a type-safe, reactive, Elm/MVI server-driven UI framework for Mindustry v160 running on Java 25.

## Architectural Principles

1. **Elm / MVI Flow**:
   State transitions are pure functions: `(Model, Event) -> UpdateResult<Model>`.
   Never mutate state directly inside UI click listeners.

2. **Record Lenses (`Lens<S, V>`)**:
   Bind state directly to Java 25 record components using method references:
   ```java
   public record ProfileState(String nickname, boolean notificationsEnabled) {}

   Lens<ProfileState, String> NICKNAME_LENS = Lens.of(
       ProfileState::nickname,
       ProfileState::withNickname
   );
   ```

3. **Zero-Flicker Partial Updates**:
   - Wrap dynamic sections into slots (`VSlot`).
   - Emit granular `Call.menuBuilderUpdate` calls targeting dynamic slots instead of re-rendering whole dialogs. This preserves player scrollbar position and input focus.

4. **Testing without Graphics Context**:
   The Virtual DOM (`VNode`) runs under headless pure JUnit 5 in milliseconds without mocks or display contexts:
   ```java
   @Test
   void shouldRenderLeaderboardRows() {
       var vnode = ProfileView.render(sampleState);
       assertThat(vnode.findButton("back")).isPresent();
   }
   ```
