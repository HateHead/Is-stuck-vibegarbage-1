# Is-stuck-vibegarbage-1
Is stuck vibe garbage 1 

I am not a nerd, this i vibegarbage and chatgpt tries to gaslight the hell out of me, maybe it helps someone, maybe it does qhat ais does - shit.


========================================
stillrunning — v0.1
========================================

PURPOSE
-------
Detect whether a file or a process is actually making progress.
Not “healthy”. Not “running”.
Just: did observable reality change?

This is a single-purpose watchdog.
CLI only. Deterministic. Scriptable.

----------------------------------------
STATES
----------------------------------------
progress : observable change detected
stalled  : target exists but no change for threshold
dead     : target missing / terminated

Exit codes:
0 = progress
1 = stalled
2 = dead

----------------------------------------
SUPPORTED TARGETS (v0.1)
----------------------------------------
--file <path>
--pid  <pid>

----------------------------------------
DEFAULTS
----------------------------------------
interval = 5 seconds
stall    = 60 seconds

----------------------------------------
DETECTION LOGIC
----------------------------------------

FILE TARGET
-----------
Each sample captures:
- file size
- mtime (nanoseconds)
- partial hash:
  - first 4 KB
  - last 4 KB (if file > 8 KB)

If ANY of these change → progress

PID TARGET
----------
Each sample reads:
- /proc/<pid>/stat
- utime + stime CPU ticks

If ticks increase → progress
If pid exists but ticks unchanged → stalled
If /proc/<pid> missing → dead

----------------------------------------
OUTPUT (ONE LINE ONLY)
----------------------------------------

Examples:

target=file:/var/log/backup.log state=progress delta=+8192B
target=pid:3812 state=stalled for=62s
target=pid:3812 state=dead

----------------------------------------
BUILD
----------------------------------------

go build -o stillrunning stillrunning.go

----------------------------------------
USAGE
----------------------------------------

stillrunning --file /path/to/file
stillrunning --pid 3812
stillrunning --file data.bin --interval 2 --stall 30

----------------------------------------
IMPLEMENTATION (single file)
----------------------------------------


package main

import (
	"crypto/sha256"
	"encoding/hex"
	"flag"
	"fmt"
	"io"
	"os"
	"strconv"
	"strings"
	"time"
)

type FileSample struct {
	Size  int64
	Mtime int64
	Hash  string
}

func sampleFile(path string) (*FileSample, error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, err
	}
	defer f.Close()

	info, err := f.Stat()
	if err != nil {
		return nil, err
	}

	h := sha256.New()
	buf := make([]byte, 4096)

	if info.Size() > 0 {
		n, _ := f.Read(buf)
		h.Write(buf[:n])
	}

	if info.Size() > int64(len(buf)) {
		f.Seek(info.Size()-int64(len(buf)), io.SeekStart)
		n, _ := f.Read(buf)
		h.Write(buf[:n])
	}

	return &FileSample{
		Size:  info.Size(),
		Mtime: info.ModTime().UnixNano(),
		Hash:  hex.EncodeToString(h.Sum(nil)),
	}, nil
}

func fileChanged(a, b *FileSample) bool {
	return a.Size != b.Size || a.Mtime != b.Mtime || a.Hash != b.Hash
}

func samplePID(pid int) (int64, error) {
	data, err := os.ReadFile(fmt.Sprintf("/proc/%d/stat", pid))
	if err != nil {
		return 0, err
	}
	fields := strings.Fields(string(data))
	if len(fields) < 17 {
		return 0, fmt.Errorf("invalid stat")
	}
	utime, _ := strconv.ParseInt(fields[13], 10, 64)
	stime, _ := strconv.ParseInt(fields[14], 10, 64)
	return utime + stime, nil
}

func main() {
	filePath := flag.String("file", "", "")
	pid := flag.Int("pid", 0, "")
	interval := flag.Int("interval", 5, "")
	stall := flag.Int("stall", 60, "")
	flag.Parse()

	start := time.Now()

	if *filePath != "" {
		prev, err := sampleFile(*filePath)
		if err != nil {
			fmt.Printf("target=file:%s state=dead\n", *filePath)
			os.Exit(2)
		}

		for {
			time.Sleep(time.Duration(*interval) * time.Second)
			cur, err := sampleFile(*filePath)
			if err != nil {
				fmt.Printf("target=file:%s state=dead\n", *filePath)
				os.Exit(2)
			}

			if fileChanged(prev, cur) {
				fmt.Printf(
					"target=file:%s state=progress delta=%dB\n",
					*filePath,
					cur.Size-prev.Size,
				)
				os.Exit(0)
			}

			if time.Since(start) > time.Duration(*stall)*time.Second {
				fmt.Printf(
					"target=file:%s state=stalled for=%ds\n",
					*filePath,
					int(time.Since(start).Seconds()),
				)
				os.Exit(1)
			}
		}
	}

	if *pid != 0 {
		prev, err := samplePID(*pid)
		if err != nil {
			fmt.Printf("target=pid:%d state=dead\n", *pid)
			os.Exit(2)
		}

		for {
			time.Sleep(time.Duration(*interval) * time.Second)
			cur, err := samplePID(*pid)
			if err != nil {
				fmt.Printf("target=pid:%d state=dead\n", *pid)
				os.Exit(2)
			}

			if cur > prev {
				fmt.Printf(
					"target=pid:%d state=progress ticks=+%d\n",
					*pid,
					cur-prev,
				)
				os.Exit(0)
			}

			if time.Since(start) > time.Duration(*stall)*time.Second {
				fmt.Printf(
					"target=pid:%d state=stalled for=%ds\n",
					*pid,
					int(time.Since(start).Seconds()),
				)
				os.Exit(1)
			}
		}
	}

	fmt.Println("no target specified")
	os.Exit(2)
}

----------------------------------------
DESIGN RULES (INTENTIONAL)
--------------------------------
