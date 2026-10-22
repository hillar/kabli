# Requirements for One Agent Turn Loop 

Scope: behavior, not code. The word **must** shows a mandatory rule. The words **do not** show a prohibition. 

## 1. Terms

| Term | Meaning |
|---|---|
| turn | one run of the loop, from operator text to one stop value |
| step | one model request plus the calls it returns |
| call | one request from the model to use a tool |
| signature | one number that identifies a call: tool name plus arguments |
| digest | one value of at least 128 bits that identifies one raw output text |
| ledger | the record that counts repeated calls |
| refusal | a call that the program blocked before dispatch |
| dispatch | the moment the program hands a call to a tool |
| pure read | a tool that cannot change the workspace |
| opaque tool | a tool the program cannot see into |
| raw result | the exact output of a tool |
| masked result | a result in which every known secret is masked |
| history | the message list sent to the model |
| pairing | exactly one result for each advertised call |
| cap | the largest allowed size of a tool result |
| splice | a middle cut with a marker that names the loss |
| spill | a file that keeps the masked full output of a cut result |
| secret | a value that must be masked |
| wake | the mechanism that ends a blocked wait |
| epoch | the span between two messages typed by the operator |
| detector | the mechanism that watches for repetition |
| domain | one text class of a response: thinking text or reply text |
| window | one passage of N words |
| cluster | the run of occurrences of one key, in which no gap is larger than the larger trigger span |
| capacity | the number of words the detector tracker covers |
| strike | one detection counted inside one epoch |
| length cut | a reply that the output limit ended |
| detector cut | a stream that the detector aborted |
| finalization | the one extra request after a budget or loop stop (15) |
| real text | visible reply text that is not only whitespace; thinking text is not real text |

1.2 Technical verbs: hash, parse, mask, truncate, allocate, free, stream, splice, spill, poll, evict.

## 2. Stop values

2.1 Each turn that ends normally must end with exactly one stop value: end_turn, max_tokens, cancelled, step_limit, repeat_loop, stream_loop, no_progress, reported, no_report, checks_failed.
2.2 Each stop value must keep one stable name. Do not change the names.
2.3 The stop value is immutable. The loop must assign it at the moment its condition occurs. Finalization, logging, and cleanup must not change it. **Precedence:**
- (1) An assigned stop value is immutable.
- (2) Finalization assigns no stop value.
- (3) Otherwise the first applicable terminal condition wins.
- (4) A nonterminal requirement (a check refusal, a report reminder, a continue) runs before a normal completion value is assigned.

2.4 Each stop value must come from the condition in this table only:

| Stop value | Condition |
|---|---|
| end_turn | A reply without calls ends the turn, and no length cut is unrecovered (4.5), and no exec-mode requirement of 8.2 or 8.3 is open |
| max_tokens | A length cut is unrecovered, or the last reply was a length cut (4.5) |
| cancelled | The cancel flag is set (5.4), or a transport failure happens while the flag is set (6.2) |
| step_limit | The step count goes over the limit (4.2) |
| repeat_loop | Refusals reach the stop limit (11.9) |
| stream_loop | The second strike in one epoch (13.13) |
| no_progress | A report is delivered under forced finalization (12.4) |
| reported | Outside finalization, a valid report passes the checks (10.2) |
| no_report | Outside finalization, three invalid reports, or a second text ending without a report (8.3, 10.2) |
| checks_failed | Outside finalization, three check refusals (8.2), or a check refusal of a report (10.2) |

2.5 A transport failure while the flag is clear is not a stop value. The turn must end with one error result to the caller. The program must flush the log (5.5). The loop must not push the failed reply to history.

## 3. Turn start

3.1 The loop must clear the report state of the last turn.
3.2 The loop must push the operator text to history before the first request.
3.3 The loop must read all secrets once per turn, not once per call.
3.4 The loop must record the contract baseline before the first request.
3.5 The loop must compact the history before the first request.

## 4. Step control

4.1 Each model request must charge one step. Refused calls, length cuts, and detector cuts also charge one step. The finalization request does not charge a step (15.6).
4.2 When the step count goes over the limit, the loop must assign step_limit, send one limit note, and run finalization.
4.3 Each auto-continue must charge the continue budget. An uncharged continue is an unbounded billing loop.
4.4 When the continue budget is used up, the loop must mark the length cut as unrecovered. A detector cut (13) is not an unrecovered cut.
4.5 The final stop must be max_tokens when a length cut was unrecovered or the last reply was a length cut. In all other cases it must be end_turn.

## 5. Cancel and interrupt

5.1 The program must keep one cancel flag. Operator input, the interrupt key, and the terminate signal must set it.
5.2 Every blocking wait must include one wake. A flag alone does not wake a blocked wait. Do not use a bare blocking read.
5.3 The loop must check the flag at the top of each step, before each call dispatch, and at each stream chunk.
5.4 On cancel, the loop must pair every open call with one result ("cancelled by user before execution") and assign cancelled.
5.5 The program must flush the session log on every stop path.
5.6 The second interrupt must end the process at once.
5.7 On a terminate signal, the program must flush the log and exit. Do not wait for the model.
5.8 When cancel races with a result, the first of the two to arrive must be the one result of the call. The loop must drop the later one and log it. The call must still get exactly one start event and one finish event.

## 6. Request and reply

6.1 The loop must send one request per step, stream text tokens to the display as they arrive, and pass each chunk through the detector (13).
6.2 A transport failure while the flag is set must assign cancelled. Otherwise 2.5 applies.
6.3 The loop must record the truncated flag of each reply and add the usage data to the totals. For an aborted stream, see 13.12.
6.4 Before history, the loop must make every call's arguments wire-safe:
- Valid JSON stays byte-identical.
- Arguments with raw control characters stay in the repaired form.
- A reply that a length cut or a detector cut ended gets the marker value.
- Complete but broken arguments stay as they are.

6.5 The loop must push each reply to history once and log it once.

## 7. History invariants

7.1 History must be append-only. Do not change bytes that were already sent.
7.2 Every advertised call must get exactly one result before any later non-tool message. Strict servers reject other histories.
7.3 Local flags must not go into the wire data.

## 8. Reply without calls

8.1 A reply cut by the output limit, with budget left, must produce one note, one continue instruction, and the next step.
8.2 In exec mode, without a report schema, with checks, and not cut by the output limit: a text ending must pass the check refusal. Three refusals must assign checks_failed. Each refusal must send one note and continue.
8.3 In exec mode, with a report schema, and not cut by the output limit: the loop must send one report reminder. A second text ending must assign no_report.
8.4 A text ending in exec mode with an open requirement of 8.2 or 8.3 is not a terminal event. It must not assign end_turn.

## 9. Reply with calls, cut by the output limit

9.1 Do not execute a cut call (length cut or detector cut). Its arguments can be incomplete.
9.2 The loop must pair every call of a cut reply with one "not executed" result.
9.3 After a length cut with budget left, the loop must continue. Without budget, the loop must stop as in 4.5.

## 10. Call iteration

For each call, in order:

10.1 **Announce.** Send exactly one start event per call. Show masked arguments. When the arguments do not parse, show the raw text with every known secret masked. The log must follow the same rule.
10.2 **Report first.** A report call with a schema must be handled before the ledger, the policy, and the dispatch.
- Validate it against the schema, and run the checks on each valid report.
- Outside finalization, a valid report that passes the checks must assign reported.
- Mask and record the accepted value.
- At most one report runs per message.
- Outside finalization, three invalid reports must assign no_report.
- Outside finalization, a check refusal must assign checks_failed.
- Pair the remaining calls with "not executed".
- **Finalization exception:** a report during finalization must not assign reported, no_report, or checks_failed. It must record the report and the check results and leave the stop value unchanged (15.4).

10.3 **Charge before dispatch.** The ledger must charge before the tool runs. A repeat must not reach the tool.
10.4 **Refusal message.** A refusal must produce one paired result. The message must state the true per-command count and name the call by its masked arguments. It must claim nothing about results and name one remedy. For shell, it must name the fix tools and state that shell state does not persist between calls.
10.5 In forced finalization, the loop must refuse a call without charging the ledger. A refused-by-policy call of this kind is not a repeat.
10.6 **Three representations.**
- The tool must receive the raw arguments and must return the raw result.
- The ledger must take its digest from the raw result, in memory. The program must not store or log the raw result for this purpose.
- The history, the display, the log, and the spill must receive the masked result only.
- Do not use a masked result as proof that two raw results are identical or different.

10.7 **Cap and splice.** A result over the cap must be spliced: head and tail, one marker with the omitted count, the do-not-infer rule, and one tool remedy.
- Report raw size and cap to the operator display.
- Cuts must land on UTF-8 boundaries.
- When head and tail overlap, cut from the start and say so.
- Do not insert a silent cap.

10.8 **Spill.** A cut result must go to one spill file of the masked result. The result must carry one note that names the path. The note marker must stay byte-identical for the compaction code. A spill file name must not contain tool arguments. The spill directory must be private to the operator. Error messages that name a path or a command must be masked.
10.9 **Count and duplicate warning.** Count the outcome of each executed call by its raw digest. When two consecutive executions of one signature return the same raw digest, append one duplicate warning with the tool name and the count. Other calls between them do not break the sequence. A refused call is not an execution. The warning is a subset of the ledger evidence, not a second count. Operator-facing text must say "same digest", not "verified identical".
10.10 A dispatched call that is not a pure read must mark the step as a write step. A refused call is not dispatched and must not mark the step as a write step.

## 11. The repeat ledger

11.1 The signature must cover the tool name plus the arguments. Object key order must not change it. Arguments that do not parse must hash as the raw text.
11.2 The ledger must keep three maps: read, shell, opaque. Each entry keeps the last raw digest and the count of identical outputs. The digest must be at least 128 bits. Identity is probabilistic; the program must claim no more than the digest supports.
11.3 The count must rise only while the output repeats. A changed output resets the count to one. A script that fails differently is progress, not a loop.
11.4 The count must be per signature, not consecutive. Alternating calls must be caught.
11.5 A dispatched shell or opaque call must clear the read map. Reads before a possible write are stale. A refused call must not clear any map.
11.6 The refusal total must never clear.
11.7 Shell and opaque calls use the shell limit. Reads use the read limit.
11.8 The loop must derive the signature at charge and at outcome count in the same way. Do not pass it in.
11.9 When refusals reach the stop limit, the loop must assign repeat_loop, send one stop note, and run finalization. When shell refusals exist, the stop note must steer to the fix tools.
11.10 The ledger must stay inside the turn. Do not carry read counts across turns.
11.11 A report call has no ledger entry. Every other call has one.

## 12. No-progress control

12.1 Count consecutive steps with tool calls and no real text. Real text resets the counter and the warned flag.
12.2 At the warn limit, send one warning with three options: succeed and answer, change the approach, abort and report the blocker.
12.3 At twice the warn limit, enter forced finalization. Keep only the report tool when a schema exists, and refuse every other call.
12.4 A report delivered under forced finalization must assign no_progress. A budget stop stays a budget stop.

## 13. Stream repetition

13.1 The stream pump must pass every chunk of the thinking text and the reply text through a detector. Do not check repetition only after the stream ends. The program must run two detector instances per response, one for each domain. They share the epoch strike counter. A window must not cross from one instance into the other.
13.2 **Two representations.**
- The word stream: the text after Unicode composition, case folding, and punctuation removal, split on whitespace. In a script without spaces, one grapheme cluster is one word.
- The separator stream: runs of punctuation or symbol characters, measured before punctuation removal, with whitespace between them ignored. Each 16 Unicode scalar values in a run must insert one reserved word into the word stream at that place.
- This is how a punctuation collapse stays visible although ordinary punctuation does not matter.

13.3 **Chunk invariance.** Each detector instance must keep partial words, partial UTF-8 sequences, partial grapheme clusters, and the open separator run between chunks. The detection result and the word index where it fires must be the same for every partition of the same text into chunks.
13.4 **Predicate.** Let N be 96 words. One window starts at each word position. One key identifies one window. One occurrence is one position where that key appears. The span of the occurrences of a cluster is the last start plus N, minus the first start. The detector must fire at the first word where either rule is true:
- Rule A: one key has, in one cluster, at least three occurrences, and their span is at least 288 words. Occurrences may overlap.
- Rule B: three keys, whose first occurrences do not overlap, each have three non-overlapping occurrences, and all nine occurrences lie inside one span of at most 864 words.

The predicate must exist as one pure function over a word list. The stream layer must give the same result as that function. All numbers are configuration. The failure shapes are the requirement: a repeated phrase or token, and a punctuation collapse.
13.5 A long response on its own must not trigger anything. Work with different content must continue.
13.6 **Domains.** The thinking instance must use the thresholds of 13.4. The reply instance must skip code spans and use stricter thresholds. Tool-call arguments must never go through either instance; the ledger guards repeated calls.
13.7 **Retained state and bounded memory.**
- The tracker keeps, per key: two independent hashes, the count of the current cluster, the first and last word positions of the current cluster, and the source byte offset of the first occurrence of the current cluster. A key match needs both hashes.
- When a new occurrence lands more than the larger trigger span (Rule B, 864 by default) after the key's last occurrence, the tracker must start a new cluster: count one, first position and first offset set to the new occurrence.
- The detector must not store raw model text.
- The capacity C is configured in words. Windows advance by one word, so the tracker keeps at most C window keys. It must evict the oldest keys first.
- The guarantee holds for a repeat whose first occurrence lies within C words of the current word. No guarantee holds outside it.
- The program must check at load time that the larger trigger span plus N is smaller than C. A configuration that fails the check must be rejected.
- The detector cost per chunk must not grow with the window size.

13.8 On the first strike in an epoch, the loop must abort the stream. The abort must use the same wake primitive as the cancel path.
13.9 **History safety.** Received bytes, displayed bytes, and committed bytes are three classes. The guarantee is history safety only.
- A reply-domain detection must trim the reply, before history sees it, at the source byte offset of the first occurrence of the current cluster, on a UTF-8 boundary. It must also drop any partial thinking block.
- A thinking-domain detection must drop the thinking block completely. Reply text that came before that block, if any, stays.
- The display may have shown the trimmed text; the loop must show one note that the stream was cut. Do not buffer the display to prevent this.
- The trimmed bytes were never part of a request, so rule 7.1 holds.

13.10 The corrective message must tell the model to stop the current approach, restate what it verified, and choose a different action. Do not send "continue from where you stopped". The text at the cut or at the step is not progress.
13.11 When the abort lands inside a tool call, the cut-reply rules apply: marker arguments (6.4), no execution (9.1), one "not executed" result per call (9.2). No continue message is sent.
13.12 An aborted stream has no final usage frame. Record one usage estimate from the streamed chunks and mark it as an estimate in the totals.
13.13 On the second strike in the same epoch, the loop must assign stream_loop, send no corrective message, show one stop reason to the operator, and run finalization.
13.14 **Epoch scope.** One epoch starts with each message typed by the operator. Program-injected messages keep the epoch. Do not use text matching to tell them apart. The strike counter carries the epoch scope. The ledger stays inside the turn.
13.15 The stream detector must add no session entry except one corrective message per strike. A step-level strike (section 14) must add no more than one message: one warning or one correction.
13.16 The per-step output limit must stay low, and provider repetition penalties must be used when they exist. The detector is the second line of defense.
13.17 A detector cut charges one step. It does not charge the continue budget. The strike counter, not the continue budget, bounds the corrective retry.
13.18 Every detection and every detector stop must write one session-log event. The event must name the detector kind (stream or step-level), the domain for a stream detection (thinking or reply), and the outcome of 13.19.
13.19 **Outcomes.** Every detection must have exactly one outcome:

| Phase | Outcome |
|---|---|
| A normal step, a corrective step, or a forced-finalization step; no strike yet in this epoch | first_strike |
| The same phases; one strike already in this epoch | terminal_strike |
| The finalization request (15) | ignored: log only |
| After the cancel flag is set, or after a stop value is assigned | ignored: log only |

## 14. Step-level repetition

Section 14 ships disabled by default. Enable a rule only when the incident logs support it.

14.1 A cycle of up to four step outcomes that repeats three times must count as one strike.
14.2 The same error that ends three steps in a row must produce one warning to the model when a write step falls between them. A fix that fails again is iteration (11.3).
14.3 When no write step falls between them, the same error that ends three steps in a row must count as one strike. Send no separate warning; the correction (13.10) is the message.
14.4 One action is one call signature. The actions of one step are its signatures, taken as a set. Four steps after operator continuation instructions must count as one strike when the Jaccard similarity of the action sets of each consecutive pair is at least 0.85. The threshold is configuration.
14.5 Strikes in this section follow 13.10, 13.13, 13.14, 13.17, and 13.19: correct once, then assign stream_loop.
14.6 All thresholds in this section are configuration.

## 15. Finalization

15.1 After step_limit, repeat_loop, or stream_loop, the loop must send one extra request that offers only the report tool. No other stop value gets this request.
15.2 This request must not be retried and must not change the stop value. A failure is not an error; the run is already over.
15.3 This request must run under the detector and under one hard output cap.
15.4 In this request, the checks must record results and must not refuse the report. The report rules of 10.2 apply with the finalization exception.
15.5 A detector abort inside this request must end the request. It must send no corrective message, must not count a strike, and must not change the stop value (13.19).
15.6 This request does not charge a step against the limit. Its usage must go into the totals.
15.7 A cancel during this request must end it. The stop value stays as assigned.

## 16. After each step

16.1 On a write step, run the after-write checks. Send one note only when the set of broken contracts changed.
16.2 Compact the history after each step. Compaction must obey 7.1. A compaction that must change bytes already sent is a declared rebuild: it must write one log event.

## 17. Display and secrets

17.1 Every call must get exactly one start event and one finish event, in that order. This covers unknown tools, broken arguments, refused calls, cut calls, and cancelled calls.
17.2 The display and the log must never receive a known secret, in a result, in arguments, in raw argument text, or in an error. Mask at the source, not at the sink.
17.3 Sort secrets longest first and remove duplicates. The longest match must win.

## 18. Memory

18.1 Short-lived values live in one arena per call. The detector state, both instances, lives in one arena per response. History lives in one arena per session.
18.2 Every container must get an explicit free on every stop path.
18.3 The debug build must report leaks at exit.

## 19. Tests

19.1 Each rule in this document must have at least one test.
19.2 The suite must prove:
- Changed output is not a repeat. Identical output still stops. Alternation is caught. Mutating calls reset reads. Refusals add up.
- A refused shell call does not clear the read map and does not mark a write step.
- A refused call between two executions of one signature does not break the duplicate-warning count; a call of another tool between them does not break it either (10.9).
- Two raw results that differ only inside a secret are not a repeat. A secret that changes between identical calls does not hide a repeat of the rest.
- The tool receives raw arguments. History, display, log, and spill receive only masked data.
- Malformed arguments that contain a secret reach the display and the log masked.
- No cap value panics. Kept parts come from the source. UTF-8 is safe.
- The longest secret wins. The display never sees a secret.
- A spill file name contains no tool arguments.
- Cut arguments stay parseable. History bytes do not change. Every call id has exactly one result.
- Cancel at each point of a step (during stream, before dispatch, during dispatch, during result delivery, during finalization) leaves every call paired once, one start and one finish event each, and the stop value unchanged where already assigned.
- One interrupt ends the turn as cancelled. A length-stop endpoint yields exactly one extra request.
- Each row of the table in 2.4 is reachable and gives that value. A transport failure with the flag clear gives an error result and no stop value. When two conditions occur, the first one wins.
- A valid report that passes the checks stops with reported, outside finalization.
- A valid, passing report during finalization leaves the stop value unchanged (10.2, 15.4).
- An exec-mode text ending with an open requirement does not assign end_turn (8.4).
- The predicate of 13.4 passes its own offline tests as a pure function, including Rule B cases just inside and just outside the 864-word bound.
- A key with more than three occurrences, including overlapping ones, gives the correct firing word and the correct trim offset.
- A key occurs once early, goes quiet for longer than the larger trigger span, then repeats enough to fire: the trim offset is the first occurrence of the late cluster, and the early occurrence does not inflate the Rule A span (13.7).
- A loop that starts in thinking text and continues into reply text: each instance judges only its own text, no window crosses the boundary, and the strike counter is shared (13.1).
- A thinking-domain detection drops the thinking block and keeps earlier reply text. A reply-domain detection trims the reply and drops partial thinking (13.9).
- The same text delivered in many chunk partitions (one chunk, one byte, random splits inside words and inside Unicode sequences) gives the same detection and the same firing word index.
- Separator runs that contain combining marks give the same count under every chunking (13.2).
- A false-positive corpus does not trigger the detector: long code files, hex dumps, JSON, tables, boilerplate.
- A token loop and a punctuation collapse trigger within bounds, with and without whitespace between the separators.
- Recorded real incidents (golden specimens, including the 74-second, 375-repeat class) trigger within bounds, with variations in chunking, Unicode, and punctuation.
- Unspaced-script text segments by grapheme.
- Detection still fires after the tracker has evicted, for a repeat inside the horizon.
- History after a trim contains no repeated span. The trim lands on a UTF-8 boundary.
- An abort inside a tool call leaves history valid for a strict server.
- The load-time check of 13.7 rejects a configuration where the larger trigger span plus N is not smaller than C, and accepts one where it is.
- A new operator message starts one epoch and clears the strike counter. A program-injected continue keeps it.
- After one detector cut and a corrective step, a clean text ending stops with end_turn, not max_tokens.
- An aborted stream records one usage estimate, marked as an estimate.
- Each row of the table in 13.19 gives its outcome.
- A failing check in finalization leaves the stop value unchanged (15.4).
- A detector abort in finalization sends no corrective message, counts no strike, and leaves the stop value unchanged (15.5).
- Finalization charges no step and adds its usage to the totals (15.6).
- A detector cut charges one step and does not charge the continue budget (13.17).
- The session-log event carries the detector kind, the domain, and the outcome (13.18).
- No stop value other than step_limit, repeat_loop, and stream_loop gets finalization (15.1).
- After stream_loop, no corrective message is sent, and finalization still runs (13.13).
- Thinking text alone does not reset the no-progress counter (12.1).
- Operator-facing text for a duplicate says "same digest" (10.9).

19.3 Tests must inject canned frames through the transport seam. No test may depend on a live model.

## 20. Prohibitions

- Do not execute a cut call.
- Do not continue without a charge.
- Do not clear the refusal total.
- Do not insert a silent cap.
- Do not show the display or the log a known secret.
- Do not change history bytes that were already sent.
- Do not leave a call without one result.
- Do not rely on a flag alone to wake a blocked wait.
- Do not charge a forced-finalization refusal as a repeat.
- Do not claim identity beyond what the digest supports.
- Do not use a masked result to decide identity.
- Do not let a refused call clear the read map or mark a write step.
- Do not push a repeated span into history.
- Do not resume a collapsed stream.
- Do not let text matching decide the epoch.
- Do not stop tracking new keys when the tracker is full.
- Do not send tool-call arguments through the detector.
- Do not let a window cross from thinking text into reply text.
- Do not let the chunk partition change a detection.
- Do not let a stale occurrence set the trim offset or the span.
- Do not charge the continue budget for a detector cut.
- Do not change a stop value after it is assigned.
- Do not assign reported, no_report, or checks_failed during finalization.
- Do not retry finalization after a detector abort.
- Do not charge a step for finalization.

