# CESIUM_binary_glTF

## Contributors

* Patrick Cozzi, [@pjcozzi](https://twitter.com/pjcozzi)
* Tom Fili, [@CesiumFili](https://twitter.com/CesiumFili)

## Status

Experimental

## Dependencies

Written against the glTF 0.8 spec.

## Overview

glTF provides two delivery options that can be also be used together:
* glTF JSON points to external binary data (geometry, key frames, skins), images, and shaders.
* glTF JSON has base64-encoded binary data, images, and shaders using data uris.

Base64-encoding increases the file size and requires extra decoding.  glTF is commonly criticized for requiring separate requests or extra space due to base64-encoding.

To solve this, this extension introduces a container format, _Binary glTF_.  In Binary glTF, glTF resources (JSON, .bin, images, and shaders) are stored in a binary blob.  The `TextDecoder` JavaScript API can be used to extract the JSON from the binary blob, which can be parses with `JSON.parse` as usual, and then the binary blog is treated as a glTF `buffer`. Informally, this is like embedding the JSON, images, and shaders in the .bin file.

The binary blob is little endian.  It has a 16-byte header followed by the glTF resources, including the JSON:
![](layout.jpg)

`magic` is the ANSI string `'glTF'`, and can be used to identify the binary blob as Binary glTF.  `version` is an `uint32` that indicates the version of the Binary glTF container format, which is currently `1`.

TODO: `jsonOffset` and `jsonLength`


Given an `arrayBuffer` with Binary glTF, Example 1 shows how to parse the header and extra the JSON.  This uses a `TextDecoder` wrapper in Cesium, [`getStringFromTypedArray`](https://github.com/AnalyticalGraphicsInc/cesium/blob/bgltf/Source/Core/getStringFromTypedArray.js).

**Example 1**: parsing Binary glTF
```javascript
var sizeOfUnit32 = Uint32Array.BYTES_PER_ELEMENT;

var magic = getStringFromTypedArray(arrayBuffer, 0, 4);
if (magic !== 'glTF') {
    // Not Binary glTF
}

var view = new DataView(arrayBuffer);
var byteOffset = sizeOfUnit32;  // Skip magic number

var version = view.getUint32(byteOffset, true);
byteOffset += sizeOfUnit32;
if (version !== 1) {
    // This only handles version 1.
}

var jsonOffset = view.getUint32(byteOffset, true);
byteOffset += sizeOfUnit32;

var jsonLength = view.getUint32(byteOffset, true);
byteOffset += sizeOfUnit32;

var jsonString = getStringFromTypedArray(arrayBuffer, jsonOffset, jsonLength);
var json = JSON.parse(jsonString)
```


TODO: below here.

The Binary glTF format we prototyped is:
```
magic number, uint32 // 'glTF'
version, uint32
jsonOffset, uint32
jsonLength, uint32
// embedded (or separate) data
```

Our loader is [here](https://github.com/AnalyticalGraphicsInc/cesium/compare/bgltf#diff-bcedb35459e5bed46206bad4990bc7f1R779).

This required some minor spec additions (which are up for discussion):
* `"self"` is the buffer that references the binary glTF file.
* `shader` can have a `uri` (as usual) or a `bufferView` so the text can be extracted from the typed array (binary glTF).
* `image` can have a `uri` (as usual) or `bufferView`, `mimeType`, `width`, and `height`, which allows users to create a JavaScript image like [this](https://github.com/AnalyticalGraphicsInc/cesium/blob/2d4b2f8694d525e65a29b8b524b1c07b9abd609c/Source/Core/loadImageFromTypedArray.js).

**File size and number of files**

Tested with our [aircraft model](https://github.com/AnalyticalGraphicsInc/cesium/tree/master/Apps/SampleData/models/CesiumAir) (we need to test with me).

            | dae               | glTF              | glTF (base64-encoded bin/png/glsl) | Binary glTF
------------|-------------------|-------------------|------------------------------------|------------
size        | 1.07 MB (3 files) | 802 KB (8 files)  | 1.03 MB                            |  806 KB
size (gzip) | 728 KB            | 677 KB            | 706 KB                             |  707 KB

(Not impressed by gzip size for this example)

Binary glTF still supports external resources and embedded base64 ones.  For example, I think a common case will be to embed .bin and shaders into a binary glTF and then have external images.

Questions
* Do we want this to become core glTF?  Or a container format?
* Any recommended tweaks to the schema?

## Examples

TODO

## Known Implementations

* Cesium ([code](https://github.com/AnalyticalGraphicsInc/cesium/blob/bgltf/Source/Scene/Model.js))

## Resources

* Discussion - [#357](https://github.com/KhronosGroup/glTF/issues/357)
* base64-encoded data in glTF - [#68](https://github.com/KhronosGroup/glTF/issues/68)