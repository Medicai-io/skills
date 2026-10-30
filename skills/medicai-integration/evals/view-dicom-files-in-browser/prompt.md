---
tags: [medicai-integration]
max_turns: 30
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

I've got a folder of DICOM files from my own knee MRI (the clinic gave me a CD and I copied it to ./knee-mri) and I want a simple web page where I can open and look at them in the browser, scroll through the slices, maybe adjust contrast. Nothing fancy, I just don't want to install a desktop DICOM program. Node or Python, whatever is simpler. Later I might want to keep a few of these online so I can show them to another doctor. The folder is on my laptop, not on this machine, so just build the page and I'll point it at the folder myself.
