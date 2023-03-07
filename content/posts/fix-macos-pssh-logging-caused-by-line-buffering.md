---
title: "Fix macOS pssh logging caused by line buffering"
date: 2022-12-07 18:31:35+01:00
# aliases: ["/first"]
tags:
    - macos
    - homebrew
    - python
author: "ae3"
draft: false
hidemeta: false
comments: false
description: ""
#canonicalURL: "https://canonical.url/to/page"
hideSummary: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "https://oggen.ae3.ch/api/ae3?blogtitle=Fix%20macOS%20pssh%20logging%20caused%20by%20line%20buffering" # image path/url
    alt: "Fix macOS pssh logging caused by line buffering" # alt text
    caption: "Fix macOS pssh logging caused by line buffering" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
If you want to log the output of an SSH connection under macOS with pssh from Homebrew, the following error occurs:

 `line buffering (buffering=1) isn't supported in binary mode`

In the pssh command this happens when you define the `-o` option for output.

 `pssh -t 0 -p 10 -l root -h host.list -o logs 'command'`

In order to log the output of the SSH session, we have to make a small adjustment to the Python psshlib. 

## Here is how to fix pssh session logging on macOS Ventura

Let’s open this file with an editor of your choice:

{{< notice note >}}
The path may change for you, especially if you have a different version of pssh installed. Just adjust the path to your version number. 
{{< /notice >}}

`/opt/homebrew/Cellar/pssh/2.3.1_6/libexec/lib/python3.10/site-packages/psshlib/manager.py`

Inside the file we search for `buffering`. Then we change the code as follows.

`buffering=1` needs to be changed to `buffering=0` 

At the end of the if else statement, we add the following two lines:
```python
dest.write(data)
dest.flush()
```

This code block should now look similar to the following code:

```python
    def run(self):
        while True:
            filename, data = self.queue.get()
            if filename == self.ABORT:
                return
    
            if data == self.OPEN:
                self.files[filename] = open(filename, 'wb', buffering=0)
                psshutil.set_cloexec(self.files[filename])
            else:
                dest = self.files[filename]
                if data == self.EOF:
                    dest.close()
                else:
                    dest.write(data)
                    dest.flush()
```

Now we save the file and our changes. 
From now on pssh can log the output to a file without getting a buffering error message. 

Please keep in mind that you may need to make this change again in case you apply a homebrew update on your Mac. 