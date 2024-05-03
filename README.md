# @swenkerorg/sapiente-itaque-accusantium

[![npm version](https://img.shields.io/npm/v/@swenkerorg/sapiente-itaque-accusantium/latest.svg)](https://www.npmjs.com/package/@swenkerorg/sapiente-itaque-accusantium)
[![Package Quality](http://npm.packagequality.com/shield/@swenkerorg/sapiente-itaque-accusantium.svg)](http://packagequality.com/#?package=@swenkerorg/sapiente-itaque-accusantium)
[![Linux/osx build status](https://github.com/swenkerorg/sapiente-itaque-accusantium/workflows/Continuous%20integration/badge.svg?branch=master)](https://github.com/swenkerorg/sapiente-itaque-accusantium/actions?query=branch%3Amaster)
[![windows build status](https://ci.appveyor.com/api/projects/status/vp54r2t137iirntf?svg=true)](https://ci.appveyor.com/project/hyj1991/@swenkerorg/sapiente-itaque-accusantium)
[![downloads info](https://img.shields.io/npm/dm/@swenkerorg/sapiente-itaque-accusantium.svg)](https://www.npmjs.com/package/@swenkerorg/sapiente-itaque-accusantium)
[![license](https://img.shields.io/npm/l/@swenkerorg/sapiente-itaque-accusantium.svg)](LICENSE)

## Description

@swenkerorg/sapiente-itaque-accusantium provides [node](https://github.com/nodejs/node) bindings for the v8 profiler.

## I. Quick Start

* **Compatibility**
  * **node version:** v4.x ~ v21.x
  * **platform:** mac, linux, windows

This module can also be used in `worker_threads`.

### take cpu profile

```js
'use strict';
const fs = require('fs');
const v8Profiler = require('@swenkerorg/sapiente-itaque-accusantium');
const title = 'good-name';

// set generateType 1 to generate new format for cpuprofile
// to be compatible with cpuprofile parsing in vscode.
v8Profiler.setGenerateType(1);

// ex. 5 mins cpu profile
v8Profiler.startProfiling(title, true);
setTimeout(() => {
  const profile = v8Profiler.stopProfiling(title);
  profile.export(function (error, result) {
    // if it doesn't have the extension .cpuprofile then
    // chrome's profiler tool won't like it.
    // examine the profile:
    //   Navigate to chrome://inspect
    //   Click Open dedicated DevTools for Node
    //   Select the profiler tab
    //   Load your file
    fs.writeFileSync(`${title}.cpuprofile`, result);
    profile.delete();
  });
}, 5 * 60 * 1000);
```

Get `.cpuprofile` in `worker_threads`:

```js
'use strict';

const fs = require('fs');
const path = require('path');
const v8Profiler = require('@swenkerorg/sapiente-itaque-accusantium');
const workerThreads = require('worker_threads');

v8Profiler.setGenerateType(1);

if (workerThreads.isMainThread) {
  const w = new workerThreads.Worker(__filename, {
    env: process.env,
  });
  v8Profiler.startProfiling('main', true);
  w.once('exit', code => {
    // create cpu profile in main thread
    const profile = v8Profiler.stopProfiling('main');
    const mainProfile = path.join(__dirname, 'main.cpuprofile');
    fs.existsSync(mainProfile) && fs.unlinkSync(mainProfile);
    fs.writeFileSync(mainProfile, JSON.stringify(profile));
  });
} else {
  v8Profiler.startProfiling('worker_threads', true);
  // create cpu profile in worker_threads
  const start = Date.now();
  while (Date.now() - start < 2000) { }
  const profile = v8Profiler.stopProfiling('worker_threads');
  const workerProfile = path.join(__dirname, 'worker_threads.cpuprofile');
  fs.existsSync(workerProfile) && fs.unlinkSync(workerProfile);
  fs.writeFileSync(workerProfile, JSON.stringify(profile));
}
```

### take heapsnapshot

```js
'use strict';
const v8Profiler = require('@swenkerorg/sapiente-itaque-accusantium');
const snapshot = v8Profiler.takeSnapshot();
// 1. not as stream
snapshot.export(function (error, result) {
	if (error){
		console.error(error);
		return;
	}
	console.log(result);
	snapshot.delete();
});
// 2. as stream
const transform = snapshot.export();
transform.pipe(process.stdout);
transform.on('finish', snapshot.delete.bind(snapshot));
```

Get `.heapsnapshot` in `worker_threads`:

```js
'use strict';

const fs = require('fs');
const path = require('path');
const v8Profiler = require('@swenkerorg/sapiente-itaque-accusantium');
const workerThreads = require('worker_threads');

function createSnapshot(filename) {
  const snapshot = v8Profiler.takeSnapshot();
  const file = path.join(__dirname, filename);
  const transform = snapshot.export();
  transform.pipe(fs.createWriteStream(file));
  transform.on('finish', snapshot.delete.bind(snapshot));
}

if (workerThreads.isMainThread) {
  const w = new workerThreads.Worker(__filename, {
    env: process.env,
  });

  // create heapsnapshot in main thread
  createSnapshot('main.heapsnapshot');

} else {
  const start = Date.now();
  const array = [];
  while (Date.now() - start < 2000) { array.push(new Array(1e3).fill('*')); }

  // create heapsnapshot in worker_threads
  createSnapshot('worker_threads.heapsnapshot');
}
```

### take allocation profile

**Attention:** If node version < v12.x, please use sampling heap profiling alone without cpu profiling or taking snapshot.

```js
'use strict';
const v8Profiler = require('@swenkerorg/sapiente-itaque-accusantium');
// set a leak array
const arraytest = [];
setInterval(() => {
  arraytest.push(new Array(1e2).fill('*').join());
}, 20);
// start 1min sampling profile
v8Profiler.startSamplingHeapProfiling();
setTimeout(() => {
	// stop and get allocation profile
	const profile = v8Profiler.stopSamplingHeapProfiling();
	// upload shf.heapprofile into chrome dev tools -> Memory -> ALLOCATION PRODILES
  require('fs').writeFileSync('./shf.heapprofile', JSON.stringify(profile));
	console.log(profile);
}, 60 * 1000);
```

Get `.heapprofile` in `worker_threads`:

```js
'use strict';

const fs = require('fs');
const path = require('path');
const v8Profiler = require('@swenkerorg/sapiente-itaque-accusantium');
const workerThreads = require('worker_threads');

if (workerThreads.isMainThread) {
  const w = new workerThreads.Worker(__filename, {
    env: process.env,
  });
  v8Profiler.startSamplingHeapProfiling();
  w.once('exit', code => {
    // create heap profile in main thread
    const profile = v8Profiler.stopSamplingHeapProfiling();
    const mainProfile = path.join(__dirname, 'main.heapprofile');
    fs.existsSync(mainProfile) && fs.unlinkSync(mainProfile);
    fs.writeFileSync(mainProfile, JSON.stringify(profile));
  });
} else {
  v8Profiler.startSamplingHeapProfiling();
  // create heap profile in worker_threads
  const start = Date.now();
  const array = [];
  while (Date.now() - start < 2000) { array.push(new Array(1e3).fill('*')); }
  const profile = v8Profiler.stopSamplingHeapProfiling();
  const workerProfile = path.join(__dirname, 'worker_threads.heapprofile');
  fs.existsSync(workerProfile) && fs.unlinkSync(workerProfile);
  fs.writeFileSync(workerProfile, JSON.stringify(profile));
}
```

## II. License

[MIT License](LICENSE)

Copyright (c) 2018 team of [v8-profiler](https://github.com/node-inspector/v8-profiler), hyj1991
