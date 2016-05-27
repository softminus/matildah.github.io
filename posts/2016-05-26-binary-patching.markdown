---
title: Reverting annoying changes in the chromium web browser with binary patching!
---

Chromium got [this](https://chromium.googlesource.com/chromium/src/+/e53608ba54b3aff711c1e1c68243417f99bcd340%5E%21/) commit, which makes the "Let me choose when to run plugin content" option 

![](../images/2016-05-26-214632_531x220_scrot.png)

in `chrome://settings/content` be ineffective for PDFium, the chromium PDF reader plugin:

![](../images/2016-05-26-214607_763x394_scrot.png)

I was mildly annoyed because I regularly keep huge PDFs ^[either 4 kilopage datasheets or papers where every dot on a scatterplot is its own PDF object/entity/whatever and not part of a bitmap] in my tabs to refer to them regularly -- but only let PDFium run when i'm actively reading them to avoid any performance impact. I looked at the [changelog](https://chromium.googlesource.com/chromium/src/+log/50.0.2661.102..51.0.2704.63?pretty=fuller&n=10000) for chromium^[This was nontrival because this is chromium version 51 and searching "chromium 51" gives you results for chromium-51, a radioactive isotope of chromium used in radiolabeling studies] and found that commit, whose description is

~~~~~~~~
Internal PDF viewer will always load if it is not disabled, even if
plugins are set to CONTENT_SETTING_BLOCK in content settings.
~~~~~~~~


(note that nothing is mentioned of the potential negative impact on performance in that commit message).

The commit has two parts, some C++ code introducing a new plugin tag called "`PluginMetadata::SECURITY_STATUS_FULLY_TRUSTED`", which when present on a plugin, creates the "you can't enable click-to-run for this plugin, it'll always run, want it or not" behavior (which we want to get rid of), but the part that actually tags the chromium PDF plugin with the `fully_trusted` tag is not in C++ code, it's in some .json file:


~~~~~~~~
-        "status": "up_to_date",
+        "status": "fully_trusted",
~~~~~~~~

Now I'm thinking "oh I just need to edit that JSON file, so the PDF plugin doesn't get tagged with `fully_trusted`, and I'll get the desired behavior back!".

Not so fast -- I can't find a file with that name anywhere on my system! I ask my package manager what files the chromium package owns and find a suspicious-looking file called /usr/lib/chromium/resources.pak -- and I suspect that all the non-object-code parts of chromium get somehow baked into that file. Before bothering to find what file format that is, I run `strings(1)` on it and try to find the `fully_trusted` string I want to eliminate:

![](../images/2016-05-26-220110_800x552_scrot.png)

Bingo! it's there, and as I guessed, that resources.pak is a file where all the strings/files are [baked together](https://www.chromium.org/developers/design-documents/linuxresourcesandlocalizedstrings). Unfortunately, because this is a binary file it is almost certainly full of fields with offset information, and fixing those up is a huge pain -- I'd rather just find out how to generate it and generate it anew from all the files it contains.


Fortunately, I don't even need to do this! "`fully_trusted`", the string i want to replace, is longer than what I want to replace it with, the string "`up_to_date`"! This means I can write it over and just pad with spaces -- remember, we can't change the length/offset of **anything** without having to figure out how the offsets work^[and fixing them up as appropriate] in this file format, either of the whole .pak file or of the json file that we are editing.

Here's how it looks in our hex editor before we go hands-on:

![](../images/2016-05-26-221730_1599x629_scrot.png)

And how it looks after we set our hex editor's edit mode to "overwrite" and replaced and padded:

![](../images/2016-05-26-221839_1599x586_scrot.png)

We check that we didn't mess up the length of the file: 

![](../images/2016-05-26-222035_586x69_scrot.png)

and here's the diff:

~~~~~~~~
35301c35301
< 001477d8: 7374 6174 7573 223a 2022 6675 6c6c 795f 7472 7573 7465 6422 2c0a 2020 2020 2020 2020 2263 6f6d  status": "fully_trusted",.        "com
---
> 001477d8: 7374 6174 7573 223a 2022 7570 5f74 6f5f 6461 7465 222c 2020 200a 2020 2020 2020 2020 2263 6f6d  status": "up_to_date",   .        "com
35309c35309
< 00147908: 7573 223a 2022 6675 6c6c 795f 7472 7573 7465 6422 2c0a 2020 2020 2020 2020 2263 6f6d 6d65 6e74  us": "fully_trusted",.        "comment
---
> 00147908: 7573 223a 2022 7570 5f74 6f5f 6461 7465 222c 2020 200a 2020 2020 2020 2020 2263 6f6d 6d65 6e74  us": "up_to_date",   .        "comment
~~~~~~~~

We replace the original resources.pak with modded .pak and restart chromium, and it works perfectly!

![](../images/2016-05-26-222718_359x79_scrot.png)
![](../images/2016-05-26-222421_1236x677_scrot.png)
