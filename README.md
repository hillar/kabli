Kabli (tuntud ka kui kartulikonks või kook) on Eesti traditsiooniline põllutööriist, mida kasutatakse kartulite mullast väljakaapimiseks.
Tegemist on varre otsa kinnitatud pika, täisnurga all või konksjalt painutatud metallist harude või teraga, millega sai mugulaid mugavalt pesast kätte ilma neid vigastamata. Sõna "kabli" (omastavas kabli, osastavas kablit) tähistab murdekeeles laiemalt ka konksu või koogu kujulist tööriista.


----



**Purpose:** Stop the AI from getting stuck, wasting time, or burning through tokens without making progress.

Think of it as three safety nets.

### 1. The AI repeats itself

The system watches the AI's text while it is being generated.

If it detects repeated passages or endless punctuation, it cuts the stream and tells the AI to stop, review what it has actually done, and try a different approach.

The AI gets one chance to recover. If it gets stuck again, the system stops the loop and tries to collect a final report.

### 2. The AI repeats the same tool call

The system remembers which tools the AI called, with which arguments, and what they returned.

If the AI keeps running the same command and getting the same result, the system can block the repeated call.

It is smart enough to distinguish a genuine change from a repeat: if a command produces a different result, that counts as progress. A command that fails differently is not automatically treated as a loop.

### 3. The AI repeats the same failed approach

Sometimes the AI changes its wording but keeps doing essentially the same thing.

An optional detector looks for repeating cycles, recurring errors, and nearly identical tool-call patterns across multiple steps. It can warn the AI or trigger the same stop-and-recovery mechanism.

This detector is disabled by default until real incident data justifies enabling it.

### Other safety limits

Even if none of the repetition detectors catches a problem, the system has additional limits:

* **Step limit:** Stops excessive model requests.

* **Continue budget:** Prevents endless automatic retries.

* **No-progress limit:** Forces the AI to produce a report when tool-using steps keep going without a meaningful reply.

* **Final report:** After certain stops, makes one last request to collect a report, if possible.

## The basic idea

It detects the problem, interrupts it, allows one meaningful recovery attempt where appropriate, and stops if the problem continues.

**In plain English: the AI is allowed to make mistakes, but it is not allowed to get stuck making the same mistake indefinitely.**

