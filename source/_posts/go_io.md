---
title: go io的一些解析
date: 2025-01-31 13:47:40
categories: 
- go
---
本次打算对go的io进行一个分析。
# go io
首先我们来查看一下go io包下的文件树。
```
├── example_test.go
├── export_test.go
├── fs
├── io.go
├── io_test.go
├── ioutil
├── multi.go
├── multi_test.go
├── pipe.go
└── pipe_test.go
```
`ioutil`是一些工具，`fs`则是`go1.16`加入的对文件系统相关的包。
所以这里我们首先来看看`io.go`。
其中包含四个最基础的接口。
```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
type Writer interface {
	Write(p []byte) (n int, err error)
}
type Closer interface {
	Close() error
}
type Seeker interface {
	Seek(offset int64, whence int) (int64, error)
}
```
当然还有非常多的组合接口。
```go
type ReadWriter interface {
	Reader
	Writer
}
type ReadCloser interface {
	Reader
	Closer
}
type WriteCloser interface {
	Writer
	Closer
}
//.......
```
这些都是go不错的设计理念，设计小接口，通过小接口组合出大接口。
接下来我们来看看其中给的一些`io struct`的设计。
## LimitedReader
```go
// A LimitedReader reads from R but limits the amount of
// data returned to just N bytes. Each call to Read
// updates N to reflect the new amount remaining.
// Read returns EOF when N <= 0 or when the underlying R returns EOF.
type LimitedReader struct {
	R Reader // underlying reader
	N int64  // max bytes remaining
}

func (l *LimitedReader) Read(p []byte) (n int, err error) {
	if l.N <= 0 {
		return 0, EOF
	}
	if int64(len(p)) > l.N {
		p = p[0:l.N]
	}
	n, err = l.R.Read(p)
	l.N -= int64(n)
	return
}
```
这个主要是限制你只能读文件的n个字节。比如下面的方法第二次读就返回error了
```GO
func TestLimitedReader(t *testing.T) {
	str := "abcdefghij"
	r := strings.NewReader(str)
	lr := &io.LimitedReader{R: r, N: 5}

	buf := make([]byte, len(str))
	n, err := lr.Read(buf)
	if err != nil {
		t.Error(err)
	}
	fmt.Println(buf, n)

	buf = make([]byte, len(str))
	n, err = lr.Read(buf)
	if err != nil {
		t.Error(err)
	}
	fmt.Println(buf, n)
}
```

## SectionReader
```GO
// SectionReader implements Read, Seek, and ReadAt on a section
// of an underlying [ReaderAt].
type SectionReader struct {
	r     ReaderAt // 底层的 ReaderAt
	base  int64    // 区间的起始位置
	off   int64    // 当前的读取位置
	limit int64    // 区间的结束位置
	n     int64    // 区间的大小
}

func (s *SectionReader) Read(p []byte) (n int, err error) {
	if s.off >= s.limit {
		return 0, EOF
	}
	if max := s.limit - s.off; int64(len(p)) > max {
		p = p[0:max]
	}
	n, err = s.r.ReadAt(p, s.off)
	s.off += int64(n)
	return
}

var errWhence = errors.New("Seek: invalid whence")
var errOffset = errors.New("Seek: invalid offset")

func (s *SectionReader) Seek(offset int64, whence int) (int64, error) {
	switch whence {
	default:
		return 0, errWhence
	case SeekStart:
		offset += s.base // 如果是相对于文件开头，就是offset+s.base
	case SeekCurrent:
		offset += s.off // 相对于当前位置，就是s.offset+s.off(Reader当前所在位置)
	case SeekEnd:
		offset += s.limit
	}
	if offset < s.base {
		return 0, errOffset
	}
	s.off = offset
	return offset - s.base, nil
}

func (s *SectionReader) ReadAt(p []byte, off int64) (n int, err error) {
	if off < 0 || off >= s.Size() {
		return 0, EOF
	}
	off += s.base
	if max := s.limit - off; int64(len(p)) > max {
		p = p[0:max]
		n, err = s.r.ReadAt(p, off)
		if err == nil {
			err = EOF
		}
		return n, err
	}
	return s.r.ReadAt(p, off)
}

// Size returns the size of the section in bytes.
func (s *SectionReader) Size() int64 { return s.limit - s.base }

// Outer returns the underlying [ReaderAt] and offsets for the section.
//
// The returned values are the same that were passed to [NewSectionReader]
// when the [SectionReader] was created.
func (s *SectionReader) Outer() (r ReaderAt, off int64, n int64) {
	return s.r, s.base, s.n
}
```
这个主要实现的是`ReaderAt`接口。
```go
type ReaderAt interface {
	ReadAt(p []byte, off int64) (n int, err error)
}
```
我们来看看`SectionReader`实现大概就明白其主要作用了。其主要就是提供两种Read方法，一种是使用内部off，一种是使用外部off。
- Seek 方法主要是随机到区间的某个位置。
- ReadAt方法则是通过参数指定偏移量来读取。
- Read方法偏移量s.base+off读取数据到p中。
  也就是提供了一个随机读取的能力和块读取的能力。接下来可以看看io包是如何测试他的。
```go
func TestSectionReader_Seek(t *testing.T) {
	// Verifies that NewSectionReader's Seeker behaves like bytes.NewReader (which is like strings.NewReader)
	br := bytes.NewReader([]byte("foo"))
	sr := io.NewSectionReader(br, 0, int64(len("foo")))

	for _, whence := range []int{io.SeekStart, io.SeekCurrent, io.SeekEnd} {
		for offset := int64(-3); offset <= 4; offset++ {
			brOff, brErr := br.Seek(offset, whence)
			srOff, srErr := sr.Seek(offset, whence)
			if (brErr != nil) != (srErr != nil) || brOff != srOff {
				t.Errorf("For whence %d, offset %d: bytes.Reader.Seek = (%v, %v) != SectionReader.Seek = (%v, %v)",
					whence, offset, brOff, brErr, srErr, srOff)
			}
		}
	}

	// And verify we can just seek past the end and get an EOF
	got, err := sr.Seek(100, io.SeekStart)
	if err != nil || got != 100 {
		t.Errorf("Seek = %v, %v; want 100, nil", got, err)
	}

	n, err := sr.Read(make([]byte, 10))
	if n != 0 || err != io.EOF {
		t.Errorf("Read = %v, %v; want 0, EOF", n, err)
	}
}
```
前面主要是测试seek方法工作是否合理，主要是与bytes.NewReader进行比较。最后的是测试Read方法和ReadAt工作是否合理，因为ReadAt是从sr.off(100)的位置开始读，所以n返回值应该是0。
整个struct就是提供一个区间读取的能力。
## OffsetWriter
```go
// An OffsetWriter maps writes at offset base to offset base+off in the underlying writer.
type OffsetWriter struct {
	w    WriterAt
	base int64 // the original offset
	off  int64 // the current offset
}

// NewOffsetWriter returns an [OffsetWriter] that writes to w
// starting at offset off.
func NewOffsetWriter(w WriterAt, off int64) *OffsetWriter {
	return &OffsetWriter{w, off, off}
}

func (o *OffsetWriter) Write(p []byte) (n int, err error) {
	n, err = o.w.WriteAt(p, o.off)
	o.off += int64(n)
	return
}

func (o *OffsetWriter) WriteAt(p []byte, off int64) (n int, err error) {
	if off < 0 {
		return 0, errOffset
	}

	off += o.base
	return o.w.WriteAt(p, off)
}

func (o *OffsetWriter) Seek(offset int64, whence int) (int64, error) {
	switch whence {
	default:
		return 0, errWhence
	case SeekStart:
		offset += o.base
	case SeekCurrent:
		offset += o.off
	}
	if offset < o.base {
		return 0, errOffset
	}
	o.off = offset
	return offset - o.base, nil
}
```
- Write方法 o.off 位置写入p。同时o.off + len(p)
- WriteAt方法则是从s.base+off位置写入怕，但是o.off不变。
- Seek泽合SectionReader一样，随机到一个位置。
  目前看来其主要提供的能力就是偏移写入。这可以来写大文件，分割开写，并发安全。第一个routine写前100个byte，第二个routine写100-200个byte。


# go io/fs
TODO

# 参考链接
- https://tonybai.com/2021/03/23/io-fs-interface-is-an-excellent-design/