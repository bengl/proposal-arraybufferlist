# ArrayBufferList

## Summary

This proposal introduces `ArrayBufferList`, an `ArrayBuffer`-like class that can
be used in the same places that `ArrayBuffer` can, much like
`SharedArrayBuffer`. The purpose is to be able to operate on sequences of
`ArrayBuffer`s in the same ways that single `ArrayBuffer`s can. It can be
thought of as a form of lazy concatenation, similar to V8's "cons strings".

## Motivation

Concatenation of data is very useful in many applications. Being able to
_conceptually_ concatenate without _physically/actually_ concatenating provides
the ability to defer true concatenation until later on, even if operations need
to be performed on the data. Today, concatenating `ArrayBuffer`s means
allocating a new `ArrayBuffer` as big as the sum of the lengths of the input
`ArrayBuffer`s, and then data needs to be copied manually, typically with a
`TypedArray` view. This is costly, and for large data, can be a bottleneck in an
application. If a sequence of `ArrayBuffer`s could be treated the same as a
single `ArrayBuffer` for all intents and purposes, manual copying wouldn't be
needed in cases where users want or need to operation on the sequence as if it
were a single one.

One example of where this is useful is in encoding data. Consider a set of
well-known byte sequences (say, for example, to indicate things like types and
lengths for subsequent data) and a cache of data (say from strings to UTF-8 byte
streams). An encoder might pull from those two data sets and add original data
of its own, stitching it all together into a contiguous memory chunk via
copying. If the encoder could avoid this copying operation, that would be
beneficial for performance.

## Details

Internally, `ArrayBufferList` stores a sequence of `ArrayBuffer`s and/or
`SharedArrayBuffer`s. When viewed with `TypedArray`s or `DataView`s, the user
perception is of a contiguous single `ArrayBuffer`. Wherever operations on
`TypedArray`s result in the creation of a new `ArrayBuffer` (for example,
copying), a new `ArraBuffer` is created, rather than a new `ArrayBufferList`.

The class has the same interface as ArrayBuffer, with the following differences:

1. The constructor takes either no arguments or an iterable of `ArrayBuffer`s,
   which constitute the inital state of the sequence.
2. `ArrayBufferList.prototype.append(arrayBuffer)`, which adds an `ArrayBuffer`
   to the end of the sequence.
3. The `.length` property refers to sum of the lengths of all the `ArrayBuffer`s
   in the sequence.
4. `ArrayBufferList.prototype.slice()` returns an `ArrayBuffer`, rather than an
   `ArrayBufferList`.
5. `ArrayBufferList` does not implement `Transferrable`.

All existing functions providing `ArrayBuffer`s continue to do so. All places
where `ArrayBuffer` can be used can also use `ArrayBufferList`.

This proposal does _not_ introduce a `SharedArrayBufferList`.

## Examples

#### Simple concatenation

```js
// Grab data from 3 endpoints, concatenate it all, and send it elsewhere
const res = await Promise.all([0, 1, 2].map(x => fetch(`https://example.com/${x}`)))
const bufs = await Promise.all(res.map(x => x.arrayBuffer()))
fetch('https://example.com/post, {
  method: 'POST',
  body: new ArrayBufferList(bufs...)
})
```

#### Encoding

```js
const cache = { /* values are all ArrayBuffers */}

function encode(list) {
  const result = new ArrayBufferList();
  for (const item of list) {
    result.append(cache[item];
  }
  return result; // use wherever ArrayBuffers are needed
}
```

## Alternatives

* `ArrayBuffer.prototype.append` could provide similar functionality, but
  transparently to users. That is, given a `ArrayBuffer`, it wouldn't be
  apparent to users whether it was backed by contiguous memory or a sequence of
  memory chunks.
* Allow for subclasses of `ArrayBuffer` to intercept data access. Then an
  `ArrayBufferList` class could exist in userland.

## Prior art

* <https://github.com/rvagg/bl> This is roughly the same concept, but for
  Node.js Buffers.

## TODO

* Should users be able to access and manipulate the sequence directly (e.g. with
  `Array`-like methods), or just have append since it will likely cover a good
  portion of possible use-cases?
