+++
title = 'Compressing PDF Files using Ghostscript is Fascinating'
date = 2024-10-16T11:07:33+07:00
images = ['/compressing-pdf-files-using-ghostscript-is-fascinating.webp']
+++

My wife was applying for a job online, she needed to upload her portfolio as part of the application process. The website had a file size limit of 10MB, but her PDF was a whopping 30MB. I searched all over the internet for a PDF compressor. I tried almost every PDF compressor in the Google Search's first page, but none of them are quite my tempo. If they managed to shrink the file at all, the quality would drop so badly it looked like a pixelated mess. I kept searching for a better way—and that’s when I stumbled upon Ghostscript, which turned out to be a lifesaver.

## What is Ghostscript?

I found a [StackOverflow question](https://askubuntu.com/questions/113544/how-can-i-reduce-the-file-size-of-a-scanned-pdf-file) that answers the question of how to reduce the file size of a PDF file. [Ghostscript](https://ghostscript.readthedocs.io/en/latest/) is an interpreter for the PostScript® language and PDF files. 

# How to Compress a PDF File using Ghostscript

To compress a PDF file using Ghostscript, you need to follow these steps:

1. Install Ghostscript on your system. I use Brew on macOS, so I run `brew install gs` to install it.
2. Open a terminal and navigate to the directory where your PDF file is located.
3. Run the following command to compress the PDF file:

```bash
gs -sDEVICE=pdfwrite -dCompatibilityLevel=1.4 -dPDFSETTINGS=/ebook -dNOPAUSE -dQUIET -dBATCH -sOutputFile=<your_output_file>.pdf <your_file>.pdf
```

The main factor to consider when compressing a PDF file is the `-dPDFSETTINGS` option. It determines the quality of the output PDF file. Here are the available options from the [Ghostscript documentation](https://ghostscript.readthedocs.io/en/latest/VectorDevices.html#controls-and-features-specific-to-postscript-and-pdf-input):

- `/screen` selects low-resolution output similar to the Acrobat Distiller (up to version X) “Screen Optimized” setting.
- `/ebook` selects medium-resolution output similar to the Acrobat Distiller (up to version X) “eBook” setting.
- `/printer` selects output similar to the Acrobat Distiller “Print Optimized” (up to version X) setting.
- `/prepress` selects output similar to Acrobat Distiller “Prepress Optimized” (up to version X) setting.
- `/default` selects output intended to be useful across a wide variety of uses, possibly at the expense of a larger output file.

With these options in mind, I tried ALL of the options to know which one is the best for my PDF file. Here's the result of the compressions:

```bash
-rw-r--r--@  1 dennypradipta  staff    32M Oct 16 10:49 input.pdf
-rw-r--r--@  1 dennypradipta  staff    19M Oct 16 11:19 output_default.pdf
-rw-r--r--@  1 dennypradipta  staff    19M Oct 16 11:18 output_ebook.pdf
-rw-r--r--@  1 dennypradipta  staff    21M Oct 16 11:18 output_prepress.pdf
-rw-r--r--@  1 dennypradipta  staff    20M Oct 16 11:18 output_printer.pdf
-rw-r--r--@  1 dennypradipta  staff   2.5M Oct 16 11:18 output_screen.pdf
```

The smallest PDF file is the `screen` option, which is about 2.5MB, and the largest PDF file is the `prepress` option, which is about 21MB (I'm not counting the input file). I was feeling pretty doubtful when I saw the difference in file size between `screen` and the original one. But when I open the `output_screen.pdf` file, I was pleasantly surprised to see that the quality did not even drop at all. Well, maybe slightly. Hell, it still looks like the original PDF file even in large screen. So, I guess the compression is not that bad after all. 

Sorry I couldn't show you the content of the PDF file, but you have my word that the quality is not some kind of pixelated cesspool.

# Conclusion

In the end, Ghostscript turned out to be the one of the best solution for compressing PDF files. While the `screen` option gave me a file small enough to meet the 10MB limit, the quality remained surprisingly good, which was a huge relief after trying so many disappointing alternatives. If you’re ever stuck with a hefty PDF that needs trimming without losing its professional look, Ghostscript is definitely worth a try. 

And remember, if all else fails, there’s always coffee. Lots of coffee.