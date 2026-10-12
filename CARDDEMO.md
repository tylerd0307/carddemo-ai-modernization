# Modernizing a Mainframe Program with an AI Agent: What I Checked and What I Found

Tyler Nguyen · September 2026

## What I did

I took one COBOL program, CBSTM03B, from AWS's open-source CardDemo credit card application and ran it through the steps that enterprise modernization platforms now automate with AI agents: discovery, code analysis, dependency mapping, target-state design, code generation, and testing and validation. Claude Code was the agent. I was the reviewer.

Scope matters here. CBSTM03B is the file-access layer for CardDemo's statement job. I ported that layer to Python. The statement logic itself lives in CBSTM03A, which I read to check the agent's claims but did not port.

## How I checked

One rule for every step: every claim had to cite a line number, and I opened the lines myself. I checked 14 claims across the first five steps, opened all 34 line citations behind the generated tests, and reran the full test suite myself instead of trusting the agent's report.

## What I found

**The facts held up.** Across all of those checks, the agent made one error: a line number off by one (it cited line 350 of CBSTM03A where the relevant code was on line 349). Its descriptions of the files, record sizes, dependencies, and job setup were correct.

**It carried over legacy bugs on its own.** I did not tell it to. It chose to keep three behaviors from the original:

- A mistyped file name comes back as a success, because the calling program sets the result code to "00" before the call (CBSTM03A lines 349, 375, and 399) and CBSTM03B never overwrites it.
- An operation a file does not support also reports success.
- The sort step keeps a 50-byte copy of a record that cuts the last two characters off the processing timestamp.

**Where the original didn't define behavior, it picked its own.** A key length outside 1 to 25 now raises an error, and the result code before any file operation is two spaces. Both are reasonable choices. Nobody approved either one.

**The tests were real.** They were written from the COBOL, not from the agent's own Python, and all 34 citations matched the lines they claimed. The suite ran each test against both EBCDIC (mainframe) and ASCII data. The agent reported 68 passing and 0 failing, and I got the same result when I ran it.

**The money check didn't apply.** This layer copies records byte for byte and never reads the dollar amounts inside them, so there was no rounding risk to find.

**My own bottleneck was reading COBOL.** I needed the agent to explain syntax, such as `PERFORM 9999-ABEND-PROGRAM` being how the program stops a job, and then I confirmed it by seeing the same pattern repeat across the code.

## What I'd want from a modernization platform

1. **A decision log the agent has to fill in.** Every preserved legacy bug and every choice made where the original was undefined gets listed for a person to approve before the code ships. In this run, that would have been five items.
2. **Citations by default.** A claim with a line number can be checked in seconds. A claim without one can't be checked at all.
3. **Two kinds of tests, kept separate.** One set proves the new code matches the old system, so old and new can run side by side. A second set tracks the known bugs that should be fixed once the new system is trusted.

## How I'd measure an agent doing this at scale

- **Spot-check failure rate:** the share of checked claims that turn out wrong. Here, 1 out of 48, and that one was cosmetic.
- **Unapproved decisions per program:** choices the agent made that no person signed off on. Here, 5.
- **Reviewer time per program:** how long a person needs to verify the agent's work.

## Bottom line

Accuracy wasn't the bottleneck. Decisions were. The agent got the facts right and still made five choices about how the new system should behave without asking anyone. That is where a person has to stay in the loop.
