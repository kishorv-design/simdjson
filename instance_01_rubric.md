# instance_01 Rubric: One Criterion per Bug (weight 1 each)

Each criterion is verifiable (file + function/class), non-redundant, and identifies exactly one introduced bug. Weights are 1 for all.

---

## B02 — Tape container start layout

**File:** `src/generic/stage2/tape_builder.h`  
**Function/class:** `tape_builder::end_container`

**Criterion:** The start tape element for a container (array or object) must pack the **next tape index in the low 32 bits** and the **saturated element count in the high 24 bits** (i.e. value is `next_tape_index(iter) | (uint64_t(cntsat) << 32)`). Any other packing (e.g. count in low bits, index in high bits) is incorrect.

**Relevant files:** `src/generic/stage2/tape_builder.h`; DOM tape consumers (e.g. `include/simdjson/internal/tape_ref-inl.h`, DOM element iteration) that read this word.

---

## B04 — Surrogate pair low surrogate range

**File:** `src/generic/stage2/stringparsing.h`  
**Function:** `handle_unicode_codepoint` (namespace `stringparsing`)

**Criterion:** A low surrogate (code_point_2 in range 0xDC00..0xDFFF) must be accepted when it follows a high surrogate. The validity check must be `(low_bit >> 10)` (i.e. reject only when low_bit >= 1024). Using `low_bit >= 0x3FF` incorrectly rejects the valid low surrogate at 0xDFFF (low_bit == 0x3FF).

**Relevant files:** `src/generic/stage2/stringparsing.h`; any implementation that includes it (e.g. haswell, arm64, fallback stringparsing_defs).

---

## B06 — Fatal error set

**File:** `include/simdjson/error-inl.h`  
**Function:** `is_fatal(error_code error)`

**Criterion:** `is_fatal()` must return true for exactly: `TAPE_ERROR`, `INCOMPLETE_ARRAY_OR_OBJECT`, `OUT_OF_ORDER_ITERATION`, and `DEPTH_ERROR`. Omitting `DEPTH_ERROR` (or any of these) is incorrect.

**Relevant files:** `include/simdjson/error-inl.h`; call sites that stop on fatal errors (e.g. ondemand document handling).

---

## B09 — Float fast path power bound

**File:** `include/simdjson/generic/numberparsing.h`  
**Function:** `compute_float_64` (namespace `numberparsing`)

**Criterion:** The fast path for exact float conversion must include power in the range `-22 <= power <= 22` (both FLT_EVAL_METHOD branches). Using `power <= 21` instead of `power <= 22` incorrectly excludes power==22 from the fast path and can produce wrong or inconsistent results for numbers like 1e22.

**Relevant files:** `include/simdjson/generic/numberparsing.h`; implementation-specific numberparsing_defs that include it.

---

## B11 — doc_index update in document_stream::next

**File:** `include/simdjson/dom/document_stream-inl.h`  
**Function:** `document_stream::next()`

**Criterion:** When loading a new batch inside the `while (error == EMPTY)` loop, `doc_index` must be set to the start of the first document in that batch: `doc_index = batch_start + parser->implementation->structural_indexes[parser->implementation->next_structural_index]` before calling `stage2_next`. Failing to update `doc_index` there leaves it pointing at the previous batch and makes `iterator::source()` and `current_index()` wrong for documents after the first in each subsequent batch.

**Relevant files:** `include/simdjson/dom/document_stream-inl.h`; `document_stream::iterator::source()` (same file).

---

## B13 — Root structural brace match

**File:** `src/generic/stage2/json_iterator.h`  
**Function:** `json_iterator::walk_document` (template, non-STREAMING branch)

**Criterion:** When the root value is `[`, the last structural character must be `]` (not `}`). The check must be `last_structural() != ']'`. Using `last_structural() != '}'` for the array case incorrectly allows a document that starts with `[` and ends with `}` (mismatched) to pass and can lead to memory corruption or wrong tape.

**Relevant files:** `src/generic/stage2/json_iterator.h`.

---

## B16 — Decimal exponent sign in parse_decimal_after_separator

**File:** `include/simdjson/generic/numberparsing.h`  
**Function:** `parse_decimal_after_separator` (namespace `numberparsing`)

**Criterion:** The decimal exponent (number of digits after the decimal point) must be stored as `first_after_period - p` (a non-positive value when p has advanced past the decimal digits). Using `p - first_after_period` inverts the sign and causes wrong float values for any number with a decimal part.

**Relevant files:** `include/simdjson/generic/numberparsing.h`; callers that use `exponent` in float computation (e.g. `parse_number`).

---

## B18 — stage1_worker thread: has_work vs stage1_thread_error order

**File:** `include/simdjson/dom/document_stream-inl.h`  
**Function/class:** `stage1_worker::start_thread` (lambda run by the worker thread)

**Criterion:** The worker must set `stage1_thread_error` (result of `run_stage1`) before setting `has_work = false` and before calling `cond_var.notify_one()`. Setting `has_work = false` before assigning `stage1_thread_error` allows the main thread to observe that work is done before the error is visible and can lead to reading UNINITIALIZED or stale `stage1_thread_error`.

**Relevant files:** `include/simdjson/dom/document_stream-inl.h`; `stage1_worker::finish()` and `load_from_stage1_thread()` (same file).
