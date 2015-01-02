---
layout: post
title: "Foxit PDF Creator can fuck your software"
date: 2014-11-28
---

I stumbled across a curious bug at work today. I'm currently updating some old bits of code managing the printing of a report ( the software's current configuration and whats not), the files had not been modified for almost ten years. At one point my XPS output crashed (it couldn't handle the <code>"\r\n"</code> chars) so I decided to take a look at the PDF version.

<!--more-->

in addition to the usual PDF Creator, I also use the Foxit suite since their reader is much more faster than Acrobat's. I specify the Foxit virtual printer as my printing output machine, and press Ctrl+P.
The resulting document did not crashed and were correctly formated, although it lacked the company logo. But the real bug was whe I came back to my software and I couldn't loading my configuration files anymore. Moreover, only the Foxit printer was enabling the bug.


After some debugging, I found out my working directory had changed by the time I launched the print, surely called by the virtual printer (which was call in the same process as my software). That part of the application was still relying on relative paths, so every filepath was made incorrect by the print operation (that's why I had a broken link for the company logo's bitmap). After some cleaning and reconstructing the full paths needed, everything went fine again.

Anyway, the lesson to learn is to assume external programs can fuck your software in various and unexpecteded ways. 

PS: <a href = "https://bugs.openjdk.java.net/browse/JDK-6710022?page=com.atlassian.jira.plugin.system.issuetabpanels:changehistory-tabpanel"> there is closed bug on OnpenJDK describing this exact bug</a> .