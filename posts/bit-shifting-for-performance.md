---
title: Bit shifting to increase performance
date: 15/09/2024
tags: [ js, performance ]
---

I've been watching [@Vercidium](https://www.youtube.com/@Vercidium)'s awesome content on YouTube. His videos are all
about performance optimization in regards to game development, and I learned a nifty performance optimization for my
JS code as a result.

<!-- more -->

In his ["I Optimised My Game Engine Up To 12000 FPS"](https://youtu.be/40JzyaOYJeY?si=5J97q3bm7MsJ3I6k) video Vercidium
shows a _bit-shifting_ technique which he has used to store information about his voxel based scene. This got me
thinking _could the same technique be used to save memory in JS code?_.

<magpie-trinket>**TLDR;** Storing a vector (x, y, z) as a 32-bit number uses a seventh of the memory compared to storing it as an object in this example.</magpie-trinket>

<table class="pure-table">
    <thead>
        <tr>
            <th>Metric</th>
            <th>Object Based</th>
            <th>Bit Shifted</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>RSS</td>
            <td>646.12 MB</td>
            <td>132.29 MB</td>
        </tr>
        <tr>
            <td>Heap Total</td>
            <td>576.83 MB</td>
            <td>87.58 MB</td>
        </tr>
        <tr>
            <td>Heap Used</td>
            <td>545.67 MB</td>
            <td>82.33 MB</td>
        </tr>
        <tr>
            <td>External</td>
            <td>0.98 MB</td>
            <td>0.98 MB</td>
        </tr>
        <tr>
            <td>Array Buffers</td>
            <td>0.01 MB</td>
            <td>0.01 MB</td>
        </tr>
    </tbody>
</table>
</magpie-trinket>

## The Plan

When creating games and rendering scenes on canvas I often start by creating a `Vector` class like the one below.

```javascript
class Vector3 {
    x = 0
    y = 0
    z = 0

    constructor(x, y, z) {
        this.x = x
        this.y = y
        this.z = z
    }
}
```

You'll see varying implementations of this in pretty much every game engine. It's used to define where something
exists with in the game world. In my JS games I'll sometimes create tens of thousands of these at a time. I've often
thought about the performance impact of this, but never really came up with a good solution. The games I make are
often rudimentary in nature.

So let's look at the amount of memory used by creating and modifying such objects. To record the memory usage I have
written the following method. Which breaks down the results as follows:

- rss: Resident Set Size – total memory allocated for the process.
- heapTotal: Total size of the allocated heap.
- heapUsed: Actual memory used by the program.
- external: Memory used by C++ objects bound to JavaScript objects.
- arrayBuffers: Memory allocated for ArrayBuffers and SharedArrayBuffers.

```javascript
function logMemoryUsage() {
    const memoryUsage = process.memoryUsage();

    console.log(`Memory Usage:
        RSS: ${memoryUsage.rss / 1024 / 1024} MB
        Heap Total: ${memoryUsage.heapTotal / 1024 / 1024} MB
        Heap Used: ${memoryUsage.heapUsed / 1024 / 1024} MB
        External: ${memoryUsage.external / 1024 / 1024} MB
        Array Buffers: ${memoryUsage.arrayBuffers / 1024 / 1024} MB
    `);
}
```

The below script creates **10 million** `Vector3` objects and updates the `x` and `y` co-ordinates of each randomly
every
millisecond for 2 seconds.

```javascript
class Vector3 {
    x = 0
    y = 0
    z = 0

    constructor(x, y, z) {
        this.x = x
        this.y = y
        this.z = z
    }
}

function randomNumber(min, max) {
    return Math.floor(Math.random() * max) + min;
}

const maxVectors = 10_000_000
const vectors = new Array(maxVectors)

for (let i = 0; i < maxVectors; i++) {
    vectors[i] = new Vector3(
        randomNumber(0, 1000),
        randomNumber(0, 1000),
        randomNumber(0, 1000)
    )
}

let demoLengthInSeconds = 2;

const updateInterval = setInterval(() => {
    for (const v of vectors) {
        v.x += randomNumber(-2, 2)
        v.y += randomNumber(-2, 2)
    }
}, 1);

setTimeout(() => {
    logMemoryUsage()
    clearInterval(updateInterval)
}, demoLengthInSeconds * 1000)
```

For the purposes of this article I will run the script with node. Which gives the following results.

```
Memory Usage:
        RSS: 646.12109375 MB
        Heap Total: 576.828125 MB
        Heap Used: 545.6746520996094 MB
        External: 0.9788284301757812 MB
        Array Buffers: 0.009958267211914062 MB
```

Using the above object based implementation as baseline, let's now look at the bit-shifter variant.

## The Bit Shifter Implementation

The bit-shifting implementation is going to work along the same lines. However, some boilerplate code is needed to `pack`
and `unpack` our vector's values. Essentially what we are doing with the `pack` method is a mask and shift of `x`, `y`,
and `z` to pack them efficiently into a single 32-bit number. The idea here being that a 32-bit number will be stored
with less memory than the class used in the object based run.

Putting these concepts into practice we have the below script.

```javascript
function pack(n, offset) {
    if (!offset) return (n & 0x3FF);
    return (n & 0x3FF) << offset;
}

function Vector3(x, y, z) {
    return (pack(x, 20) | pack(y, 10) | pack(z));
}

function unpack(packed, offset) {
    if (!offset) return packed & 0x3FF;
    return (packed >> offset) & 0x3FF;
}

function getX(vector3) {
    return unpack(vector3, 20);
}

function getY(vector3) {
    return unpack(vector3, 10);
}

function getZ(vector3) {
    return unpack(vector3);
}

function setX(vector3, x) {
    vector3 &= ~(0x3FF << 20);
    vector3 |= pack(x, 20);
    return vector3;
}

function setY(vector3, y) {
    vector3 &= ~(0x3FF << 10);
    vector3 |= pack(y, 10);
    return vector3;
}

function setZ(vector3, z) {
    vector3 &= ~0x3FF;
    vector3 |= pack(z);
    return vector3;
}

function randomNumber(min, max) {
    return Math.floor(Math.random() * max) + min;
}

const maxVectors = 10_000_000
const vectors = new Array(maxVectors)

for (let i = 0; i < maxVectors; i++) {
    vectors[i] = Vector3(
        randomNumber(0, 1000),
        randomNumber(0, 1000),
        randomNumber(0, 1000)
    )
}

let demoLengthInSeconds = 2;

const updateInterval = setInterval(() => {
    for (const v of vectors) {
        setX(v, getX(v) + randomNumber(-2, 2))
        setY(v, getY(v) + randomNumber(-2, 2))
    }
}, 1);

setTimeout(() => {
    logMemoryUsage()
    clearInterval(updateInterval)
}, demoLengthInSeconds * 1000)
```

<chicken-asks>
Will the calls to `pack` and `unpack` affect the performance?
</chicken-asks>

<magpie-replies>
Good question, there may be a more efficient way of doing this. But in regard to this test, the impact of calling `pack`
and `unpack` seem to be negligible.
</magpie-replies>

Again this is run with node, `v18.19.0`. The resulting memory usage is as follows.

```
Memory Usage:
        RSS: 132.28515625 MB
        Heap Total: 87.578125 MB
        Heap Used: 82.33492279052734 MB
        External: 0.9788284301757812 MB
        Array Buffers: 0.009958267211914062 MB
```

## The Results

<table class="pure-table">
    <thead>
        <tr>
            <th>Metric</th>
            <th>Object Based</th>
            <th>Bit Shifted</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>RSS</td>
            <td>646.12 MB</td>
            <td>132.29 MB</td>
        </tr>
        <tr>
            <td>Heap Total</td>
            <td>576.83 MB</td>
            <td>87.58 MB</td>
        </tr>
        <tr>
            <td>Heap Used</td>
            <td>545.67 MB</td>
            <td>82.33 MB</td>
        </tr>
        <tr>
            <td>External</td>
            <td>0.98 MB</td>
            <td>0.98 MB</td>
        </tr>
        <tr>
            <td>Array Buffers</td>
            <td>0.01 MB</td>
            <td>0.01 MB</td>
        </tr>
    </tbody>
</table>

As we can see from the above table the object based implementation used around **7 times** more memory than the bit
shifted example.

When it comes to creating my next game I will certainly be using this technique. While it does require extra code, the
code required is still fairly minimal. Especially seeing as it uses a seventh of the memory.
