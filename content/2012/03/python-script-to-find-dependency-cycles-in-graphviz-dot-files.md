Title: Python script to find dependency cycles in GraphViz dot files
Date: 2012-03-28 22:05
Author: admin
Category: Puppet
Tags: dot, graph, graphviz, puppet, python
Slug: python-script-to-find-dependency-cycles-in-graphviz-dot-files

Using [GraphViz](http://www.graphviz.org/) to describe configurations is
relatively popular in the software and systems architecture world; the
simple text-based format makes it quiet simple, and the directed graph
(dot file) is a simple method to store a graph of information flow or
component relationships. [Puppet](http://puppetlabs.com) includes
builtin support for [generating dot
graphs](http://docs.puppetlabs.com/guides/faq.html#how-do-i-use-puppets-graphing-support)
of its configuration resources and relationships (both those specified
by the user, and all relationships including ones generated by Puppet
itself).

One of the common uses for puppet's graphs is to identify dependency
cycles, as cyclic dependencies cause an error condition. However,
Puppet's own
[FAQ](http://docs.puppetlabs.com/guides/faq.html#how-do-i-use-puppets-graphing-support)
only mentions using the `dot` command to generate a PNG graphical
representation of the the graph. When debugging a recent problem with
puppet, I ended up with a message like:  

` Could not apply complete catalog: Found dependency cycles in the following relationships: Service[puppet] => File[/var/lib/puppet/yaml/foreman], File[/var/lib/puppet/yaml] => File[/var/lib/puppet/yaml/foreman], Service[puppet] => File[/etc/puppet/node.rb], File[fileserver.conf] => Service[apache], File[namespaceauth.conf] => Service[apache], File[puppet.conf] => Service[apache], File[puppet.conf] => Service[puppet], Service[puppet] => File[foreman-report.rb], File[/var/lib/puppet/yaml/foreman] => Package[puppet-server], Service[foreman-proxy] => Package[puppet-server], File[foreman-proxy-settings.yml] => Package[puppet-server], Package[foreman-proxy] => Package[puppet-server], User[foreman-proxy] => Package[puppet-server], File[/var/lib/puppet/yaml] => Package[puppet-server], File[foreman-report.rb] => Package[puppet-server], File[/etc/puppet/node.rb] => Package[puppet-server], File[/var/lib/puppet/yaml/foreman] => Exec[create_puppetmaster_certs], Service[foreman-proxy] => Exec[create_puppetmaster_certs], File[foreman-proxy-settings.yml] => Exec[create_puppetmaster_certs], Package[foreman-proxy] => Exec[create_puppetmaster_certs], User[foreman-proxy] => Exec[create_puppetmaster_certs], File[/var/lib/puppet/yaml] => Exec[create_puppetmaster_certs], File[foreman-report.rb] => Exec[create_puppetmaster_certs], File[/etc/puppet/node.rb] => Exec[create_puppetmaster_certs], File[/var/lib/puppet/yaml/foreman] => File[/etc/puppet/environments], Service[foreman-proxy] => File[/etc/puppet/environments], File[foreman-proxy-settings.yml] => File[/etc/puppet/environments], Package[foreman-proxy] => File[/etc/puppet/environments], User[foreman-proxy] => File[/etc/puppet/environments], File[/var/lib/puppet/yaml] => File[/etc/puppet/environments], File[foreman-report.rb] => File[/etc/puppet/environments], File[/etc/puppet/node.rb] => File[/etc/puppet/environments], File[foreman-proxy-settings.yml] => Service[foreman-proxy], Package[foreman-proxy] => Service[foreman-proxy], Service[puppet]`  
Not exactly easy to follow, or to pull much meaning out of, even if I
did some slight reformatting by putting in some newlines. I then
generated the png as described in the Puppet FAQs, but even scrolling
back and forth on this for 20 minutes didn't help: *(note: link is to
the original 14405x665px png)*  
[![dot file
png](/GFX/relationships.dot.small.png)](/GFX/relationships.dot.png)

With a little research, I managed to find the
[NetworkX](http://networkx.lanl.gov) package for Python; according to
their site, "NetworkX is a Python language software package for the
creation, manipulation, and study of the structure, dynamics, and
functions of complex networks." Among its features are the ability to
[read dot
files](http://networkx.lanl.gov/reference/drawing.html#module-networkx.drawing.nx_pydot)
using the [pydot](http://code.google.com/p/pydot/) library, and the
ability to [find simple
cycles](http://networkx.lanl.gov/reference/generated/networkx.algorithms.cycles.simple_cycles.html#networkx.algorithms.cycles.simple_cycles)
within a graph. In about 20 minutes, I hacked together the dead-simple
script below.

Given a dot file (of the type generated, for example, by Puppet), this
will output all of the cycles found within the graph. I ran this script
on the `expanded_relationships.dot` file from Puppet, which had 99 nodes
(nodes on the graph, not puppet clients), and got the following output:

~~~~{.text}
['File[foreman-report.rb]', 'File[puppet.conf]', 'Service[puppet]', 'File[foreman-report.rb]']
['File[foreman-report.rb]', 'Package[puppet-server]', 'File[puppet.conf]', 'Service[puppet]', 'File[foreman-report.rb]']
['Package[puppet-server]', 'File[puppet.conf]', 'Service[puppet]', 'Service[foreman-proxy]', 'Package[puppet-server]']
['Package[puppet-server]', 'File[puppet.conf]', 'Service[puppet]', 'File[/etc/puppet/node.rb]', 'Package[puppet-server]']
['Package[puppet-server]', 'File[puppet.conf]', 'Service[puppet]', 'File[foreman-proxy-settings.yml]', 'Service[foreman-proxy]', 'Package[puppet-server]']
['Package[puppet-server]', 'File[puppet.conf]', 'Service[puppet]', 'File[foreman-proxy-settings.yml]', 'Package[puppet-server]']
['Package[puppet-server]', 'File[puppet.conf]', 'Service[puppet]', 'User[foreman-proxy]', 'File[/etc/puppet/node.rb]', 'Package[puppet-server]']
['Package[puppet-server]', 'File[puppet.conf]', 'Service[puppet]', 'User[foreman-proxy]', 'Package[puppet-server]']
['Package[puppet-server]', 'File[puppet.conf]', 'Service[puppet]', 'Package[foreman-proxy]', 'Service[foreman-proxy]', 'Package[puppet-server]']
['Package[puppet-server]', 'File[puppet.conf]', 'Service[puppet]', 'Package[foreman-proxy]', 'File[foreman-proxy-settings.yml]', 'Service[foreman-proxy]', 'Package[puppet-server]']
['Package[puppet-server]', 'File[puppet.conf]', 'Service[puppet]', 'Package[foreman-proxy]', 'File[foreman-proxy-settings.yml]', 'Package[puppet-server]']
['Package[puppet-server]', 'File[puppet.conf]', 'Service[puppet]', 'Package[foreman-proxy]', 'Package[puppet-server]']
['Package[puppet-server]', 'File[puppet.conf]', 'Service[puppet]', 'File[/var/lib/puppet/yaml/foreman]', 'Package[puppet-server]']
['File[/etc/puppet/node.rb]', 'File[puppet.conf]', 'Service[puppet]', 'File[/etc/puppet/node.rb]']
['File[/etc/puppet/node.rb]', 'File[puppet.conf]', 'Service[puppet]', 'User[foreman-proxy]', 'File[/etc/puppet/node.rb]']
['File[puppet.conf]', 'Service[puppet]', 'Service[foreman-proxy]', 'File[puppet.conf]']
['File[puppet.conf]', 'Service[puppet]', 'Package[foreman-proxy]', 'Service[foreman-proxy]', 'File[puppet.conf]']
['File[puppet.conf]', 'Service[puppet]', 'Package[foreman-proxy]', 'File[puppet.conf]']
['File[puppet.conf]', 'Service[puppet]', 'Package[foreman-proxy]', 'File[foreman-proxy-settings.yml]', 'Service[foreman-proxy]', 'File[puppet.conf]']
['File[puppet.conf]', 'Service[puppet]', 'Package[foreman-proxy]', 'File[foreman-proxy-settings.yml]', 'File[puppet.conf]']
['File[puppet.conf]', 'Service[puppet]', 'File[/var/lib/puppet/yaml/foreman]', 'File[puppet.conf]']
['File[puppet.conf]', 'Service[puppet]', 'File[foreman-proxy-settings.yml]', 'Service[foreman-proxy]', 'File[puppet.conf]']
['File[puppet.conf]', 'Service[puppet]', 'File[foreman-proxy-settings.yml]', 'File[puppet.conf]']
['File[puppet.conf]', 'Service[puppet]', 'User[foreman-proxy]', 'File[puppet.conf]']
~~~~

It probably would have taken me hours to come up with that by hand, let
alone realize that `Service[puppet]` is the common item in all of them,
and hence the problem. With that little tidbit of information, I managed
to track down an extraneous "require puppet" that this all originated
from. I sincerely hope that this script will save someone else at least
as much time as it took me to write (I know it will for me...).

You can always obtain the latest version of this script from
[dot\_find\_cycles.py
(SVN)](http://svn.jasonantman.com/misc-scripts/dot_find_cycles.py), or
by checking out my [misc-scripts SVN
repository](http://svn.jasonantman.com/misc-scripts/). It's free for use
and distribution, provided that you leave my copyright/attribution and
source URL notice intact, update the changelog, and send any
features/fixes back to me. The script is written in Python, and depends
on the python-networkx, graphviz-python, and pydot packages (all of
which are available as packages in the default repos of Fedora and
CentOS at least). For Puppet purposes, I'd recommend running this on
`expanded_relationships.dot` to get the full information.

~~~~{.python}
#!/usr/bin/env python

"""
dot_find_cycles.py - uses Pydot and NetworkX to find cycles in a dot file directed graph.

Very helpful for 

By Jason Antman  2012.

Free for all use, provided that you send any changes you make back to me, update the changelog, and keep this comment intact.

REQUIREMENTS:
Python
python-networkx - 
graphviz-python - 
pydot - 
(all of these are available as native packages at least on CentOS)

USAGE:
dot_find_cycles.py /path/to/file.dot

The canonical source of this script can always be found from:


$HeadURL: http://svn.jasonantman.com/misc-scripts/dot_find_cycles.py $
$LastChangedRevision: 33 $

CHANGELOG:
    Wednesday 2012-03-28 Jason Antman :
        - initial script creation
"""

import sys
from os import path, access, R_OK
import networkx as nx

def usage():
    sys.stderr.write("dot_find_cycles.py by Jason Antman \n")
    sys.stderr.write("  finds cycles in dot file graphs, such as those from Puppet\n\n")
    sys.stderr.write("USAGE: dot_find_cycles.py /path/to/file.dot\n")

def main():

    path = ""
    if (len(sys.argv) > 1):
        path = sys.argv[1]
    else:
        usage()
        sys.exit(1)

    try:
        fh = open(path)
    except IOError as e:
        sys.stderr.write("ERROR: could not read file " + path + "\n")
        usage()
        sys.exit(1)

    # read in the specified file, create a networkx DiGraph
    G = nx.DiGraph(nx.read_dot(path))

    C = nx.simple_cycles(G)
    if(len(C) < 1):
        sys.exit(0)
    for i in C:
        print i
    
# Run
if __name__ == "__main__":
    main()
~~~~