# file_integrity_watcher.py
# Usage: python file_integrity_watcher.py /path/to/watch
# NOTE: run this only on directories you own and control.

import os
import sys
import time
import hashlib

WATCH_INTERVAL = 5  # seconds

def hash_file(path):
    h = hashlib.sha256()
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            h.update(chunk)
    return h.hexdigest()

def snapshot_dir(root):
    snap = {}
    for dirpath, _, filenames in os.walk(root):
        for fname in filenames:
            full = os.path.join(dirpath, fname)
            try:
                snap[full] = (os.path.getmtime(full), os.path.getsize(full), hash_file(full))
            except (PermissionError, OSError):
                # skip files we can't access
                continue
    return snap

def compare_snapshots(old, new):
    old_keys = set(old.keys())
    new_keys = set(new.keys())

    added = new_keys - old_keys
    removed = old_keys - new_keys
    changed = {p for p in (old_keys & new_keys) if old[p][2] != new[p][2]}

    return added, removed, changed

def main():
    if len(sys.argv) < 2:
        print("Usage: python file_integrity_watcher.py /path/to/watch")
        return
    root = sys.argv[1]
    if not os.path.isdir(root):
        print("Not a directory:", root); return

    print("Taking initial snapshot of", root)
    prev = snapshot_dir(root)
    print(f"Watching {len(prev)} files. Press Ctrl+C to stop.")
    try:
        while True:
            time.sleep(WATCH_INTERVAL)
            curr = snapshot_dir(root)
            added, removed, changed = compare_snapshots(prev, curr)
            if added or removed or changed:
                ts = time.strftime("%Y-%m-%d %H:%M:%S")
                print(f"\n[{ts}] Changes detected:")
                if added:
                    for p in sorted(added):
                        print("  + added:  ", p)
                if removed:
                    for p in sorted(removed):
                        print("  - removed:", p)
                if changed:
                    for p in sorted(changed):
                        print("  * changed:", p)
                prev = curr
    except KeyboardInterrupt:
        print("\nStopped.")

if __name__ == "__main__":
    main()
