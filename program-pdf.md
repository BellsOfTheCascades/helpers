# Tips and tools for formatting and printing concert program PDFs

## Tools

Apart from Adobe Acrobat Reader and the PDF viewer that comes with your operating system, there are several command-line tools that, once you get the hang of them, are actually easier (and definitely more powerful) for manipulating PDFs.

- [pdftk](https://linux.die.net/man/1/pdftk) is installed on most Linux distros and [can be installed on a Mac](https://stackoverflow.com/a/58236185/19148969). [PDFtk Free](https://www.pdflabs.com/tools/pdftk-the-pdf-toolkit/) is available for Windows and has a graphical user interface. Other GUIs for pdftk include [PDF Chain](https://pdfchain.sourceforge.io/), [Pdftk4all](http://pdftk4all.sourceforge.net/), and [PDFTK Builder](http://www.angusj.com/pdftkb/#pdftkbuilder). (Each tool seems to enjoy using a different style of capitalization.)
- [pdfjam](https://github.com/rrthomas/pdfjam) has most of the same functionality as pdftk and runs natively on both Mac and Linux.
- [cpdf](https://github.com/coherentgraphics/cpdf-binaries) also has similar functionality, and has installable binaries for Windows, Mac and Linux.
- [Ghostscript](https://www.ghostscript.com) is a PostScript interpreter for PDFs and can do quite a lot; some of the tools above use it under the hood. It is installed on virtually all Mac and Linux machines automatically (though it may be an older version). [View its releases](https://www.ghostscript.com/releases/gsdnld.html) for Windows and updated Linux installs; to install a recent version on a Mac [use Homebrew](https://formulae.brew.sh/formula/ghostscript).

Linux tools with GUIs that can manipulate PDFs include [LibreOffice Draw](https://www.libreoffice.org/discover/draw/), [Okular](https://okular.kde.org/), [Scribus](https://www.scribus.net/) and [Xournal++](https://xournalpp.github.io/).

While image programs like Photoshop, Glimpse and GIMP can open and save PDFs, they usually can't handle multi-page PDFs well and will generally make PDFs with much larger file sizes.

## Saving PDFs

When possible PDFs should be created (sometimes described as "printed") directly from word processing or layout programs. This ensures the text of the PDF is actually text, and not an image -- which both makes the PDF more accessible (when it's posted on a website) and ensures the text stays crisp when sent to the printer.

If given options, ensure **fonts are embedded** and **images (or the entire PDF) are set to 300dpi**.

## Splitting a PDF

Concert programs often have a color cover/inside cover, and one or more black-and-white inserts. To split the completed program PDF into outside and inside files for separate printing, use `pdftk`

Where `program.pdf` is the name of the full file, `program_outside.pdf` and `program_inside.pdf` are the names of the files you're creating, the cover and inside cover are pages 1 and 2, and the inside portion consists of pages 3 and 4:

```sh
pdftk program.pdf cat 1-2 output program_outside.pdf
pdftk program.pdf cat 3-4 output program_inside.pdf
```

If the inside portion is just page 3, use this as the second command instead:

```sh
pdftk program.pdf cat 3 output program_inside.pdf
```

## Rotating a PDF

Some printers (weirdly) only accept PDFs in portrait orientation, and programs are generally in landcape. To rotate a PDF, you can again use `pdftk`. The following command rotates the file counter-clockwise and outputs `program_outside_rotated.pdf` (don't try to output to the same filename you're inputting):

```sh
pdftk program_outside.pdf cat 1-endwest output program_outside_rotated.pdf
```

To rotate clockwise, use `1-endeast` instead of `1-endwest`.

## Duplicating a PDF page

If your inside content is meant to be a single half-page insert, you'll want to create a 2-page version of it to print back-to-back.

Using `pdftk`:

```sh
pdftk program_inside.pdf cat 1-end 1-end output program_inside_print.pdf
```

## Reducing the file size of a PDF

In most cases, if your PDF is destined for a printer then you should prioritize print quality (resolution) over file size. However, if you need to create a reduced-size version or just want to save disk space, you can use Ghostscript, which will dramatically reduce the file size while maintaining quality.

To ensure everything is downscaled to 300dpi (the highest resolution you'd generally need for printing), in which `program.pdf` is the current file and `program_reduced.pdf` is the new smaller file:

```sh
gs \
 -q -dNOPAUSE -dBATCH -dSAFER \
 -sDEVICE=pdfwrite \
 -dCompatibilityLevel=1.4 \
 -dPDFSETTINGS=/screen \
 -dEmbedAllFonts=true -dSubsetFonts=true \
 -dNOTRANSPARENCY \
 -dColorImageDownsampleType=/Bicubic \
 -dColorImageResolution=300 \
 -dGrayImageDownsampleType=/Bicubic \
 -dGrayImageResolution=300 \
 -dMonoImageDownsampleType=/Bicubic \
 -dMonoImageResolution=300 \
 -sOutputFile=program_reduced.pdf program.pdf
 ```


Unless the program contains only pretty large type, the lowest resolution you can really go for and still be readable on the screen is 150dpi. Here's that command, in which `program.pdf` is the current file and `program_reduced.pdf` is the new smaller file:

```sh
gs \
 -q -dNOPAUSE -dBATCH -dSAFER \
 -sDEVICE=pdfwrite \
 -dCompatibilityLevel=1.4 \
 -dPDFSETTINGS=/screen \
 -dEmbedAllFonts=true -dSubsetFonts=true \
 -dNOTRANSPARENCY \
 -dColorImageDownsampleType=/Bicubic \
 -dColorImageResolution=150 \
 -dGrayImageDownsampleType=/Bicubic \
 -dGrayImageResolution=150 \
 -dMonoImageDownsampleType=/Bicubic \
 -dMonoImageResolution=150 \
 -sOutputFile=program_reduced.pdf program.pdf
 ```

### Notes on these GhostScript commands

- The `-dCompatibilityLevel` in the commands above is set to `1.4`, which results in dramatically (90% smaller) file sizes than `1.3`. [PDF 1.4 was released in 2001](https://www.prepressure.com/pdf/basics/version), so it's hard to imagine many systems or software not being able to support it, but if you do run into issues then you can change that option to `1.3`.
- The `-dNOTRANSPARENCY` item in the commands does what it implies. Should you have any transparent or semi-transparent images or areas and want to preserve them, remove that option.
