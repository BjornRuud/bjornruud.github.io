+++
title = "Batch Image Download With wget"
+++
# Batch Image Download With wget

[wget](http://www.gnu.org/software/wget/) is often used to download single files from the command line, but it can also mirror a website locally or just download part of a website. By specifying the right parameters we can make wget act as batch downloader, retrieving only the files we want.

In this example we assume a website with a sequence of pages, where each page links to the next in the sequence and they all contain a JPEG image. We want to download all the images to the current directory. The following command line does this:

```
$ wget --recursive --level=inf --no-directories --no-parent --accept *.jpg URL
```

Or if you prefer, the shorter but more obscure:

```
$ wget -r -l inf -nd -np -A *.jpg URL
```

Let’s take a look at the parameters:

```
--recursive
    Makes wget follow links from the start page.

--level=inf
    This allows for infinite recursion. In combination with another option that limits the
    recursion depth, like –no-parent, we don’t need to know the necessary depth. Otherwise
    you should specify a number to set a limit.

--no-directories
    Default behaviour is to recreate the directory structure of the website. This option
    makes wget put all files in the same directory.

--no-parent
    Do not follow links to pages above the starting page in the hierarchy.

--accept *.jpg
    Here we specify what kind of file to download. The parameter can be a comma-separated list.
```

For other options and details about those listed here, check the wget man page.

---

In one scenario I used wget the files had to be zero padded (like `img-01.jpg` instead of `img-1.jpg`). For a single directory this did the job (for filenames with two digits):

```
$ rename 's/-([0-9])\./-0$1\./' *.jpg
```

For a directory tree this was used:

```
$ find . "*.jpg" -type f -print0 | xargs --null rename 's/-([0-9])\./-0$1\./'
```
