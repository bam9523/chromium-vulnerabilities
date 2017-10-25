Notes on Vulnerabilities
====================================


CVE-2013-6622
------------------------------------

### 10/18/2017 Notes (initial investigation)

 * "Use after free" vulnerability.
   This seems to correspond to CWE-416: Use After Free, described as "referencing memory after it has been freed.
   As one would expect, the CVE points out that this fits in the DoS category of threats.

 * Curiously, although the CVE states that the error occured in the "Blink" engine, it is clearly located within the WebKit engine.
   All of the links provided seem to point to that reality.
   The Blink engine is a separate project in a separate repository; thank GNU I don't have to scour a different repository this time.

 * Looks like I was lucky for this one; the CVE came with a description of the fix commit record.
   A cursory look at the fix commit record indicates that the fix did indeed address the vulnerability.

 * Last major update was 5/03/2014 and last modified date was 9/18/2017 even though the vulnerability fix was committed 10/07/2013.
   Should look into whether anything major about it changed.

 * Note: the files were moved into a "media" subdirectory in 2016, so any `git blame` on the current repository will have to go into there.

 * The bug was reported by someone under the name of "cloudfuzzer".
   To reproduce it, they created a test case, which is obviously what I will take note of in the YAML file.
   Another open-and-shut part of the analysis.

 * Bug fix occurred on lines 373-370 and lines 394-398.
   These lines correspond to lines 571-577 and lines 592-595 in the current version of the file, from what I can tell.
   Very important information for `git blame`!

 * Even though acolwell added clear comments to describe what they were doing and why, I'll admit they are a little difficult to parse.
   From what I can tell, the vulnerability itself occurs when a media element (such as a video) is moved between documents with precise timing.
   This can result in a crash, as the test case presumably demonstrates.
   The bug report notes that `gc`, which I will assume refers to Chrome's "garbage collector", must be enabled.

 * Reading the code a little more closely, it appears that the fix checks `m_should_delay_load_event`, which... is exactly what it says on the tin, to determine which document's load delay should increment.
   The variable itself appears in other places throughout the code for similar checks.
   If there is an old document, the load delay is decremented, since there's no chance that a load event could be sent from the destructor.
   ...that is, the fix increases the load delay if the old document is present, preventing the corresponding media player (the element being moved) from loading within the old document while being destroyed.
   The media element is moved to the new document.
   Then and only then is the old document's load delay decreased.
   ...it's a lot to swallow, but I'm confident I'll be able to understand if I go over the context of the method call (`HTMLMediaElement::didMoveToNewDocument`) in the rest of the program.


### Finding the VCC

 * Looking at the present Chromium repository, it appears the change occurs in what is now 569-578.

 * To find the VCC, I will make use of the `-L` switch with `git log`, which allows me to the history of all past commits on a given line range and file.
   Since we know which lines we are targeting, this should let us step through the history of the commits to see what has occurred.
   I will also make use of the `-u` flag to generate patches and the `--topo-order --graph` flags in case there are any branches I need to worry about.
   Then, I can simply search for the date `Oct 7` in the output to start my search.

 * The git command is as follows:

```
git log --topo-order --graph -u -L 569,578:HTMLMediaElement.cpp
```

 * Searching through, it appears that the if statement containing the offending use after free vulnerability was introduced in December 26, 2011.
   *However*, the commit message claims that there was "\[n\]o behavior change".
   I am skeptical, but for the moment I am inclined to believe their claim since it appears the commit was reviewed and verified before merging.
   As such, I suspect the vulnerability was introduced in the code this commit replaces.

 * My main concern now is that the commit removed the former `MediaElement::willMoveToNewOwnerDocument` and consolidated it with `MediaElement::didMoveToNewOwnerDocument` into the method `HTMLELement::didMoveToNewDocument`.
   This means that the use after free vulnerability could have existed in *either or both* of those methods, assuming that no regression occurred during the consolidation (which I am still slightly skeptical of).

 * I do believe I understand *why* the vulnerability occurred, even if I do not know where.
   Before the fix, the load event delay for the old document was being decremented before the media element was destroyed.
   This meant that it was possible for a precisely-timed load event sent to the old document to occur after the element in the old document was destroyed.
   Since the memory for the element had already been freed at that point, a crash can occur.

  * I am not seeing any evidence that the vulnerability could have occurred before the previously-mentioned combining commit.
    The logic creating the vulnerability did not exist in the pre-commit code, and I do not see anywhere in the original code where a use-after-free could have occurred.
    This is very strange to me, since the commit message claims the functionality is "identical", and the patch clearly overwent a thorough review before it was merged.
    Maybe it was an oversight due to the many files that were changed?

  * Based on this information, I believe I can argue in favor of `e92208d3534e3f78d7340d662b21085b9f609f84` as being the VCC.
