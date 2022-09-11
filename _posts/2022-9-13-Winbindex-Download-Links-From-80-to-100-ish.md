---
layout: post
title: Winbindex Download Links - From 80% to 100%&NoBreak;(-&NoBreak;ish)
image: /images/Winbindex-Download-Links-From-80-to-100-ish/Download-link-candidates-popup.png
---

About two years ago [I announced Winbindex - the Windows Binaries Index]({{ site.baseurl }}/Introducing-Winbindex-the-Windows-Binaries-Index/). I described how I downloaded all the Windows 10 update packages I could find, and how, with the help of [VirusTotal](https://www.virustotal.com/), I was able to generate download links for the files from the update packages that were submitted to the service. If you didn't read the announcement blog post, I suggest you read it for the motivation behind the project and for the full technical details. Below is a quick recap, followed by the small updates Winbindex got during these two years. After bringing you up to date, I'll tell you how I was able to generate download links for all files, even those that weren't submitted to VirusTotal, and what are the limitations of the method I used.

# A quick recap

The update files in an update package are divided to assemblies, each assembly having a manifest file and a folder with delta files ([forward and reverse differentials](https://docs.microsoft.com/en-us/windows/deployment/update/psfxwhitepaper)). In my original research two years ago, I assumed that the delta files aren't useful without the base files, so I focused on the manifest files. From the manifest files, I was able to get the SHA256 hash of the files that are part of each assembly. Then, I used the hash to search for each file in VirusTotal. If a file was submitted before, I could retrieve enough information to be able to generate a download link to the file in the Microsoft Symbol Server. When I did the initial indexing, 80.6% of the files were submitted, and these files could (and still can) be downloaded on [Winbindex](https://winbindex.m417z.com/).

# Winbindex updates

Since the announcement two years ago, Winbindex got a couple of updates:

* Files from ISO images of all 13 Windows 10 versions were added to the index (soon to be 14 with Windows 10 version 22H2). This means that files which don’t appear in any update package, but appear in the initial Windows release are now also indexed.
* The index generation scripts [are now on GitHub](https://github.com/m417z/winbindex/tree/gh-pages/data), and are scheduled to run automatically via GitHub Actions. This means that Winbindex is kept up to date with Windows updates automatically, and the new files are usually indexed a couple of hours after an update becomes available.
* Microsoft Visual C++ Redistributable files, which are the most commonly searched files in Winbindex, got special treatment - Winbindex [shows a helpful message](https://twitter.com/m417z/status/1342497839235158019) that suggests the correct redistributable version, along with the download links.
* More file types have download links. Originally, only exe, dll and sys files had download links. Due to user feedback, [additional PE file extensions were added](https://github.com/m417z/winbindex/commit/9a80b4362c87770630c95b6855f988343ef3a366#diff-35b2280c10028b9bf7f529f490ff325512552e0143f1f20d0352b5f5ce639d2dR366), such as cpl, efi and scr.
* Files from the Windows 11 ISO image were added to the index, and the scripts were updated to support Windows 11 updates. One of the changes was in the manifest files, which are now compressed with a proprietary format. Luckily, I found the [SXSEXP](https://github.com/hfiref0x/SXSEXP) tool by [hfiref0x](https://github.com/hfiref0x) which I used to decompress the files, but since the tool is Windows-only, the Winbindex update scripts now require Windows (they were running on Linux before).
* Minor UI improvements.

At the time of writing these lines (September 10), 348,993 PE files are indexed, 285,235 of which have download links (81.7%). As a reminder, the initial indexing included 134,515 PE files, 108,470 of which had download links (80.6%). As you can see, we keep the 80%-ish metric, which isn't bad. But can we do better?

# Delta file headers

A couple of weeks ago, [slipstream/RoL](https://github.com/Wack0) contacted me and shared his [DeltaDownloader](https://github.com/Wack0/DeltaDownloader) project, suggesting that I might be interested. The DeltaDownloader tool is able to find Microsoft Symbol Server links for delta compressed PE files using just the information in the delta compression header. To see how it's done, you can refer to the project's README file. Here's the technical part for your convenience:

> The delta compression header for a PE file contains the following information:
> 
> - Size of output file
> - `TimeDateStamp` of output file
> - `(VirtualAddress, PointerToRawData)` for each section of output file
> 
> PE files on the Microsoft symbols server are stored with keys of `(OriginalFileName, TimeDateStamp, SizeOfImage)`.
> 
> Two of those are known (the original file name being the filename of the delta compressed file, the TimeDateStamp in the delta compression header).
> 
> The other can be determined from the size of the output file and the VirtualAddress/PointerToRawData of the last section:
> 
> - `OutputSize - Sections.Last().PointerToRawData` is the size of the last section plus PE signatures.
> - This value can be rounded up to a page size, leading to a low number of potential `SizeOfImage` values to check.
> - These can be checked by HEAD requests to the Microsoft symbols server: `302` means the correct value was discovered.
> 
> Enough of the functionality of `msdelta.dll` has been reimplemented in C# to allow for obtaining the required values from the delta compression header.

It turns out that delta file headers contain a bunch of information, so delta files can be pretty useful even if the base files aren't available. DeltaDownloader uses this information to generate a list of candidate URLs, and then tries them all against the Microsoft Symbol Server to filter out the invalid URLs.

This capability is a great addition for Winbindex. Although trying all candidate URLs is not feasible for the huge amount of indexed files, being able to get the list of candidate URLs right from Winbindex is a great improvement compared to what Winbindex had to offer for these files before (no links at all).

# Integrating DeltaDownloader into Winbindex

I updated the Winbindex scripts to use DeltaDownloader and grab the useful information from the delta files - the `TimeDateStamp` value and the `(VirtualAddress, PointerToRawData)` values for the last section. I also downloaded past update packages and updated existing indexed files with information from the delta files. Not all files could be updated this way - update packages for old Windows 10 versions (version 1803 and older) include the actual files instead of delta files, and many old update packages are no longer available in the [Microsoft Update Catalog](https://www.catalog.update.microsoft.com/Home.aspx). But many newer files were updated successfully, and what's more important, Winbindex will now gather this information automatically for new updates.

# The result

![Download link candidates popup]({{ site.baseurl }}/images/Winbindex-Download-Links-From-80-to-100-ish/Download-link-candidates-popup.png)

You can check out the result on the website, for example [here for win32k.sys](https://winbindex.m417z.com/?file=win32k.sys). Note how the files which don't have a single download link have a hint on the download button, and clicking on it shows a list of candidate links. You can just try these links one by one (usually one of the first links is the correct one), or you can feed this list to a download tool such as curl, wget or aria2.

In addition to having more files available to download, newly indexed files are now available sooner. Files are indexed automatically as soon as an update package is available, but it can take days or weeks to get the information for the download link from VirusTotal. With the information from delta files, download links also become available as soon as an update package is available.
