.. title: Detecting excessively suppressed redirects in MediaWiki
.. date: 2018-05-06 05:00:00 +0300
.. tags: mediawiki, python, pywikibot
.. category: mediawiki
.. type: text

A simple Python script to detect excessively redirected pages in MediaWiki.

<!-- TEASER_END -->

> The original question was originally posted on June 21, 2014 at [StackOverflow](http://webmasters.stackexchange.com/questions/65138/how-do-i-find-excessively-suppressed-redirects-in-mediawiki)

I don't do much Python, but now the following script looks a bit funny to me since almost years are gone. :)

```python
#!/usr/bin/python
# -*- coding: utf-8  -*-

import pywikibot
import re
import sys

try:
    # Win32
    from msvcrt import getch
except ImportError:
    # UNIX
    def getch():
        import sys, tty, termios
        fd = sys.stdin.fileno()
        old = termios.tcgetattr(fd)
        try:
            tty.setraw(fd)
            return sys.stdin.read(1)
        finally:
            termios.tcsetattr(fd, termios.TCSADRAIN, old)

def process_excessive_redirects(modify = False, pause = False):

    wiki = pywikibot.Site()
    alt_link_re = re.compile('\[\[\s*([^\|\]]+)\s*\|\s*([^\]]+)\s*\]\]')

    redirects_index = {}
    print 'Parsing redirects:'
    for redirect in wiki.allpages(filterredir = True):
        print '\t', redirect.title().encode('utf8'), '->',
        redirects_index[redirect.title()] = redirect.getRedirectTarget().title()
        print redirects_index[redirect.title()].encode('utf8')

    print 'Processing:'
    for page in wiki.allpages(filterredir = False):
        print '\t', page.title().encode('utf8'), '-',
        statistics = {'modification_count': 0} # python 3: nonlocal
        def fix_redirect(match_object):
            target = match_object.group(1)
            title = match_object.group(2)
            if title.replace("_", " ") in redirects_index.keys() and redirects_index[title] == target:
                if statistics['modification_count'] == 0:
                    print
                print '\t\texcessive redirect', target, '~~~>', title, '~~~>', target
                statistics['modification_count'] += 1
                return '[[' + title + ']]'
            return match_object.group(0)
        text = alt_link_re.sub(fix_redirect, page.get())
        if statistics['modification_count'] > 0:
            print "\t\t", statistics['modification_count'], 'excessive redirect(s) detected.',
            if modify:
                print 'Fixing redirects...',
                page.put(text, str(statistics['modification_count']) + ' excessive redirect(s) fixed')
                if pause:
                    print 'Press any key . . .'
                    getch()
            else:
                print
        else:
            print 'clean!'

def main(*args):
    modify = False
    pause = False
    for arg in pywikibot.handleArgs(*args):
        if arg == '--modify':
            modify = True
        elif arg == '--pause':
            pause = True
    process_excessive_redirects(modify = modify, pause = pause)

if __name__ == '__main__':
    main()
```
