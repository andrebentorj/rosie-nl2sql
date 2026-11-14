# The widget: what a person sees

`widget/dist/rosie-widget.js` is one file with no build step and no dependency. It is served
by the backend at `/api/assistant/widget.js`. This page says what it shows and why, so a
change to it can be judged against the intent. Installing it is in
[INSTALL.md](INSTALL.md), step 6; theming is in `contexts/example/README.md`.

## An answer is a card, read from the top

1. **The verdict**, in words and with an icon, before any number: `Cross-checked`,
   `Checks passed`, `Approved query`, `Read with care`, `Checks failed`, `Contradicted`,
   `Not checked`, `No answer`. An answer the certificate has nothing to say about
   (`NOT_APPLICABLE`, or no certificate) has no verdict row at all.
2. **The figure.** One number for a one-value answer; "N rows" for a table; a short list of
   label and value for one row of several columns. When the certificate contradicts a single
   figure, both values are shown, the returned one struck through, each with the route the
   engine took, word for word. On a grouped answer the two values belong to one group of
   many, so the figure stays "N rows" and the evidence line says which group.
3. **What the model said.** Under a contradicted figure it is shown faint and without bold.
4. **The evidence**: the certificate's sentences, the engine's own words. What passed is
   written on the line, not in a tooltip. When the question review is on
   (`AI_QUESTION_REVIEW`), its reading is a separate line marked as an opinion, outside the
   certificate, and it never changes the verdict.
5. **Actions**: copy, the SQL, open the results, the two votes. **Copy takes the card, not
   only the sentence**: the verdict, the figures with their routes, the text and the
   evidence, so a figure pasted somewhere else carries its caveat.

An answer that did not arrive (`unfinished` in the response: a limit was reached, or the
query meant as the answer did not run) is a `No answer` card. It never reads as "0 rows".

## The character shows the verdict of the last answer

| The last answer | Face | Badge |
|---|---|---|
| none yet, a clarification, no verdict | at rest | the green status light |
| question sent | three dots where the eyes are | the status light |
| `CROSS_CHECKED`, `CHECKS_PASSED`, an approved query | at rest | green, a tick |
| `FLAGGED` by a warning | one eyelid lower than the other, flat mouth | amber, an exclamation mark |
| `NOT_CHECKED` | both eyelids half down, flat mouth | grey, a question mark |
| `CONTRADICTED`, or `FLAGGED` by a failed check | worried eyelids, mouth down | red, a cross |
| the request failed, or the answer is unfinished | flat eyes, flat mouth | grey, a dash |
| the security hold | grey body, still | grey, a dash |

- **She is glad for a second and a half, and only for a cross-checked answer or an approved
  query that raised nothing.** Not for an answer that merely passed, never beside a figure
  the certificate questions, and not when the question review says the question was another
  one.
- **The state stays until the next question or a new conversation.** No timer takes her back
  to rest, so the badge is still there when the panel is closed.
- **At rest she is the drawing she has always been**, with the same float, breathing and
  lights. The faces are layers that are invisible at rest. The unit suite pins the resting
  shapes; do not change them in passing.
- The verdict is never on the character alone: it is always written on the card.
- `prefers-reduced-motion`: no float, no nod, no blink, still dots, loops paused. Every state
  is a shape and a badge, so it is still told apart.
- `brand.hideMascot` replaces her with a small neutral launcher in the theme colour, in the
  header and on the results screen as well.

## The results screen

- Opens by itself for an answer of more than one row, and again from "Open results" on that
  answer's card. Closing it returns the focus to that button.
- The launcher is hidden while it is open. Focus lands on the title.
- A caveat is a banner above the table (`.rosie-results-verdict`). What speaks for the figure
  is there too, more quietly (`.rosie-results-evidence`).
- **Numbers**: a column whose values are all numbers is right-aligned, with the decimals of
  its most precise value. Columns named like identifiers, years or codes get no thousands
  separator. A value with a leading zero, an exponent or more than 15 digits is left as the
  text it arrived as. Numbers are written the English way (`20,848.62`) whatever the
  browser's locale, like the model's sentence beside them.
- **A missing value is written `null`**, in faint ink; an empty text is an empty cell.
- **Sorting is by value**: money that arrives as text sorts as a number, a missing value last.
- **A full export that fails downloads nothing** and says so on that screen. The rows on
  screen are not saved in its place.
- A cell cut by the column width has its whole value in a tooltip.

## Charts

Chosen from the shape of the result (`chartPlan`), never by the model. Bars start at zero
and go the way of their sign. Columns share an axis only within a factor of 10; the others
are left to the table and the chart says so. A result with two rows for the same period and
category gets no chart.

## Not verified

Rendered and tested in Chrome only. Safari, Firefox, a screen reader and a phone's keyboard
have not been tried on this version.
