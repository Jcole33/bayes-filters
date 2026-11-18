# The GUI replays recorded runs instead of running filters live

Status: accepted

Filters are run to completion first and their belief at every step is recorded; the GUI only replays recordings. This lets the viewer scrub backward and single-step, keeps the animation smooth when one filter is slow, and guarantees the viewer shows exactly the runs that were scored. The cost is disk space for recordings, kept bounded by recording only a few seeds per benchmark by default, and no live tweaking of a filter mid-run.
