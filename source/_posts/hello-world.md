---
title: TEST BLOG
---
Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

## Quick Start

### Create a new post

``` bash
$ hexo new "My New Post"
```

`More info`: [Writing](https://hexo.io/docs/writing.html)

### Run server

``` bash
$ hexo server
```

More info: [Server](https://hexo.io/docs/server.html)

### Generate static files

``` bash
$ hexo generate
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy
```

More info: [Deployment](https://hexo.io/docs/one-command-deployment.html)


```java
long currentUsed;
do {
    currentUsed = this.used.get();
    newUsed = currentUsed + bytes;
    long newUsedWithOverhead = (long)(newUsed * overheadConstant);
    if (logger.isTraceEnabled()) {
        logger.trace("Adding [{}][{}] to used bytes [new used: [{}], limit: {} [{}], estimate: {} [{}]]",
                new ByteSizeValue(bytes), label, new ByteSizeValue(newUsed),
                memoryBytesLimit, new ByteSizeValue(memoryBytesLimit),
                newUsedWithOverhead, new ByteSizeValue(newUsedWithOverhead));
    }
    if (memoryBytesLimit > 0 && newUsedWithOverhead > memoryBytesLimit) {
        logger.warn("New used memory {} [{}] from field [{}] would be larger than configured breaker: {} [{}], breaking",
                newUsedWithOverhead, new ByteSizeValue(newUsedWithOverhead), label,
                memoryBytesLimit, new ByteSizeValue(memoryBytesLimit));
        circuitBreak(label, newUsedWithOverhead);
    }
    // Attempt to set the new used value, but make sure it hasn't changed
    // underneath us, if it has, keep trying until we are able to set it
} while (!this.used.compareAndSet(currentUsed, newUsed));
```
