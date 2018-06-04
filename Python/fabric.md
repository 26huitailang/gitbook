### [pip installation /usr/local/opt/python/bin/python2.7: bad interpreter: No such file or directory](https://stackoverflow.com/questions/31768128/pip-installation-usr-local-opt-python-bin-python2-7-bad-interpreter-no-such-f)

I got same problem. If I run `brew link --overwrite python2`. There was still `zsh: /usr/local/bin//fab: bad interpreter: /usr/local/opt/python/bin/python2.7: no such file or directory`.


```
cd /usr/local/opt/
mv python2 python
```

Solved it! Now we can use python2 version fabric.
