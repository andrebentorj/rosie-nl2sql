# The widget: what a person sees

`widget/dist/rosie-widget.js` is one file with no build step and no dependency. It is served
by the backend at `/api/assistant/widget.js`. This page says what it shows and why, so a
change to it can be judged against the intent. Installing it is in
[INSTALL.md](INSTALL.md), step 6; theming is in `contexts/example/README.md`.

## An answer is a card, read from the top

1. **The verdict**, in words and with an icon, before any number: `No repeated rows`,
   `Checks passed`, `Approved query`, `Declared metric`, `Read with care`, `Checks failed`,
   `Contradicted`, `Not checked`, `No answer`. An answer the certificate has nothing to say
   about (`NOT_APPLICABLE`, or no certificate) has no verdict row at all, unless it ran an
   approved query, which is always named.
2. **The figure.** One value for a one-value answer; "N rows" for a table; a short list of
   label and value for one row of several columns. A value that is not a number (a name, a
   date, `true`, `false`) is the figure too, as it came and a size smaller; a missing value
   is written `null` and an empty text `(empty text)`, both faint, so that `false`, nothing
   and no value read three different ways. When the certificate contradicts an
   answer that is a single value (one row, one column, no groups in the certificate), both
   values are shown, the returned one struck through, each with the route the engine took,
   word for word. On a grouped answer the two values belong to one group: the figure stays
   "N rows", or the one row is shown whole, the group beside its value, when a filter left a
   single group, and the evidence line says which group.
3. **What the model said.** Under a contradicted figure it is shown faint and without bold.
   When the response carries `narrativeReplaced` (the model's sentence did not carry the
   figure of the result and the engine put its own in its place), the sentence is drawn as a
   caution, in the amber band of a second reading that disagrees: it is a warning, not
   ordinary prose.
4. **The evidence**: the certificate's sentences, the engine's own words. What passed is
   written on the line, not in a tooltip, as plain text under a hairline with no band
   behind it; a band is for what was flagged, what was not checked, and who approved. A second derivation that agreed is said whatever
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

## A certificate that raised nothing is not painted as a right answer

| The answer | Card | Icon | Evidence |
|---|---|---|---|
| `CHECKS_PASSED`, `CROSS_CHECKED` | plain (`data-tone="plain"`): grey edge, grey words | a small grey tick | plain text, no band |
| `NOT_CHECKED` | mute: grey edge, grey words | a dashed circle | a grey band that starts "Not checked." |
| an approved query or a declared metric whose certificate raised nothing | green (`data-tone="ok"`) | a person, or a tick for a metric | the approval or the metric in a green band, the checks in plain text |
| an approved query with no certificate (the checks are off), or with a check skipped | plain, named `Approved query` | a person | the approval in its blue band |

- **Green is for one case only**: the answer ran an approved query or a declared metric
  compiled by the engine, and its certificate is `CHECKS_PASSED`, `CROSS_CHECKED` or
  `NOT_APPLICABLE` with every assertion a pass, nothing skipped and no second count that
  differs. A person defined what that number means and the checks found nothing against it.
  With no certificate at all nobody looked, so the card is plain and the approval line is
  still shown.
- **Why.** Checks that pass say that nothing known to be wrong was found, not that the
  answer is the right one. Counted on a benchmark ([BIRD.md](BIRD.md), "Right answers by
  verdict"), `CHECKS_PASSED` answers were right 62% to 89% of the time and `CROSS_CHECKED`
  16% to 99%, with the model. A green card and a green tick on the character said more
  than that.
- **`Not checked` must keep reading as "nobody looked"**, so it is not the same card: its
  own icon and words, a grey band, and the character with half-closed eyes and a question
  mark. A passed check has none of the three.

`CROSS_CHECKED` is shown as **No repeated rows**, with the plain tone and the tick of
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
| `CHECKS_PASSED`, `CROSS_CHECKED` | at rest | the status light, no badge |
| question sent | three dots where the eyes are | the status light |
| an approved query or a declared metric whose certificate raised nothing | at rest | green, a tick |
| `FLAGGED` by a warning | one eyelid lower than the other, flat mouth | amber, an exclamation mark |
| `NOT_CHECKED` | both eyelids half down, flat mouth | grey, a question mark |
| `CONTRADICTED`, or `FLAGGED` by a failed check | worried eyelids, mouth down | red, a cross |
| the request failed, or the answer is unfinished | flat eyes, flat mouth | grey, a dash |
| the security hold | grey body, still | grey, a dash |

- **She is glad for a second and a half, and only for an approved query or a declared
  metric compiled by the engine that raised nothing**: the two answers where a person
  defined what the number means, under the rule for green above (a certificate that ran,
  nothing flagged, nothing skipped). Not for an answer that merely passed, not for
  `CROSS_CHECKED` (a second derivation does not know the question), never beside a figure
  the certificate questions (`CONTRADICTED`, `FLAGGED`, `NOT_CHECKED`, no answer), not when
  there is no certificate, and not when the question review says the question was another
  one (the card and the badge stay as they were, only the gladness stops).
- **The green tick badge goes with the green card and with nothing else.** An answer that
  merely passed leaves her exactly as she is at rest.
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
  is there too, more quietly (`.rosie-results-evidence`, grey for checks that passed, blue
  for an approval), and so is the second reading of
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
- Formatting is done on the decimal digits, never through a float. Since 1.33.0 (widget
  2026.11.17) the thousands separator and the decimal mark follow the widget's locale: the
  `locale` attribute, else the browser's language, else English. `20848.62` reads `20,848.62`
  in English and `20.848,62` in German; the digits never change. The model's sentence
  beside them is the model's and may still write `20,848.62`. `locale="en"` keeps the English
  way everywhere.
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
- **Separator and encoding (since 1.33.0).** Every file starts with a UTF-8 byte order mark,
  so Excel reads accents. Where the widget's locale writes decimals with a comma (German,
  French, Spanish, Italian...), the cells are separated by `;` and a cell that is a
  plain decimal number is written with a decimal comma (`20848,62`), which is what Excel
  expects there; codes and text are untouched. Elsewhere the separator is `,` and the decimal
  mark `.`, as before. A program that reads exported files from such a browser must accept `;`.
- The header row is escaped like the cells (quotes, the separator, line breaks, and the
  apostrophe in front of a cell that starts like a formula). The caveat rows too.
- All three exports (selected rows, current view, full results) go through one function.
  `POST /export` returns the verdict the answer's handle was signed with (`verdict`), so a
  client that is not this widget can write the same rows.

## Messages, waiting and the element's attributes

- **Messages are written for the person asking**, never the browser's or the server's raw
  text: "Cannot reach Rosie. Check your connection and try again." for a network failure,
  "Your session has expired. Reload the page and ask again." for a 401, "You have asked as
  many questions as this assistant allows for now." with the time to wait for a 429, "Many
  people are asking right now." for a busy server, and so on. A sentence the server wrote for
  a person (the safety filter, a question too long) is kept. When the error body carries a
  `reference`, it is shown on its own line ("Reference: 3f2a9c10") to quote to whoever looks
  after Rosie.
- **A refused chat-page request** (`403` with `code: "E_CSRF"`, see "Signed in on Rosie's
  own chat page") says Rosie could not confirm the question came from this page, to reload
  and ask again, and the code to quote. Any other `403` keeps "Your account cannot use Rosie
  for this."
- **A security hold** reads "Rosie is paused", with the server's plain message
  (`securityMessage`), whom to tell (`supportContact` from `project.json`) and the code.
- **A busy server** (`503` with `code: "E_BUSY"`) is asked again twice, after `retryAfter`
  seconds and then twice that (at most 20 s each), with "Waiting for a free slot..." meanwhile.
- **A refused question** (`blocked: true` on `/ask`) shows no thumbs and no "nothing was
  checked" line.

| Attribute or event | Default | What it does |
|---|---|---|
| `locale` | the browser's language, else `en` | BCP 47 tag (`en-GB`, `de-DE`) for number separators and the CSV rule above; the words are always English |
| `timeout-seconds` | `130` | How long the widget waits for an answer before it stops the question (the server allows 120 s) |
| `layout` | floating | `page` for a page that is only the chat (see "The full-page layout" below). Anything else, or no attribute, is the floating launcher and panel |
| `rosie:auth-expired` (event) | | Fired on the element on a `401`; bubbles and crosses the shadow root; `detail.path` is the route. A host page can refresh its token and ask again |

Requests to the widget's own origin carry the header `X-Rosie-Widget-Version`; across
origins the header is left out, because a server before 1.33.0 refuses it in the preflight.
`/ask` always carries the version in its body (`widgetVersion`), so the server records which
widget asked from any origin. Rosie's CORS layer allows the header since 1.33.0; a reverse
proxy that answers CORS preflights itself must allow it too.

## The full-page layout

`layout="page"` is for a page whose whole purpose is the chat, such as the chat page Rosie
serves itself. The floating mode, the default, does not change at all.

- There is no launcher, no close button and no expand button. "New conversation" stays.
- The panel opens by itself once `/project-info` has answered (or failed), so the first thing
  drawn is the project's greeting and chips, or the hold card. It stays open: Escape does not
  close it, and Tab is not held inside it, so the page's own controls (a "Sign out" link) can
  be reached. It is a region of the page, not a dialog.
- The panel fills the element, with no radius and no shadow. The page gives the element its
  place and size; an element with no height shows nothing. Under a 48px top bar:

  ```html
  <style>rosie-widget { position: fixed; top: 48px; left: 0; right: 0; bottom: 0; }</style>
  <rosie-widget layout="page" api-url="/api/assistant"></rosie-widget>
  ```

  A flex child (`flex: 1; min-height: 0`) works as well.
- The purple header, the messages, the disclaimer and the input bar keep their full-width
  backgrounds and hold their content in a centred 880px column, with 12px on each side when
  the page is narrower. The text keeps the widget's own size (13px).
- The results screen opens over the page as it does over a host app. Inside a positioned
  element (`position: fixed` above), it stays under a top bar that sets a `z-index`.

## Signed in on Rosie's own chat page

Rosie's chat page signs people in and keeps them signed in with a session cookie. The widget
on that page does not need, and does not hold, the deployment's API token:

- The page puts a CSRF token in its head: `<meta name="rosie-csrf-token" content="...">`.
- When that meta is present and `api-url` is on the page's own origin, every request (the
  project info, questions, export, explain, feedback) carries the header `X-Rosie-CSRF` with
  the token and is sent with `credentials: 'same-origin'`, so the session cookie goes with it.
  The meta is read again at each request, so a token the page replaces is picked up.
- On such a page an `api-token` attribute is ignored, also when it is set later, and the
  console says so. No `Authorization` header is sent.
- A value that is not a plain token (16 to 512 of `A-Z a-z 0-9 . _ ~ + / = -`) is not sent.
- Across origins the meta is never read and the token never leaves the page. An existing
  embedding (`api-token`, a same-origin proxy, or a cross-origin `api-url`) works exactly as
  before.
- A `401` (the session ended) fires `rosie:auth-expired` as everywhere else; the page decides
  whether to send the person to sign in again.
- The server requires the header on every call that is not a `GET` and carries the session
  cookie, together with an `Origin` equal to the page's public URL; otherwise it answers `403`
  with `code: "E_CSRF"`. A widget older than this one does not send the header, so the chat
  page needs this widget or later; the server's own `/api/assistant/widget.js` always matches.

## Charts

Chosen from the shape of the result (`chartPlan`), never by the model. Bars start at zero
and go the way of their sign. Columns share an axis only within a factor of 10; the others
are left to the table and the chart says so. A result with two rows for the same period and
category gets no chart.

## Not verified

Rendered and tested in Chrome only. Safari, Firefox, a screen reader and a phone's keyboard
have not been tried on this version.
