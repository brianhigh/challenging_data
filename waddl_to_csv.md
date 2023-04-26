---
title: "WADDL to CSV"
author: "Brian High"
date: "2023-04-24"
output:
  html_document:
    keep_md: yes
editor_options: 
  chunk_output_type: console
---



## Read WADDL file and convert to CSV

We will use an example "text" data file, a modified subset of a file we were told 
came from the [Washington Animal Disease Diagnostic Laboratory](https://waddl.vetmed.wsu.edu/) (WADDL). We were asked to read the file into R, but we were not told anything 
about the file format or structure, except that it was a "text" file.

This particular file format is challenging because the functions you would normally 
use for reading "text" files will not work without using some special options. 
Further, the data are not arranged in a familiar and [tidy](https://tidyr.tidyverse.org/articles/tidy-data.html) "columns are variables" structure.

We will explore how to address the issues that come up when reading this file: 

- Unexpected "embedded nulls" warning
- Unexpected "invalid multibyte string" error
- Unusual file encoding type
- Less common separator/delimiter
- Missing header (no column names)
- Columns with no values (which we will remove)
- Rows missing any useful data (which we will remove)
- Unusual data structure (not "tidy", as in not "columns are variables")

For the curious, we also provide more details in three appendices:

- Appendix I: What's an "embedded null"?
- Appendix II: What's UTF-16LE?
- Appendix III: Good nulls and bad nulls

## Setup

First, we will load our R packages, set options, and set file paths.


```r
# Attach packages, installing as needed
if (!requireNamespace("pacman", quietly = TRUE)) install.packages("pacman")
pacman::p_load(formatR, knitr, dplyr, tidyr, stringr, purrr, readr)

# Set options
options(readr.show_col_types = FALSE, readr.show_progress = FALSE)

# Set file paths
txt_file <- file.path("data", "waddl_data_example.txt")
csv_file <- file.path("data", "waddl_data_example.csv")
```

## Data file inspection

The data file we were given had "Sensititre" in the name and ended with a ".TXT" 
suffix. An internet search revealed that [Sensititre](https://www.thermofisher.com/order/catalog/product/V3000-VZ) is a laboratory instrument 
used for antimicrobial sensitivity testing. We could not find a description of the 
file format and we were not provided with a code book to explain the variables. 
There is commercial software available for purchase that was designed to
work with this instrument called the Thermo Scientific™ Sensititre™ SWIN™ Software 
System. We do not have this software, nor could we find a manual for it available 
online.

The file can be opened in Notepad and [Notepad++](https://notepad-plus-plus.org/), 
but will not show much when opened in R's text editor. In Notepad++ we turned on 
View -> Show Symbol -> Show All Characters. We see it has no column headers, 
is tab delimited (shown as arrows), contains NUL characters, and the lines end 
with CR LF (carriage-return line-feed), the standard text file line ending 
characters used on Windows systems. In Notepad++, from the Encoding menu, we 
see "UTF-16 LE BOM". 

Opening the file on a Mac with [BBEdit](https://www.barebones.com/products/bbedit/), 
we selected View -> Text Display -> Show Invisibles. This shows the tabs as grey 
triangles and upside-down red question marks where there are no data values 
(empty fields). Maybe that is the symbol for NULs. At the bottom in the status 
bar, it says "Text File", "Unicode (UTF-16 Little-Endian)" and "Windows (CRLF)". 
So, that's the same file encoding as Notepad++ reported, but without the "BOM". 
If you select the file encoding shown in the status bar of BBEdit, you see that 
there is another choice for "Unicode (UTF-16 Little-Endian, no BOM)", so that 
implies the encoding of this file is "with BOM".

So, we expect we can read this file into R as "text", because of the file suffix, 
but the NUL characters and atypical file encoding might need special treatment. 
Normally, we would expect a text file to not have any NUL characters and to be 
encoded as UTF-8, ANSI (Windows-1252 or CP-1252), ASCII, or Latin-1 
(ISO/IEC 8859-1). It's interesting that, when opened in R's text editor, the file 
does not show more than just the first character of the file ("1"). Maybe the 
encoding type or NUL characters are not supported by R's text editor.  

We could convert the file format in either Notepad++ or BBedit to UTF-8 (or convert 
with a shell utility like `iconv`) and make our lives a little easier, but we 
will try to do all processing in R for better automation and reproducibility.

## Embedded nulls

We will start with `read.table()`, specifying a tab delimiter and no header, 
and reading in just the first line. When we read it in we get some unexpected 
messages.


```r
# Try reading with read.table, just the first row
try(read.table(txt_file, sep = "\t", header = FALSE, nrows = 1))
```

```
## Warning in read.table(txt_file, sep = "\t", header = FALSE, nrows = 1): line 1
## appears to contain embedded nulls
```

```
## Error in type.convert.default(data[[i]], as.is = as.is[i], dec = dec,  : 
##   invalid multibyte string at '<ff><fe>1'
```

We get a warning "appears to contain embedded nulls". We saw NUL characters in 
Notepad++, and red upside-down question marks in BBEdit for empty fields, so 
perhaps that's what this is about. The warning will go away if we use `skipNul = TRUE`.


```r
# Try reading with read.table, just the first row
try(read.table(txt_file, sep = "\t", header = FALSE, nrows = 1, skipNul = TRUE))
```

```
## Error in type.convert.default(data[[i]], as.is = as.is[i], dec = dec,  : 
##   invalid multibyte string at '<ff><fe>1'
```

However, we still see the "invalid multibyte string" error.

## Invalid multibyte strings

The "invalid multibyte string" error is a clue that the file is not encoded as 
expected (usually UTF-8, ANSI, or ASCII), so we check this with `guess_encoding()`.


```r
# Check file encoding
readr::guess_encoding(txt_file, n_max = 1)
```

```
## # A tibble: 4 × 2
##   encoding   confidence
##   <chr>           <dbl>
## 1 UTF-16LE         1   
## 2 ISO-8859-1       0.68
## 3 ISO-8859-2       0.54
## 4 UTF-32LE         0.25
```

And the guesses are listed with the most likely encoding, "UTF-16LE", at the the 
top of the list with a probability of 1. Notepad++ showed the same encoding, 
but added "BOM". We can verify that encoding by checking the first four bytes 
of the file for the "byte order mark" (BOM). For UTF-16, we would expect a 
two-byte BOM, so checking the next two bytes after that will also let us see the 
first two-byte character of data.


```r
# Look for a byte order mark (BOM) in the first four bytes to verify encoding
readBin(txt_file, what = "raw", n = 4)
```

```
## [1] ff fe 31 00
```

This shows four pairs of symbols. Each pair represents the hexadecimal values 
for a byte (8 bits). With a 16-bit encoding, each character in the file is 
represented by 2 bytes. The first two bytes are `ff fe`, which is the "BOM" 
associated with UTF-16LE encoding. The next two bytes are `31 00`, which 
represents the character "1" in UTF-16LE. "1" is the first character of data 
in this file. These four bytes confirm UTF-16LE with BOM encoding for this file.

- See: https://www.unicode.org/faq/utf_bom.html#bom4
- And: https://en.wikipedia.org/wiki/Byte_order_mark#UTF-16

The BOM is the source of the "invalid multibyte string" error because `read.table()` 
does not know this file is encoded as UTF-16LE with BOM. 

When we set `fileEncoding = "UTF-16LE"`, reading the first line, there are no 
errors. We do not have to specify "BOM", and the BOM will be removed automatically.


```r
read.table(txt_file, sep = "\t", header = FALSE, nrows = 1, skipNul = TRUE, fileEncoding = "UTF-16LE") %>%
    as_tibble()
```

```
## # A tibble: 1 × 341
##      V1    V2 V3     V4    V5    V6    V7       V8 V9    V10   V11   V12   V13  
##   <int> <int> <chr>  <lgl> <lgl> <lgl> <chr> <int> <lgl> <chr> <chr> <chr> <lgl>
## 1     1 99050 2099-… NA    NA    NA    2099…     2 NA    S.AUR ANY   ANY   NA   
## # … with 328 more variables: V14 <int>, V15 <lgl>, V16 <lgl>, V17 <lgl>,
## #   V18 <lgl>, V19 <int>, V20 <int>, V21 <lgl>, V22 <lgl>, V23 <chr>,
## #   V24 <lgl>, V25 <lgl>, V26 <lgl>, V27 <lgl>, V28 <lgl>, V29 <lgl>,
## #   V30 <lgl>, V31 <lgl>, V32 <lgl>, V33 <lgl>, V34 <lgl>, V35 <lgl>,
## #   V36 <lgl>, V37 <lgl>, V38 <lgl>, V39 <lgl>, V40 <int>, V41 <chr>,
## #   V42 <lgl>, V43 <lgl>, V44 <lgl>, V45 <lgl>, V46 <lgl>, V47 <lgl>,
## #   V48 <lgl>, V49 <lgl>, V50 <lgl>, V51 <lgl>, V52 <lgl>, V53 <lgl>, …
```

## Read the whole file

We get no errors when we try to read all of the rows.


```r
# Read file
df <- read.table(txt_file, sep = "\t", header = FALSE, skipNul = TRUE, fileEncoding = "UTF-16LE")
```

Yay! Let's take a look...


```r
# View data
df[1:5, 1:50]
```

```
##   V1    V2        V3 V4 V5 V6          V7 V8 V9   V10 V11 V12 V13 V14 V15 V16
## 1  1 99050 2099-0707 NA NA NA 2099-0707.2  2 NA S.AUR ANY ANY  NA  -1  NA  NA
## 2  2 99050 2099-0707 NA NA NA 2099-0707.2  2 NA S.AUR ANY ANY  NA  -1  NA  NA
## 3  3 99050 2099-0707 NA NA NA 2099-0707.2  2 NA S.AUR ANY ANY  NA  -1  NA  NA
## 4  4 99057 2099-0031 NA NA NA 2099-0031.1  1 NA E.COL ANY EQU  NA  -1  NA  NA
## 5  5 99058 2099-0129 NA NA NA 2099-0129.1  1 NA E.COL   U CAN  NA  -1  NA  NA
##   V17 V18 V19 V20 V21 V22 V23 V24 V25 V26 V27 V28 V29 V30 V31 V32 V33 V34 V35
## 1  NA  NA   0   0  NA  NA   +  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
## 2  NA  NA   0   0  NA  NA   +  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
## 3  NA  NA   0   0  NA  NA   +  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
## 4  NA  NA   0   0  NA  NA      NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
## 5  NA  NA   0   0  NA  NA      NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
##   V36 V37 V38 V39 V40                 V41    V42        V43    V44    V45
## 1  NA  NA  NA  NA  24 2099-05-25 13:27:05                                
## 2  NA  NA  NA  NA  24 2099-06-01 09:14:28                                
## 3  NA  NA  NA  NA  24 2099-06-01 13:42:48 AMIKAC <=      16 NOINTP AMOCLA
## 4  NA  NA  NA  NA  18 2099-06-02 10:24:16 AMIKAC <=       4   SUSC       
## 5  NA  NA  NA  NA  24 2099-06-05 08:22:22 AMIKAC <=       4   SUSC AMOCLA
##          V46   V47    V48        V49    V50
## 1                                          
## 2                                          
## 3  =     0.5 INTER AMPICI  =       4 RESIST
## 4                  AMPICI  >      32 RESIST
## 5  =       4  SUSC AMPICI  =       4   SUSC
```

We see there are some blank fields in columns V23, V42, and columns after that. 
We would prefer to see NAs here. Let's see what's really in those empty fields.


```r
unique(df$V23)
```

```
## [1] "+" ""  "-"
```

```r
unique(df$V42)
```

```
## [1] ""       "AMIKAC"
```

The blank fields are really the empty string (""). We will read the file again 
setting `na.strings = ""`.


```r
# Read file
df <- read.table(txt_file, sep = "\t", header = FALSE, skipNul = TRUE, fileEncoding = "UTF-16LE", na.strings = "")

# View data
df[1:5, 1:50]
```

```
##   V1    V2        V3 V4 V5 V6          V7 V8 V9   V10 V11 V12 V13 V14 V15 V16
## 1  1 99050 2099-0707 NA NA NA 2099-0707.2  2 NA S.AUR ANY ANY  NA  -1  NA  NA
## 2  2 99050 2099-0707 NA NA NA 2099-0707.2  2 NA S.AUR ANY ANY  NA  -1  NA  NA
## 3  3 99050 2099-0707 NA NA NA 2099-0707.2  2 NA S.AUR ANY ANY  NA  -1  NA  NA
## 4  4 99057 2099-0031 NA NA NA 2099-0031.1  1 NA E.COL ANY EQU  NA  -1  NA  NA
## 5  5 99058 2099-0129 NA NA NA 2099-0129.1  1 NA E.COL   U CAN  NA  -1  NA  NA
##   V17 V18 V19 V20 V21 V22  V23 V24 V25 V26 V27 V28 V29 V30 V31 V32 V33 V34 V35
## 1  NA  NA   0   0  NA  NA    +  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
## 2  NA  NA   0   0  NA  NA    +  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
## 3  NA  NA   0   0  NA  NA    +  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
## 4  NA  NA   0   0  NA  NA <NA>  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
## 5  NA  NA   0   0  NA  NA <NA>  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
##   V36 V37 V38 V39 V40                 V41    V42        V43    V44    V45
## 1  NA  NA  NA  NA  24 2099-05-25 13:27:05   <NA>       <NA>   <NA>   <NA>
## 2  NA  NA  NA  NA  24 2099-06-01 09:14:28   <NA>       <NA>   <NA>   <NA>
## 3  NA  NA  NA  NA  24 2099-06-01 13:42:48 AMIKAC <=      16 NOINTP AMOCLA
## 4  NA  NA  NA  NA  18 2099-06-02 10:24:16 AMIKAC <=       4   SUSC   <NA>
## 5  NA  NA  NA  NA  24 2099-06-05 08:22:22 AMIKAC <=       4   SUSC AMOCLA
##          V46   V47    V48        V49    V50
## 1       <NA>  <NA>   <NA>       <NA>   <NA>
## 2       <NA>  <NA>   <NA>       <NA>   <NA>
## 3  =     0.5 INTER AMPICI  =       4 RESIST
## 4       <NA>  <NA> AMPICI  >      32 RESIST
## 5  =       4  SUSC AMPICI  =       4   SUSC
```

So, now the blank fields have NA instead.

## Using `utils::read.delim()`

Alternatively, we can read the file with `read.delim()`. We can also omit 
`sep = "\t"` when using this function with this file.


```r
# Read file
df1 <- read.delim(txt_file, header = FALSE, skipNul = TRUE, fileEncoding = "UTF-16LE", na.strings = "")

# Test to see if results are equal
all.equal(df, df1)
```

```
## [1] TRUE
```

## Using `readr::read_tsv()`

We have been using classic "base R" functions from the built-in `utils` package 
to read the file. We can use the `readr` function `read_tsv()` instead. `readr` 
is one of the [core tidyverse packages](https://www.tidyverse.org/packages/). 
The "tsv" in the function name stands for [tab separated values](https://en.wikipedia.org/wiki/Tab-separated_values), so we don't 
need to specify the delimiter.


```r
df2 <- read_tsv(txt_file, locale = locale(encoding = "UTF-16LE"), col_names = FALSE, na = "")
```

The results, after some further processing, are identical:


```r
df1 <- df %>%
    as_tibble() %>%
    mutate(V41 = as.POSIXct(V41, tz = "UTC")) %>%
    mutate(across(where(is.character), ~str_trim(.x)))
df2 <- df2 %>%
    set_names(str_replace_all(names(.), "^X", "V")) %>%
    mutate(across(where(is.character), ~str_trim(.x)))
all.equal(df1, df2)
```

```
## [1] TRUE
```

## Using `readr::read_delim()`

We can also try `read_delim()`, but we need to specify the delimiter, and we 
get some warnings. 


```r
df2 <- read_delim(txt_file, locale = locale(encoding = "UTF-16LE"), col_names = FALSE, na = "", delim = "\t")
```

```
## Warning: One or more parsing issues, call `problems()` on your data frame for details,
## e.g.:
##   dat <- vroom(...)
##   problems(dat)
```

```r
problems()
```

```
## # A tibble: 5,458 × 5
##      row   col expected           actual file                              
##    <int> <int> <chr>              <chr>  <chr>                             
##  1     1     4 1/0/T/F/TRUE/FALSE ""     /tmp/Rtmp76Hngx/file2d7533523b5ac3
##  2     1     5 1/0/T/F/TRUE/FALSE ""     /tmp/Rtmp76Hngx/file2d7533523b5ac3
##  3     1     6 1/0/T/F/TRUE/FALSE ""     /tmp/Rtmp76Hngx/file2d7533523b5ac3
##  4     1     9 1/0/T/F/TRUE/FALSE ""     /tmp/Rtmp76Hngx/file2d7533523b5ac3
##  5     1    13 1/0/T/F/TRUE/FALSE ""     /tmp/Rtmp76Hngx/file2d7533523b5ac3
##  6     1    15 1/0/T/F/TRUE/FALSE ""     /tmp/Rtmp76Hngx/file2d7533523b5ac3
##  7     1    16 1/0/T/F/TRUE/FALSE ""     /tmp/Rtmp76Hngx/file2d7533523b5ac3
##  8     1    17 1/0/T/F/TRUE/FALSE ""     /tmp/Rtmp76Hngx/file2d7533523b5ac3
##  9     1    18 1/0/T/F/TRUE/FALSE ""     /tmp/Rtmp76Hngx/file2d7533523b5ac3
## 10     1    21 1/0/T/F/TRUE/FALSE ""     /tmp/Rtmp76Hngx/file2d7533523b5ac3
## # … with 5,448 more rows
## # ℹ Use `print(n = ...)` to see more rows
```

We can ignore those warnings, since they just relate to the guessed 
column types and the NA value of "". If this function were to guess the 
column types *after* doing the NA conversion, the errors would be avoided, but 
the function does not seem to be smart enough to do that.

The results are the same, however, so we can silence the warnings.


```r
df2 <- read_delim(txt_file, locale = locale(encoding = "UTF-16LE"), col_names = FALSE, na = "", delim = "\t") %>%
    suppressWarnings()

df2 <- df2 %>%
    set_names(str_replace_all(names(.), "^X", "V")) %>%
    mutate(across(where(is.character), ~str_trim(.x)))
all.equal(df1, df2)
```

```
## [1] TRUE
```

Even so, that's still messier than simply using `read_tsv()` with this file. 

## Using `data.table::fread()`

We might also consider using `fread()` from the `data.table` package. 


```r
pacman::p_load(data.table)
```

`fread()` doesn't support UTF-16 encoding, so we need to modify the file 
encoding beforehand with an extra step. We can do this with `readLines()`. 

We also have to remove the BOM, so we'll do that with `str_remove()`. 

There are blank lines alternating with the data lines, and if we don't 
remove them, `fread()` will only read the first line. So, we'll use 
`blank.lines.skip = TRUE` to fix this.


```r
df3 <- readLines(txt_file, encoding = "UTF-16LE", skipNul = TRUE) %>%
    str_remove("^\xff\xfe") %>%
    fread(text = ., sep = "\t", header = FALSE, na.strings = "", blank.lines.skip = TRUE)
```

If we have the [iconv](https://en.wikipedia.org/wiki/Iconv) shell utility 
installed, we can use the `cmd` argument to perform the encoding conversion as 
`fread()` reads the file.


```r
df4 <- fread(cmd = paste("iconv -f UTF-16LE -t UTF-8", txt_file), sep = "\t", header = FALSE, na.strings = "")
all.equal(df3, df4)
```

```
## [1] TRUE
```

We get identical results with either of these `fread()` approaches.

It would be nicer to use R's `iconv()` function instead of the shell utility, 
but the `iconv()` function does not support dropping embedded nulls.

As with `read_delim()`, these two `fread` variations are messier than the earlier 
alternatives, and so it would be difficult to justify using `fread` for this file.

## Using `vroom::vroom()`

The final alternative we will try is `vroom()`. 


```r
pacman::p_load(vroom)
```

We can use the same arguments as for `read_tsv()`, but we can omit the `na` 
argument. We will add `show_col_types = FALSE` to silence the chatter.


```r
df5 <- vroom(txt_file, locale = locale(encoding = "UTF-16LE"), col_names = FALSE, show_col_types = FALSE)

df5 <- df5 %>%
    set_names(str_replace_all(names(.), "^X", "V")) %>%
    mutate(across(where(is.character), ~str_trim(.x)))
all.equal(df1, df5)
```

```
## [1] TRUE
```

We are calling `vroom()` with so few arguments, it's interesting that it still 
gives equivalent results.

`vroom()` is [supposed to be faster](https://vroom.r-lib.org/) than any of the 
previous alternatives we have tried. And it's straightforward to use. So, that's 
probably the best choice, at least for larger files.

## Auto-detecting file encoding

If all we did to investigate the file before running any R was to open it in 
Notepad, and if we only saw that the file appeared to be whitespace delimited and 
lacked a header line, then we could have used a more generic `vroom()` approach. 

That is, if we test for encoding first, then we can read this file without knowing 
anything about its encoding, embedded nulls, or tab delimiter.


```r
guess_locale <- function(file, n_max = 10000, threshold = 0.2, ...) {
    require(readr)
    file_locale <- readr::locale()
    file_encoding <- readr::guess_encoding(file, n_max = 1, threshold = threshold)[1, ]$encoding
    if (!is.na(file_encoding) & is.character(file_encoding) & str_length(file_encoding) > 0) {
        file_locale <- readr::locale(encoding = file_encoding, ...)
    }
    return(file_locale)
}

df5 <- vroom(txt_file, locale = guess_locale(txt_file), col_names = FALSE, show_col_types = FALSE)

df5 <- df5 %>%
    set_names(str_replace_all(names(.), "^X", "V")) %>%
    mutate(across(where(is.character), ~str_trim(.x)))
all.equal(df1, df5)
```

```
## [1] TRUE
```

In other words, if we just made `vroom()` a little smarter, smart enough to guess 
the encoding by itself, then we could ignore the encoding issue entirely. 

The locale also contains other default values (like Timezone) which may not be 
appropriate for our file, so we would still want to be aware of those aspects.

## Check dimensions


```r
# Check dimensions
dim(df)
```

```
## [1]  20 341
```

That's a lot of columns! Maybe more than we need...

# Remove columns with no data

When we looked at the lines, we saw lots of empty values, so let's try to 
remove columns that contain no data, as they are just taking up space.


```r
# Remove columns containing only NA values
df <- df %>%
    select(where(~sum(is.na(.x)) < length(.x)))

# Check dimensions
dim(df)
```

```
## [1]  20 146
```

```r
# View data
df[1:5, 1:20]
```

```
##   V1    V2        V3          V7 V8   V10 V11 V12 V14 V19 V20  V23 V40
## 1  1 99050 2099-0707 2099-0707.2  2 S.AUR ANY ANY  -1   0   0    +  24
## 2  2 99050 2099-0707 2099-0707.2  2 S.AUR ANY ANY  -1   0   0    +  24
## 3  3 99050 2099-0707 2099-0707.2  2 S.AUR ANY ANY  -1   0   0    +  24
## 4  4 99057 2099-0031 2099-0031.1  1 E.COL ANY EQU  -1   0   0 <NA>  18
## 5  5 99058 2099-0129 2099-0129.1  1 E.COL   U CAN  -1   0   0 <NA>  24
##                   V41    V42        V43    V44    V45        V46   V47
## 1 2099-05-25 13:27:05   <NA>       <NA>   <NA>   <NA>       <NA>  <NA>
## 2 2099-06-01 09:14:28   <NA>       <NA>   <NA>   <NA>       <NA>  <NA>
## 3 2099-06-01 13:42:48 AMIKAC <=      16 NOINTP AMOCLA  =     0.5 INTER
## 4 2099-06-02 10:24:16 AMIKAC <=       4   SUSC   <NA>       <NA>  <NA>
## 5 2099-06-05 08:22:22 AMIKAC <=       4   SUSC AMOCLA  =       4  SUSC
```

## Remove rows with no drug data

By inspection, we see the drug data (name, quantity, etc.) starts at column V42. 
Some rows do not have any data from that column onward, so we will remove those 
rows, as we don't need them.


```r
# Find the numerical index of column V42 to use for subsetting
drug_start <- grep("V42", names(df))
drug_start
```

```
## [1] 15
```

```r
# Remove rows containing no drug data, e.g., where columns V42 onward are all NA
df <- df %>%
    filter(!if_all(all_of(drug_start:ncol(df)), ~is.na(.)))

# Check dimensions
dim(df)
```

```
## [1]  18 146
```

## Examine structure of drug data

When we look at the columns of drug data, we see a strange structure.


```r
# View data
df %>%
    select(V42:V50) %>%
    head()
```

```
##      V42        V43    V44    V45        V46    V47    V48        V49    V50
## 1 AMIKAC <=      16 NOINTP AMOCLA  =     0.5  INTER AMPICI  =       4 RESIST
## 2 AMIKAC <=       4   SUSC   <NA>       <NA>   <NA> AMPICI  >      32 RESIST
## 3 AMIKAC <=       4   SUSC AMOCLA  =       4   SUSC AMPICI  =       4   SUSC
## 4 AMIKAC <=       4   SUSC AMOCLA  =       2   SUSC AMPICI  >       8 RESIST
## 5 AMIKAC  >      32 RESIST AMOCLA  >       8 RESIST AMPICI  >       8 RESIST
## 6 AMIKAC  >      32 RESIST AMOCLA  >       8 RESIST AMPICI  >       8 RESIST
```

By inspection, we see columns containing drug data are in sets of 3:
drug_name drug_quantity drug_qualifier, example: "AMIKAC" " <= 16" "NOINTP".

## Reshape drug columns

We would prefer a structure where there were two colums per drug: one for the 
quantity and the other for the qualifier, which we will call "RIS". We want 
the two colums to be named after the drug, which is contained in the first 
of the three columns for each set. So, we will extract these first.


```r
# Identify columns containing only a single abbreviated drug name (or NA)
drug_cols <- names(df)[drug_start:ncol(df)][drug_start:ncol(df)%%3 == 0]
col_vals <- map(drug_cols, ~table(df[, .x])) %>%
    setNames(drug_cols)

# Confirm all of these variables only contain 1 value matching name pattern
table(unlist(map(col_vals, length)))
```

```
## 
##  1 
## 44
```

```r
table(unlist(map(col_vals, ~grepl("^[A-Z]+$", names(.x)[1]))))
```

```
## 
## TRUE 
##   44
```

```r
# Extract the drug names and column number
drugs <- sapply(col_vals, names)
drug_nums <- as.numeric(gsub("^V", "", drug_cols))

# Rename drug columns in sets of 3 columns for each drug: name, value, and RIS
drugs_df <- tibble(drug = drugs, drug_num = drug_nums) %>%
    mutate(col1_old = paste0("V", drug_nums), col2_old = paste0("V", drug_nums + 1), col3_old = paste0("V",
        drug_nums + 2)) %>%
    mutate(col1_new = paste0(drug, "_NAME"), col2_new = paste0(drug, "_VAL"), col3_new = paste0(drug,
        "_RIS"))
lookup <- with(drugs_df, c(col1_old, col2_old, col3_old) %>%
    setNames(c(col1_new, col2_new, col3_new)))
df <- rename(df, any_of(lookup))
```

## Final cleanup

Now, all we need to do is remove extra spaces from the value columns, abbreviate 
the names of the drug value columns, and remove the original drug name columns. 
We won't worry about providing column names for the non-drug columns right now.


```r
# Clean up data
df <- df %>%
    mutate(across(ends_with("_VAL"), ~str_trim(str_replace_all(.x, "\\s+", " ")))) %>%
    rename_with(~str_remove(., "_VAL")) %>%
    select(-ends_with("_NAME"))

# View results
df[1:5, 15:20]
```

```
##   AMIKAC AMIKAC_RIS AMOCLA AMOCLA_RIS AMPICI AMPICI_RIS
## 1  <= 16     NOINTP  = 0.5      INTER    = 4     RESIST
## 2   <= 4       SUSC   <NA>       <NA>   > 32     RESIST
## 3   <= 4       SUSC    = 4       SUSC    = 4       SUSC
## 4   <= 4       SUSC    = 2       SUSC    > 8     RESIST
## 5   > 32     RESIST    > 8     RESIST    > 8     RESIST
```

After saving the results to a CSV file, we are done.


```r
# Save result as CSV
write_csv(df, csv_file)
```

## Appendix I: What's an "embedded null"?

A "embedded null" is a non-printing character (NUL) that is sometimes used to 
terminate a character string. You can see them using `readBin()`. This will 
show the hexadecimal equivalents of the characters. The nulls are the 00 values.


```r
x <- readBin(txt_file, what = "raw", n = 16)
x
```

```
##  [1] ff fe 31 00 09 00 39 00 39 00 30 00 35 00 30 00
```

And we can "decode" the hexadecimal characters with `rawToChar()`, if we remove
the nulls first:


```r
paste(rawToChar(x[x != 0], multiple = TRUE), collapse = " ")
```

```
## [1] "\xff \xfe 1 \t 9 9 0 5 0"
```

## Appendix II: What's UTF-16LE?

UTF-16 is a file encoding for 16-bit (2-byte) Unicode characters. We suspected we 
had an UTF-16LE file because of the two special bytes starting the file. These 
particular bytes, `FF FE` specify that the encoding is "little endian" (LE), giving us 
UTF-16LE encoding. 

- See: https://en.wikipedia.org/wiki/Endianness
- And: https://www.tutorialspoint.com/big-endian-and-little-endian

If we had a UTF-32LE file, it would have started with `FF FE 00 00`. If the file 
was UTF-8, it would either not have a BOM or the BOM would be `EF BB BF`. 

Because UTF-8 is represented with single bytes, there is no big or little "end". 
Endianness is to specify byte order, so you need more than one byte per character to 
have a byte order. So, the difference between "LE" and "BE" are just the order 
of the two bytes in each multibyte character. We can see the difference with 
an example that compares the bytes of LE and BE representing the character "1":


```r
x <- "1"
Encoding(x) <- "UTF-8"
x8 <- charToRaw(x)
x16LE <- iconv(x, "UTF-8", "UTF-16LE", toRaw = TRUE)
x16BE <- iconv(x, "UTF-8", "UTF-16BE", toRaw = TRUE)
kable(t(c(x = x, `UTF-8` = x8, `UTF-16LE` = x16LE, `UTF-16BE` = x16BE)))
```



|x  |UTF-8        |UTF-16LE              |UTF-16BE              |
|:--|:------------|:---------------------|:---------------------|
|1  |as.raw(0x31) |as.raw(c(0x31, 0x00)) |as.raw(c(0x00, 0x31)) |

Those `00` values look like the dreaded "embedded nulls", right? But now we see 
those are valid after all. They are simply one of the two bytes of some UTF-16 
characters. That's why the warning said, *appears* to contain embedded nulls. 

## Appendix III: Good nulls and bad nulls

It turns out we actually need to "skip" the embedded nulls (`0x00 0x00`), or we 
will just read a few values.


```r
read.table(txt_file, nrows = 1, fileEncoding = "UTF-16LE") %>%
    suppressWarnings()
```

```
##   V1    V2        V3
## 1  1 99050 2099-0707
```

The reason is that we actually do have real embedded nulls after the third "cell".


```r
readBin(txt_file, what = "raw", n = 100)
```

```
##   [1] ff fe 31 00 09 00 39 00 39 00 30 00 35 00 30 00 09 00 32 00 30 00 39 00 39
##  [26] 00 2d 00 30 00 37 00 30 00 37 00 09 00 00 00 09 00 00 00 09 00 00 00 09 00
##  [51] 32 00 30 00 39 00 39 00 2d 00 30 00 37 00 30 00 37 00 2e 00 32 00 09 00 32
##  [76] 00 09 00 00 00 09 00 53 00 2e 00 41 00 55 00 52 00 09 00 41 00 4e 00 59 00
```

You can see that between some tabs (`\t` or `09 00`) we have `00 00` values. These
are the real embedded nulls because there are two of them together, so 16 bits of 
zeros.

I'm guessing these are the actual `NA` values of this file format. Since we don't 
have column headings, or any code book at all, we can't be sure, but if the tab 
is actually the delimiter, then this is how the empty "cells" are presented in 
this file. So, maybe we can try setting the null as a NA value:


```r
try(read.table(txt_file, nrows = 1, fileEncoding = "UTF-16LE", na.strings = "\x00"))
```

```
Error: nul character not allowed (line 1)
```

Unfortunately we are not allowed to enter nulls here. So, we just skip them. 
That will leave the empty cells truly empty ("") and so we use the empty 
string as the NA string with `na.strings = ""`. 

At least now we have a better idea of what embedded nulls are, why 
they're there (probably), and why it's (probably) okay to skip them, at least in 
this situation.
