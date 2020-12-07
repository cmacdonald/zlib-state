# zlib-state

Low-level interface to the zlib library that enables capturing the decoding state.

## Install

From PyPi:

```
pip install zlib-state
```

From source:

```
python setup.py install
```

Tested on ubuntu/macos/windows with python 3.5-3.9.

## GzipStateFile

Wraps Decompressor as a buffered reader.

Based on my benchmarking, this is somewhat slower than python's gzip.

A typical usage pattern looks like:

```python
import zlib_state

TARGET_LINE = 5000 # pick back up after around the 5,000th line
# Specify keep_last_state=True to tell object to grab and keep the state and pos after each block
with zlib_state.GzipStateFile('testdata/frankenstein.txt.gz', keep_last_state=True) as f:
    for i, line in enumerate(f):
        if i == TARGET_LINE:
            state, pos = f.last_state, f.last_state_pos

with zlib_state.GzipStateFile('testdata/frankenstein.txt.gz', keep_last_state=True) as f:
    f.zseek(pos, state)
    remainder = f.read()
```

## Decompressor

Very basic decompression object that's picky and unforgiving.

Based on my benchmarking, this can iterate over gzip files faster then python's gzip.

A typical usage pattern looks like:

```python
import zlib_state

decomp = zlib_state.Decompressor(32 + 15) # from zlib; 32 indicates gzip header, 15 window size
block_count = 0
with open('testdata/frankenstein.txt.gz', 'rb') as f:
    while not decomp.eof():
        needed_input = decomp.needs_input()
        if needed_input > 0:
            # decomp needs more input, and it tells you how much.
            decomp.feed_input(f.read(needed_input))
        # next_chunk may be empty (e.g., if finished with gzip headers) or may contain data.
        # It sends as much as it has left in its output buffer, or asks zlib to continue.
        next_chunk = decomp.read() # you can also pass a maximum size to take and/or a buffer to write to
        if decomp.block_boundary():
            block_count += 1
            # When it reaches the end of a deflate block, it always stops. At these times, you can grab the state
            # if you wish.
            if block_count == 4: # resume after the 4th block
                state = decomp.get_state() # includes zdict, bits, byte -- everything it needs to resume from pos
                pos = decomp.total_in() # the current position in the binary file to resume from
    print(f'{block_count} blocks processed')
    # resume from somewhere in the file. Only possible spots are the block boundaries, given the state
    f.seek(pos)
    decomp = zlib_state.Decompressor(-15) # from zlib; 15 window size, negative means no headers
    decomp.set_state(*state)
    while not decomp.eof():
        needed_input = decomp.needs_input()
        if needed_input > 0:
            # decomp needs more input, and it tells you how much.
            decomp.feed_input(f.read(needed_input))
        next_chunk = decomp.read()
```

