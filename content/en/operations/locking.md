# One image, several protocols, one lock

Every server in this family serialises the driver itself, because
[`go-filesystems/interface`](https://github.com/go-filesystems/interface)
promises nothing about concurrent path-based calls — and each of them holds
**its own** lock, knowing nothing about the others.

Serving one image over three protocols at once would therefore have no common
lock anywhere. So the image is wrapped **once**, here, and the same wrapper is
handed to all of them.

## Stated as measured, so as not to overstate it

Eight goroutines writing through an **unwrapped** fat32 driver under `-race`
produce **no race today**.

This is insurance on a contract, not a reproduction of a defect — ext4 and ntfs
are not fat32, and a driver update is not a thing this program should have to
re-audit.

## The wrapper carries capabilities through

It does not hide them:

- a driver that answers `Opener` gets a **locked `File`** back;
- one whose `File` answers `WritableFile` keeps its **positional writes**.

The difference between those and the whole-file fallback was measured at
**~70×** elsewhere in the family, so a wrapper that erased them would be a
performance defect disguised as safety.

## Under `--isolate` the lock cannot help

A child process opens the image itself, so there is no shared wrapper between
children. That is why an image served writable by two protocols is
[refused](isolation.md#the-rule-that-makes-it-honest) rather than quietly
served without a lock.
