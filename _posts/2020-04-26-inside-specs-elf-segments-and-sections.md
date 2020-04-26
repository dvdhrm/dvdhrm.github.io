---
layout: post
caption: "Inside Specs: ELF Segments and Sections"
categories: [fedora]
tags: [elf, binary, executable, linking, segments, sections, format]
---
The *ELF data format* divides object files into **segments** and **sections**,
which has for long caused confusion. Both terms **segment** and **section** can
be used interchangeably in almost all cases in the English language
([[1]](https://www.lexico.com/en/definition/section),
[[2]](https://www.lexico.com/en/definition/segment)). What is often overlooked is
that the ELF specification explicitely meant both to mean almost the same. They
merely provide two views of the same data, but use different terms to allow
referring to them more easily.

When we look at the defining specification
([**gABI**](https://refspecs.linuxfoundation.org/elf/gabi4+/contents.html):
*System V Application Binary Interface*) we find this quote in the introduction:

> Object files participate in program linking (building a program) and program
> execution (running a program). For convenience and efficiency, the object file
> format provides parallel views of a file's contents, reflecting the differing
> needs of those activities.

This is, in my opinion, a crucial detail often overlooked. The ELF data format
explicitly provides two views of the *same data*. The difference between
segments and sections is thus not what data they contain, but how they index
the *same data*. The specification goes a step further:

> A program header table tells the system how to create a process image. Files
> used to build a process image (execute a program) must have a program header
> table; relocatable files do not need one.
>
> A section header table contains information describing the file's sections.
> Every section has an entry in the table; each entry gives information such as
> the section name, the section size, and so on. Files used during linking must
> have a section header table; other object files may or may not have one.

Keep in mind that the *program header table* is effectively a *segment header
table*. Therefore, the specification explicitly says that these two data views
do not have to be present in a specific file. Depending on the use case, the
format allows for *only segments* or *only sections*.

To summarize, an ELF object file contains data and machine code of a program,
which itself is divided into many parts. The ELF format then provides two
different views of this same content: *segments* and *sections*. However, these
are views of the data present in the file, they do not define the content, but
merely index it.

As a closing note, we must acknowledge how all this evolved over time, though.
While the ELF specification provides this neat dual-view, a lot of this freedom
is not actually used in most ELF files. Instead, most files are effectively
split into many small sections, and the segments merely provide a grouping of
sequential sections in the file. Sections have become the tool that drives the
data in ELF files, and segments have become a view of that data. But this was
a purely artifical interpretation and is not rooted in the ELF data format.
