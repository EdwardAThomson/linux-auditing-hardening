# linux-auditing-hardening
In this repo I will outline a process for performinga Linux audit. I will include simple scripts plus accompanying text to describe what to do. The script is intended to gather information that would indicate if a system has a number of high / medium impact issues.

The scripts provided are not exhausitve in the checks that they perform. I don't assume the context of the system you are looking at. Consequently, I don't outline all things that you should check for. Some of the issues that the scripts find may be of greater or lesser importance depending on what your machine actually does.

Running a script without having a process is a waste of time, so simply providing a script with no explanation is not particularly helpful. Once you have performed the audit, it is sensible to follow up by fixing the problems. Auditing without fixing (hardening) is somewhat pointless, but trying to harden a system without knowing what needs to be fixed is naive. This is why I try to provide some understanding of how to interpret the results. Once you have a fair understanding then you can start to fix these problems.
