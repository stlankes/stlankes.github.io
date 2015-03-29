---
layout: post
title: "Maverick’s new SMB2 stack isn’t compatible to many current NAS systems"
date: 2013-10-25T14:31:00.362000+01:00 
tags: [OSX]
thumb: devnull.jpg
share: true
comments: true
---

Apple shifts in OS X 10.9 (Mavericks) [from AFP file sharing to SMB2](http://appleinsider.com/articles/13/06/11/apple-shifts-from-afp-file-sharing-to-smb2-in-os-x-109-mavericks).
However, the current implementation seems to be a buggy and is not compatible with my NAS system from Synology.

A workaround is to force the using of SMB1. This could be done by the creation of the file ~/Library/Preferences/nsmb.conf with following content:

	[default]
	smb_neg=smb1_only

If Apple solves this problem, you could switch back to SMB2 by deleting the file. With

	man nsmb.conf

you will get further information about the SMB configuration file.

**Edit:** It seems to be a bug in Synology DiskStation Manager (DSM) und its fixed in their new [release](http://www.synology.com/releaseNote_enu/DS212j.php?lang=us).
