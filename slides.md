---
title: "E2E testing with AWS Textract"
theme: white
slideNumber: true
revealjs-url: https://unpkg.com/reveal.js@4.6.1
header-includes: |
  <link href="https://fonts.googleapis.com/css2?family=Work+Sans:wght@300;400;500&display=swap" rel="stylesheet">
  <style>
  :root {
    --r-heading1-size: 1.5em;
  }
   .reveal h1, .reveal h2, .reveal h3, .reveal h4, .reveal h5, .reveal h6 {
      font-weight: 300;
      text-transform: none;
      font-family: var(--heading-font);
      color: #e00073;
   } 
   .reveal {
        font-size: var(--main-font-size);
        font-family: var(--main-font); 
        font-weight: 400;
        font-size: 32px;
        height: calc(100% - 60px);
   }
   .reveal strong {
        font-weight: 500;
   }
   .reveal a {
      color: #0069CC;
   }
   .reveal a:hover, .reveal a:active {
      color: #007EF5;
   }
   .reveal u {
      text-decoration-color: #e00073;
   }
   .reveal ul {
     list-style-type: square;
     list-style-color: #e00073;
   }
   .reveal li::marker {
     color: #e00073;
   }
   .reveal li {
     margin-bottom: 0.75rem;
   }
   :root {
        --main-font: 'Work Sans', sans-serif;
       --heading-font: 'Work Sans', sans-serif;
   }
   .reveal-viewport {
     background-color: #fff;
   }
   .reveal-viewport::after {
      content: "@coderbyheart | CC-BY-NC-SA-4.0";
      position: absolute;
      bottom: 0; 
      left: 0;
      width: 100%; 
      height: 60px;
      background-color: #191919;
      color: white;
      font-family: var(--main-font);
      font-size: 22px;
      font-weight: 500;
      line-height: 60px;
      padding-left: 10vw;
   }
   #markus-tacker img {
      border-radius: 100%;
    }
    #markus-tacker ul {
      font-size: 80%;
      margin-top: 6rem;
    }
    #markus-tacker div.column:first-child {
      text-align: center;
    }
    #title-slide h1 {
      color: #e00073;
      font-weight: 500;
      font-size: 60px;
    }
    #title-slide h1:after {
      content: "How we verify what Stripe renders";
      display: block;
      color: #222;
      padding: 1rem;
      margin-top: 2rem;
      font-weight: 300;
      font-size: 32px;
    }
    #title-slide:after {
      content: "September 2026";
      font-size: 22px;
      color: #191919;
      font-style: italic;
    }
    figcaption {
      display: none;
    }
    div.column {
      text-align: left;
      width: calc(50% - 1rem); 
      margin-right: 1rem;
    }
    div.column + div.column {
      margin-left: 1rem;
      margin-right: 0;
    }
    .slide p {
      text-align: left;
    }
    div.text-center p {
      text-align: center;
    }
    .reveal pre {
      box-shadow: none;
    }
    .reveal .slide-number {
      bottom: initial;
      top: 8px;
      color: #333;
      background-color: transparent;
      font-size: 36px;
    }
    .reveal pre code {
      font-size: 1rem;
      line-height: 1.2;
    }
  </style>
---

# Markus Tacker

:::::::::::: {.columns}

::: {.column width=40%}

![Markus Tacker](./me-2026.jpg){width=50%}

:::

::: {.column width=48%}

<small>Pronouns: he/him</small>

Principal R&D Engineer  
Nordic Semiconductor  
Oslo, Norway

[coderbyheart.com/socials](https://coderbyheart.com/socials)

:::

::::::::::::

# The project

Stripe integration for nRF Cloud:  
Creates Stripe invoices from nRF Cloud billing data

::: {style="margin-top:2rem"}

Every invoice is a **legal document**.

:::

# The invoicing pipeline

_Very simplified:_

- Invoices are created in Stripe at the end of each month
- Stripe sends the PDF to the customer

# The legal requirement

Norwegian law
([Altinn](https://info.altinn.no/en/start-and-run-business/accounts-and-auditing/accounting/invoices-sales-documentation/))
requires every sales document to carry:

- Seller business name
- Organization number with the `MVA` suffix
- `Foretaksregisteret` (the Enterprise Register line)
- Buyer name and address
- Date of issue and payment deadline
- A clear description of each service
- Total price and remuneration

# Finance department rules

- **VAT only for Norwegian customers**
- **Currency conversion note** — state the exchange rate, source (Norges Bank /
  Deutsche Bundesbank) and quote date
- **NOK VAT in foreign currency** — Norwegian customers billed in non-NOK still
  get VAT reported in NOK at the effective Norges Bank rate
- **"All prices in NOK."** footer on every NOK invoice (the `kr` symbol is
  ambiguous)
- **Issue date** — last day of the billed month (post-paid) / first day
  (pre-paid), **due 30 days** later
- **Automatic-charge footer** — charged invoices state the payment term in lieu
  of a due date

# The catch

Invoices are **rendered by Stripe**, not by us.

- We create the invoice through the API
- Stripe renders the PDF from an invoice **template**
- The final document contains additional information we pass along when creating
  the invoice

# How to test?

We cannot assert on rendered output with unit tests:

- our implementation only does the API calls
- multiple API calls needed to create an invoice (create customer, create
  invoice, ...)
- API calls are not what the customer receives: they get a PDF!
- Finance does not read unit tests!

# E2E tests — the top of the testing pyramid

- Drive the **full pipeline** through deployed AWS infrastructure
- Talk to the **real Stripe sandbox** — no mocking of the third party under test
- Assert on the **rendered PDF** via OCR
- Few, slow, expensive — but high confidence

# The testing pyramid

![Testing pyramid](./testing-pyramid.svg)

# The approach

Run the **real pipeline** against the **real Stripe sandbox**:

1. Deploy an ephemeral stack to AWS
2. Seed the dependent services (mocked)
3. Invoke the deployed lambda
4. Let Stripe create and render the invoice
5. Download the rendered PDF
6. **OCR it** and assert on the text

# Architecture

![Architecture](./architecture.svg)

# Why OCR?

Stripe's invoice PDFs embed fonts whose text layer maps punctuation (`-`, `(`,
`:`) to `NUL`.

```text
2026-06-01   →   2026\x0006\x0001
```

Reading the text layer directly (e.g. with pdf.js) **mangles values**.

Amazon Textract OCRs the document — it reads the **glyphs as they are drawn** —
and preserves punctuation, spaces and line breaks.

Also provides document structure (Layout, Forms, Tables, Queries, Signatures)
but this is not needed for us.

# 1. Upload the PDF

```typescript
import { PutObjectCommand, S3Client } from "@aws-sdk/client-s3";

const s3 = new S3Client();
const key = `invoice-${randomUUID()}.pdf`;

await s3.send(
  new PutObjectCommand({
    Bucket: documentsBucketName,
    Key: key,
    Body: pdfBytes,
    ContentType: "application/pdf",
  }),
);
```

# 2. Start text detection

```typescript
import {
  StartDocumentTextDetectionCommand,
  TextractClient,
} from "@aws-sdk/client-textract";

const textract = new TextractClient();

const { JobId } = await textract.send(
  new StartDocumentTextDetectionCommand({
    DocumentLocation: {
      S3Object: { Bucket: documentsBucketName, Name: key },
    },
  }),
);
```

`StartDocumentTextDetection` kicks off an **asynchronous** job. It returns a
`JobId` immediately and processes the document in the background.

# 3. Poll for the result

```typescript
const { JobStatus, Blocks } = await waitForIt(async () => {
  const result = await textract.send(
    new GetDocumentTextDetectionCommand({ JobId }),
  );
  if (result.JobStatus === "IN_PROGRESS") throw new Error("not done yet");
  return result;
}, 40); // ~3 minutes of backoff

const lines = (Blocks ?? [])
  .filter((b) => b.BlockType === "LINE" && b.Text)
  .map((b) => b.Text);

const text = lines.join("\n");
```

# Example

![Invoice](./invoice.png){width=50%}

# Raw text example

![Raw text restult](./raw-text.png)

Visualized on AWS Console

# What gets asserted

On the **Stripe invoice** (structured API):

- Line items match description, quantity and cents
- Currency and collection method
- VAT declaration (tax rate, behavior, amount)
- Issue date and 30-day due date
- Metadata (organization id, service period)

On the **rendered PDF** (OCR'd text):

- Altinn seller / buyer / date requirements
- NOK footer note presence (currency-dependent)
- Currency conversion rate note
- NOK VAT basis and amount for foreign-currency Norwegian sales
- Per-project grouping headers

# CI strategy

OCR tests are **slow** (Textract) and **cost money**, so they run only when a
change can affect invoice rendering:

```yaml
ocr-e2e:
  needs: [stack-prefix, changes, tests]
  if: ${{ needs.changes.outputs.ocr == 'true' }}
  strategy:
    fail-fast: true
    max-parallel: 1
    matrix:
      spec:
        - chargedInvoiceDocument.ocr.spec.ts
        - vatDeclarationNorwegianCustomer.ocr.spec.ts
        - perProjectGrouping.ocr.spec.ts
        # … one job per spec
```

Each spec runs as its **own job** in a `fail-fast`, `max-parallel: 1` matrix, so
the first failure cancels the rest.

# CI pipeline

![CI pipeline](./ci-pipeline.png)

# Artifacts for visual inspection

Every rendered invoice PDF is written to the git-ignored `e2e-invoices/` folder
and uploaded as a CI artifact:

```yaml
- name: Upload the rendered invoice PDF
  uses: actions/upload-artifact@v4
  with:
    name: e2e-invoices-${{ matrix.spec }}
    path: e2e-invoices/
    retention-days: 7
```

A failing assertion includes the full OCR'd text in its message, and the PDF is
a click away.

# Takeaways

- **Test the real thing.** Mocking the third party you integrate with defeats
  the purpose — you learn what it actually does.
- **OCR what you cannot read.** When a rendered document is the contract, read
  it back. Textract reads glyphs, not a broken text layer.
- **Gate expensive tests.** Detect whether a change can affect rendering, and
  only then pay for OCR.
- **Keep the artifacts.** Save the rendered PDFs so a failure is one click from
  a visual diagnosis.
- **Clean up after yourself.** The sandbox is shared; track and tear down what
  you create.

# Thank you

<div class="text-center">

Please share your feedback!

<small>[coderbyheart.com/socials](https://coderbyheart.com/socials)</small>

</div>
