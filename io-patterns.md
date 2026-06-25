> Everything in Go is based on I/O interfaces - This guide will teach you how Go's elegant I/O system works from the ground up.

## Quick Links
- [Concurrency Guide](Concurrency.md)
- [Table of Contents](#table-of-contents)
- [Core Philosophy](#core-philosophy)
- [Foundation Interfaces](#foundation-interfaces)
- [Real-World Examples](#real-world-examples)
- [Best Practices](#best-practices)

---

## Table of Contents

**Quick Navigation:**
- [Core Philosophy](#core-philosophy) - Go's I/O design principles
- [Foundation Interfaces](#foundation-interfaces) - io.Reader & io.Writer
- [Interface Hierarchy](#the-interface-hierarchy) - Complete interface family
- [Common Implementations](#common-implementations) - Files, buffers, network
- [Data Flow Patterns](#data-flow-patterns) - Composition & chaining
- [Utility Functions](#utility-functions) - MultiReader, MultiWriter, etc.
- [Buffered I/O](#buffered-io) - bufio package
- [Real-World Examples](#real-world-examples) - Practical applications
- [Advanced Patterns](#advanced-patterns) - LimitReader, SectionReader
- [Performance Guide](#performance-guide) - Optimization tips
- [Best Practices](#best-practices) - Common pitfalls & solutions

1. Core Philosophy
2. Foundation Interfaces
3. The Interface Hierarchy
4. Common Implementations
5. Data Flow Patterns
6. Utility Functions
7. Buffered I/O
8. Real-World Examples
9. Advanced Patterns
10. Performance Guide
11. Best Practices

---

## Core Philosophy

Go's I/O model is built on a simple but powerful principle: **abstract data sources and destinations as streams that implement standard interfaces**.

### Why This Matters

```
Traditional Approach (Language Specific APIs):
- File API for files
- Socket API for network
- Buffer API for memory
- Different functions for each

Go's Approach (Unified Interfaces):
- io.Reader for reading from ANY source
- io.Writer for writing to ANY destination
- Same functions work everywhere

```

### The Big Picture

```
┌─────────────────────────────────────────────────────────┐
│                    YOUR APPLICATION                      │
│                                                          │
│    Uses io.Reader & io.Writer interfaces only          │
└────────────────┬────────────────────────────────────────┘
                 │
                 │ Standard interfaces
                 │
┌────────────────┴────────────────────────────────────────┐
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │  Files   │  │ Network  │  │  Memory  │  │ Pipes  │ │
│  │  (disk)  │  │ (TCP/UDP)│  │ (buffer) │  │ (IPC)  │ │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘ │
│                                                          │
│       All implement io.Reader & io.Writer                │
└─────────────────────────────────────────────────────────┘

```

---

## Foundation Interfaces

### io.Reader - The Heart of Input

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

```

### How io.Reader Works

```
┌─────────────────────────────────────────────────────┐
│ 1. You provide a byte slice (buffer)                │
│    buffer := make([]byte, 1024)                     │
└───────────────────┬─────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────┐
│ 2. Call Read() on any Reader                        │
│    n, err := reader.Read(buffer)                    │
└───────────────────┬─────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────┐
│ 3. Reader fills your buffer with data               │
│    buffer now contains: [H][e][l][l][o][...][0][0] │
│                         ← n bytes →                 │
└───────────────────┬─────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────┐
│ 4. You process the data: buffer[:n]                 │
│    Returns: n = number of bytes read                │
│             err = error or io.EOF when done         │
└─────────────────────────────────────────────────────┘

```

### Complete Reading Pattern

```go
func readExample() {
    file, err := os.Open("data.txt")
    if err != nil {
        log.Fatal(err)
    }
    defer file.Close()

    // Create buffer
    buffer := make([]byte, 1024)

    // Read loop
    for {
        n, err := file.Read(buffer)
        if err == io.EOF {
            break // Done reading
        }
        if err != nil {
            log.Fatal(err)
        }

        // Process the n bytes we read
        fmt.Printf("Read %d bytes: %s\n", n, buffer[:n])
    }
}

```

> 💡 Key Point: The caller provides the buffer. This allows for buffer reuse and memory control.
>

---

### io.Writer - The Heart of Output

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}

```

### How io.Writer Works

```
┌─────────────────────────────────────────────────────┐
│ 1. You have data to write                           │
│    data := []byte("Hello, World!")                  │
└───────────────────┬─────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────┐
│ 2. Call Write() on any Writer                       │
│    n, err := writer.Write(data)                     │
└───────────────────┬─────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────┐
│ 3. Writer consumes your data and writes it          │
│    Could be: file, network, memory, stdout, etc.    │
└───────────────────┬─────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────┐
│ 4. Returns: n = bytes written (usually len(data))   │
│             err = any error encountered             │
└─────────────────────────────────────────────────────┘

```

### Complete Writing Pattern

```go
func writeExample() {
    file, err := os.Create("output.txt")
    if err != nil {
        log.Fatal(err)
    }
    defer file.Close()

    data := []byte("Hello, World!\n")

    n, err := file.Write(data)
    if err != nil {
        log.Fatal(err)
    }

    fmt.Printf("Wrote %d bytes\n", n)
}

```

> ⚠️ Important: Write may write fewer bytes than requested. Always check the returned n value.
>

---

## The Interface Hierarchy

### Complete Interface Family Tree

```
                    ┌──────────────┐
                    │  io.Reader   │
                    └──────┬───────┘
                           │
        ┏━━━━━━━━━━━━━━━━━━┻━━━━━━━━━━━━━━━━━━┓
        ┃                                      ┃
┌───────▼────────┐                    ┌────────▼────────┐
│ io.ReadCloser  │                    │ io.ReadSeeker   │
│                │                    │                 │
│ + Close()      │                    │ + Seek()        │
└────────────────┘                    └─────────────────┘

                    ┌──────────────┐
                    │  io.Writer   │
                    └──────┬───────┘
                           │
        ┏━━━━━━━━━━━━━━━━━━┻━━━━━━━━━━━━━━━━━━┓
        ┃                                      ┃
┌───────▼────────┐                    ┌────────▼────────┐
│ io.WriteCloser │                    │ io.WriteSeeker  │
│                │                    │                 │
│ + Close()      │                    │ + Seek()        │
└────────────────┘                    └─────────────────┘

            ┌──────────────┐
            │ io.ReadWriter│
            │              │
            │ Read + Write │
            └──────┬───────┘
                   │
        ┌──────────▼───────────┐
        │ io.ReadWriteCloser   │
        │                      │
        │ Read+Write+Close     │
        └──────────────────────┘

```

### Core Interface Definitions

```go
// Basic I/O
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Combine reading and writing
type ReadWriter interface {
    Reader
    Writer
}

// Resource management
type Closer interface {
    Close() error
}

type ReadCloser interface {
    Reader
    Closer
}

type WriteCloser interface {
    Writer
    Closer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}

// Random access
type Seeker interface {
    Seek(offset int64, whence int) (int64, error)
}

type ReadSeeker interface {
    Reader
    Seeker
}

type WriteSeeker interface {
    Writer
    Seeker
}

// Position-based I/O
type ReaderAt interface {
    ReadAt(p []byte, off int64) (n int, err error)
}

type WriterAt interface {
    WriteAt(p []byte, off int64) (n int, err error)
}

```

---

## Common Implementations

### Type Implementation Matrix

| Type | Reader | Writer | Closer | Seeker | ReaderAt | WriterAt |
| --- | --- | --- | --- | --- | --- | --- |
| `os.File` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `bytes.Buffer` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `bytes.Reader` | ✅ | ❌ | ❌ | ✅ | ✅ | ❌ |
| `strings.Reader` | ✅ | ❌ | ❌ | ✅ | ✅ | ❌ |
| `net.Conn` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `bufio.Reader` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `bufio.Writer` | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `io.Pipe` | ✅/✅ | ✅/✅ | ✅ | ❌ | ❌ | ❌ |

### File System: os.File

```go
// os.File is the most complete implementation
file, err := os.Open("data.txt")     // ReadOnly
file, err := os.Create("data.txt")   // WriteOnly, truncates
file, err := os.OpenFile("data.txt", os.O_RDWR|os.O_CREATE, 0644)

// Implements everything:
file.Read(buffer)            // io.Reader
file.Write(data)             // io.Writer
file.Seek(0, io.SeekStart)   // io.Seeker
file.ReadAt(buf, 100)        // io.ReaderAt
file.WriteAt(data, 200)      // io.WriterAt
file.Close()                 // io.Closer

```

**Visual: os.File Operations**

```
File on Disk: [HELLO WORLD EXAMPLE DATA...]
              ↑       ↑               ↑
Position:     0      10              20

file.Read(buf)        → reads from current position, advances it
file.ReadAt(buf, 10)  → reads from position 10, doesn't move current
file.Seek(5, 0)       → moves current position to 5
file.Write(data)      → writes at current position, advances it
file.WriteAt(data,15) → writes at position 15, doesn't move current

```

---

### Memory: bytes.Buffer

```go
var buf bytes.Buffer

// Writing
buf.Write([]byte("Hello"))
buf.WriteString(" World")
buf.WriteByte('!')

// Reading
data := buf.Bytes()      // Get all data
str := buf.String()      // Get as string

// Read and consume
p := make([]byte, 5)
n, _ := buf.Read(p)      // Reads and removes from buffer

```

**Visual: Buffer Behavior**

```
Initial:    []                          (empty)
                                        read pos: 0, write pos: 0

Write("Hi"):  [H][i]                    write pos: 2
              ↑
              read pos: 0

Read(1):      [H][i]                    read 'H', consumed
                 ↑
                 read pos: 1

Write("!"):   [H][i][!]                 write pos: 3
                 ↑
                 read pos: 1

Bytes():      [H][i][!]                 get all (doesn't consume)

```

---

### Memory: bytes.Reader and strings.Reader

```go
// Immutable, seekable readers
data := []byte("Hello World")
reader := bytes.NewReader(data)

// Reading
buf := make([]byte, 5)
reader.Read(buf)           // "Hello"

// Seeking
reader.Seek(6, io.SeekStart)  // Go to position 6
reader.Read(buf)           // "World"

// Random access
reader.ReadAt(buf[:3], 0)  // "Hel" without moving position

// Size and position
size := reader.Size()      // Total size
pos, _ := reader.Seek(0, io.SeekCurrent)  // Current position

```

---

### Network: net.Conn

```go
conn, err := net.Dial("tcp", "example.com:80")
if err != nil {
    log.Fatal(err)
}
defer conn.Close()

// Write HTTP request
fmt.Fprintf(conn, "GET / HTTP/1.0\r\n\r\n")

// Read response
buffer := make([]byte, 1024)
n, err := conn.Read(buffer)

```

**Visual: Network I/O**

```
┌──────────────┐                           ┌──────────────┐
│  Your App    │                           │ Remote Server│
│              │                           │              │
│ conn.Write() │───── TCP Packet ────────▶│              │
│              │      [HTTP Request]       │              │
│              │                           │              │
│              │◀──── TCP Packet ──────────│ Sends Data   │
│  conn.Read() │      [HTTP Response]      │              │
│              │                           │              │
└──────────────┘                           └──────────────┘

Connection is bidirectional: both Read and Write work

```

---

## Data Flow Patterns

### Pattern 1: Direct Copy with io.Copy

```go
func io.Copy(dst Writer, src Reader) (written int64, err error)

```

**How it works internally:**

```
┌──────────────────────────────────────────────────────────┐
│  io.Copy Implementation (simplified)                     │
│                                                          │
│  1. Create internal buffer: buf := make([]byte, 32KB)   │
│  2. Loop:                                                │
│     a. src.Read(buf)      → fill buffer                  │
│     b. dst.Write(buf[:n]) → write what we read          │
│  3. Repeat until src returns io.EOF                      │
└──────────────────────────────────────────────────────────┘

Visual Flow:

[Source]  ─Read→  [32KB Buffer]  ─Write→  [Destination]
   ↑                                            │
   └────────────── Loop ─────────────────────────┘

```

**Practical Examples:**

```go
// Copy file to file
func copyFile(src, dst string) error {
    source, err := os.Open(src)
    if err != nil {
        return err
    }
    defer source.Close()

    destination, err := os.Create(dst)
    if err != nil {
        return err
    }
    defer destination.Close()

    _, err = io.Copy(destination, source)
    return err
}

// Download file from URL
func downloadFile(url, filepath string) error {
    resp, err := http.Get(url)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    out, err := os.Create(filepath)
    if err != nil {
        return err
    }
    defer out.Close()

    _, err = io.Copy(out, resp.Body)
    return err
}

// Copy to stdout
func printFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer file.Close()

    _, err = io.Copy(os.Stdout, file)
    return err
}

```

---

### Pattern 2: Chaining/Composition (The Power Pattern!)

This is where Go's I/O system truly shines. You can wrap readers and writers to add functionality.

```
┌─────────┐    ┌──────────┐    ┌─────────┐    ┌──────────┐
│  Source │───▶│ Wrapper1 │───▶│Wrapper2 │───▶│ Consumer │
│ Reader  │    │  Reader  │    │ Reader  │    │          │
└─────────┘    └──────────┘    └─────────┘    └──────────┘
   ↑               ↑                ↑               ↑
   │               │                │               │
 Raw Data    Adds Feature1    Adds Feature2    Your Code

```

### Example 1: Compressed JSON File

```go
func readCompressedJSON(filename string) (*MyData, error) {
    // 1. Open file
    file, err := os.Open(filename)
    if err != nil {
        return nil, err
    }
    defer file.Close()

    // 2. Add gzip decompression
    gzReader, err := gzip.NewReader(file)
    if err != nil {
        return nil, err
    }
    defer gzReader.Close()

    // 3. Add JSON decoding
    decoder := json.NewDecoder(gzReader)

    // 4. Decode into struct
    var data MyData
    err = decoder.Decode(&data)
    return &data, err
}

```

**Visual Flow:**

```
┌──────────────┐
│ data.json.gz │  File on disk (compressed)
└──────┬───────┘
       │ os.Open() → returns io.ReadCloser
       ↓
┌──────────────┐
│   os.File    │  Raw bytes from disk
└──────┬───────┘
       │ gzip.NewReader() → wraps file
       ↓
┌──────────────┐
│ gzip.Reader  │  Decompresses bytes on-the-fly
└──────┬───────┘  Still looks like io.Reader!
       │ json.NewDecoder() → wraps gzReader
       ↓
┌──────────────┐
│json.Decoder  │  Parses JSON from decompressed bytes
└──────┬───────┘
       │ Decode()
       ↓
┌──────────────┐
│   MyData     │  Your struct populated with data
└──────────────┘

Each layer only knows about the layer directly above it!

```

### Example 2: Writing Compressed Encrypted Data

```go
func writeEncryptedCompressed(filename string, data []byte) error {
    // 1. Create output file
    file, err := os.Create(filename)
    if err != nil {
        return err
    }
    defer file.Close()

    // 2. Add encryption
    block, _ := aes.NewCipher(key)
    stream := cipher.NewCFBEncrypter(block, iv)
    encWriter := &cipher.StreamWriter{S: stream, W: file}

    // 3. Add compression
    gzWriter := gzip.NewWriter(encWriter)
    defer gzWriter.Close()

    // 4. Write data (will be compressed, then encrypted, then written)
    _, err = gzWriter.Write(data)
    return err
}

```

**Visual Flow (Writing):**

```
Your Data: []byte("sensitive information")
    │
    │ Write()
    ↓
┌──────────────┐
│ gzip.Writer  │  Compresses data
└──────┬───────┘
       │ compressed bytes
       ↓
┌──────────────┐
│cipher.Writer │  Encrypts compressed data
└──────┬───────┘
       │ encrypted bytes
       ↓
┌──────────────┐
│   os.File    │  Writes to disk
└──────┬───────┘
       ↓
  data.bin.gz.enc on disk

```

---

## Utility Functions

### io.MultiReader - Sequential Reading

```go
r1 := strings.NewReader("Hello ")
r2 := strings.NewReader("World")
r3 := strings.NewReader("!")

mr := io.MultiReader(r1, r2, r3)

// Read from all three in sequence
io.Copy(os.Stdout, mr)  // Output: "Hello World!"

```

**Visual:**

```
┌─────────┐
│ Reader1 │──┐
│"Hello " │  │
└─────────┘  │
             │    ┌──────────────┐      Read()     ┌──────────┐
┌─────────┐  ├───▶│ MultiReader  │──────────────▶ │ Consumer │
│ Reader2 │  │    └──────────────┘  Sequential    └──────────┘
│"World"  │  │         reads all
└─────────┘  │    in order until EOF
             │
┌─────────┐  │
│ Reader3 │──┘
│   "!"   │
└─────────┘

Process:
1. Read from Reader1 until EOF
2. Move to Reader2, read until EOF
3. Move to Reader3, read until EOF
4. Return EOF to consumer

```

**Use Case: Combining Configuration Sources**

```go
func loadConfig() ([]byte, error) {
    // Combine: defaults + user config + environment overrides
    defaults := strings.NewReader(`{"timeout": 30}`)
    userConfig, _ := os.Open("user.json")
    envConfig := strings.NewReader(os.Getenv("APP_CONFIG"))

    combined := io.MultiReader(defaults, userConfig, envConfig)
    return io.ReadAll(combined)
}

```

---

### io.MultiWriter - Fan-Out Writing

```go
w1 := &bytes.Buffer{}
w2 := &bytes.Buffer{}
w3 := &bytes.Buffer{}

mw := io.MultiWriter(w1, w2, w3)

mw.Write([]byte("broadcast"))
// All three buffers now contain "broadcast"

```

**Visual:**

```
                          ┌──────────┐
                    ┌────▶│ Writer1  │─▶ File
                    │     └──────────┘
Input Data  ──Write─┤     ┌──────────┐
  "Hello"           ├────▶│ Writer2  │─▶ Network
                    │     └──────────┘
                    │     ┌──────────┐
                    └────▶│ Writer3  │─▶ Logger
                          └──────────┘

Each Write() call goes to ALL writers

```

**Use Case: Logging While Processing**

```go
func processWithLogging(data []byte) error {
    // Write to file AND log at the same time
    file, _ := os.Create("output.dat")
    defer file.Close()

    logFile, _ := os.Create("processing.log")
    defer logFile.Close()

    // Every write goes to both
    mw := io.MultiWriter(file, logFile, os.Stdout)

    _, err := mw.Write(data)
    return err
}

```

---

### io.TeeReader - Tap Into Stream

```go
r := strings.NewReader("Hello World")
var buf bytes.Buffer

// Create tee: reads from r, copies to buf
tee := io.TeeReader(r, &buf)

// Read from tee
data, _ := io.ReadAll(tee)

// Now both 'data' and 'buf' contain "Hello World"
fmt.Println(string(data))  // Hello World
fmt.Println(buf.String())  // Hello World

```

**Visual:**

```
                    ┌──────────────────┐
                    │   Source Reader  │
                    └────────┬─────────┘
                             │
                     Read from source
                             │
                  ┌──────────▼──────────┐
                  │    io.TeeReader     │
                  │                     │
                  │  Duplicates data    │
                  └──┬──────────────┬───┘
                     │              │
         Return to   │              │  Write to
           caller    │              │  secondary writer
                     ↓              ↓
               [Your Code]    [Log/Backup/etc]

```

**Use Case: Download with Progress**

```go
func downloadWithProgress(url string, dest string) error {
    resp, _ := http.Get(url)
    defer resp.Body.Close()

    out, _ := os.Create(dest)
    defer out.Close()

    // Create progress tracker
    progress := &progressWriter{total: resp.ContentLength}

    // Tee: data goes to file AND progress tracker
    tee := io.TeeReader(resp.Body, progress)

    io.Copy(out, tee)  // Download and track progress simultaneously
    return nil
}

type progressWriter struct {
    total   int64
    written int64
}

func (pw *progressWriter) Write(p []byte) (int, error) {
    n := len(p)
    pw.written += int64(n)
    fmt.Printf("\rDownloading: %d%%", pw.written*100/pw.total)
    return n, nil
}

```

---

### io.Pipe - In-Memory Connection

```go
pr, pw := io.Pipe()
// pr = pipe reader (io.Reader)
// pw = pipe writer (io.Writer)

```

**Visual:**

```
┌────────────────┐                      ┌────────────────┐
│  Goroutine 1   │                      │  Goroutine 2   │
│                │                      │                │
│  pw.Write()───────── data ──────────▶│   pr.Read()   │
│                │                      │                │
│  blocks until  │      In-Memory      │  blocks until  │
│  data is read  │       Buffer        │  data written  │
└────────────────┘                      └────────────────┘

Key: Synchronous! Writer blocks until reader reads.

```

**Use Case: Streaming Data Generation**

```go
func generateAndCompress() error {
    pr, pw := io.Pipe()

    // Goroutine 1: Generate data
    go func() {
        defer pw.Close()
        for i := 0; i < 1000; i++ {
            fmt.Fprintf(pw, "Line %d\n", i)
        }
    }()

    // Goroutine 2: Compress and save
    file, _ := os.Create("output.gz")
    defer file.Close()

    gzWriter := gzip.NewWriter(file)
    defer gzWriter.Close()

    // Data flows: generator → pipe → gzip → file
    io.Copy(gzWriter, pr)
    return nil
}

```

---

## Buffered I/O

### Why Buffering Matters

**Without Buffering:**

```
Your Code                            Kernel/Disk
   │                                     │
   ├─ Write 1 byte  ────syscall────────▶ │ (expensive)
   ├─ Write 1 byte  ────syscall────────▶ │ (expensive)
   ├─ Write 1 byte  ────syscall────────▶ │ (expensive)
   ├─ Write 1 byte  ────syscall────────▶ │ (expensive)
   │                                     │
   1000 writes = 1000 syscalls = SLOW ❌

```

**With Buffering:**

```
Your Code                Buffer (4KB)           Kernel/Disk
   │                         │                      │
   ├─ Write 1 byte ────────▶ │                      │
   ├─ Write 1 byte ────────▶ │                      │
   ├─ Write 1 byte ────────▶ │                      │
   ├─ Write 1 byte ────────▶ │                      │
   │  ... (4095 more)        │                      │
   │                         │                      │
   │                         ├──────syscall────────▶│
   │                         │   (4KB at once)      │
   │                                                │
   1000 writes = 1 syscall = FAST ✅

```

---

### bufio.Reader

```go
type Reader struct {
    // Wraps any io.Reader and adds buffering
}

```

**Key Methods:**

```go
// Standard
Read(p []byte) (n int, err error)           // Read into buffer
ReadByte() (byte, error)                    // Read single byte
ReadRune() (r rune, size int, err error)    // Read UTF-8 character

// Line-oriented
ReadBytes(delim byte) ([]byte, error)       // Read until delimiter
ReadString(delim byte) (string, error)      // Read string until delimiter
ReadLine() (line []byte, isPrefix bool, err error)

// Advanced
Peek(n int) ([]byte, error)                 // Look ahead without consuming
Discard(n int) (discarded int, err error)   // Skip bytes

```

**Visual: How bufio.Reader Works**

```
┌─────────────────────────────────────────────────────┐
│              Underlying io.Reader                   │
│                    (File/Network/etc)               │
└────────────────────┬────────────────────────────────┘
                     │
                     │ Reads large chunks
                     ↓
┌──────────────────────────────────────────────────────┐
│           bufio.Reader Internal Buffer               │
│  [H][e][l][l][o][ ][W][o][r][l][d][...][...]       │
│   ↑                                                  │
│   Current read position                              │
└────────────────────┬─────────────────────────────────┘
                     │
                     │ Returns small amounts as needed
                     ↓
               Your application
            ReadByte() → 'H'
            ReadByte() → 'e'
            ReadString('\n') → "llo World\n"

```

**Example: Efficient Line Reading**

```go
func processLogFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer file.Close()

    // Wrap in buffered reader
    reader := bufio.NewReader(file)

    lineNum := 0
    for {
        line, err := reader.ReadString('\n')
        if err == io.EOF {
            break
        }
        if err != nil {
            return err
        }

        lineNum++
        // Process line
        if strings.Contains(line, "ERROR") {
            fmt.Printf("Line %d: %s", lineNum, line)
        }
    }
    return nil
}

```

---

### bufio.Scanner

Scanner is a higher-level API for line-by-line reading.

```go
scanner := bufio.NewScanner(reader)
for scanner.Scan() {
    line := scanner.Text()
    // process line
}
if err := scanner.Err(); err != nil {
    log.Fatal(err)
}

```

**Visual: Scanner Process**

```
┌─────────────────────┐
│  Underlying Reader  │
│                     │
└──────────┬──────────┘
           │
           ↓
┌─────────────────────────────────────┐
│      bufio.Scanner                  │
│  ┌────────────────────────┐         │
│  │    Internal Buffer     │         │
│  │  [data................]│         │
│  └────────────────────────┘         │
│         │                           │
│         ↓ Split by newlines         │
│  "line 1\n" ──┐                     │
│  "line 2\n" ──┤ Queue of tokens     │
│  "line 3\n" ──┘                     │
└────────┬────────────────────────────┘
         │ scanner.Scan()
         ↓
    scanner.Text() → "line 1"
    scanner.Text() → "line 2"
    scanner.Text() → "line 3"

```

**Example: Word Count**

```go
func wordCount(reader io.Reader) int {
    scanner := bufio.NewScanner(reader)
    scanner.Split(bufio.ScanWords)  // Split by words instead of lines

    count := 0
    for scanner.Scan() {
        count++
    }
    return count
}

```

**Custom Split Function:**

```go
// Split by comma instead of newlines
func scanCommas(data []byte, atEOF bool) (advance int, token []byte, err error) {
    for i := 0; i < len(data); i++ {
        if data[i] == ',' {
            return i + 1, data[:i], nil
        }
    }
    if atEOF && len(data) > 0 {
        return len(data), data, nil
    }
    return 0, nil, nil
}

scanner := bufio.NewScanner(reader)
scanner.Split(scanCommas)

```

---

### bufio.Writer

```go
writer := bufio.NewWriter(underlyingWriter)
writer.WriteString("Hello")
writer.WriteByte('\n')
writer.Flush()  // IMPORTANT: flush to write buffered data

```

**Visual: Writer Buffer**

```
Your Code
   │ WriteString("Hello")
   │ WriteString("World")
   ↓
┌──────────────────────────────┐
│    bufio.Writer Buffer       │
│  [H][e][l][l][o][W][o][r][l] │ ← Accumulates
└──────────────────────────────┘
   │
   │ Buffer full OR Flush() called
   ↓
┌──────────────────────────────┐
│     Underlying Writer        │
│      (File/Network)          │
└──────────────────────────────┘

```

> ⚠️ Critical: Always call Flush() or defer writer.Flush() - buffered data won't be written otherwise!
>

**Example: Fast CSV Writing**

```go
func writeCSV(filename string, rows [][]string) error {
    file, err := os.Create(filename)
    if err != nil {
        return err
    }
    defer file.Close()

    // Wrap in buffered writer
    writer := bufio.NewWriter(file)
    defer writer.Flush()  // Critical!

    for _, row := range rows {
        line := strings.Join(row, ",") + "\n"
        writer.WriteString(line)
    }

    return nil
}

```

---

## Real-World Examples

### Example 1: HTTP Streaming Response

**Problem:** Download large file without loading into memory.

```go
func streamingDownload(url, dest string) error {
    // 1. Start HTTP request
    resp, err := http.Get(url)
    if err != nil {
        return err
    }
    defer resp.Body.Close()  // resp.Body is io.ReadCloser

    // 2. Create destination
    out, err := os.Create(dest)
    if err != nil {
        return err
    }
    defer out.Close()

    // 3. Stream directly from response to file
    written, err := io.Copy(out, resp.Body)
    fmt.Printf("Downloaded %d bytes\n", written)
    return err
}

```

**Visual Flow:**

```
Internet ─▶ HTTP Response Body ─▶ io.Copy ─▶ File on Disk
            (io.ReadCloser)      [32KB buf]   (io.Writer)

Memory usage: Only 32KB buffer
Time complexity: O(1) memory, O(n) time

```

---

### Example 2: Data Processing Pipeline

**Problem:** Read compressed log file, filter errors, write to new file.

```go
func filterErrorLogs(input, output string) error {
    // 1. Open input
    inFile, err := os.Open(input)
    if err != nil {
        return err
    }
    defer inFile.Close()

    // 2. Add decompression
    gzReader, err := gzip.NewReader(inFile)
    if err != nil {
        return err
    }
    defer gzReader.Close()

    // 3. Add buffered line reading
    scanner := bufio.NewScanner(gzReader)

    // 4. Create output
    outFile, err := os.Create(output)
    if err != nil {
        return err
    }
    defer outFile.Close()

    // 5. Add buffered writing
    writer := bufio.NewWriter(outFile)
    defer writer.Flush()

    // 6. Process line by line
    for scanner.Scan() {
        line := scanner.Text()
        if strings.Contains(line, "ERROR") {
            writer.WriteString(line + "\n")
        }
    }

    return scanner.Err()
}

```

**Complete Pipeline Visualization:**

```
┌────────────────┐
│ app.log.gz     │ Compressed file (100MB)
└────────┬───────┘
         │ os.Open()
         ↓
┌────────────────┐
│   os.File      │ ReadCloser
└────────┬───────┘
         │ gzip.NewReader()
         ↓
┌────────────────┐
│  gzip.Reader   │ Decompresses (now ~1GB)
└────────┬───────┘
         │ bufio.NewScanner()
         ↓
┌────────────────┐
│ bufio.Scanner  │ Splits into lines
└────────┬───────┘
         │ scanner.Scan()
         ↓
┌────────────────────────┐
│  Filter Logic          │ Check for "ERROR"
│  (Your Code)           │
└────────┬───────────────┘
         │ If ERROR: writer.WriteString()
         ↓
┌────────────────┐
│ bufio.Writer   │ Buffers writes
└────────┬───────┘
         │ Flush()
         ↓
┌────────────────┐
│   os.File      │ errors.log
└────────────────┘

Memory Usage: ~8KB (4KB read buffer + 4KB write buffer)
Throughput: Can process gigabytes with minimal RAM

```

---

### Example 3: Kafka-to-ClickHouse Bridge

**Your Domain:** Stream processing with Kafka and ClickHouse.

```go
func streamKafkaToClickHouse(kafkaBrokers []string, clickhouseURL string) error {
    // 1. Create Kafka consumer
    consumer, err := kafka.NewConsumer(&kafka.ConfigMap{
        "bootstrap.servers": strings.Join(kafkaBrokers, ","),
        "group.id":          "clickhouse-writer",
    })
    if err != nil {
        return err
    }
    defer consumer.Close()

    consumer.Subscribe("events", nil)

    // 2. Create buffer for batch
    var batch bytes.Buffer
    batchSize := 0
    maxBatch := 10000

    // 3. Flush function
    flush := func() error {
        if batchSize == 0 {
            return nil
        }

        // Create HTTP request with batch as body
        req, err := http.NewRequest("POST",
            clickhouseURL+"/",
            &batch)  // batch is io.Reader!
        if err != nil {
            return err
        }

        resp, err := http.DefaultClient.Do(req)
        if err != nil {
            return err
        }
        defer resp.Body.Close()

        // Reset batch
        batch.Reset()
        batchSize = 0
        return nil
    }

    // 4. Consume and batch
    for {
        msg, err := consumer.ReadMessage(time.Second)
        if err != nil {
            continue
        }

        // Write to batch buffer
        batch.Write(msg.Value)
        batch.WriteByte('\n')
        batchSize++

        // Flush when batch full
        if batchSize >= maxBatch {
            if err := flush(); err != nil {
                log.Printf("Flush error: %v", err)
            }
        }
    }
}

```

**Visual Flow:**

```
┌──────────┐
│  Kafka   │ Messages stream in
└────┬─────┘
     │ msg.Value ([]byte)
     ↓
┌─────────────────┐
│ bytes.Buffer    │ Accumulate batch
│ (implements     │ [msg1\nmsg2\nmsg3...]
│  io.Reader)     │
└────┬────────────┘
     │ When full or timeout
     ↓
┌─────────────────┐
│ http.Request    │ Body is io.Reader
│ Body: &batch    │
└────┬────────────┘
     │ HTTP POST
     ↓
┌─────────────────┐
│  ClickHouse     │ INSERT via HTTP
│  (HTTP API)     │
└─────────────────┘

Benefits:
- Streaming: No need to store all messages
- Efficient: Batched inserts
- Memory: bytes.Buffer reused

```

---

### Example 4: Custom Transform Reader

**Problem:** Need to transform data on-the-fly while reading.

```go
// UppercaseReader transforms all text to uppercase
type UppercaseReader struct {
    src io.Reader
}

func (u *UppercaseReader) Read(p []byte) (n int, err error) {
    // Read from source
    n, err = u.src.Read(p)

    // Transform the bytes
    for i := 0; i < n; i++ {
        if p[i] >= 'a' && p[i] <= 'z' {
            p[i] = p[i] - 32  // Convert to uppercase
        }
    }

    return n, err
}

// Usage
func example() {
    file, _ := os.Open("input.txt")
    upper := &UppercaseReader{src: file}

    io.Copy(os.Stdout, upper)  // Prints uppercase version
}

```

**Visual:**

```
┌──────────┐
│ File     │ "hello world"
└────┬─────┘
     │ Read()
     ↓
┌──────────────────┐
│ UppercaseReader  │ Transform: a-z → A-Z
└────┬─────────────┘
     │ Read()
     ↓
Output: "HELLO WORLD"

```

---

## Advanced Patterns

### Pattern 1: io.LimitReader - Read Only N Bytes

```go
r := strings.NewReader("Hello World!")
limited := io.LimitReader(r, 5)

data, _ := io.ReadAll(limited)
fmt.Println(string(data))  // "Hello"

```

**Use Case: Safe Reading of Untrusted Input**

```go
func readRequestBody(r io.Reader) ([]byte, error) {
    // Prevent reading more than 1MB
    limited := io.LimitReader(r, 1024*1024)
    return io.ReadAll(limited)
}

```

---

### Pattern 2: io.SectionReader - Read Subsection

```go
data := strings.NewReader("0123456789")
section := io.NewSectionReader(data, 3, 4)  // offset=3, length=4

buf, _ := io.ReadAll(section)
fmt.Println(string(buf))  // "3456"

```

**Visual:**

```
Full Data:  [0][1][2][3][4][5][6][7][8][9]
                      ↑         ↑
                   offset=3  length=4
Section:            [3][4][5][6]

```

**Use Case: Reading Parts of Large Files**

```go
func readFileSegment(filename string, offset, length int64) ([]byte, error) {
    file, err := os.Open(filename)
    if err != nil {
        return nil, err
    }
    defer file.Close()

    section := io.NewSectionReader(file, offset, length)
    return io.ReadAll(section)
}

```

---

### Pattern 3: Context-Aware Reading

```go
func readWithTimeout(r io.Reader, timeout time.Duration) ([]byte, error) {
    ctx, cancel := context.WithTimeout(context.Background(), timeout)
    defer cancel()

    // Create pipe
    pr, pw := io.Pipe()

    // Start reading in goroutine
    errChan := make(chan error, 1)
    go func() {
        _, err := io.Copy(pw, r)
        pw.Close()
        errChan <- err
    }()

    // Read with timeout
    done := make(chan struct{})
    var data []byte
    var readErr error

    go func() {
        data, readErr = io.ReadAll(pr)
        close(done)
    }()

    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    case <-done:
        return data, readErr
    }
}

```

---

### Pattern 4: Counting Bytes Read/Written

```go
type CountingReader struct {
    src   io.Reader
    count int64
}

func (c *CountingReader) Read(p []byte) (n int, err error) {
    n, err = c.src.Read(p)
    c.count += int64(n)
    return n, err
}

func (c *CountingReader) BytesRead() int64 {
    return c.count
}

```

**Use Case: Progress Tracking**

```go
func downloadWithProgress(url string) error {
    resp, _ := http.Get(url)
    defer resp.Body.Close()

    counter := &CountingReader{src: resp.Body}
    total := resp.ContentLength

    go func() {
        for {
            time.Sleep(time.Second)
            pct := float64(counter.BytesRead()) / float64(total) * 100
            fmt.Printf("\rProgress: %.1f%%", pct)
        }
    }()

    io.Copy(os.Stdout, counter)
    return nil
}

```

---

## Performance Guide

### Buffer Size Optimization

```
┌────────────────────────────────────────────────────┐
│             Buffer Size vs Performance             │
│                                                    │
│  Speed ▲                                           │
│        │     ╱────────────────                     │
│        │    ╱                                      │
│        │   ╱                                       │
│        │  ╱                                        │
│        │ ╱                                         │
│        └──────────────────────────▶               │
│         256B  4KB  32KB  64KB  1MB                │
│                                                    │
│  Sweet spot: 4KB - 64KB for most use cases        │
└────────────────────────────────────────────────────┘

```

**Recommendations:**

```go
// File I/O
const fileBufferSize = 4096         // 4KB - good balance

// Network I/O
const networkBufferSize = 8192      // 8KB - accounts for MTU

// Bulk transfer
const bulkBufferSize = 32768        // 32KB - for large files

// High-speed transfer
const maxBufferSize = 65536         // 64KB - maximum efficiency

```

---

### Memory Management Tips

### ❌ Bad: Creating New Buffers in Loop

```go
for _, item := range items {
    buf := make([]byte, 1024)  // Allocates each iteration
    reader.Read(buf)
    process(buf)
}

```

### ✅ Good: Reuse Buffers

```go
buf := make([]byte, 1024)  // Allocate once
for _, item := range items {
    n, err := reader.Read(buf)
    if err != nil {
        break
    }
    process(buf[:n])
}

```

---

### When to Use What

| Scenario | Use This | Why |
| --- | --- | --- |
| Reading entire file | `io.ReadAll()` | Simple, file fits in memory |
| Large file processing | `bufio.Scanner` | Line-by-line, low memory |
| Binary data | `io.Copy` with buffer | Efficient streaming |
| Network streaming | `bufio.Reader/Writer` | Reduces syscalls |
| Small writes | `bufio.Writer` | Batches small writes |
| Transformation | Custom Reader/Writer | On-the-fly processing |

---

## Best Practices

### 1. Always Close Resources

```go
// ✅ Good
file, err := os.Open("data.txt")
if err != nil {
    return err
}
defer file.Close()  // Guarantees cleanup

// ❌ Bad
file, _ := os.Open("data.txt")
// Forgot to close - resource leak!

```

---

### 2. Check All Errors

```go
// ✅ Good
n, err := reader.Read(buffer)
if err != nil && err != io.EOF {
    return fmt.Errorf("read failed: %w", err)
}

// ❌ Bad
reader.Read(buffer)  // Ignoring errors

```

---

### 3. Flush Buffered Writers

```go
// ✅ Good
writer := bufio.NewWriter(file)
defer writer.Flush()  // Ensures data is written

// ❌ Bad
writer := bufio.NewWriter(file)
// Exit without flush - data loss!

```

---

### 4. Use io.Copy for Large Data

```go
// ✅ Good - Streaming
io.Copy(dst, src)  // Constant memory

// ❌ Bad - Loads everything
data, _ := io.ReadAll(src)  // Could be gigabytes!
dst.Write(data)

```

---

### 5. Prefer Composition Over Modification

```go
// ✅ Good - Wrap existing readers
compressed := gzip.NewReader(file)
buffered := bufio.NewReader(compressed)

// ❌ Bad - Modifying data in place
// More complex, error-prone

```

---

### 6. Handle EOF Correctly

```go
// ✅ Good
for {
    n, err := reader.Read(buffer)
    if n > 0 {
        process(buffer[:n])  // Process data first
    }
    if err == io.EOF {
        break  // Normal end
    }
    if err != nil {
        return err  // Real error
    }
}

// ❌ Bad
for {
    n, err := reader.Read(buffer)
    if err != nil {  // Could miss last chunk if EOF!
        break
    }
    process(buffer[:n])
}

```

---

## Common Patterns Summary

### Pattern Reference Table

| Pattern | Code | Use When |
| --- | --- | --- |
| **Simple Read** | `data, _ := io.ReadAll(r)` | Small files, fits in memory |
| **Chunked Read** | `n, _ := r.Read(buf)` | Large files, control memory |
| **Line Reading** | `scanner.Scan()` | Text files, line-by-line |
| **Copy** | `io.Copy(dst, src)` | File copy, streaming |
| **Transform** | Custom Reader wrapper | On-the-fly processing |
| **Fan-out** | `io.MultiWriter()` | Write to multiple destinations |
| **Tee** | `io.TeeReader()` | Read + log/backup |
| **Chain** | Nested wrappers | Multiple transformations |
| **Buffer** | `bufio.NewReader/Writer()` | Frequent small I/O |
| **Pipe** | `io.Pipe()` | Connect goroutines |

---

## Key Takeaways 🎯

1. **Everything is an interface** - `io.Reader` and `io.Writer` are universal
2. **Composition is king** - Wrap readers/writers to add functionality
3. **Buffering is crucial** - Use `bufio` for performance
4. **Stream, don't load** - Process data as it flows
5. **Close and flush** - Always clean up resources
6. **Check errors** - I/O operations can fail
7. **Reuse buffers** - Allocate once, use many times

---

## Further Reading

- **Official Docs**: https://pkg.go.dev/io
- **bufio Package**: https://pkg.go.dev/bufio
- **Effective Go**: https://go.dev/doc/effective_go#data
- **io/fs Package**: Modern filesystem abstractions (Go 1.16+)

---

## Quick Reference Card

```go
// READING
io.ReadAll(r)              // Read everything
io.Copy(dst, src)          // Stream copy
bufio.NewReader(r)         // Add buffering
bufio.NewScanner(r)        // Line-by-line

// WRITING
io.WriteString(w, s)       // Write string
io.MultiWriter(w1,w2...)   // Write to many
bufio.NewWriter(w)         // Add buffering

// UTILITIES
io.LimitReader(r, n)       // Read max n bytes
io.TeeReader(r, w)         // Read + copy
io.Pipe()                  // In-memory connection
io.MultiReader(r1,r2...)   // Concatenate readers

// PATTERNS
defer file.Close()         // Always close
defer writer.Flush()       // Always flush buffered
if err == io.EOF           // End of stream

```

---

**Happy streaming! 🚀**

> Remember: In Go, everything flows through io.Reader and io.Writer. Master these interfaces, and you master I/O.
>
