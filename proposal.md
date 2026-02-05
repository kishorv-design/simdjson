Repo Map
- Public API surface: include/simdjson.h, include/simdjson/dom.h, include/simdjson/ondemand.h
- Runtime dispatch and ISA selection: include/simdjson/implementation.h, src/implementation.cpp, include/simdjson/*/implementation.h
- DOM parsing front-end: include/simdjson/dom/parser*.h, include/simdjson/dom/document*.h, include/simdjson/dom/element*.h
- On-demand parsing front-end: include/simdjson/generic/ondemand/* (parser, document, value, iterators)
- Stage 1 (structural indexing + UTF-8 validation): src/generic/stage1/* (json_structural_indexer, json_scanner, json_string_scanner, utf8_validator)
- Stage 2 (tape building + string parsing): src/generic/stage2/* (json_iterator, tape_builder, tape_writer, stringparsing)
- Numeric and atom parsing: include/simdjson/generic/numberparsing.h, include/simdjson/generic/atomparsing.h
- Input buffers and padding: include/simdjson/padded_string*.h, include/simdjson/padded_string_view*.h
- Streaming and multi-document parsing: dom::document_stream, ondemand::document_stream (threaded stage1 worker)
- Concurrency mechanisms: stage1_worker thread (SIMDJSON_THREADS_ENABLED), thread_local ondemand::parser, atomic active implementation pointer
- Data flow (DOM): parse/load -> ensure_capacity -> stage1 structural indexes -> stage2 tape build -> element/object/array accessors
- Data flow (on-demand): iterate -> stage1 structural indexes -> lazy json_iterator traversal -> value/object/array accessors
- I/O boundary: dom::parser::read_file for load/load_many; raw buffers for parse/iterate paths

Bug Candidates

### B01
- ID: B01
- Location: include/simdjson/dom/parser-inl.h, parser::read_file (around L38-L92)
- Bug type: padding omission / memory safety
- Proposed change: allocate the buffer with new char[len] instead of allocate_padded_buffer(len).
- Trigger conditions: load() or load_many() on any file; SIMD reads past the end.
- Expected symptom: sporadic OOB reads, parse errors, or ASan failures.
- Why its hard: failure appears in stage1; depends on allocator and file size.
- Suggested detection: ASan/UBSan runs on load/load_many; fuzz file sizes.
- Exercise value: High
- Stealth: High

### B02
- ID: B02
- Location: include/simdjson/dom/parser-inl.h, parser::read_file (around L84-L88)
- Bug type: error handling gap
- Proposed change: treat short reads as success (only fail when bytes_read == 0).
- Trigger conditions: partial reads on network/virtual filesystems.
- Expected symptom: parse succeeds but silently ignores trailing bytes.
- Why its hard: symptoms appear as missing fields, not I/O errors.
- Suggested detection: integration tests with injected short reads; checksum comparison.
- Exercise value: Medium
- Stealth: High

### B03
- ID: B03
- Location: include/simdjson/dom/parser-inl.h, parser::parse_into_document (around L131-L134)
- Bug type: boundary handling
- Proposed change: advance buf by 3 for BOM but forget to decrement len.
- Trigger conditions: UTF-8 BOM prefixed input.
- Expected symptom: extra bytes read; stage1 failures or corrupted tape.
- Why its hard: only BOM inputs; error surfaces deep in parser.
- Suggested detection: tests with BOM-prefixed JSON.
- Exercise value: Medium
- Stealth: Medium

### B04
- ID: B04
- Location: include/simdjson/dom/parser-inl.h, parser::parse_into_document (around L118-L129)
- Bug type: stale buffer / padding mismatch
- Proposed change: after copying into loaded_bytes, forget to reassign buf to loaded_bytes.
- Trigger conditions: parse from unpadded buffers with realloc_if_needed=true.
- Expected symptom: OOB reads or nondeterministic parse failures.
- Why its hard: copy exists so code looks safe; failure depends on buffer lifetime.
- Suggested detection: ASan with short-lived buffers; fuzz unpadded inputs.
- Exercise value: High
- Stealth: High

### B05
- ID: B05
- Location: include/simdjson/dom/parser-inl.h, parser::ensure_capacity (around L225-L243)
- Bug type: allocation mismatch
- Proposed change: remove the target_document.capacity() check and only resize parser buffers.
- Trigger conditions: large documents or deep strings.
- Expected symptom: tape/string buffer overflow during stage2.
- Why its hard: root cause far from crash site.
- Suggested detection: ASan with large string-heavy JSON.
- Exercise value: High
- Stealth: Medium

### B06
- ID: B06
- Location: include/simdjson/dom/parser-inl.h, parser::set_max_capacity (around L247-L252)
- Bug type: config handling
- Proposed change: allow max_capacity values below MINIMAL_DOCUMENT_CAPACITY without clamping.
- Trigger conditions: callers set small max_capacity values.
- Expected symptom: CAPACITY errors on otherwise valid small documents.
- Why its hard: error looks like input-size issue; config is far upstream.
- Suggested detection: tests that set max_capacity then parse small inputs.
- Exercise value: Medium
- Stealth: Medium

### B07
- ID: B07
- Location: include/simdjson/dom/parser.h, parser::parse(const char*, size_t) (around L237-L238)
- Bug type: incorrect default
- Proposed change: default realloc_if_needed to false for C-string overload.
- Trigger conditions: C strings without SIMDJSON_PADDING.
- Expected symptom: OOB reads or intermittent parse errors.
- Why its hard: looks like a performance tweak; allocator-dependent.
- Suggested detection: ASan on parse(const char*, len) with tight buffers.
- Exercise value: High
- Stealth: High

### B08
- ID: B08
- Location: include/simdjson/dom/document_stream-inl.h, document_stream::run_stage1 (around L275-L281)
- Bug type: boundary handling
- Proposed change: use remaining < batch_size instead of <=.
- Trigger conditions: remaining exactly equals batch_size.
- Expected symptom: last batch treated as partial, dropping the final document.
- Why its hard: only specific input lengths trigger it.
- Suggested detection: tests where len is exactly batch_size.
- Exercise value: Medium
- Stealth: High

### B09
- ID: B09
- Location: include/simdjson/dom/document_stream-inl.h, document_stream::next_batch_start (around L271-L273)
- Bug type: off-by-one
- Proposed change: compute using structural_indexes[n+1] instead of [n].
- Trigger conditions: streaming_partial/final boundaries.
- Expected symptom: skipped or repeated bytes between batches.
- Why its hard: depends on stage1 metadata; failures appear downstream.
- Suggested detection: property tests with varied batch_size and input lengths.
- Exercise value: High
- Stealth: Medium

### B10
- ID: B10
- Location: include/simdjson/dom/document_stream-inl.h, stage1_worker::finish (around L16-L23)
- Bug type: concurrency / spurious wakeup
- Proposed change: call cond_var.wait(lock) without a predicate.
- Trigger conditions: spurious wakeups in threaded parse_many.
- Expected symptom: stage2 runs before stage1 completes; random parse errors.
- Why its hard: timing-dependent and intermittent.
- Suggested detection: ThreadSanitizer + stress tests with small batches.
- Exercise value: High
- Stealth: High

### B11
- ID: B11
- Location: include/simdjson/dom/document_stream-inl.h, iterator::source (around L218-L226)
- Bug type: slice boundary error
- Proposed change: use next_structural_index instead of next_structural_index - 1 for arrays/objects.
- Trigger conditions: back-to-back documents in a stream.
- Expected symptom: source() includes first byte of next document.
- Why its hard: parsing still succeeds; only debug tooling is wrong.
- Suggested detection: tests that compare source() to original input slices.
- Exercise value: Medium
- Stealth: Medium

### B12
- ID: B12
- Location: include/simdjson/dom/element.h, element::at_pointer (around L380-L402)
- Bug type: API contract drift
- Proposed change: unescape pointer segments before matching keys in element::at_pointer only.
- Trigger conditions: keys containing escapes and mixed API usage.
- Expected symptom: element::at_pointer works but object/array at_pointer fails.
- Why its hard: cross-API inconsistency; looks like user error.
- Suggested detection: tests comparing element/object/array pointer behavior.
- Exercise value: Medium
- Stealth: High

### B13
- ID: B13
- Location: include/simdjson/dom/element.h, element::operator< (around L479-L484)
- Bug type: ordering invariant violation
- Proposed change: compare tape pointer addresses without validating same document base.
- Trigger conditions: comparisons between elements from different documents.
- Expected symptom: unstable ordering in std::set/map; occasional corruption.
- Why its hard: requires cross-document usage; symptoms are nondeterministic.
- Suggested detection: tests that insert elements from multiple documents into ordered containers.
- Exercise value: Medium
- Stealth: High

### B14
- ID: B14
- Location: include/simdjson/dom/object.h, object::at_key_case_insensitive (around L227-L237)
- Bug type: logic regression
- Proposed change: use key_equals instead of key_equals_case_insensitive.
- Trigger conditions: case-insensitive lookups with mixed-case keys.
- Expected symptom: NO_SUCH_FIELD for valid keys.
- Why its hard: only affects one API; looks like data issue.
- Suggested detection: unit tests with mixed-case key lookups.
- Exercise value: Low
- Stealth: Medium

### B15
- ID: B15
- Location: include/simdjson/dom/array.h, array::size (around L79-L83)
- Bug type: incorrect accounting
- Proposed change: return number_of_slots() instead of element count.
- Trigger conditions: arrays with nested arrays/objects.
- Expected symptom: inflated size; downstream allocations wrong.
- Why its hard: parsing still succeeds; size-only consumers fail.
- Suggested detection: tests comparing size() to iteration count.
- Exercise value: Medium
- Stealth: Medium

### B16
- ID: B16
- Location: include/simdjson/dom/document-inl.h, document::root (around L40-L60)
- Bug type: state leak
- Proposed change: return cached root element without checking parser.valid after new parse.
- Trigger conditions: reuse parser with a failed parse followed by a successful one.
- Expected symptom: old root element returned from previous document.
- Why its hard: appears as stale data, not a parse error.
- Suggested detection: tests that parse invalid then valid JSON on same parser.
- Exercise value: High
- Stealth: High

### B17
- ID: B17
- Location: include/simdjson/generic/ondemand/parser-inl.h, parser::iterate(std::string&) (around L99-L101)
- Bug type: padding assumption
- Proposed change: call iterate(padded_string_view(json, json.size())) without extending capacity.
- Trigger conditions: std::string with capacity == size.
- Expected symptom: OOB reads or sporadic failures.
- Why its hard: allocator-dependent; only some strings trigger it.
- Suggested detection: ASan with strings where capacity==size.
- Exercise value: High
- Stealth: High

### B18
- ID: B18
- Location: include/simdjson/generic/ondemand/parser-inl.h, parser::iterate_many(std::string&) (around L158-L162)
- Bug type: padding check bypass
- Proposed change: forward to pointer overload without padding validation.
- Trigger conditions: under-padded std::string inputs in iterate_many.
- Expected symptom: stage1 reads past end; intermittent parse errors.
- Why its hard: only one overload is affected.
- Suggested detection: tests comparing iterate_many(std::string) vs padded_string_view.
- Exercise value: Medium
- Stealth: Medium

### B19
- ID: B19
- Location: include/simdjson/generic/ondemand/parser-inl.h, parser::iterate_many (around L136-L144)
- Bug type: mode-specific boundary handling
- Proposed change: ignore allow_comma_separated special-case that forces batch_size >= len.
- Trigger conditions: allow_comma_separated=true with a large document.
- Expected symptom: parsing fails mid-document only in comma-separated mode.
- Why its hard: mode-specific; not covered by typical tests.
- Suggested detection: iterate_many tests with allow_comma_separated and large docs.
- Exercise value: Medium
- Stealth: High

### B20
- ID: B20
- Location: include/simdjson/generic/ondemand/parser-inl.h, parser::get_parser (around L198-L214)
- Bug type: concurrency / shared state
- Proposed change: use a single static parser instead of thread_local.
- Trigger conditions: concurrent parsing in multiple threads.
- Expected symptom: data races, corrupted string buffers, random failures.
- Why its hard: nondeterministic timing; hard to reproduce.
- Suggested detection: ThreadSanitizer with parallel parse workloads.
- Exercise value: High
- Stealth: High

### B21
- ID: B21
- Location: include/simdjson/generic/ondemand/parser-inl.h, release_parser (around L202-L208)
- Bug type: use-after-free
- Proposed change: delete parser but keep thread_local pointer non-null.
- Trigger conditions: get_parser called after release_parser.
- Expected symptom: UAF crashes or corrupted state.
- Why its hard: release_parser is rarely used; errors are delayed.
- Suggested detection: tests calling release_parser then parse again.
- Exercise value: Medium
- Stealth: High

### B22
- ID: B22
- Location: include/simdjson/generic/ondemand/parser.h, parser::set_max_capacity (around L274-L278)
- Bug type: incorrect clamp
- Proposed change: invert the MINIMAL_DOCUMENT_CAPACITY clamp logic.
- Trigger conditions: caller sets max_capacity larger than minimal.
- Expected symptom: max_capacity silently shrinks; unexpected CAPACITY errors.
- Why its hard: appears as capacity limitation elsewhere.
- Suggested detection: tests that set max_capacity and parse larger inputs.
- Exercise value: Medium
- Stealth: Medium

### B23
- ID: B23
- Location: include/simdjson/generic/ondemand/document_stream-inl.h, document_stream::next_document (around L319-L321)
- Bug type: state leak / buffer overflow
- Proposed change: remove reset of doc.iter._string_buf_loc.
- Trigger conditions: streaming multiple string-heavy documents.
- Expected symptom: string corruption across documents; occasional buffer overflow.
- Why its hard: depends on data size and order; errors are non-local.
- Suggested detection: stream tests with many strings; ASan/UBSan.
- Exercise value: High
- Stealth: High

### B24
- ID: B24
- Location: include/simdjson/generic/ondemand/document_stream-inl.h, document_stream::start (around L222-L225)
- Bug type: mode flag misconfiguration
- Proposed change: omit doc.iter._streaming = true.
- Trigger conditions: iterate_many on concatenated documents.
- Expected symptom: iterator treats stream as single doc; later docs missing.
- Why its hard: only in streaming mode; first document parses fine.
- Suggested detection: tests for multiple concatenated documents.
- Exercise value: Medium
- Stealth: High

### B25
- ID: B25
- Location: include/simdjson/generic/ondemand/document_stream-inl.h, document_stream::next (around L296-L299)
- Bug type: stale pointer
- Proposed change: skip re-anchoring json_iterator after loading a new batch.
- Trigger conditions: streams that span multiple batches.
- Expected symptom: values read from previous batch; random parse errors.
- Why its hard: depends on batch boundaries and input length.
- Suggested detection: tests with small batch_size to force multiple batches.
- Exercise value: High
- Stealth: High

### B26
- ID: B26
- Location: include/simdjson/generic/ondemand/document_stream-inl.h, document_stream::run_stage1 (around L328-L336)
- Bug type: boundary handling
- Proposed change: treat remaining == batch_size as streaming_partial.
- Trigger conditions: remaining exactly batch_size.
- Expected symptom: final document dropped or truncated.
- Why its hard: input-length specific.
- Suggested detection: tests where len equals batch_size.
- Exercise value: Medium
- Stealth: High

### B27
- ID: B27
- Location: include/simdjson/generic/ondemand/document.h, document::at_pointer (around L662-L701)
- Bug type: stateful ordering
- Proposed change: remove the implicit rewind between at_pointer calls.
- Trigger conditions: multiple at_pointer calls on the same document.
- Expected symptom: NO_SUCH_FIELD or wrong values depending on prior access.
- Why its hard: order-dependent and non-obvious.
- Suggested detection: tests with multiple pointer lookups in different orders.
- Exercise value: Medium
- Stealth: High

### B28
- ID: B28
- Location: include/simdjson/generic/ondemand/document.h, document::count_elements (around L356-L368)
- Bug type: state leak
- Proposed change: omit the rewind after counting elements.
- Trigger conditions: count_elements() followed by iteration.
- Expected symptom: iteration returns empty or OUT_OF_ORDER in debug.
- Why its hard: only when count_elements is used.
- Suggested detection: unit tests that count then iterate.
- Exercise value: Medium
- Stealth: Medium

### B29
- ID: B29
- Location: include/simdjson/generic/ondemand/document.h, document::count_fields (around L369-L383)
- Bug type: state leak
- Proposed change: omit the rewind after counting fields.
- Trigger conditions: count_fields() followed by find_field/iteration.
- Expected symptom: missing early fields.
- Why its hard: appears as user misuse.
- Suggested detection: tests that count_fields then access fields.
- Exercise value: Medium
- Stealth: Medium

### B30
- ID: B30
- Location: include/simdjson/generic/ondemand/document.h, document::get_number_type (around L546-L569)
- Bug type: misclassification
- Proposed change: classify using is_integer() without validating token terminator.
- Trigger conditions: exponentials like "1e3".
- Expected symptom: get_number_type reports integer; get_number fails later.
- Why its hard: mismatch only for some numeric formats.
- Suggested detection: tests comparing get_number_type vs get_number on exponentials.
- Exercise value: Medium
- Stealth: High

### B31
- ID: B31
- Location: include/simdjson/generic/ondemand/value.h, value::at_pointer (around L621-L663)
- Bug type: unintended rewind
- Proposed change: delegate to document::at_pointer, which rewinds the document.
- Trigger conditions: at_pointer called mid-iteration.
- Expected symptom: subsequent iteration invalidated; OUT_OF_ORDER errors.
- Why its hard: only shows when mixing pointer access and iteration.
- Suggested detection: tests that call at_pointer then continue iterating.
- Exercise value: High
- Stealth: High

### B32
- ID: B32
- Location: include/simdjson/generic/ondemand/object.h, object::find_field_unordered (around L117-L119)
- Bug type: search regression
- Proposed change: remove fallback scan from the beginning when key is not found.
- Trigger conditions: keys accessed out of order.
- Expected symptom: NO_SUCH_FIELD for valid keys.
- Why its hard: looks like user ordering mistake.
- Suggested detection: out-of-order key access tests using operator[].
- Exercise value: Medium
- Stealth: Medium

### B33
- ID: B33
- Location: src/generic/stage1/json_structural_indexer.h, trim_partial_utf8 (around L155-L173)
- Bug type: UTF-8 boundary handling
- Proposed change: treat any byte >= 0x80 as a leading byte in short buffers.
- Trigger conditions: multibyte UTF-8 split across batch boundary.
- Expected symptom: valid bytes trimmed; UTF8_ERROR on valid input.
- Why its hard: only at specific batch boundaries.
- Suggested detection: streaming tests with multibyte chars crossing batches.
- Exercise value: Medium
- Stealth: High

### B34
- ID: B34
- Location: src/generic/stage1/json_structural_indexer.h, finish (around L261-L263)
- Bug type: validation omission
- Proposed change: drop unescaped_chars_error check.
- Trigger conditions: control characters inside strings.
- Expected symptom: invalid JSON accepted; downstream parsing may fail.
- Why its hard: error manifests later; only on invalid inputs.
- Suggested detection: tests with unescaped control chars.
- Exercise value: Medium
- Stealth: Medium

### B35
- ID: B35
- Location: src/generic/stage1/json_structural_indexer.h, finish streaming_final (around L331-L338)
- Bug type: metadata corruption
- Proposed change: swap structural_indexes[n] and structural_indexes[n+1].
- Trigger conditions: streaming_final with trailing garbage.
- Expected symptom: truncated_bytes incorrect; batch boundaries wrong.
- Why its hard: only in streaming_final mode.
- Suggested detection: tests that verify truncated_bytes on truncated streams.
- Exercise value: High
- Stealth: High

### B36
- ID: B36
- Location: src/generic/stage1/json_structural_indexer.h, finish streaming_partial (around L295-L301)
- Bug type: boundary handling
- Proposed change: do not decrement n_structural_indexes when have_unclosed_string.
- Trigger conditions: strings split across batch boundary.
- Expected symptom: STRING_ERROR in stage2 on valid inputs.
- Why its hard: only for long strings at batch edges.
- Suggested detection: streaming tests with long strings.
- Exercise value: Medium
- Stealth: High

### B37
- ID: B37
- Location: src/generic/stage1/json_structural_indexer.h, index (around L195-L197)
- Bug type: incorrect empty-input handling
- Proposed change: return SUCCESS when len == 0.
- Trigger conditions: empty or all-whitespace inputs.
- Expected symptom: stage2 runs with no structurals; unexpected errors.
- Why its hard: only empty inputs; failures are downstream.
- Suggested detection: unit tests for empty input.
- Exercise value: Low
- Stealth: Medium

### B38
- ID: B38
- Location: src/generic/stage1/json_scanner.h, json_scanner::next (around L147-L156)
- Bug type: misclassification
- Proposed change: compute nonquote_scalar without masking out quotes.
- Trigger conditions: adjacent string + scalar tokens without whitespace.
- Expected symptom: structural indexes misaligned; parse failures.
- Why its hard: only certain token adjacency patterns.
- Suggested detection: tests with "\"x\"true" style inputs.
- Exercise value: Medium
- Stealth: High

### B39
- ID: B39
- Location: src/generic/stage1/json_scanner.h, json_scanner::finish (around L159-L161)
- Bug type: validation omission
- Proposed change: always return SUCCESS even on UNCLOSED_STRING.
- Trigger conditions: unterminated strings at EOF.
- Expected symptom: stage2 errors or corrupted tape.
- Why its hard: error is delayed; only malformed inputs.
- Suggested detection: tests with unterminated strings.
- Exercise value: Low
- Stealth: Medium

### B40
- ID: B40
- Location: src/generic/stage1/json_string_scanner.h, json_string_scanner::next (around L72-L79)
- Bug type: cross-block state bug
- Proposed change: update prev_in_string using in_string >> 62 instead of >> 63.
- Trigger conditions: strings ending exactly on 64-byte boundaries.
- Expected symptom: stage1 believes it is inside a string; parse fails later.
- Why its hard: alignment-specific and intermittent.
- Suggested detection: tests with strings sized to 64-byte boundaries.
- Exercise value: High
- Stealth: High

### B41
- ID: B41
- Location: src/generic/stage1/utf8_validator.h, generic_validate_utf8 (around L27-L31)
- Bug type: uninitialized memory use
- Proposed change: remove zero-initialization of remainder block.
- Trigger conditions: inputs not multiple of 64 bytes.
- Expected symptom: sporadic UTF8_ERROR or acceptance depending on stack content.
- Why its hard: nondeterministic and environment-dependent.
- Suggested detection: MSan/ASan on UTF-8 validation.
- Exercise value: Medium
- Stealth: High

### B42
- ID: B42
- Location: src/generic/stage1/find_next_document_index.h, find_next_document_index (around L63-L70)
- Bug type: boundary detection error
- Proposed change: treat ':' as a boundary instead of excluding it.
- Trigger conditions: objects near batch boundaries.
- Expected symptom: premature document boundary; truncated docs.
- Why its hard: depends on batch size and object layout.
- Suggested detection: streaming tests with objects around batch edges.
- Exercise value: Medium
- Stealth: High

### B43
- ID: B43
- Location: src/generic/stage1/json_minifier.h, json_minifier::finish (around L42-L46)
- Bug type: error handling gap
- Proposed change: ignore scanner.finish error and always return SUCCESS.
- Trigger conditions: malformed JSON passed to minify.
- Expected symptom: minify returns output for invalid JSON.
- Why its hard: only affects minify; errors are silent.
- Suggested detection: tests that minify malformed JSON and assert error.
- Exercise value: Low
- Stealth: Medium

### B44
- ID: B44
- Location: src/generic/stage2/json_iterator.h, json_iterator::walk_document (around L135-L141)
- Bug type: validation omission
- Proposed change: remove the root closing check (last_structural) for non-streaming.
- Trigger conditions: unclosed root array/object.
- Expected symptom: tape corruption; DOM traversal failures.
- Why its hard: failure appears in later access, not at parse boundary.
- Suggested detection: tests with unclosed root containers.
- Exercise value: High
- Stealth: Medium

### B45
- ID: B45
- Location: src/generic/stage2/json_iterator.h, json_iterator::walk_document (around L232-L237)
- Bug type: trailing content acceptance
- Proposed change: allow extra tokens by changing != to >.
- Trigger conditions: valid JSON followed by garbage.
- Expected symptom: invalid inputs parse as valid.
- Why its hard: silent acceptance; only visible in validation workflows.
- Suggested detection: tests appending garbage after valid JSON.
- Exercise value: Medium
- Stealth: Medium

### B46
- ID: B46
- Location: src/generic/stage2/tape_builder.h, visit_root_number (around L178-L197)
- Bug type: terminator handling
- Proposed change: pad the copied buffer with '\0' instead of spaces.
- Trigger conditions: top-level numeric documents.
- Expected symptom: NUMBER_ERROR for valid numbers.
- Why its hard: only root primitives; nested numbers still work.
- Suggested detection: tests for parsing single-number documents.
- Exercise value: Medium
- Stealth: High

### B47
- ID: B47
- Location: src/generic/stage2/tape_builder.h, visit_string (around L157-L166)
- Bug type: policy inconsistency
- Proposed change: enable allow_replacement for object keys only.
- Trigger conditions: invalid Unicode in keys.
- Expected symptom: keys silently normalized while values still error.
- Why its hard: inconsistent behavior across key/value paths.
- Suggested detection: tests with invalid Unicode in keys and values.
- Exercise value: Medium
- Stealth: High

### B48
- ID: B48
- Location: src/generic/stage2/tape_builder.h, start_container (around L255-L258)
- Bug type: structural corruption
- Proposed change: remove tape.skip so start element is written immediately.
- Trigger conditions: nested arrays/objects.
- Expected symptom: broken tape offsets; DOM traversal returns nonsense.
- Why its hard: parsing completes; failures occur on access.
- Suggested detection: tests with nested objects and arrays.
- Exercise value: High
- Stealth: Medium

### B49
- ID: B49
- Location: src/generic/stage2/tape_builder.h, increment_count (around L150-L152)
- Bug type: counting regression
- Proposed change: increment counts only when visiting keys (not values).
- Trigger conditions: arrays/objects of any size.
- Expected symptom: size() undercounts; downstream sizing wrong.
- Why its hard: parsing succeeds; only size-dependent logic fails.
- Suggested detection: tests comparing size() to iteration count.
- Exercise value: Medium
- Stealth: Medium

### B50
- ID: B50
- Location: src/generic/stage2/tape_writer.h, tape_writer::append_u64 (around L72-L76)
- Bug type: tape layout corruption
- Proposed change: use append2 (double-slot) for UINT64 like INT64.
- Trigger conditions: large unsigned integers in DOM.
- Expected symptom: tape misalignment; later elements read incorrectly.
- Why its hard: manifests as downstream traversal errors.
- Suggested detection: tests with mixed UINT64/other values and traversal.
- Exercise value: High
- Stealth: Medium

### B51
- ID: B51
- Location: src/generic/stage2/json_iterator.h, json_iterator::advance (around L62-L70)
- Bug type: off-by-one
- Proposed change: pre-increment next_structural before dereferencing.
- Trigger conditions: any document with multiple tokens.
- Expected symptom: first token of a value is skipped; parse errors.
- Why its hard: errors look like malformed JSON.
- Suggested detection: unit tests for minimal JSON tokens.
- Exercise value: Medium
- Stealth: Medium

### B52
- ID: B52
- Location: src/generic/stage2/json_iterator.h, json_iterator::remaining_len (around L72-L74)
- Bug type: length miscalculation
- Proposed change: compute remaining_len from *next_structural instead of *(next_structural-1).
- Trigger conditions: root primitive parsing that uses remaining_len.
- Expected symptom: insufficient padding copy; parse_number fails.
- Why its hard: only root primitives affected.
- Suggested detection: tests for root primitives with tight buffers.
- Exercise value: Medium
- Stealth: High

### B53
- ID: B53
- Location: src/generic/stage2/stringparsing.h, handle_unicode_codepoint (around L90-L95)
- Bug type: incorrect error handling
- Proposed change: return substitution codepoint even when allow_replacement=false for low surrogates.
- Trigger conditions: isolated low surrogate sequences.
- Expected symptom: invalid JSON accepted silently.
- Why its hard: only invalid Unicode; looks like data issue.
- Suggested detection: tests with isolated low surrogates.
- Exercise value: Medium
- Stealth: High

### B54
- ID: B54
- Location: src/generic/stage2/stringparsing.h, parse_string (around L179-L182)
- Bug type: validation gap
- Proposed change: treat unknown escapes as literal characters.
- Trigger conditions: strings with invalid escapes (e.g., "\v").
- Expected symptom: invalid JSON accepted; values differ from strict behavior.
- Why its hard: silent acceptance of invalid input.
- Suggested detection: tests with invalid escape sequences.
- Exercise value: Medium
- Stealth: Medium

### B55
- ID: B55
- Location: include/simdjson/generic/numberparsing.h, parse_number (around L600-L605)
- Bug type: spec violation
- Proposed change: remove leading-zero check for integers.
- Trigger conditions: numbers like 01 or -01.
- Expected symptom: invalid JSON accepted.
- Why its hard: only invalid inputs; acceptance is silent.
- Suggested detection: tests with leading-zero numbers.
- Exercise value: Low
- Stealth: Medium

### B56
- ID: B56
- Location: include/simdjson/generic/numberparsing.h, parse_exponent (around L427-L441)
- Bug type: overflow handling regression
- Proposed change: stop truncating 19+ digit exponents.
- Trigger conditions: huge exponent strings.
- Expected symptom: exponent overflow, wrong results or crashes.
- Why its hard: only extreme inputs.
- Suggested detection: tests with very large exponents.
- Exercise value: Medium
- Stealth: High

### B57
- ID: B57
- Location: include/simdjson/generic/numberparsing.h, write_float/parse_number (around L525-L529, L659-L663)
- Bug type: numeric semantics regression
- Proposed change: drop special handling for negative zero.
- Trigger conditions: inputs like "-0" or "-0e-999".
- Expected symptom: -0 becomes +0, breaking sign-sensitive consumers.
- Why its hard: edge-case numeric behavior; not covered by most tests.
- Suggested detection: tests for negative zero parsing.
- Exercise value: Low
- Stealth: Medium

### B58
- ID: B58
- Location: include/simdjson/generic/numberparsing.h, parse_double_in_string (around L1296-L1299)
- Bug type: validation gap
- Proposed change: allow whitespace before closing quote.
- Trigger conditions: numeric strings like "1 " passed to get_double_in_string.
- Expected symptom: invalid numeric strings accepted silently.
- Why its hard: only affects *_in_string APIs.
- Suggested detection: tests with trailing whitespace inside numeric strings.
- Exercise value: Low
- Stealth: Medium

### B59
- ID: B59
- Location: include/simdjson/generic/numberparsing.h, integer_string_finisher table (around L688-L707)
- Bug type: whitespace handling regression
- Proposed change: mark '\n' as NUMBER_ERROR instead of SUCCESS.
- Trigger conditions: numbers followed by newline (common in streams).
- Expected symptom: NUMBER_ERROR on valid inputs.
- Why its hard: appears only with certain whitespace.
- Suggested detection: tests for numbers followed by various whitespace.
- Exercise value: Medium
- Stealth: Medium

### B60
- ID: B60
- Location: include/simdjson/generic/numberparsing.h, get_number_type (around L1118-L1147)
- Bug type: boundary misclassification
- Proposed change: compare 18 bytes instead of 19 against the signed boundary string.
- Trigger conditions: values near 9223372036854775808.
- Expected symptom: signed vs unsigned classification flips at the boundary.
- Why its hard: only boundary values; subtle downstream effects.
- Suggested detection: tests around signed/unsigned boundary values.
- Exercise value: Medium
- Stealth: High

### B61
- ID: B61
- Location: include/simdjson/generic/atomparsing.h, is_valid_true_atom (around L37-L38)
- Bug type: terminator check error
- Proposed change: validate terminator at src[3] instead of src[4].
- Trigger conditions: tokens adjacent to other characters.
- Expected symptom: invalid "truX" accepted or valid "trueX" rejected.
- Why its hard: failures surface as generic T_ATOM_ERROR elsewhere.
- Suggested detection: tests with true followed by non-whitespace.
- Exercise value: Low
- Stealth: Medium

### B62
- ID: B62
- Location: include/simdjson/generic/atomparsing.h, is_valid_false_atom (around L49-L50)
- Bug type: off-by-one in atom match
- Proposed change: compare src+2 with "lse" instead of src+1 with "alse".
- Trigger conditions: false tokens in any JSON.
- Expected symptom: false parsing fails or malformed tokens accepted.
- Why its hard: looks like corrupted JSON; error reports are generic.
- Suggested detection: unit tests for false parsing in DOM and on-demand.
- Exercise value: Low
- Stealth: Medium

### B63
- ID: B63
- Location: include/simdjson/padded_string-inl.h, allocate_padded_buffer (around L22-L42)
- Bug type: conditional padding omission
- Proposed change: for length < SIMDJSON_PADDING, allocate only length.
- Trigger conditions: small documents (common).
- Expected symptom: OOB reads on small inputs; intermittent crashes.
- Why its hard: only small inputs; failures appear in stage1.
- Suggested detection: ASan with tiny JSON inputs across entrypoints.
- Exercise value: High
- Stealth: High

### B64
- ID: B64
- Location: src/implementation.cpp, available_implementation_list::detect_best_supported (around L286-L292)
- Bug type: capability check error
- Proposed change: accept any overlap (supported & required) != 0 instead of full set.
- Trigger conditions: CPUs missing some required SIMD features.
- Expected symptom: illegal instruction crashes on unsupported CPUs.
- Why its hard: hardware-dependent; fails only on certain machines.
- Suggested detection: CI on older CPUs or emulators; runtime feature tests.
- Exercise value: High
- Stealth: High

Top 10 Recommended Set
- B01: Padding omission in read_file yields realistic, intermittent memory faults.
- B05: Capacity mismatch between parser and document causes hard-to-trace corruption.
- B08: Boundary condition in run_stage1 drops final documents in streams.
- B10: Concurrency bug in stage1_worker creates nondeterministic failures.
- B20: Shared ondemand parser across threads creates high-value race condition exercise.
- B23: Missing string buffer reset causes subtle cross-document data corruption.
- B35: streaming_final index swap breaks truncated_bytes and batch boundaries.
- B40: Cross-block string state bug is alignment-dependent and subtle.
- B45: Accepting trailing garbage changes validation semantics without obvious crashes.
- B63: Missing padding for small inputs produces elusive OOB reads.
