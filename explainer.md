# Keyboard Map

DRAFT: garykac@

## Overview

An API that returns a map from a `KeyboardEvent.code` value to a string that can be shown
to the user to identify that particular key on the keyboard.
 
### Goals include:

* Being able to convert from a `code` value (with no modifiers) into a string that
	is suitable for showing to the user to identify the physical key.
* Adding an event when the active keyboard layout changes.

### Non-goals include:

* Identifying a keyboard layout or locale
* Being able to convert from a `code` value a set of modifiers into the `key` value that
	would be generated by the current locale and layout with those modifiers present.
* Being able to convert from a `key` value into the set of `code` values required to
	generate that value, based on the current locale and layout.

For additional information, see Mozilla's
[Internationalize Your Keyboard Controls](https://hacks.mozilla.org/2017/03/internationalize-your-keyboard-controls/)
document. In the "What's missing" section of that document, it mentions the need for an
API like the one described here.

## Background

### `code` and `key` values

On a [KeyboardEvent](https://w3c.github.io/uievents/#idl-keyboardevent),
the [`code`](https://w3c.github.io/uievents/#dom-keyboardevent-code) attribute encodes
a value that represents the physical location of the key that was pressed. This value ignores
the current locale (e.g., "en-US"), layout (e.g., "dvorak") and modifier state (e.g., "Shift + Control"),
so it is ideally suited for applications (like games) that want to use the keyboard as a set of
generic buttons. The idea behind the `code` attribute is that it provides a platform-neutral
[scancode](https://en.wikipedia.org/wiki/Scancode) for each physical key.

The [`key`](https://w3c.github.io/uievents/#dom-keyboardevent-key) attribute, on the other hand,
contains the value that is generated by the key press, accounting for the locale, layout, and modifier
keys. Almost every Unicode character is a valid `key` attribute, along with a number of special
named values (see [KeyboardEvent key attribute values](https://w3c.github.io/uievents-key/#key-attr-values)),
so there are thousands of possible `key` values.

As a simple example of how `code` and `key` are related, consider the key which is located immediately
to the right of the Tab key. The `code` for this key is always `"KeyQ"`, and when this key is pressed on a
standard en-US layout with no modifiers, the `key` value is `"q"`. On the same layout, but with the
Shift key held down, the `key` value is `"Q"`. However, on a standard French keyboard, the `key` values
would be `"a"` and `"A"`, respectively; for Russian, `"й"` and `"Й"`.

### Keyboard layouts

When more than one keyboard layout is installed on a computer, they are typically arranged
in a priority list. The first layout on this list (the one highest in priority) is the
current layout that is used to generate the appropriate `key` value when a key is pressed.

For layouts that generate non-Latin characters (as with Russian), there is the concept of
the highest priority keyboard layout that is Latin-capable. This is the the first layout
in the priority list that produces Latin characters. For example, a QWERTY,
AZERTY or Dvorak layout.

For keyboard shortcuts, this Latin value is what is commonly used for shortcuts in
applications (even when the native writing system is not Latin-based). Keyboards in areas
where the common layout does not produce Latin
characters will have both the native and the Latin characaters printed on the 
physical key caps to make this easier for users..

Note the assumption that the labels on the keyboard match the currently active keyboard
locale. This is usually, but not always, true, but since there is no way to know that the actual labels
are, this is the best surrogate.

## User Scenarios

### Keypress instructions in Games

Applications (like games) that use the KeyboardEvent `code` attribute to handle key events will often
need to present a message to the user that references a particular key, for example in a
control settings dialog.

E.g., If a game supports the standard WASD keys (to move up/left/down/right), then the instructions for
the game need to be able to tell the user which keys to press. On a US-English keyboard, they are
'W', 'A', 'S', 'D', but for French (AZERTY) layout the user would be told to use 'Z', 'Q', 'S', 'D'.

### Describe keyboard shortcuts

Some applications make use of keyboard shortcuts that are based on the position of the key
rather than the symbol that is generated.

E.g., For Undo/Cut/Copy/Paste, an application may support the 'Z', 'X', 'C' and 'V' keys
along the bottom of the keyboard because they are easy to type with one hand in conjunction
with the (left) control key. This is easily done using the `code` values `"KeyZ"`, `"KeyX"`,
`"KeyC"` and `"KeyV"`. However, in order to have the UI display the shortcuts, the app needs to know
which `key` value will be generated by that key. On a US-English keyboard, the `"KeyZ"`
key is labeled "Z", but on a French keyboard it is "W" and on a German keyboard it is 
labeled "Y".

E.g., a drawing app may have a number of drawing modes arranged from left to right on the
screen and may wish to have keyboard shortcuts that correspond to the screen position.
E.g., an application may have tools for Pan, Move, Rotate and Scale and assign "Q", "W",
"E" and "R" shortcuts so that the shortcut key's relative position matches the display.

## Examples

### `getLayoutMap()`

The standard use for this API is to call it to get the current mapping and then use that
mapping table to update the UI.

```
navigator.keyboard.getLayoutMap().then(function(keyMap) {
  // Use keyMap
});
```

where `keyMap` is a dictionary where `code` maps to `key`, e.g.:

```
{
  'KeyA': 'a',
  'KeyB': 'b',
  ...
}
```

The mapping data can be used as follows:

```
navigator.keyboard.getLayoutMap().then(function(keyMap) {
  var keyUp = keyMap.get("KeyW");
  showUserDialog("Press " + keyUp + " to move up.");
});
```

### `layoutchange` Event

To detect when the keyboard layout has changed, pages can listen for the `layoutchange` event,
which will fire whenever the current keyboard layout changes.

```
navigator.keyboard.addEventListener("layoutchange", function(){
	// Refresh the game control settings page with current layout.
	updateGameControlSettingPage();
});
```

## Alternative Proposals

We initially considered having an API that would return the mapping for a single key, but
received feedback that it would be preferable to have the entire map all at once (since
that would avoid needing to ask for dozens of keys individually).

The simple single-key API was:

```
DOMString getKey(DOMString code)
```

which would have been used as follows for getting a single value:

```
var keyUp = navigator.keyboard.getKey("KeyW");
```

or for building a map of multiple values:

```
var keymap = {};
var keys = ["KeyW", "KeyA", "KeyS", "KeyD", "Key", "Key", "Key", "Key", "Key"];
keys.foreach(function(code) {
	keymap[code] = navigator.keyboard.getKey(code);
});
```

Requiring that this API return a Promise complicates matters a bit. The simple API 
would become:

```
Promise<DOMString> getKey(DOMString code)
```

getting a single value:

```
navigator.keyboard.getKey("KeyW").then(function(val) {
  var keyUp = val;
  // Do something with it.
});
```

and getting multiple values:

```
var keymap = {};
var k = navigator.keyboard.getKey;
Promise.all([k("KeyW"),k("KeyA"),k("KeyS"),k("KeyD"),
             k("KeyI"),k("KeyJ"),k("KeyK"),k("KeyL")]).then(values => {
               // Need to associate each return value with the code that produced it.
               // Index into |values| must match order of Promises.
               keymap["KeyW"] = values[0];
               keymap["KeyA"] = values[1];
               keymap["KeyS"] = values[2];
               keymap["KeyD"] = values[3];
               keymap["KeyI"] = values[4];
               keymap["KeyJ"] = values[5];
               keymap["KeyK"] = values[6];
               keymap["KeyL"] = values[7];
             }, () => {
               // Unable to get map
             })
```

This is awkward and error-prone to add or remove keys from the list.
Since we know that a primary use case (games) requires a relatively large number of 
keys to be mapped, we determined that this approach was unacceptable.
