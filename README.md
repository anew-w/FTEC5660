# FTEC5660 Homework 1: Receipt Chain

Build a LangChain pipeline that reads every supermarket receipt in a folder
with the vision-capable DeepSeek Flash model and answers these two questions:

1. How much money did I spend in total for these bills?
2. How much would I have had to pay without the discount?

For this homework, **amount spent** means the final payment after the receipt's
rounding line. **Without the discount** means the sum of the original positive
item prices: add back every promotion, coupon, member, app, packaging-damage,
and percentage discount, but do not add back rounding.

## Student task

Only edit the two functions in `hw1.py` that contain `### YOUR CODE HERE`:

- `build_chain()` creates your LangChain chain.
- `answer_queries()` runs the chain on the receipt images and returns one final
  response for each question.

You may use prompt chaining, routing, parallel calls, reflection, or a
combination. Your final responses should each contain one HKD amount. Do not
hard-code filenames or public answers; grading uses unseen receipt folders.

## Setup and public test

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Put your DeepSeek key after `DEEPSEEK_API_KEY=` in `.env`, then run:

```bash
python3 hw1.py --image-folder public_test
```

The program creates `results.csv` in the current directory. Its columns are
`query`, `model_response`, and `correctness`. The public answers are in
`public_test/ground_truth.json`. The starter intentionally returns the dummy
response `please design your chain to answer these two queries.` so it runs
before you add any API code.

The required model is `deepseek-v4-flash-vision-exp`, the vision-capable
DeepSeek Flash model. JPEG, PNG, GIF, and WebP inputs are accepted by the
homework runner.


## Homework 1 solution

### Chain Architecture

```mermaid
flowchart TD
    A[Receipt Images<br/>public_test/] --> B[image_data_url]
    B --> C[Base64 Data URLs]
    C --> D[chain.batch<br/>max_concurrency=5]

    subgraph build_chain
        E[System Prompt<br/>receipt analysis rules] --> F[ChatPromptTemplate]
        G[Human Message<br/>text + image_url] --> F
        F --> H[ChatDeepSeek<br/>deepseek-v4-flash-vision-exp]
    end

    D --> H
    H --> I[Model Response<br/>JSON or free text]
    I --> J[JSON Parsing]
    J -->|success| K[final_paid<br/>without_discount]
    J -->|fail| L[Regex Fallback<br/>_MONEY_RE]
    L --> K
    K --> M[Decimal Summation]
    M --> N[Return Results]
    N --> O[results.csv]
```

### Solution Description

My solution uses a **single vision-language model chain** with `deepseek-v4-flash-vision-exp` as the backbone. The `build_chain()` function constructs a `ChatPromptTemplate` with a detailed system message that instructs the model to extract two monetary values from each receipt image:

1. **FINAL_PAID**: The amount after the `ROUNDING` adjustment, found on the payment method line (e.g., OCTOPUS, CASH, VISA).
2. **WITHOUT_DISCOUNT**: Computed as `SUBTOTAL + sum(abs(discount_lines))`, excluding the rounding amount.

The system prompt includes an explicit verification step, asking the model to double-check the SUBTOTAL amount and list every discount line before returning. This reduces OCR errors on blurry receipts.

The `answer_queries()` function processes all receipts **in parallel** using `chain.batch()` with `max_concurrency=5`. Each response is parsed via JSON first; if that fails, a regex fallback extracts the first two numeric amounts. All values are accumulated using Python `Decimal` to avoid floating-point precision issues. The final aggregated answers are formatted as `HK$XXXX.XX` strings and returned, which `main()` then writes to `results.csv` alongside automatic correctness checking against `ground_truth.json`.

