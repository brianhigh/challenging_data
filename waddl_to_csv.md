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

We will use an example "text" data file, a modified subset of a file from the 
[Washington Animal Disease Diagnostic Laboratory](https://waddl.vetmed.wsu.edu/) (WADDL).

This file format is challenging because the functions you would normally 
use for reading "text" files will not work without using some special options. 
Further, the data are not arranged in familiar "columns are variables" structure.

We will explore how to address the issues that come up when reading this file: 

- Unexpected "embedded nulls" error
- Unexpected "invalid multibyte string" error
- Unexpected file encoding type
- Unexpected delimiter (tab)
- Missing "elements" (values)
- Missing header
- Columns with no values (to remove)
- Rows missing any useful data (to remove)
- Unusual data structure (not "columns are variables")

## Setup

First, we will load our R packages, set options, and set file paths.


```r
# Attach packages, installing as needed
if (!requireNamespace("pacman", quietly = TRUE)) install.packages("pacman")
pacman::p_load(formatR, knitr, readr, dplyr, tidyr, stringr,
    purrr)

# Set options
options(readr.show_col_types = FALSE, readr.show_progress = FALSE)

# Set file paths
txt_file <- file.path("data", "waddl_data_example.txt")
csv_file <- file.path("data", "waddl_data_example.csv")
```

## Embedded Nulls

We expect a normal "text" file, because of the file suffix: `.txt`. If we knew it 
was a CSV file, we would use `read.csv()`, but from the `.txt` suffix we do not 
know the delimiter. It could be a comma, tab, space, or just about anything. So, 
we will start by trying `read.table()`. 

But when we read it in we get some unexpected error messages.


```r
# Try reading with read.table, just the first row
try(read.table(txt_file, nrows = 1))
```

```
## Warning in read.table(txt_file, nrows = 1): line 1 appears to contain embedded
## nulls
```

```
## Error in type.convert.default(data[[i]], as.is = as.is[i], dec = dec,  : 
##   invalid multibyte string at '<ff><fe>1'
```

The error about "embedded nulls" can be easily dealt with using `skipNul = TRUE`.


```r
# Try reading with read.table, just the first row
try(read.table(txt_file, nrows = 1, skipNul = TRUE))
```

```
## Error in type.convert.default(data[[i]], as.is = as.is[i], dec = dec,  : 
##   invalid multibyte string at '<ff><fe>1'
```

## File encoding

The "invalid multibyte string" error is a clue that the file is not encoded as 
expected (usually UTF-8 or ASCII), so we check to see what it might be instead.


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

And the guesses are listed with the most likely encoding at the the top of the 
list with a probability of 1. We can verify that by checking the first two bytes 
for the "byte order mark" (BOM),


```r
# Look for a byte order mark (BOM) in the first two bytes
# to verify encoding
readBin(txt_file, what = "raw", n = 2)
```

```
## [1] ff fe
```

We see a BOM of `ff fe`, verifying the UTF-16LE encoding.

- See: https://www.unicode.org/faq/utf_bom.html#bom4
- And: https://en.wikipedia.org/wiki/Byte_order_mark#UTF-16

When we set `fileEncoding = "UTF-16LE"`, reading the first line, there are no 
errors. The BOM was removed automatically.


```r
read.table(txt_file, nrows = 1, fileEncoding = "UTF-16LE", skipNul = TRUE)
```

```
##   V1    V2        V3          V4 V5    V6  V7  V8 V9 V10 V11 V12 V13        V14
## 1  1 99050 2099-0707 2099-0707.2  2 S.AUR ANY ANY -1   0   0   +  24 2099-05-25
##        V15
## 1 13:27:05
```

## Read more rows

We get an error when we try to read more rows.


```r
# Let's try reading more rows
try(read.table(txt_file, nrows = 5, fileEncoding = "UTF-16LE",
    skipNul = TRUE))
```

```
## Error in scan(file = file, what = what, sep = sep, quote = quote, dec = dec,  : 
##   line 1 did not have 111 elements
```

This error means that some rows are missing some values. We will look at whole 
lines to see what's going on. Let's start with the first five lines.


```r
# Let's read in whole lines to see what they really look
# like
readLines(txt_file, encoding = "UTF-16LE", n = 5, skipNul = TRUE)
```

```
## [1] "\xff\xfe1\t99050\t2099-0707\t\t\t\t2099-0707.2\t2\t\tS.AUR\tANY\tANY\t\t-1\t\t\t\t\t0\t0\t\t\t+\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t24\t2099-05-25 13:27:05\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t"                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     
## [2] ""                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     
## [3] "2\t99050\t2099-0707\t\t\t\t2099-0707.2\t2\t\tS.AUR\tANY\tANY\t\t-1\t\t\t\t\t0\t0\t\t\t+\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t24\t2099-06-01 09:14:28\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t"                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             
## [4] ""                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     
## [5] "3\t99050\t2099-0707\t\t\t\t2099-0707.2\t2\t\tS.AUR\tANY\tANY\t\t-1\t\t\t\t\t0\t0\t\t\t+\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t24\t2099-06-01 13:42:48\tAMIKAC\t<=      16\tNOINTP\tAMOCLA\t =     0.5\tINTER\tAMPICI\t =       4\tRESIST\t\t\t\tCEFAZO\t<=       2\tSUSC\tCEFOVE\t =       4\tRESIST\tCEFPOD\t =       4\tINTER\t\t\t\t\t\t\t\t\t\tCEPHAL\t<=       2\tSUSC\tCHLORA\t<=       8\tSUSC\t\t\t\t\t\t\tCLINDA\t<=     0.5\tSUSC\t\t\t\tDOXYCY\t<=    0.12\tSUSC\tENROFL\t<=    0.25\tNOINTP\tERYTH\t<=    0.25\tSUSC\t\t\t\tGENTAM\t<=       4\tSUSC\tIMIPEN\t<=       1\tSUSC\tMARBOF\t<=       1\tNOINTP\tMINOCY\t<=     0.5\tSUSC\t\t\t\tNITRO\t<=      16\tSUSC\t\t\t\tOXACIL\t =       2\tSUSC\t\t\t\tPENICI\t >       8\tRESIST\t\t\t\tPRADOF\t<=    0.25\tNOINTP\tRIFAMP\t<=       1\tSUSC\t\t\t\t\t\t\tTETRA\t<=    0.25\tSUSC\t\t\t\t\t\t\t\t\t\t\t\t\tTRISUL\t<=       2\tSUSC\t\t\t\t\t\t\tVANCOM\t<=       1\tSUSC\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t\t"
```

By inspection, we see the delimiter is tab (`\t`), there are blank lines, and 
no headers, with no specific value for "missingness", just the delimiter with
no value of any kind. When read into R, this would be the empty string ("").

## Set NA strings

Let's use what we know now about the file structure and try again using 
`na.strings = ""` to tell R that an empty string should be converted to `NA`.


```r
df <- try(read.table(txt_file, header = FALSE, na.strings = "",
    fileEncoding = "UTF-16LE", skipNul = TRUE))
```

```
## Error in scan(file = file, what = what, sep = sep, quote = quote, dec = dec,  : 
##   line 1 did not have 111 elements
```

We will get the same error as before. That's because `read.table()` needs to know 
the delimiter with `sep = "\t"`.

## Setting the delimiter

We don't have to set `blank.lines.skip = TRUE`, since that's the default, but we 
do need to set `sep = "\t"`.


```r
# Read file as tab-delimited UTF-16LE, skipping nulls, with
# '' set to NA
df <- read.table(txt_file, header = FALSE, na.strings = "", sep = "\t",
    fileEncoding = "UTF-16LE", skipNul = TRUE)
```

If we use `read.delim()` instead of `read.table()`, we don't have to specify either 
`sep = "\t"` or `blank.lines.skip = TRUE` since the defaults will cover those.


```r
# Read file as whitepace-delimited UTF-16LE, skipping
# nulls, with '' set to NA
df <- read.delim(txt_file, header = FALSE, na.strings = "", fileEncoding = "UTF-16LE",
    skipNul = TRUE)
```

## Using `readr::read_delim`

We have been using `base`-R functions to read the file. We could use `readr` 
functions instead, but they do not handle the embedded nulls. So, you could 
work around that by processing with `base`-R's `readLines`  and `writeLines` 
first.


```r
# Do the same with readr::read_delim(), but after using
# readLines/writeLines since read_delim can't handle the
# embedded nulls
temp <- tempfile()
readLines(txt_file, encoding = "UTF-16LE", skipNul = TRUE) %>%
    writeLines(temp)
df <- read_delim(temp, delim = "\t", col_names = FALSE, na = "",
    name_repair = make.names)
res <- file.remove(temp)
```

We might also consider using `fread` from the `data.table` package, but it 
doesn't support UTF-16 encoding. So, again, we would be forced to read the 
file with something else first. It's hard to justify the extra trouble.

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
Some rows do not have any data from that column onwards, so we will remove those 
rows, as we don't need them.


```r
# Find the numerical index of column V42 to use for
# subsetting
drug_start <- grep("V42", names(df))
drug_start
```

```
## [1] 15
```

```r
# Remove rows containing no drug data, e.g., where columns
# V42 onward are all NA
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
# Identify columns containing only a single abbreviated
# drug name (or NA)
drug_cols <- names(df)[drug_start:ncol(df)][drug_start:ncol(df)%%3 ==
    0]
col_vals <- map(drug_cols, ~table(df[, .x])) %>%
    setNames(drug_cols)

# Confirm all of these variables only contain 1 value
# matching name pattern
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

# Rename drug columns in sets of 3 columns for each drug:
# name, value, and RIS
drugs_df <- data.frame(drug = drugs, drug_num = drug_nums) %>%
    mutate(col1_old = paste0("V", drug_nums), col2_old = paste0("V",
        drug_nums + 1), col3_old = paste0("V", drug_nums + 2)) %>%
    mutate(col1_new = paste0(drug, "_NAME"), col2_new = paste0(drug,
        "_VAL"), col3_new = paste0(drug, "_RIS"))
lookup <- with(drugs_df, c(col1_old, col2_old, col3_old) %>%
    setNames(c(col1_new, col2_new, col3_new)))
df <- rename(df, any_of(lookup))
```

## Final cleanup

Now, all we need to do is remove extra spaces from the value columns, abbreviate 
the names of the drug value columns, and remove the original drug name columns. 
We won't worry about proving column names for the non-drug columns right now.


```r
# Clean up data
df <- df %>%
    mutate(across(ends_with("_VAL"), ~str_replace_all(.x, "\\s+",
        " "))) %>%
    rename_with(~str_remove(., "._VAL")) %>%
    select(-ends_with("_NAME"))

# Save result as CSV
write_csv(df, csv_file)
```

After saving the results to a CSV file, we are done.
