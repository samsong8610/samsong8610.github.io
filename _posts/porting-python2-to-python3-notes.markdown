# Some Notes About Porting Python2 codes to Python3

## str and bytes

In python2, str represents single byte string and unicode is multiple bytes string.
In python3, str is default to unicode, and add a bytes type to represents raw bytes array.

encode/decode

## map() function

In python2, return list.
In python3, return an iterator.

[This question in stackoverflow](http://stackoverflow.com/questions/1303347/getting-a-map-to-return-a-list-in-python-3-x)
https://docs.python.org/2.7/library/functions.html#map

## dict

In python3, there no iterkeys(), itervalues(), iteritems(), use iter(d.keys()), iter(d.values()), iter(d.items()) instead.
http://legacy.python.org/dev/peps/pep-0469/
