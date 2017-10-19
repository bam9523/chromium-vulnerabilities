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

[](Later: Add notes for 10/20/2017)
