---
layout: post
title: Writing ZIP files to the App Engine Blobstore (en)
---

{{ page.title }}
================

To export big (e.g. 50 MB) collections of data from Google AppEngine the ZIP Format seems to be a good
choice. Since App Engigne Application can not handle chunks of data bigger than 1 MB we have to build the 
ZIP file pice by pice on external storage.

The most obvious place to do so is the blobstore. Since App Engine 1.4.3 SDK you can [write directly to the blobstore][1]. Using the Python [zipfile][2] module you can directly generate zipfiles. The usage of `StringIO` as [occasionaly described][3] will only work for outputting ZIP files smaller than MB.

So the ZIPfile has to be written directly into the blobstore. Unfortunately the blobstore file emulation layer does not work with the zipfile module: the `tell()` method is missing from the file-like objects and the zipfile module uses it for providing features for appending ZIP files to executables and the like.

To get arrount this issue, I created a stripped down version of the zipfile module which works with the blobstore. You can get the code as [zipfile_blobstore.py][4] on GitHub.

With that out of the way it is relatively easy to create ZIP files bigger than 1 MB:

<script src="https://gist.github.com/950860.js?file=zib_datastore.py"></script>

This exports a series of PDF files saved in the datastore into a ZIP in the blobstore.

[1]: http://code.google.com/appengine/docs/python/blobstore/overview.html#Writing_Files_to_the_Blobstore
[2]: http://docs.python.org/library/zipfile.html
[3]: http://stackoverflow.com/questions/963800/zipping-dynamic-files-in-app-engine-python
[4]: https://gist.github.com/950846
