# The widget: what a person sees

`widget/dist/rosie-widget.js` is one file with no build step and no dependency. It is served
by the backend at `/api/assistant/widget.js`. This page says what it shows and why, so a
change to it can be judged against the intent. Installing it is in
[INSTALL.md](INSTALL.md), step 6; theming is in `contexts/example/README.md`.

## An answer is a card, read from the top

1. **The verdict**, in words and with an icon, before any number: `No repeated rows`,
   `Checks passed`, `Approved query`, `Read with care`, `Checks failed`, `Contradicted`,
   `Not checked`, `No answer`. An answer the certificate has nothing to say about
   (`NOT_APPLICABLE`, or no certificate) has no verdict row at all.
2. **The figure.** One number for a one-value answer; "N rows" for a table; a short list of
   label and value for one row of several columns. When the certificate contradicts an
   answer that is a single value (one row, one column, no groups in the certificate), both
   values are shown, the returned one struck through, each with the route the engine took,
   word for word. On a grouped answer the two values belong to one group: the figure stays
   "N rows", or the one row is shown whole, the group beside its value, when a filter left a
   single group, and the evidence line says which group.
3. **What the model said.** Under a contradicted figure it is shown faint and without bold.
4. **The evidence**: the certificate's sentences, the engine's own words. What passed is
   written on the line, not in a tooltip. A second derivation that agreed is said whatever
   the verdict is, so an answer that is `Not checked` for another reason shows both facts.
   When the question review is on (`AI_QUESTION_REVIEW`), its reading is a separate line
   marked as an opinion, outside the certificate, and it never changes the verdict. A
   reading that says the question was another one (`does_not_answer`) is written as a
   disagreement, in the amber of a caution, under a verdict that stays what it was.
5. **Actions**: copy, the SQL, open the results, the two votes. **Copy takes the card, not
   only the sentence**: the verdict, the figures with their routes, the text and the
   evidence, so a figure pasted somewhere else carries its caveat. **The values copied are
   the ones that arrived**, with no separator and no added zero (`20848.62`, not
   `20,848.62`): a code pasted into another system must be the code.

`CROSS_CHECKED` is shown as **No repeated rows**, with the tone and the tick of
`Checks passed`. Its line says what was established and what was not: "The figure is the
same when each record is taken once. Whether this is the right question was not checked.",
followed by the engine's sentence. The verdict code in the API is unchanged. It was renamed
because a second derivation that agrees is a negative finding (the join repeated nothing),
and on a benchmark its share of right answers went from 16% to 93% with the model, so it
cannot be the best-looking verdict.

An answer that did not arrive (`unfinished` in the response: a limit was reached, or the
query meant as the answer did not run) is a `No answer` card. It never reads as "0 rows".

An answer in words only (no statement ran, so there is no certificate) carries one quiet
line where the evidence would be: "No query ran for this answer: nothing here was checked."
It is not a verdict and not a warning. A question back to the user (a clarification) is not
an answer and has no such line.

## The character shows the verdict of the last answer

| The last answer | Face | Badge |
|---|---|---|
| none yet, a clarification, no verdict | at rest | the green status light |
| question sent | three dots where the eyes are | the status light |
| `CHECKS_PASSED`, `CROSS_CHECKED` (the same as it), an approved query | at rest | green, a tick |
| `FLAGGED` by a warning | one eyelid lower than the other, flat mouth | amber, an exclamation mark |
| `NOT_CHECKED` | both eyelids half down, flat mouth | grey, a question mark |
| `CONTRADICTED`, or `FLAGGED` by a failed check | worried eyelids, mouth down | red, a cross |
| the request failed, or the answer is unfinished | flat eyes, flat mouth | grey, a dash |
| the security hold | grey body, still | grey, a dash |

- **She is glad for a second and a half, and only for an approved query or a declared
  metric compiled by the engine that raised nothing**: the two answers where a person
  defined what the number means. Not for an answer that merely passed, not for
  `CROSS_CHECKED` (a second derivation does not know the question), never beside a figure
  the certificate questions (`CONTRADICTED`, `FLAGGED`, `NOT_CHECKED`, no answer), and not
  when the question review says the question was another one.
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
  is there too, more quietly (`.rosie-results-evidence`), and so is the second reading of
  the question when the card has one (`.rosie-results-review`, amber when it disagrees): a
  table opens before the card is read, so it carries the same evidence as the card.
- **Numbers** follow the rule in the next section, the same on the card, in the table and in
  the chart labels.
- **A missing value is written `null`**, in faint ink; an empty text is an empty cell.
- **Sorting is a total order**, so the result does not depend on where the rows started:
  numbers first, by value; then everything else as text; a missing value last. Money that
  arrives as text sorts as a number, and so does a decimal too long for a float (it is
  compared digit by digit: sign, whole part, fraction). Only the order inside the numbers
  and inside the texts turns with the direction of the sort.
- **A full export that fails downloads nothing** and says so on that screen. The rows on
  screen are not saved in its place.
- A cell cut by the column width has its whole value in a tooltip.

## Numbers: the digits on screen are the digits that arrived

The widget may add thousands separators and trailing zeros to a number, and nothing else.

- **It never rounds.** `0.00001` is written `0.00001`, `0.004875` keeps its six places. A
  value that is not zero is never written as zero. The one exception is float noise: a value
  that arrives as a JSON number is written with the 15 significant digits a double holds
  exactly, so `0.1 + 0.2` reads `0.3`.
- **Separators and lining-up zeros need a reason.** They are added only to a column there
  is a reason to read as a measure: the statement computes it with `COUNT`, `SUM` or `AVG`
  (the widget reads the outer `SELECT` of the answer's SQL), or its name reads like a
  measure (`total`, `amount`, `order_count`, `price`...), or a value arrived as a number
  with a fraction, which no identifier has. A name that reads like an identifier, a year or
  a code (`customer_id`, `InvoiceId`, `year`, `price_code`) cancels the last two.
- **Without a reason the value is written as it came.** A code that arrives as text
  (`"5102"`, `"84713012"`, a phone, a barcode, `"201301"`) and an integer nobody computed
  (a store number, `202511`) get no separator. `MAX(id)` is not read as a measure.
- **Zeros are a minimum, never a maximum.** In a measure column `195.1` is written `195.10`
  under `523.06` (up to four places are added to line a column up); a value with more places
  than the others keeps them all.
- **Too many digits for a float is text.** A value with a leading zero, an exponent or more
  than 15 significant digits, on either side of the decimal point, is left as the text it
  arrived as, and its column is not formatted. It still sorts by value.
- Formatting is done on the decimal digits, never through a float, and numbers are written
  the English way (`20,848.62`) whatever the browser's locale, like the model's sentence
  beside them.
- The name tests and the reading of the SQL are guesses, not knowledge of the data. They
  are built so that a wrong guess costs a separator and never a digit: a measure the widget
  does not recognise reads `12345` where `12,345` would be easier, and that is the side it
  errs on. Column metadata from the server would replace the guesses; it does not exist yet.

## The CSV file carries the verdict

- **The cells are the values that arrived**, never the formatted ones.
- **A file whose answer has a caveat starts with it.** For `CONTRADICTED`, `FLAGGED` and
  `NOT_CHECKED` the first row is `Rosie verdict`, the verdict code and one sentence; the
  next rows are `Rosie note` and each sentence of the certificate that questions the answer;
  then one empty row, the header, the rows. A second reading that disagrees adds a
  `Rosie question review` row, whatever the verdict. The file name ends with the verdict
  (`rosie-export-2026-11-15-contradicted.csv`), which survives someone deleting those rows.
- **An answer with nothing raised has no such rows**: its file starts with the header, as it
  always has, so nothing that reads these files changes for an answer that passed.
- **Why on top and not at the end or in a column.** A section after 50,000 rows is not read.
  A verdict column on every row changes the shape of the table and is sorted, filtered and
  summed like data. Rows on top are the first thing a person sees in a spreadsheet, and the
  empty row keeps them apart from the table. A program that expects the header on the first
  line fails on a file with a caveat, and it should: that file is not one to load without
  being read.
- **A file that is part of the result says so** in a `Rosie note` row: the rows on screen of
  a larger answer, or a full export that stopped at the export limit.
- The header row is escaped like the cells (quotes, commas, line breaks, and the apostrophe
  in front of a cell that starts like a formula). The caveat rows too.
- All three exports (selected rows, current view, full results) go through one function.
  `POST /export` returns the verdict the answer's handle was signed with (`verdict`), so a
  client that is not this widget can write the same rows.

## Charts

Chosen from the shape of the result (`chartPlan`), never by the model. Bars start at zero
and go the way of their sign. Columns share an axis only within a factor of 10; the others
are left to the table and the chart says so. A result with two rows for the same period and
category gets no chart.

## Not verified

Rendered and tested in Chrome only. Safari, Firefox, a screen reader and a phone's keyboard
have not been tried on this version.
