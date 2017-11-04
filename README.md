# textmate-bug-repro README

## Extension Overview

This repository demonstrates inconsistent behavior in features that rely on `getWordAtText` for word patterns that can include spaces.

It relies on a very simple language definition with two key features:

1. A [word pattern](./language-configuration.json) that:
    * Allows spaces
    * Identifiers in this language are surrounded by double quoted strings and allows double quoted strings in the identifier by escaping them with an extra double quote. For example, `"An ""Identifier"" Example"`
2. A [simple grammar](./syntaxes/bug.tmLanguage.json):
    * A root token scope of "bug"
    * A "punctuation" token for semicolon

## Repro Steps

To observe the inconsistencies, launch this extension or install it from [.vsix](./textmate-bug-repro-0.0.1.vsix) and open this extension directory.

Open the [test file](./testfile.bug) with the following contents:

```
hi ; test ; "this "" is "" a test"

"this "" is "" a test"

hi ; test ; "this "" is "" a test"
```

Place the cursor in the word `is` on each of the lines and press `F2` to trigger a rename action. With the default "Dark+" theme, the behavior is as follows:

* Line 1: `is` appears in the rename box
* Line 3: `"this "" is "" a test"` appears in the rename box
* Line 5: `is` appears in the rename box

Line 3 behaves as expected, but lines 1 and 5 do not.

Now open the [workspace settings](./.vscode/settings.json) and uncomment out the `editor.tokenColorCustomizations` settings.

Go back to [test file](./testfile.bug) and repeat the rename steps from above. This time the behavior is as follows:

* Line 1: `"this "" is "" a test"` appears in the rename box
* Line 3: `"this "" is "" a test"` appears in the rename box
* Line 5: `is` appears in the rename box

Now Line 1 and line 3 behave as expected, but Line 5 still does not.

This is explained in a couple ways:

* Line 3 always works because there are no other tokens on the line, so the line can be treated as a single token and processed by the word pattern
* Line 1 works after enabling the coloring because coloring `;` helps [LineToken.tokenAt](https://github.com/Microsoft/vscode/blob/master/src/vs/editor/common/core/lineTokens.ts#L141) return the correct token. With the `;` colorized it returns a token starting after the `;`, otherwise it returns a token starting at the beginning of the line. This then applies the word pattern to the text `hi ; test ; "this "" is "" a test"`. The word pattern matches on `hi` at this line in [wordHelper](https://github.com/Microsoft/vscode/blob/master/src/vs/editor/common/model/wordHelper.ts#L121), and since it doesn't have a space in it then it tries to use `getWordAtPosFast` which won't work because the regex DOES allow spaces in it.
* Line 5 never works because [this line in textModelWithTokens](https://github.com/Microsoft/vscode/blob/master/src/vs/editor/common/model/textModelWithTokens.ts#L437) claims it is not tokenized (an off by 1 error?). Due to this, it behaves the same as line 1 before adding the color because it treats the entire line as a token/starts searching from offset 0. If I debug, though, I can see that `this._lines` has 5 entries, and the 5th entry has a valid `_lineTokens` value, so it should not follow this path.

Now, to make it work in all three, simply add a blank line to the end of the file, forcing VS Code to recognize that line 5 is tokenized. If you go back to [test file](./testfile.bug) and repeat the rename steps from above the behavior is as follows:

* Line 1: `"this "" is "" a test"` appears in the rename box
* Line 3: `"this "" is "" a test"` appears in the rename box
* Line 5: `"this "" is "" a test"` appears in the rename box