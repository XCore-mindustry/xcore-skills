# Menus & Text Input

Mindustry provides a server-side API for showing interactive menus and text input dialogs to players. The pattern is: register a handler once → get an ID → use that ID with `Call.menu()` or `Call.textInput()` whenever you want to show it.

**Import**: `import mindustry.ui.Menus;`

## Menus

### Register a menu handler (once, typically in `init()`)

```java
int myMenuId = Menus.registerMenu((player, option) -> {
    // option = index of the button pressed (0-based, left-to-right, top-to-bottom)
    // option = -1 if the player closed/dismissed the menu
    switch (option) {
        case 0 -> player.sendMessage("You chose A");
        case 1 -> player.sendMessage("You chose B");
        case 2 -> player.sendMessage("You chose C");
        case -1 -> player.sendMessage("Menu closed");
    }
});
```

### Show the menu to a player

```java
Call.menu(
    player.con,         // target connection
    myMenuId,           // the registered menu ID
    "Menu Title",       // title text
    "Choose an option", // message body
    new String[][]{     // button grid (rows × columns)
        {"Option A", "Option B"},  // row 1: two buttons
        {"Option C"}               // row 2: one button
    }
);
```

### Button grid layout

The `String[][]` defines a grid of buttons:
- Each inner array is a **row**
- Buttons in the same row appear side-by-side
- Rows stack vertically

```java
// Single column
new String[][]{{"Yes"}, {"No"}, {"Cancel"}}

// Two columns
new String[][]{{"Accept", "Decline"}}

// Mixed
new String[][]{
    {"Option 1", "Option 2", "Option 3"},
    {"Back"}
}
```

Button indices are assigned left-to-right, top-to-bottom:
```
[0: "Option 1"] [1: "Option 2"] [2: "Option 3"]
[3: "Back"]
```

### Follow-up menus

Update an existing menu without re-registering:

```java
// Show updated content in the same menu
Call.followUpMenu(player.con, myMenuId, "New Title", "Updated message",
    new String[][]{{"New Option A", "New Option B"}}
);

// Hide a follow-up menu
Call.hideFollowUpMenu(player.con, myMenuId);
```

---

## Text Input

### Register a text input handler (once)

```java
int myInputId = Menus.registerTextInput((player, text) -> {
    if (text == null) {
        // player cancelled / closed the dialog
        player.sendMessage("Input cancelled");
        return;
    }
    player.sendMessage("You entered: " + text);
});
```

### Show text input to a player

```java
Call.textInput(
    player.con,        // target connection
    myInputId,         // registered input ID
    "Enter Name",      // title
    "Type your name:", // message
    100,               // max character length
    "default text",    // pre-filled text (can be empty "")
    false              // numeric-only keyboard? (true = numbers only)
);
```

---

## Important notes

1. **Register once, reuse the ID.** Don't call `Menus.registerMenu()` every time you want to show a menu — register in `init()` and store the ID.

2. **IDs are global integers.** Different plugins share the same ID space. Each `registerMenu()` / `registerTextInput()` call returns a unique ID.

3. **One menu at a time per player.** Showing a new menu replaces the previous one on the client.

4. **Option -1 means dismissed.** Always handle this case — the player might press Escape or click outside.

5. **Text input returns null on cancel.** Always null-check the text parameter.

6. **Rich text in menus.** Title, message, and button text all support Mindustry color markup: `[accent]`, `[scarlet]`, `[#ff0000]`, `[]` (reset), etc.

## Patterns

### Confirmation dialog

```java
int confirmMenu = Menus.registerMenu((player, option) -> {
    if (option == 0) {
        // confirmed
        doTheAction(player);
    }
    // option 1 or -1 = cancelled, do nothing
});

void askConfirmation(Player player, String message) {
    Call.menu(player.con, confirmMenu, "Confirm", message,
        new String[][]{{"[green]Yes", "[scarlet]No"}}
    );
}
```

### Paginated list

```java
int listMenu = Menus.registerMenu((player, option) -> {
    int page = getPlayerPage(player);
    var items = getItems();
    int pageSize = 5;

    if (option == pageSize) {
        // "Next" button
        setPlayerPage(player, page + 1);
        showPage(player, page + 1);
    } else if (option == pageSize + 1) {
        // "Previous" button
        setPlayerPage(player, page - 1);
        showPage(player, page - 1);
    } else if (option >= 0) {
        int index = page * pageSize + option;
        handleSelection(player, items.get(index));
    }
});

void showPage(Player player, int page) {
    var items = getItems();
    int pageSize = 5;
    int start = page * pageSize;
    int end = Math.min(start + pageSize, items.size);

    String[][] buttons = new String[end - start + 1][];
    for (int i = start; i < end; i++) {
        buttons[i - start] = new String[]{items.get(i).name};
    }
    // Navigation row
    buttons[end - start] = new String[]{"Next >>", "<< Previous"};

    Call.menu(player.con, listMenu,
        "Items (page " + (page + 1) + ")",
        "Select an item:",
        buttons
    );
}
```

### Multi-step wizard

```java
int step1Menu, step2Menu;

// init()
step1Menu = Menus.registerMenu((player, option) -> {
    if (option >= 0) {
        storeStep1Choice(player, option);
        showStep2(player);
    }
});

step2Menu = Menus.registerMenu((player, option) -> {
    if (option >= 0) {
        int step1 = getStep1Choice(player);
        finishWizard(player, step1, option);
    }
});
```
